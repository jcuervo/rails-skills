# Inbound Email (Action Mailbox)

The **inbound** counterpart to the rest of this skill: Action Mailbox routes
incoming email to mailbox classes the way the router sends requests to controllers.
Mail from an ingress (a provider webhook or an SMTP relay) becomes an
`ActionMailbox::InboundEmail`, gets routed by pattern, and is processed by a mailbox.

## Quick Pattern

```bash
bin/rails action_mailbox:install      # creates app/mailboxes/application_mailbox.rb + migrations
bin/rails db:migrate                  # InboundEmail table + Active Storage (raw email is stored as a blob)
bin/rails g mailbox forwards          # app/mailboxes/forwards_mailbox.rb
```
```ruby
# app/mailboxes/application_mailbox.rb — the routing table (pattern => mailbox)
class ApplicationMailbox < ActionMailbox::Base
  routing(/^support@/i  => :support)
  routing(/@replies\./i => :replies)
  # routing :all => :default        # catch-all
end
```
```ruby
# app/mailboxes/support_mailbox.rb
class SupportMailbox < ApplicationMailbox
  def process
    ticket = Ticket.create!(
      from:    mail.from.first,           # `mail` is a Mail::Message
      subject: mail.subject,
      body:    mail.decoded,              # or mail.text_part / mail.html_part
    )
    mail.attachments.each { |a| ticket.files.attach(io: StringIO.new(a.decoded), filename: a.filename) }
  end
end
```

Inbound email is stored via **Active Storage** (`../rails-storage/`) — the raw
message is kept as a blob, so Active Storage must be installed (the installer wires
it). Attachments on the message are read from `mail.attachments`; persisting them is
`../rails-storage/`.

## When to pick / not pick

- **Pick** Action Mailbox when your app must *receive and act on* email — support
  tickets from inbound mail, reply-by-email threads, parsing forwarded receipts,
  ingesting documents sent to an address.
- **Don't pick** it if you only *send* (the rest of this skill), or for fetching mail
  from an existing IMAP/POP mailbox you poll yourself (Action Mailbox is push, via an
  ingress that the provider/MTA calls) — that's a different integration.

## Configure an ingress

The ingress is **dictated by who receives your mail** — match the provider holding
your MX record (or the relay in front of your app), not a personal preference.

```ruby
# config/environments/production.rb
config.action_mailbox.ingress = :relay   # :relay | :mailgun | :mandrill | :postmark | :sendgrid
                                          # built-in MTA paths: :exim | :postfix | :qmail
```

| Ingress | How mail arrives | Credential to set |
|---|---|---|
| `:relay` | Your MTA/SMTP relay POSTs to `/rails/action_mailbox/relay/inbound_emails` | `action_mailbox.ingress_password` |
| `:postmark` | Postmark inbound webhook | `action_mailbox.ingress_password` |
| `:sendgrid` | SendGrid Inbound Parse webhook | `action_mailbox.ingress_password` |
| `:mailgun` | Mailgun route → webhook | `action_mailbox.mailgun_signing_key` |
| `:mandrill` | Mandrill inbound webhook | `action_mailbox.mandrill_api_key` |

`:relay`/`:postmark`/`:sendgrid` authenticate with HTTP basic auth (`action_mailbox.ingress_password`);
`:mailgun`/`:mandrill` verify a provider signature with their key. The exact credential keys are
version-specific — confirm against the [Action Mailbox guide](https://guides.rubyonrails.org/action_mailbox_basics.html)
for your Rails version. Set them in encrypted credentials (`../rails-security/`), never in code:

```bash
bin/rails credentials:edit
```
```yaml
# add to the encrypted file:
action_mailbox:
  ingress_password: <generated>
```

Run `bin/rails action_mailbox:install --help` (and `bin/rails g mailbox --help`) to
confirm the generators' current options for your Rails version before running them.

## Deep Dive

- **Routing** lives only in `ApplicationMailbox`; patterns match against
  `to`/`cc`/`bcc` (regex, string, `->(inbound){}` lambda, or a mailbox responding to
  `match?`). First match wins; add a `:all` catch-all last.
- **`process`** is the one method a mailbox implements. Inside it, `mail` is a
  [`Mail::Message`](https://github.com/mikel/mail) — `mail.from`, `mail.subject`,
  `mail.text_part`, `mail.html_part`, `mail.attachments`.
- **Callbacks + lifecycle:** `before_processing`/`after_processing`, and
  `bounced!` / `delivered!` move the `InboundEmail` through its status
  (`pending → processing → delivered | failed | bounced`). Bounce with a real mailer
  reply via `bounce_with SomeMailer.message(...)` (or `bounce_now_with` to send
  synchronously) — that bounce is an *outbound* mail, built with the rest of this
  skill.
- **Idempotency:** processing can be retried; key off `mail.message_id` so a
  redelivered message doesn't create duplicate records.
- **Local development** uses the built-in conductor — no provider needed (see Verify).

## Common Pitfalls

- **Active Storage not set up** → install fails or raw emails can't be stored; the
  installer adds it, but on an existing app confirm `bin/rails active_storage:install`
  has run. (`../rails-storage/`)
- **Heavy work inline in `process`** → blocks the ingress request; offload to a job
  (`../rails-jobs/`) and keep `process` thin.
- **No `:all` / no matching route** → mail silently goes unprocessed; add a catch-all.
- **Trusting unauthenticated webhooks** → set the ingress password/signing key so a
  forged POST can't inject email (`../rails-security/`).
- **Non-idempotent processing** → provider retries create duplicates; dedupe on
  `message_id`.

## Verify

```bash
# Route a test message straight through (no provider needed) and confirm it was processed.
# This is the public entry point — preferred over poking at router internals, which are private.
bin/rails runner "ApplicationMailbox.route Mail.new(to: 'support@x.com', from: 'a@b.com', subject: 'Test', body: 'hello')" 2>/dev/null
bin/rails runner "p ActionMailbox::InboundEmail.count; p Ticket.where(subject: 'Test').exists?" 2>/dev/null   # InboundEmail created + SupportMailbox ran

# The dev conductor (compose + deliver inbound mail by hand) is reachable:
bin/rails server -d && sleep 3
curl -s -o /dev/null -w "conductor=%{http_code}\n" localhost:3000/rails/conductor/action_mailbox/inbound_emails   # 200 in development
kill "$(cat tmp/pids/server.pid)"
```

Then route to `../rails-storage/` (attachments + raw-email blobs), `../rails-jobs/`
(process asynchronously), `../rails-security/` (ingress auth), and `../rails-testing/`
(`ActionMailbox::TestHelper#receive_inbound_email_from_mail`).

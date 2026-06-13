---
name: rails-mailers
description: Send transactional email from a Rails 8.1 app with Action Mailer — mailer classes, multipart HTML+text templates, layouts, attachments, inline images, previews, and delivery via SMTP (the Rails default, pointed at any provider) or Postmark / SendGrid / SES. Menu-driven delivery selection with a Recommended default; detects any configured delivery method and credentials first, sends asynchronously through Active Job (deliver_later), and verifies mail renders and delivers in development. Apply when adding a mailer, building email templates, configuring email delivery, attaching files to email, or previewing/testing mail.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[mailer-or-email]"
---

# rails-mailers

## Purpose

Owns **transactional email** through Action Mailer: the mailer classes, multipart
templates (HTML + text), layouts, attachments/inline images, previews, and the
delivery configuration that puts mail on the wire. It sends **asynchronously** by
default via Active Job (`deliver_later` — the queue is `../rails-jobs/`). Applies to
API-only and full-stack apps alike (password resets, receipts, notifications).

## When to Apply

Use this skill when the task is:

- Creating a mailer and its actions (welcome, password reset, receipt)
- Building email templates (HTML + plain-text, layouts, inline images)
- Configuring delivery (SMTP, Postmark, SendGrid, SES) and credentials
- Attaching files to an email
- Previewing or testing email rendering/delivery

Do **not** use this skill when the task is:

- The job queue that runs `deliver_later` → read `../rails-jobs/SKILL.md` (this skill builds the mail; that one runs the delivery job)
- The password-reset *auth flow* that triggers the email → read `../rails-auth/SKILL.md` (it calls the mailer; this skill builds it)
- Marketing/bulk campaigns (this is **transactional** email) — out of suite scope
- The model data in the email → read `../rails-models/SKILL.md`
- Attaching uploaded files via Active Storage → read `../rails-storage/SKILL.md` (then attach in the mailer)
- Production deliverability ops, bounce/webhook handling, monitoring → read `../rails-deploy/SKILL.md`
- Secrets/credentials handling in depth → read `../rails-security/SKILL.md`
- Testing mailers → read `../rails-testing/SKILL.md`

## Detect Before You Generate

```bash
grep -rnE "delivery_method|smtp_settings|postmark|sendgrid|aws_sdk|ses" config/environments/*.rb config/initializers/*.rb 2>/dev/null
grep -E "^    (postmark-rails|sendgrid|aws-sdk-rails|aws-sdk-ses|mail)" Gemfile.lock
ls app/mailers app/views/*_mailer 2>/dev/null               # existing mailers + templates
grep -rn "default_url_options" config/environments/*.rb     # URLs in emails need a host
```

- If a delivery method is **already configured**, use it — don't re-ask the menu.
- `config.action_mailer.delivery_method` per environment tells you the current
  transport (`:smtp`, `:test`, a provider). Development defaults to not actually
  sending; previews + `letter_opener` are common.
- Mailer links need `config.action_mailer.default_url_options = { host: ... }` set
  per environment — check it's present before building templated URLs.
- Honor any email provider recorded in `STACK.md`.

## Menu

One delivery menu (the rest — templating, attachments, previews — are patterns, not
picks). Via `AskUserQuestion`, Rails-default marked **Recommended**; ask unless
configured.

### Delivery method
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **SMTP** *(Recommended)* | Rails default transport; point it at any provider's SMTP (Postmark, SES, Gmail, Mailgun). Universal, simple, provider-portable. | [delivery.md](references/delivery.md) |
| Postmark (API) | Transactional-focused, excellent deliverability + analytics via `postmark-rails`. Provider lock to Postmark. | [delivery.md](references/delivery.md) |
| SendGrid | Ubiquitous; via SMTP or `sendgrid-actionmailer`. Good scale, more config. | [delivery.md](references/delivery.md) |
| Amazon SES | Cheapest at scale; via SMTP or `aws-sdk-rails`. AWS setup + sandbox/verification overhead. | [delivery.md](references/delivery.md) |

## Decision Flow

- **Delivery:** default to **SMTP** — it's the framework default and works with every
  provider, so you stay portable (swap hosts by changing `smtp_settings`). Choose a
  **provider gem (Postmark/SES/SendGrid API)** when you want that provider's
  deliverability features, analytics, or API-based sending over SMTP. Either way,
  credentials live in encrypted credentials, never in code (`../rails-security/`).
- **Async by default:** `deliver_later` (enqueues via `../rails-jobs/`) so a slow
  mail server never blocks the request. Use `deliver_now` only for scripts/tests or
  when you specifically need synchronous send.
- **Always multipart:** ship an HTML **and** a text part — see
  [writing-mailers.md](references/writing-mailers.md). Text-only-missing hurts
  deliverability and accessibility.
- **Templated URLs need a host** per environment; set `default_url_options`.

## Problem → Reference

| Task | Read |
|---|---|
| Configure delivery (SMTP / Postmark / SendGrid / SES) + credentials | [references/delivery.md](references/delivery.md) |
| Write a mailer: actions, multipart templates, layouts, attachments, deliver_later | [references/writing-mailers.md](references/writing-mailers.md) |
| Preview emails + test mail rendering/delivery | [references/previews-and-testing.md](references/previews-and-testing.md) |

## Verify

A mailer change is unfinished until mail renders both parts and "delivers" in dev:

```bash
# Renders without error and produces HTML + text parts:
bin/rails runner "m = UserMailer.welcome(User.first || User.new(email_address:'a@b.com')); puts m.subject; puts m.parts.map(&:content_type)" 2>/dev/null
# Delivery is captured in development (test/letter_opener) rather than really sent:
bin/rails runner "ActionMailer::Base.delivery_method=:test; UserMailer.welcome(User.first || User.new(email_address: 'a@b.com')).deliver_now; p ActionMailer::Base.deliveries.size" 2>/dev/null   # => 1
# Preview is reachable in dev:
bin/rails server -d && sleep 3
curl -s -o /dev/null -w "preview=%{http_code}\n" "localhost:3000/rails/mailers"      # 200 in development
kill "$(cat tmp/pids/server.pid)"
```

Then route to `../rails-jobs/` (the delivery queue), `../rails-testing/` (mailer
specs + preview-as-test), and record the provider in `STACK.md`.

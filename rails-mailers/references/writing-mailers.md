# Writing Mailers

Mailer classes, multipart HTML+text templates, layouts, attachments, and async
delivery. The rules: **always ship both an HTML and a text part**, and **send with
`deliver_later`** so mail never blocks a request.

## Quick Pattern

```bash
bin/rails g mailer User welcome password_reset      # app/mailers/user_mailer.rb + views + preview
```
```ruby
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  default from: "no-reply@example.com"
  def welcome(user)
    @user = user                                    # instance vars are available in templates
    @url  = dashboard_url                           # needs default_url_options host (delivery.md)
    mail to: @user.email_address, subject: "Welcome!"
  end
end
```
```erb
<%# app/views/user_mailer/welcome.html.erb  — HTML part %>
<h1>Welcome, <%= @user.email_address %></h1>
<p><%= link_to "Open your dashboard", @url %></p>
```
```erb
<%# app/views/user_mailer/welcome.text.erb  — TEXT part (same content, plain) %>
Welcome, <%= @user.email_address %>
Open your dashboard: <%= @url %>
```

Two template files with the **same action name** (`welcome.html.erb` +
`welcome.text.erb`) → Action Mailer assembles a **multipart/alternative** message
automatically. Ship both.

## Sending (async by default)

```ruby
UserMailer.welcome(user).deliver_later          # enqueues a mailer job (../rails-jobs/) — preferred
UserMailer.welcome(user).deliver_now            # synchronous — scripts/tests only
UserMailer.welcome(user).deliver_later(wait: 1.hour)   # delayed
```

`deliver_later` runs on your Active Job backend; a slow mail server then never blocks
the request. The queue/retry behavior is `../rails-jobs/`.

## Layouts, attachments, inline images

```ruby
# Layout: app/views/layouts/mailer.html.erb + mailer.text.erb wrap every email.
class UserMailer < ApplicationMailer
  def receipt(order)
    @order = order
    attachments["receipt-#{order.id}.pdf"] = order.render_pdf          # file attachment
    attachments.inline["logo.png"] = File.read(Rails.root.join("app/assets/images/logo.png"))
    mail to: order.user.email_address, subject: "Your receipt"
  end
end
```
```erb
<%# reference an inline image in the HTML part: %>
<%= image_tag attachments["logo.png"].url %>
```

- **Active Storage attachments:** to attach an uploaded file, read its bytes and
  assign to `attachments[...]` — the upload itself is `../rails-storage/`.
- **`mail` parameters:** `to`, `cc`, `bcc`, `from`, `reply_to`, `subject`.

## Deep Dive

- **`ApplicationMailer`** is the shared base (default from, layout) — put common
  config there.
- **Parameterized mailers:** `UserMailer.with(user: user).welcome` exposes `params`
  in the action — cleaner for shared before-actions.
- **i18n:** subjects/bodies localize via locale files; the subject can be omitted in
  `mail` and resolved from `config/locales`.
- **URLs:** always use `*_url` (not `*_path`) in emails — they need the host from
  `default_url_options` ([delivery.md](delivery.md)).
- **Keep logic out of templates:** compute in the mailer action; templates render.
- **User field name:** the snippets use `email_address` (the column the Rails 8 auth
  generator creates — `../rails-auth/`). On an app with a hand-rolled `email` column,
  substitute your actual attribute.

## Common Pitfalls

- **HTML-only email (no text part)** → worse deliverability + unreadable in
  text-only clients; always add `.text.erb`.
- **`*_path` in an email** → broken relative links; use `*_url` with a host.
- **`deliver_now` in a request** → blocks on the mail server; use `deliver_later`.
- **Missing `default_url_options` host** → `*_url` raises when rendering.
- **Giant inline-attached images** → big emails / spam flags; keep assets small or
  host them.

## Verify

```bash
bin/rails runner "m = UserMailer.welcome(User.first || User.new(email_address:'a@b.com')); p m.subject; p m.parts.map(&:content_type)" 2>/dev/null   # multipart: text/html + text/plain
bin/rails runner "m = UserMailer.welcome(User.first || User.new(email_address: 'a@b.com')); p m.body.encoded.include?('Welcome')" 2>/dev/null   # renders content
# Async path enqueues a job:
bin/rails runner "ActiveJob::Base.queue_adapter=:test; UserMailer.welcome(User.first || User.new(email_address: 'a@b.com')).deliver_later; p ActiveJob::Base.queue_adapter.enqueued_jobs.size" 2>/dev/null   # => 1
```

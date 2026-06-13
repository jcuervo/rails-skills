# Delivery Configuration

How mail leaves the app. The menu pick is **SMTP** (Rails default, provider-portable);
provider gems (Postmark/SendGrid/SES) trade portability for that provider's
features. Credentials always live in **encrypted credentials**, never in code
(`../rails-security/`).

## SMTP (Recommended — works with any provider)

```ruby
# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  address:   "smtp.postmarkapp.com",                 # or SES/SendGrid/Mailgun host
  port:      587,
  user_name: Rails.application.credentials.dig(:smtp, :user),
  password:  Rails.application.credentials.dig(:smtp, :password),
  authentication: :plain,
  enable_starttls_auto: true
}
config.action_mailer.default_url_options = { host: "app.example.com", protocol: "https" }
```

- **Pick when** you want to stay provider-portable — switch hosts by changing
  `address`/credentials, no gem swap. The default and simplest path.
- Most providers (Postmark, SES, SendGrid, Mailgun) offer SMTP endpoints; SMTP is the
  lowest-common-denominator that always works.

## Postmark (API, transactional-focused)

```ruby
# Gemfile: gem "postmark-rails"   (confirm the delivery symbol + settings key in its README)
config.action_mailer.delivery_method = :postmark
config.action_mailer.postmark_settings = { api_token: Rails.application.credentials.dig(:postmark, :api_token) }
```

- **Pick when** you want best-in-class transactional deliverability, message streams,
  and analytics via the API. Accepts provider lock-in to Postmark.

## SendGrid

```ruby
# Either SMTP (smtp.sendgrid.net, user "apikey") — preferred for portability —
# or the API via: gem "sendgrid-actionmailer"  (confirm the delivery symbol + settings key in its README)
config.action_mailer.delivery_method = :sendgrid_actionmailer
config.action_mailer.sendgrid_actionmailer_settings = { api_key: Rails.application.credentials.dig(:sendgrid, :api_key) }
```

- **Pick when** you're standardized on SendGrid. SMTP mode keeps you portable; the
  gem unlocks API-only features.

## Amazon SES (cheapest at scale)

```ruby
# Gemfile: gem "aws-sdk-rails"  (confirm current gem name/namespace for the Mailer integration)
config.action_mailer.delivery_method = :ses               # provided by the AWS SDK Rails integration
```

- **Pick when** you send at high volume and want the lowest cost, and you already
  run on AWS. **Overhead:** SES starts in a **sandbox** (only verified recipients)
  until you request production access; you must verify your domain (SPF/DKIM).
- SES also offers SMTP credentials — using SMTP mode keeps config uniform and
  portable.

## Cross-cutting

- **Credentials, not ENV-in-code:** `bin/rails credentials:edit` →
  `Rails.application.credentials.dig(...)`. Never commit API tokens/passwords
  (`../rails-security/`).
- **`default_url_options[:host]`** must be set per environment or templated URLs
  (`*_url` helpers) raise. Development uses `localhost:3000`.
- **From address & domain auth:** set a consistent `default from:`; deliverability
  (SPF/DKIM/DMARC DNS) is a domain/ops concern — `../rails-deploy/`.
- **Development** typically doesn't really send: use `:test`, `letter_opener`, or
  `:smtp` pointed at Mailcatcher/Mailpit. See
  [previews-and-testing.md](previews-and-testing.md).

## Common Pitfalls

- **Secrets in `smtp_settings` as literals** → leaked credentials; use credentials.
- **Missing `default_url_options`** → `*_url` in templates raises "missing host".
- **SES sandbox surprise** → mail silently only reaches verified addresses until
  production access is granted.
- **No domain auth (SPF/DKIM)** → mail lands in spam regardless of provider;
  configure DNS (`../rails-deploy/`).
- **Sending synchronously** from a request (`deliver_now`) → a slow SMTP server
  blocks the response; use `deliver_later` (`../rails-jobs/`).

## Verify

```bash
bin/rails runner "p ActionMailer::Base.delivery_method"                 # the configured transport
bin/rails runner "p Rails.application.config.action_mailer.default_url_options"   # host set
# Captured delivery in dev (doesn't really send):
bin/rails runner "ActionMailer::Base.delivery_method=:test; UserMailer.welcome(User.first || User.new(email_address: 'a@b.com')).deliver_now; p ActionMailer::Base.deliveries.last&.to" 2>/dev/null
# Provider gem present (if chosen):
bundle list | grep -E "postmark|sendgrid|aws-sdk" || echo "SMTP (no provider gem)"
```

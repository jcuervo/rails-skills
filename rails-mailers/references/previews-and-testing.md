# Mailer Previews & Testing

See an email in a browser without sending it, and assert it renders/delivers in the
suite. Previews are a development tool; the test patterns here hand off to
[`../rails-testing/`](../../rails-testing/SKILL.md) for the actual suite setup.

## Previews (development)

```ruby
# test/mailers/previews/user_mailer_preview.rb  (RSpec: spec/mailers/previews/)
class UserMailerPreview < ActionMailer::Preview
  def welcome
    UserMailer.welcome(User.first || User.new(email_address: "preview@example.com"))
  end
end
```

- Visit **`/rails/mailers`** in development to list previews; click one to see the
  rendered HTML (and text) part. No mail is sent.
- `bin/rails g mailer` scaffolds a preview file. Each public method = one preview.
- Previews use **real data** you assemble — build a representative object (don't rely
  on a specific DB row existing).

## Catching mail locally (instead of sending)

- **`:test` delivery** — mail goes into `ActionMailer::Base.deliveries` (no send);
  used by tests.
- **`letter_opener`** (`gem "letter_opener"`, dev) — pops each email in the browser
  on `deliver`.
- **Mailpit / Mailcatcher** — run a local SMTP catcher; point `:smtp` at it to see
  real-wire formatting. Good for debugging multipart/encoding.

## Testing mailers (hand-off to rails-testing)

```ruby
# Minitest — test/mailers/user_mailer_test.rb
require "test_helper"
class UserMailerTest < ActionMailer::TestCase
  test "welcome renders and addresses correctly" do
    mail = UserMailer.welcome(users(:alice))
    assert_equal "Welcome!", mail.subject
    assert_equal ["alice@example.com"], mail.to
    assert_match "Welcome", mail.body.encoded
  end
end
```
```ruby
# Assert an action ENQUEUES a mailer job (../rails-jobs/ + ../rails-testing/):
# (positional mailers use args:; parameterized `.with(...)` mailers assert differently —
#  confirm the helper signature for your Rails/test framework version)
assert_enqueued_email_with(UserMailer, :welcome, args: [user]) do
  user.send_welcome     # the code under test calls deliver_later
end
```

- **Preview-as-test:** `rails-controller-testing`/built-in helpers can render previews
  in tests so a broken template fails CI. The framework choice (Minitest/RSpec) and
  setup are [`../rails-testing/`](../../rails-testing/references/framework-minitest-rspec.md).
- Assert **delivery vs enqueue** correctly: `deliver_now` → check `deliveries`;
  `deliver_later` → assert the job is enqueued, not that it sent.

## Common Pitfalls

- **Preview depends on `User.first`** existing → blows up on an empty DB; build an
  in-memory object instead.
- **Testing `deliveries` when the code uses `deliver_later`** → nothing in
  `deliveries` (it enqueued a job); assert the enqueue instead.
- **No text-part assertion** → an HTML-only regression slips through; assert both
  parts exist.
- **Previews left pointing at sensitive real data** → use safe sample data.
- **`/rails/mailers` 404 in production** → previews are development-only by design;
  don't expect them in prod.

## Verify

```bash
bin/rails server -d && sleep 3
curl -s -o /dev/null -w "previews=%{http_code}\n" "localhost:3000/rails/mailers"            # 200 (dev)
curl -s -o /dev/null -w "one=%{http_code}\n" "localhost:3000/rails/mailers/user_mailer/welcome"   # 200 renders
kill "$(cat tmp/pids/server.pid)"
# Mailer test passes:
bin/rails test test/mailers 2>/dev/null || bundle exec rspec spec/mailers 2>/dev/null || echo "(add mailer tests via ../rails-testing/)"
```

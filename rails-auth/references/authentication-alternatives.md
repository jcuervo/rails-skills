# Authentication Alternatives — Devise, Rodauth, authentication-zero

When the Rails 8 built-in generator ([rails8-authentication.md](rails8-authentication.md))
isn't the right fit. Pick **one** — never run two auth systems.

## Devise (batteries-included)

```ruby
# Gemfile: gem "devise"   (confirm current generator commands in the Devise README)
bin/rails generate devise:install
bin/rails generate devise User      # adds Devise modules to User + migration
bin/rails db:migrate
```
```ruby
class DashboardController < ApplicationController
  before_action :authenticate_user!   # Devise helper
  def show = @user = current_user
end
```

- **Pick when** you want registration, `confirmable`, `recoverable`, `lockable`,
  `rememberable`, `trackable` out of the box and accept Devise's abstraction/magic.
  Huge ecosystem, fast to ship.
- **Don't pick when** you want to own/read all your auth code, or need features
  Devise doesn't do cleanly (modern MFA/WebAuthn — Rodauth is stronger there).
- Modules are opt-in on the model (`devise :database_authenticatable, :registerable,
  :recoverable, ...`). Views: `rails g devise:views` to customize.
- API/JWT: pair with `devise-jwt` — see [api-tokens.md](api-tokens.md).

## Rodauth (rodauth-rails) — most complete + secure

```ruby
# Gemfile: gem "rodauth-rails"   (confirm current generator commands in the rodauth-rails README)
bin/rails generate rodauth:install      # creates the Rodauth app, account table, migration
bin/rails db:migrate
```

- **Pick when** you need a deep, security-forward feature set: multi-factor (TOTP,
  SMS, WebAuthn), email auth/passwordless, account verification, password
  complexity, audit logging — all first-class and well-tested.
- **Don't pick when** the team isn't ready for Rodauth's different architecture (a
  Roda app mounted in Rails, configuration-DSL driven) — it's the steepest of the
  three.
- Enable features in the Rodauth configuration block; routes are handled by the
  mounted Rodauth app.

## authentication-zero (owned code, more pre-wired)

```ruby
# Gemfile (dev): gem "authentication-zero"
# ⚠ NAME CLASH: authentication-zero's generator is also invoked as `authentication`,
# which now collides with Rails 8's BUILT-IN `bin/rails generate authentication`.
# Running the wrong one silently generates the other system. Confirm the exact,
# current command in the authentication-zero README before running it, and check
# `bin/rails generate --help | grep authentication` to see which generator resolves.
bin/rails db:migrate
```

- **Pick when** you like the built-in's "own your code" philosophy but want more
  generated up front (sign-up, password reset, email verification, sudo/2FA scaffolds,
  rate-limit hooks) than Rails ships.
- **Don't pick when** the Rails 8 built-in already covers your needs (then there's no
  reason to add a gem) or you'd rather have a maintained gem managing it (Devise).
- It's a **generator**, not a runtime dependency — it writes code, then you remove
  the gem. Verify the current generator command in its README, as it overlaps the
  built-in `authentication` generator name.

## Choosing among them

| Need | Pick |
|---|---|
| Own the code, minimal, standard features | Rails 8 built-in (the Recommended default) |
| Registration/confirmable/lockable now, gem-managed | Devise |
| MFA/WebAuthn/passwordless, security-first | Rodauth |
| Own the code but more scaffolded than built-in | authentication-zero |

## Common Pitfalls

- **Two auth systems at once** (built-in + Devise) → conflicting sessions/helpers;
  commit to one.
- **Devise view/locale assumptions** → run `devise:views` and set up flash/i18n.
- **Rodauth treated like a gem you sprinkle in** → it's an architecture; budget time
  to learn its DSL.
- **authentication-zero generator name clash** with the Rails built-in → verify the
  exact command; running the wrong one generates the other system.
- **Skipping `bcrypt`** with any password-based option → `has_secure_password`
  errors.

## Verify

```bash
bundle list | grep -E "devise|rodauth|authentication-zero"    # the chosen gem is present (or absent for built-in)
bin/rails db:migrate
bin/rails server -d && sleep 3
# A protected action blocks the unauthenticated (helper name varies by choice):
curl -s -o /dev/null -w "protected=%{http_code}\n" localhost:3000/dashboard     # 302 (Web) / 401 (API)
# The login route the chosen system mounts is reachable (Devise: /users/sign_in):
curl -s -o /dev/null -w "login=%{http_code}\n" localhost:3000/users/sign_in 2>/dev/null
kill "$(cat tmp/pids/server.pid)"
```

# Rails 8 Built-in Authentication (Recommended)

Rails 8 ships an authentication **generator** that writes real, owned code — no gem
lock-in. It covers **sign-in/sign-out** and **password reset** out of the box;
**sign-up/registration you add yourself** (deliberately). DB-backed sessions
identified by a signed cookie.

## Quick Pattern — generate + wire

```bash
bin/rails generate authentication
bin/rails db:migrate            # creates users + sessions tables
# bcrypt must be active for has_secure_password (the generator uncomments it):
grep -q '^gem "bcrypt"' Gemfile || echo 'add gem "bcrypt"'
```

What the generator creates (confirm exact list on your version with
`bin/rails generate authentication --help` and by reading the generated files):

- `app/models/user.rb` — `has_secure_password`, normalizes email
- `app/models/session.rb` — one row per active session (DB-backed)
- `app/models/current.rb` — `Current.session` / `Current.user` (per-request)
- `app/controllers/concerns/authentication.rb` — the filter + helpers
- `app/controllers/sessions_controller.rb` — login/logout
- `app/controllers/passwords_controller.rb` + `passwords_mailer` — reset flow
- routes: `resource :session`, `resources :passwords`
- login/password views (Web)

## How it protects controllers

The `Authentication` concern is included in `ApplicationController` and adds a
`before_action` requiring a session by default. Opt specific actions out:

```ruby
class HomeController < ApplicationController
  allow_unauthenticated_access only: %i[index]   # public action
end

class DashboardController < ApplicationController
  # inherits require_authentication → redirects to login when signed out
  def show = @user = Current.user
end
```

`Current.user` / `Current.session` give the signed-in user anywhere in the request.
(Exact helper names — `require_authentication`, `allow_unauthenticated_access`,
`start_new_session_for`, `terminate_session` — come from the generated concern; read
it, since details evolve across 8.x point releases.)

## Adding sign-up (registration)

The generator omits registration by design. Add it:

```ruby
# config/routes.rb
resource :registration, only: %i[new create]
```
```ruby
class RegistrationsController < ApplicationController
  allow_unauthenticated_access only: %i[new create]
  def create
    @user = User.new(registration_params)        # registration_params: ../rails-controllers/
    if @user.save
      start_new_session_for @user                 # log them in (helper from the concern)
      redirect_to root_path, notice: "Welcome!"
    else
      render :new, status: :unprocessable_entity   # 422 so the form re-renders (../rails-hotwire/)
    end
  end
  private
  def registration_params = params.expect(user: [:email_address, :password, :password_confirmation])
end
```

## Password reset

Generated end to end: `PasswordsController` + `PasswordsMailer` use
`generates_token_for` (signed, expiring — see `../rails-models/` enums-and-scopes.md).
The email *delivery* config (SMTP/Postmark) is `../rails-mailers/`; this skill owns
the flow, not the transport.

## API-only note

In `--api` mode the generator's cookie/session approach needs adapting to a token
surface — see [api-tokens.md](api-tokens.md). The same `User` +
`has_secure_password` backs both; you swap the session-cookie step for token
issuance.

## When to pick / not pick

- **Pick when** you want to own and understand your auth code, avoid gem lock-in,
  and need standard sign-in + reset. The default for new Rails 8 apps.
- **Don't pick when** you need lots of pre-built features now (confirmable, lockable,
  MFA, WebAuthn) — reach for Devise or Rodauth
  ([authentication-alternatives.md](authentication-alternatives.md)) instead of
  re-building them.

## Common Pitfalls

- **bcrypt not active** → `has_secure_password` raises; ensure `gem "bcrypt"` is in
  the Gemfile and bundled.
- **Expecting sign-up to exist** → it doesn't; add a `RegistrationsController`.
- **Forgetting `allow_unauthenticated_access`** on public actions → login wall on
  pages that should be open.
- **200 on a failed login/registration form** → Turbo won't re-render; use
  `status: :unprocessable_entity` (`../rails-hotwire/` forms-and-responses.md).
- **Not rotating the session on login** → use the generated session-start helper,
  which creates a fresh session row (mitigates fixation; `../rails-security/`).

## Verify

```bash
bin/rails db:migrate
bin/rails runner "u=User.create!(email_address:'a@b.com', password:'secret123', password_confirmation:'secret123'); p u.authenticate('secret123') ? 'pw ok' : 'fail'"
bin/rails server -d && sleep 3
# Protected page redirects to login when signed out:
curl -s -o /dev/null -w "protected=%{http_code}\n" localhost:3000/dashboard      # expect 302
# Login page is reachable:
curl -s -o /dev/null -w "login=%{http_code}\n" localhost:3000/session/new        # expect 200
kill "$(cat tmp/pids/server.pid)"
```

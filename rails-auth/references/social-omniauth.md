# Social / Third-Party Login (OmniAuth)

Social login (Google, GitHub, etc.) is **additive** — it layers onto whatever
authentication you chose ([rails8-authentication.md](rails8-authentication.md) or an
alternative); it does **not** replace password auth. OmniAuth is the standard
mechanism.

## Quick Pattern

```ruby
# Gemfile:
gem "omniauth"
gem "omniauth-google-oauth2"          # one strategy gem per provider
gem "omniauth-rails_csrf_protection"  # REQUIRED — protects the OAuth request phase
```
```ruby
# config/initializers/omniauth.rb
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :google_oauth2, Rails.application.credentials.dig(:google, :client_id),
                           Rails.application.credentials.dig(:google, :client_secret)
end
```
```ruby
# config/routes.rb
get  "/auth/:provider/callback", to: "sessions/omniauth#create"
get  "/auth/failure",           to: "sessions/omniauth#failure"
```
```ruby
class Sessions::OmniauthController < ApplicationController
  allow_unauthenticated_access
  def create
    auth = request.env["omniauth.auth"]
    user = User.find_or_create_from_omniauth(auth)   # find-or-create by provider+uid (../rails-models/)
    start_new_session_for user                       # your authn's session helper
    redirect_to root_path, notice: "Signed in with #{auth.provider}"
  end
end
```

The login *button* posts to `/auth/google_oauth2` via a **POST** form (the CSRF gem
requires POST to start the flow). The button markup is `../rails-hotwire/`.

## Linking to accounts

Store provider identities so a user can have password + multiple socials:

```ruby
# Identity: belongs_to :user, columns: provider, uid, (email)
# find_or_create_from_omniauth: match an existing user by email or create one,
# then attach the (provider, uid) identity. Model design → ../rails-models/.
```

## When to pick / not pick

- **Add OmniAuth when** users expect "Sign in with Google/GitHub," or you want to
  avoid storing passwords for some users.
- **Don't treat it as your whole auth system** — you still need account records,
  session handling, and (usually) a password path for non-social users.

## Deep Dive

- **`omniauth-rails_csrf_protection` is mandatory.** Without it the request phase is
  CSRF-vulnerable; it forces the initiating request to be a POST. (Security context:
  `../rails-security/`.)
- **Credentials, not ENV in code.** Client id/secret live in encrypted credentials
  (`bin/rails credentials:edit`) — `../rails-security/`. Never commit them.
- **Email collisions / account takeover.** Decide deliberately whether a social
  login auto-links to an existing email-matched account (convenient but a takeover
  risk if the provider doesn't verify email). Prefer linking only verified emails.
- **One strategy gem per provider** (`omniauth-github`, `omniauth-google-oauth2`).
- Provider callback URLs must be registered in the provider's console and match
  your `/auth/:provider/callback` route per environment.

## Common Pitfalls

- **Missing `omniauth-rails_csrf_protection`** → CSRF hole + "GET not allowed" errors
  when you try to start the flow with a link instead of a POST.
- **Secrets in `omniauth.rb` as string literals** → leaked credentials; use
  `Rails.application.credentials`.
- **Auto-linking unverified emails** → account takeover; only link verified ones.
- **No password fallback** when you intended social-only but a provider is down →
  decide the recovery path.
- **Callback URL mismatch** between provider console and route → OAuth errors.

## Verify

```bash
bundle list | grep -E "omniauth"                            # omniauth + strategy + csrf-protection present
bin/rails runner "p Rails.application.config.middleware.to_a.map(&:to_s).grep(/OmniAuth/)"   # middleware mounted
bin/rails routes | grep -E "auth/.*callback"                # callback route exists
bin/rails server -d && sleep 3
# Starting the flow redirects to the provider (302 to accounts.google.com):
curl -s -o /dev/null -w "start=%{http_code}\n" -X POST localhost:3000/auth/google_oauth2
kill "$(cat tmp/pids/server.pid)"
```

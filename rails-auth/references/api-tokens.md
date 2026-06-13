# API Token Authentication (mechanism)

How an API authenticates requests with a token. This reference owns the
**mechanism** — minting, storing, verifying, and revoking tokens. The **HTTP
surface** that wraps it (the `Authorization: Bearer` header format, the 401/403
response shape, `WWW-Authenticate`) is `../rails-api/` token-auth-surface.md. Build
the mechanism here; present it there.

## First decision: opaque token vs JWT

| | Opaque (random) token | JWT (signed claims) |
|---|---|---|
| Storage | a row per token (DB) | stateless; nothing stored |
| Revoke | delete the row (instant) | hard — needs a denylist/short TTL |
| Payload | none (look up the row) | self-contained claims |
| Best for | most apps; easy revocation | cross-service, short-lived, stateless |

**Default to opaque DB-backed tokens** unless you specifically need stateless,
cross-service JWTs — revocation simplicity usually wins.

## Quick Pattern — opaque token (Recommended default)

```ruby
# Migration: api_tokens(user_id, token_digest, last_used_at, expires_at)  — store a DIGEST, not the token
class ApiToken < ApplicationRecord
  belongs_to :user
  before_create { self.token_digest = self.class.digest(@raw = SecureRandom.urlsafe_base64(24)) }
  attr_reader :raw
  def self.digest(t) = Digest::SHA256.hexdigest(t)
  def self.authenticate(raw) = find_by(token_digest: digest(raw.to_s))&.then { |t| t.expired? ? nil : t.user }
  def expired? = expires_at&.past?
end
```
```ruby
# Verifying a request — the 401/403 shaping lives in ../rails-api/ token-auth-surface.md
class Api::BaseController < ActionController::API
  before_action :authenticate!
  private
  def authenticate!
    raw = request.authorization&.delete_prefix("Bearer ")
    Current.user = ApiToken.authenticate(raw) or render_unauthorized   # render_unauthorized: ../rails-api/
  end
end
```

- Return the **raw** token to the client **once** at creation; store only its
  SHA-256 digest. A DB leak then can't reveal usable tokens.
- Scope/expire tokens; record `last_used_at` for auditing.

## JWT alternative

```ruby
# Gemfile: gem "jwt"   (or devise-jwt if you're on Devise)
secret = Rails.application.secret_key_base    # the canonical accessor (not credentials.secret_key_base)
token = JWT.encode({ sub: user.id, exp: 15.minutes.from_now.to_i }, secret, "HS256")
payload = JWT.decode(token, secret, true, algorithm: "HS256").first
```

- **Pick JWT when** multiple services must verify the token without a shared DB, or
  you want stateless short-lived access tokens (+ a refresh-token flow).
- **The revocation problem:** a valid JWT can't be un-issued before expiry. Mitigate
  with short TTLs + refresh tokens, or a denylist (which reintroduces state — at
  which point opaque tokens are often simpler).
- **`devise-jwt`** wires JWT issuance/denylist into Devise if that's your authn
  ([authentication-alternatives.md](authentication-alternatives.md)).

## Rails-native building block

`has_secure_password` covers password verification for issuing a token at login;
`generates_token_for` (`../rails-models/` enums-and-scopes.md) is ideal for **signed,
expiring single-purpose tokens** (email confirm, password reset) — not usually for
long-lived API access, where a stored opaque token with revocation is better.

## When to pick / not pick

- **Pick token auth when** the client is non-browser (mobile, another service, CLI)
  or a browser SPA that holds a token. APIs are stateless — no cookie session
  (`../rails-controllers/` sessions-flash-cookies.md).
- **Don't use tokens for** a classic server-rendered Web app — cookie sessions
  ([rails8-authentication.md](rails8-authentication.md)) are simpler and safer there.

## Common Pitfalls

- **Storing raw tokens** in the DB → a dump leaks live credentials; store a digest.
- **No expiry/revocation** → a leaked token is valid forever; expire + allow delete.
- **JWT with no revocation plan** → can't kill a compromised token; short TTL +
  refresh or denylist.
- **Token in the URL/query** → logged everywhere; header only (`../rails-api/`).
- **Re-implementing the HTTP response shape here** → that's `../rails-api/`
  token-auth-surface.md; this file mints/verifies, that one formats 401/403.

## Verify

```bash
bin/rails db:migrate
bin/rails runner "u=User.create!(email_address:'a@b.com', password:'secret123', password_confirmation:'secret123'); t=u.api_tokens.create!; raw=t.raw; p ApiToken.authenticate(raw)&.id == u.id"   # => true round-trip
bin/rails runner "p ApiToken.authenticate('garbage')"     # => nil (rejects bad token)
bin/rails server -d && sleep 3
curl -s -o /dev/null -w "no-token=%{http_code}\n" localhost:3000/v1/posts                       # 401
curl -s -o /dev/null -w "bad-token=%{http_code}\n" localhost:3000/v1/posts -H "Authorization: Bearer nope"   # 401
kill "$(cat tmp/pids/server.pid)"
```

# CSRF & Mass-Assignment (the security lens)

Two classic Rails attack surfaces. The **mechanics** of strong params and CSRF tokens
are owned by `../rails-controllers/` (actions-and-params.md,
sessions-flash-cookies.md); this file is the **security lens** — why they matter and
how to not undermine them. No duplication of the how-to; this is the what-could-go-wrong.

## CSRF protection

```ruby
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception      # on by default for ActionController::Base (Web)
end
```

- **Keep it on for Web.** `protect_from_forgery` verifies a per-session token on
  state-changing requests; `form_with`/`button_to` inject it automatically. Turning it
  off (or `skip_forgery_protection` broadly) opens CSRF.
- **APIs** (`ActionController::API`) don't include CSRF middleware — they're stateless
  and authenticate per-request with a token (`../rails-auth/`, `../rails-api/`), not a
  cookie session. That's correct *for token APIs*; the risk appears when you mix a
  **cookie-session** browser flow into an API and disable CSRF — then you've removed
  the protection a cookie session needs.
- **OmniAuth** request phase needs `omniauth-rails_csrf_protection` (POST + token) —
  `../rails-auth/` social-omniauth.md.
- **Don't blanket-skip** CSRF for "it's just an API endpoint" if that endpoint is
  reachable with the session cookie.

## Mass-assignment

```ruby
# The boundary: ONLY permitted keys reach the model.
params.expect(user: [:name, :email])          # Rails 8 — ../rails-controllers/
# NEVER:
User.new(params[:user])                        # mass-assignment hole: role/admin can be set
```

- **Strong params are a security control**, not a formality: an unpermitted-but-sent
  `admin=true`/`role=...` must be **dropped**. Permit the minimal set; never pass raw
  `params` to `new`/`update`/`create`.
- **Sensitive attributes** (`admin`, `role`, `account_id`, `price`) should never be in
  the permitted list for user-facing forms — set them server-side from authorization
  context (`../rails-auth/`), not from params.
- `params.expect` (Rails 8) additionally rejects wrong-shape input with a 400, a small
  hardening win over `require/permit` — `../rails-controllers/` actions-and-params.md.

## Related injection surfaces (escape, don't trust)

- **SQL injection:** use parameterized queries (`where("x = ?", v)` / `where(x: v)`),
  never string-interpolate user input into SQL. Model/query patterns:
  `../rails-models/`.
- **XSS:** ERB auto-escapes; `html_safe`/`raw`/`<%==` bypass it — only on content you
  generated/sanitized. CSP ([csp-and-headers.md](csp-and-headers.md)) is the
  defense-in-depth backstop.
- **Open redirects:** `redirect_to params[:return_to]` can send users to an attacker
  site — allowlist redirect targets.
- Brakeman ([scanning-and-audit.md](scanning-and-audit.md)) flags these statically.

## Common Pitfalls

- **`skip_forgery_protection` / `protect_from_forgery` off** on a cookie-session Web
  controller → CSRF hole.
- **`User.new(params[:user])`** (or permitting everything) → mass-assignment
  privilege escalation; permit the minimal set.
- **Permitting sensitive attrs** (`:role`, `:admin`) from a user form → set them from
  authorization, not params.
- **`raw`/`html_safe` on user input** → stored XSS; escape or sanitize.
- **Interpolating params into `where("... #{x}")`** → SQL injection; parameterize.

## Verify

```bash
bin/rails server -d && sleep 3
# A state-changing POST without a CSRF token is rejected (Web):
curl -s -o /dev/null -w "no-token=%{http_code}\n" -X POST localhost:3000/posts -d "post[title]=x"   # 422/403 (forgery) for Web
kill "$(cat tmp/pids/server.pid)"
# Mass-assignment guard: unpermitted key is dropped
bin/rails runner "p ActionController::Parameters.new(user:{name:'a', admin:true}).expect(user: [:name]).to_h"   # => {"name"=>"a"} (admin gone)
# Brakeman flags injection/mass-assignment issues:
bin/brakeman -q --no-pager 2>/dev/null | grep -iE "mass assignment|sql injection|cross-site|redirect" || echo "✓ none flagged"
```

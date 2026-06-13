# Sessions, Flash & Cookies (full-stack)

State that rides along with the request: the session, one-shot flash messages,
and cookies. **This reference is for Web apps.** API-only apps
(`config.api_only`) are stateless by design — they carry identity in a token/
header, not a cookie session; skip most of this file and see the API note at the
end.

## Quick Pattern (Web)

```ruby
session[:cart_id] = cart.id          # persists across requests for this user
session[:cart_id]                    # read it back
reset_session                        # on login/logout — prevents session fixation

redirect_to dashboard_path, notice: "Welcome back!"   # flash.notice, shown once
flash[:alert] = "Something went wrong"                 # arbitrary flash key
flash.now[:alert] = "Fix the form"                     # for a render (same request), not a redirect

cookies.signed[:prefs] = { value: "dark", expires: 1.year }   # tamper-evident
cookies.encrypted[:token] = secret                            # confidential
cookies.delete(:prefs)
```

## Deep Dive

- **Session store:** Rails defaults to an encrypted **cookie** store (`config/
  initializers/session_store.rb`). It's signed+encrypted but size-limited
  (browsers cap a cookie at ~4KB) and sent every request — store an id, not objects. For large/server-side
  sessions use the `activerecord-session_store` gem or a cache store
  (`../rails-performance/`).
- **`reset_session` on privilege change:** call it on login and logout to rotate
  the session id and defeat fixation. Auth flows own this — `../rails-auth/`.
- **Flash:** `flash[:notice]`/`flash[:alert]` survive **one redirect**;
  `flash.now[...]` is for the *current* render (no redirect). The convention
  helpers `notice:`/`alert:` on `redirect_to` set `flash[:notice]`/`[:alert]`.
  Rendering the flash in the layout is a view concern (`../rails-hotwire/`).
- **Cookies:** plain `cookies[:k]` is readable/forgeable by the client. Use
  `cookies.signed[...]` for integrity (client can read, not alter) and
  `cookies.encrypted[...]` for confidentiality. Always set `expires`, and let
  Rails manage `httponly`/`secure`/`same_site` defaults (hardening is
  `../rails-security/`).
- **CSRF:** Web forms rely on `protect_from_forgery` (on by default in
  `ActionController::Base`). That's a security concern — owned by
  `../rails-security/`; just don't disable it on Web controllers.

## API-only note

`ActionController::API` excludes the cookie/session/flash middleware by default.
APIs should be **stateless**: authenticate each request with a token
(`Authorization: Bearer …`) rather than a session cookie. Token/session strategy
for APIs is owned by `../rails-auth/` and `../rails-api/`. Don't bolt
cookie-session state onto an API to share auth with a browser unless you
deliberately run a same-site cookie design — decide that in `../rails-auth/`.

## Common Pitfalls

- **Storing objects/large data in the session** → 4KB cookie overflow or bloated
  payload on every request. Store an id; load the record.
- **`flash` vs `flash.now` mix-up** → message shows on the wrong request (or not at
  all): `flash` for redirects, `flash.now` for renders.
- **Trusting plain `cookies[:k]`** → client can forge it; use `signed`/`encrypted`.
- **Forgetting `reset_session` at login/logout** → session fixation risk.
- **Adding session middleware to an API app** to share browser auth → usually the
  wrong design; prefer tokens (`../rails-auth/`).

## Verify

```bash
grep -nE "session_store" config/initializers/session_store.rb 2>/dev/null   # store configured (Web)
bin/rails server -d && sleep 3
# Session round-trips via cookie jar (Web): set then read across requests
curl -s -c /tmp/jar -o /dev/null localhost:3000/ && curl -s -b /tmp/jar -o /dev/null -w "ok=%{http_code}\n" localhost:3000/
kill "$(cat tmp/pids/server.pid)"
# API-only: confirm the session/cookie/flash middleware is actually absent from the stack
bin/rails middleware | grep -iE "session|cookies|flash" || echo "no session stack (stateless / api_only)"
```

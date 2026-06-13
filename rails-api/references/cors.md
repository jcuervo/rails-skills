# CORS

Cross-Origin Resource Sharing lets a browser app on a *different origin* call your
API. Configured via the `rack-cors` gem. **Needed only for browser clients on
another origin** — server-to-server and same-origin calls don't use CORS, and
native mobile apps don't enforce it.

## Quick Pattern

```ruby
# Gemfile — add it, or uncomment if present (--api apps have historically shipped
# it commented; confirm with `grep rack-cors Gemfile`):
gem "rack-cors"
```
```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins Rails.env.production? ? %w[https://app.example.com] : %w[localhost:3000 localhost:5173]
    resource "*",
      headers: :any,
      methods: %i[get post put patch delete options head],
      expose:  %w[Link Current-Page Total-Count],   # so JS can read pagination headers
      credentials: false
  end
end
```

## When to pick / not pick

- **Configure CORS when** a browser SPA, a different subdomain, or a third-party
  web client calls the API from another origin.
- **Skip it when** the API is same-origin (served with its own frontend),
  server-to-server only, or consumed by native mobile (no browser CORS).

## Deep Dive

- **Lock down `origins` in production.** List exact hosts. Never ship
  `origins "*"` together with `credentials: true` — browsers reject it, and it's a
  security hole regardless (`../rails-security/`).
- **`credentials: true`** is required only if the browser sends cookies/auth via
  `withCredentials`. Token-in-header APIs (the usual design —
  [token-auth-surface.md](token-auth-surface.md)) keep `credentials: false` and a
  concrete origin list.
- **`expose`** any custom response header the client JS must read (pagination Link/
  Current-Page headers from [pagination.md](pagination.md)) — by default JS can't
  read non-simple headers.
- **Preflight:** browsers send an `OPTIONS` preflight for non-simple requests;
  `rack-cors` answers it. Include `options` in `methods`.
- This is middleware ordering: `insert_before 0` puts CORS at the top of the stack
  so preflights short-circuit before routing.

## Common Pitfalls

- **`origins "*"` in production** → any site can call the API; lock to known hosts.
- **`*` origins + `credentials: true`** → silently broken (browsers refuse) and
  insecure; use explicit origins with credentials.
- **Forgetting `expose`** → client JS can't read pagination/Link headers even
  though curl sees them.
- **CORS as "auth"** → CORS restricts *browsers*, not curl/servers; it is not an
  authorization mechanism. Real protection is `../rails-auth/` + `../rails-security/`.
- **Editing `cors.rb` without restarting** → initializer changes need a server
  restart to take effect.

## Verify

```bash
bin/rails server -d && sleep 3
# Preflight: an allowed origin gets Access-Control-Allow-Origin back
curl -s -i -X OPTIONS localhost:3000/v1/posts \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: GET" | grep -i "access-control-allow-origin"
# A disallowed origin should NOT be echoed back
curl -s -i -X OPTIONS localhost:3000/v1/posts -H "Origin: http://evil.example" \
  -H "Access-Control-Request-Method: GET" | grep -i "access-control-allow-origin" || echo "✓ evil origin not allowed"
kill "$(cat tmp/pids/server.pid)"
```

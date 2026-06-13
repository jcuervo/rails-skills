# Rate Limiting

Throttle abusive/expensive requests. The menu pick is the **Rails 8 built-in
`rate_limit`** (no gem, cache-backed); **rack-attack** for richer global policies.
Always throttle authentication endpoints (`../rails-auth/`).

## Rails 8 built-in `rate_limit` (Recommended)

```ruby
class SessionsController < ApplicationController
  rate_limit to: 10, within: 3.minutes, only: :create          # 10 POSTs / 3 min per IP → 429
  # Multiple, named limits (confirm the `name:` API for your version):
  rate_limit to: 100, within: 1.minute, name: "burst"
end
```

- Backed by `config.action_controller.cache_store` (falls back to
  `config.cache_store` → Solid Cache by default; `../rails-performance/`
  cache-store.md). **A cache store must be configured** or it can't count.
- **Per-IP by default**; exceeding the limit returns **429 Too Many Requests**.
  Customize the key/response with `by:`/`with:` (verify the exact options against the
  Rails 8.1 API docs — this API is new and evolving).
- **Pick when** you want simple per-controller/endpoint throttling (login, signup,
  password reset, expensive actions) with zero dependencies. The default.
- **Don't pick when** you need global blocklists, allowlists, or fail2ban-style
  escalation across the whole app — use rack-attack.

## rack-attack (richer global policies)

```ruby
# Gemfile: gem "rack-attack"
# config/initializers/rack_attack.rb
class Rack::Attack
  throttle("req/ip", limit: 300, period: 5.minutes) { |req| req.ip }
  throttle("logins/email", limit: 5, period: 60.seconds) do |req|
    req.params["email"]&.downcase if req.path == "/session" && req.post?
  end
  blocklist("block-bad-ip") { |req| BadIps.include?(req.ip) }
  safelist("allow-internal") { |req| req.ip.start_with?("10.") }
end
```

- **Pick when** you need app-wide throttling at the middleware layer, blocklists/
  allowlists, throttles keyed on arbitrary request data (email, token), or
  fail2ban-style progressive bans.
- **Don't pick when** a couple of per-endpoint limits suffice — the built-in is
  simpler and needs no gem.
- Also needs a cache store (it uses `Rails.cache` / its own configured store).
- Mounts as middleware, so it runs before controllers — good for cheaply shedding
  abusive traffic.

## Cross-cutting

- **Throttle auth endpoints first:** login, signup, password reset, token issuance
  are the highest-value targets — `../rails-auth/` flows should be rate-limited here.
- **Behind a proxy/CDN:** `req.ip` must reflect the real client — configure trusted
  proxies so you don't throttle everyone behind the load balancer as one IP
  (`../rails-deploy/`).
- **429 handling:** give clients a clear 429 (+ `Retry-After` where possible); for an
  API, render it in the error envelope (`../rails-api/`).

## Common Pitfalls

- **No cache store configured** → the built-in `rate_limit` can't count; ensure a
  cache store (Solid Cache by default) is set.
- **Real IP not resolved behind a proxy** → either everyone shares one IP (all
  throttled) or the header is spoofable; configure trusted proxies.
- **Rate-limiting only the happy path** → attackers hit signup/reset/token endpoints;
  cover all auth actions.
- **Limit too tight** → legitimate users get 429s; tune `to:`/`within:` to real
  traffic.
- **Relying on rate limiting as the only auth defense** → combine with strong auth
  (`../rails-auth/`), not instead of it.

## Verify

```bash
bin/rails runner "p Rails.application.config.cache_store"          # a store exists (rate_limit needs it)
bin/rails server -d && sleep 3
# Hammer the limited endpoint; expect 429 after the threshold:
for i in $(seq 1 15); do curl -s -o /dev/null -w "%{http_code} " -X POST localhost:3000/session; done; echo
kill "$(cat tmp/pids/server.pid)"
# rack-attack (if used) is in the middleware stack:
bin/rails middleware 2>/dev/null | grep -i "Rack::Attack" || echo "(built-in rate_limit, no middleware)"
```

# The Request Lifecycle (index / orientation)

A map of what happens between a TCP request and a rendered response, and where
each concern is owned. This is an **orientation reference** — read it to place a
problem in the right layer, then jump to the file (or sibling skill) that owns it.
It ends in runnable inspection steps.

## The path of a request

```
Browser/client
  → Web server (Puma; Thruster in front — ../rails-deploy/)
  → Rack middleware stack  ← sessions, cookies, CSRF, params parsing, hosts
  → Router (config/routes.rb)            ← routing.md
  → Controller instance
      → before_action filters            ← concerns-and-filters.md (+ auth: ../rails-auth/)
      → around_action (wraps the action) ← e.g. locale switch: i18n.md
      → action method                    ← actions-and-params.md
          → strong params                ← actions-and-params.md
          → model / service call         ← ../rails-models/
      → render / redirect / head         ← responses.md
          → body: JSON                   ← ../rails-api/
          → body: HTML / Turbo           ← ../rails-hotwire/
      → after_action filters
  → Rack middleware (response side)
  → client
```

## Inspect the actual stack

```bash
bin/rails middleware          # the real Rack stack for THIS app (api_only trims it)
bin/rails routes              # the routing table
```

An API-only app's `bin/rails middleware` is noticeably shorter — no
`ActionDispatch::Flash`, `Cookies`, or session middleware. That difference is the
concrete meaning of `config.api_only`, and why this skill branches on it.

## Who owns what (so you don't duplicate)

| Stage | Owned by |
|---|---|
| Web server, Thruster, deploy | `../rails-deploy/` |
| Security middleware (CSRF, CSP, rate limit, host auth) | `../rails-security/` |
| Routing | [routing.md](routing.md) |
| Filters, concerns, error mapping | [concerns-and-filters.md](concerns-and-filters.md) |
| Authentication / authorization filters | `../rails-auth/` |
| Action + strong params | [actions-and-params.md](actions-and-params.md) |
| Domain logic / persistence | `../rails-models/` |
| Response mechanics (status, format, redirect) | [responses.md](responses.md) |
| Response body — JSON | `../rails-api/` |
| Response body — HTML / Turbo | `../rails-hotwire/` |
| Session/flash/cookies state | [sessions-flash-cookies.md](sessions-flash-cookies.md) |
| Per-request locale selection / i18n | [i18n.md](i18n.md) |

## Custom middleware (rare)

Insert app-specific Rack middleware only for truly cross-cutting request concerns
that aren't a filter:

```ruby
# config/application.rb
config.middleware.use MyRequestTagger
config.middleware.insert_before ActionDispatch::Executor, MyMiddleware
```

Most "every request" logic is better as a `before_action` (controller-scoped,
testable) than middleware. Reach for middleware only when it must run outside the
controller (e.g. before routing).

## Common Pitfalls

- **Putting request logic in middleware that belongs in a filter** → harder to
  test and scope; prefer `before_action`.
- **Assuming flash/session exist in an API app** → they're not in the stack;
  confirm with `bin/rails middleware`.
- **Confusing render vs redirect ownership** → mechanics here, body in the API/
  Hotwire skill.

## Verify

```bash
bin/rails middleware | wc -l                 # stack present; compare Web vs api_only
bin/rails middleware | grep -iE "flash|cookies|session" || echo "no session stack (api_only)"
bin/rails routes | wc -l                      # routes load
```

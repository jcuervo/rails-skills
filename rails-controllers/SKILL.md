---
name: rails-controllers
description: Build the request layer of a Rails 8.1 app — routing, RESTful resources, controllers, actions, strong parameters (Rails 8 params.expect and the require/permit fallback), filters/before_actions, controller concerns, rescue_from error handling, the session/flash/cookies surface, and per-request locale selection / i18n — for both API-only and full-stack apps. Branches on config.api_only (API skips flash/views, Web uses them), detects existing routes/controllers before generating, presents the params and routing idioms as menus with a Recommended default, and verifies routes resolve and actions respond. Apply when adding routes or controllers, shaping strong params, sharing controller behavior, handling request errors, wiring sessions/flash, or setting up internationalization/locale switching.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[resource-or-controller]"
---

# rails-controllers

## Purpose

Owns the **request layer that both app types share**: how a URL becomes a route,
a route reaches a controller action, params are filtered safely, cross-cutting
behavior is shared via filters and concerns, errors become responses, and (for
Web) sessions/flash/cookies carry state. It stops at the boundary of *how the
response body is built* — JSON serialization is `../rails-api/`, HTML/Turbo
rendering is `../rails-hotwire/`. This skill is the plumbing between the route and
that boundary, and it branches throughout on `config.api_only`.

## When to Apply

Use this skill when the task is:

- Adding or restructuring routes (`resources`, nesting, namespaces, constraints)
- Generating or editing a controller and its actions
- Filtering request input with strong parameters
- Sharing behavior across controllers (`before_action`, controller concerns)
- Turning exceptions into HTTP responses (`rescue_from`, status codes)
- Sessions, flash, and cookies (full-stack)
- Internationalization: per-request locale selection + the `I18n` catalog API (`t`/`l`)

Do **not** use this skill when the task is:

- Modeling data, validations, or queries behind the controller → read `../rails-models/SKILL.md`
- **Building the JSON response body** — serializers, versioning, pagination, error envelopes → read `../rails-api/SKILL.md`
- **Building the HTML response** — views, forms, Turbo Frames/Streams, Stimulus → read `../rails-hotwire/SKILL.md`
- Authentication / authorization (who may call the action) → read `../rails-auth/SKILL.md`
- File uploads handled in a controller → read `../rails-storage/SKILL.md`
- Controller/request/integration tests → read `../rails-testing/SKILL.md`
- Rate limiting, CSRF, CSP, security headers → read `../rails-security/SKILL.md`
- Request-level caching / performance → read `../rails-performance/SKILL.md`

## Detect Before You Generate

```bash
grep -nE "api_only" config/application.rb                 # API-only vs full-stack — branches everything
bin/rails routes | head -30                               # existing route table
ls app/controllers app/controllers/concerns 2>/dev/null   # existing controllers + shared concerns
sed -n 's/.*rails (\(.*\))/Rails \1/p' Gemfile.lock | head -1   # version → params.expect availability
ls config/locales/ 2>/dev/null; grep -nE "default_locale|available_locales|i18n.fallbacks" config/application.rb config/environments/*.rb 2>/dev/null  # existing i18n setup
```

- `config.api_only == true` → **API mode**: no flash/cookies-based UI state, no
  view rendering; responses are JSON/`head`. Skip the sessions/flash reference.
- `config.api_only` absent/false → **Web mode**: flash, sessions, cookies, and
  HTML responses are in play.
- If `STACK.md` exists, read it for the app type and test framework.
- Read an existing controller before editing; extend its style rather than
  imposing a new one.

## Menu

Two idiom-level decisions, each asked via `AskUserQuestion` with the Rails 8
default marked **Recommended** — unless the codebase already uses one
consistently, in which case match it silently.

### Strong-parameters idiom
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **`params.expect`** *(Recommended)* | Rails 8 idiom; stricter — rejects tampered/wrong-shape params with a 400, not a 500. | [actions-and-params.md](references/actions-and-params.md) |
| `params.require(...).permit(...)` | Universally compatible (all supported versions); the right call on Rails < 8 or to match an existing codebase. | [actions-and-params.md](references/actions-and-params.md) |

### Routing idiom
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Resourceful (`resources`/`resource`)** *(Recommended)* | Conventional REST routes + named helpers; least surprise, best tooling. | [routing.md](references/routing.md) |
| Explicit/custom (`get`/`post ... => "ctrl#action"`) | One-off non-CRUD endpoints; use sparingly alongside resourceful routes, not instead of them. | [routing.md](references/routing.md) |

`config.api_only` is **not** a menu — it's detected and branches behavior
throughout (it was chosen in `../rails-scaffold/`).

## Decision Flow

- **Routing:** model endpoints as resources first; add `member`/`collection`
  routes for the rare non-CRUD action. Reach for a fully custom route only for
  genuinely non-resourceful endpoints (webhooks, callbacks). Nest at most one
  level deep — beyond that, use shallow nesting.
- **Params:** `params.expect` on Rails 8 for the safety upgrade; fall back to
  `require/permit` for older apps or to stay consistent with surrounding code.
- **Sharing behavior:** a `before_action` for per-request setup/guards; a
  **controller concern** when several controllers share a cohesive cluster of
  behavior. Don't extract a concern for a single controller.
- **Errors:** `rescue_from` in a base controller to map domain exceptions to
  status codes once, rather than rescuing in every action. API vs Web differ in
  the *body* (JSON envelope vs error page) — that body is `../rails-api/` /
  `../rails-hotwire/`; the *status mapping* is here.
- **Where logic goes:** keep actions thin — params in, model/service call,
  response out. Business logic belongs in models (`../rails-models/`) or service
  objects, not the controller.

## Problem → Reference

| Task | Read |
|---|---|
| Routes: `resources`, nesting, namespaces, constraints, `member`/`collection` | [references/routing.md](references/routing.md) |
| Actions, RESTful CRUD, strong params (`expect` / `require`/`permit`) | [references/actions-and-params.md](references/actions-and-params.md) |
| Building responses: formats, status codes, redirects, `respond_to` (API vs Web) | [references/responses.md](references/responses.md) |
| `before_action`/filters, controller concerns, `rescue_from` error handling | [references/concerns-and-filters.md](references/concerns-and-filters.md) |
| Sessions, flash, cookies (full-stack) | [references/sessions-flash-cookies.md](references/sessions-flash-cookies.md) |
| Internationalization: per-request locale, translation/localization (`t`/`l`), fallbacks | [references/i18n.md](references/i18n.md) |
| The full request lifecycle: middleware → routing → action → response | [references/request-lifecycle.md](references/request-lifecycle.md) |

## Verify

A route/controller change is unfinished until the route resolves and the action
actually responds:

```bash
bin/rails routes -g <resource>                         # the route exists + maps to the action
bin/rails runner "p Rails.application.routes.recognize_path('/<path>')"  # path resolves to controller#action
# Hit it (boot daemonized; API returns JSON, Web returns HTML):
bin/rails server -d && sleep 3
curl -fsS -o /dev/null -w "%{http_code}\n" localhost:3000/<path>        # expected 2xx/3xx
kill "$(cat tmp/pids/server.pid)"
bin/rails test test/controllers 2>/dev/null || bundle exec rspec spec/requests 2>/dev/null  # request specs green
```

Then route on to the response layer (`../rails-api/` or `../rails-hotwire/`),
`../rails-auth/` for guarding actions, and `../rails-testing/` for request specs.

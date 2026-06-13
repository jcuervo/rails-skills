# App Type

The first and most consequential pick: it sets `config.api_only`, gates the
frontend questions, and changes which sibling skills are in play. Ask via
`AskUserQuestion` before anything else.

## Menu

| Option | `rails new` | One-line trade-off |
|---|---|---|
| **Full-stack (Hotwire)** *(Recommended)* | *(no flag)* | Server-rendered HTML + Turbo/Stimulus; richest Rails experience, least JS. |
| API-only | `--api` | Lean JSON backend; no views/assets/Hotwire; pair with any frontend. |
| API + separate frontend | `--api` *(this repo)* | Same as API-only, but you also stand up a distinct SPA/native app elsewhere. |

## Decision Flow

- **Will Rails render the HTML the user sees?** Yes → full-stack. No → API-only.
- **Is the client a mobile app, a third-party consumer, or a JS SPA you build
  separately?** → API-only (and treat the frontend as its own project).
- **Unsure / want fastest path to a working UI** → full-stack. Hotwire gives you
  interactive UI without a separate frontend build, and you can still expose JSON
  later via `../rails-api/SKILL.md`.
- A full-stack app can serve JSON too (just add API controllers); an `--api` app
  cannot easily grow server-rendered views back without undoing `--api`. When in
  doubt and the UI story is unclear, **full-stack is the less reversible-costly
  default.**

## What each pick changes downstream

| | Full-stack | API-only (`--api`) |
|---|---|---|
| Middleware stack | Full (cookies, flash, sessions) | Trimmed (no cookies/flash by default) |
| `ApplicationController` | `< ActionController::Base` | `< ActionController::API` |
| Views / asset pipeline | Generated | Skipped |
| Hotwire (Turbo/Stimulus) | Included | Skipped |
| Step 3 (CSS/JS) interview | **Asked** | **Skipped** |
| Primary sibling for request layer | `../rails-hotwire/` + `../rails-controllers/` | `../rails-api/` + `../rails-controllers/` |

## API + separate frontend

When the user wants a JSON API **and** a separate client:

- This Rails repo is scaffolded with `--api`.
- The frontend is a **separate project** — do not try to colocate a full SPA build
  inside the API app. For React Native, point at `rn-scaffold-expo`. For a web
  SPA, scaffold it in its own directory.
- Flag the cross-cutting concerns early: **CORS** (owned by `../rails-api/`),
  **token/JWT auth** (owned by `../rails-auth/`), and a stable **error envelope +
  versioning** (owned by `../rails-api/`). Note them in `STACK.md` so the API
  surface is designed for an external consumer from day one.

## Pitfalls

- Picking `--api` then later needing server-rendered pages (admin, marketing,
  email-confirmation landing) means re-adding the asset pipeline and middleware by
  hand. If even a small amount of server HTML is likely, prefer full-stack and
  expose JSON selectively.
- `--api` does **not** mean "no HTML ever" — but it removes the scaffolding that
  makes HTML cheap. Decide based on the *primary* surface.

## Verify

```bash
grep -n "config.api_only" config/application.rb   # true only for --api apps
grep -n "ActionController::API\|ActionController::Base" app/controllers/application_controller.rb
```

A full-stack app shows `api_only` absent/false and `ActionController::Base`; an
API app shows `config.api_only = true` and `ActionController::API`.

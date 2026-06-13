---
name: rails-scaffold
description: Stand up a new Ruby on Rails 8.1 app (API-only, full-stack Hotwire, or API+frontend) through a guided, menu-driven interview. The agent asks about app type, database, asset pipeline, CSS/JS, test framework, auth, and the Rails 8 omakase backends (Solid Queue/Cache/Cable, Kamal, Thruster) — each as a vetted menu with a Recommended default — assembles the right `rails new` flags, runs the generator, verifies the app boots, then routes to sibling rails-* skills to wire each concern. Also detects an existing app and routes without re-scaffolding. Apply when creating a new Rails project, choosing `rails new` flags, or deciding how to bootstrap a Rails codebase.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[app-name]"
---

# rails-scaffold

## Purpose

Source of truth for standing up a new Rails 8.1 application. This is the
**greenfield entry point** of the suite: it runs an end-to-end guided interview,
maps each answer to a `rails new` flag, runs the generator, confirms the app
boots, and then hands off to the sibling `rails-*` skills that own the deeper
wiring of each concern. It is an **orchestrator** — it owns the *generate + route*
step, not the deep per-concern implementation.

Its brownfield mirror is `../rails-upgrade/SKILL.md` (moving an *existing* older
app to the latest Ruby/Rails). This skill handles new apps and light routing for
already-modern ones.

## When to Apply

Use this skill when the task is:

- Creating a new Rails app from scratch (`rails new`)
- Deciding which `rails new` flags to use (database, `--api`, `--css`, `--javascript`, `--skip-*`)
- Bootstrapping the initial structure and choosing the omakase backends
- Routing an existing app to the right concern skill for a follow-up task

Do **not** use this skill when the task is:

- Upgrading an existing app's Rails/Ruby version → read `../rails-upgrade/SKILL.md`
- Modeling data, migrations, or DB choice on an existing app → read `../rails-models/SKILL.md`
- Routing/controllers work → read `../rails-controllers/SKILL.md`
- API serialization/versioning → read `../rails-api/SKILL.md`
- Frontend / Turbo / Stimulus / real-time → read `../rails-hotwire/SKILL.md`
- Authentication or authorization → read `../rails-auth/SKILL.md`
- Background jobs, mailers, file storage → read `../rails-jobs/`, `../rails-mailers/`, `../rails-storage/`
- Tests → read `../rails-testing/SKILL.md`
- A retrospective security/quality audit of an existing app → use the `rails-audit` skill

## Detect Before You Ask

Before presenting **any** menu, inspect the working directory:

```bash
ls Gemfile config/application.rb bin/rails 2>/dev/null   # is this already a Rails app?
ruby -v && rails -v 2>/dev/null                          # toolchain present?
```

- **An existing Rails app is present** (`config/application.rb` exists) → do **not** scaffold. Read [references/existing-app.md](references/existing-app.md), detect its picks, and route to the right sibling skill.
- **Empty dir / no Rails app** → proceed with the new-app interview below.
- **Confirm the installed generator's real flags** before assembling a command — defaults drift between point releases:

```bash
rails new --help        # authoritative flag list + defaults for the INSTALLED Rails
```

Never hardcode a flag the installed generator doesn't list. The menus below
reflect Rails **8.1**; `rails new --help` is the tiebreaker.

## The Interview (orchestration spine)

Run the concerns **in this order** — earlier answers constrain later ones (app
type gates whether CSS/JS/Hotwire are even asked). Full flow, command assembly,
and the post-generate routing map live in
[references/new-app-interview.md](references/new-app-interview.md).

| # | Concern | Menu lives in | Maps to `rails new` flag |
|---|---|---|---|
| 1 | **App type** | [app-type.md](references/app-type.md) | `--api` / full-stack / API+SPA |
| 2 | **Database** | [database-selection.md](references/database-selection.md) | `-d/--database` |
| 3 | **Asset pipeline + CSS + JS** *(full-stack only)* | [frontend-stack.md](references/frontend-stack.md) | `-a`, `-c/--css`, `-j/--javascript`, `--skip-hotwire` |
| 4 | **Omakase backends** | [omakase-defaults.md](references/omakase-defaults.md) | `--skip-solid`, `--skip-kamal`, `--skip-thruster`, `--skip-ci`, `--skip-rubocop`, `--skip-brakeman`, `--devcontainer` |
| 5 | **Test framework** | defer to `../rails-testing/SKILL.md` | `-T/--skip-test` (skip Minitest for RSpec) |
| 6 | **Auth** | defer to `../rails-auth/SKILL.md` | *(post-generate, no flag)* |

Concerns 5 and 6 are **decided here but wired by siblings**: this skill records
the choice and sets the relevant `rails new` flag (e.g. `-T` when the user picks
RSpec so Minitest scaffolding isn't generated), then routes.

## Menu Summary

Full menus with trade-offs and pitfalls live in the reference files. Recommended
= the Rails 8 omakase default; the agent still asks unless the answer is forced
(e.g. `--api` removes the CSS/JS questions).

| Concern | Options (Recommended in **bold**) |
|---|---|
| App type | **Full-stack (Hotwire)**, API-only, API + separate frontend |
| Database | **SQLite**, PostgreSQL, MySQL, MariaDB, Trilogy |
| Asset pipeline | **Propshaft**, Sprockets, none |
| CSS | **Tailwind**, Bootstrap, Bulma, PostCSS, Sass, plain CSS |
| JavaScript | **Import maps**, esbuild, Bun, Vite *(via gem)*, Rollup, Webpack |
| Job/Cache/Cable | **Solid Queue + Solid Cache + Solid Cable**, skip Solid (add Sidekiq/Redis later) |
| Deploy tooling | **Kamal 2 + Thruster**, skip (PaaS/other) |
| Test framework | **Minitest**, RSpec |

"Recommended" means: when the user has no preference and asks the agent to
choose, this is the pick. The agent surfaces the menu — it does not silently
default — **unless** the dir already constrains it (existing app, or a flag the
chosen app type forecloses).

## Verification After Scaffold

After `rails new` finishes, the scaffold is **not done** until all of these pass:

1. `bin/rails about` prints the expected Ruby (4.0.x) and Rails (8.1.x) versions
2. `bin/rails db:prepare` succeeds against the chosen database
3. `bin/rails server` boots and `GET /up` returns **200** (the health check)
4. Full-stack: the welcome page renders; API-only: a smoke JSON endpoint returns
5. The chosen test framework runs green (`bin/rails test` or `bundle exec rspec`)
6. `bin/rubocop` and `bin/brakeman` run clean (unless skipped)

If any check fails, fix it before handing off. (The `STACK.md` record is a
hand-off artifact, written at the end — see Hand-off below — not a runnable
pass/fail check.) See
[references/new-app-interview.md](references/new-app-interview.md) for the exact
commands per app type.

## Problem → Reference

| Task | Read |
|---|---|
| Run the full new-app interview (orchestrates everything) | [references/new-app-interview.md](references/new-app-interview.md) |
| Choose API-only vs full-stack vs API+frontend | [references/app-type.md](references/app-type.md) |
| Pick the database (the `-d` flag) | [references/database-selection.md](references/database-selection.md) |
| Pick asset pipeline + CSS + JS bundler | [references/frontend-stack.md](references/frontend-stack.md) |
| Keep/skip Solid trio, Kamal, Thruster, CI, RuboCop, Brakeman | [references/omakase-defaults.md](references/omakase-defaults.md) |
| Detect an existing app and route instead of scaffolding | [references/existing-app.md](references/existing-app.md) |

## Hand-off

After a successful scaffold, the typical routing is: model the domain
(`../rails-models/`) → request layer (`../rails-controllers/` + `../rails-api/`
or `../rails-hotwire/`) → auth (`../rails-auth/`) → tests
(`../rails-testing/`) → jobs/mailers/storage → security/performance → deploy
(`../rails-deploy/`). Record the picks in `STACK.md` so siblings detect them
instead of re-asking.

---
name: rails-upgrade
description: Upgrade an existing (brownfield) Rails app's framework version toward the latest Ruby 4.0.x / Rails 8.1.x. Detects the current Ruby/Rails, plans an incremental one-minor-version-at-a-time path, and drives the per-step loop — bump the Gemfile, bundle update, run bin/rails app:update, reconcile config/initializers/new_framework_defaults_X_Y.rb, flip config.load_defaults, triage deprecations, and keep the test suite green between each step. Then guides the omakase stack modernizations (classic→Zeitwerk, Sprockets→Propshaft, Webpacker→importmaps/jsbundling, Sidekiq+Redis→Solid Queue/Cache/Cable, secrets.yml→encrypted credentials) by routing to the sibling rails-* skills that own each destination state. This skill owns the JOURNEY; siblings own the destination. Apply when upgrading Rails/Ruby versions, running app:update, reconciling framework defaults, migrating the autoloader, or modernizing the omakase stack of an older app.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[target-version]"
---

# rails-upgrade

## Purpose

The **brownfield entry point** of the suite — the mirror of `../rails-scaffold/`
(which owns *new* apps). It drives **framework version upgrades** of an existing app
toward the latest Ruby/Rails: detect the current versions, plan an incremental path,
and run the per-step upgrade loop until the app is on Rails 8.1 / Ruby 4.0. It then
guides the **omakase stack modernizations** by **routing to the sibling skills that
own each destination state** — this skill owns the *journey* (how to get there
safely), siblings own the *destination* (what modern looks like). Built last on
purpose: it orchestrates swaps via menus the other skills define.

## When to Apply

Use this skill when the task is:

- Upgrading an app's Rails and/or Ruby version (e.g. 6.1 → 7.0 → 7.1 → 7.2 → 8.0 → 8.1)
- Running `bin/rails app:update` and reconciling the changes
- Adopting `new_framework_defaults_*.rb` / flipping `config.load_defaults`
- Migrating the autoloader (classic → Zeitwerk)
- Modernizing the omakase stack of an older app (assets, JS, jobs/cache/cable, secrets)
- Triaging deprecation warnings ahead of a version bump

Do **not** use this skill when the task is:

- Standing up a **new** app → read `../rails-scaffold/SKILL.md` (greenfield entry point)
- Adding a concern to an **already-modern** app (it's already on latest) → go straight to the relevant sibling (`../rails-models/`, `../rails-hotwire/`, etc.)
- A specific destination-state implementation — *how Propshaft/Solid Queue/etc. work* → read the owning sibling (this skill routes you there)
- Upgrading a single gem unrelated to the framework version → ordinary `bundle update <gem>`
- A retrospective audit of the old app → use the upstream `rails-audit` skill

## Detect Before You Generate

```bash
ruby -v; cat .ruby-version 2>/dev/null                                   # current Ruby
grep -E "^    rails \(" Gemfile.lock | head -1                           # current Rails (actual)
grep -n "load_defaults" config/application.rb                            # which defaults version is active
grep -rn "config.autoloader\|:classic\|:zeitwerk" config/ 2>/dev/null    # autoloader mode
ls config/initializers/new_framework_defaults_*.rb 2>/dev/null           # a half-finished upgrade in progress?
grep -E "^    (webpacker|sprockets|sidekiq|redis)" Gemfile.lock          # legacy omakase to modernize
ls config/secrets.yml config/credentials.yml.enc 2>/dev/null             # secrets style
bundle exec rspec --version 2>/dev/null || bin/rails test --help 2>/dev/null   # the safety net exists?
```

- **A real test suite is the prerequisite.** Upgrading without tests is flying blind
  — if coverage is thin, route to `../rails-testing/` to build a safety net *before*
  bumping versions.
- A leftover `new_framework_defaults_*.rb` or an un-flipped `load_defaults` means a
  **prior upgrade is mid-flight** — finish that step before starting the next.
- Detect the legacy stack (Webpacker/Sprockets/Sidekiq/secrets.yml) to plan the
  modernizations ([stack-modernization.md](references/stack-modernization.md)).

## Menu

One genuine menu — the **approach**. The stack swaps are not menus here; each routes
to the sibling that owns its menu (see Stack Modernization below).

### Upgrade approach
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Incremental (one minor at a time)** *(Recommended)* | Bump one minor version, get green, deploy, repeat. The Rails-recommended path; smallest blast radius, easiest to debug. | [upgrade-path.md](references/upgrade-path.md) |
| Dual-boot (next_rails / BootBoot) | Run old + new Rails side by side in CI so you fix the new version without blocking the old. Best for large/long upgrades. | [deprecations-and-dualboot.md](references/deprecations-and-dualboot.md) |
| Direct jump | Skip straight to the target in one go. Only for **small apps with strong tests**; otherwise the debugging surface explodes. | [upgrade-path.md](references/upgrade-path.md) |

Present this via `AskUserQuestion` — default to **Incremental** unless the app is
large/long-lived (offer **dual-boot**) or tiny with a strong suite (**direct jump**
is then viable). (The version-upgrade journey is **app-type-agnostic** — it applies
identically to API-only and full-stack apps; the app-type-specific *stack swaps*
route to the owning sibling, which branches on `config.api_only` itself.)

## Decision Flow

- **Approach:** **incremental, one minor at a time** is the default and almost always
  right — each step is small enough to debug, and you keep a deployable app
  throughout. **Dual-boot** when the upgrade is large/long and you need to keep
  shipping on the old version while preparing the new. **Direct jump** only for a
  small app with a strong suite.
- **The per-step loop** (one per minor version) — see
  [upgrade-path.md](references/upgrade-path.md):
  1. Bump Ruby/Rails in the `Gemfile`; `bundle update rails` (+ blocking gems).
  2. `bin/rails app:update` — reconcile config/files interactively.
  3. Get the suite **green** *without* flipping defaults yet (`../rails-testing/`).
  4. Adopt `new_framework_defaults_X_Y.rb` **incrementally**, then flip
     `config.load_defaults` ([framework-defaults.md](references/framework-defaults.md)).
  5. Triage deprecations ([deprecations-and-dualboot.md](references/deprecations-and-dualboot.md)).
  6. Commit + deploy that version before the next bump.
- **Autoloader:** classic → **Zeitwerk** is required to reach Rails 7+
  ([zeitwerk.md](references/zeitwerk.md)) — and it's the #1 "works in dev, breaks in
  prod (eager load)" trap.
- **Modernize after you're current.** Get the version up first; then adopt the
  omakase stack by routing to siblings — don't mix a version bump and a stack swap in
  the same step.

## Stack Modernization → routes to siblings

This skill owns the *sequencing*; each sibling owns the *destination menu*. Adopt
these **after** the version is current, one at a time:

| Legacy → Modern | Routed to (owns the destination) | This skill covers |
|---|---|---|
| classic autoloader → **Zeitwerk** | (here) [zeitwerk.md](references/zeitwerk.md) | the migration itself |
| Sprockets → **Propshaft** | `../rails-scaffold/` (frontend-stack.md), `../rails-hotwire/` (assets-and-css.md) | when/how to switch in the journey |
| Webpacker → **import maps / jsbundling / Vite** | `../rails-hotwire/` (assets-and-css.md) | retiring Webpacker safely |
| Sidekiq+Redis → **Solid Queue** | `../rails-jobs/` | swapping the Active Job backend |
| Redis cache → **Solid Cache** | `../rails-performance/` (cache-store.md) | swapping the cache store |
| Redis Cable → **Solid Cable** | `../rails-hotwire/` (real-time.md) | swapping the cable adapter |
| `secrets.yml` → **encrypted credentials** | `../rails-security/` (secrets-and-credentials.md) | migrating the secrets store |
| add **Kamal/Thruster/CI** | `../rails-deploy/` | adopting modern deploy |
| Minitest⇄RSpec, add system tests | `../rails-testing/` | shoring up the safety net first |

## Problem → Reference

| Task | Read |
|---|---|
| Plan the path + run the per-version upgrade loop | [references/upgrade-path.md](references/upgrade-path.md) |
| Reconcile `new_framework_defaults_*.rb` + flip `load_defaults` | [references/framework-defaults.md](references/framework-defaults.md) |
| Migrate the autoloader classic → Zeitwerk | [references/zeitwerk.md](references/zeitwerk.md) |
| Sequence the omakase stack swaps (routes to siblings) | [references/stack-modernization.md](references/stack-modernization.md) |
| Triage deprecations + dual-boot (next_rails) for big upgrades | [references/deprecations-and-dualboot.md](references/deprecations-and-dualboot.md) |

## Verify

Each step is unfinished until the app boots, the suite is green, and (in prod-like
eager load) nothing breaks:

```bash
grep -E "^    rails \(" Gemfile.lock | head -1                # landed on the intended version
bin/rails about                                               # Ruby + Rails as expected
bin/rails zeitwerk:check                                      # autoloading is correct (eager-load safe)
RAILS_ENV=production bin/rails zeitwerk:check 2>/dev/null     # the prod eager-load trap
bin/rails test 2>/dev/null || bundle exec rspec 2>/dev/null   # suite green at this step
bin/rails runner "p Rails.application.config.load_defaults rescue p Rails.version"   # defaults level
```

When the app is current (Rails 8.1 / Ruby 4.0) and green, the upgrade journey is
done — hand off to the siblings to adopt each modern pick, and to `../rails-deploy/`
to ship it. Record the landed versions + remaining modernizations in `STACK.md`.

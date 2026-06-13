# Existing App — Detect & Route

When the working directory is **already a Rails app**, do not scaffold. The job
shifts from *generate* to *detect what's installed and route to the right sibling
skill*. This file is the detection + routing playbook.

## Confirm it's a Rails app

```bash
test -f config/application.rb && test -f bin/rails && echo "Rails app" || echo "not a Rails app"
```

If both exist, it's a Rails app — never run `rails new` over it.

## Read versions first

```bash
ruby -v                                   # actual Ruby
grep -E "^ruby " Gemfile .ruby-version 2>/dev/null
grep -E "gem ['\"]rails['\"]" Gemfile
bin/rails about                           # Rails + Ruby + middleware in one shot
```

- **Rails < 8.1 (or Ruby < 4.0) and the user wants to modernize** → this is an
  **upgrade**, not a scaffold. Route to `../rails-upgrade/SKILL.md`.
- **Rails 8.1.x / modern** → detect picks (below) and route to the concern skill
  for the actual task.

## Detect existing picks (so siblings don't re-ask)

One sweep of `Gemfile`/`config` tells you what's already chosen. Read `STACK.md`
first if it exists — it's the fastest source of truth.

```bash
# App type
grep -n "config.api_only" config/application.rb

# Database
grep -nE "adapter:" config/database.yml | head -1

# Asset pipeline / CSS / JS (full-stack)
bundle list | grep -E "propshaft|sprockets"
bundle list | grep -E "tailwindcss-rails|bootstrap|bulma|dartsass-rails|cssbundling-rails"
test -f config/importmap.rb && echo "importmap" || (test -f package.json && echo "bundler")

# Hotwire
bundle list | grep -E "turbo-rails|stimulus-rails"

# Jobs / cache / cable backend
bundle list | grep -E "solid_queue|solid_cache|solid_cable|sidekiq|good_job"

# Auth
bundle list | grep -E "devise|rodauth|authentication-zero|bcrypt"
grep -rln "has_secure_password" app/models 2>/dev/null

# Authorization
bundle list | grep -E "pundit|action_policy|cancancan"

# Test framework
bundle list | grep -E "rspec-rails|minitest" ; ls -d spec test 2>/dev/null

# Deploy
ls config/deploy.yml 2>/dev/null && echo "Kamal"
```

For any concern where a pick is detected, **use it silently** — do not present the
menu. The menu is only for *unmet* concerns.

## Route to the right sibling

Match the user's actual task to the owning skill:

| The task is about… | Route to |
|---|---|
| Upgrading Rails/Ruby version | `../rails-upgrade/SKILL.md` |
| Models, associations, migrations, DB schema, search | `../rails-models/SKILL.md` |
| Routes, controllers, strong params, sessions | `../rails-controllers/SKILL.md` |
| JSON serialization, versioning, pagination, API docs, CORS | `../rails-api/SKILL.md` |
| Turbo, Stimulus, views, components, real-time/Action Cable | `../rails-hotwire/SKILL.md` |
| Login, signup, sessions, roles, permissions | `../rails-auth/SKILL.md` |
| Background jobs, scheduling, recurring tasks | `../rails-jobs/SKILL.md` |
| Transactional email, mailer templates, previews | `../rails-mailers/SKILL.md` |
| File uploads, Active Storage, variants | `../rails-storage/SKILL.md` |
| Tests, factories, coverage, system tests | `../rails-testing/SKILL.md` |
| Proactive hardening (CSP, rate limit, secrets, Brakeman) | `../rails-security/SKILL.md` |
| Caching, N+1, performance tuning | `../rails-performance/SKILL.md` |
| Deploy, Kamal, Docker, CI/CD, observability | `../rails-deploy/SKILL.md` |
| Retrospective audit → PDF report | `rails-audit` skill (upstream) |

## When the task is "add a concern that isn't installed yet"

This is still in-scope for routing: the sibling skill owns the menu for its
concern. For example, "add background jobs to this app" → route to
`../rails-jobs/`, which presents the Solid Queue / Sidekiq / GoodJob menu and
respects whatever the existing app already implies (e.g. Solid already present →
use it). `rails-scaffold` only owns the **`rails new`-time** menus; everything
post-generate belongs to the sibling that owns that concern.

## Pitfalls

- Running `rails new` inside an existing app (or a subdir of one) — always run the
  `config/application.rb` check first.
- Re-asking a menu when the pick is already installed — defeats the "ask-once"
  rule. Detect before asking.
- Treating an old-Rails app as a scaffold target — it's an **upgrade**; route to
  `../rails-upgrade/`, don't generate a fresh app beside it.

## Verify (routing was correct)

After routing, the sibling skill's own Verify section is the gate. `rails-scaffold`
has done its job once it has (a) confirmed the app is real, (b) recorded detected
picks (ideally into/already in `STACK.md`), and (c) handed off to the correct
sibling for the actual task.

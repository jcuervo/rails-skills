# Rails Skills — Project Brief

> **Status:** Draft for discussion. Nothing is built yet. This brief defines the
> scope, shape, and conventions of the skill suite so we can align before writing
> any `SKILL.md` files.

## 1. Goal

Build a suite of composable **Agent Skills** that help Claude scaffold, build,
test, harden, and ship a **Ruby on Rails** application — whether it's a JSON
**API** backend or a full-stack **Hotwire web** app — using the latest Ruby and
Rails, while staying **unopinionated** about library choices.

The suite mirrors the house style of the existing `rn-*` skills (e.g.
`rn-scaffold-expo`, `rn-testing`, `rn-release-eas`): many focused skills, each a
self-contained `SKILL.md` with a `references/` directory, presenting **vetted
menus** of alternatives rather than dictating one stack.

## 2. Decisions locked in

| Dimension | Decision | Implication |
|---|---|---|
| **Application type** | **Both** API-only and full-stack Web | Shared core skills + API-specific + Web-specific skills |
| **Opinionation** | **Menu-driven** | Each concern presents 2–5 vetted options via `AskUserQuestion`; agent implements the pick. A **Recommended** default is marked but the agent asks unless the project already has a pick installed. |
| **Granularity** | **Many focused skills** | One skill per concern, composable and precisely invocable |
| **Scope** | **Full lifecycle** | Core app dev + deploy/ops/observability/performance/real-time |

## 3. Target versions

| Component | Target | Notes |
|---|---|---|
| **Ruby** | **4.0.x** (latest stable, 4.0.5 — May 2026) | Ruby went 3.4 → **4.0**; there is no 3.5. Skills assume 4.0+ syntax/features. |
| **Rails** | **8.1.x** (latest stable, 8.1.3 — March 2026) | Default to Rails 8 conventions: Solid Queue/Cache/Cable, Propshaft, Kamal 2, Thruster, import maps, built-in auth generator, SQLite-production-ready. |
| **Compatibility floor** | Rails 7.2 / Ruby 3.3 noted where relevant | Menus call out when an option requires 8.x. |

The suite leads with **Rails 8 omakase defaults** as the "Recommended" menu
entry, then offers alternatives — that's how "menu-driven" and "latest Rails"
coexist without conflict.

## 4. Skill suite (proposed)

**14 skills** across five layers (down from 16 after merging
database→models, realtime→hotwire, observability→deploy — each merged topic stays
distinctly invocable from its host skill's `references/`; then +1 for
`rails-upgrade`). Names are `rails-*`. Each is one skill directory with
`SKILL.md` + `references/`. Later additions fold four more first-class Rails
subsystems into their natural hosts (still 14 skills, same distinctly-invocable
rule): inbound email (Action Mailbox)→mailers, rich text (Action Text)→hotwire,
i18n→controllers, GraphQL→api.

### Layer 0 — Foundation (entry points)
| Skill | Purpose | Key menus / decisions |
|---|---|---|
| **rails-scaffold** | **Greenfield entry point.** Orchestrator running an **end-to-end guided interview** for new apps: asks app type → DB → asset pipeline → CSS/JS → test framework → auth → job/cache/cable backends in sequence, runs `rails new` with the chosen flags, wires the picks, then hands off to sibling skills. Also detects + routes for existing apps. | App type (API-only / full-stack / API+frontend) · every concern below is asked |
| **rails-upgrade** | **Brownfield entry point** — the mirror of `rails-scaffold`. Drives **framework version upgrades** of an existing app toward the latest Ruby/Rails: detects current Ruby/Rails, plans the upgrade path (one minor at a time), bumps the `Gemfile`, runs `bin/rails app:update`, reconciles `config/initializers/new_framework_defaults_*.rb`, triages deprecations, and guides the omakase stack swaps. Then routes to sibling skills to adopt modern picks. Upgrades the *journey*; siblings own the *destination state*. | Upgrade path (target version steps) · Zeitwerk (classic → zeitwerk) · asset pipeline (Sprockets → Propshaft) · JS (Webpacker → importmaps / jsbundling / Vite) · jobs/cache/cable (Sidekiq+Redis → Solid Queue/Cache/Cable) · secrets (secrets.yml → encrypted credentials) |

### Layer 1 — Domain / data
| Skill | Purpose | Key menus / decisions |
|---|---|---|
| **rails-models** | ActiveRecord modeling **+ persistence** (merged): associations, validations, scopes, enums, callbacks, concerns, STI vs delegated types, value objects — **plus** DB adapter, migrations, indexing, multi-DB, full-text search, seeds. Persistence topics live in `references/` and stay distinctly invocable. | Inheritance strategy · DB adapter (PostgreSQL / MySQL / SQLite) · search (pg_search / Elastic / Meilisearch) |
| **rails-storage** | File uploads via Active Storage: variants, direct uploads, processing | Local / S3 / GCS / Azure · image processor (Vips / MiniMagick) |

### Layer 2 — Web / request layer
| Skill | Purpose | Key menus / decisions |
|---|---|---|
| **rails-controllers** | Routing, REST, controllers, strong params, concerns, sessions, flash, request lifecycle (shared by API + Web) **+ i18n** (per-request locale selection + the `I18n` catalog API; lives in `references/` and stays distinctly invocable) | RESTful structure · params patterns · response handling |
| **rails-api** | API-only concerns: serialization, versioning, pagination, error envelopes, OpenAPI docs, CORS, token auth surface **+ GraphQL** (graphql-ruby as an alternative API style; REST stays Recommended; lives in `references/` and stays distinctly invocable) | API style (REST / GraphQL) · Serializer (Jbuilder / Alba / JSONAPI-serializer / Blueprinter) · versioning strategy · pagination (Pagy / Kaminari) · docs (rswag / OpenAPI) |
| **rails-hotwire** | Full-stack frontend **+ real-time + rich text** (merged): Turbo (Drive/Frames/Streams), Stimulus, views, components, assets — **plus** Action Cable + Turbo Stream broadcasting (Solid Cable / Redis), **plus** Action Text / Trix rich-text editing. Real-time and rich-text live in `references/` and stay distinctly invocable. | Views (partials / ViewComponent / Phlex) · CSS (Tailwind / Bootstrap / vanilla + Propshaft) · JS (import maps / esbuild / Vite) · Cable adapter (Solid Cable / Redis) |

### Layer 3 — Cross-cutting app concerns
| Skill | Purpose | Key menus / decisions |
|---|---|---|
| **rails-auth** | Authentication **and** authorization (unified). Offers the top auth solutions as a menu and provides per-solution integration guidance (install → wire → routes/controllers → views or token surface → tests). | AuthN: Rails 8 generator + `has_secure_password` *(Recommended)* / Devise / Rodauth (rodauth-rails) / authentication-zero · social/3rd-party via OmniAuth · API token (Devise-JWT / custom) · AuthZ: Pundit / Action Policy / CanCanCan |
| **rails-jobs** | Background jobs via Active Job: queues, retries, scheduling, recurring tasks | Backend (Solid Queue / Sidekiq / GoodJob) · scheduler (Solid Queue recurring / sidekiq-cron / whenever) |
| **rails-mailers** | Action Mailer: transactional email, previews, delivery, templates **+ inbound email** (Action Mailbox: ingress, mailbox routing, processing; lives in `references/` and stays distinctly invocable) | Delivery (SMTP / Postmark / SendGrid / SES) · multipart/templating |

### Layer 4 — Quality
| Skill | Purpose | Key menus / decisions |
|---|---|---|
| **rails-testing** | Test strategy and setup | Framework (Minitest / RSpec) · data (fixtures / FactoryBot) · system tests (Capybara + driver) · coverage (SimpleCov) |
| **rails-security** | **Proactive** build-time hardening: strong params, CSRF, CSP, secrets/credentials, rate limiting, dependency scanning. Complements the existing **`rails-audit`** skill (retrospective vulnerability review) — build-secure vs find-insecure. Security is also embedded cross-cutting in `rails-auth`, `rails-api`, and `rails-deploy`. | Static analysis (Brakeman) · rate limiting (rack-attack / Rails 8 built-in) · secrets handling |
| **rails-performance** | Caching and optimization | Cache store (Solid Cache / Redis / Memcached) · N+1 detection (Bullet / strict_loading) · fragment/russian-doll caching |

### Layer 5 — Lifecycle / ops
| Skill | Purpose | Key menus / decisions |
|---|---|---|
| **rails-deploy** | Ship it **+ observability** (merged): Kamal 2, Docker, Thruster, CI/CD, asset precompile, health checks, zero-downtime — **plus** logs, errors, metrics, APM. Observability lives in `references/` and stays distinctly invocable. | Orchestration (Kamal / Docker Compose / PaaS) · CI (GitHub Actions / others) · error tracking (Sentry / Honeybadger / AppSignal) |

### Relationship to the existing `rails-audit` skill

`rails-audit` ([jcuervo/rails-audit-claude-skill](https://github.com/jcuervo/rails-audit-claude-skill))
is **referenced, not forked**. It is a separately-maintained repo that performs a
*retrospective* audit (coding standards, testing, security, architecture,
performance → PDF deck). Our suite is *proactive / build-time*. They are
complements: **build-it-right** vs **find-what's-wrong**.

Integration plan:
- **Not part of the 13 authored skills.** It stays upstream and standalone.
- **First-class member of the collection by reference** — listed in the suite
  README/index as the "audit / review" skill.
- **Rubric reuse, not duplication.** Our quality skills (`rails-security`,
  `rails-testing`, `rails-performance`) treat the audit's
  `security-checklist.md` and `coding-standards-reference.md` as the canonical
  rubric they build *toward*, and cross-link to it ("for a full retrospective
  audit → `rails-audit`"). One definition of "good," used in both directions.
- **Bundling (only if we package — see Q6):** include via **git submodule**
  pointing at the upstream repo, preserving sync and attribution to jcuervo.

## 5. Skill anatomy (house style)

Each skill follows the `rn-*` template:

```
rails-<concern>/
  SKILL.md                 # frontmatter + when-to-apply + menu index
  references/
    <option-or-topic>.md   # per-option deep dives / patterns
```

`SKILL.md` structure:
1. **Frontmatter** — `name`, `description`, `metadata` (owner/status); `user-invocable` + `argument-hint` where it makes sense.
2. **Purpose** — one paragraph.
3. **When to Apply / When NOT to Apply** — with cross-links to sibling skills ("a test → read `../rails-testing/SKILL.md`").
4. **Menu** — table of vetted options, one-line trade-off each, one marked **Recommended** (the Rails 8 default).
5. **Decision Flow** — criteria, not opinions, for picking.
6. **Per-option deep dives** (in `references/`): Quick Pattern (install + minimum wiring) · When to pick / not pick · Deep Dive · Common Pitfalls · **Verify** (runnable steps to confirm it works).
7. **Problem → Reference table** mapping tasks to files.

## 6. Cross-cutting conventions

- **Menu-first, ask-once.** Present the menu via `AskUserQuestion`; if the project already has a pick installed (detected from `Gemfile`/config), use it silently.
- **Detect before generate.** Always read `Gemfile`/`Gemfile.lock`/`config/` to learn Ruby/Rails version and existing choices before acting.
- **API vs Web awareness.** Shared skills branch on `config.api_only`. API-only skips Hotwire; Web skips serializers.
- **Menu everything.** Every concern is asked — including the omakase choices (queue/cache/cable backend, asset pipeline, JS bundler). The Rails 8 default is marked **Recommended** within each menu, but the agent still asks unless a pick is already installed. Maximally unopinionated.
- **Idempotent guidance.** Patterns are safe to apply to an existing app, not just a fresh `rails new`.
- **Verify, don't just write.** Every skill ends with runnable verification — boot the app / run the generator / hit the endpoint / run the relevant tests — to confirm the wiring works before finishing.
- **Cross-linking.** Skills reference siblings by relative path, never duplicate each other's coverage.

## 7. Proposed build order

1. **rails-scaffold** (the orchestrator — defines menus the others reuse)
2. **rails-models** + **rails-controllers** (the spine; models now includes persistence/DB)
3. **rails-api** and **rails-hotwire** (the two app-type branches; hotwire now includes real-time)
4. **rails-auth**, **rails-testing** (highest-leverage cross-cutting)
5. **rails-jobs**, **rails-mailers**, **rails-storage**
6. **rails-security**, **rails-performance**
7. **rails-deploy** (now includes observability)
8. **rails-upgrade** (built **last**: it orchestrates the omakase stack swaps —
   Propshaft, importmaps, Solid Queue/Cache/Cable, encrypted credentials — by
   routing to sibling skills, so those menus must exist first. Mirror of
   `rails-scaffold`, but for brownfield.)

## 8. Open questions for discussion

1. **Skill count.** ✅ Resolved — merged to 13 (database→models, realtime→hotwire, observability→deploy), each merged topic still distinctly invocable; then **+1 `rails-upgrade`** (brownfield framework-version upgrades) = **14**.
2. **rails-auth shape.** ✅ Resolved — **unified** (authn + authz in one skill). Offers top auth solutions as a menu with per-solution integration guidance.
3. **Distribution.** ✅ Resolved — **public GitHub repo**. Needs an **installation guide** (clone / copy into `~/.claude/skills/`, or add as a plugin marketplace). `rails-audit` included as a **git submodule** per §4.
4. **Naming/namespace.** Plain `rails-*` (assumed default unless you object).
5. **Scaffold orchestrator depth.** ✅ Resolved — **guided end-to-end interview** for greenfield; detect-and-route for existing apps.
6. **"Unopinionated" boundary.** ✅ Resolved — **menu everything** (incl. omakase choices); Rails 8 default marked Recommended within each menu.
7. **Validation/runnability.** ✅ Resolved — **docs + runnable verification** in every skill.

**All open questions resolved. Brief is locked — ready to build.**

## 9. Repo deliverables (public GitHub)

Beyond the 14 skills, the public repo needs:
- **README.md** — what the suite is, the skill index/table, the menu philosophy.
- **INSTALL.md** (or README section) — install paths: clone into `~/.claude/skills/`, copy individual skills, or add as a plugin marketplace; plus `git submodule update --init` for the bundled `rails-audit`.
- **LICENSE** — pick one (MIT suggested) so the public repo is usable; note `rails-audit` submodule keeps its own upstream terms.
- **`.gitmodules`** — pinning `rails-audit` to [jcuervo/rails-audit-claude-skill](https://github.com/jcuervo/rails-audit-claude-skill).
- **CONTRIBUTING.md** (optional) — how to add a new menu option / skill.

---

*Next step: brief is locked. Build in the order in §7, starting with `rails-scaffold` (the orchestrator that defines the menus the rest reuse).*

Sources: [Rails releases](https://rubyonrails.org/category/releases) · [endoflife.date/rails](https://endoflife.date/rails) · [Ruby 4.0.5 release](https://www.ruby-lang.org/en/news/2026/05/20/ruby-4-0-5-released/) · [endoflife.date/ruby](https://endoflife.date/ruby)

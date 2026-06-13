# Rails Skills

A suite of **Claude Code Agent Skills** for building, testing, hardening, and
shipping **Ruby on Rails** applications — whether a JSON **API** backend or a
full-stack **Hotwire** web app — on the latest **Ruby 4.0.x** and **Rails 8.1.x**.

The defining principle is **maximally unopinionated, menu-driven**: every concern
(database, asset pipeline, JS bundler, auth, jobs backend, cache store, deploy
target, …) is presented as a **vetted menu of options** via `AskUserQuestion`, with
the Rails 8 omakase default marked **Recommended** — but the agent *asks* rather
than dictating, unless your project already constrains the pick. You choose the
stack; the skills implement it correctly and verify it works.

> These are **markdown skill files** (`SKILL.md` + `references/`), not a Rails app
> or a gem. They teach Claude Code *how* to build Rails apps; they run against
> *your* target project.

## What makes this suite different

- **Menu-first, ask-once.** Vetted options with one-line trade-offs; detect what's
  already installed and use it silently instead of re-asking.
- **Detect before generate.** Every skill reads your `Gemfile`/`Gemfile.lock`/
  `config/` to learn your Ruby/Rails version and existing choices before acting.
- **API vs Web aware.** Shared skills branch on `config.api_only` — API-only skips
  Hotwire; Web skips serializers.
- **Idempotent.** Patterns are safe on an existing app, not only a fresh `rails new`.
- **Verify, don't just write.** Every skill ends in runnable verification — boot the
  app, run the generator, hit the endpoint, run the tests — to confirm the wiring
  works.
- **Composable, precisely invocable.** One skill per concern; they cross-link by
  relative path and never duplicate each other's coverage.

## The skills

Fourteen authored `rails-*` skills across six layers, plus the referenced
`rails-audit` (bundled as a git submodule):

### Layer 0 — Entry points
| Skill | Use it to… |
|---|---|
| [`rails-scaffold`](rails-scaffold/SKILL.md) | **Greenfield.** Run a guided `rails new` interview (app type → DB → assets/CSS/JS → omakase backends → test framework → auth), generate the app, verify it boots, and route to the siblings. |
| [`rails-upgrade`](rails-upgrade/SKILL.md) | **Brownfield.** Upgrade an existing app's framework version toward the latest Ruby/Rails — incremental path, `app:update`, framework-defaults flip, Zeitwerk, then route the omakase stack swaps to the siblings. |

### Layer 1 — Domain / data
| Skill | Use it to… |
|---|---|
| [`rails-models`](rails-models/SKILL.md) | ActiveRecord modeling **+ persistence**: associations, validations, scopes, enums, inheritance (STI/delegated types), migrations, indexing, multi-DB, full-text search, seeds. |
| [`rails-storage`](rails-storage/SKILL.md) | File uploads via Active Storage: attachments, image variants (libvips/MiniMagick), direct uploads, services (local/S3/GCS/Azure). |

### Layer 2 — Web / request layer
| Skill | Use it to… |
|---|---|
| [`rails-controllers`](rails-controllers/SKILL.md) | Routing, REST, controllers, strong params (`params.expect`), filters/concerns, error handling, sessions/flash, i18n/locale switching. |
| [`rails-api`](rails-api/SKILL.md) | JSON APIs: serialization (Jbuilder/Alba/…), versioning, pagination, error envelopes, CORS, OpenAPI docs, token surface — plus **GraphQL** (graphql-ruby) as the alternative API style. |
| [`rails-hotwire`](rails-hotwire/SKILL.md) | Full-stack frontend **+ real-time**: Turbo (Drive/Frames/Streams/morphing), Stimulus, view layer (partials/ViewComponent/Phlex), Action Cable broadcasting, Action Text rich text. |

### Layer 3 — Cross-cutting app concerns
| Skill | Use it to… |
|---|---|
| [`rails-auth`](rails-auth/SKILL.md) | Authentication **and** authorization: Rails 8 generator / Devise / Rodauth / authentication-zero, OmniAuth, API tokens, Pundit / Action Policy / CanCanCan. |
| [`rails-jobs`](rails-jobs/SKILL.md) | Background jobs via Active Job: Solid Queue / Sidekiq / GoodJob, retries/idempotency, recurring tasks. |
| [`rails-mailers`](rails-mailers/SKILL.md) | Transactional email: mailers, multipart templates, previews, delivery (SMTP/Postmark/SendGrid/SES) — plus **inbound email** (Action Mailbox). |

### Layer 4 — Quality
| Skill | Use it to… |
|---|---|
| [`rails-testing`](rails-testing/SKILL.md) | Test strategy: Minitest/RSpec, fixtures/FactoryBot, model/request/system tests, coverage (SimpleCov). |
| [`rails-security`](rails-security/SKILL.md) | Proactive hardening: rate limiting, secrets, CSP/headers, CSRF, mass-assignment, scanning (Brakeman/bundler-audit). |
| [`rails-performance`](rails-performance/SKILL.md) | Caching + optimization: cache store (Solid Cache/Redis), N+1 detection, fragment/HTTP caching, query tuning. |

### Layer 5 — Lifecycle / ops
| Skill | Use it to… |
|---|---|
| [`rails-deploy`](rails-deploy/SKILL.md) | Ship it **+ observability**: Kamal 2, the production Docker image + Thruster, CI/CD, `/up` health + zero-downtime, logs/errors/metrics/APM. |

### Referenced (bundled as a submodule)
| Skill | Use it to… |
|---|---|
| `rails-audit` ([upstream](https://github.com/jcuervo/rails-audit-claude-skill)) | **Retrospective** audit of an existing app (coding standards, security, performance → report). The **find-insecure** complement to `rails-security`'s **build-secure**. Maintained upstream; included here via git submodule. |

## How they fit together

```
                rails-scaffold (new)        rails-upgrade (existing → latest)
                       │                              │
                       ▼                              ▼
   rails-models ──► rails-controllers ──► rails-api  |  rails-hotwire
        │                                    │             │
        └──────────► rails-auth ─────────────┴─────────────┘
                       │
   rails-jobs · rails-mailers · rails-storage
                       │
   rails-testing · rails-security · rails-performance
                       │
                       ▼
                 rails-deploy  (+ observability)
```

`rails-scaffold` writes a `STACK.md` recording every pick; sibling skills read it so
they don't re-ask. `rails-upgrade` owns the *journey* to the latest version and
routes each modernization to the sibling that owns the *destination state*.

## Installation

See **[INSTALL.md](INSTALL.md)** for the full options. Quickest path — copy the
skills into your Claude Code skills directory:

```bash
git clone --recurse-submodules https://github.com/jcuervo/rails-skills.git
cp -R rails-skills/rails-* ~/.claude/skills/        # personal scope
```

Then, in Claude Code working on a Rails project, invoke a skill (e.g.
`/rails-scaffold`) or just describe the task — Claude routes to the right skill.

## License

[MIT](LICENSE) © 2026 Richard Gonzales. The bundled `rails-audit` submodule retains
its own upstream license and authorship.

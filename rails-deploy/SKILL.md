---
name: rails-deploy
description: Ship a Rails 8.1 app to production and observe it in production — orchestration (Kamal 2, the Rails 8 default, or Docker Compose / a PaaS), the production Docker image with Thruster, asset precompile, CI/CD (GitHub Actions, which ships with Rails 8), the /up health check and zero-downtime cutover, secret injection, and database migration on deploy — PLUS observability (structured logs, error tracking via Sentry/Honeybadger/AppSignal, metrics, APM, uptime), which lives in references/ and stays distinctly invocable. Menu-driven for orchestration, CI, and error tracking with a Recommended default; detects the generated Dockerfile/deploy.yml/CI workflow first, and verifies a real deploy passes its health check. Apply when deploying, containerizing, setting up CI/CD, configuring zero-downtime/health checks, injecting secrets, or wiring logs/errors/metrics/APM.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[deploy-or-observability]"
---

# rails-deploy

## Purpose

Owns **shipping to production and observing it there** — the merged deploy +
observability skill. Deploy covers orchestration (Kamal 2/Docker Compose/PaaS), the
production Docker image (with Thruster), asset precompile, CI/CD, the `/up` health
check + zero-downtime cutover, secret injection, and migrating the DB on deploy.
**Observability** (structured logs, error tracking, metrics, APM, uptime) is the
merged topic, distinctly invocable in [observability.md](references/observability.md).
This is the **last mile**: sibling skills build the app; this one runs it in
production and tells you when it breaks.

## When to Apply

Use this skill when the task is:

- Deploying the app (Kamal, Docker Compose, or a PaaS)
- Building/optimizing the production Docker image, asset precompile
- Setting up CI/CD (test + scan + deploy pipeline)
- Health checks, zero-downtime deploys, rollbacks, migrating on deploy
- Injecting secrets at deploy time (registry creds, master key, env)
- Observability: logs, error tracking, metrics, APM, uptime monitoring

Do **not** use this skill when the task is:

- Choosing the secrets *strategy* in the app (encrypted credentials vs ENV) → read `../rails-security/SKILL.md` (this skill *injects* them at deploy; that one decides the store)
- Application-level rate limiting / CSP / scanning → read `../rails-security/SKILL.md` (CI *runs* the scanners; that skill *configures* them)
- The background-worker code/backend → read `../rails-jobs/SKILL.md` (this skill *runs* workers in prod — Solid Queue in Puma or a separate process)
- App-level caching choice → read `../rails-performance/SKILL.md`
- Running the test suite's content → read `../rails-testing/SKILL.md` (CI *invokes* it)
- DB schema/migration *authoring* → read `../rails-models/SKILL.md` (this skill *runs* `db:migrate` on deploy)
- A retrospective audit → use the upstream `rails-audit` skill

## Detect Before You Generate

```bash
ls Dockerfile .dockerignore config/deploy.yml .kamal/secrets 2>/dev/null   # Rails 8 generates these (unless --skip-kamal)
ls .github/workflows/ci.yml 2>/dev/null                                    # CI workflow (unless --skip-ci)
bundle list | grep -E "kamal|thruster"                                     # deploy tooling present
grep -E "^    (sentry-ruby|sentry-rails|honeybadger|appsignal|rollbar|lograge)" Gemfile.lock   # observability gems
grep -rn "config.force_ssl|health" config/ 2>/dev/null; grep -rn "/up" config/routes.rb
```

- **Rails 8 generates `Dockerfile`, `.dockerignore`, `config/deploy.yml`,
  `.kamal/secrets`, and `.github/workflows/ci.yml`** unless scaffolded with
  `--skip-kamal`/`--skip-ci`. Detect and **extend** them — don't regenerate.
- `/up` (Rails::HealthController) exists by default — use it for health checks.
- If an error-tracker/log gem is installed, match it.
- Secret *values* come from `../rails-security/`'s chosen store; this skill wires
  their *injection*. Honor `STACK.md`'s deploy + observability picks.

## Menu

Three menus. Each via `AskUserQuestion`, Rails-default marked **Recommended**; ask
unless already configured.

### Orchestration / deploy target
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Kamal 2** *(Recommended)* | Rails 8 default; deploys Docker containers to your own servers with kamal-proxy (zero-downtime, auto-SSL). No PaaS lock-in, low cost. You own the host. | [kamal.md](references/kamal.md) |
| Docker Compose (self-managed) | Single-host Compose; simplest "just run the containers." You manage proxy/SSL/zero-downtime yourself. | [orchestration-alternatives.md](references/orchestration-alternatives.md) |
| PaaS (Heroku / Render / Fly.io) | Managed platform; least ops, highest convenience, more cost + some lock-in. | [orchestration-alternatives.md](references/orchestration-alternatives.md) |

### CI/CD
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **GitHub Actions** *(Recommended)* | Ships with Rails 8 (`.github/workflows/ci.yml`: tests + Brakeman + RuboCop). Extend with a deploy job. | [ci-cd.md](references/ci-cd.md) |
| GitLab CI / CircleCI / other | Use to match your VCS/host; same stages, different YAML. | [ci-cd.md](references/ci-cd.md) |

### Error tracking *(observability)*
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Sentry** *(Recommended)* | `sentry-ruby` + `sentry-rails`; ubiquitous, great Rails integration, traces + errors. | [observability.md](references/observability.md) |
| Honeybadger | Errors + uptime + check-ins in one; simple, Rails-friendly. | [observability.md](references/observability.md) |
| AppSignal | Errors **and** APM/metrics in one product; good all-in-one. | [observability.md](references/observability.md) |

Others (Rollbar, etc.) follow the same shape — see [observability.md](references/observability.md).

## Decision Flow

- **Orchestration:** default to **Kamal 2** — it's the Rails 8 default, deploys to
  your own (cheap) servers with zero-downtime + auto-SSL via kamal-proxy, no PaaS
  lock-in. Choose a **PaaS** to minimize ops (accept cost/lock-in). **Docker
  Compose** for a simple single-host setup you manage. Same Docker image underneath
  all three.
- **CI/CD:** keep the generated **GitHub Actions** workflow (tests + Brakeman +
  RuboCop) and add a **deploy job** gated on green ([ci-cd.md](references/ci-cd.md)).
  Run the scanners `../rails-security/` configured; invoke the suite `../rails-testing/`
  owns.
- **Health & zero-downtime:** use `/up`; Kamal boots the new container, waits for
  `/up`, then cuts traffic over ([health-and-zero-downtime.md](references/health-and-zero-downtime.md)).
  Migrate on deploy carefully (backward-compatible migrations).
- **Secrets:** inject at deploy — `RAILS_MASTER_KEY`/registry creds via
  `.kamal/secrets` (Kamal) or platform env. The *store* is `../rails-security/`; this
  skill delivers them to the container.
- **Workers in prod:** run Solid Queue in Puma (`SOLID_QUEUE_IN_PUMA=1`) or as a
  separate Kamal role/process (`../rails-jobs/`).
- **Observability:** ship structured logs, an **error tracker** (Sentry by default),
  metrics/APM, and uptime checks ([observability.md](references/observability.md)) —
  you can't operate what you can't see.

## Problem → Reference

| Task | Read |
|---|---|
| Deploy with Kamal 2 (deploy.yml, secrets, proxy, setup/deploy, rollback) | [references/kamal.md](references/kamal.md) |
| Deploy via Docker Compose or a PaaS | [references/orchestration-alternatives.md](references/orchestration-alternatives.md) |
| Production Dockerfile, Thruster, asset precompile, image size | [references/docker-and-assets.md](references/docker-and-assets.md) |
| CI/CD pipeline (test + scan + deploy gating) | [references/ci-cd.md](references/ci-cd.md) |
| Health checks, zero-downtime cutover, migrate-on-deploy, rollback | [references/health-and-zero-downtime.md](references/health-and-zero-downtime.md) |
| Observability: logs, error tracking, metrics, APM, uptime | [references/observability.md](references/observability.md) |

## Verify

A deploy is unfinished until the new release passes its health check in production:

```bash
# Image builds and boots locally first:
docker build -t app . && docker run --rm -e RAILS_MASTER_KEY=$(cat config/master.key) -p 3000:3000 app & sleep 8
curl -fsS -o /dev/null -w "up=%{http_code}\n" localhost:3000/up        # expect 200
docker stop $(docker ps -q --filter ancestor=app) 2>/dev/null
# Kamal: config is valid and a deploy reaches a healthy release:
kamal config 2>/dev/null | head -5                                     # renders config
# kamal deploy        # → builds, pushes, boots, waits for /up, cuts over
# kamal app exec "bin/rails runner 'puts Rails.env'"                   # prod container responds
# CI runs the gate:
grep -rqi "brakeman\|rails test\|rspec" .github/workflows/ && echo "✓ CI runs tests + scan"
# Observability wired (error tracker initializes):
bin/rails runner "p defined?(Sentry) || defined?(Honeybadger) || defined?(Appsignal) || 'add an error tracker'" 2>/dev/null
```

This is the **terminal skill** of the suite. After a healthy deploy, the loop back
to development is via the sibling skills; record the deploy target + observability
stack in `STACK.md`. For a brownfield app reaching modern Rails first, that journey
is `../rails-upgrade/`.

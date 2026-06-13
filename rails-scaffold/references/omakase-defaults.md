# Omakase Backends

Rails 8 ships a "batteries-included" stack — the omakase defaults. Per the suite's
**menu-everything** rule, ask keep-or-skip for each rather than assuming. Each
maps to a `--skip-*` (or opt-in) flag. Recommended = keep the default. A single
multi-select `AskUserQuestion` ("which omakase defaults do you want to keep?")
works well here.

## Menu

| Backend | Keep (default) | Skip flag | What it is |
|---|---|---|---|
| **Solid trio** *(Recommended: keep)* | Solid Queue + Cache + Cable | `--skip-solid` | DB-backed jobs, cache, and WebSockets — **no Redis required**. |
| **Kamal 2** *(Recommended: keep)* | `config/deploy.yml` + Dockerfile | `--skip-kamal` | Zero-downtime container deploys to any SSH host. |
| **Thruster** *(Recommended: keep)* | Wraps Puma | `--skip-thruster` | HTTP/2, TLS, X-Sendfile, and asset caching in front of Puma. |
| **CI workflow** *(Recommended: keep)* | `.github/workflows/ci.yml` | `--skip-ci` | GitHub Actions: runs tests, RuboCop, Brakeman. |
| **RuboCop** *(Recommended: keep)* | `rubocop-rails-omakase` | `--skip-rubocop` | DHH's omakase style rules + `bin/rubocop`. |
| **Brakeman** *(Recommended: keep)* | `bin/brakeman` | `--skip-brakeman` | Static security scanner. |
| **Dev container** *(default: off)* | `.devcontainer/` | `--devcontainer` *(opt-in)* | VS Code / Codespaces reproducible dev env. |

## Decision Flow

### Solid trio (`--skip-solid`)
- **Keep** for almost everything: it removes the Redis dependency and works on the
  app's own database. This is the single biggest Rails 8 simplification.
- **Skip** only if you've *already decided* on **Sidekiq** or **GoodJob**, or you
  run a Redis-backed cache/cable elsewhere. If the user wants Sidekiq: pass
  `--skip-solid` here, then let `../rails-jobs/SKILL.md` install Sidekiq and
  `../rails-performance/SKILL.md` choose the cache store. Don't half-install both.
- SQLite + Solid is fully supported; you don't need Postgres for Solid.

### Kamal 2 (`--skip-kamal`)
- **Keep** if you'll deploy to your own VMs/bare metal/any SSH host — Kamal is the
  Rails 8 deploy story and `../rails-deploy/` builds on it.
- **Skip** if you deploy to a **PaaS** (Heroku, Render, Fly.io) or a Kubernetes
  platform that owns the deploy. Keeping it is harmless (just unused files), so
  when unsure, keep.

### Thruster (`--skip-thruster`)
- **Keep** by default — it's a thin, useful front for Puma (asset caching, HTTP/2).
- **Skip** if a reverse proxy / CDN / load balancer already does TLS + caching and
  you don't want the extra layer.

### CI / RuboCop / Brakeman
- **Keep all three** unless the team has an existing, conflicting setup. They're
  cheap, and `../rails-security/` + `../rails-testing/` assume Brakeman/RuboCop
  exist. Skip RuboCop only if the team uses **Standard** or a custom style; skip CI
  only if CI is defined elsewhere (GitLab CI, CircleCI, Buildkite).

### Dev container (`--devcontainer`)
- **Off by default.** Add `--devcontainer` when the team uses Codespaces or wants a
  one-command reproducible environment (it provisions the DB + Redis-or-not in
  containers). Opt-in, not skip.

## How picks interact with siblings

| If user picks… | Then at scaffold | And later |
|---|---|---|
| Sidekiq for jobs | `--skip-solid` | `../rails-jobs/` installs Sidekiq + Redis |
| Redis cache | `--skip-solid` (or keep Solid Queue/Cable, swap cache) | `../rails-performance/` sets cache store |
| Heroku/Render deploy | `--skip-kamal` | `../rails-deploy/` covers the PaaS path |
| Standard style | `--skip-rubocop` | `../rails-testing/`/CI wires Standard |

> Note: `--skip-solid` removes **all three** Solid components. If you want Solid
> Cache/Cable but Sidekiq for jobs, keep Solid here and let `../rails-jobs/`
> override just the Active Job adapter — don't `--skip-solid` for that case.

## Pitfalls

- `--skip-solid` **and** not installing any job backend → `Active Job` has no
  durable queue; jobs run inline/async-in-memory and vanish on restart. Always
  follow `--skip-solid` with a real backend via `../rails-jobs/`.
- Keeping Kamal but never configuring it is fine (dormant files); skipping it then
  wanting it back means hand-creating `config/deploy.yml` — when unsure, keep.
- Skipping Brakeman/RuboCop quietly removes the CI's lint/scan steps too, weakening
  `../rails-security/`'s baseline. Skip only with a replacement in mind.

## Verify

```bash
# Solid trio present?
bundle list | grep -E "solid_queue|solid_cache|solid_cable" || echo "Solid skipped"
ls config/queue.yml config/cache.yml config/cable.yml 2>/dev/null
# Kamal / Thruster
ls config/deploy.yml 2>/dev/null && echo "Kamal kept"
bundle list | grep thruster || echo "Thruster skipped"
# CI / RuboCop / Brakeman
ls .github/workflows/ci.yml 2>/dev/null && echo "CI kept"
bin/rubocop --version 2>/dev/null && bin/brakeman --version 2>/dev/null
# Dev container
ls .devcontainer/devcontainer.json 2>/dev/null && echo "devcontainer on"
```

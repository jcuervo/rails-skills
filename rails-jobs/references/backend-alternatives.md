# Backend Alternatives — Sidekiq & GoodJob

When Solid Queue ([solid-queue.md](solid-queue.md)) isn't the fit. Both are mature;
pick **one** backend and write jobs against the Active Job API so the choice stays
swappable ([writing-jobs.md](writing-jobs.md)).

## Sidekiq (Redis-backed, high throughput)

```ruby
# Gemfile: gem "sidekiq"
# config/application.rb (or an initializer/env file):
config.active_job.queue_adapter = :sidekiq
```
```yaml
# config/sidekiq.yml
:concurrency: 10
:queues: [real_time, default, low]
```
```bash
bundle exec sidekiq          # runs the worker process (needs Redis running)
# Web UI: mount Sidekiq::Web in routes (behind auth — ../rails-auth/, ../rails-security/)
```

- **Pick when** you need very high throughput, a battle-tested ecosystem, the
  Sidekiq Web dashboard, and you're willing to run/operate **Redis**.
- **Don't pick when** you'd rather avoid Redis (Solid Queue/GoodJob are DB-backed).
- Through Active Job, jobs are portable; for Sidekiq-specific features (batches,
  native `Sidekiq::Job`) you can drop to its API, but that ties code to Sidekiq.
- **If you chose `--skip-solid` at scaffold** to use Sidekiq, that was the right
  call — `../rails-scaffold/` keeps Solid out, then you install Sidekiq here.

## GoodJob (Postgres-backed, dashboard)

```ruby
# Gemfile: gem "good_job"
config.active_job.queue_adapter = :good_job
```
```bash
bin/rails g good_job:install && bin/rails db:migrate
bundle exec good_job start          # separate process; or run in-process via config
```

- **Pick when** you're **Postgres-only** and want a feature-rich, Redis-free backend
  with a built-in dashboard, cron, concurrency controls, and batches — and prefer
  GoodJob's maturity/features over Solid Queue.
- **Don't pick when** you're not on Postgres (it relies on Postgres advisory locks /
  LISTEN-NOTIFY) or Solid Queue already covers your needs.

## Choosing

| Need | Pick |
|---|---|
| Default, no Redis, Rails 8 | Solid Queue *(Recommended)* |
| Max throughput, Redis ecosystem, Web UI | Sidekiq |
| Postgres-only, dashboard, Redis-free, feature-rich | GoodJob |

## Common Pitfalls

- **Redis not running / wrong URL** (Sidekiq) → jobs never process; set
  `REDIS_URL` and confirm connectivity.
- **GoodJob on non-Postgres** → it depends on Postgres features; won't work on
  SQLite/MySQL.
- **Coupling job code to a backend's native API** → loses portability; stick to
  Active Job unless you truly need backend-specific features.
- **Unauthenticated Sidekiq/GoodJob web dashboard** → exposes job data/controls;
  gate it behind auth (`../rails-auth/`, `../rails-security/`).
- **Two backends configured** → ambiguous adapter; pick one per environment.

## Verify

```bash
bundle list | grep -E "sidekiq|good_job"                  # the chosen backend is installed
bin/rails runner "p ActiveJob::Base.queue_adapter.class.name"   # matches the choice
# Sidekiq: Redis reachable + worker drains a job
redis-cli ping 2>/dev/null && bundle exec sidekiq & W=$!; sleep 3; kill $W 2>/dev/null
# GoodJob:
bundle exec good_job start & G=$!; sleep 3; kill $G 2>/dev/null
```

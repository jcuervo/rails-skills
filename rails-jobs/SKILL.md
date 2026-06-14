---
name: rails-jobs
description: Add background jobs to a Rails 8.1 app via Active Job — choose the backend (Solid Queue, the Rails 8 default, or Sidekiq / GoodJob), write jobs with retries/discards/idempotency, enqueue with perform_later, and schedule recurring tasks (Solid Queue recurring, sidekiq-cron, or whenever). Menu-driven with a Recommended default per menu; detects the installed backend (Solid Queue is preconfigured in Rails 8, or skipped via --skip-solid), applies equally to API-only and full-stack apps, and verifies a job actually enqueues and runs. Apply when moving work off the request thread, configuring a queue backend, adding retries/error handling to a job, or scheduling recurring/cron work.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[job-or-task]"
---

# rails-jobs

## Purpose

Owns **background/asynchronous work** through Active Job: the queue backend, the
jobs themselves (retries, discards, idempotency, arguments), enqueuing, and
recurring/scheduled tasks. Active Job is the backend-agnostic API; this skill picks
the adapter beneath it and writes jobs against the stable API so swapping backends
doesn't rewrite job code. Applies equally to API-only and full-stack apps — work
that shouldn't block a request (email, image processing, third-party calls, bulk
updates) belongs here.

## When to Apply

Use this skill when the task is:

- Moving slow/external work off the request thread (`deliver_later`, processing, API calls)
- Choosing or configuring a queue backend (Solid Queue / Sidekiq / GoodJob)
- Writing an Active Job with retries, discards, priorities, or idempotency
- Scheduling recurring or cron-style jobs
- Purging/anonymizing data past a retention window on a schedule (data retention)
- Running/operating the worker process locally

Do **not** use this skill when the task is:

- Sending the email itself (mailer classes, templates, delivery) → read `../rails-mailers/SKILL.md` (this skill runs `deliver_later`; that one builds the mail)
- Processing an uploaded file's variants → read `../rails-storage/SKILL.md` (jobs may *drive* it; Active Storage owns the transform)
- The model data a job touches → read `../rails-models/SKILL.md`
- Real-time broadcasting from a job (Turbo Streams over Cable) → read [`../rails-hotwire/`](../rails-hotwire/references/real-time.md) (real-time.md)
- Deploying/operating workers in production (Kamal, supervision, scaling) → read `../rails-deploy/SKILL.md`
- Caching (Solid Cache) → read `../rails-performance/SKILL.md`
- Testing jobs → read `../rails-testing/SKILL.md`

## Detect Before You Generate

```bash
grep -E "^    (solid_queue|sidekiq|good_job|resque)" Gemfile.lock     # installed backend
grep -nE "active_job|queue_adapter" config/application.rb config/environments/*.rb   # configured adapter
ls config/queue.yml config/recurring.yml db/queue_schema.rb 2>/dev/null   # Solid Queue present (Rails 8 default)
cat config/cable.yml >/dev/null 2>&1; bin/rails about 2>/dev/null | grep -i "rails ver"
```

- **Solid Queue is preconfigured in Rails 8** unless the app was scaffolded with
  `--skip-solid`. If `config/queue.yml` / `db/queue_schema.rb` exist, it's already
  there — use it; don't add another backend without reason.
- If Sidekiq/GoodJob is installed, match it — don't re-ask the menu.
- Read the configured `config.active_job.queue_adapter` per environment before
  changing it.
- Honor any jobs backend recorded in `STACK.md` by `../rails-scaffold/`. If its
  **`Compliance`** row lists `gdpr`/`ccpa`/`hipaa`, **proactively offer** a scheduled
  retention purge for models holding personal data — see
  [data-retention.md](references/data-retention.md). `none`/absent → add one only when
  asked; offering still means asking before scheduling a destructive job.

## Menu

Two menus. Each via `AskUserQuestion`, Rails-default marked **Recommended**; ask
unless the backend is already installed.

### Job backend
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Solid Queue** *(Recommended)* | Rails 8 default; DB-backed, no Redis, supports priorities/concurrency/recurring. Runs in Puma or a separate supervisor. | [solid-queue.md](references/solid-queue.md) |
| Sidekiq | Redis-backed, very high throughput, mature ecosystem/Web UI. Adds Redis to operate. | [backend-alternatives.md](references/backend-alternatives.md) |
| GoodJob | Postgres-backed (advisory locks), feature-rich, dashboard; Postgres-only. | [backend-alternatives.md](references/backend-alternatives.md) |

### Recurring / scheduled tasks
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Solid Queue recurring** *(Recommended)* | Built-in recurring tasks via `config/recurring.yml`; no extra gem. | [scheduling-recurring.md](references/scheduling-recurring.md) |
| sidekiq-cron / sidekiq-scheduler | Cron-style recurring on Sidekiq; pick if you're already on Sidekiq. | [scheduling-recurring.md](references/scheduling-recurring.md) |
| whenever (system cron) | Generates a crontab that runs rake/runner tasks; OS-level, backend-agnostic. | [scheduling-recurring.md](references/scheduling-recurring.md) |

## Decision Flow

- **Backend:** default to **Solid Queue** — it ships with Rails 8, needs no Redis,
  and handles priorities, concurrency limits, and recurring jobs. Choose **Sidekiq**
  for very high throughput / an established Redis + Sidekiq operational story.
  Choose **GoodJob** for a Postgres-only shop wanting its dashboard/features without
  Redis. Don't run two job backends.
- **Where the worker runs:** Solid Queue can run **inside Puma** (`SOLID_QUEUE_IN_PUMA=1`,
  simplest) or as a **separate supervisor** (`bin/jobs`) for isolation/scaling — that
  operational choice is `../rails-deploy/`; this skill makes the job *run* locally.
- **Recurring:** **Solid Queue recurring** unless you're on Sidekiq
  (sidekiq-cron) or want OS cron independent of the backend (whenever).
- **Data retention:** a scheduled purge is a recurring job that deletes or anonymizes
  data past a retention window. Hard-delete rows with no value past the window
  (`in_batches.delete_all`); anonymize rows you must keep via the model's erase path
  (`../rails-models/` `data-protection.md`). The window is a **policy value the team
  sets** — never hard-code a guessed number. See [data-retention.md](references/data-retention.md).
- **Job design:** keep jobs **small, idempotent, and retry-safe** — pass record IDs
  (GlobalID) not objects, make re-runs harmless, and decide `retry_on` vs
  `discard_on` per error class. See [writing-jobs.md](references/writing-jobs.md).

## Problem → Reference

| Task | Read |
|---|---|
| Solid Queue: install, run (Puma vs `bin/jobs`), queues, concurrency | [references/solid-queue.md](references/solid-queue.md) |
| Sidekiq or GoodJob: when + how to set up | [references/backend-alternatives.md](references/backend-alternatives.md) |
| Write a job: perform_later, arguments, retry_on/discard_on, idempotency | [references/writing-jobs.md](references/writing-jobs.md) |
| Recurring/scheduled tasks (Solid Queue recurring / sidekiq-cron / whenever) | [references/scheduling-recurring.md](references/scheduling-recurring.md) |
| Data retention: scheduled purge/anonymize past a window (batching, idempotency, the Compliance switch) | [references/data-retention.md](references/data-retention.md) |

## Verify

A jobs change is unfinished until a job actually enqueues and a worker runs it:

```bash
bin/rails db:prepare                                   # Solid Queue tables exist (if used)
# Enqueue a job and confirm it lands on the queue:
bin/rails runner "ExampleJob.perform_later('hi'); p ActiveJob::Base.queue_adapter.class.name" 2>/dev/null
# Run the worker and confirm it processes (Solid Queue):
bin/jobs &  JOBS=$!; sleep 3; kill $JOBS 2>/dev/null    # or SOLID_QUEUE_IN_PUMA=1 with the web server
# Inline adapter makes a job run synchronously in a quick check:
bin/rails runner "ActiveJob::Base.queue_adapter = :inline; ExampleJob.perform_later('hi')" 2>/dev/null
```

Then route to `../rails-mailers/` (deliver_later), `../rails-storage/` (process
uploads), `../rails-testing/` (job specs), and `../rails-deploy/` (run workers in
prod). Record the backend + scheduler in `STACK.md`.

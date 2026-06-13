# Solid Queue (Recommended backend)

The Rails 8 default Active Job backend: **database-backed**, no Redis. Preconfigured
in new Rails 8 apps (unless scaffolded with `--skip-solid`). Supports priorities,
concurrency limits, delayed jobs, bulk enqueue, and recurring tasks.

## Quick Pattern — it's likely already there

```bash
# New Rails 8 app: Solid Queue is wired. Just make sure its tables exist:
bin/rails db:prepare         # loads db/queue_schema.rb into the queue database
# If adding to an app that skipped it:
# bundle add solid_queue && bin/rails solid_queue:install && bin/rails db:prepare
```

```ruby
# config/environments/production.rb (set by the scaffold when Solid is kept)
config.active_job.queue_adapter = :solid_queue
config.solid_queue.connects_to = { database: { writing: :queue } }   # its own DB (config/database.yml)
```

Solid Queue uses a **separate `queue` database** by default (see
[`../rails-models/` multi-database.md](../../rails-models/references/multi-database.md)
— that's where the `queue` entry in `config/database.yml` comes from). SQLite + Solid
works out of the box.

## Running the worker

Two ways — the choice is operational (`../rails-deploy/` owns prod):

```bash
# A) Inside Puma (simplest): the supervisor runs in the web process.
SOLID_QUEUE_IN_PUMA=1 bin/rails server         # plugin starts/supervises Solid Queue

# B) Separate supervisor process (isolation/scaling):
bin/jobs                                        # runs the Solid Queue supervisor + workers
```

`bin/dev` (Procfile.dev) commonly runs `bin/jobs` alongside the server in
development. Confirm the exact binstub/command for your version
(`ls bin/jobs`, `bin/jobs --help`).

## Queues, priorities, concurrency

```ruby
class ReportJob < ApplicationJob
  queue_as :low                       # route to a named queue
end
ReportJob.set(priority: 10).perform_later(report)   # numeric priority (lower runs first by default)
```

```yaml
# config/queue.yml — worker pools, polling, concurrency per queue (confirm keys for your version)
production:
  workers:
    - queues: [real_time, default]
      threads: 5
    - queues: [low]
      threads: 2
```

- **Concurrency controls** (limit N jobs of a kind running at once) are a Solid
  Queue feature — declare them on the job; check the current DSL in the Solid Queue
  README, as it evolves.
- Define queues by **role/latency** (`real_time`, `default`, `low`), not by job
  name, so workers can be allocated by importance.

## When to pick / not pick

- **Pick when** you want zero extra infrastructure (no Redis), you're on Rails 8,
  and your throughput is moderate-to-high. The default for new apps.
- **Don't pick when** you need Sidekiq-class throughput at very high volume or
  already operate Redis + Sidekiq — see
  [backend-alternatives.md](backend-alternatives.md).

## Common Pitfalls

- **Queue tables missing** → `bin/rails db:prepare` didn't load `queue_schema.rb`;
  jobs enqueue but nothing processes. Confirm the `queue` DB exists.
- **No worker running** → in dev you must run `bin/jobs` (or `SOLID_QUEUE_IN_PUMA=1`);
  enqueued jobs just sit there otherwise.
- **One mega-queue** → a slow low-priority job starves latency-sensitive work; split
  queues and allocate workers.
- **Assuming Redis config** → Solid Queue is DB-backed; there's no Redis URL to set.
- **Forgetting the `queue` DB in production** (separate volume/connection) →
  `../rails-deploy/` covers persistence; the DB must survive deploys.

## Verify

```bash
bin/rails db:prepare
bin/rails runner "p ActiveJob::Base.queue_adapter.class.name"     # => ...SolidQueue... (or your adapter)
bin/rails runner "ExampleJob.perform_later('hi'); p SolidQueue::Job.count rescue p 'enqueued'"   # row appears
bin/jobs & JOBS=$!; sleep 3; kill $JOBS 2>/dev/null               # worker boots (and begins draining the queue)
```

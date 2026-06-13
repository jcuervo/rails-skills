# Recurring & Scheduled Tasks

Running work on a schedule (nightly reports, cleanup, polling). The menu pick is
**Solid Queue recurring** (built-in, no extra gem); alternatives match your backend
or use OS cron.

## Solid Queue recurring (Recommended)

```yaml
# config/recurring.yml  (confirm the filename/keys for your Solid Queue version)
production:
  cleanup_sessions:
    class: CleanupSessionsJob
    schedule: every day at 3am          # human cron-ish syntax
  poll_inbox:
    class: PollInboxJob
    schedule: every 15 minutes
```

- **Pick when** you're on Solid Queue (the Rails 8 default) — recurring tasks are
  built in, stored in the DB, no extra dependency. The recurring scheduler runs as
  part of the Solid Queue supervisor (`bin/jobs` / `SOLID_QUEUE_IN_PUMA`).
- Each entry maps a schedule to an Active Job class. The job still follows
  [writing-jobs.md](writing-jobs.md) (idempotent, retry-safe).
- Verify the exact `config/recurring.yml` schema against the Solid Queue README — the
  schedule DSL and file location have evolved.

## sidekiq-cron / sidekiq-scheduler (if on Sidekiq)

```ruby
# Gemfile: gem "sidekiq-cron"
# config/initializers/sidekiq.rb or a schedule.yml loaded at boot:
Sidekiq::Cron::Job.create(name: "Nightly report", cron: "0 2 * * *", class: "NightlyReportJob")
```

- **Pick when** your backend is Sidekiq — keep scheduling in the same system you
  already operate.
- **Don't pick when** you're not on Sidekiq; use Solid Queue recurring or whenever.

## whenever (system cron, backend-agnostic)

```ruby
# Gemfile: gem "whenever", require: false
# config/schedule.rb
every 1.day, at: "4:30 am" do
  runner "NightlyReportJob.perform_later"     # enqueue via your Active Job backend
end
# Deploy: `whenever --update-crontab` writes the OS crontab (often a Kamal/deploy hook)
```

- **Pick when** you want OS-level cron independent of the job backend, or to run
  rake/runner tasks on a schedule without a backend scheduler.
- **Don't pick when** the backend's own scheduler suffices — whenever adds a crontab
  to manage and deploy. **Caveat:** OS cron lives on **one host**; in a multi-host
  deploy you must ensure it runs on exactly one, or you get duplicate runs
  (`../rails-deploy/`).

## Design notes

- **Always enqueue, don't do work in the scheduler.** A recurring entry should
  `perform_later` a job, so the work runs on a worker with retries — not in the
  scheduler thread/cron process.
- **Idempotency matters more here** — a schedule can fire late, twice, or overlap;
  guard the job ([writing-jobs.md](writing-jobs.md)).
- **Timezones:** schedules run in the app/server timezone; set it explicitly so
  "3am" means what you expect.

## Common Pitfalls

- **Heavy work directly in cron/scheduler** → no retries, blocks the scheduler;
  enqueue a job instead.
- **whenever cron on every host** → N duplicate runs in a multi-host deploy; pin to
  one host.
- **Overlapping runs** of a long task → use a concurrency limit / lock so run N+1
  doesn't start before N finishes.
- **Stale `recurring.yml` schema** across Solid Queue versions → verify keys/format.
- **Wrong timezone** → "nightly" job runs midday; set the app TZ.

## Verify

```bash
# Solid Queue recurring config is present and parses:
test -f config/recurring.yml && echo "✓ recurring.yml present"
# The scheduled job class exists and is enqueuable:
bin/rails runner "p NightlyReportJob.respond_to?(:perform_later)" 2>/dev/null
# whenever: the generated crontab is valid
bundle exec whenever 2>/dev/null | head -5 || echo "(whenever not used)"
# Run the supervisor briefly; recurring tasks register on boot:
bin/jobs & J=$!; sleep 3; kill $J 2>/dev/null
```

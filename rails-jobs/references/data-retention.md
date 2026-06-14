# Data Retention & Automated Purge

The scheduled side of data minimization: a **recurring job that deletes or
anonymizes data past its retention window** — old soft-deleted records, expired
logs, stale exports, abandoned signups. This is the *storage-limitation* primitive
(don't keep personal data longer than needed).

> **Scope guard.** This builds the *purge mechanism*. **Retention periods are a legal
> /policy decision** (how long is "needed"?), not a code default — set them with
> counsel, never guess a number in a skill. We give you the lever and a safe place to
> declare the window; the value is yours. Whether a row counts as personal data and
> *how* to scrub it is owned by `../rails-models/` ([data-protection.md](../../rails-models/references/data-protection.md)).

This reference owns **scheduled execution**: the recurring purge job, batching, and
cutoffs. It composes three things already defined elsewhere — don't duplicate them:

- **What "erase/anonymize" means per model** → [data-protection.md](../../rails-models/references/data-protection.md)
  (delete-vs-anonymize, cascade correctness, `encrypts`).
- **How a recurring task is scheduled** → [scheduling-recurring.md](scheduling-recurring.md)
  (Solid Queue recurring / sidekiq-cron / whenever).
- **Idempotent, retry-safe job shape** → [writing-jobs.md](writing-jobs.md).

## When this applies (the switch)

Read the `STACK.md` `Compliance` row (set by `../rails-scaffold/`). If it lists
`gdpr`, `ccpa`, or `hipaa`, **proactively offer** a retention purge for models that
hold personal data — at the moment you add such a model or its retention is
discussed. `none`/absent → add a purge only when the task explicitly asks. Offering
is asking; nothing here auto-schedules a destructive job.

Non-compliance retention (trimming logs/tmp data for cost) is fine to suggest too —
it's the same mechanism without the legal framing.

## Quick Pattern

A retention window declared on the model, an idempotent batched purge job, and a
recurring entry that enqueues it.

```ruby
# app/models/audit.rb — the window lives next to the data it governs
RETENTION = 2.years                                  # policy value — set deliberately
scope :expired, -> { where("created_at < ?", RETENTION.ago) }
```
```ruby
# app/jobs/retention_purge_job.rb
class RetentionPurgeJob < ApplicationJob
  queue_as :maintenance
  def perform
    Audit.expired.in_batches(of: 1_000).delete_all   # bulk delete; see "batching" below
    User.soft_deleted.expired.find_each(&:erase_personal_data!)  # anonymize via rails-models
  end
end
```
```yaml
# config/recurring.yml — Solid Queue (verify schema for your version; see scheduling-recurring.md)
production:
  retention_purge:
    class: RetentionPurgeJob
    schedule: every day at 3am
```

## Decision: hard-delete vs anonymize on expiry
Same fork as erasure-on-request, applied on a timer — reuse the model's logic, don't
re-decide it here.
| On expiry | Use when | How |
|---|---|---|
| **Hard delete** | the row has no value past its window (logs, sessions, tmp exports) | `expired.in_batches.delete_all` (fast, skips callbacks/cascade) |
| **Anonymize** | the row must persist for accounting/analytics but the PII must go | call the model's `erase_personal_data!` (`data-protection.md`) per record |

## Deep Dive

- **Batch, always.** A purge can match millions of rows; one unbounded `delete_all`
  locks the table and bloats the transaction. `in_batches(of: N).delete_all` deletes
  in chunks. Use `find_each`/`find_in_batches` when you must run per-record logic
  (anonymize, cascade, callbacks) — slower but safe.
- **`delete_all` skips `dependent:`** — it issues one SQL `DELETE` and does **not**
  cascade or run callbacks. If the expired record has child rows holding PII, either
  purge children first (in dependency order) or use `destroy`/`erase_personal_data!`.
  Getting the association graph right is the same problem as request erasure — see
  [data-protection.md](../../rails-models/references/data-protection.md).
- **Idempotent by construction.** A retention job is naturally re-runnable: a second
  pass simply finds nothing new in the window. Keep it that way — don't make the job
  depend on a "last run" side effect that breaks if it runs twice or late
  ([writing-jobs.md](writing-jobs.md)).
- **Purge the whole footprint.** Expiring a record means its **attachments**
  (Active Storage blobs — `../rails-storage/`), **search index** entries
  ([search.md](../../rails-models/references/search.md)), and any **derived copies** go too. A row deleted
  from one table but lingering in a blob store isn't purged.
- **Built-in cleanups you already have.** Solid Queue prunes its own finished job
  rows, and Rails' session/cache stores have their own expiry — don't hand-roll those.
  Check the Solid Queue config for finished-execution retention before writing a job
  to clean its tables ([solid-queue.md](solid-queue.md)).
- **The audit trail is data too.** If you run [audit-trail.md](../../rails-models/references/audit-trail.md),
  its rows have their *own* retention window — and may need to outlive the records
  they describe (that's often the point). Set its window deliberately; don't let a
  blanket purge erase the log you keep for forensics.

## Common Pitfalls

- **Unbounded `delete_all` on a huge table** → long lock, replication lag, bloated
  WAL. Batch it.
- **`delete_all` orphaning child PII** → the parent's gone, the personal data in its
  children remains. Cascade explicitly or use the model's erase path.
- **Hard-coding a retention number in the skill/job without confirming the policy** →
  you've made a legal decision in code. Declare it as a named constant/config the
  team sets, and surface it for confirmation.
- **Purging under legal hold** → some records must be retained despite the window
  (litigation, tax). Exclude held records from the `expired` scope.
- **Overlapping long purges** → a daily purge that runs >24h doubles up. Add a
  concurrency limit/lock (Solid Queue concurrency controls / [scheduling-recurring.md](scheduling-recurring.md)).
- **Deleting the row but not its blobs/index** → partial purge that looks complete.
- **Timezone drift** → "3am" purge runs midday; set the app TZ (scheduling-recurring.md).

## Verify

```bash
# the retention scope selects only past-window rows
bin/rails runner 'p Audit.expired.where("created_at >= ?", Audit::RETENTION.ago).count'  # => 0 (nothing in-window is "expired")

# the purge job is enqueuable and idempotent (second run is a no-op, no error)
bin/rails runner 'p RetentionPurgeJob.respond_to?(:perform_later)'                        # => true
bin/rails runner 'RetentionPurgeJob.perform_now; RetentionPurgeJob.perform_now; puts "idempotent re-run ok"'

# expired rows actually go; in-window rows survive
bin/rails runner 'old=Audit.create!(created_at: 5.years.ago, action:"x", auditable: User.first); RetentionPurgeJob.perform_now; p Audit.exists?(old.id)'  # => false

# the recurring entry is wired (Solid Queue)
test -f config/recurring.yml && grep -q RetentionPurgeJob config/recurring.yml && echo "scheduled ok"
```

Each step proves an *effect* — the scope is bounded correctly, the job is
re-runnable, expired data is gone while in-window data survives, and the schedule is
registered. Record the per-model retention windows (and where they're set) in
`STACK.md` so the next change keeps them deliberate.

# Writing Jobs (Active Job patterns)

Backend-agnostic job code: enqueue, arguments, retries vs discards, and the rule
that matters most — **jobs must be idempotent and retry-safe**. Written against the
Active Job API so the backend ([solid-queue.md](solid-queue.md) /
[backend-alternatives.md](backend-alternatives.md)) stays swappable.

## Quick Pattern

```ruby
# app/jobs/charge_payment_job.rb
class ChargePaymentJob < ApplicationJob
  queue_as :default

  retry_on Stripe::RateLimitError, wait: :polynomially_longer, attempts: 5
  discard_on ActiveJob::DeserializationError   # the record was deleted; nothing to do

  def perform(order_id)
    order = Order.find_by(id: order_id) or return   # idempotent: gone → no-op
    return if order.paid?                            # idempotent: already done → no-op
    order.charge!
  end
end
```
```ruby
ChargePaymentJob.perform_later(order.id)               # enqueue
ChargePaymentJob.set(wait: 5.minutes).perform_later(order.id)   # delayed
ChargePaymentJob.set(priority: 10, queue: :low).perform_later(order.id)
```

## Pass IDs, not objects (GlobalID)

```ruby
SomeJob.perform_later(user)        # works — Active Job serializes via GlobalID
SomeJob.perform_later(user.id)     # preferred — explicit, smaller, survives deletes cleanly
```

Active Job serializes AR objects with GlobalID and re-loads them on perform; if the
record is **deleted** before the job runs, that raises `DeserializationError`. Passing
the **id** and `find_by` (returning on nil) makes deletion a harmless no-op. Only
pass primitive, serializable arguments (ids, strings, numbers, hashes) — not
procs/complex objects.

## Retries & discards

- **`retry_on ErrorClass, wait:, attempts:`** — transient failures (rate limits,
  network). `wait: :polynomially_longer` backs off. After `attempts`, it re-raises
  (or runs the optional block).
- **`discard_on ErrorClass`** — permanent failures where retrying is pointless
  (record gone, invalid input). Drops the job without erroring.
- **Default:** unhandled exceptions bubble to the backend's retry/dead-set. Decide
  per error class rather than letting everything retry forever.

## Idempotency (the load-bearing rule)

A job can run **more than once** (retries, at-least-once delivery, crashes). Design
every job so a second run is safe:

- Check-then-act guards (`return if order.paid?`).
- Use unique constraints / upserts for "create once" work (`../rails-models/`).
- Avoid side effects that double up (don't send two emails) — guard with state or a
  dedupe key.

## Deep Dive

- **`perform_later` vs `perform_now`:** `_later` enqueues; `_now` runs inline (useful
  in tests or when already in a worker). In a request, always `_later`.
- **Callbacks:** `before_perform`/`after_perform`/`around_perform` for cross-cutting
  concerns (instrumentation, tenant scoping).
- **`ActiveJob::Base.queue_adapter = :inline`/`:test`** — inline runs synchronously;
  test adapter records enqueues for assertions (`../rails-testing/`).
- **Don't enqueue inside a DB transaction** before commit — the worker may pick it up
  before the row is visible. Enqueue `after_commit` (or use the transactional
  integrity your backend offers).
- **Mailers:** `SomeMailer.welcome(user).deliver_later` enqueues a mailer job — the
  mail itself is `../rails-mailers/`.

## Common Pitfalls

- **Non-idempotent jobs** → retries cause double charges/emails; guard every side
  effect.
- **Passing whole objects + record deleted** → `DeserializationError`; pass ids +
  `find_by ... or return`, or `discard_on` it.
- **Enqueuing before commit** → worker runs before the data exists; enqueue
  `after_commit`.
- **`retry_on` everything forever** → poison jobs loop; `discard_on` permanent
  errors and cap `attempts`.
- **Huge arguments** (serializing a big payload) → bloated queue; pass an id and load
  it in `perform`.

## Verify

```bash
# Test adapter records the enqueue (no worker needed):
bin/rails runner "ActiveJob::Base.queue_adapter = :test; ChargePaymentJob.perform_later(1); p ActiveJob::Base.queue_adapter.enqueued_jobs.size"   # => 1
# Idempotency smoke: running perform twice has the same end state (inline):
bin/rails runner "ActiveJob::Base.queue_adapter=:inline; 2.times { ChargePaymentJob.perform_later(Order.first&.id) }; p 'ran twice safely'" 2>/dev/null
# Deleted-record path is a no-op, not a crash:
bin/rails runner "ChargePaymentJob.new.perform(999999); p 'no-op on missing record'" 2>/dev/null
```

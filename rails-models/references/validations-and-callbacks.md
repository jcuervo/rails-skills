# Validations, Constraints, Normalization & Callbacks

The model's correctness layer: validations for user-facing messages, DB
constraints for integrity, `normalizes` for clean input, concerns for shared
behavior — and a clear-eyed take on **when not to use callbacks**.

## Quick Pattern

```ruby
class User < ApplicationRecord
  normalizes :email, with: ->(e) { e.strip.downcase }   # Rails 7.1+

  validates :email, presence: true, uniqueness: true,
                    format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :age, numericality: { greater_than_or_equal_to: 0 }, allow_nil: true
end
```

Back the uniqueness validation with a DB unique index, and presence with
`NOT NULL` (see [migrations.md](migrations.md)) — validations alone race under
concurrency.

## Validations vs DB constraints — use both

| Concern | Model validation | DB constraint |
|---|---|---|
| Friendly error on a form/API | ✅ | — |
| Survives concurrent inserts | ❌ (race) | ✅ unique index |
| Survives raw SQL / other apps | ❌ | ✅ `NOT NULL`, FK, `CHECK` |
| Conditional / cross-record logic | ✅ (`if:`, custom) | hard/awkward |

Validations are the UX layer; constraints are the integrity layer. They are not
redundant — they cover different failure modes.

## `normalizes` (prefer over a `before_validation` callback)

`normalizes :attr, with: ->(v) { ... }` runs on assignment and before validation,
and also normalizes values used in finders (`User.find_by(email: " A@B.com ")`).
Reach for it instead of a `before_validation` callback for input cleanup.

## Concerns (shared model behavior)

```ruby
# app/models/concerns/sluggable.rb
module Sluggable
  extend ActiveSupport::Concern
  included do
    before_validation :set_slug, on: :create
    validates :slug, presence: true, uniqueness: true
  end
  private
  def set_slug = self.slug ||= title.to_s.parameterize
end
```

Use concerns to share *cohesive* behavior across models. Don't use them as a
junk drawer — if a concern only one model includes, it's just indirection.

## When NOT to use callbacks

Callbacks are fine for logic *intrinsic to persisting this record*. They become a
liability when they:

- **Touch other records** (`after_save` that updates a sibling) → hidden coupling;
  move to an explicit service or a job (`../rails-jobs/`).
- **Call external services** (send email, hit an API) inside the save transaction
  → a slow/failing third party rolls back your write. Enqueue a job instead.
- **Encode business workflow** → untestable in isolation, fire on every path
  including seeds/imports. Prefer an explicit method you call deliberately.

Order of preference: `normalizes` / value object / explicit method or service →
*then* a callback, only if the behavior truly belongs to persistence.

## Common Pitfalls

- **Uniqueness validation without a unique index** → duplicate rows under load.
- **`before_save` that calls a mailer/HTTP API** → coupling + rollback risk;
  enqueue post-commit (`after_commit on: :create`) or a job.
- **`validate` (instance) vs `validates` (declarative)** confusion → custom checks
  use `validate :method_name`.
- **Skipping validations silently** (`update_column`, `update_all`, `save(validate:
  false)`) → corrupt data; only the DB constraint will catch it. Another reason to
  add constraints.
- **`after_commit` callbacks not firing in tests** when using a transactional
  fixture strategy → coordinate with `../rails-testing/SKILL.md`.

## Verify

```bash
bin/rails runner "u=User.new(email:' A@B.COM '); u.valid?; p u.email"          # normalized => 'a@b.com'
bin/rails runner "u=User.new; u.valid?; p u.errors.details"                     # presence/format fire
bin/rails runner "User.create!(email:'x@y.com'); p (User.new(email:'x@y.com').tap(&:valid?).errors[:email])"  # uniqueness
bin/rails runner "p User.create(email:'x@y.com').persisted? rescue p \$!.class"  # DB unique index also rejects dupes
```

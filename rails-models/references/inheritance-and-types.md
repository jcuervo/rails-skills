# Type & Inheritance Modeling

The menu deep dive for "I have variants of a thing." Four shapes:
**plain model**, **delegated types**, **STI**, **polymorphic**. Pick by how much
the variants' *columns* diverge — not by instinct.

## Decision in one line each

| Shape | Use when | Storage |
|---|---|---|
| **Plain model** *(Recommended default)* | No real variance yet | One model, one table |
| Delegated types | Variants share some columns but **diverge** in others | Parent table + one table per child type |
| STI | Variants share **almost all** columns; behavior differs, schema barely | One table + a `type` column |
| Polymorphic | "Belongs to **any of** several parents" (not inheritance) | `*_type` + `*_id` on the child |

Don't model inheritance before two real variants exist. A `kind`/`category`
enum column ([enums-and-scopes.md](enums-and-scopes.md)) covers many "types"
without any of the below.

## Plain model

The default. One class, one table. Add an enum for lightweight "kinds." Reach for
the others only when columns genuinely diverge or a single association must span
types.

## Delegated types (modern Rails answer for diverging variants)

```ruby
# Parent holds shared columns; `entryable` delegates to the concrete type.
class Entry < ApplicationRecord
  delegated_type :entryable, types: %w[Message Comment], dependent: :destroy
end
module Entryable
  extend ActiveSupport::Concern
  included { has_one :entry, as: :entryable, touch: true }
end
class Message < ApplicationRecord; include Entryable; end   # message-only columns
class Comment < ApplicationRecord; include Entryable; end   # comment-only columns
```

- **Pick when** variants share a meaningful set of columns (timestamps,
  account_id, common scopes) *and* each has its own distinct columns. No wide
  table of mostly-NULLs.
- **Don't pick when** there's no shared parent behavior (then they're just two
  models) or variants are nearly identical (then STI is simpler).
- Migration: a parent table (`entries`: `entryable_type`, `entryable_id`, shared
  cols) + one table per child.

## Single Table Inheritance (STI)

```ruby
class Notification < ApplicationRecord; end           # needs a string `type` column
class EmailNotification < Notification; end
class SmsNotification   < Notification; end
```

- **Pick when** variants share nearly all columns and differ mostly in behavior
  (methods, validations, defaults).
- **Don't pick when** each variant needs its own columns → you get a wide table
  full of NULLs and contention. Use delegated types instead.
- Add `add_column :notifications, :type, :string` and index it. `Notification.all`
  returns mixed subclasses; `EmailNotification.all` scopes by `type`.

## Polymorphic association (not inheritance)

```ruby
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true   # commentable_type + commentable_id
end
class Post  < ApplicationRecord; has_many :comments, as: :commentable; end
class Photo < ApplicationRecord; has_many :comments, as: :commentable; end
```

- **Pick when** one child type belongs to **any of several** parent types.
- **Trade-off:** the polymorphic column pair can't have a single FK constraint —
  integrity is weaker. Index `[commentable_type, commentable_id]` together.

## Common Pitfalls

- **STI wide-table syndrome** — every subclass's columns live in one table; lots
  of NULLs, lock contention. Switch to delegated types.
- **Polymorphic + no composite index** → slow lookups; always
  `add_index :comments, [:commentable_type, :commentable_id]`.
- **Reaching for STI/delegated types with one variant** → premature; use an enum.
- **`type` column collision** — STI reserves `type`; if you need a non-STI column
  named type, set `self.inheritance_column = :something_else`.

## Verify

```bash
# Delegated types:
bin/rails runner "e=Entry.create!(entryable: Message.new); p [e.entryable_type, e.message?]"
# STI:
bin/rails runner "p EmailNotification.create!.type; p Notification.count"
# Polymorphic:
bin/rails runner "p Post.create!.comments.create!(commentable_type:'Post')&.commentable_type"
bin/rails runner "p ActiveRecord::Base.connection.indexes(:comments).map(&:columns)"  # composite index present
```

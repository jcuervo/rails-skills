# Associations

How models relate: `belongs_to` / `has_many` / `has_many :through` /
`has_one` / polymorphic, plus the integrity and performance knobs
(`dependent:`, foreign keys, `counter_cache`, `inverse_of`).

## Quick Pattern

```ruby
class Author < ApplicationRecord
  has_many :posts, dependent: :destroy
  has_many :comments, through: :posts
end

class Post < ApplicationRecord
  belongs_to :author, counter_cache: true   # maintains authors.posts_count
  has_many :comments, dependent: :destroy
end
```

Back every `belongs_to` with a DB foreign key + index (see
[migrations.md](migrations.md)):

```ruby
add_reference :posts, :author, foreign_key: true, index: true, null: false
```

## When to pick which

- **`belongs_to` / `has_many`** — the ordinary one-to-many. `belongs_to` is
  **required by default** in modern Rails (presence-validated); pass
  `optional: true` only when the FK genuinely may be null.
- **`has_many :through`** — many-to-many *with* a real join model that carries its
  own data/behavior (e.g. `Enrollment` between `Student` and `Course`). Prefer
  this over `has_and_belongs_to_many`, which has no join model and no place to
  grow.
- **`has_one :through`** — a single related record reached via an intermediary.
- **Polymorphic** — one association that can point at multiple model types; see
  [inheritance-and-types.md](inheritance-and-types.md).

## Deep Dive

- **`dependent:`** decides what happens to children when the parent is deleted:
  `:destroy` (run callbacks, slow, safe), `:delete_all` (one SQL statement, skips
  callbacks), `:nullify`, `:restrict_with_error`. Match it to a DB-level
  `on_delete` only if you bypass ActiveRecord; otherwise `dependent:` + a plain FK
  is the norm.
- **`counter_cache: true`** stores `posts_count` on the parent so `author.posts.size`
  is one column read, not a `COUNT(*)`. Add the integer column
  (`add_column :authors, :posts_count, :integer, default: 0, null: false`) and
  backfill. Use `counter_cache: true` on the `belongs_to` side.
- **`inverse_of`** lets Rails connect both sides of an association in memory,
  avoiding a reload. Rails infers it for standard names; set it explicitly when
  you use custom `class_name`/`foreign_key`.
- **N+1** is a query concern: load associations with `includes`/`preload`/`eager_load`.
  Detection and tuning live in `../rails-performance/SKILL.md` — don't duplicate it
  here; just always reach for `includes` when you render a collection's children.

## Common Pitfalls

- **`dependent: :destroy` on a huge collection** runs one DELETE per child +
  callbacks → slow/timeouts. Use `:delete_all` when no child callbacks matter, or
  a background job (`../rails-jobs/`).
- **Missing FK/index behind a `belongs_to`** → orphaned rows and slow joins.
- **`has_and_belongs_to_many`** for anything that will ever need attributes on the
  relationship → migrate to `has_many :through` instead.
- **Forgetting `optional: true`** on a legitimately-nullable `belongs_to` → spurious
  validation failures after the required-by-default change.
- **Counter cache drift** if rows are created via raw SQL bypassing callbacks →
  reconcile with `Author.reset_counters(id, :posts)`.

## Verify

```bash
bin/rails runner "a=Author.create!(name:'x'); a.posts.create!(title:'t'); p a.reload.posts_count"  # => 1 if counter_cache wired
bin/rails runner "p Post.reflect_on_association(:author).options"   # shows counter_cache/optional/etc.
bin/rails runner "p Author.includes(:posts).first&.posts&.loaded?"  # eager-load works
bin/rails runner "p ActiveRecord::Base.connection.foreign_keys(:posts).map(&:to_table)"  # FK exists
```

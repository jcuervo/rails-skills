# Migrations, Indexes & Schema Format

Owns how the database schema *changes over time*: writing reversible migrations,
indexing, DB-level constraints, and the `schema.rb` vs `structure.sql` choice.
This is a pattern reference, not a single pick — but every section ends in
runnable Verify.

## Quick Pattern (generate + run)

```bash
# Generate from a name Rails understands — it infers the change:
bin/rails g migration AddPublishedAtToPosts published_at:datetime:index
bin/rails g migration CreatePosts title:string body:text author:references
bin/rails g model Post title:string body:text   # model + migration + test together

bin/rails db:migrate                              # apply
bin/rails db:rollback STEP=1                       # undo the last migration
bin/rails db:migrate:status                        # see up/down state
```

A generated migration with reversible operations (`create_table`, `add_column`,
`add_index`, `add_reference`) needs no `down` — Rails reverses it automatically.

## Writing migrations that reverse cleanly

```ruby
class AddSlugToPosts < ActiveRecord::Migration[8.1]
  def change
    add_column :posts, :slug, :string
    add_index  :posts, :slug, unique: true
  end
end
```

- Use `change` with reversible operations. For anything Rails can't auto-reverse
  (raw SQL, data backfills), write explicit `up`/`down`, or wrap the
  irreversible part in `reversible { |dir| dir.up { ... }; dir.down { ... } }`.
- Use `up`/`down` (not `change`) when a migration both alters schema **and**
  backfills data, so the rollback is unambiguous.

## Indexes & DB constraints (integrity that survives concurrency)

Model validations are for messages; the database is for truth. Add both.

```ruby
add_index   :users, :email, unique: true              # enforce uniqueness in the DB
add_check_constraint :products, "price >= 0", name: "price_non_negative"
add_foreign_key :posts, :users                        # referential integrity
change_column_null :posts, :title, false              # NOT NULL
```

- `belongs_to` is **required by default** (validates presence). Add a matching
  `NOT NULL` + foreign key in the migration so the DB agrees.
- Index every foreign key and every column you filter/sort on. `add_reference
  ..., index: true` (default) and `foreign_key: true` together.
- A uniqueness *validation* without a unique *index* races under load — two
  requests can both pass validation and both insert. Always back it with an index.

## Zero-downtime / large tables

- Adding an index on a big Postgres table locks writes — use
  `add_index :t, :c, algorithm: :concurrently` and `disable_ddl_transaction!` in
  that migration.
- Adding a `NOT NULL` column with a default on a huge table can rewrite it on
  older engines; add the column, backfill in batches, then add the constraint.
- Consider the `strong_migrations` gem to catch these at author time — flag it as
  an option; it is not in the Rails default stack. Cross-link
  `../rails-performance/SKILL.md` for query-side concerns.

## Schema format → `schema.rb` (default) vs `structure.sql`

- **`schema.rb`** (Ruby, adapter-agnostic, diff-friendly) is the default. Keep it
  until a migration uses something it can't express.
- Switch **once, app-wide** when you adopt DB-native features `schema.rb` drops:
  Postgres extensions, triggers, generated/stored columns, partial-index
  expressions, custom types, `CHECK` with expressions.

```ruby
# config/application.rb
config.active_record.schema_format = :sql   # then: rm db/schema.rb; bin/rails db:prepare
```

After switching, `db/structure.sql` is the source of truth — commit it; CI loads
it with `db:test:prepare`.

## Common Pitfalls

- **Editing a migration that already ran** in shared environments → schema drift.
  Write a new migration instead; only edit un-merged, un-run ones.
- **`change` with raw `execute`** that isn't reversible → `db:rollback` blows up.
  Use `up`/`down` or `reversible`.
- **Uniqueness validation without a DB unique index** → duplicate rows under
  concurrency (see above).
- **Forgetting `bin/rails db:prepare`/`db:migrate` in another env** → schema file
  and DB disagree; commit the regenerated `schema.rb`/`structure.sql`.
- **Mismatched migration version bracket** (`Migration[8.1]`) vs installed Rails →
  confirm with `bin/rails -v`; the generator stamps the right one.

## Verify

```bash
bin/rails db:migrate                                  # applies cleanly
bin/rails db:rollback && bin/rails db:migrate          # down then up both succeed
bin/rails db:migrate:status | tail -5                  # all target migrations "up"
bin/rails runner "p ActiveRecord::Base.connection.indexes(:posts).map(&:columns)"  # index present
git diff --stat db/schema.rb db/structure.sql          # schema file regenerated + committed
```

The rollback→migrate round-trip is the real test: a migration that can't reverse
is a migration that will strand a teammate.

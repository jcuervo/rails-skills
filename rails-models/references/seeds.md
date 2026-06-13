# Seeds & Sample Data

`db/seeds.rb` defines the baseline data an app needs to run (reference data,
an admin user, lookup tables) and, optionally, sample data for development.
Run with `bin/rails db:seed` (and automatically as part of `bin/rails db:setup`
and `db:prepare` on a fresh DB).

## Quick Pattern (idempotent seeds)

```ruby
# db/seeds.rb — safe to run repeatedly
Category::DEFAULTS.each do |name|
  Category.find_or_create_by!(name: name)
end

admin = User.find_or_create_by!(email: "admin@example.com") do |u|
  u.password = ENV.fetch("ADMIN_PASSWORD", "change-me")
  u.admin = true
end
puts "Seeded #{Category.count} categories, admin=#{admin.email}"
```

The rule: **seeds must be idempotent** — `find_or_create_by!`, not `create!`.
Re-running `db:seed` should converge, not duplicate. `db:prepare` may run seeds on
a freshly-created DB, so they cannot assume an empty table.

## Reference data vs sample data

- **Reference/baseline data** (statuses, plans, an initial admin) belongs in
  `db/seeds.rb` and is part of every environment's setup.
- **Sample/demo data** (hundreds of fake posts) is dev-only. Guard it:

```ruby
if Rails.env.development?
  100.times { |i| Post.create!(title: "Sample #{i}", author: admin) }
end
```

For large/structured sample sets, split files under `db/seeds/` and load them
from `seeds.rb`, or define your own rake task (e.g. a `db:seed:development` task
you add — Rails ships only `db:seed`, not an environment-specific variant).

## Relationship to test data

Seeds are **not** your test fixtures/factories. Test data (fixtures, FactoryBot)
is owned by `../rails-testing/SKILL.md` — don't seed the test DB from
`seeds.rb`. Keep the two separate: seeds boot a usable app; factories build
objects for examples.

## Faker & volume

`faker` (dev/test group) generates realistic sample values. For big volumes,
batch with `insert_all`/`upsert_all` (skips validations/callbacks — only for
trusted generated data) instead of thousands of `create!` calls.

## Common Pitfalls

- **Non-idempotent seeds** (`create!`) → duplicate rows on the second run, or a
  uniqueness error that aborts `db:prepare`.
- **Seeds depending on data they don't create** → order matters; create parents
  before children, or use `find_or_create_by!` throughout.
- **Secrets hard-coded in seeds** → pull the admin password from ENV/credentials
  (`../rails-security/`), never commit a real one.
- **Slow seeds from per-row `create!`** → use `insert_all` for bulk generated rows.
- **Sample data leaking into production** → guard demo blocks with
  `Rails.env.development?`.

## Verify

```bash
bin/rails db:seed                     # runs clean
bin/rails db:seed                     # second run: no duplicates, no errors (idempotent)
bin/rails runner "p [Category.count, User.where(admin: true).count]"   # expected baseline present
```

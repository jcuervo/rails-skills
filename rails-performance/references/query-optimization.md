# Query Optimization & Profiling

Make the database do less work. Indexes, narrow selects, counts via counter caches,
read replicas, and `EXPLAIN` — plus how to **measure** so you optimize the real
bottleneck. The migration/model *implementation* is `../rails-models/`; this is the
performance lens and how to find what to change.

## Measure first (rack-mini-profiler + query counts)

```ruby
# Gemfile (:development): gem "rack-mini-profiler"   (+ optional "memory_profiler", "stackprof")
```
- `rack-mini-profiler` shows a per-request speed badge with the SQL breakdown — the
  fastest way to see *which* queries dominate a page.
- For a scripted count, subscribe to `sql.active_record` (see Verify) or read the dev
  log. **Optimize the measured bottleneck**, not the one you assume.

## Indexes (the biggest single win)

- Index every column you **filter** (`where`), **sort** (`order`), or **join**/FK on.
  A missing index turns a lookup into a full table scan.
- **Composite indexes** for multi-column filters (`[account_id, created_at]`), ordered
  by selectivity; the order matters.
- **Partial/unique indexes** for conditional/uniqueness constraints.
- Writing them is `../rails-models/`
  [migrations.md](../../rails-models/references/migrations.md); decide *what* to index
  from the query patterns and `EXPLAIN`.

```sql
-- See whether a query uses an index or scans:
EXPLAIN ANALYZE SELECT * FROM posts WHERE author_id = 1 ORDER BY created_at DESC;
```
```ruby
bin/rails runner "puts Post.where(author_id: 1).order(created_at: :desc).explain"
```

## Load less data

```ruby
Post.select(:id, :title)                 # only the columns you need (lighter objects)
Post.pluck(:email)                       # array of values, no AR objects at all
Post.where(...).exists?                  # existence check without loading rows
User.in_batches(of: 1000) { |batch| ... }# process big sets without loading all at once
```

- **`select`/`pluck`** avoid hydrating full rows when you need a few columns/values.
- **`exists?`** over `present?`/`any?` (which may load records) for a pure check.
- **`find_each`/`in_batches`** for large datasets so you don't load the whole table
  into memory.

## Counts & aggregates

- **`counter_cache`** stores `posts_count` on the parent so `author.posts.size` is one
  column read, not a `COUNT(*)` per call — removes a common N+1
  (`../rails-models/` associations.md).
- Database aggregates (`group`, `sum`, `count`) run in SQL — far faster than loading
  rows and summing in Ruby.

## Read replicas (read-heavy apps)

- Route reads to a replica to take load off the primary; Rails' automatic role
  switching + `connected_to(role: :reading)` is configured in
  `../rails-models/` [multi-database.md](../../rails-models/references/multi-database.md).
- Mind replication lag for read-after-write paths (the delay window / read from
  primary when you must).

## Common Pitfalls

- **Optimizing without measuring** → effort on the wrong query; profile first.
- **Missing index on a filtered/sorted/joined column** → full scans that get worse
  with data growth.
- **`SELECT *` everywhere** → loading/serializing columns you don't use; `select`/
  `pluck`.
- **`COUNT(*)` in a loop** → use `counter_cache` or a grouped aggregate.
- **Counting/summing in Ruby** what SQL can do → load + iterate is far slower than a
  `sum`/`group`.
- **Loading a whole table** to process it → `find_each`/`in_batches`.

## Verify

```bash
# EXPLAIN shows index usage (no full scan on an indexed filter):
bin/rails runner "puts Post.where(author_id: 1).order(created_at: :desc).limit(10).explain" 2>/dev/null
# select/pluck issue a narrower query:
bin/rails runner "puts Post.select(:id,:title).limit(5).to_sql; puts Post.pluck(:id).inspect" 2>/dev/null
# Query count for the optimized path (compare to the naive one):
bin/rails runner "n=0; ActiveSupport::Notifications.subscribe('sql.active_record'){|*| n+=1}; Author.first&.posts&.size; puts \"queries=#{n}\"" 2>/dev/null
# rack-mini-profiler present in dev:
bundle list | grep rack-mini-profiler || echo "(add rack-mini-profiler to profile requests)"
```

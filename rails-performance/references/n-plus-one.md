# N+1 Queries

The most common Rails performance bug: rendering a collection fires one query per
item for its associations. The menu pick is **Bullet** (dev-time detection);
`strict_loading` is the built-in, enforce-at-the-model alternative. **Fix N+1 before
caching** — caching a bad query just hides it.

## The bug + the fix (eager loading)

```ruby
# N+1: 1 query for posts + N queries for each post.author
Post.limit(10).each { |p| p.author.name }          # 11 queries

# Fixed: eager-load the association
Post.includes(:author).limit(10).each { |p| p.author.name }   # 2 queries
```

- **`includes`** — Rails decides between a separate preload query and a join.
- **`preload`** — forces separate queries (good when you don't filter on the
  association).
- **`eager_load`** — forces a `LEFT JOIN` (needed when you `where`/`order` on the
  association).
- Nested: `Post.includes(author: :company, comments: :user)`.

Association *implementation* (what associations exist, FKs, indexes) is
`../rails-models/` [associations.md](../../rails-models/references/associations.md);
this file is about *detecting and eliminating* the N+1.

## Bullet (Recommended — dev/test detection)

```ruby
# Gemfile (:development, :test): gem "bullet"
# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.bullet_logger = true
  Bullet.raise = true          # in TEST: fail the suite on an N+1
end
```

- Flags **N+1 queries**, **unused eager loading** (you `includes`'d something you
  didn't use), and **missing counter caches** — as you click around in dev, or as test
  failures.
- **Pick when** you want low-friction detection during development without changing
  model code. The common default.
- **Don't pick for production** — it's a dev/test tool. Combine with `strict_loading`
  to *enforce* in app code.

## `strict_loading` (built-in enforcement)

```ruby
# Per-association / per-record / per-relation:
Post.strict_loading.includes(:author)               # raises if you touch a non-eager-loaded assoc
class Post < ApplicationRecord
  self.strict_loading_by_default = true             # model-wide
end
# config/application.rb — app-wide default + mode:
config.active_record.strict_loading_by_default = true
config.active_record.action_on_strict_loading_violation = :raise   # or :log
```

- **Pick when** you want to **guarantee** no accidental lazy loads — a violation
  raises (or logs), forcing explicit eager loading. No gem.
- **Don't pick when** the team isn't ready for hard failures everywhere — start with
  `:log`, or enable per-model, then ratchet up.

## Deep Dive

- **Detect with a query count**, not vibes: subscribe to `sql.active_record` (see
  Verify) or read the dev log's "X queries" to prove the count dropped.
- **Don't over-eager-load:** `includes` you never use wastes memory/queries — Bullet
  flags this too.
- **Serializers/views walking associations** are a prime N+1 source — eager-load what
  you render (`../rails-api/` serialization.md, `../rails-hotwire/`).
- **`counter_cache`** removes `COUNT` N+1s for `.size` on collections
  (`../rails-models/` associations.md).

## Common Pitfalls

- **Caching over an N+1** → hides it until the cache misses, then it stampedes; fix
  the query first.
- **`includes` but still N+1** because you filtered on the association → use
  `eager_load` (forces the JOIN) so the condition can apply.
- **Over-eager-loading** unused associations → wasted queries/memory.
- **N+1 only under real data** → a 2-record dev DB hides it; test with representative
  volume.
- **`strict_loading` flipped to `:raise` app-wide cold** → surprise production errors;
  start `:log`/per-model.

## Verify

```bash
# Prove the query count drops with eager loading:
bin/rails runner "n=0; ActiveSupport::Notifications.subscribe('sql.active_record'){|*| n+=1}; Post.limit(5).each{|p| p.author&.name}; puts \"lazy=#{n}\"" 2>/dev/null
bin/rails runner "n=0; ActiveSupport::Notifications.subscribe('sql.active_record'){|*| n+=1}; Post.includes(:author).limit(5).each{|p| p.author&.name}; puts \"eager=#{n}\"" 2>/dev/null
# Bullet present (dev) / strict_loading configured:
bundle list | grep bullet || bin/rails runner "p Rails.application.config.active_record.strict_loading_by_default" 2>/dev/null
```

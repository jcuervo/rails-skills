# Multiple Databases, Replicas & Sharding

Rails has first-class multi-DB support: a primary + read replica, separate
databases per domain, and horizontal sharding. Reach for it when one database is
genuinely a bottleneck or a boundary — not by default.

## Where the Solid backends already live

Rails 8 omakase puts **Solid Queue / Cache / Cable** in their own databases by
default (see `config/database.yml` — `queue`, `cache`, `cable` entries pointing at
separate SQLite files or schemas). That *is* multi-database, configured for you.
Adding Sidekiq/Redis instead is `../rails-jobs/` / `../rails-performance/`. This
reference is about **your application data** spanning databases.

## Quick Pattern — primary + read replica

```yaml
# config/database.yml (production)
production:
  primary:
    <<: *default
    database: app_production
  primary_replica:
    <<: *default
    database: app_production
    replica: true
    host: replica.db.internal
```

```ruby
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to database: { writing: :primary, reading: :primary_replica }
end

# Route reads to the replica for a block:
ActiveRecord::Base.connected_to(role: :reading) { Post.count }
```

Enable automatic role switching (reads → replica, writes → primary, with a
delay window after writes) via the middleware:

```ruby
# config/environments/production.rb
config.active_record.database_selector = { delay: 2.seconds }
config.active_record.database_resolver = ActiveRecord::Middleware::DatabaseSelector::Resolver
config.active_record.database_resolver_context = ActiveRecord::Middleware::DatabaseSelector::Resolver::Session
```

## Separate databases per domain

```ruby
class AnalyticsRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to database: { writing: :analytics }
end
class Event < AnalyticsRecord; end   # lives in the analytics DB
```

Each additional database gets its own migrations path
(`config.paths["db/migrate"]` per connection) and its own `schema.rb`/
`structure.sql`. `bin/rails db:migrate` runs them all; target one with
`bin/rails db:migrate:primary` / `:analytics`.

## Horizontal sharding

```ruby
connects_to shards: {
  default:  { writing: :primary,  reading: :primary_replica },
  shard_two:{ writing: :primary_2, reading: :primary_2_replica }
}
ActiveRecord::Base.connected_to(shard: :shard_two) { Order.create!(...) }
```

Pick sharding only when a single primary can't hold the write volume/data — it's
a real operational commitment (cross-shard queries don't exist; you route).

## When to pick / not pick

- **Replica** — read-heavy app, reporting queries crowding out writes. Cheap win.
- **Separate DB** — a bounded domain (analytics, audit logs) with different
  scaling/retention, or a DB you don't want app failures to cascade into.
- **Sharding** — genuine write-throughput/data-size limits on one primary. Last
  resort; adds routing burden to every query.
- **Don't** split databases for organization alone — schemas/namespaces handle
  that without the operational cost.

## Common Pitfalls

- **Reading your own write off a stale replica** → use the `delay:` window or
  `connected_to(role: :writing)` for read-after-write paths.
- **Forgetting per-database migrations** → a new table lands in the wrong DB;
  set the connection's migrations path and run the namespaced migrate task.
- **Cross-database joins/FKs** → not possible across separate databases; model the
  boundary deliberately.
- **Transactions don't span databases** → a write to two DBs isn't atomic; design
  for it (outbox/jobs).

## Verify

```bash
bin/rails runner "p ActiveRecord::Base.configurations.configs_for(env_name: Rails.env).map(&:name)"  # lists configured DBs
bin/rails runner "ActiveRecord::Base.connected_to(role: :reading) { p Post.count }"   # replica read works
bin/rails -T 2>/dev/null | grep -E "db:migrate:(primary|analytics)"   # confirm the actual namespaced tasks Rails generated
bin/rails db:migrate:status 2>/dev/null | tail -5                     # migration state across configured DBs
```

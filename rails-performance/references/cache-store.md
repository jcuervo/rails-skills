# Cache Store

The backing store for all of Rails' caching (`Rails.cache`, fragment caching, and —
by default — `../rails-security/` rate limiting). The menu pick is **Solid Cache**
(Rails 8 default, DB-backed); Redis/Memcached are the in-memory alternatives.

## Solid Cache (Recommended — Rails 8 default)

```ruby
# config/environments/production.rb (set by the scaffold when Solid is kept)
config.cache_store = :solid_cache_store
```
```bash
bin/rails db:prepare      # loads db/cache_schema.rb into the cache database
```

- DB-backed (its own `cache` database — `../rails-models/`
  [multi-database.md](../../rails-models/references/multi-database.md)); **no Redis**.
  Because disk is cheap, it can hold a **much larger** cache than a RAM store, with
  encryption + automatic size-based expiry.
- **Pick when** you're on Rails 8 and want a big, durable cache with zero extra infra.
  The default.
- **Don't pick when** you need the absolute lowest latency for tiny hot keys, or you
  already operate Redis and want one store for cache + other uses.
- Latency is slightly higher than RAM but fine for fragment/low-level caching; the
  capacity tradeoff usually wins.

## Redis

```ruby
# Gemfile: gem "redis"   (and optionally "hiredis-client" for speed)
config.cache_store = :redis_cache_store, { url: ENV["REDIS_URL"] }
```

- **Pick when** you want lowest-latency in-memory caching, a store **shared across
  app hosts**, or you already run Redis (for Sidekiq, Action Cable, etc.) and want one
  system.
- **Don't pick when** you'd rather avoid operating Redis — Solid Cache covers most
  needs without it.
- Set a sensible `expires_in` and maxmemory eviction policy so it doesn't grow
  unbounded.

## Memcached (dalli)

```ruby
# Gemfile: gem "dalli"
config.cache_store = :mem_cache_store, ENV["MEMCACHE_SERVERS"]
```

- **Pick when** you want a simple, fast, purely-volatile cache and already run
  Memcached.
- **Don't pick when** you need durability or large capacity (it's RAM + volatile) —
  Solid Cache or Redis fit better.

## What the store backs

- `Rails.cache.fetch/read/write` (low-level) and **fragment caching**
  ([caching-strategies.md](caching-strategies.md)).
- **Rate limiting** (Rails 8 built-in) uses `config.action_controller.cache_store` →
  `config.cache_store` by default (`../rails-security/` rate-limiting.md) — so a
  configured store is a prerequisite there too.
- `:null_store` in test/dev when you want caching disabled; `:memory_store` for a
  small per-process dev cache.

## Common Pitfalls

- **`:null_store` left on in production** → every `fetch` recomputes; confirm the prod
  store is the real one.
- **No `expires_in` on volatile stores** (Redis/Memcached) → unbounded growth /
  eviction surprises; set TTLs.
- **Assuming Redis when it's Solid Cache** (or vice versa) → wrong ops mental model;
  detect the configured store first.
- **Per-process `:memory_store` across multiple app servers** → cache not shared;
  hits on one host miss on another. Use Solid Cache/Redis/Memcached for shared cache.
- **Forgetting the `cache` DB** (Solid Cache) → `db:prepare` must create it.

## Verify

```bash
bin/rails runner "p Rails.cache.class.name"                       # the configured store
bin/rails runner "Rails.cache.write('k', 42, expires_in: 1.minute); p Rails.cache.read('k')"   # => 42 round-trip
bin/rails runner "p Rails.cache.fetch('once'){ 7 }; p Rails.cache.fetch('once'){ raise 'miss!' }"  # => 7, 7 (2nd is a hit, no raise)
ls db/cache_schema.rb 2>/dev/null && echo "✓ Solid Cache schema present"
```

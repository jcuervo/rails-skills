---
name: rails-performance
description: Make a Rails 8.1 app fast — choose a cache store (Solid Cache, the Rails 8 default, or Redis / Memcached), detect and fix N+1 queries (Bullet in dev or built-in strict_loading), apply fragment / Russian-doll / collection / low-level / HTTP caching, and optimize queries with indexes, select/pluck, counter caches, and read replicas. Menu-driven for the genuine choices (cache store, N+1 detection) with a Recommended default; detects the configured cache store first, and verifies improvements by measuring (query counts, cache hits, timings) rather than guessing. Apply when an endpoint is slow, fixing N+1 queries, adding caching, tuning database queries, or setting up performance measurement.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[endpoint-or-concern]"
---

# rails-performance

## Purpose

Owns **caching and optimization**: the cache store, N+1 detection and eager loading,
fragment/Russian-doll/HTTP caching, and query-level tuning (indexes, `select`/`pluck`,
counter caches, replicas). The governing principle is **measure, don't guess** — every
change is justified by a query count, a cache-hit, or a timing, not a hunch. Applies to
API-only (query/HTTP/low-level caching) and full-stack apps (also fragment/view
caching). Query *implementation* lives in `../rails-models/`; this skill is the
performance lens and the caching layer on top.

## When to Apply

Use this skill when the task is:

- An endpoint/page is slow and you need to find and fix why
- Eliminating N+1 queries
- Adding caching (fragment, Russian-doll, collection, low-level, HTTP)
- Choosing/configuring a cache store
- Query tuning: indexes, `select`/`pluck`, counter caches, replicas, `EXPLAIN`
- Setting up measurement/profiling

Do **not** use this skill when the task is:

- Writing the migration/index itself, association/query *implementation* → read `../rails-models/SKILL.md` (this skill says *what* to index/eager-load; that one writes it)
- The job that offloads slow work → read `../rails-jobs/SKILL.md` (move slow work async)
- Background/real-time delivery performance → read `../rails-hotwire/SKILL.md`
- Serializer choice for API payloads → read `../rails-api/SKILL.md`
- Production scaling, CDN, Thruster, app-server tuning, APM → read `../rails-deploy/SKILL.md`
- Load/perf *testing* harness → read `../rails-testing/SKILL.md`
- A retrospective performance audit → use the upstream `rails-audit` skill

## Detect Before You Generate

```bash
grep -rnE "cache_store" config/environments/*.rb config/application.rb     # configured cache store
grep -E "^    (solid_cache|redis|dalli|bullet|rack-mini-profiler)" Gemfile.lock
grep -rn "fresh_when\|stale?\|Rails.cache\|<% cache\|strict_loading" app 2>/dev/null   # live caching/N+1 controls in use
ls config/cache.yml db/cache_schema.rb 2>/dev/null                         # Solid Cache present (Rails 8 default)
grep -nE "api_only" config/application.rb
```

- **Solid Cache is the Rails 8 default cache store** unless `--skip-solid`. If
  `config/cache.yml`/`db/cache_schema.rb` exist, it's already there — use it.
- If Redis/Memcached (dalli) is configured, match it — don't re-ask.
- **Measure first:** before optimizing, capture the current query count/timing for the
  slow path so you can prove the change helped.
- Honor any cache store recorded in `STACK.md`.

## Menu

Two genuine menus (caching strategies and query tuning are patterns, not pick-one).
Each via `AskUserQuestion`, Rails-default marked **Recommended**; ask unless configured.

### Cache store → `config.cache_store`
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Solid Cache** *(Recommended)* | Rails 8 default; DB-backed, no Redis, huge capacity on disk-cheap storage. Slightly higher latency than RAM. | [cache-store.md](references/cache-store.md) |
| Redis | In-memory, lowest latency, shared across hosts; one more service to run. Common when you already run Redis. | [cache-store.md](references/cache-store.md) |
| Memcached (dalli) | Classic in-memory cache; simple, fast, volatile. | [cache-store.md](references/cache-store.md) |

### N+1 detection
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Bullet (dev/test)** *(Recommended)* | Flags N+1s, unused eager loads, and missing counter caches while you develop. Dev-time only. | [n-plus-one.md](references/n-plus-one.md) |
| `strict_loading` (built-in) | Raises (or logs) when a lazy association loads; enforce no-N+1 at the model/app level. No gem. | [n-plus-one.md](references/n-plus-one.md) |

## Decision Flow

- **Measure before and after.** Capture query count + timing (logs,
  `rack-mini-profiler`, `ActiveSupport::Notifications`) for the slow path; optimize;
  re-measure. A change without a measured delta is a guess.
- **Cache store:** **Solid Cache** by default (Rails 8, no Redis, cheap large
  capacity). **Redis** when you need lowest latency / already run it / want shared
  ephemeral state. **Memcached** for a simple volatile cache. The store backs
  fragment, low-level, *and* rate-limit (`../rails-security/`) caching.
- **N+1 first, caching second.** Fix N+1s with eager loading
  ([n-plus-one.md](references/n-plus-one.md)) before reaching for caches — caching a
  bad query hides the problem. **Bullet** in development surfaces them; `strict_loading`
  enforces it.
- **Cache the expensive, stable thing.** Fragment/Russian-doll for view trees, HTTP
  conditional (`fresh_when`/etag) to skip rendering entirely, low-level
  `Rails.cache.fetch` for costly computations
  ([caching-strategies.md](references/caching-strategies.md)).
- **Query tuning:** index what you filter/sort/join on, `select`/`pluck` to avoid
  loading whole rows, counter caches for counts, replicas for read-heavy loads
  ([query-optimization.md](references/query-optimization.md)) — implemented in
  `../rails-models/`.

## Problem → Reference

| Task | Read |
|---|---|
| Choose/configure the cache store (Solid Cache / Redis / Memcached) | [references/cache-store.md](references/cache-store.md) |
| Detect + fix N+1 queries (Bullet / strict_loading / eager loading) | [references/n-plus-one.md](references/n-plus-one.md) |
| Fragment / Russian-doll / collection / low-level / HTTP caching | [references/caching-strategies.md](references/caching-strategies.md) |
| Query tuning: indexes, select/pluck, counter caches, replicas, EXPLAIN, profiling | [references/query-optimization.md](references/query-optimization.md) |

## Verify

A performance change is unfinished until a measurement shows it helped:

```bash
# Query count for a path drops after eager loading (the N+1 proof):
bin/rails runner "require 'benchmark'; n=0; ActiveSupport::Notifications.subscribe('sql.active_record'){ n+=1 }; Post.includes(:author).limit(10).each{|p| p.author.name }; puts \"queries=#{n}\"" 2>/dev/null
# Cache store is the expected one and round-trips:
bin/rails runner "p Rails.cache.class.name; Rails.cache.write('k',1); p Rails.cache.read('k')"   # => store name, then 1
# Cache hit on second render (fragment/low-level): warm once, then read is a hit
bin/rails runner "Rails.cache.fetch('expensive'){ 42 }; p Rails.cache.fetch('expensive'){ raise 'should not run' }"   # => 42 (no raise = hit)
# HTTP conditional GET returns 304 on the second request:
bin/rails server -d && sleep 3
ET=$(curl -s -D - -o /dev/null localhost:3000/posts/1 | awk 'tolower($1)=="etag:"{print $2}' | tr -d '\r'); \
  curl -s -o /dev/null -w "conditional=%{http_code}\n" -H "If-None-Match: $ET" localhost:3000/posts/1   # expect 304
kill "$(cat tmp/pids/server.pid)"
```

Then route to `../rails-models/` (write the indexes/eager loads), `../rails-jobs/`
(offload slow work), and `../rails-deploy/` (CDN, app-server tuning, APM). Record the
cache store in `STACK.md`.

# Caching Strategies

Once N+1s are fixed ([n-plus-one.md](n-plus-one.md)), cache the expensive, stable
work. Five layers, cheapest-to-skip first: **HTTP conditional** (skip rendering
entirely) → **fragment / Russian-doll** (cache view trees) → **collection** → **low-
level** (`Rails.cache.fetch`). All use the configured store
([cache-store.md](cache-store.md)).

## HTTP conditional caching (skip the work)

```ruby
def show
  @post = Post.find(params[:id])
  fresh_when @post                       # sets ETag + Last-Modified from the record
  # or: fresh_when etag: @post, last_modified: @post.updated_at, public: true
end
```

- If the client's `If-None-Match`/`If-Modified-Since` matches, Rails returns **304 Not
  Modified** with **no body and no view rendering** — the cheapest possible response.
- `stale?` is the imperative form when you need to branch. Great for APIs and
  show pages.
- Works for API and Web alike; the body isn't built on a 304.

## Fragment & Russian-doll caching (Web)

```erb
<% cache @post do %>                          <!-- key includes @post.cache_key_with_version -->
  <%= render @post %>
  <% cache @post.author do %>                 <!-- nested = Russian-doll -->
    <%= render @post.author %>
  <% end %>
<% end %>
```

- The cache key is the record's `cache_key_with_version` (id + `updated_at`), so an
  update **auto-expires** the fragment — no manual invalidation.
- **Russian-doll:** nest fragments so an inner change only busts the inner shells.
  `touch: true` on `belongs_to` bubbles a child update up to expire the parent
  (`../rails-models/` associations.md).
- **Pick fragment caching when** a view subtree is expensive and changes
  infrequently.

## Collection caching

```erb
<%= render partial: "posts/post", collection: @posts, cached: true %>
```

- Caches each item's fragment and fetches them in **one multi-read** — much faster
  than per-item `cache` blocks for lists. Requires the partial to be cache-friendly.

## Low-level caching (`Rails.cache.fetch`)

```ruby
@stats = Rails.cache.fetch("dashboard/stats/#{current_account.id}", expires_in: 10.minutes) do
  expensive_aggregate_query                  # only runs on a miss
end
```

- **Pick for** caching arbitrary expensive computations/queries (aggregations, API
  responses, rendered chunks) — not tied to a view.
- **Key design is everything:** include every input that changes the result
  (account id, filters, a version). A stale key serves wrong data.
- Use `race_condition_ttl` to avoid a stampede when a hot key expires.

## Deep Dive

- **Cache the stable + expensive.** Don't cache cheap or rapidly-changing data — the
  invalidation cost/staleness outweighs the win.
- **Invalidation is the hard part.** Prefer key-based expiry (`cache_key_with_version`,
  input-derived keys) over manual `delete` — keys that change on update are
  self-expiring.
- **Stampede protection:** `Rails.cache.fetch(..., race_condition_ttl:)` and/or compute
  in a job (`../rails-jobs/`) for very hot keys.
- **Don't cache personalized fragments globally** — scope keys by user/account or you
  leak one user's view to another.

## Common Pitfalls

- **Caching before fixing N+1** → the miss path stampedes the DB.
- **Bad cache keys** (missing an input) → stale/wrong data served; include all
  inputs + a version.
- **Caching per-user data under a shared key** → data leak across users; scope the key.
- **No expiry on volatile stores** → stale data lingers; set `expires_in` or use
  key-based expiry.
- **`fresh_when` after already rendering** → no savings; set it early in the action
  before building the body.

## Verify

```bash
# Low-level cache hit (second fetch must NOT recompute):
bin/rails runner "Rails.cache.fetch('k'){ 1 }; p Rails.cache.fetch('k'){ raise 'recomputed!' }"   # => 1 (no raise)
# HTTP conditional GET → 304 on the second request:
bin/rails server -d && sleep 3
ET=$(curl -s -D - -o /dev/null localhost:3000/posts/1 | awk 'tolower($1)=="etag:"{print $2}' | tr -d '\r')
curl -s -o /dev/null -w "conditional=%{http_code}\n" -H "If-None-Match: $ET" localhost:3000/posts/1   # expect 304
kill "$(cat tmp/pids/server.pid)"
```

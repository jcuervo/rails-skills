# Pagination

Bounding collection endpoints. The menu pick is **Pagy** for speed and low memory;
Kaminari if it's already in the app; cursor/keyset for large feeds.

> Pagy's public API has shifted across major versions (6 → 7 → 8 → 9). Detect the
> installed version (`grep pagy Gemfile.lock`) and confirm the exact method names
> (`pagy`, `pagy_metadata`, headers helpers) against that version's README before
> copying the snippets below.

## Pagy (Recommended)

```ruby
# Gemfile: gem "pagy"
# config/initializers/pagy.rb — require the extras you use (headers, metadata)
# Mix Pagy::Backend into the API base controller (api_only: ActionController::API;
# full-stack JSON endpoints inherit from ActionController::Base instead):
class Api::BaseController < ActionController::API
  include Pagy::Backend
end

def index
  pagy, posts = pagy(Post.all)            # method name/signature: confirm for your Pagy version
  pagy_headers_merge(pagy)                # Link + Current-Page/Total headers for API clients
  render json: posts
end
```

- **Pick when** you want the fastest, lowest-memory pagination; first-class API
  support via Link/metadata headers.
- **Don't pick when** the app already standardizes on Kaminari — consistency wins.
- Expose pagination as **response headers** (or a `meta` block) so clients get
  `total`, `current page`, and next/prev links without parsing the body.

## Kaminari (ubiquitous, scope-based)

```ruby
# Gemfile: gem "kaminari"
posts = Post.page(params[:page]).per(25)
render json: {
  data: posts,
  meta: { current_page: posts.current_page, total_pages: posts.total_pages, total: posts.total_count }
}
```

- **Pick when** it's already installed, or you want model scopes (`Post.page(2)`)
  and view helpers (full-stack) out of the box.
- **Don't pick when** you're starting fresh and care about memory/throughput — Pagy
  is lighter.

## Cursor / keyset (large or infinite feeds)

```ruby
# Pagy has a keyset extra; or hand-roll:
scope = Post.where("id < ?", params[:before]).order(id: :desc).limit(25)
render json: { data: scope, next_before: scope.last&.id }
```

- **Pick when** offset pagination drifts (rows inserted/deleted between pages) or
  slows on deep pages of a big table (`OFFSET 100000` scans).
- **Don't pick when** clients need to jump to arbitrary page numbers — keyset is
  next/prev only.
- Requires a stable, indexed sort key (usually `id` or `created_at,id`). Index it —
  see `../rails-models/` (migrations.md).

## Common Pitfalls

- **Unbounded `index`** (no pagination) → one request loads the whole table; always
  cap collection endpoints.
- **Offset pagination on huge tables** → deep pages get slow and can skip/repeat
  rows under concurrent writes; switch to keyset (`../rails-performance/`).
- **Pagination metadata only in the body** → clients must parse data to find the
  next page; emit headers too.
- **Copying Pagy snippets across a major version bump** → method/signature
  changes; verify against the installed version (note above).

## Verify

```bash
bin/rails server -d && sleep 3
# Page size is enforced and metadata/headers are present:
curl -fsS "localhost:3000/v1/posts?page=1" -D - -o /dev/null | grep -iE "current-page|total|link"
curl -fsS "localhost:3000/v1/posts?page=2" -o /tmp/p2.json -w "%{http_code}\n" && head -c 200 /tmp/p2.json
kill "$(cat tmp/pids/server.pid)"
```

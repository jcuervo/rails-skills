# Real-Time — Action Cable & Turbo Stream Broadcasting

The merged real-time topic, distinctly invocable. Live updates in Hotwire are
mostly **Turbo Stream broadcasting over WebSockets** — you rarely write raw Action
Cable. The menu pick is the **Cable adapter**: Solid Cable (Rails 8 default) vs
Redis vs async (dev only).

## Quick Pattern — broadcast model changes to all viewers

```erb
<!-- the page subscribes to a stream -->
<%= turbo_stream_from @post %>
<div id="comments"><%= render @post.comments %></div>
```
```ruby
class Comment < ApplicationRecord
  belongs_to :post
  # Broadcast a Turbo Stream to subscribers of `post` whenever a comment changes:
  broadcasts_to ->(comment) { comment.post }, inserts_by: :prepend, target: "comments"
end
```

> `broadcasts_to` / `broadcast_*_to` / `broadcast_*_later_to` and their options
> (`inserts_by:`, `target:`) are the `turbo-rails` model API — confirm the exact
> method and option names against the installed version
> (`bundle list | grep turbo-rails`) before relying on them.

When a comment is created, every connected client viewing that post gets the new
comment prepended — no controller code, no custom JS. For whole-page freshness,
`broadcasts_refreshes` + morphing ([turbo.md](turbo.md)).

## Broadcast from a job (don't block the request)

```ruby
# Heavy/async broadcasts belong in a job — ../rails-jobs/
class Comment < ApplicationRecord
  after_create_commit { broadcast_prepend_later_to post, target: "comments" }
end
```

`*_later_to` enqueues via Active Job so the WebSocket fan-out happens off the
request thread — pair with `../rails-jobs/` for the queue.

## Cable adapter menu → `config/cable.yml`

```yaml
# config/cable.yml — Rails 8 default
production:
  adapter: solid_cable
  # Redis alternative:
  # adapter: redis
  # url: <%= ENV["REDIS_URL"] %>
development:
  adapter: async      # in-process; dev/test only
```

- **Solid Cable** *(Recommended)* — DB-backed pub/sub, ships with Rails 8, no Redis
  to run/operate. Great for most apps; uses the `cable` database
  ([`../rails-models/references/multi-database.md`](../../rails-models/references/multi-database.md)
  explains where it lives).
- **Redis** — lowest latency and best at very high fan-out; one more service to run,
  secure, and monitor. Pick if you already run Redis or hit Solid Cable's ceiling.
- **Async** — in-process, single-process only. **Never production** (other workers
  won't receive broadcasts). Dev/test default.

## Deep Dive

- **`turbo_stream_from`** opens the WebSocket subscription and names the stream;
  `broadcasts_to`/`broadcast_*_to` publish to it. The stream name on both sides must
  match (model GID or string).
- **Raw Action Cable channels** (`bin/rails g channel`) are for non-Turbo custom
  protocols (presence, cursors, live metrics). Most apps never need them — Turbo
  Stream broadcasting covers "update this HTML for everyone."
- **Authorization:** `turbo_stream_from` streams are signed (clients can't subscribe
  to arbitrary streams), but a custom channel's `subscribed` must authorize the
  connection (`../rails-auth/`). Don't broadcast data a viewer shouldn't see.
- **Connection identity:** `ApplicationCable::Connection` `identified_by
  :current_user` ties sockets to users — auth mechanism is `../rails-auth/`.
- **Scaling:** WebSocket connections hold a server thread/process; Thruster/Puma
  tuning and standalone cable servers are `../rails-deploy/`.

## Common Pitfalls

- **`async` adapter in production** → broadcasts only reach the process that
  published them; other workers/clients see nothing. Use Solid Cable or Redis.
- **Stream name mismatch** between `turbo_stream_from` and `broadcasts_to` → silent
  no-op; no update arrives.
- **Broadcasting synchronously in the request** for heavy fan-out → slow responses;
  use `*_later_to` + a job (`../rails-jobs/`).
- **Broadcasting unauthorized data** → users receive HTML for records they can't
  see; scope what you broadcast and authorize channels.
- **Forgetting the `cable` DB is set up** (Solid Cable) → `db:prepare` must create
  it; check `config/database.yml`.

## Verify

```bash
grep -E "adapter:" config/cable.yml                      # adapter is solid_cable/redis (not async) for prod
bin/rails runner "p ActionCable.server.config.cable['adapter'] rescue p 'cable configured'"
bin/rails server -d && sleep 3
# Page opens a stream subscription:
curl -fsS localhost:3000/posts/1 -o /tmp/s.html && grep -q "turbo-cable-stream-source\|turbo_stream_from" /tmp/s.html && echo "✓ subscribes to a stream"
# Broadcasting a change enqueues/delivers (observe logs or a test):
bin/rails runner "c=Comment.last; c&.touch; p 'broadcast triggered'" 2>/dev/null
kill "$(cat tmp/pids/server.pid)"
```

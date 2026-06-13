# Turbo — Drive, Frames, Streams & Morphing

Turbo delivers SPA-like speed from server-rendered HTML, no custom JS. Four tools,
in escalating specificity: **Drive** (whole-page), **Frames** (a region),
**Streams** (surgical DOM ops), and **morphing page refreshes** (Turbo 8). Reach
for the least powerful one that does the job.

## Turbo Drive (free, on by default)

`turbo-rails` makes link clicks and form submits AJAX navigations that swap
`<body>` and merge `<head>` — no full reload. Mostly automatic. Opt a link out
with `data-turbo="false"`. This is why full-stack feels fast with zero effort.

## Turbo Frames — scope navigation to a region

```erb
<%= turbo_frame_tag dom_id(@post) do %>
  <%= link_to "Edit", edit_post_path(@post) %>   <!-- navigates INSIDE this frame -->
<% end %>
```

```erb
<!-- edit.html.erb must render a matching frame id -->
<%= turbo_frame_tag dom_id(@post) do %>
  <%= form_with model: @post do |f| %> ... <% end %>
<% end %>
```

- **Pick when** a link/form should update just one part of the page (inline edit,
  a tab, lazy-loaded content via `src:`/`loading: :lazy`).
- The response must contain a frame with the **same id**, or Turbo errors with
  "content missing." Frame ids are the contract.

## Turbo Streams — surgical, multi-target DOM updates

```ruby
# controller — respond with a .turbo_stream format
respond_to do |format|
  format.turbo_stream   # renders create.turbo_stream.erb
  format.html { redirect_to @post }
end
```
```erb
<!-- create.turbo_stream.erb -->
<%= turbo_stream.prepend "posts", @post %>
<%= turbo_stream.update "posts_count", Post.count %>
<%= turbo_stream.replace dom_id(@post), partial: "posts/post", locals: { post: @post } %>
```

Actions: `append`, `prepend`, `replace`, `update`, `remove`, `before`, `after`.

- **Pick when** one action changes several places (add the row *and* bump the
  counter) or you need precise insert/remove.
- Streams over WebSocket (broadcasting) are [real-time.md](real-time.md).

## Turbo 8 morphing page refreshes

```erb
<!-- in <head> (app/views/layouts/application.html.erb) -->
<%= turbo_refreshes_with method: :morph, scroll: :preserve %>
```
```ruby
# model — broadcast a page refresh to all viewers on change
class Post < ApplicationRecord
  broadcasts_refreshes
end
```

- Morphing diffs the new HTML into the existing DOM (via idiomorph), preserving
  scroll position and focus — instead of replacing the whole `<body>`.
- **Pick when** a "re-render the page" update should feel seamless, or you want
  broadcast-driven full-page freshness without hand-writing per-element streams.
- Verify the exact `turbo_refreshes_with` / `broadcasts_refreshes` API against the
  installed `turbo-rails` version — these are Turbo 8 features; detect with
  `bundle list | grep turbo-rails`.

## Common Pitfalls

- **Frame id mismatch** between the trigger frame and the response frame → "content
  missing" / nothing updates. Use `dom_id(record)` on both sides.
- **Returning 200 on an invalid form** → Turbo won't re-render the errored form;
  render with `status: :unprocessable_entity` ([forms-and-responses.md](forms-and-responses.md)).
- **Reaching for Streams when a Frame suffices** → more complexity than needed; use
  the least-powerful tool.
- **Forgetting `format.html` fallback** in `respond_to` → non-Turbo requests (or
  tests) get no response.
- **Heavy logic in `.turbo_stream.erb`** → keep it to render calls; logic belongs in
  the model/controller.

## Verify

```bash
bin/rails server -d && sleep 3
# Frame present on the page:
curl -fsS localhost:3000/posts -o /tmp/i.html && grep -qE "turbo-frame" /tmp/i.html && echo "✓ frame rendered"
# Turbo Stream response on create (TURBO_STREAM accept):
curl -s -X POST localhost:3000/posts -H "Accept: text/vnd.turbo-stream.html" \
  -d "post[title]=t&post[body]=b" | grep -q "turbo-stream" && echo "✓ stream response"
# Morphing meta in the layout:
grep -rq "turbo_refreshes_with" app/views/layouts/ && echo "✓ morphing configured"
kill "$(cat tmp/pids/server.pid)"
```

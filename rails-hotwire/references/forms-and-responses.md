# Forms & Turbo-Correct Responses

`form_with` builds forms that work with Turbo out of the box — but the controller
must answer with the **right status** or Turbo silently does nothing. The single
most important rule: **re-render an invalid form with
`status: :unprocessable_entity`**. Routing/status mechanics are owned by
[`../rails-controllers/`](../../rails-controllers/references/responses.md); this
file is the view+Turbo contract.

## Quick Pattern

```erb
<%= form_with model: @post do |f| %>
  <% if @post.errors.any? %>
    <div id="error_explanation"><%= pluralize(@post.errors.count, "error") %></div>
  <% end %>
  <%= f.text_field :title %>
  <%= f.submit %>
<% end %>
```
```ruby
def create
  @post = Post.new(post_params)        # post_params: ../rails-controllers/ (actions-and-params.md)
  if @post.save
    redirect_to @post, notice: "Created."          # success → redirect (PRG)
  else
    render :new, status: :unprocessable_entity      # FAILURE → 422 so Turbo replaces the form
  end
end
```

`form_with` is remote (Turbo) by default — it submits via Turbo Drive and expects a
**non-2xx** status to re-render the form with errors. A 200 here leaves the form
untouched and the user confused.

## The status contract (why 422 matters)

| Outcome | Status | What Turbo does |
|---|---|---|
| Saved | 303/302 via `redirect_to` | follows redirect (PRG, no double-submit) |
| Invalid | **422 `:unprocessable_entity`** | renders the response, replacing the form |
| Invalid but 200 | 200 | **nothing** — form looks frozen (the classic bug) |

## Turbo Stream form responses

For inline/partial updates instead of a redirect, answer with a stream:

```ruby
respond_to do |format|
  if @post.save
    format.turbo_stream  # create.turbo_stream.erb: prepend the row, clear the form
    format.html { redirect_to @post }
  else
    format.turbo_stream { render turbo_stream: turbo_stream.replace(@post, partial: "form", locals: { post: @post }), status: :unprocessable_entity }
    format.html { render :new, status: :unprocessable_entity }
  end
end
```

See [turbo.md](turbo.md) for stream actions and frame targeting.

## Deep Dive

- **`form_with model:`** infers URL, method, and field names from the record (and
  whether it's create vs update). Prefer it over hand-built `form_tag`.
- **Inside a Turbo Frame:** a form in a `turbo_frame_tag` updates only that frame;
  the response must include a matching frame id ([turbo.md](turbo.md)).
- **CSRF** tokens are injected by `form_with` automatically; don't disable
  `protect_from_forgery` (security: `../rails-security/`).
- **File uploads** in forms (`f.file_field`, direct upload) → `../rails-storage/`.
- **Flash + Turbo:** flash shown after a redirect works normally; to flash within a
  stream render, include the flash partial as another stream target.

## Common Pitfalls

- **200 on validation failure** → the #1 Hotwire form bug; always `status:
  :unprocessable_entity` on re-render.
- **Missing error display** in the template → user sees a frozen form with no
  reason; render `@post.errors`.
- **`form_with local: true`** to "fix" a broken form → that disables Turbo instead
  of fixing the status code; fix the 422 instead.
- **Form inside a frame but response has no matching frame** → "content missing."
- **Redirecting with 200 instead of 3xx** → `redirect_to` already issues the right
  status; don't override it to 200.

## Verify

```bash
bin/rails server -d && sleep 3
# Invalid submit returns 422 (the load-bearing check):
curl -s -o /dev/null -w "invalid=%{http_code}\n" -X POST localhost:3000/posts \
  -H "Accept: text/vnd.turbo-stream.html, text/html" -d "post[title]="          # expect 422
# Valid submit redirects (3xx) then GETs the resource:
curl -s -o /dev/null -w "valid=%{http_code}\n" -X POST localhost:3000/posts \
  -d "post[title]=t&post[body]=b"                                                # expect 302/303
kill "$(cat tmp/pids/server.pid)"
```

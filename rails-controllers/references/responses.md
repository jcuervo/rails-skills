# Responses: Formats, Status Codes & Redirects

How a controller finishes a request — choosing a format, an HTTP status, a
redirect, or `head`. This reference owns the *mechanics* of responding; it does
**not** own the response *body*: JSON serialization is `../rails-api/SKILL.md`,
HTML/Turbo rendering is `../rails-hotwire/SKILL.md`. The split: **how to respond**
here, **what's in the body** there. Behavior branches on `config.api_only`.

## Quick Pattern

### API mode (`config.api_only`)

```ruby
def create
  @post = Post.new(post_params)
  if @post.save
    render json: @post, status: :created, location: post_url(@post)
  else
    render json: { errors: @post.errors }, status: :unprocessable_entity
  end
end

def destroy
  Post.find(params[:id]).destroy!
  head :no_content                      # 204, empty body
end
```

### Web mode (full-stack)

```ruby
def create
  @post = Post.new(post_params)
  if @post.save
    redirect_to @post, notice: "Post created."     # PRG: redirect after write
  else
    render :new, status: :unprocessable_entity      # re-render form with errors
  end
end
```

## Status codes that matter

| Situation | Status | Symbol |
|---|---|---|
| Created a resource | 201 | `:created` |
| Updated / OK with body | 200 | `:ok` |
| Success, no body | 204 | `:no_content` |
| Validation failed | 422 | `:unprocessable_entity` |
| Not found (`RecordNotFound`) | 404 | `:not_found` |
| Unauthenticated / unauthorized | 401 / 403 | `:unauthorized` / `:forbidden` |
| Bad/tampered params (`params.expect`) | 400 | `:bad_request` |

- **Render the form with `status: :unprocessable_entity`** on validation failure —
  Turbo needs the non-2xx status to replace the form; a 200 here breaks Turbo
  Drive form handling.
- Map exceptions to these statuses once via `rescue_from` —
  see [concerns-and-filters.md](concerns-and-filters.md) — not per action.

## Deep Dive

- **`respond_to`** for multi-format endpoints (HTML *and* JSON from one action):

  ```ruby
  respond_to do |format|
    format.html { redirect_to @post }
    format.json { render json: @post, status: :created }
    format.turbo_stream  # → ../rails-hotwire/ owns the .turbo_stream template
  end
  ```
  API-only apps usually skip `respond_to` (JSON only). Full-stack apps that also
  serve JSON use it.
- **Redirect vs render:** redirect after a successful write (Post/Redirect/Get —
  avoids duplicate submits); render to show a form again with errors. `redirect_to`
  issues a new request; `render` reuses the current one (so instance vars persist).
- **`head`** for bodiless responses (`head :no_content`, `head :forbidden`).
- **`location:`** header on `:created` so clients can follow the new resource.
- The actual JSON structure / error envelope (`../rails-api/`) and the HTML/Turbo
  templates (`../rails-hotwire/`) are owned by those skills — call into them; don't
  re-document them here.

## Common Pitfalls

- **200 on a failed form render** → Turbo won't swap the invalid form; always
  `status: :unprocessable_entity`.
- **Rendering after redirecting** (`DoubleRenderError`) → `return` after a
  `redirect_to`/`render`, or guard with `and return`.
- **`render json:` in an API app for errors but no consistent envelope** → define
  the envelope once in `../rails-api/`; keep status mapping here.
- **Forgetting `:created`/`location` on POST** → REST clients can't discover the
  new resource URL.
- **Using `redirect_to` in an API-only app** → APIs return status + body, not
  browser redirects (except deliberate 3xx).

## Verify

```bash
bin/rails server -d && sleep 3
# Correct statuses on the happy and sad paths:
curl -fsS -o /dev/null -w "create=%{http_code}\n" -X POST localhost:3000/posts \
  -H 'Content-Type: application/json' -d '{"post":{"title":"t","body":"b"}}'   # expect 201
curl -s  -o /dev/null -w "invalid=%{http_code}\n" -X POST localhost:3000/posts \
  -H 'Content-Type: application/json' -d '{"post":{"title":""}}'              # expect 422
curl -s  -o /dev/null -w "missing=%{http_code}\n" localhost:3000/posts/999999  # expect 404
kill "$(cat tmp/pids/server.pid)"
```

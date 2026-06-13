# Actions & Strong Parameters

Thin RESTful actions and safe input filtering. The menu pick is **`params.expect`**
(Rails 8); `params.require(...).permit(...)` is the compatible fallback.

## Quick Pattern (thin RESTful controller)

```ruby
class PostsController < ApplicationController
  before_action :set_post, only: %i[show update destroy]

  def index  = render_collection(Post.all)
  def show; end

  def create
    @post = Post.new(post_params)
    @post.save ? respond_created(@post) : respond_unprocessable(@post)
  end

  def update
    @post.update(post_params) ? respond_ok(@post) : respond_unprocessable(@post)
  end

  def destroy
    @post.destroy!
    head :no_content
  end

  private

  def set_post = @post = Post.find(params[:id])

  # Rails 8 idiom — strict shape, 400 on tampering:
  def post_params = params.expect(post: [:title, :body, tag_ids: []])
end
```

(The `respond_*` helpers stand in for the response layer — JSON shaping is
`../rails-api/`, HTML/Turbo is `../rails-hotwire/`. Keep the action thin: params
in, model call, response out.)

## `params.expect` (Recommended, Rails 8) vs `require/permit`

```ruby
# Rails 8 — params.expect:
params.expect(post: [:title, :body])              # scalars listed in a single array
params.expect(post: [:title, comments: [[:body]]])# nested ARRAY of hashes => DOUBLE brackets
params.expect(ids: [])                            # top-level array of scalars

# Equivalent, all versions — require/permit:
params.require(:post).permit(:title, :body)
params.require(:post).permit(:title, comments: [:body])
```

- `expect` rejects anything that doesn't match the declared **shape** — e.g. a
  scalar where a hash was expected — as **400 Bad Request**. (A plain missing key
  under `require` is already rescued to 400 in modern Rails; the real upgrade is
  that `expect` also rejects *wrong-type/wrong-shape* input that `permit` would let
  through or surface as a 500.) The key safety win is the **double-bracket** rule:
  `comments: [[:body]]` declares "an array of hashes," which `permit` would express
  as `comments: [:body]`. Verify the exact behavior against the target app's Rails
  version (detect step above) before relying on the status.
- **Pick `expect`** on Rails 8 for new code. **Pick `require/permit`** on Rails <
  8, or to stay consistent inside a codebase that already uses it everywhere.
- Confirm availability on the target app: `params.expect` exists only on Rails 8+
  — detect the version (Detect Before You Generate) before standardizing on it.

## Deep Dive

- **Skinny actions:** an action should read params, call a model/service, and pick
  a response. Multi-step logic → a model method (`../rails-models/`) or a service
  object. A 40-line action is a smell.
- **`before_action` for setup/guards** (`set_post`, auth) keeps actions DRY — see
  [concerns-and-filters.md](concerns-and-filters.md).
- **`find` vs `find_by`:** `find` raises `RecordNotFound` (→ 404 via `rescue_from`);
  `find_by` returns nil (you handle it). Use `find` for "must exist" show/update.
- **Nested attributes:** permit them explicitly
  (`params.expect(post: [:title, comments_attributes: [[:id, :body, :_destroy]]])`)
  and pair with `accepts_nested_attributes_for` on the model.
- **API vs Web:** the *action* is largely the same; the *response* differs —
  branch there, not here.

## Common Pitfalls

- **Mass assignment via unfiltered params** (`Post.new(params[:post])`) → never;
  always go through a permitted/expected method.
- **Single brackets for an array of hashes under `expect`** → silently wrong shape;
  arrays of hashes need `[[...]]`.
- **`params.expect` on Rails < 8** → `NoMethodError`; use `require/permit` there.
- **Fat actions** doing validations/calculations → move to the model/service.
- **`find_by` without a nil check** → `NoMethodError` on the next line instead of a
  clean 404.

## Verify

```bash
# Param filtering drops unpermitted keys (no request context needed):
bin/rails runner "p ActionController::Parameters.new(post: {title: 't', admin: true}).expect(post: [:title]).to_h"   # => {"title"=>"t"} — admin dropped
bin/rails runner "p ActionController::Parameters.new(post: {title: 't', admin: true}).require(:post).permit(:title).to_h"  # same, require/permit form
# End to end — valid create succeeds, wrong-shape params are rejected:
bin/rails server -d && sleep 3
curl -fsS -o /dev/null -w "valid=%{http_code}\n"  -X POST localhost:3000/posts -H 'Content-Type: application/json' -d '{"post":{"title":"x","body":"y"}}'   # 2xx
curl -s  -o /dev/null -w "tampered=%{http_code}\n" -X POST localhost:3000/posts -H 'Content-Type: application/json' -d '{"post":"not-a-hash"}'              # 400 under expect
kill "$(cat tmp/pids/server.pid)"
```

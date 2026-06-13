# Serialization

The menu deep dive for turning models into JSON. Pick **one** serializer and use
it consistently — a mixed codebase produces inconsistent response shapes.

## Menu recap

| Option | Infra | Best for |
|---|---|---|
| **Jbuilder** *(Recommended)* | ships with Rails | Omakase, flexible, low/medium volume |
| Alba | one gem | Performance-sensitive endpoints, clean DSL |
| jsonapi-serializer | one gem | Clients that require the JSON:API spec |
| Blueprinter | one gem | Simple view-based DSL, middle ground |

## Jbuilder (Recommended — Rails default)

Jbuilder ships with the full-stack Rails Gemfile; confirm it's present
(`grep jbuilder Gemfile.lock`) — pure `--api` apps sometimes drop it.

```ruby
# app/views/posts/show.json.jbuilder
json.id     @post.id
json.title  @post.title
json.author { json.(@post.author, :id, :name) }
json.comments @post.comments, :id, :body
```
```ruby
def show = @post = Post.find(params[:id])   # renders show.json.jbuilder
```

- **Pick when** you want zero new dependencies and template-style flexibility, and
  volume is modest.
- **Don't pick when** an endpoint is hot and large — Jbuilder's per-render template
  evaluation is slower than a compiled serializer. Measure with
  `../rails-performance/`.

## Alba (performance pick)

```ruby
# Gemfile: gem "alba"
class PostResource
  include Alba::Resource
  attributes :id, :title
  one :author, resource: AuthorResource
  many :comments, resource: CommentResource
end
PostResource.new(@post).serialize        # in the controller: render json: ...
```

- **Pick when** you need speed and a clean, explicit resource DSL with low
  overhead.
- **Don't pick when** you specifically need JSON:API output (use jsonapi-serializer)
  or want to avoid any new gem (Jbuilder).

## jsonapi-serializer (JSON:API spec)

```ruby
# Gemfile: gem "jsonapi-serializer"
class PostSerializer
  include JSONAPI::Serializer
  attributes :title, :created_at
  belongs_to :author
  has_many :comments
end
render json: PostSerializer.new(@post, include: [:author]).serializable_hash
```

- **Pick when** clients expect the JSON:API media type (typed resources,
  `data`/`included`, relationship links).
- **Don't pick when** you don't need that envelope — its structure is verbose for
  simple APIs.

## Blueprinter (middle ground)

```ruby
# Gemfile: gem "blueprinter"
class PostBlueprint < Blueprinter::Base
  identifier :id
  fields :title, :created_at
  association :author, blueprint: AuthorBlueprint
  view(:summary) { fields :title }     # named views for different shapes
end
render json: PostBlueprint.render(@post, view: :summary)
```

- **Pick when** you want named views/variants and a simple DSL without Jbuilder
  templates.

## Cross-cutting

- **N+1:** any serializer that walks associations triggers N+1 unless you
  `includes(...)` in the query. That's a `../rails-models/` (eager load) +
  `../rails-performance/` concern — always eager-load what you serialize.
- **Consistency:** the **error** body is a separate concern — see
  [error-envelopes.md](error-envelopes.md). Keep success and error shapes coherent.

## Common Pitfalls

- **Mixing serializers** across endpoints → inconsistent payloads; standardize.
- **Serializing without eager loading** → N+1 on every nested association.
- **Leaking sensitive attributes** (password digests, internal flags) by dumping
  the whole record → list fields explicitly; never `render json: @user` raw.
- **Assuming Jbuilder is present in `--api`** → verify in `Gemfile.lock`.

## Verify

```bash
bin/rails runner "p PostSerializer rescue p 'jbuilder/views' " 2>/dev/null   # serializer class loads (if gem-based)
bin/rails server -d && sleep 3
curl -fsS -H "Accept: application/json" localhost:3000/posts/1 -w "\n%{http_code} %{content_type}\n" | head -20  # JSON shape + content type
kill "$(cat tmp/pids/server.pid)"
```

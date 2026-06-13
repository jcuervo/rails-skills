# OpenAPI / API Documentation

Machine-readable docs for the API. The menu pick is **rswag** — generate the
OpenAPI spec from request specs so the docs can't drift from the real behavior.

## rswag (Recommended — spec-first, test-generated)

```ruby
# Gemfile (test/dev as appropriate):
gem "rswag-specs"
gem "rswag-api"
gem "rswag-ui"
```
```ruby
# spec/requests/posts_spec.rb — an executable spec that ALSO emits OpenAPI
require "swagger_helper"
RSpec.describe "Posts API", type: :request do
  path "/v1/posts/{id}" do
    get "Fetch a post" do
      tags "Posts"; produces "application/json"
      parameter name: :id, in: :path, type: :integer
      response 200, "found" do
        schema type: :object, properties: { id: { type: :integer }, title: { type: :string } }
        run_test!
      end
      response 404, "not found" do run_test! end
    end
  end
end
```
```bash
bundle exec rake rswag:specs:swaggerize   # runs the specs, writes swagger/v1/swagger.yaml
# rswag-ui serves it at /api-docs
```

- **Pick when** you already use RSpec (`../rails-testing/`) — docs become a
  byproduct of passing tests, so they stay accurate.
- **Don't pick when** the team uses Minitest exclusively or wants a hand-authored
  contract-first spec (see below).

## Hand-written OpenAPI (contract-first)

```
openapi.yaml            # the source of truth, checked in
```
- Author `openapi.yaml` by hand (or generate a stub), serve it via Swagger UI /
  Redoc, and optionally validate requests/responses against it with the
  `committee` gem.
- **Pick when** the API contract is designed up front (multiple teams/clients
  agree on it before implementation), or you're not on RSpec.
- **Don't pick when** you'd rather not hand-maintain a spec that can silently drift
  from the code — rswag couples it to tests instead.

## When to skip

A young internal API with one in-house consumer may not need published docs yet.
Add them when external/multiple consumers appear. Don't gold-plate prematurely.

## Deep Dive

- **Drift is the enemy.** Hand-written docs rot; test-generated docs (rswag) and
  request-validated docs (committee) stay honest because something fails when they
  diverge.
- **Version the spec** alongside the API version ([versioning.md](versioning.md)) —
  `swagger/v1/`, `swagger/v2/`.
- **Document the error envelope** ([error-envelopes.md](error-envelopes.md)) as a
  reusable schema component so every error response references it.
- Keep `rswag-ui`/Swagger UI **gated** in production (auth or internal-only) if the
  API isn't public — it exposes your whole surface.

## Common Pitfalls

- **Docs that drift from reality** → consumers code against lies. Generate from
  tests or validate against the spec.
- **Publishing Swagger UI unauthenticated** on a private API → surface disclosure;
  gate it.
- **Documenting only happy paths** → clients can't handle 4xx; document the error
  envelope and key error statuses.
- **One spec file for a versioned API** → version the spec with the API.

## Verify

```bash
bundle exec rake rswag:specs:swaggerize 2>/dev/null && ls swagger/**/swagger.yaml   # spec regenerates from passing specs
bin/rails server -d && sleep 3
curl -fsS localhost:3000/api-docs -o /dev/null -w "docs-ui=%{http_code}\n"           # Swagger UI served (rswag-ui)
kill "$(cat tmp/pids/server.pid)"
# Spec is valid OpenAPI (if a linter is available):
npx --yes @redocly/cli lint swagger/v1/swagger.yaml 2>/dev/null || echo "(optional) lint the spec"
```

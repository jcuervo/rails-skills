# API Versioning

How clients pin to a stable contract. The menu pick is **URL path** for its
explicitness; don't version at all until you actually break compatibility.

## Quick Pattern — URL path (Recommended)

```ruby
# config/routes.rb
namespace :v1 do
  resources :posts
end
```
```
app/controllers/v1/posts_controller.rb  →  class V1::PostsController < Api::BaseController
```

- Routes, controllers, and (optionally) serializers live under a `v1` namespace.
- A base `Api::BaseController < ActionController::API` (or a `V1::BaseController`
  under it) holds shared `rescue_from`, auth filters, and the error envelope — every
  versioned controller inherits from it. See
  [error-envelopes.md](error-envelopes.md).

## When to pick which

- **URL path** — default. Trivial to route, test (`curl /v1/posts`), cache, and
  reason about. Slightly "uglier" URLs; that's the cost.
- **Accept-header / media type** (`Accept: application/vnd.app.v1+json`) — clean
  URLs and "proper" REST content negotiation, but harder to test by hand, cache
  correctly (must `Vary: Accept`), and debug. Pick only when a client contract
  requires it.
- **No versioning (yet)** — the right default for a young API. Versioning is a cost;
  add `/v1` the moment you ship a breaking change, not before.

## Deep Dive — Accept-header negotiation

```ruby
# A routing constraint that inspects the Accept header
class ApiVersion
  def initialize(version) = @version = version
  def matches?(req) = req.headers["Accept"]&.include?("application/vnd.app.v#{@version}+json")
end
namespace :api, defaults: { format: :json } do
  scope module: :v1, constraints: ApiVersion.new(1) { resources :posts }
end
```

- Always send `Vary: Accept` so caches don't serve a v1 body to a v2 request.
- Provide a sane default version when the header is absent (usually the latest, or
  a deliberate 406).

## Managing change across versions

- **Additive changes** (new optional field) don't need a new version — add them to
  the current one.
- **Breaking changes** (removed/renamed field, changed type, different status) need
  a new version. Keep the old version running until clients migrate; track a
  deprecation/sunset window (`Deprecation`/`Sunset` headers).
- Share unchanged logic via a base controller/concern; override only what diverges.

## Common Pitfalls

- **Versioning on day one** → ceremony with no payoff; ship unversioned (or `/v1`)
  and branch only when you break the contract.
- **Header negotiation without `Vary: Accept`** → cache serves the wrong version.
- **Duplicating entire controllers per version** → copy-paste drift; inherit from a
  shared base and override the deltas.
- **Changing a field's meaning within a version** → silent client breakage; that's
  a new version by definition.

## Verify

```bash
bin/rails routes | grep -E "v1|api"                       # versioned routes present
bin/rails server -d && sleep 3
curl -fsS localhost:3000/v1/posts -w "\n%{http_code}\n" -o /dev/null   # path version resolves
# Header negotiation (if used): same path, version chosen by Accept
curl -fsS localhost:3000/api/posts -H "Accept: application/vnd.app.v1+json" -D - -o /dev/null | grep -i "vary"
kill "$(cat tmp/pids/server.pid)"
```

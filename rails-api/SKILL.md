---
name: rails-api
description: Build the JSON API surface of a Rails 8.1 app — response serialization (Jbuilder, Alba, jsonapi-serializer, Blueprinter), API versioning, pagination (Pagy/Kaminari), consistent error envelopes, CORS, OpenAPI/Swagger docs, and the token-auth surface — for API-only apps and the JSON endpoints of full-stack apps. Covers both the REST path and GraphQL (graphql-ruby) as the alternative API style. Menu-driven: presents the API style (REST vs GraphQL) plus serializer, versioning, pagination, and docs options as vetted menus with a Recommended default, detects api_only and any installed serializer/pagination/graphql gem first, and verifies endpoints return well-shaped responses with correct status codes. Apply when shaping JSON responses, versioning an API, paginating a collection endpoint, standardizing error responses, enabling CORS, documenting an API, or building a GraphQL schema.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[resource-or-endpoint]"
---

# rails-api

## Purpose

Owns the **JSON response surface**: given a controller action (routing, params,
and status mapping are `../rails-controllers/`), this skill decides how the body
is serialized, how the API is versioned and paginated, how errors are enveloped
consistently, how CORS is configured, and how the API is documented. It applies to
**API-only apps** and to the **JSON endpoints of full-stack apps** (an app can
serve HTML via `../rails-hotwire/` *and* JSON via this skill). It does not own
authentication — only the API-side *surface* of it (Bearer tokens, 401 shape);
the mechanism is `../rails-auth/`.

## When to Apply

Use this skill when the task is:

- Shaping a JSON response body (which serializer, what fields, nesting)
- Versioning an API (`/v1`, header negotiation)
- Paginating a collection endpoint
- Standardizing error responses into a consistent envelope
- Enabling/locking down CORS for browser or mobile clients
- Generating OpenAPI/Swagger documentation
- Designing the API-side token-auth surface (request header, 401 body)
- Building a **GraphQL** API (graphql-ruby): schema, types, the single endpoint

Do **not** use this skill when the task is:

- Routes, controllers, actions, strong params, status-code mapping → read `../rails-controllers/SKILL.md`
- The data/model behind the endpoint → read `../rails-models/SKILL.md`
- HTML / Turbo / Stimulus responses → read `../rails-hotwire/SKILL.md`
- The actual authentication/authorization mechanism → read `../rails-auth/SKILL.md`
- Background processing triggered by an endpoint → read `../rails-jobs/SKILL.md`
- Request/integration specs for the API → read `../rails-testing/SKILL.md`
- Rate limiting / security headers → read `../rails-security/SKILL.md`
- N+1 in serialization, caching responses → read `../rails-performance/SKILL.md`

## Detect Before You Generate

```bash
grep -nE "api_only" config/application.rb                          # api_only vs full-stack JSON endpoints
grep -E "^    (jbuilder|alba|jsonapi-serializer|blueprinter|pagy|kaminari|rack-cors|rswag)" Gemfile.lock   # installed picks
ls app/serializers app/views/**/*.json.jbuilder 2>/dev/null        # existing serialization style
ls config/initializers/cors.rb 2>/dev/null                         # CORS already configured?
grep -rnE "namespace :v[0-9]|/v[0-9]" config/routes.rb             # existing versioning scheme
grep -E "^ +graphql " Gemfile.lock; ls app/graphql 2>/dev/null     # GraphQL already in use? → graphql.md, skip the REST menus
```

- If a serializer/pagination gem is **already installed**, use it silently — don't
  re-ask the menu; match the existing style.
- `config.api_only == true` → pure JSON app; `ActionController::API` base, CORS
  usually required for browsers, no view layer.
- A full-stack app reaching this skill is adding JSON endpoints **alongside** HTML
  — branch the controller with `respond_to`/format (`../rails-controllers/`).
- If `STACK.md` records an API serializer/docs choice, honor it.

## Menu

Five menued decisions, each via `AskUserQuestion`, Rails-default marked
**Recommended**; ask unless the gem is already installed. The **API style** menu
comes first — if it resolves to GraphQL, the serializer/versioning/pagination menus
below don't apply (the schema, schema evolution, and connections replace them); only
CORS and token-auth carry over.

### API style
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **REST + JSON** *(Recommended)* | The Rails-omakase path; resourceful endpoints + a serializer, cache-friendly on URLs. The rest of this skill. | the menus below |
| GraphQL (graphql-ruby) | One typed `POST /graphql`; clients pick fields. Great for diverse clients, but adds a schema layer + N+1/cost control you own. | [graphql.md](references/graphql.md) |

### Serializer
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Jbuilder** *(Recommended)* | Ships with Rails; template-based JSON in `*.json.jbuilder`. Zero new deps, flexible, but slower at high volume. | [serialization.md](references/serialization.md) |
| Alba | Fast, modern, dependency-light declarative serializer. The common performance pick. | [serialization.md](references/serialization.md) |
| jsonapi-serializer | Enforces the JSON:API spec (typed resources, relationships). Pick when clients expect JSON:API. | [serialization.md](references/serialization.md) |
| Blueprinter | Simple view-based DSL, good middle ground between Jbuilder and Alba. | [serialization.md](references/serialization.md) |

### Versioning strategy
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **URL path (`/v1/...`)** *(Recommended)* | `namespace :v1`; explicit, cache-friendly, trivial to route and test. Most common. | [versioning.md](references/versioning.md) |
| Accept-header (media type) | `Accept: application/vnd.app.v1+json`; clean URLs, but harder to test/curl/cache. | [versioning.md](references/versioning.md) |
| No versioning (yet) | Don't add versioning before a second version is real; add `/v1` when you actually break compatibility. | [versioning.md](references/versioning.md) |

### Pagination
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Pagy** *(Recommended)* | Fastest, lowest-memory; headers + metadata for APIs. The modern default. | [pagination.md](references/pagination.md) |
| Kaminari | Feature-rich, scope-based (`page`/`per`); heavier but ubiquitous. | [pagination.md](references/pagination.md) |
| Cursor/keyset | For large/infinite feeds where offset pagination drifts and slows. | [pagination.md](references/pagination.md) |

### API documentation
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **rswag** *(Recommended)* | Generates OpenAPI/Swagger from request specs — docs that stay true because tests produce them. | [openapi-docs.md](references/openapi-docs.md) |
| Hand-written OpenAPI | A checked-in `openapi.yaml` served by Swagger UI; full control, manual upkeep. | [openapi-docs.md](references/openapi-docs.md) |
| None | Skip until the API has external consumers. | [openapi-docs.md](references/openapi-docs.md) |

## Decision Flow

- **API style:** default to **REST** — it's the omakase path and HTTP-cache-friendly.
  Choose **GraphQL** when clients are diverse and want to shape their own payloads, or
  over/under-fetching across many REST endpoints is a real pain; accept the schema
  layer, dataloader/N+1 vigilance, and query-cost limits it requires. The two can
  coexist in one app. GraphQL → [graphql.md](references/graphql.md); the menus below
  are the REST path.
- **Serializer:** staying omakase / low volume → Jbuilder. Hot endpoints or large
  payloads → Alba (or jsonapi-serializer if clients want JSON:API). Pick **one**
  and use it consistently — mixing serializers fragments the response shape.
- **Versioning:** don't version prematurely. When you must, **URL path** unless a
  client contract demands media-type negotiation. Namespace controllers to match
  (`app/controllers/v1/`).
- **Pagination:** Pagy for new APIs; match Kaminari if it's already in the app.
  Switch to cursor/keyset only when offset pagination measurably hurts (deep pages
  on big tables) — coordinate with `../rails-performance/`.
- **Errors:** define **one** envelope shape and emit it from a single `rescue_from`
  layer in the base API controller. Status-code mapping is `../rails-controllers/`;
  the JSON *body* of the error is here.
- **CORS:** required when a browser app on another origin calls the API. Lock
  `origins` down to known hosts in production — never ship `origins "*"` with
  credentials.

## Problem → Reference

| Task | Read |
|---|---|
| Choose + wire a serializer (Jbuilder/Alba/jsonapi-serializer/Blueprinter) | [references/serialization.md](references/serialization.md) |
| Version the API (URL path / header / when not to) | [references/versioning.md](references/versioning.md) |
| Paginate a collection endpoint (Pagy/Kaminari/cursor) | [references/pagination.md](references/pagination.md) |
| Standardize error responses into one envelope | [references/error-envelopes.md](references/error-envelopes.md) |
| Configure CORS for browser/mobile clients | [references/cors.md](references/cors.md) |
| Generate OpenAPI/Swagger docs (rswag / hand-written) | [references/openapi-docs.md](references/openapi-docs.md) |
| Design the API-side token-auth surface (Bearer header, 401 body) | [references/token-auth-surface.md](references/token-auth-surface.md) |
| Build a GraphQL API instead of REST (schema, types, one endpoint, N+1) | [references/graphql.md](references/graphql.md) |

## Verify

An API change is unfinished until a real request returns well-shaped JSON with the
right status and content type:

```bash
bin/rails server -d && sleep 3
BASE=localhost:3000
curl -fsS -H "Accept: application/json" $BASE/v1/posts -w "\n%{http_code} %{content_type}\n"   # 200 + application/json
curl -fsS -H "Accept: application/json" "$BASE/v1/posts?page=2" -D - -o /dev/null | grep -i "current-page\|link\|total"  # pagination headers
curl -s  -H "Accept: application/json" $BASE/v1/posts/999999 -w "\n%{http_code}\n"             # 404 in the error envelope
kill "$(cat tmp/pids/server.pid)"
bundle exec rspec spec/requests 2>/dev/null || bin/rails test test/integration 2>/dev/null      # request specs green
```

Then route to `../rails-auth/` to protect endpoints, `../rails-testing/` for
request specs, and record the serializer/docs picks in `STACK.md`.

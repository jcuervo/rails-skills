# GraphQL API (graphql-ruby)

The **alternative to the REST path** the rest of this skill describes. Instead of
many serialized endpoints, GraphQL exposes **one** `POST /graphql` endpoint with a
typed schema; clients ask for exactly the fields they want. Picking GraphQL replaces
this skill's **serializer**, **versioning**, and **pagination** menus with the schema,
schema evolution, and connection-based pagination respectively.

## Quick Pattern

```bash
bundle add graphql
bin/rails generate graphql:install          # app/graphql/, AppSchema, GraphqlController, POST /graphql route
                                            #   add --api on an api_only app to skip the GraphiQL HTML IDE
bundle install                              # picks up graphiql-rails (dev IDE), unless --skip-graphiql
```
```ruby
# app/graphql/types/query_type.rb — the read schema
module Types
  class QueryType < Types::BaseObject
    field :articles, [Types::ArticleType], null: false
    def articles = Article.all          # resolver
  end
end
```
```ruby
# app/graphql/types/article_type.rb — a type maps a model to fields
module Types
  class ArticleType < Types::BaseObject
    field :id, ID, null: false
    field :title, String, null: false
    field :comments, [Types::CommentType], null: false
  end
end
```
```bash
# query it (one endpoint, client picks the fields):
curl -fsS localhost:3000/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{ articles { title comments { body } } }"}'
```

`graphql:install` supports `--api`, `--skip-graphiql`, `--relay`, `--batch`, and
`--schema NAME`; run `bin/rails generate graphql:install --help` to confirm the
options for your installed gem version before generating.

## When to pick / not pick

- **Pick GraphQL** when clients are diverse and want to shape their own payloads
  (mobile + web + third parties), when over/under-fetching across many REST endpoints
  is a real pain, or when a strongly-typed, introspectable schema is a product
  requirement.
- **Stick with REST** (the rest of this skill) when the API is CRUD-shaped and
  cache-friendly (HTTP caching on URLs is far simpler than caching a single POST),
  when consumers are few and stable, or when you want the lowest-ceremony path —
  GraphQL adds a schema layer, N+1 vigilance, and query-cost control you must own.
- **They can coexist:** a GraphQL endpoint can live alongside REST endpoints in the
  same app; route auth applies to both (`../rails-auth/`).

## Deep Dive

- **One endpoint, typed schema:** `GraphqlController#execute` runs queries against
  `AppSchema`. Types live in `app/graphql/types`; mutations in `app/graphql/mutations`.
- **N+1 is the headline risk.** Field resolvers fan out into per-record queries.
  Solve with batching — `graphql-batch` or `dataloader` (built into graphql-ruby) —
  not eager-`includes` alone, since the client decides which associations load.
  Coordinate with `../rails-performance/`.
- **Pagination = connections.** GraphQL's cursor-based **connections**
  (`field :articles, Types::ArticleType.connection_type`) replace Pagy/Kaminari;
  they're keyset-style by design (`pagination.md` covers the REST equivalent).
- **No URL versioning.** Evolve the schema instead: add fields freely, `deprecation_reason`
  old ones, avoid breaking changes. This replaces `versioning.md`.
- **Auth + authorization:** authenticate in `GraphqlController` (set `context[:current_user]`
  from the token surface — `token-auth-surface.md`), authorize per-field/type with
  graphql-ruby's `authorized?`/`visible?` hooks or Pundit/Action Policy
  (`../rails-auth/`).
- **Errors** aren't HTTP status codes — GraphQL returns `200` with an `errors` array.
  Raise `GraphQL::ExecutionError` for handled failures; this is a different model from
  REST `error-envelopes.md`.
- **Cost control:** cap `max_depth`/`max_complexity` on the schema so a malicious deep
  query can't DoS you.

## Common Pitfalls

- **Unbatched resolvers** → severe N+1 under nested queries; add a dataloader from day
  one.
- **Expecting HTTP status codes** → errors come back `200` with an `errors` key;
  clients must inspect the body.
- **Shipping GraphiQL to production** → use `--api` or mount the IDE dev-only.
- **No depth/complexity limit** → a single nested query can exhaust the DB; set caps.
- **Reinventing versioning** → don't add `/v2`; deprecate and evolve the schema.
- **CORS still applies** for browser clients hitting `/graphql` from another origin
  (`cors.md`).

## Verify

```bash
bin/rails server -d && sleep 3
# A real query returns data the client asked for, HTTP 200:
curl -fsS localhost:3000/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{ __schema { queryType { name } } }"}' -w "\n%{http_code}\n"   # 200 + JSON with data.__schema
# Schema loads without type errors:
bin/rails runner "p AppSchema.query.fields.keys" 2>/dev/null                    # lists your root query fields
kill "$(cat tmp/pids/server.pid)"
```
(Substitute your schema's constant if you passed `--schema`.)

Then route to `../rails-performance/` (dataloader/N+1), `../rails-auth/` (per-field
authorization), `../rails-testing/` (schema/request specs), and record the GraphQL
choice in `STACK.md`.

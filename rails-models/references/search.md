# Full-Text & Fuzzy Search

The menu deep dive for searching model data. Four options, escalating in power
and ops cost. **Start at the lowest tier that meets the need** — most apps never
leave DB-native.

## Menu recap

| Option | Infra | Best for |
|---|---|---|
| **DB-native** *(Recommended)* | none | Small/medium tables; "good enough" matching |
| `pg_search` | none (Postgres) | Real search on Postgres without a new service |
| Meilisearch | one service | Typo-tolerant, relevant search; light ops |
| Elasticsearch / OpenSearch | cluster | Large scale, aggregations, existing ES infra |

## Tier 1 — DB-native (Recommended start)

**Postgres full-text:**

```ruby
scope :search, ->(q) {
  where("to_tsvector('english', title || ' ' || body) @@ plainto_tsquery('english', ?)", q)
}
```
Add a GIN index (ideally on a stored generated `tsvector` column) for speed — see
[migrations.md](migrations.md); a `tsvector` generated column is a reason to move
to `structure.sql`.

**SQLite:** use an FTS5 virtual table. **Any adapter, tiny tables:**
`where("title LIKE ?", "%#{q}%")` — fine for prototypes, but unindexed and
case/locale-naive.

- **Pick when** the dataset is modest and you want zero new infrastructure.
- **Don't pick when** you need typo-tolerance, ranking across many fields, or
  faceting at scale.

## Tier 2 — `pg_search` (Postgres only)

```ruby
# Gemfile: gem "pg_search"
class Post < ApplicationRecord
  include PgSearch::Model
  pg_search_scope :search, against: %i[title body],
    using: { tsearch: { prefix: true }, trigram: {} }   # trigram needs pg_trgm extension
end
Post.search("rails")
```

- **Pick when** you're on Postgres and want multi-column full-text + fuzzy
  (trigram) matching with a clean scope, no extra service.
- **Don't pick when** you're not on Postgres, or you need a dedicated engine's
  relevance/typo tolerance at scale.
- Enable the `pg_trgm` extension in a migration (`enable_extension "pg_trgm"`) for
  trigram/`similarity`.

## Tier 3 — Meilisearch

```ruby
# Gemfile: gem "meilisearch-rails"  (confirm the current gem name + module casing in its README)
class Product < ApplicationRecord
  include MeiliSearch::Rails
  meilisearch do
    attribute :name, :description
  end
end
Product.search("labtop")   # typo-tolerant: matches "laptop"
```

- **Pick when** you want excellent out-of-the-box relevance + typo tolerance with
  far less ops weight than Elasticsearch. Runs as a single service.
- **Don't pick when** you can't run another service, or you need ES-grade
  aggregations/analytics.
- Indexing happens via callbacks/jobs; reindex with the gem's rake task. Keep the
  service URL/key in credentials (`../rails-security/`).

## Tier 4 — Elasticsearch / OpenSearch

```ruby
# Gemfile: gem "searchkick"   (or elasticsearch-rails)
class Product < ApplicationRecord
  searchkick
end
Product.search("laptop", fields: [:name], misspellings: { below: 5 })
```

- **Pick when** you operate at large scale, need rich aggregations/analytics, or
  already run an ES/OpenSearch cluster.
- **Don't pick when** a simpler tier suffices — it's the heaviest to run, secure,
  and keep in sync.

## Common Pitfalls

- **`LIKE '%q%'` on a large table** → full scan, no index. Move up a tier.
- **External engine drift** — the index and the DB disagree after a failed
  callback/job. Schedule periodic reindex; do index writes in a job
  (`../rails-jobs/`), not inline in the request.
- **Searching unindexed `tsvector` expressions** → slow; add a GIN index on a
  stored generated column.
- **Trigram without the extension** → `pg_search` trigram errors until
  `enable_extension "pg_trgm"` runs.

## Verify

```bash
# DB-native (Postgres):
bin/rails runner "p Post.search('hello').to_sql"
# pg_search:
bin/rails runner "p Post.respond_to?(:search) && Post.search('x').to_a.class"
bin/rails runner "p ActiveRecord::Base.connection.extensions.include?('pg_trgm')"   # if using trigram
# External engine reachable (Meili/ES) — expect a 200/JSON:
curl -fsS "$SEARCH_URL/health" 2>/dev/null && echo " ✓ engine up"
```

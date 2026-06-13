---
name: rails-models
description: Design and evolve the ActiveRecord domain layer of a Rails 8.1 app — models, associations, validations, scopes, enums, callbacks, concerns, and type modeling (plain / STI / delegated types / polymorphic) — plus the persistence layer beneath them: migrations, indexing, constraints, schema format, multiple databases, full-text search, and seeds. Menu-driven: presents type-modeling, schema-format, and search options as vetted menus with a Recommended default, detects the app's adapter and existing models first, and verifies every change against a booted schema. Apply when adding or changing a model, writing a migration, indexing a table, modeling inheritance/variants, wiring search, splitting databases, or seeding data.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[model-or-table]"
---

# rails-models

## Purpose

Owns the **domain and persistence layer** of a Rails app: the ActiveRecord models
themselves (associations, validations, scopes, enums, callbacks, concerns, value
objects, and how you model inheritance/variants) **and** the database underneath
them (migrations, indexes, constraints, schema format, multiple databases,
full-text search, seeds). This is the merged "models + persistence" skill — the
DB depth lives in `references/` and stays distinctly invocable. It is **shared by
API-only and full-stack apps**: the domain layer is the same regardless of the
request surface on top of it.

## When to Apply

Use this skill when the task is:

- Creating or changing a model — associations, validations, scopes, enums, callbacks, concerns
- Modeling variants/inheritance (single model vs STI vs delegated types vs polymorphic)
- Writing, editing, or reversing a migration; adding indexes or DB constraints
- Choosing the schema format (`schema.rb` vs `structure.sql`)
- Splitting into multiple databases, adding a read replica, or sharding
- Adding full-text or fuzzy search over a model
- Writing seeds / sample data

Do **not** use this skill when the task is:

- Standing up a brand-new app or choosing the `-d` adapter for `rails new` → read `../rails-scaffold/SKILL.md` (it owns the *bootstrap* adapter pick; everything after the adapter is set is owned here)
- Routing, controllers, strong params, request handling → read `../rails-controllers/SKILL.md`
- JSON serialization / API response shaping of a model → read `../rails-api/SKILL.md`
- Turbo/views/forms rendering a model → read `../rails-hotwire/SKILL.md`
- File attachments on a model (Active Storage) → read `../rails-storage/SKILL.md`
- Background processing of model data → read `../rails-jobs/SKILL.md`
- Model/unit tests, factories vs fixtures → read `../rails-testing/SKILL.md`
- N+1 detection, query/caching performance → read `../rails-performance/SKILL.md`
- A retrospective audit of the data layer → use the `rails-audit` skill

## Detect Before You Generate

Read the app's existing choices before adding anything — never assume:

```bash
grep -E "^    rails \(" Gemfile.lock | head -1                  # actual Rails version (anchored to the gem line)
grep -nE "adapter:|replica:|database:" config/database.yml      # live adapter + any multi-DB/replica config
grep -nE "schema_format|migration_paths" config/application.rb config/environments/*.rb 2>/dev/null
ls db/schema.rb db/*structure.sql 2>/dev/null                    # which schema format is in use
ls app/models/                                                  # existing models + concerns
```

- The **adapter is already chosen** (by `rails-scaffold` or a prior dev). Detect it; adapter-specific features (Postgres JSONB/arrays/`pg_search`, SQLite FTS5) follow from it — do not re-ask the `-d` question here.
- If `STACK.md` exists at the app root, read it — it records the scaffold picks.
- If a model/table already exists, **read it before editing** and write an idempotent migration that adds only what's missing.

## Menu

Three genuinely-menued decisions in this skill. Each presents via
`AskUserQuestion` with the Rails-idiomatic default marked **Recommended**; the
agent asks unless the app already constrains the pick.

### Type / inheritance modeling
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Plain model (no inheritance)** *(Recommended)* | One model, one table. The default — reach for the others only when variance demands it. | [inheritance-and-types.md](references/inheritance-and-types.md) |
| Delegated types | Shared columns in a parent table, type-specific columns in child tables. Modern Rails answer when variants diverge. | [inheritance-and-types.md](references/inheritance-and-types.md) |
| Single Table Inheritance (STI) | All variants in one table + a `type` column. Fine when variants share almost all columns. | [inheritance-and-types.md](references/inheritance-and-types.md) |
| Polymorphic association | One association points at many model types (`*_type`/`*_id`). For "belongs to any of these," not inheritance. | [inheritance-and-types.md](references/inheritance-and-types.md) |

### Schema format → `config.active_record.schema_format`
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **`schema.rb`** *(Recommended)* | Ruby, adapter-agnostic, diff-friendly. The Rails default. | [migrations.md](references/migrations.md) |
| `structure.sql` | Raw SQL dump. Required when you use DB-native features `schema.rb` can't express (custom types, triggers, partial-index expressions, extensions). | [migrations.md](references/migrations.md) |

### Full-text / fuzzy search
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **DB-native** *(Recommended)* | Postgres full-text (`to_tsvector`) / SQLite FTS5 / `LIKE`. Zero new infra; start here. | [search.md](references/search.md) |
| `pg_search` | Wraps Postgres full-text/trigram in a clean model API. Postgres-only; the common "real search, no new service" step up. | [search.md](references/search.md) |
| Meilisearch | Dedicated typo-tolerant search engine; great relevance, light ops. Adds a service. | [search.md](references/search.md) |
| Elasticsearch / OpenSearch | Heaviest; pick for large-scale, aggregations, or existing ES infra. | [search.md](references/search.md) |

## Decision Flow

- **Modeling variants?** Same columns across types → STI. Diverging columns →
  delegated types. "Belongs to any of N parents" (comments on posts *and* photos)
  → polymorphic. No real variance → plain model. Don't reach for inheritance
  before you have two real variants.
- **Schema format?** Stay on `schema.rb` until a migration uses a feature it can't
  represent (Postgres extensions, triggers, generated columns, `CHECK` with
  expressions) — then switch to `structure.sql` once, app-wide.
- **Search?** Prototype/small table → DB-native `LIKE` or Postgres FTS. Real
  search on Postgres → `pg_search`. Typo-tolerance/relevance without running ES →
  Meilisearch. Massive scale or existing cluster → Elasticsearch.
- **Validations vs DB constraints?** Do both — model validations for UX messages,
  DB constraints (`NOT NULL`, unique index, FK, `CHECK`) for integrity that
  survives concurrency and direct SQL. See
  [validations-and-callbacks.md](references/validations-and-callbacks.md).
- **Callback or not?** Prefer `normalizes`, value objects, or an explicit service
  over a callback whenever the logic isn't intrinsic to persistence. Callbacks
  that touch other records or external services are a smell — see the reference.

## Problem → Reference

| Task | Read |
|---|---|
| Write / edit / reverse a migration; add indexes, constraints, schema format | [references/migrations.md](references/migrations.md) |
| Add associations (belongs_to/has_many/through/polymorphic), FKs, `dependent:`, counter caches | [references/associations.md](references/associations.md) |
| Validations, DB constraints, `normalizes`, concerns, when NOT to use callbacks | [references/validations-and-callbacks.md](references/validations-and-callbacks.md) |
| Model variants: plain / STI / delegated types / polymorphic | [references/inheritance-and-types.md](references/inheritance-and-types.md) |
| Enums, scopes, value objects, `generates_token_for` | [references/enums-and-scopes.md](references/enums-and-scopes.md) |
| Full-text / fuzzy search (DB-native / pg_search / Meilisearch / Elastic) | [references/search.md](references/search.md) |
| Multiple databases, read replicas, sharding, where Solid backends live | [references/multi-database.md](references/multi-database.md) |
| Seeds and sample data | [references/seeds.md](references/seeds.md) |

## Verify

Every model/migration change is unfinished until the schema and the model agree
and a real query runs. Generic gate (each reference adds specifics):

```bash
bin/rails db:migrate                                   # apply; must be reversible
bin/rails db:rollback && bin/rails db:migrate          # prove down/up both work
bin/rails runner "p <Model>.column_names"              # schema reflects the change
bin/rails runner "<Model>.new.valid?; p <Model>.new.errors.to_hash"   # validations load
bin/rails db:prepare                                   # schema file regenerates clean
```

Then route on: request layer (`../rails-controllers/` + `../rails-api/` or
`../rails-hotwire/`) → tests (`../rails-testing/`). Record any new model-layer
choices (search engine, multi-DB) in `STACK.md`.

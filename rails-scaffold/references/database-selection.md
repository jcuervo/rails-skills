# Database Selection

Picks the `-d/--database` flag for `rails new`. This is the **bootstrap** choice
only — it sets the adapter, `config/database.yml`, and the gem. Deep schema design,
migrations, indexing, multi-database setup, and full-text search are owned by
`../rails-models/SKILL.md`. Cross-link there after scaffolding.

## Menu

| Option | `-d` value | One-line trade-off |
|---|---|---|
| **SQLite** *(Recommended)* | `sqlite3` | Zero-setup, file-based, production-ready in Rails 8 (WAL + tuned defaults). Best for solo/small apps, prototypes, edge/single-server deploys. |
| PostgreSQL | `postgresql` | The default "serious" choice: concurrency, rich types (JSONB, arrays), extensions (pg_search, PostGIS), mature managed hosting. |
| MySQL | `mysql` | Ubiquitous, well-understood; pick when org/infra standardizes on it. Uses the `mysql2` gem. |
| MariaDB | `mariadb-mysql` *(or `mariadb-trilogy`)* | MySQL-compatible fork; same role as MySQL where MariaDB is the house DB. Use `mariadb-trilogy` to get the native-dep-free Trilogy client; `mariadb-mysql` to stay on the `mysql2` gem. |
| Trilogy | `trilogy` | Modern MySQL-compatible client (GitHub's), no libmysqlclient native dep. Pick for MySQL-family without the C library. |

> Rails also accepts `oracle`, `sqlserver`, `jdbc`, and `none`. These are niche;
> confirm with `rails new --help` and only use on explicit request.

## Decision Flow

- **Solo app, prototype, or single-server deploy, and you don't need Postgres
  extensions?** → SQLite. Rails 8 made it genuinely production-viable; don't
  reflexively reach for Postgres.
- **Need high write concurrency, multiple app servers, JSONB/array columns,
  full-text via `pg_search`, or PostGIS?** → PostgreSQL.
- **Org already runs MySQL/MariaDB, or a managed MySQL is mandated?** → MySQL /
  MariaDB. Prefer **Trilogy** if you want to avoid the `libmysqlclient` native
  build.
- **Genuinely unsure but expect to grow** → PostgreSQL is the safest "will scale
  and won't surprise me" pick; SQLite is the safest "I want zero ops today" pick.
  State the trade and let the user choose.

## Notes that affect the flag now (not later)

- **SQLite in production** is opt-in but first-class in Rails 8. If the user picks
  SQLite *and* plans to deploy, note it in `STACK.md`; `../rails-deploy/` covers
  the persistent-volume requirement (the DB file must survive deploys).
- **Solid Queue/Cache/Cable** (omakase backends) run on the **same** database by
  default. SQLite + Solid works out of the box. With Postgres/MySQL, Solid uses
  that DB too unless you split it — that split is a `../rails-models/` (multi-DB)
  and `../rails-jobs/` concern, not a scaffold concern.
- Switching adapters later is non-trivial (SQL dialect differences, `schema.rb`
  vs `structure.sql`). Choosing now matters; pass an explicit `-d` even for SQLite
  if a switch is plausible, to make intent visible.

## Pitfalls

- Picking Postgres/MySQL but not having the server installed → `bin/rails
  db:prepare` fails. Verify the DB server is reachable (or offer to start it /
  use a Docker container) before declaring the scaffold done.
- `mysql` needs `libmysqlclient` headers to build `mysql2`; on a bare machine this
  fails. Trilogy sidesteps it.
- Defaulting to Postgres "because it's professional" when the app is a single-user
  tool adds ops weight for no benefit — SQLite is the considered default in Rails 8.

## Verify

```bash
grep -n "adapter:" config/database.yml          # confirms the chosen adapter
bin/rails db:prepare                             # creates/loads schema — must succeed
bin/rails runner "puts ActiveRecord::Base.connection.adapter_name"
```

The `runner` line prints `SQLite`, `PostgreSQL`, `Mysql2`, or `Trilogy` —
confirming the app actually connects on the chosen adapter.

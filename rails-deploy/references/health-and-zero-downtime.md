# Health Checks, Zero-Downtime & Migrating on Deploy

How a release goes out **without dropping requests**: the `/up` health check, the
proxy's health-checked cutover, migrating the database safely, and rolling back.

## The `/up` health check

Rails ships `/up` (Rails::HealthController) by default: **200** if the app booted
with no exceptions, **500** otherwise.

```ruby
# config/routes.rb (already present in new apps)
get "up" => "rails/health#show", as: :rails_health_check
```

- Deploy tooling (kamal-proxy, Kubernetes probes, load balancers) polls `/up` to
  decide when a new container is ready.
- **Caveat:** `/up` only proves the app **booted** — it does **not** check the DB,
  cache, or external dependencies. For a deeper check, add a custom health endpoint
  that pings critical dependencies (but keep it cheap — it's polled constantly).
- **Silence health-check logs** so they don't flood production logs (Rails 8 supports
  a health-check log-silencing config — confirm the exact setting for your version).

## Zero-downtime cutover (how it works)

```
deploy → build image → push → boot NEW container → poll /up until 200
       → kamal-proxy switches traffic NEW←  → drain + stop OLD
```

- **Kamal 2 / kamal-proxy** does this automatically ([kamal.md](kamal.md)): the old
  release keeps serving until the new one is healthy, then traffic cuts over with no
  gap.
- A plain Compose run has **no** health-checked cutover — there's a restart gap; use
  Kamal (or a PaaS) for true zero-downtime.

## Migrating the database on deploy (the careful part)

Run `db:migrate` as part of deploy — but make migrations **backward-compatible** so
the **old** code can run against the **new** schema during the cutover window (both
versions are live briefly).

- **Safe (expand):** add a nullable column, add a table, add an index
  (`algorithm: :concurrently` on Postgres). Old code ignores it.
- **Unsafe (contract) in one step:** rename/drop a column, add `NOT NULL` to a
  populated column, change a type. The running old code breaks.
- **The expand/contract pattern:** ship in phases — (1) expand (add new, write to
  both), (2) backfill + switch reads, (3) contract (drop old) in a *later* deploy
  once no code references it. Migration authoring is `../rails-models/`
  [migrations.md](../../rails-models/references/migrations.md); this is the *deploy
  sequencing* of it.
- **Where migrate runs:** Kamal can run it as a pre-deploy hook / `kamal app exec`;
  PaaS uses a release phase ([orchestration-alternatives.md](orchestration-alternatives.md)).

## Rollback

```bash
kamal rollback           # re-point traffic to the previous release (fast — image still on host)
```

- Code rollback is instant (previous image). **Schema rollback is not symmetric** — a
  migration that ran is still applied. This is *why* migrations must be
  backward-compatible: rollback restores old code, which must still work on the new
  schema.
- Don't bundle a destructive (contract) migration with the deploy that needs
  rollback-ability; sequence it after you're confident.

## Common Pitfalls

- **Treating `/up` as a full health check** → it only proves boot; DB outages still
  return 200. Add a dependency check if you need one.
- **A breaking migration deployed with the code that needs it** → during cutover the
  old release errors; use expand/contract.
- **`NOT NULL`/rename/drop in one deploy** → old code breaks mid-cutover; phase it.
- **Assuming rollback undoes the migration** → it doesn't; design migrations so old
  code tolerates the new schema.
- **Health-check log spam** → noisy prod logs; silence the health path.

## Verify

```bash
grep -n "rails/health#show\|\"up\"" config/routes.rb && echo "✓ /up route present"
bin/rails server -d && sleep 3
curl -fsS -o /dev/null -w "up=%{http_code}\n" localhost:3000/up        # 200 when booted
kill "$(cat tmp/pids/server.pid)"
# Migration reversibility (expand/contract discipline) — down then up both succeed:
bin/rails db:migrate && bin/rails db:rollback && bin/rails db:migrate
# After a real deploy: the new release answers /up over TLS
# curl -fsS https://app.example.com/up
```

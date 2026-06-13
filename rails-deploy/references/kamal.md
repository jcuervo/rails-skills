# Kamal 2 (Recommended orchestration)

The Rails 8 default deploy tool: ships your Docker image to **your own servers**,
with **kamal-proxy** providing zero-downtime cutover and automatic SSL. No PaaS
lock-in, runs on cheap VPS hosts. Rails 8 generates `config/deploy.yml` +
`.kamal/secrets` for you.

## Quick Pattern

```bash
# Rails 8 generated config/deploy.yml, .kamal/secrets, Dockerfile, .dockerignore.
# First time on fresh servers:
kamal setup            # installs Docker on the hosts, boots kamal-proxy, deploys
# Subsequent releases:
kamal deploy           # build → push → boot new container → wait for /up → cut over
kamal rollback         # revert to the previous release
```

```yaml
# config/deploy.yml (fill in real values)
service: myapp
image: your-registry/myapp
servers:
  web:
    - 192.0.2.10                       # your host(s)
proxy:
  ssl: true
  host: app.example.com                # kamal-proxy terminates TLS + auto-provisions a cert
registry:
  server: ghcr.io
  username: your-user
  password:
    - KAMAL_REGISTRY_PASSWORD          # name resolved from .kamal/secrets
env:
  secret:
    - RAILS_MASTER_KEY                 # injected into the container from .kamal/secrets
  clear:
    SOLID_QUEUE_IN_PUMA: 1             # run jobs in Puma (../rails-jobs/)
```

```bash
# .kamal/secrets — sources the actual values (from ENV / a password manager / credentials)
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD
RAILS_MASTER_KEY=$(cat config/master.key)
```

## kamal-proxy (the Kamal 2 change)

- Kamal 2 replaced Traefik with **kamal-proxy**: fast **zero-downtime** deploys,
  **automatic SSL** (Let's Encrypt) via the `proxy.host`, and **multiple apps on one
  server**.
- The proxy health-checks the new container on `/up` before switching traffic — see
  [health-and-zero-downtime.md](health-and-zero-downtime.md).

## Workers, accessories, multiple roles

```yaml
servers:
  web:
    - 192.0.2.10
  job:                                  # a separate role for workers (../rails-jobs/)
    hosts: [192.0.2.11]
    cmd: bin/jobs
accessories:
  db:                                   # managed alongside, e.g. Postgres (or use a managed DB)
    image: postgres:16
    host: 192.0.2.12
    env: ...
```

- Run Solid Queue **in Puma** (`SOLID_QUEUE_IN_PUMA=1`, simplest) or as a **separate
  `job` role** for isolation/scaling (`../rails-jobs/`).
- **accessories** run supporting containers (DB, Redis); for managed DBs, point the
  app's `DATABASE_URL` at them instead.

## When to pick / not pick

- **Pick when** you want low-cost, lock-in-free deploys to your own hosts with
  zero-downtime + SSL handled, and you're comfortable owning a server. The Rails 8
  default.
- **Don't pick when** you want zero server ops (→ PaaS) or you're on a single
  throwaway host with no uptime needs (→ plain Compose) —
  [orchestration-alternatives.md](orchestration-alternatives.md).

## Common Pitfalls

- **Secrets committed in `deploy.yml`** → use `.kamal/secrets` (which itself reads
  from ENV/a manager) and keep real values out of the repo (`../rails-security/`).
- **`/up` failing on boot** → kamal-proxy never cuts over; the deploy "hangs" then
  rolls back. Fix boot errors / DB connectivity first
  ([health-and-zero-downtime.md](health-and-zero-downtime.md)).
- **No persistent volume for SQLite/Solid DBs** → data lost on redeploy; mount a
  volume or use a managed/accessory DB. SQLite-in-prod needs the file to survive.
- **Registry auth missing** → push fails; set `KAMAL_REGISTRY_PASSWORD`.
- **First deploy uses `kamal deploy` instead of `kamal setup`** → hosts aren't
  prepared; `setup` bootstraps them.

## Verify

```bash
kamal config 2>/dev/null | head -10                 # config renders (valid deploy.yml + secrets resolve)
kamal version 2>/dev/null
# After a deploy, the prod app is healthy + responds:
# kamal app exec "bin/rails runner 'puts Rails.env'"     # => production
# curl -fsS https://app.example.com/up                    # => 200 over TLS (kamal-proxy)
test -f .kamal/secrets && echo "✓ secrets file present (values sourced from ENV/manager)"
```

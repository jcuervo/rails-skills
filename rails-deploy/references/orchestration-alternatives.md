# Orchestration Alternatives — Docker Compose & PaaS

When Kamal 2 ([kamal.md](kamal.md)) isn't the fit. All three deploy the **same
production Docker image** ([docker-and-assets.md](docker-and-assets.md)); they differ
in how much ops you own.

## Docker Compose (self-managed, single host)

```yaml
# compose.yaml (production)
services:
  web:
    image: your-registry/myapp
    environment:
      RAILS_MASTER_KEY: ${RAILS_MASTER_KEY}
      DATABASE_URL: ${DATABASE_URL}
    ports: ["80:80"]                # Thruster listens on 80 inside the image
    restart: unless-stopped
```

- **Pick when** you want the simplest "pull the image and run it" on one host, and
  don't need zero-downtime cutover or managed SSL.
- **Don't pick when** you need zero-downtime deploys, auto-SSL, or multi-host —
  that's exactly what Kamal adds on top of Compose-style running.
- **You own** the reverse proxy/TLS (or rely on Thruster + a manual cert), restarts,
  and the "stop old / start new" gap (there *is* a gap without a proxy doing
  health-checked cutover).

## PaaS (Heroku / Render / Fly.io)

```
# Heroku-style: push, the platform builds + runs; Procfile or the Dockerfile drives it.
web: bin/thrust bin/rails server          # Thruster in front of Puma
release: bin/rails db:migrate             # release-phase migration (Heroku/Render)
```

- **Pick when** you want **minimal ops** — the platform handles servers, TLS, health
  checks, rolling deploys, scaling, logs. Fastest path to "it's live."
- **Don't pick when** cost at scale or lock-in matters, or you need control the
  platform doesn't expose. Kamal trades convenience for ownership.
- **Release-phase hooks** run `db:migrate` before traffic shifts (Heroku `release:`,
  Render preDeploy). Platform env vars carry secrets (`../rails-security/`).
- Most PaaS can build from the **Dockerfile** Rails generates, or their own
  buildpacks — prefer the Dockerfile for parity with other targets.

## Kubernetes (brief note)

For large/multi-service orgs already on K8s: deploy the same image as a Deployment +
Service, with the `/up` endpoint as the liveness/readiness probe and a HorizontalPodAutoscaler.
Full K8s is beyond this skill's scope — the image + `/up` are what it needs; cluster
config is a platform concern.

## Choosing

| Want | Pick |
|---|---|
| Own cheap servers, zero-downtime + SSL handled, no lock-in | Kamal 2 *(Recommended)* |
| Dead-simple single host, you manage proxy/SSL | Docker Compose |
| Minimal ops, managed everything, accept cost/lock-in | PaaS |
| Already a K8s shop | Kubernetes (image + `/up` probe) |

## Common Pitfalls

- **Compose without a health-checked cutover** → a request gap on every deploy;
  acceptable for low-stakes, not for zero-downtime (use Kamal).
- **PaaS without a release-phase migration** → new code hits an un-migrated DB;
  configure the release hook.
- **Different build per target** (buildpack here, Dockerfile there) → "works on one,
  not the other"; standardize on the Dockerfile image.
- **Secrets in compose.yaml / committed** → use platform env / a secrets file
  (`../rails-security/`).

## Verify

```bash
# Compose: the image boots and /up is healthy
docker compose -f compose.yaml up -d 2>/dev/null && sleep 8
curl -fsS -o /dev/null -w "up=%{http_code}\n" localhost/up 2>/dev/null      # expect 200
docker compose -f compose.yaml down 2>/dev/null
# PaaS: a release migration hook is configured (Procfile/render.yaml/fly.toml)
grep -rniE "release:|preDeploy|release_command" Procfile render.yaml fly.toml 2>/dev/null || echo "(configure a release-phase db:migrate)"
```

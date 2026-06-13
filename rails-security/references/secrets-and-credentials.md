# Secrets & Credentials

Keep secrets (API keys, tokens, encryption keys, DB passwords) **out of the repo and
out of logs**. The menu pick is **encrypted credentials** (Rails-native); ENV for
platform-injected config; an external manager for org-scale rotation/audit.

## Encrypted credentials (Recommended)

```bash
bin/rails credentials:edit                       # opens the decrypted file in $EDITOR, re-encrypts on save
bin/rails credentials:edit --environment production   # per-environment credentials
```
```yaml
# (decrypted view) config/credentials.yml.enc
secret_key_base: ...
stripe:
  api_key: sk_live_...
```
```ruby
Rails.application.credentials.dig(:stripe, :api_key)
```

- `config/credentials.yml.enc` is **encrypted at rest** and safe to commit; the
  **`master.key`** (or `RAILS_MASTER_KEY` env) decrypts it and is **never committed**
  (`.gitignore`d by default).
- **Pick when** you want versioned, encrypted-in-repo secrets with no extra infra.
  The Rails default.
- **Don't pick when** your platform mandates ENV injection, or you need centralized
  rotation/audit across many apps (external manager).
- Per-environment credentials keep prod keys out of dev; share the prod key only with
  prod/CI.

## ENV vars (+ dotenv in dev)

```ruby
ENV.fetch("STRIPE_API_KEY")        # fetch (raises if missing) over ENV[] (nil surprises)
```
```ruby
# Gemfile (dev/test): gem "dotenv-rails"  → loads .env (which is .gitignore'd)
```

- **Pick when** your host (Heroku/Kamal/K8s/CI) injects config as environment
  variables — the 12-factor approach.
- **Don't pick when** you'd rather keep secrets versioned with the code (credentials)
  or need an audit trail.
- **`.env` must be git-ignored**; commit a `.env.example` with **blank** values as
  documentation. Use `ENV.fetch` so a missing var fails loudly, not silently as nil.

## External secrets manager

- Vault / AWS Secrets Manager / GCP Secret Manager / Doppler: fetched at boot or
  injected by the platform; central rotation, access policies, and audit logs.
- **Pick when** you operate many services, need rotation/audit/least-privilege, or
  compliance requires it.
- **Don't pick when** a single app's needs are met by credentials/ENV — it adds infra
  and a fetch dependency at boot.
- Injection at deploy time is `../rails-deploy/` (Kamal secrets, K8s secrets).

## Cross-cutting rules

- **Never commit a real secret.** `secret_key_base`, API keys, passwords belong in
  credentials/ENV/manager — not in source, not in `config/*.yml`, not in tests.
- **Never log secrets.** Rails filters params named like `password`/`token` via
  `config.filter_parameters` — extend it for your secret-bearing params so they're
  `[FILTERED]` in logs.
- **Rotate on exposure.** If a key lands in a commit/log, rotate it — removing the
  commit isn't enough (it's in history/forks).

## Common Pitfalls

- **Committed `.env` / `master.key`** → secret leak; git-ignore both, rotate if leaked.
- **`ENV["X"]` (nil) instead of `ENV.fetch("X")`** → silent misconfig; a missing
  secret should crash boot, not run insecurely.
- **Secrets echoed into logs** (a `params` dump, an exception with the token) →
  extend `filter_parameters`; scrub error reports (`../rails-deploy/`).
- **Same prod secret in dev** → blast radius; use per-environment credentials.
- **Storing secrets in JS/`importmap`/frontend** → anything shipped to the browser is
  public; only server-side holds secrets.

## Verify

```bash
bin/rails runner "p Rails.application.credentials.secret_key_base.present?"   # credentials decrypt
git check-ignore .env config/master.key 2>/dev/null && echo "✓ secrets git-ignored" || echo "⚠ ensure .env/master.key are ignored"
# Nothing secret-looking is committed:
git grep -nE "(secret_key_base|sk_live_|api[_-]?key|password)\s*[:=]\s*[\"'][A-Za-z0-9]{8}" -- . ':!*.enc' ':!*.example' 2>/dev/null | head
# Sensitive params are filtered in logs:
bin/rails runner "p Rails.application.config.filter_parameters" | grep -iE "password|token|secret"
```

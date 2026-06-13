# CI/CD

Continuous integration (test + scan on every push) and continuous deployment (ship
green builds). The menu pick is **GitHub Actions** — Rails 8 generates
`.github/workflows/ci.yml` for you. This skill owns the **pipeline**; the *tests* it
runs are `../rails-testing/` and the *scanners* are `../rails-security/`.

## The generated CI (extend it)

Rails 8's `.github/workflows/ci.yml` already runs, on push/PR:

- **Brakeman** (static security scan — `../rails-security/`),
- **RuboCop** (style),
- **importmap audit** (JS deps, for importmap apps),
- the **test suite** (`bin/rails test` / `test:system` — `../rails-testing/`),
  with a DB service for tests.

Keep these as the **gate**; add a deploy job that runs only when they pass.

## Add a deploy job (CD)

```yaml
# .github/workflows/ci.yml (add a deploy job gated on the test/scan jobs)
  deploy:
    needs: [test, scan]                 # only deploy if CI is green
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with: { bundler-cache: true }
      - run: gem install kamal
      - run: kamal deploy               # or: kamal redeploy
        env:
          KAMAL_REGISTRY_PASSWORD: ${{ secrets.KAMAL_REGISTRY_PASSWORD }}
          RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
```

- **Gate deploy on green** (`needs:` the test + scan jobs) — never ship a red build.
- **Secrets** live in the CI provider's secret store (GitHub repo/environment
  secrets), injected as env — never in the YAML (`../rails-security/`).
- **Branch/environment gating:** deploy from `main` (or a tagged release); use GitHub
  **Environments** with required reviewers for production approval.

## Pipeline shape (any provider)

```
push → [ lint | scan(Brakeman, bundler-audit) | test(unit, system) ]  → (all green) → build image → deploy → smoke /up
```

- **GitLab CI / CircleCI / others:** same stages, different YAML. Pick to match your
  VCS/host; the *content* (run the scanners, run the suite, gate deploy) is identical.
- **Caching:** cache gems (`bundler-cache`) and Docker layers for fast pipelines.
- **Smoke after deploy:** curl `/up` on the new release as a final step so a broken
  deploy fails loudly ([health-and-zero-downtime.md](health-and-zero-downtime.md)).

## Common Pitfalls

- **Deploy not gated on tests/scan** → red builds reach production; `needs:` the gate.
- **Secrets in workflow YAML** → leaked credentials; use the provider's secret store.
- **Deploying from every branch** → accidental prod deploys; gate on `main`/tags +
  an Environment approval.
- **No post-deploy smoke check** → a deploy "succeeds" but the app is down; curl
  `/up` after.
- **CI green but scanners disabled** → security regressions merge; keep Brakeman +
  bundler-audit in the gate (`../rails-security/` scanning-and-audit.md).

## Verify

```bash
ls .github/workflows/ci.yml && echo "✓ CI workflow present"
grep -niE "brakeman|rubocop|rails test|rspec|importmap audit" .github/workflows/ci.yml   # the gate runs scan + tests
grep -niE "deploy|kamal" .github/workflows/*.yml 2>/dev/null || echo "(add a gated deploy job)"
# Locally reproduce the gate before pushing:
bin/rubocop -q 2>/dev/null; bin/brakeman -q --no-pager 2>/dev/null | tail -3; bin/rails test 2>/dev/null | tail -3
```

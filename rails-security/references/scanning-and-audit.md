# Vulnerability Scanning & the rails-audit Relationship

Automated checks that catch security regressions as code/dependencies change:
**Brakeman** (static analysis of your code), **bundler-audit** (known CVEs in your
gems), and **importmap audit** (JS deps). Run them in CI so every change is checked.
This is the proactive, *continuous* counterpart to the upstream **`rails-audit`**
skill's deeper *retrospective* review.

## Brakeman (static analysis — ships with Rails 8)

```bash
# New Rails 8 apps include brakeman + a CI job (unless --skip-brakeman / --skip-ci).
bin/brakeman                       # full scan
bin/brakeman -q --no-pager         # quiet, CI-friendly
bin/brakeman -I                    # interactively manage false positives → config/brakeman.ignore
```

- Finds SQL injection, mass-assignment, XSS, unsafe redirects, command injection,
  insecure config — by analyzing source (no app boot needed).
- **Run in CI and fail the build on new warnings.** Triage false positives into
  `config/brakeman.ignore` so the signal stays clean.
- Pairs with the manual hardening in
  [csrf-and-mass-assignment.md](csrf-and-mass-assignment.md) — Brakeman catches what
  you missed.

## bundler-audit (dependency CVEs)

```bash
# Gemfile (dev/test): gem "bundler-audit"
bundle exec bundler-audit check --update     # refresh the advisory DB, then check Gemfile.lock
```

- Flags gems with **known published vulnerabilities** (CVEs) and insecure sources.
  Your own code can be perfect while a dependency is the hole.
- Run in CI (with `--update`) so a newly-disclosed CVE in a pinned gem fails the
  build until you bump it.
- Complement with **Dependabot**/Renovate for automated dependency-bump PRs (repo/CI
  config — `../rails-deploy/`).

## importmap audit (JS dependencies)

```bash
bin/importmap audit            # checks pinned JS packages for known vulnerabilities (importmap apps)
```

- For import-map apps (`../rails-hotwire/` assets-and-css.md), this audits the pinned
  JS. Bundler apps audit via `npm/yarn audit` instead.

## Wire it into CI

The Rails 8 default `.github/workflows/ci.yml` already runs Brakeman; add
bundler-audit + importmap audit as steps so all three gate merges. The **CI pipeline
structure** is `../rails-deploy/`; this skill just ensures the security steps exist
and pass.

## Relationship to the upstream `rails-audit` skill

| | This skill (`rails-security`) | `rails-audit` (upstream) |
|---|---|---|
| Mode | **Proactive / continuous** — harden as you build, scan in CI | **Retrospective** — deep review of an existing app |
| Output | Controls in place + green scanners | A findings report (coding standards, security, perf, architecture) |
| Cadence | Every commit | Periodic / on-demand |

- `rails-audit` is **referenced, not duplicated** — it's a separately-maintained
  upstream skill bundled as a git submodule. Its **`security-checklist.md`** is the
  **canonical rubric** this skill builds toward: the controls here (rate limiting,
  CSP, secrets, CSRF, mass-assignment, scanning) map to that checklist's items.
- **Hand off to `rails-audit`** for a full retrospective review (PDF/report) of an
  existing codebase; come **back here** to fix what it finds, proactively.

## Common Pitfalls

- **Scanners not in CI** → security regressions merge unnoticed; gate on Brakeman +
  bundler-audit.
- **Ignoring Brakeman by silencing instead of fixing** → real holes hidden in
  `brakeman.ignore`; only ignore confirmed false positives.
- **bundler-audit without `--update`** → stale advisory DB misses new CVEs.
- **Auditing code but not dependencies (or vice-versa)** → both are attack surfaces;
  run all three.
- **Treating a green scan as "secure"** → tools catch known patterns, not design
  flaws; combine with the manual hardening here and a periodic `rails-audit` review.

## Verify

```bash
bin/brakeman -q --no-pager 2>/dev/null | tail -8                       # runs; review warnings
bundle exec bundler-audit check --update 2>/dev/null | tail -8         # advisory check (no known CVEs = clean)
bin/importmap audit 2>/dev/null | tail -5 || echo "(bundler app: use npm/yarn audit)"
grep -rqi "brakeman" .github/workflows/ 2>/dev/null && echo "✓ Brakeman in CI" || echo "⚠ add scanners to CI (../rails-deploy/)"
```

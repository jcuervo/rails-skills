---
name: rails-security
description: Proactively harden a Rails 8.1 app at build time — rate limiting (Rails 8 built-in rate_limit or rack-attack), secrets/credentials management (encrypted credentials, ENV, external managers), Content Security Policy and security headers, CSRF protection, strong-parameters/mass-assignment as a security control, and dependency/static scanning (Brakeman, bundler-audit, importmap audit). Menu-driven for the genuine choices (rate limiting, secrets) with a Recommended default; detects what's already configured first, branches on config.api_only (CSRF/CSP differ for token APIs), and verifies each control actually blocks what it should. This is BUILD-secure; for a retrospective FIND-insecure audit, use the upstream rails-audit skill (its security checklist is the rubric this skill builds toward). Apply when adding rate limiting, handling secrets, setting CSP/headers, hardening forms/params, or wiring vulnerability scanning.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[control-or-area]"
---

# rails-security

## Purpose

**Proactive, build-time hardening** — the controls you wire *as you build* so the
app is secure by construction: rate limiting, secrets management, Content Security
Policy + security headers, CSRF, strong-params/mass-assignment hygiene, and
dependency/static scanning. It is the **build-secure** complement to the upstream
**`rails-audit`** skill, which is the **find-insecure** retrospective review
(coding standards + security checklist → report). This skill treats `rails-audit`'s
`security-checklist.md` as the **canonical rubric it builds toward** — same
definition of "good," applied proactively here and retrospectively there. Security
is also embedded in siblings (`../rails-auth/`, `../rails-api/`, `../rails-deploy/`);
this skill owns the cross-cutting hardening that isn't any one of those.

## When to Apply

Use this skill when the task is:

- Adding rate limiting / throttling (login, API, expensive endpoints)
- Managing secrets and credentials (API keys, tokens, encryption keys)
- Setting a Content Security Policy or security response headers
- Hardening CSRF protection and strong-parameters/mass-assignment
- Wiring vulnerability scanning (Brakeman, bundler-audit, importmap audit) into the workflow

Do **not** use this skill when the task is:

- The authentication/authorization *mechanism* → read `../rails-auth/SKILL.md` (this skill hardens around it: rate-limit login, secure session cookies)
- A **retrospective security audit** of an existing app → use the upstream **`rails-audit`** skill (this skill is proactive; that one is the review)
- The strong-params *mechanics* (`params.expect`/`permit`) → read `../rails-controllers/SKILL.md` (here it's the security lens on them)
- API CORS / token surface → read `../rails-api/SKILL.md`
- Storing/serving uploaded files securely (signed URLs, content-type) → read `../rails-storage/SKILL.md`
- Production TLS, secret injection at deploy, WAF/CDN, image-CVE scanning → read `../rails-deploy/SKILL.md`
- Security-focused tests → read `../rails-testing/SKILL.md`

## Detect Before You Generate

```bash
grep -rnE "rate_limit|Rack::Attack" app/controllers config/initializers 2>/dev/null   # rate limiting present?
ls config/credentials.yml.enc config/credentials/*.yml.enc 2>/dev/null                 # encrypted credentials
grep -rnE "content_security_policy|force_ssl|stricter?_csrf|protect_from_forgery" config/ app/controllers 2>/dev/null
grep -E "^    (rack-attack|brakeman|bundler-audit|secure_headers)" Gemfile.lock
grep -nE "api_only" config/application.rb                                              # CSRF/CSP differ for token APIs
```

- If a control is **already configured**, extend it — don't re-ask or re-scaffold.
- New Rails 8 apps ship **Brakeman** and a **CI workflow** (unless `--skip-brakeman`/
  `--skip-ci`) — detect and build on them rather than adding parallel tooling.
- `config.api_only == true` → CSRF token protection and CSP for HTML don't apply the
  same way; APIs rely on token auth (`../rails-auth/`, `../rails-api/`) and CORS.
- Honor any security tooling recorded in `STACK.md`.

## Menu

Two genuine menus (the rest — CSP/headers, CSRF, scanning — are hardening practices,
not pick-one choices). Each via `AskUserQuestion`, Rails-default marked
**Recommended**; ask unless already configured.

### Rate limiting
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Rails 8 built-in `rate_limit`** *(Recommended)* | `rate_limit to:, within:` in a controller; cache-backed, per-IP, 429. No gem. Great for login/endpoint throttling. | [rate-limiting.md](references/rate-limiting.md) |
| rack-attack | Middleware with richer rules: blocklists/allowlists, fail2ban, by-arbitrary-key throttles. Pick for complex/global policies. | [rate-limiting.md](references/rate-limiting.md) |

### Secrets management
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Encrypted credentials** *(Recommended)* | Rails-native `credentials.yml.enc` + `master.key`/`RAILS_MASTER_KEY`; versioned, encrypted at rest. | [secrets-and-credentials.md](references/secrets-and-credentials.md) |
| ENV vars (+ dotenv in dev) | 12-factor; injected by the platform. Ubiquitous; you own keeping them out of logs/repos. | [secrets-and-credentials.md](references/secrets-and-credentials.md) |
| External secrets manager | Vault / AWS Secrets Manager / GCP Secret Manager; central rotation/audit. For larger orgs. | [secrets-and-credentials.md](references/secrets-and-credentials.md) |

## Decision Flow

- **Rate limiting:** default to the **Rails 8 built-in `rate_limit`** for endpoint/
  login throttling — no dependency, cache-backed. Choose **rack-attack** when you
  need global bl(/allow)lists, fail2ban-style escalation, or throttles keyed on
  things other than IP. Always rate-limit auth endpoints (`../rails-auth/`).
- **Secrets:** **encrypted credentials** by default (versioned + encrypted in-repo,
  key out of repo). Use **ENV** when your platform injects config that way (still
  never commit it). **External manager** when you need central rotation/audit across
  many services. Never commit a real secret; never log one.
- **CSP/headers:** ship a Content Security Policy and the standard hardening headers
  ([csp-and-headers.md](references/csp-and-headers.md)) — start in report-only, then
  enforce. `force_ssl` in production.
- **CSRF/params:** keep `protect_from_forgery` on for Web; treat strong params as a
  security boundary (mass-assignment) — the lens on the mechanics
  `../rails-controllers/` owns ([csrf-and-mass-assignment.md](references/csrf-and-mass-assignment.md)).
- **Scanning:** run **Brakeman** (static) + **bundler-audit** (dependency CVEs) +
  importmap audit in CI ([scanning-and-audit.md](references/scanning-and-audit.md));
  for a full retrospective review, hand to the upstream **`rails-audit`** skill.

## Problem → Reference

| Task | Read |
|---|---|
| Rate limiting (Rails 8 built-in vs rack-attack) | [references/rate-limiting.md](references/rate-limiting.md) |
| Secrets/credentials (encrypted credentials / ENV / external) | [references/secrets-and-credentials.md](references/secrets-and-credentials.md) |
| Content Security Policy + security headers + force_ssl | [references/csp-and-headers.md](references/csp-and-headers.md) |
| CSRF protection + strong-params/mass-assignment as a security control | [references/csrf-and-mass-assignment.md](references/csrf-and-mass-assignment.md) |
| Brakeman, bundler-audit, importmap audit + the rails-audit relationship | [references/scanning-and-audit.md](references/scanning-and-audit.md) |

## Verify

A hardening change is unfinished until the control actually blocks what it should:

```bash
bin/rails server -d && sleep 3
BASE=localhost:3000
# Rate limit kicks in after the threshold (returns 429):
for i in $(seq 1 15); do curl -s -o /dev/null -w "%{http_code} " -X POST $BASE/session; done; echo   # expect 429s after the limit
# Security headers present:
curl -s -I $BASE/ | grep -iE "content-security-policy|x-frame-options|strict-transport-security|x-content-type-options"
kill "$(cat tmp/pids/server.pid)"
# Scanners run clean (or surface findings):
bin/brakeman -q --no-pager 2>/dev/null | tail -5
bundle exec bundler-audit check --update 2>/dev/null | tail -5
# No secrets committed:
git grep -nE "(secret_key_base|password|api[_-]?key)\s*[:=]\s*[\"'][A-Za-z0-9]" -- . ':!*.enc' 2>/dev/null | head
```

Then route to `../rails-deploy/` (TLS, secret injection, image scanning), `../rails-testing/`
(security tests), and the upstream **`rails-audit`** for a full review. Record the
rate-limit + secrets picks in `STACK.md`.

---
name: rails-auth
description: Add authentication AND authorization to a Rails 8.1 app, unified in one skill. Authentication menu — Rails 8 built-in generator with has_secure_password (Recommended), Devise, Rodauth (rodauth-rails), or authentication-zero — plus social login via OmniAuth and API token strategies. Authorization menu — Pundit (Recommended), Action Policy, or CanCanCan. Menu-driven with a Recommended default per menu; detects any installed auth/authz gem and config.api_only first (token surface for APIs, session/cookie flow for Web), gives per-solution integration guidance (install → wire → routes/controllers → views or token surface → tests), and verifies a real login/authorize round-trip. Apply when adding login/logout, password reset, sign-up, social login, API tokens, or role/permission checks.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[authn-or-authz]"
---

# rails-auth

## Purpose

The single home for **who you are** (authentication) and **what you may do**
(authorization). It offers the top solutions for each as a menu and gives
per-solution integration guidance: install → wire → routes/controllers → views (Web)
or token surface (API) → tests. It branches on `config.api_only` — Web apps use
cookie-backed sessions and login views; API apps use a token surface (the HTTP
contract for which is `../rails-api/` token-auth-surface.md; the *mechanism* is
here). It is the mechanism that `../rails-api/`'s token surface and
`../rails-hotwire/`'s login forms defer to.

## When to Apply

Use this skill when the task is:

- Adding login/logout, sessions, password reset, or sign-up/registration
- Social / third-party login (Google, GitHub) via OmniAuth
- Issuing/verifying API tokens (Bearer auth) for an API
- Authorization: roles, permissions, policy checks ("can this user do X?")
- Securing controllers/actions behind authentication or a policy

Do **not** use this skill when the task is:

- The HTTP *surface* of API token auth (header format, 401/403 shape) → read `../rails-api/` (`references/token-auth-surface.md`); the *mechanism* is here
- Building the login *form* markup / Turbo behavior → read `../rails-hotwire/SKILL.md` (this skill provides the controllers/flow it posts to)
- Routes, controllers, `before_action` plumbing in general → read `../rails-controllers/SKILL.md`
- The `User` model's data/validations/associations → read `../rails-models/SKILL.md`
- Sending the password-reset email (delivery/templates) → read `../rails-mailers/SKILL.md`
- CSRF, rate-limiting login, secrets, security headers → read `../rails-security/SKILL.md`
- Testing the auth flow → read `../rails-testing/SKILL.md`
- A retrospective audit of auth posture → use the `rails-audit` skill

## Detect Before You Generate

```bash
grep -nE "api_only" config/application.rb                          # token surface vs cookie sessions
grep -E "^    (devise|rodauth|rodauth-rails|authentication-zero|omniauth|pundit|action_policy|cancancan|bcrypt|jwt)" Gemfile.lock   # installed picks
ls app/controllers/concerns/authentication.rb app/models/session.rb 2>/dev/null   # Rails 8 built-in generator already run?
ls app/policies 2>/dev/null                                        # Pundit/Action Policy present?
grep -rnE "has_secure_password|devise|rodauth" app/models 2>/dev/null
```

- If an auth/authz gem is **already installed**, use it silently — match it, don't
  re-ask the menu.
- If `app/controllers/concerns/authentication.rb` + `app/models/session.rb` exist,
  the **Rails 8 built-in generator already ran** — extend it (add registration,
  policies), don't regenerate.
- `config.api_only == true` → token surface (see [api-tokens.md](references/api-tokens.md));
  Web → session/cookie flow.
- Honor any auth pick recorded in `STACK.md` by `../rails-scaffold/`.

## Menu

Two independent menus — pick one authn **and** one authz. Each via
`AskUserQuestion`, Rails-default marked **Recommended**; ask unless installed.

### Authentication
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Rails 8 built-in generator** *(Recommended)* | `bin/rails generate authentication` — real code you own, DB-backed sessions, `has_secure_password`, password reset. Minimal, no gem lock-in. You add sign-up. | [rails8-authentication.md](references/rails8-authentication.md) |
| Devise | Batteries-included (registration, confirmable, recoverable, lockable). Fast to ship; heavier, more magic, opinionated. | [authentication-alternatives.md](references/authentication-alternatives.md) |
| Rodauth (rodauth-rails) | Most complete + secure feature set (MFA, WebAuthn, account verification) with a different architecture. Steeper, very powerful. | [authentication-alternatives.md](references/authentication-alternatives.md) |
| authentication-zero | Generates an opinionated, modern auth scaffold (like the built-in, with more features pre-wired). Code you own. | [authentication-alternatives.md](references/authentication-alternatives.md) |

Add-ons (any authn): **social login** → [social-omniauth.md](references/social-omniauth.md) · **API tokens** → [api-tokens.md](references/api-tokens.md).

### Authorization
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Pundit** *(Recommended)* | Plain Ruby policy objects (one per model); explicit, testable, minimal magic. The common default. | [authorization.md](references/authorization.md) |
| Action Policy | Policy objects with more built-ins (caching, scoping, tests, GraphQL); scales to complex rules. | [authorization.md](references/authorization.md) |
| CanCanCan | Central `Ability` class defining all rules in one place; concise for simple apps, can sprawl. | [authorization.md](references/authorization.md) |

## Decision Flow

- **Authentication:** default to the **Rails 8 built-in generator** — you own the
  code, no gem lock-in, and it covers sign-in + password reset (you add
  registration). Choose **Devise** when you want registration/confirmable/lockable
  out of the box and accept the abstraction. Choose **Rodauth** when you need
  serious features (MFA, WebAuthn, passwordless) and will invest in its model.
  **authentication-zero** when you like owning the code but want more pre-wired than
  the built-in. Don't run two auth systems.
- **Web vs API:** Web → cookie-backed sessions + login views. API → token surface
  ([api-tokens.md](references/api-tokens.md)); the HTTP contract (401/403,
  `WWW-Authenticate`) is `../rails-api/`. The same `User`/credential model can back
  both.
- **Social login** is **additive** — layer OmniAuth onto any authn choice; it
  doesn't replace password auth ([social-omniauth.md](references/social-omniauth.md)).
- **Authorization:** **Pundit** for explicit per-model policies (default). **Action
  Policy** when rules get complex and you want scoping/caching/test helpers.
  **CanCanCan** for small apps wanting all rules in one `Ability`. AuthZ is separate
  from authN — pick both.
- **AuthN ≠ AuthZ:** 401 = not authenticated; 403 = authenticated but not permitted.
  Keep the two layers distinct.

## Problem → Reference

| Task | Read |
|---|---|
| Rails 8 built-in auth: generate, sign-in, password reset, add sign-up | [references/rails8-authentication.md](references/rails8-authentication.md) |
| Devise / Rodauth / authentication-zero: when + how to integrate | [references/authentication-alternatives.md](references/authentication-alternatives.md) |
| Authorization: Pundit / Action Policy / CanCanCan policies | [references/authorization.md](references/authorization.md) |
| Social / third-party login via OmniAuth | [references/social-omniauth.md](references/social-omniauth.md) |
| API token auth (opaque / JWT), tying into the API token surface | [references/api-tokens.md](references/api-tokens.md) |

## Verify

Auth is unfinished until a real credential round-trips and an unauthorized request
is actually blocked:

```bash
bin/rails db:migrate                                   # auth tables exist (users/sessions)
bin/rails server -d && sleep 3
BASE=localhost:3000
# Web: protected page redirects to login when unauthenticated
curl -s -o /dev/null -w "protected=%{http_code}\n" $BASE/dashboard          # expect 302 → login (or 401 API)
# A bad login is rejected; a good login establishes a session/token (see each reference's Verify)
kill "$(cat tmp/pids/server.pid)"
# Authorization: a policy denies a forbidden action (rescue is intentionally lenient — prints a hint if no policy yet)
bin/rails runner "p PostPolicy.new(User.new, Post.new).update? rescue p 'add a policy'" 2>/dev/null
```

Then route to `../rails-testing/` for auth/policy specs, `../rails-security/` for
login rate-limiting + secrets, and record the authn/authz picks in `STACK.md`.

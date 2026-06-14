# New-App Interview — Orchestration Spine

This is the end-to-end flow `rails-scaffold` runs for a greenfield app: gather
answers in order, assemble one `rails new` command, run it, verify it boots, then
route to siblings. Each step links to the menu that owns its options.

## 0. Preconditions

```bash
# Is this already a Rails app? If config/application.rb exists, STOP and read existing-app.md.
test -f config/application.rb && echo "EXISTING APP — do not scaffold" || echo "clean — proceed"

ruby -v          # expect 4.0.x (suite target). Note actual; do not block on mismatch.
rails -v         # expect Rails 8.1.x. If older, the user may want ../rails-upgrade after; for NEW apps, install/use 8.1.
rails new --help # authoritative flags + defaults for the INSTALLED generator
```

If Rails isn't installed or is older than 8.1, surface that and offer to
`gem install rails` (suggest the user run `! gem install rails -v "~> 8.1"` so the
install output lands in the session). Do not silently scaffold on an old Rails.

**Detect-before-ask also applies to a partially-initialized dir.** If the target
directory is non-empty and already carries config that implies a pick (a
`Gemfile` pinning a database adapter, a `config/importmap.rb`, a CSS gem), treat
that pick as made and **skip its menu** — only ask the concerns that are still
unmet. A truly empty dir asks everything.

## 1. App type → [app-type.md](app-type.md)

Ask first; it gates later questions.

- **API-only** → adds `--api`; **skips** steps 3 (CSS/JS/Hotwire) entirely.
- **Full-stack (Hotwire)** → no `--api`; asks the full frontend stack in step 3.
- **API + separate frontend** → `--api` for this repo; note the frontend is a
  separate project (point at `rn-scaffold-expo` for RN, or a JS SPA), and the API
  app still skips step 3.

Record `config.api_only` intent — every sibling skill branches on it.

## 2. Database → [database-selection.md](database-selection.md)

Pick one adapter → `-d <adapter>`. SQLite is the Rails 8 default and is
production-capable; Postgres is the common pick when a team wants concurrency,
extensions, or managed hosting. This sets the `rails new` flag only — deep schema,
migrations, indexing, multi-DB, and full-text search are owned by
`../rails-models/SKILL.md`.

## 3. Frontend stack *(full-stack only)* → [frontend-stack.md](frontend-stack.md)

Skipped entirely for API-only. Three sub-picks, each its own `AskUserQuestion`:

- **Asset pipeline:** Propshaft *(Recommended)* / Sprockets / none → `-a`
- **CSS:** Tailwind / Bootstrap / Bulma / PostCSS / Sass / plain → `-c <css>` (omit for plain)
- **JavaScript:** import maps *(Recommended)* / esbuild / Bun / Rollup / Webpack → `-j <js>`
- **Hotwire:** keep Turbo+Stimulus *(Recommended)* or `--skip-hotwire`

Deep wiring (ViewComponent/Phlex, Stimulus controllers, Turbo Streams,
Action Cable) is owned by `../rails-hotwire/SKILL.md`. Here we only set flags.

## 4. Omakase backends → [omakase-defaults.md](omakase-defaults.md)

The Rails 8 "batteries": ask keep-or-skip for each as a single multi-select where
sensible.

- **Solid trio** (Queue/Cache/Cable, DB-backed, no Redis): keep *(Recommended)* or `--skip-solid`
- **Kamal 2** deploy config: keep *(Recommended)* or `--skip-kamal`
- **Thruster** (HTTP/2 + asset caching in front of Puma): keep *(Recommended)* or `--skip-thruster`
- **CI workflow** (`.github/workflows/ci.yml`): keep *(Recommended)* or `--skip-ci`
- **RuboCop** (omakase style): keep *(Recommended)* or `--skip-rubocop`
- **Brakeman** (security scan): keep *(Recommended)* or `--skip-brakeman`
- **Dev container**: add `--devcontainer` or not *(default: not generated)*

Defer the *operational* depth (Kamal secrets, deploy targets, observability) to
`../rails-deploy/SKILL.md`. Job backend alternatives (Sidekiq/GoodJob) are owned
by `../rails-jobs/SKILL.md` — if the user wants Sidekiq, still keep Solid out
with `--skip-solid` here, then let `rails-jobs` install Sidekiq.

## 5. Test framework → defer to [`../rails-testing/SKILL.md`](../../rails-testing/SKILL.md)

Ask Minitest *(Recommended, default)* vs RSpec.

- **Minitest** → no flag; Rails generates `test/`.
- **RSpec** → add `-T` (skip Minitest scaffolding) so `rails-testing` installs
  `rspec-rails` cleanly without orphaned `test/` files.

Record the choice; `rails-testing` does the actual wiring after generation.

## 6. Auth → defer to [`../rails-auth/SKILL.md`](../../rails-auth/SKILL.md)

No `rails new` flag. Just record the intended approach (Rails 8 built-in generator
*(Recommended)* / Devise / Rodauth / authentication-zero, plus authZ pick) so the
hand-off invokes `rails-auth` with context. Do not run any auth generator here.

## 7. Compliance posture *(optional)*

No `rails new` flag, **default `none`** — this step never scaffolds anything here.
Ask **one optional** question: *does this app carry data-protection, audit, or
accessibility obligations?* Offer the tags as a skippable multi-select:

- **none** *(default)* — nothing recorded; compliance primitives stay **reactive**
  (applied only if explicitly requested later). Keeps the unopinionated default.
- **gdpr / ccpa** — personal-data rights: encryption at rest, consent, data export,
  erasure/anonymization → owned by `../rails-models/` (`data-protection.md`).
- **audit-trail** — who-changed-what change logging → `../rails-models/`
  (`audit-trail.md`).
- **soc2** — primarily an audit-trail / access-logging obligation →
  `../rails-models/` (`audit-trail.md`).
- **hipaa** — protected health info implies *both* an audit trail and personal-data
  handling; siblings treat `hipaa` as a trigger for both primitives.
- **wcag / section508** *(full-stack only)* — accessibility obligation → escalates
  `../rails-hotwire/` (`accessibility.md`) to a proactive per-view pass + an axe test
  gate. (A11y is baseline for any full-stack app regardless; this tag *enforces* it.)

Write the answer to the `STACK.md` `Compliance` row (see skeleton in step 10). It
generates nothing now — it only flips the relevant siblings from "ask when prompted"
to "**proactively offer** the primitive at the right moment." This is a *switch*, not
a mandate: the agent still asks before wiring anything, and `none`/absent preserves
the suite's default behavior.

## 8. Assemble + run

Build the command from the recorded answers. Example shapes:

```bash
# Full-stack, Postgres, Tailwind, import maps, RSpec, keep omakase:
rails new blog -d postgresql -c tailwind -j importmap -T

# API-only, SQLite (default), keep Solid + Kamal, Minitest:
rails new api_app --api

# Full-stack, MySQL, esbuild + Bootstrap, no Solid (Sidekiq later), no Kamal (Heroku):
rails new shop -d mysql -j esbuild -c bootstrap --skip-solid --skip-kamal
```

Rules for assembly:
- Omit a flag whenever the user picked the default (e.g. SQLite, Propshaft,
  import maps) — a shorter command is clearer. **Exception:** pass an explicit
  `-d` even for SQLite when the team is likely to switch DBs, so intent is recorded.
- `--api` and any frontend flag (`-c`, `-j`, `--skip-hotwire`) are mutually
  exclusive — never emit both.
- Run from the parent dir; the app name is the directory created.

Then:

```bash
cd <app-name>
# Rails 8 generates bin/setup; fall back if a slim template omitted it.
if [ -x bin/setup ]; then bin/setup; else bundle install && bin/rails db:prepare; fi
```

## 9. Verify (gate — all must pass)

```bash
bin/rails about                       # confirm Ruby 4.0.x / Rails 8.1.x
bin/rails db:prepare                  # schema load/create on chosen adapter
bin/rails server -d                   # boot daemonized (robust in non-interactive shells)
sleep 3 && curl -fsS localhost:3000/up && echo " ✓ /up 200"   # health check
kill "$(cat tmp/pids/server.pid)"     # stop via pidfile, not job control
```

- **Full-stack:** also `curl -fsS localhost:3000/` returns the welcome/root page.
- **API-only:** generate or hit a trivial endpoint to confirm JSON renders.
- **Tests:** `bin/rails test` (Minitest) or `bundle exec rspec` (after
  `rails-testing` wires it) runs green.
- **Lint/scan:** `bin/rubocop` and `bin/brakeman` clean, unless skipped.

If `/up` is not 200, the app is not bootable — fix before routing onward.

## 10. Record + route

Write `STACK.md` at the app root capturing every pick (one row per concern,
including the flag used). Siblings read it to avoid re-asking. Then route in
dependency order:

```
rails-models → (rails-controllers + rails-api | rails-hotwire) → rails-auth
  → rails-testing → rails-jobs / rails-mailers / rails-storage
  → rails-security / rails-performance → rails-deploy
```

Suggested `STACK.md` skeleton:

```markdown
# Stack

| Concern | Pick | rails new flag |
|---|---|---|
| App type | full-stack | — |
| Database | PostgreSQL | -d postgresql |
| Asset pipeline | Propshaft | — (default) |
| CSS | Tailwind | -c tailwind |
| JavaScript | import maps | — (default) |
| Solid (Q/C/Cable) | kept | — |
| Deploy | Kamal + Thruster | — |
| Test framework | RSpec | -T (then rails-testing) |
| Auth | Rails 8 generator + Pundit | — (then rails-auth) |
| Compliance | none | — |
```

The **`Compliance`** row is the suite's opt-in compliance switch. Its value is a
comma-separated list of tags (or `none`):

| Tag | Siblings that proactively offer a primitive when set |
|---|---|
| `none` *(default)* | — (compliance primitives stay reactive: applied only when asked) |
| `gdpr` / `ccpa` | `../rails-models/` `data-protection.md` — encryption at rest, consent, export, erasure |
| `audit-trail` | `../rails-models/` `audit-trail.md` — change logging |
| `soc2` | `../rails-models/` `audit-trail.md` — change logging (audit/access focus) |
| `hipaa` | both of the above (a trigger for both primitives everywhere) |
| `wcag` / `section508` | `../rails-hotwire/` `accessibility.md` — escalate to a per-view a11y pass + axe gate *(full-stack only)* |

Siblings read this row in their **Detect Before You Generate** step. A set tag means
"offer the primitive at the right moment"; it never auto-generates without asking.

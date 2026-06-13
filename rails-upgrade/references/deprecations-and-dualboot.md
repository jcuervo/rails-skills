# Deprecations & Dual-Booting

Two tools for a safe upgrade: **triaging deprecation warnings** (your early-warning
system for the *next* version) and **dual-booting** (running old + new Rails side by
side so a long upgrade doesn't block shipping).

## Deprecations are your roadmap to the next version

Each Rails version **deprecates** things one version before it **removes** them. So
the deprecation warnings on version N tell you exactly what will break on N+1 — fix
them *before* you bump.

```ruby
# config/environments/test.rb (or development) — make deprecations impossible to ignore:
config.active_support.deprecation = :raise          # fail the test that triggers a deprecation
# or :log to collect them first, then ratchet to :raise
config.active_support.disallowed_deprecation = :raise
config.active_support.disallowed_deprecation_warnings = ["specific message to ban first"]
```

- **Workflow:** start `:log`, collect every deprecation the suite surfaces, fix them,
  then flip to `:raise` so no new ones creep in. A clean deprecation log on version N
  means N+1 will be far smoother.
- Fix the **app's** deprecations and watch for **gem** deprecations (a dependency
  using soon-removed APIs) — those need a gem bump or replacement.

## Dual-booting (next_rails / BootBoot)

For a **large/long upgrade**, dual-boot so CI runs the suite against **both** the
current and the next Rails from the **same branch** — you fix the new version
incrementally without blocking releases on the old one.

```ruby
# Gemfile (with bootboot / next_rails):
# A second lockfile (Gemfile.next.lock) pins the NEXT Rails; DEPENDENCIES_NEXT=1 selects it.
```
```bash
gem install next_rails           # tooling around BootBoot's dual-boot
bundle install                   # current
DEPENDENCIES_NEXT=1 bundle install   # the "next" set
DEPENDENCIES_NEXT=1 bin/rails test   # run the suite on the next Rails
```

- **Pick dual-boot when** the upgrade spans weeks and the app must keep shipping —
  CI's "next" job goes from red to green as you fix things, and you flip over when it's
  fully green.
- **Critical caveat:** while dual-booting, **do NOT advance `config.load_defaults`** —
  that flips all new defaults at once and breaks the *current* boot. Keep defaults on
  the old version until you've fully cut over, then adopt them incrementally
  ([framework-defaults.md](framework-defaults.md)).
- Confirm the current `next_rails`/BootBoot setup against their READMEs — the exact
  files/env vars evolve.

## The test suite is the safety net

- Everything here assumes a **real test suite** — it's what tells you a version bump
  or a flipped default broke something. If coverage is thin, **build it first** via
  [`../rails-testing/`](../../rails-testing/SKILL.md) before upgrading. Upgrading
  without tests is the root cause of most upgrade horror stories.
- Run the suite **in `RAILS_ENV=production`** too (eager load) for the Zeitwerk class
  of failures ([zeitwerk.md](zeitwerk.md)).

## Common Pitfalls

- **Ignoring deprecation warnings** → they become hard errors on the next version with
  no warning; triage them on the current version.
- **Advancing `load_defaults` while dual-booting** → breaks the current boot; keep it
  pinned until cutover.
- **Dual-booting a tiny app** → overhead for no benefit; just go incremental
  ([upgrade-path.md](upgrade-path.md)).
- **Trusting a green dev run** → run `:raise` deprecations and prod eager-load too.
- **No suite** → you can't know what broke; build tests first.

## Verify

```bash
# Deprecations are surfaced (and ideally raised) in test:
grep -rn "active_support.deprecation" config/environments/ 2>/dev/null
bin/rails test 2>/dev/null || bundle exec rspec 2>/dev/null     # green with deprecations raised = ready for next bump
# Dual-boot (if used): the "next" set installs and the suite runs against it
DEPENDENCIES_NEXT=1 bundle install 2>/dev/null && DEPENDENCIES_NEXT=1 bin/rails test 2>/dev/null | tail -3 || echo "(not dual-booting)"
```

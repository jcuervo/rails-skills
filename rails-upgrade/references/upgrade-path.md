# Upgrade Path & the Per-Version Loop

Plan the route from the current version to the target, then run the **same loop once
per minor version**. The golden rule (from the Rails upgrade guide): **one minor
version at a time, green and deployed between each**.

## Plan the path

```bash
ruby -v; grep -E "^    rails \(" Gemfile.lock | head -1     # where you are
```

Example route to the suite target:

```
5.2 → 6.0 → 6.1 → 7.0 → 7.1 → 7.2 → 8.0 → 8.1     (Rails, one minor at a time)
Ruby: bump alongside, keeping a version Rails supports at each step
```

- **Rails and Ruby move together** but not necessarily in lockstep: at each Rails
  step, run a Ruby version that Rails supports. Often: get Rails up one minor, then
  bump Ruby to a version the new Rails prefers.
- **Zeitwerk gate:** classic autoloader is removed in Rails 7.0 — you must migrate to
  Zeitwerk ([zeitwerk.md](zeitwerk.md)) *before* crossing into 7.x.
- Check each version's section of the **Rails upgrade guide** for the specific
  breaking changes — don't rely on memory; the guide is the source of truth.

## The per-version loop (repeat for each minor)

```bash
# 1. Bump the version in the Gemfile, then update Rails + what blocks it:
#    gem "rails", "~> 7.0.0"
bundle update rails                       # add blocking gems as needed; keep the diff minimal

# 2. Reconcile config/files interactively:
bin/rails app:update                      # review EACH prompt; diff before accepting overwrites

# 3. Get the suite green BEFORE adopting new defaults:
bin/rails test    # or: bundle exec rspec   (../rails-testing/)

# 4. Adopt new framework defaults incrementally, then flip load_defaults:
#    → framework-defaults.md

# 5. Triage deprecations:
#    → deprecations-and-dualboot.md

# 6. Commit + deploy THIS version before the next bump.
git commit -am "Upgrade to Rails 7.0"     # one version per PR/commit — small, reviewable, revertable
```

- **One version per PR/commit.** A reviewable, deployable, revertable step. A
  "5.2→8.1 in one PR" is unreviewable and undebuggable.
- **`app:update` is interactive — read every prompt.** It wants to overwrite config
  files you may have customized (`boot.rb`, `application.rb`, initializers). Diff each
  change; accept the framework's, then re-apply your customizations on top.
- **Update gems deliberately.** `bundle update rails` plus only the gems that block
  it; a full `bundle update` changes too much at once. Replace abandoned gems that
  don't support the new Rails.

## Direct jump (small apps only)

For a **small app with a strong test suite**, you may jump straight to the target:
bump to the final version, `app:update`, fix everything at once. The debugging
surface is the *sum* of all intermediate breaking changes, so this only works when
the app is small and the suite catches regressions. Otherwise: incremental.

## Common Pitfalls

- **Skipping minors** → you hit several versions' breaking changes at once with no
  way to bisect which broke what.
- **Accepting `app:update` overwrites blindly** → loses your config customizations;
  diff each.
- **Flipping `load_defaults` in the same step as the version bump** → two big changes
  tangled together; get green first, *then* adopt defaults
  ([framework-defaults.md](framework-defaults.md)).
- **No deploy between steps** → a latent break compounds with the next version's
  changes; ship each step.
- **Upgrading with a weak/absent suite** → regressions ship silently; build tests
  first (`../rails-testing/`).

## Verify

```bash
grep -E "^    rails \(" Gemfile.lock | head -1     # the step landed on the intended version
bin/rails about                                    # Ruby + Rails as expected
bundle install && bin/rails runner "puts 'boots'"  # app boots on the new version
bin/rails test 2>/dev/null || bundle exec rspec 2>/dev/null   # green at this step before the next bump
```

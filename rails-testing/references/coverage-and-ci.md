# Coverage, Parallelism & Running in CI

Measuring what's tested (SimpleCov), running the suite faster (parallel), and where
it runs for real (CI). This skill makes the suite **runnable and measured**; the CI
*pipeline* that gates merges/deploys is
[`../rails-deploy/`](../../rails-deploy/SKILL.md) — don't duplicate the workflow here.

## SimpleCov (Recommended)

```ruby
# Gemfile (:test): gem "simplecov", require: false
```
```ruby
# top of test/test_helper.rb (Minitest) or spec/rails_helper.rb (RSpec) — MUST be first:
require "simplecov"
SimpleCov.start "rails" do
  enable_coverage :branch          # line + branch coverage
  add_filter "/test/"; add_filter "/spec/"
end
```

- Generates `coverage/index.html` after a run. The `"rails"` profile pre-groups
  models/controllers/etc. (Confirm the profile name and the `enable_coverage
  :branch` API against your installed SimpleCov — these have shifted across major
  versions.)
- **`require: false` + load it first** — SimpleCov must start before your app code
  loads or it under-reports.
- Coverage is a **signal, not a target.** Don't chase 100%; watch for untested
  critical paths. A test that exists only to raise the number is noise.

## Parallel tests (faster local + CI)

```ruby
# Minitest — test/test_helper.rb
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
end
```
- Rails parallelizes Minitest across processes, each with its own database
  (e.g. `app_test-0`, `-1`, …). `db:test:prepare` sets them up. (Confirm the
  `parallelize` signature and worker-DB naming for your Rails version — both are
  version-sensitive.)
- RSpec parallelizes via the `parallel_tests` gem.
- **SimpleCov + parallel** needs result merging (SimpleCov handles process merging;
  ensure a single final report). Verify the merged report isn't fragmented.

## Running in CI (this skill's part)

A Rails 8 app scaffolded without `--skip-ci` ships `.github/workflows/ci.yml` that
runs the suite. This skill's job is to ensure the suite is **green and reproducible**:

- `bin/rails db:test:prepare` (or `db:prepare`) loads the schema in CI.
- Tests must not depend on dev-only data, local time zones, or external services
  (stub them).
- System tests need a headless browser available in the CI image.

The pipeline's structure (jobs, caching, deploy gating, Brakeman/RuboCop steps) is
owned by [`../rails-deploy/`](../../rails-deploy/SKILL.md) and
[`../rails-security/`](../../rails-security/SKILL.md). Hand the green suite to them.

## Deep Dive

- **Coverage groups** (`add_group`) split the report by layer for readability.
- **Minimum coverage gate** (`SimpleCov.minimum_coverage 90`) can fail CI below a
  threshold — adopt cautiously; a hard gate encourages gaming.
- **Flaky tests** erode trust faster than missing ones — quarantine and fix, don't
  retry-until-green blindly.
- **Seed for reproducibility:** tests run in random order with a seed; reproduce a
  failure with `bin/rails test --seed N` / `rspec --seed N`.

## Common Pitfalls

- **SimpleCov started after app load** → wildly low/wrong numbers; require it first.
- **Coverage as a KPI** → tests written to touch lines, not assert behavior.
- **Parallel tests sharing a database** → random failures; Rails gives each worker
  its own DB — ensure `db:test:prepare` ran.
- **Tests depending on external services/time** → flake in CI; stub HTTP, freeze
  time.
- **Duplicating the CI workflow here** → it's owned by `../rails-deploy/`; this skill
  just guarantees a green, reproducible run.

## Verify

```bash
# A full run produces a coverage report:
bin/rails test 2>/dev/null || bundle exec rspec 2>/dev/null
ls coverage/index.html && echo "✓ coverage report generated"
# Parallel workers each got a DB (Minitest):
bin/rails test 2>&1 | grep -iE "parallel|workers" || echo "(serial run)"
# Reproduce by seed:
echo "re-run a failure deterministically with: bin/rails test --seed <N>  /  rspec --seed <N>"
```

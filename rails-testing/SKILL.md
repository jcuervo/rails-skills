---
name: rails-testing
description: Set up and write the test suite for a Rails 8.1 app — choose the framework (Minitest, the Rails default, or RSpec), the test-data strategy (fixtures or FactoryBot), the system-test driver (Selenium/Chrome by default, or a CDP driver like Cuprite), and coverage (SimpleCov) — then write model, request/controller, and system tests across the pyramid. Menu-driven with a Recommended default per menu; detects the framework already chosen at scaffold time (honors the -T flag and any installed rspec-rails) and config.api_only (API skips system/browser tests), and verifies the suite actually runs green. Apply when adding tests, setting up the test stack, writing factories/fixtures, testing an endpoint or a UI flow, or wiring coverage/CI.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[what-to-test]"
---

# rails-testing

## Purpose

Owns the **test strategy and the tests themselves** across the pyramid: model/unit,
request/integration, and system/browser. It picks the framework, the test-data
approach, the system driver, and coverage, then provides the patterns for writing
each layer. It detects what `../rails-scaffold/` already chose (Minitest vs RSpec —
the `-T` flag) and branches on `config.api_only` (API-only apps skip system/browser
tests). Other skills point here for "and a test → `../rails-testing/`."

## When to Apply

Use this skill when the task is:

- Setting up the test framework / data strategy / coverage for an app
- Writing model, request/controller, or system tests
- Creating factories (FactoryBot) or fixtures
- Testing a UI flow in a real/headless browser
- Wiring SimpleCov, parallel tests, or CI test runs

Do **not** use this skill when the task is:

- The thing being tested's implementation — models → `../rails-models/`, controllers → `../rails-controllers/`, JSON → `../rails-api/`, UI → `../rails-hotwire/`, auth → `../rails-auth/`
- Choosing the framework **flag** at `rails new` time → read `../rails-scaffold/SKILL.md` (it sets `-T`; this skill does the actual wiring)
- Running the CI pipeline / deploy gates → read `../rails-deploy/SKILL.md` (this skill makes tests runnable; that one runs them in CI)
- Security-specific scanning (Brakeman) → read `../rails-security/SKILL.md`
- Performance benchmarking → read `../rails-performance/SKILL.md`
- A retrospective audit of test coverage/quality → use the `rails-audit` skill

## Detect Before You Generate

```bash
ls spec/ test/ 2>/dev/null                                  # rspec (spec/) vs minitest (test/)
grep -E "^    (rspec-rails|factory_bot_rails|capybara|selenium-webdriver|cuprite|simplecov)" Gemfile.lock
grep -nE "api_only" config/application.rb                   # API-only → skip system/browser tests
ls test/fixtures spec/factories 2>/dev/null                 # existing data strategy
test -f STACK.md && grep -i -E "test|rspec|minitest" STACK.md   # the scaffold's recorded pick
```

- **Framework is usually already decided.** `spec/` or `rspec-rails` in the lockfile
  → RSpec; `test/` → Minitest. Use it silently — don't re-ask. The `-T` flag at
  scaffold time signals RSpec was intended.
- If `factory_bot`/fixtures already exist, match that data strategy.
- `config.api_only == true` → no system/browser tests; focus on model + request
  specs. Capybara/Selenium are Web-only.
- Honor `STACK.md`'s recorded framework.

## Menu

Four menus; the framework one is usually pre-decided by detection. Each via
`AskUserQuestion`, Rails-default marked **Recommended**; ask only what's unset.

### Framework
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Minitest** *(Recommended)* | Rails default; ships in `test/`, fast, minimal, plain Ruby assertions. | [framework-minitest-rspec.md](references/framework-minitest-rspec.md) |
| RSpec | Expressive DSL, huge ecosystem, rich matchers; the most common community choice. Needs `-T` + `rspec-rails`. | [framework-minitest-rspec.md](references/framework-minitest-rspec.md) |

### Test data
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Fixtures** *(Recommended)* | Rails default; fast (loaded once, in a transaction), simple YAML. Can get brittle/global at scale. | [test-data.md](references/test-data.md) |
| FactoryBot | Per-test object construction, readable, flexible (traits, associations). Slower if overused; the community default. | [test-data.md](references/test-data.md) |

### System-test driver *(Web only)*
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Selenium + Chrome (headless)** *(Recommended)* | Rails default driver; real browser, broad compatibility. | [system-tests.md](references/system-tests.md) |
| Cuprite (CDP) | Drives headless Chrome directly over CDP — no Selenium/driver binaries, fast. | [system-tests.md](references/system-tests.md) |
| rack_test (no JS) | In-process, fastest, but **no JavaScript** — can't test Turbo/Stimulus behavior. | [system-tests.md](references/system-tests.md) |

### Coverage
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **SimpleCov** *(Recommended)* | Line/branch coverage report; standard, near-zero setup. | [coverage-and-ci.md](references/coverage-and-ci.md) |
| None | Skip until coverage signal is wanted. | [coverage-and-ci.md](references/coverage-and-ci.md) |

## Decision Flow

- **Framework:** use what's installed. New app with no preference → **Minitest**
  (Rails default, zero deps). Team prefers the expressive DSL/ecosystem → **RSpec**.
  Don't run both.
- **Test data:** **Fixtures** for speed and simplicity (Rails default); **FactoryBot**
  when you want readable per-test construction with traits/associations. You *can*
  mix (fixtures for reference data, factories for the unit under test) but pick a
  primary.
- **Pyramid:** lots of fast **model** tests, a solid layer of **request** tests
  (the API/controller contract), and a **few** high-value **system** tests for
  critical user journeys. Don't invert it — system tests are slow and flaky-prone.
- **System driver:** **Selenium** (default, compatible) unless you want Cuprite's
  speed/no-driver-binaries. Use **rack_test** only for non-JS pages — it can't
  exercise Turbo/Stimulus (`../rails-hotwire/`).
- **API-only:** skip system tests entirely; request specs are your top layer.

## Problem → Reference

| Task | Read |
|---|---|
| Choose + set up the framework (Minitest / RSpec) | [references/framework-minitest-rspec.md](references/framework-minitest-rspec.md) |
| Test data: fixtures vs FactoryBot, traits, associations | [references/test-data.md](references/test-data.md) |
| Model/unit tests: validations, associations, scopes, methods | [references/model-tests.md](references/model-tests.md) |
| Request/controller/integration tests (API + Web), auth in tests | [references/request-tests.md](references/request-tests.md) |
| System/browser tests: Capybara, drivers, Turbo/JS, the transaction gotcha | [references/system-tests.md](references/system-tests.md) |
| Coverage (SimpleCov), parallel tests, running in CI | [references/coverage-and-ci.md](references/coverage-and-ci.md) |

## Verify

The suite is only "set up" once it actually runs green:

```bash
# Minitest:
bin/rails test && bin/rails test:system 2>/dev/null        # unit/integration, then system (Web)
# RSpec:
bundle exec rspec                                          # whole suite
# Coverage report generated (if SimpleCov wired):
ls coverage/index.html 2>/dev/null && echo "✓ coverage report"
```

A green run that exercises model + request (+ a system test for Web) is the gate.
Then route to `../rails-deploy/` to run the suite in CI and record the test stack in
`STACK.md`.

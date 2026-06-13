# System / Browser Tests (Web only)

The top of the pyramid for **Web** apps: drive a real (or headless) browser through
critical user journeys with Capybara. **API-only apps skip this entirely** — request
specs are their top layer. Keep system tests **few and high-value** — they're slow
and the most flake-prone.

## Quick Pattern

```ruby
# Minitest — test/system/posts_test.rb
require "application_system_test_case"
class PostsTest < ApplicationSystemTestCase
  test "creating a post" do
    sign_in_as users(:alice)          # ../rails-auth/
    visit new_post_path
    fill_in "Title", with: "Hello"
    click_on "Create Post"
    assert_text "Post created"        # waits for the DOM (Capybara auto-waits)
  end
end
```
```ruby
# RSpec — spec/system/posts_spec.rb
require "rails_helper"
RSpec.describe "Posts", type: :system do
  it "creates a post" do
    driven_by(:selenium, using: :headless_chrome)
    # ... visit / fill_in / click_on / expect(page).to have_text(...)
  end
end
```

Rails generates `test/system` (Minitest) with `ApplicationSystemTestCase` already
configured. RSpec system specs wrap the same Rails system-test machinery.

## Driver menu → `driven_by`

| Driver | Trade-off |
|---|---|
| **Selenium + headless Chrome** *(Recommended)* | Rails default; real browser, 1400×1400, broadly compatible. Needs Chrome + driver. |
| Cuprite (CDP) | Drives headless Chrome over the DevTools Protocol — no Selenium/driver binaries, faster. Add `gem "cuprite"`. |
| rack_test | In-process, fastest, **no JavaScript** — fine for plain server-rendered pages, useless for Turbo/Stimulus. |

```ruby
# Minitest default (test/application_system_test_case.rb):
driven_by :selenium, using: :headless_chrome, screen_size: [1400, 1400]
```

> The default driver/flags/screen size are whatever `rails new` wrote into
> `test/application_system_test_case.rb` — **read that file to confirm the current
> default for your Rails version** (driver name, `:headless_chrome` spelling, and
> screen size can shift between releases) rather than assuming the values above.

## The transaction gotcha (read this before debugging flakes)

System tests run a real HTTP server in a **separate thread** from the test. If your
data is created in a transaction that hasn't committed, the browser's thread can't
see it. Rails system tests — and recent `rspec-rails` versions — share a single DB
connection across threads to handle this automatically, so it often "just works."
**Symptom if it doesn't:** records exist in the test but the page renders empty. If
you hit that, check your data-cleaning strategy (the shared-connection setting, or
switch that spec to a truncation strategy) — verify what your `rspec-rails`/
`database_cleaner` version does before changing it rather than reflexively disabling
transactional fixtures.

## Deep Dive

- **Test journeys, not pages:** sign in → create → see it listed → edit. A handful
  of these covers the app's spine; don't system-test every form (request tests do
  that cheaper).
- **Capybara auto-waits** for elements (`has_text?`, `has_selector?`) — never add
  `sleep`; use the matchers, which retry until the timeout.
- **Turbo/Stimulus** behavior ([`../rails-hotwire/`](../../rails-hotwire/SKILL.md))
  needs a **JS-capable** driver (Selenium/Cuprite), not rack_test.
- **Screenshots on failure** are captured automatically (`tmp/screenshots`) — use
  them to debug.
- **Run headless in CI**; headful locally when you need to watch.

## Common Pitfalls

- **System-testing everything** → slow, flaky suite; keep them few and critical.
- **rack_test for a Turbo/JS flow** → no JS runs; the test passes/fails for the wrong
  reason. Use Selenium/Cuprite.
- **`sleep` to "fix" timing** → flaky and slow; rely on Capybara's auto-wait matchers.
- **Data invisible to the browser** → the transaction gotcha above; adjust the
  cleaning strategy.
- **Asserting on `current_path` before the action completes** → use `assert_text`/
  `have_current_path` which wait.

## Verify

```bash
# Minitest system tests (boots a browser):
bin/rails test:system 2>/dev/null && echo "✓ system tests green"
# RSpec:
bundle exec rspec spec/system 2>/dev/null && echo "✓ system specs green"
# Confirm a JS-capable driver is available for Turbo flows:
bundle list | grep -E "selenium-webdriver|cuprite" || echo "no JS driver — Turbo flows can't be system-tested"
```

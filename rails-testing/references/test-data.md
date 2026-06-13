# Test Data — Fixtures vs FactoryBot

How tests get their records. The menu pick is **fixtures** (Rails default, fast);
**FactoryBot** is the community favorite for readable per-test construction. You can
mix, but pick a primary.

## Fixtures (Recommended — Rails default)

```yaml
# test/fixtures/posts.yml
welcome:
  title: "Welcome"
  body: "First post"
  author: alice          # references authors(:alice) by name
```
```ruby
post = posts(:welcome)   # available in any test; loaded once per suite, wrapped in a transaction
```

- **Pick when** you want speed and simplicity: fixtures load **once** into the test
  DB and each test runs in a transaction that rolls back — very fast.
- **Don't pick when** the data graph gets complex and a global YAML set becomes
  brittle (every test depends on shared rows; one edit ripples).
- ERB works in fixtures (`<%= %>`) for dynamic values. References are by fixture
  name. Keep the set small and meaningful.

## FactoryBot (per-test construction)

```ruby
# Gemfile (:development, :test): gem "factory_bot_rails"
# test/factories/posts.rb  (or spec/factories/)
FactoryBot.define do
  factory :post do
    title { "A title" }
    body  { "Body" }
    association :author          # builds/links an author factory
    trait(:published) { published_at { Time.current } }
  end
end
```
```ruby
post   = create(:post, :published)     # persisted
draft  = build(:post)                  # in-memory, not saved
attrs  = attributes_for(:post)         # hash of attributes
```

- **Pick when** you want each test to declare exactly the object it needs — readable,
  flexible, with traits and associations. The community default.
- **Don't pick when** overuse makes suites slow: `create(:x)` hits the DB; prefer
  `build`/`build_stubbed` when you don't need persistence.
- Add `include FactoryBot::Syntax::Methods` (RSpec config or test helper) so you can
  call `create`/`build` without the `FactoryBot.` prefix.

## Mixing strategies

A common pragmatic split: **fixtures for stable reference data** (countries, plans)
and **factories for the records under test**. Don't build the same concept both
ways — pick one source per concept.

## Faker for values

`faker` (dev/test) generates realistic attribute values inside factories. Keep
values deterministic where a test asserts on them (don't `Faker` a field you then
assert equals a specific string).

## Common Pitfalls

- **`create` everywhere** → slow suite; use `build`/`build_stubbed` unless the test
  needs a persisted row or a query.
- **Giant interdependent fixture set** → brittle; one change breaks many tests.
- **Factories with required associations always `create`d** → an unwanted object
  graph per test; use `build_stubbed` or override associations.
- **Random values where you assert exact equality** → flaky tests; fix the value.
- **Fixtures + factories building the same record** → duplicate/ambiguous data.

## Verify

```bash
# Fixtures load and are addressable:
bin/rails test test/models 2>/dev/null && echo "✓ fixtures load" \
  || bundle exec rspec spec/models 2>/dev/null
# FactoryBot factories are all valid (great smoke test):
bin/rails runner "require 'factory_bot'; FactoryBot.factories.each { |f| FactoryBot.build(f.name) }; puts '✓ factories build'" 2>/dev/null \
  || echo "(FactoryBot not installed / using fixtures)"
```

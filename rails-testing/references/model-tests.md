# Model / Unit Tests

The base of the pyramid: fast tests for validations, associations, scopes, enums,
and methods on your models ([`../rails-models/`](../../rails-models/SKILL.md) owns
the implementation). These should be the **most numerous** tests — they boot fast
and pin domain behavior.

## Quick Pattern

```ruby
# Minitest — test/models/post_test.rb
require "test_helper"
class PostTest < ActiveSupport::TestCase
  test "is invalid without a title" do
    assert_not Post.new(body: "x").valid?
  end
  test "published scope returns only published" do
    Post.create!(title: "a", status: :published)
    Post.create!(title: "b", status: :draft)
    assert_equal 1, Post.published.count
  end
end
```
```ruby
# RSpec — spec/models/post_spec.rb
require "rails_helper"
RSpec.describe Post, type: :model do
  it { is_expected.to validate_presence_of(:title) }   # shoulda-matchers
  it "scopes to published" do
    create(:post, status: :published); create(:post, status: :draft)
    expect(Post.published.count).to eq(1)
  end
end
```

## What to test (and what not to)

- **Test:** custom validations, scopes, enums, business methods, `normalizes`,
  callbacks' observable effects, association `dependent:` behavior, value objects.
- **Don't test:** Rails itself (that `validates :title, presence: true` works) —
  unless you're asserting *your* rule fires. Test behavior you wrote, not the
  framework.
- **DB constraints** (unique index, FK) deserve a test that the *model* surfaces the
  error, plus trust the migration for the DB layer ([`../rails-models/`](../../rails-models/references/migrations.md)).

## `shoulda-matchers` (optional, both frameworks)

```ruby
# Gemfile (:test): gem "shoulda-matchers"
should validate_presence_of(:title)             # Minitest
it { is_expected.to belong_to(:author) }        # RSpec
```
Concise one-liners for common validations/associations. Optional — plain assertions
work fine.

## Deep Dive

- **Keep them fast:** prefer `build`/`build_stubbed` over `create` when you don't
  need persistence ([test-data.md](test-data.md)); model tests shouldn't need a
  browser or HTTP.
- **One behavior per test** → clear failures. Assert the specific outcome, not a
  broad "valid?".
- **Time-dependent logic** → freeze time (`travel_to` / `freeze_time` in Rails) so
  tests are deterministic.
- **Edge cases over happy path** → nils, boundaries, uniqueness races (the race
  itself is a DB-constraint concern, but test the model error).

## Common Pitfalls

- **Testing framework behavior** → low-value, brittle; test your rules.
- **`create` when `build` suffices** → slow suite.
- **Asserting only `valid?`/`invalid?`** without checking *which* error → a test
  that passes for the wrong reason; assert on `errors[:attr]`.
- **Non-deterministic time/random** → flakiness; freeze time, fix seeds/values.

## Verify

```bash
# Minitest:
bin/rails test test/models                              # all model tests green
bin/rails test test/models/post_test.rb:5               # a single test by line
# RSpec:
bundle exec rspec spec/models                           # all model specs green
```

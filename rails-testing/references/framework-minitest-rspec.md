# Framework — Minitest vs RSpec

The menu deep dive for the test framework. **Use whatever is already installed**
(detect first); for a greenfield choice, Minitest is the Rails default and RSpec is
the most common community pick. Never run both.

## Minitest (Recommended — Rails default)

Ships with Rails in `test/`. No extra gems; tests are plain Ruby with assertions.

```ruby
# test/models/post_test.rb
require "test_helper"
class PostTest < ActiveSupport::TestCase
  test "requires a title" do
    post = Post.new
    assert_not post.valid?
    assert_includes post.errors[:title], "can't be blank"
  end
end
```

- Run: `bin/rails test`, one file `bin/rails test test/models/post_test.rb`, one line
  `bin/rails test test/models/post_test.rb:7`.
- Test types map to dirs: `test/models`, `test/controllers`, `test/integration`,
  `test/system`, `test/mailers`, `test/jobs`.
- **Pick when** you want the default, fastest-to-boot, zero-dependency option that
  every Rails dev can read.

## RSpec (expressive DSL, big ecosystem)

```ruby
# Gemfile (:development, :test): gem "rspec-rails"
bin/rails generate rspec:install      # spec/spec_helper.rb, spec/rails_helper.rb, .rspec
```
```ruby
# spec/models/post_spec.rb
require "rails_helper"
RSpec.describe Post, type: :model do
  it "requires a title" do
    expect(Post.new).not_to be_valid
  end
end
```

- Run: `bundle exec rspec`, one file `bundle exec rspec spec/models/post_spec.rb`,
  one line `bundle exec rspec spec/models/post_spec.rb:5`.
- Spec types: `spec/models`, `spec/requests`, `spec/system`, `spec/mailers`,
  `spec/jobs` (+ `type:` metadata).
- **Pick when** you want the expressive `describe/it/expect` DSL, rich matchers,
  shared examples, and the large gem ecosystem (`shoulda-matchers`, `rspec-rails`
  helpers). Most community guides assume it.
- New apps adopting RSpec are scaffolded with `-T` (`../rails-scaffold/`) so Rails
  doesn't generate a `test/` dir to orphan.

## Choosing

| You want | Pick |
|---|---|
| Rails default, minimal, fastest boot | Minitest |
| Expressive DSL, matchers, big ecosystem | RSpec |
| To match an existing suite | whatever's installed (detect) |

Generators follow the framework: with `rspec-rails`, `bin/rails g model` creates
specs, not `test/` files. With `factory_bot_rails`, they create factories too
([test-data.md](test-data.md)).

## Common Pitfalls

- **Both frameworks installed** → duplicate/confusing suites; commit to one.
- **RSpec without `-T` on a new app** → an orphaned `test/` dir alongside `spec/`;
  remove it or scaffold with `-T`.
- **Forgetting `rails_helper` vs `spec_helper`** → Rails isn't loaded; model/request
  specs need `require "rails_helper"`.
- **Asserting on framework internals** instead of behavior → brittle tests; assert
  outcomes.

## Verify

```bash
ls spec/ test/ 2>/dev/null                       # exactly one of these is your suite
# Minitest:
bin/rails test 2>/dev/null && echo "✓ minitest runs"
# RSpec:
bundle exec rspec --dry-run 2>/dev/null && echo "✓ rspec loads specs"
```

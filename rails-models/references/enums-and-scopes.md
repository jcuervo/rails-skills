# Enums, Scopes, Value Objects & Tokens

The model's query- and domain-API surface: `enum` for fixed states, `scope` for
reusable queries, value objects for cohesive multi-column concepts, and
`generates_token_for` for signed, expiring tokens.

## Enums (Rails 7+ syntax)

```ruby
class Post < ApplicationRecord
  enum :status, { draft: 0, published: 1, archived: 2 }, default: :draft
  # or positional: enum :status, %i[draft published archived]
end
```

- Modern syntax is `enum :name, ...` (name as the first argument). Options
  (`default`, `prefix`, `suffix`, `scopes`) are written **without** leading
  underscores on current Rails (`prefix: true`, not `_prefix: true`); the
  underscored form was the older spelling. If you're unsure on the target app's
  version, check `ActiveRecord::Enum` docs for that release — the detect step
  above already establishes the version.
- Gives you `post.published?`, `post.published!`, `Post.published`, and
  `Post.statuses`.
- **Prefer an integer-backed hash** (`{ draft: 0, ... }`) over a bare array: the
  mapping is explicit, so reordering or removing a value doesn't silently
  re-map existing rows. Use `prefix:`/`suffix:` when two enums share value names.
- Add a DB default and (optionally) a `CHECK`/index — the enum lives in Ruby; the
  column is a plain integer.

## Scopes

```ruby
class Post < ApplicationRecord
  scope :recent,    -> { order(created_at: :desc) }
  scope :authored_by, ->(user) { where(author: user) }
  scope :published_since, ->(date) { published.where("published_at >= ?", date) }
end
```

- Scopes are chainable and return relations, so they compose:
  `Post.published.recent.authored_by(current_user)`.
- A scope **must** return a relation; a scope that can return `nil` breaks
  chaining. For "find one or nil," use a class method, not a scope.
- Keep scopes thin and query-only. Multi-step business logic belongs in a class
  method or a query object, not a giant scope.

## Value objects (cohesive multi-column concepts)

```ruby
# composed_of: wrap (amount_cents, currency) as a Money value object
class Product < ApplicationRecord
  composed_of :price, class_name: "Money",
              mapping: { price_cents: :cents, price_currency: :currency }
end
```

Use a value object (`composed_of`, or a plain PORO + `serialize`/custom
`ActiveRecord::Type`) when several columns only make sense together (money,
coordinates, a date range). It keeps invariants in one place instead of scattering
them across the model.

## Signed, expiring tokens → `generates_token_for` (Rails 7.1+)

```ruby
class User < ApplicationRecord
  generates_token_for :password_reset, expires_in: 15.minutes do
    password_salt&.last(10)   # invalidates the token once the password changes
  end
end
token = user.generate_token_for(:password_reset)
User.find_by_token_for(:password_reset, token)   # nil if expired/tampered/invalidated
```

Prefer this over hand-rolled token columns for password resets, email
confirmation, and magic links — it's signed, self-expiring, and needs no extra
column. Auth flows that use it are owned by `../rails-auth/SKILL.md`.

## Common Pitfalls

- **Array-backed enum reordered** → integer values shift and existing rows
  silently change meaning. Use an explicit hash.
- **Scope that returns `nil`** (e.g. a bare `find_by` inside) → breaks `.merge`/
  chaining. Return a relation or make it a class method.
- **Enum value colliding with a model method** (`enum :status, %i[valid ...]`
  shadows `valid?`) → name states to avoid reserved methods; use `prefix:`.
- **Business logic creeping into scopes** → untestable SQL soup; extract a query
  object.

## Verify

```bash
bin/rails runner "p Post.statuses; p Post.new.draft?"                 # enum mapping + predicate
bin/rails runner "Post.create!(title:'t', status: :published); p Post.published.count"  # enum scope
bin/rails runner "p Post.recent.to_sql"                                # scope composes into SQL
bin/rails runner "t=User.first&.generate_token_for(:password_reset); p User.find_by_token_for(:password_reset, t)&.id"  # token round-trips
```

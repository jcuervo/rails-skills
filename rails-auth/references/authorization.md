# Authorization — Pundit, Action Policy & CanCanCan

Authorization answers "**may this user do this?**" — separate from authentication
("who are you?"). The menu pick is **Pundit** for explicit, testable policy objects.
A denied action is **403 Forbidden** (vs 401 for unauthenticated).

## Pundit (Recommended)

```ruby
# Gemfile: gem "pundit"
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  def update? = user.admin? || record.author_id == user.id
  def destroy? = user.admin?

  class Scope < ApplicationPolicy::Scope
    def resolve = user.admin? ? scope.all : scope.where(author: user)
  end
end
```
```ruby
class PostsController < ApplicationController
  include Pundit::Authorization
  def update
    @post = Post.find(params[:id])
    authorize @post                      # raises Pundit::NotAuthorizedError → map to 403
    @post.update!(post_params)
  end
  def index = @posts = policy_scope(Post)   # scope to what the user may see
end
```
```ruby
# Map the denial to 403 once (../rails-controllers/ concerns-and-filters.md):
rescue_from Pundit::NotAuthorizedError do
  render_error(:forbidden, "Not allowed")    # envelope: ../rails-api/ error-envelopes.md (API) / a 403 page (Web)
end
```

- **Pick when** you want one explicit policy object per model, plain-Ruby methods,
  easy unit testing, minimal magic. The common default.
- **Don't pick when** rules are highly cross-cutting and you want built-in caching/
  scoping/test helpers (Action Policy) or prefer one central rules file (CanCanCan).
- `verify_authorized`/`verify_policy_scoped` after-actions catch controllers you
  forgot to authorize — turn them on.

## Action Policy (more built-in)

```ruby
# Gemfile: gem "action_policy"
class PostPolicy < ApplicationPolicy
  def update? = user.admin? || record.author_id == user.id
end
```
```ruby
class PostsController < ApplicationController
  def update
    @post = Post.find(params[:id])
    authorize! @post              # Action Policy
  end
end
```

- **Pick when** authorization gets complex: built-in policy caching, explicit
  scoping, first-class test matchers, namespaced policies, GraphQL/Action Cable
  integration. Scales further than Pundit with less boilerplate.
- **Don't pick when** Pundit's simplicity already covers you — Action Policy's extra
  concepts aren't worth it for basic rules.

## CanCanCan (central Ability)

```ruby
# Gemfile: gem "cancancan"
class Ability
  include CanCan::Ability
  def initialize(user)
    user ||= User.new
    can :manage, :all if user.admin?
    can :update, Post, author_id: user.id
  end
end
```
```ruby
class PostsController < ApplicationController
  load_and_authorize_resource     # auto-loads @post + authorizes from Ability
end
```

- **Pick when** the app is small/medium and you like **all rules in one place**
  (`Ability`), with `can?`/`cannot?` helpers in views.
- **Don't pick when** the `Ability` class grows into an unreadable mega-file —
  per-model policies (Pundit/Action Policy) scale better as rules multiply.

## Roles & the user model

Authorization reads roles/attributes off the `User` (e.g. `user.admin?`,
`user.role`, a `roles` join). Modeling roles (enum, join table, `rolify`) is a
`../rails-models/` concern; the policy *consumes* them. Keep "what roles exist" in
the model and "what each role may do" in the policy.

## Common Pitfalls

- **Confusing 401 and 403** → unauthenticated = 401 (send to login); authenticated
  but forbidden = 403. Map the policy denial to **403**.
- **Forgetting to authorize an action** → silent access; enable Pundit's
  `verify_authorized` / Action Policy checks.
- **Unscoped index** → users see others' records; use `policy_scope`/scopes, not just
  per-record checks.
- **Putting role definitions in the policy** → mixing "who is admin" with "what admin
  may do"; keep roles on the model.
- **Two authorization libraries** → pick one; mixed idioms confuse the team.

## Verify

```bash
bundle list | grep -E "pundit|action_policy|cancancan"      # one authz lib present
# Policy denies/permits correctly (Pundit example):
bin/rails runner "admin=User.new(admin:true); other=User.new; p PostPolicy.new(other, Post.new(author_id: 999)).update?; p PostPolicy.new(admin, Post.new).update?"   # => false, true
bin/rails server -d && sleep 3
curl -s -o /dev/null -w "forbidden=%{http_code}\n" -X PATCH localhost:3000/posts/1   # 403 when not permitted (or 401/302 if unauthenticated)
kill "$(cat tmp/pids/server.pid)"
```

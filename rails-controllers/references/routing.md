# Routing

`config/routes.rb` maps URLs to controller actions. The menu pick is
**resourceful first** — model endpoints as REST resources and add the rare
non-CRUD route deliberately.

## Quick Pattern

```ruby
Rails.application.routes.draw do
  resources :posts do
    resources :comments, shallow: true     # nested, but shallow to keep URLs flat
    member     do post :publish end        # /posts/:id/publish  (do…end is the safe idiom)
    collection do get  :drafts  end        # /posts/drafts
  end

  namespace :admin { resources :users }    # /admin/users, Admin::UsersController
  resource :session, only: %i[new create destroy]   # singular: no :id

  root "posts#index"
  get "up" => "rails/health#show"          # health check (Rails default)
end
```

## When to pick which

- **`resources`** — the default for any CRUD noun. Generates the 7 actions +
  named helpers (`posts_path`, `edit_post_path`). Pare with `only:`/`except:`.
- **`resource`** (singular) — a thing with no collection/`:id` (the current user's
  `session`, `profile`).
- **Explicit `get/post … => "ctrl#action"`** — genuinely non-resourceful endpoints
  (webhooks, OAuth callbacks, health). Use sparingly *alongside* resources, not as
  a replacement for them.

## Deep Dive

- **Nesting:** nest at most one level; deep nesting makes ugly URLs and brittle
  helpers. Use `shallow: true` so index/new/create stay nested but
  show/edit/update/destroy go flat (`/comments/:id`).
- **`member` vs `collection`:** `member` adds `:id` (acts on one record);
  `collection` doesn't (acts on the set).
- **Namespaces vs scopes:** `namespace :admin` changes URL *and* module *and*
  helper prefix. `scope module: :admin` changes only the controller module;
  `scope path: "admin"` only the URL. Pick the one that matches what you actually
  want to change.
- **Constraints & formats:** `constraints(subdomain: "api")` or
  `constraints(->(req){ ... })` to gate routes; `defaults: { format: :json }` to
  fix a format. For **API versioning** routes (`/v1`, header negotiation), the
  strategy is owned by `../rails-api/SKILL.md` — route here, decide there.
- **Direct/route helpers:** prefer named-helper paths (`post_path(post)`) over
  hardcoded strings in code and tests.

## Common Pitfalls

- **Deeply nested resources** → `/posts/1/comments/2/likes/3`; use shallow nesting.
- **Catch-all `match "*path"`** placed above specific routes → shadows them; order
  matters, put wildcards last.
- **Custom routes where a resource fits** → loses helpers/conventions and confuses
  tooling. Model it as a resource with a `member` action instead.
- **Forgetting `only:`** on `resources` → seven routes when you need two; leaves
  unhandled actions returning confusing errors.
- **Namespace vs scope mix-up** → controller module/helper prefixes don't match the
  URL; pick the right tool above.

## Verify

```bash
bin/rails routes -g posts                              # the expected verbs/paths/helpers exist
bin/rails routes -c PostsController                    # all routes for one controller
bin/rails runner "p Rails.application.routes.recognize_path('/posts/1/publish', method: :post)"  # => {controller:'posts', action:'publish', id:'1'}
bin/rails runner "include Rails.application.routes.url_helpers; p post_path(1)"   # helper resolves
```

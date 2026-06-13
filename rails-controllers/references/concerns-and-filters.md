# Filters, Controller Concerns & Error Handling

Sharing cross-cutting controller behavior: `before_action`/`after_action`/
`around_action` filters, controller concerns for cohesive shared clusters, and
`rescue_from` to map exceptions to HTTP responses in one place.

## Quick Pattern

```ruby
class ApplicationController < ActionController::Base   # ::API for api_only apps
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActiveRecord::RecordInvalid,  with: :unprocessable

  private

  def not_found      = respond_error(:not_found, "Not found")
  def unprocessable(e) = respond_error(:unprocessable_entity, e.record.errors)
end

class PostsController < ApplicationController
  before_action :set_post, only: %i[show update destroy]
  private
  def set_post = @post = Post.find(params[:id])   # raises → handled by rescue_from
end
```

## Filters — `before_action` and friends

- **`before_action`** for setup (`set_post`) and guards (auth checks, which belong
  to `../rails-auth/`). Scope with `only:`/`except:`. Halt the request by rendering
  or redirecting inside the filter — that stops the action from running.
- **`around_action`** to wrap an action (timing, locale, `with_lock`).
- **`after_action`** rarely — most "after" work is a job (`../rails-jobs/`).
- **`skip_before_action`** in a subclass to opt a controller out of an inherited
  filter (e.g. skip auth on a public action) — name the filter explicitly.

## Controller concerns (cohesive shared clusters)

```ruby
# app/controllers/concerns/paginatable.rb
module Paginatable
  extend ActiveSupport::Concern
  included { before_action :set_page }
  private
  def set_page = @page = params.fetch(:page, 1).to_i
end
class PostsController < ApplicationController; include Paginatable; end
```

- **Pick a concern** when several controllers share a *cohesive* behavior
  (pagination setup, locale switching, a common guard).
- **Don't** extract a concern used by one controller — that's just indirection;
  keep it a private method.
- A concern that's really domain logic belongs in a model/service, not a
  controller concern.

## `rescue_from` — map exceptions once

Define exception→status mapping in the base controller so every action benefits.
The **status mapping** lives here; the **response body** (JSON envelope vs error
page) is `../rails-api/` / `../rails-hotwire/` — call into them from the handler.

```ruby
rescue_from Pundit::NotAuthorizedError, with: :forbidden   # authz: ../rails-auth/
```

- Order matters: more specific exception classes before their ancestors.
- Don't `rescue_from StandardError` broadly — it swallows bugs; let the framework
  return 500 and surface them in error tracking (`../rails-deploy/`).

## Common Pitfalls

- **Auth logic re-implemented per controller** → put the guard filter in the base
  controller / a concern; owned by `../rails-auth/`.
- **`before_action` that renders but doesn't stop** → it does halt the chain;
  forgetting that and also rendering in the action → `DoubleRenderError`.
- **Over-broad `rescue_from StandardError`** → hides real errors; rescue specific
  classes.
- **Concern as a junk drawer** → unrelated methods bundled together; keep concerns
  cohesive and single-purpose.
- **`skip_before_action` without naming the filter** → skips the wrong thing or
  errors; always pass the filter name.

## Verify

```bash
bin/rails server -d && sleep 3
curl -s -o /dev/null -w "notfound=%{http_code}\n" localhost:3000/posts/999999   # rescue_from => 404
kill "$(cat tmp/pids/server.pid)"
# Filter actually runs / halts:
bin/rails runner "p PostsController._process_action_callbacks.map(&:filter).grep(Symbol)"   # lists before_actions
```

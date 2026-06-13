# Request / Controller / Integration Tests

The middle of the pyramid — the **request layer contract**: routes resolve, actions
respond with the right status/body, auth gates work. For an API this is your **top**
layer (no system tests). Implementation is
[`../rails-controllers/`](../../rails-controllers/SKILL.md) and
[`../rails-api/`](../../rails-api/SKILL.md); this file tests it.

## Quick Pattern

```ruby
# Minitest — test/integration/posts_test.rb
require "test_helper"
class PostsTest < ActionDispatch::IntegrationTest
  test "creates a post" do
    assert_difference "Post.count", 1 do
      post posts_url, params: { post: { title: "t", body: "b" } }
    end
    assert_response :redirect          # Web; API would be :created
  end
  test "rejects invalid" do
    post posts_url, params: { post: { title: "" } }
    assert_response :unprocessable_entity      # the 422 contract
  end
end
```
```ruby
# RSpec — spec/requests/posts_spec.rb
require "rails_helper"
RSpec.describe "Posts", type: :request do
  it "returns JSON" do
    get "/v1/posts", headers: { "Accept" => "application/json" }
    expect(response).to have_http_status(:ok)
    expect(response.content_type).to match(%r{application/json})
  end
end
```

Prefer **request** specs (full Rack stack: routing → middleware → controller →
response) over old controller-unit specs — they test the real contract.

## Testing the status contract

The high-value assertions mirror what the request layer promises:

- **201/`:created`** on create (API), **3xx redirect** on Web create (PRG).
- **422/`:unprocessable_entity`** on validation failure (the Turbo + API contract —
  [`../rails-hotwire/`](../../rails-hotwire/references/forms-and-responses.md),
  [`../rails-api/`](../../rails-api/references/error-envelopes.md)).
- **404** for missing, **401** unauthenticated, **403** forbidden.
- For APIs, assert the **JSON shape** and the **error envelope**, not just the status.

## Authentication in tests

Sign in before hitting protected endpoints — the mechanism is
[`../rails-auth/`](../../rails-auth/SKILL.md):

```ruby
# Helper that establishes a session/token, then:
post session_url, params: { email_address: "a@b.com", password: "secret123" }   # Web login
# API: set the header
get "/v1/posts", headers: { "Authorization" => "Bearer #{token}" }
```
Extract a `sign_in_as(user)` test helper so every protected-endpoint test isn't
repeating the login dance.

## Deep Dive

- **Integration (Minitest) / request (RSpec)** drive real HTTP via `get/post/...`
  with `params:`/`headers:`. Assert `response.status`, `response.parsed_body`,
  redirects (`assert_redirected_to`), and side effects (`assert_difference`).
- **`assert_difference`/`expect { }.to change`** assert DB effects of an action.
- **Format/headers matter:** send `Accept`/`Content-Type` to exercise JSON vs HTML
  vs Turbo Stream branches ([`../rails-controllers/`](../../rails-controllers/references/responses.md)).
- **Don't reach into controller internals** (assigns) — test the response the client
  actually gets.

## Common Pitfalls

- **Controller-unit tests over request tests** → test less of the real stack; prefer
  request/integration specs.
- **Not asserting the 422 path** → the most common request-layer bug ships untested.
- **Repeating login in every test** → extract `sign_in_as`.
- **Asserting `assigns(:post)`** (RSpec removed it by default) → assert on the
  response/JSON instead.
- **Missing `Accept` header** → you test the HTML branch when you meant JSON (or vice
  versa).

## Verify

```bash
# Minitest:
bin/rails test test/integration test/controllers
# RSpec:
bundle exec rspec spec/requests
# Smoke a real endpoint matches the test's expectation:
bin/rails server -d && sleep 3
curl -s -o /dev/null -w "%{http_code}\n" -X POST localhost:3000/posts -d "post[title]="   # invalid → 422 (Turbo/API re-render contract); success would be 3xx (Web) / 201 (API)
kill "$(cat tmp/pids/server.pid)"
```

# Error Envelopes

One consistent JSON shape for every error, emitted from a single place. The
**status-code mapping** (which exception → which HTTP status) is owned by
`../rails-controllers/` (concerns-and-filters.md, `rescue_from`); this reference
owns the **JSON body** that goes with that status.

## Quick Pattern — one envelope, one rescue layer

```ruby
class Api::BaseController < ActionController::API
  rescue_from ActiveRecord::RecordNotFound do |e|
    render_error(:not_found, "#{e.model} not found")
  end
  rescue_from ActiveRecord::RecordInvalid do |e|
    render_error(:unprocessable_entity, "Validation failed", details: e.record.errors)
  end
  rescue_from ActionController::ParameterMissing do |e|
    render_error(:bad_request, e.message)
  end

  private

  def render_error(status, message, details: nil)
    body = { error: { status: Rack::Utils::SYMBOL_TO_STATUS_CODE[status], message: message } }
    body[:error][:details] = details if details
    render json: body, status: status
  end
end
```

Every controller inherits from `Api::BaseController`, so all errors share the shape:

```json
{ "error": { "status": 422, "message": "Validation failed",
             "details": { "title": ["can't be blank"] } } }
```

## Design the envelope once

Decide and document these, then never deviate:

- **Top-level key** — `error` (single) vs `errors` (array). Pick one. JSON:API uses
  an `errors` array of objects; if you chose jsonapi-serializer
  ([serialization.md](serialization.md)), match its error format.
- **Fields** — `status`, `message`, optional machine-readable `code`, optional
  `details` (per-field validation messages).
- **Validation errors** — surface `record.errors` so clients can map them to form
  fields. Keep the structure identical across endpoints.

## Deep Dive

- **Status mapping lives in the controller layer.** This file assumes the
  `rescue_from` handlers exist (`../rails-controllers/`); it standardizes their
  *output*. Don't duplicate the status table here.
- **Don't leak internals.** In production, a 500's body is a generic message — the
  stack trace goes to error tracking (`../rails-deploy/`), not the client. Rails'
  `config.consider_all_requests_local = false` in production already hides
  traces; your generic 500 handler should render the envelope, not the exception.
- **Auth errors** (401/403) use the same envelope; the *mechanism* that raises them
  is `../rails-auth/` — see [token-auth-surface.md](token-auth-surface.md) for the
  401 body specifically.
- **Validation vs business errors.** A failed `save` → 422 with field details; a
  domain rule violation → a custom exception class mapped to 422/409 with a `code`.

## Common Pitfalls

- **Per-action `rescue`** → divergent error shapes; centralize in the base
  controller.
- **Inconsistent key** (`error` here, `errors` there, bare string elsewhere) →
  clients can't write one parser. Pick one shape.
- **Leaking exception messages/stack traces** in production responses → information
  disclosure (`../rails-security/`). Generic message + logged detail.
- **HTML error pages from a JSON API** → an unhandled error falls through to Rails'
  HTML 500 page; ensure `ActionController::API` + your `rescue_from` cover the
  cases, and set `Accept`/format handling so errors render JSON.

## Verify

```bash
bin/rails server -d && sleep 3
BASE=localhost:3000
curl -s -H "Accept: application/json" $BASE/v1/posts/999999 -w "\n%{http_code}\n" | tee /tmp/e.json   # 404 + envelope
grep -q '"error"' /tmp/e.json && echo "✓ consistent error key"
curl -s -H "Accept: application/json" -X POST $BASE/v1/posts -d '{"post":{"title":""}}' -H 'Content-Type: application/json' | grep -q '"details"' && echo "✓ validation details present"
kill "$(cat tmp/pids/server.pid)"
```

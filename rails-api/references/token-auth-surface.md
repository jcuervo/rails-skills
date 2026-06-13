# Token-Auth Surface (API side)

The **API-facing surface** of authentication: how a client presents a token, what
an unauthenticated response looks like, and how the endpoint signals auth failure.
The **authentication mechanism itself** — issuing tokens, verifying passwords,
sessions, JWT vs opaque tokens, OAuth — is owned by `../rails-auth/SKILL.md`. This
reference is the contract between that mechanism and HTTP; it does **not**
re-document how to build auth.

## The contract (what API clients see)

```
Request:   Authorization: Bearer <token>
200/2xx →  authenticated, normal response
401     →  missing/invalid/expired token   (WWW-Authenticate: Bearer)
403     →  authenticated but not permitted  (authorization, not authentication)
```

## Quick Pattern — reading the header, returning 401

```ruby
class Api::BaseController < ActionController::API
  before_action :authenticate!

  private

  def authenticate!
    token = request.authorization&.match(/\ABearer (.+)\z/)&.captures&.first
    Current.user = ApiToken.authenticate(token)   # ← mechanism: ../rails-auth/ api-tokens.md
    render_unauthorized unless Current.user
  end

  def render_unauthorized
    response.set_header("WWW-Authenticate", 'Bearer realm="api"')
    render json: { error: { status: 401, message: "Invalid or missing token" } },
           status: :unauthorized                            # envelope: error-envelopes.md
  end
end
```

`ApiToken.authenticate` (and how tokens are minted/rotated/expired) is a
`../rails-auth/` concern — see its `api-tokens.md`; this file only standardizes the
*header parsing* and the *401/403 response shape*. (The two files share this exact
method/identity contract: `ApiToken.authenticate(raw)` → the user, held in
`Current.user`.)

## Deep Dive

- **Stateless, header-based.** APIs carry identity in the `Authorization` header,
  not a cookie session (see `../rails-controllers/` sessions-flash-cookies.md — APIs
  are stateless by design). Don't bolt cookie sessions onto a token API.
- **401 vs 403.** 401 = "who are you?" (no/invalid token). 403 = "I know who you
  are, you may not do this" (authorization). Authorization rules live in
  `../rails-auth/` (Pundit/Action Policy); map their failures to 403 with the same
  envelope ([error-envelopes.md](error-envelopes.md)).
- **`WWW-Authenticate` header** on 401 is the spec-correct signal of the scheme.
- **Don't reveal which part failed.** "Invalid or missing token" — not "user not
  found" vs "wrong token" — to avoid enumeration (`../rails-security/`).
- **Token in the header, not the URL.** Never accept the token as a query param —
  it leaks into logs, history, and referrers.
- **CORS interaction:** browser clients sending `Authorization` need that origin
  allowed and (only if using cookies) `credentials: true` — see [cors.md](cors.md).

## Common Pitfalls

- **Re-implementing auth here** → this is the surface only; the mechanism is
  `../rails-auth/`. Don't duplicate token issuance/verification logic across skills.
- **Returning 200 with an error body** for auth failure → clients can't detect it;
  use 401/403 status.
- **Token in query string** → logged everywhere; header only.
- **Leaking which credential failed** → enumeration vector; generic 401 message.
- **No `WWW-Authenticate` on 401** → technically non-conformant; add it.

## Verify

```bash
bin/rails server -d && sleep 3
BASE=localhost:3000
curl -s -o /dev/null -w "no-token=%{http_code}\n"   $BASE/v1/posts                                  # expect 401
curl -s -o /dev/null -w "bad-token=%{http_code}\n"  $BASE/v1/posts -H "Authorization: Bearer nope"  # expect 401
curl -s -i $BASE/v1/posts | grep -i "www-authenticate" && echo "✓ scheme advertised on 401"
# A valid token (mint via ../rails-auth/) should return 2xx:
# curl -s -o /dev/null -w "good=%{http_code}\n" $BASE/v1/posts -H "Authorization: Bearer $TOKEN"
kill "$(cat tmp/pids/server.pid)"
```

# Content Security Policy & Security Headers

Defense-in-depth response headers: a Content Security Policy (CSP) to constrain what
the browser will load/execute, plus the standard hardening headers and forced TLS.
A **Web** concern primarily — for an API the headers matter less (no browser
rendering your responses), but `force_ssl` and HSTS still apply.

## Content Security Policy

```ruby
# config/initializers/content_security_policy.rb  (Rails ships this commented — enable it)
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.script_src  :self                    # avoid :unsafe_inline — use nonces
    policy.style_src   :self
    policy.img_src     :self, :data, "https://cdn.example.com"
    policy.connect_src :self
    policy.object_src  :none
  end
  # Nonces for any unavoidable inline <script>/<style>:
  config.content_security_policy_nonce_generator = ->(req) { req.session.id.to_s }
  config.content_security_policy_nonce_directives = %w[script-src style-src]
end
```

- **Start in report-only**, watch violations, then enforce:
  `config.content_security_policy_report_only = true` → ship → review reports → flip
  to enforce. This avoids breaking the app on day one.
- **Nonces over `:unsafe_inline`.** Importmap/Turbo/Stimulus work with `:self`; use a
  nonce (`csp_meta_tag`, `nonce: true` on `javascript_*_tag`) for genuinely-inline
  scripts. `:unsafe_inline`/`:unsafe_eval` defeat much of CSP's value.
- The exact DSL/config keys evolve — verify against the Rails 8.1 Security guide.

## Standard security headers

```ruby
# config/application.rb (or an initializer)
config.force_ssl = true   # production: redirects http→https AND sets HSTS + secure cookies
config.action_dispatch.default_headers.merge!(
  "X-Frame-Options" => "SAMEORIGIN",            # clickjacking (Rails default)
  "X-Content-Type-Options" => "nosniff",        # MIME-sniffing (Rails default)
  "Referrer-Policy" => "strict-origin-when-cross-origin",
  "Permissions-Policy" => "geolocation=(), camera=()"
)
```

- **`force_ssl`** in production is the big one: it forces HTTPS, enables **HSTS**, and
  marks session/CSRF cookies `secure`. TLS termination/cert provisioning is
  `../rails-deploy/`.
- Rails already sends `X-Frame-Options` and `X-Content-Type-Options` by default;
  add `Referrer-Policy` and a `Permissions-Policy`.
- The **`secure_headers` gem** is an alternative, richer way to manage all of this in
  one place — optional; the built-in config covers most needs.

## Cookies

Session/CSRF cookies should be `httponly`, `secure` (via `force_ssl`), and
`same_site: :lax` (Rails defaults). Don't store secrets in cookies; signed/encrypted
cookies for integrity/confidentiality are `../rails-controllers/`
sessions-flash-cookies.md.

## Common Pitfalls

- **Enforcing CSP cold** → breaks inline scripts/3rd-party assets immediately; go
  report-only first, then enforce.
- **`:unsafe_inline` to "make it work"** → guts CSP; use nonces and `:self`.
- **No `force_ssl` in production** → traffic/cookies in cleartext, no HSTS; enable it.
- **CSP that forgets a needed source** (CDN, analytics, fonts) → assets silently
  blocked; the report-only phase surfaces these.
- **Treating headers as the whole story** → headers are defense-in-depth, not a
  substitute for auth/validation/escaping.

## Verify

```bash
bin/rails server -d && sleep 3
# CSP + hardening headers present:
curl -s -I localhost:3000/ | grep -iE "content-security-policy|x-frame-options|x-content-type-options|referrer-policy|permissions-policy"
kill "$(cat tmp/pids/server.pid)"
# force_ssl on in production config:
grep -rn "force_ssl" config/environments/production.rb
# CSP initializer is enabled (not all-commented):
grep -q "^[^#].*content_security_policy" config/initializers/content_security_policy.rb && echo "✓ CSP active"
```

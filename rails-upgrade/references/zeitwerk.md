# Autoloader Migration — Classic → Zeitwerk

Rails 6 introduced Zeitwerk and **Rails 7 removed the classic autoloader** — so any
app crossing into 7.x **must** be on Zeitwerk first (confirm the exact
version boundary against the Rails upgrade guide for your source version). This is the single most common
"works in development, breaks in production" upgrade trap, because production
**eager-loads** everything and exposes naming mismatches that lazy dev loading hides.

## Why it bites in production specifically

- **Development** lazy-loads: a misnamed file is only loaded when first referenced, so
  the app boots fine and tests touching that path may pass.
- **Production** eager-loads on boot: Zeitwerk walks every file and asserts each maps
  to the constant its path implies. A `app/models/api_client.rb` defining
  `APIClient` (not `ApiClient`) **fails to boot in production** while working in dev.
- That's why you run `zeitwerk:check` (which eager-loads) before deploying, and test
  in `RAILS_ENV=production`.

## Migrating

```ruby
# config/application.rb — Rails 6.x lets you opt in before the 7.0 forced switch:
config.load_defaults 6.0          # Zeitwerk is the default from 6.0
# (legacy apps may have: config.autoloader = :classic  ← remove this)
```

```bash
bin/rails zeitwerk:check          # reports files whose name doesn't match their constant
```

Fix what it reports — the usual offenders:

- **Acronyms:** `APIClient`/`HTTPServer` vs file `api_client.rb`/`http_server.rb`.
  Either rename the file/constant, or teach Zeitwerk via
  `Rails.autoloaders.main.inflector.inflect("api" => "API", ...)`.
- **Files not defining the expected constant** (a file that defines nothing, or the
  wrong name) → rename or split.
- **`require`/`require_dependency`** of autoloadable files → remove them; let Zeitwerk
  manage loading. Manual `require` of app code fights the autoloader.
- **Nonstandard `autoload_paths`** → Zeitwerk has stricter rules; align directory
  structure to constant nesting.

## Do this on Rails 6 first

Migrate to Zeitwerk **while still on Rails 6** (where classic is still available as a
fallback to compare against), get `zeitwerk:check` clean and the suite green, deploy,
*then* proceed to 7.0. Don't enter 7.0 with autoloading errors — you'll have removed
your fallback.

## Common Pitfalls

- **Testing only in development** → lazy loading hides the failures; run
  `RAILS_ENV=production bin/rails zeitwerk:check` and boot prod locally.
- **Acronym constants** without an inflector rule → eager-load failures; add inflections.
- **Leftover `require_dependency`/`require` of app files** → conflicts with Zeitwerk;
  remove them.
- **Crossing into Rails 7 with a dirty `zeitwerk:check`** → app won't boot in prod and
  classic is gone; finish on 6.x.
- **`config.autoloader = :classic` left in config** → ignored/removed in 7; delete it.

## Verify

```bash
bin/rails zeitwerk:check                                  # clean = all files map to constants
RAILS_ENV=production bin/rails zeitwerk:check             # the real test: eager-load like prod
RAILS_ENV=production bin/rails runner "puts 'eager-load boots'"   # prod boot succeeds
bin/rails test 2>/dev/null || bundle exec rspec 2>/dev/null
```

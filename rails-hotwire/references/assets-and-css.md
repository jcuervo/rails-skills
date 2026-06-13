# Assets & CSS (working within the chosen stack)

The **asset pipeline (Propshaft), CSS framework, and JS approach were chosen at
scaffold time** (`-a`/`-c`/`-j` — owned by `../rails-scaffold/` frontend-stack.md).
This reference does **not** re-pick them; it explains how to *work within* whatever
the app already has, and how Hotwire's JS (Stimulus/Turbo) gets served. Detect the
stack first, then follow the matching section.

## Detect what you're in

```bash
bundle list | grep -E "propshaft|sprockets"                  # asset pipeline
test -f config/importmap.rb && echo "JS: import maps"
test -f package.json && grep -q '"build"' package.json && echo "JS: bundler (esbuild/bun/vite)"
grep -E "tailwind|bootstrap|bulma|dartsass|cssbundling" Gemfile.lock   # CSS
```

## Propshaft (Rails 8 default pipeline)

Propshaft serves **already-built, digested** assets — it does not transpile or
bundle. It expects CSS/JS to arrive either as static files or written into
`app/assets/builds/` by a bundler. `<%= stylesheet_link_tag "application" %>` /
`javascript_importmap_tags` reference digested paths.

- Add static images/fonts under `app/assets/`; reference with `asset_path`/
  `image_tag` so the digest resolves.
- Don't expect Sprockets-style `//= require` directives — Propshaft has no manifest
  concatenation; bundling (if any) is the JS tool's job.

## JS path A — import maps (default)

```ruby
# config/importmap.rb
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin_all_from "app/javascript/controllers", under: "controllers"
```

- No Node, no build step — pin packages, ship ES modules. Stimulus controllers in
  `app/javascript/controllers` auto-register via `controllers/index.js`.
- Add a package: `bin/importmap pin <name>` (vendors or pins a CDN URL).
- **Limit:** packages that require a build (JSX/TS, heavy deps) don't work via
  pinning → that app was scaffolded with a bundler instead.

## JS path B — bundler (esbuild / Bun / Vite)

```jsonc
// package.json scripts (jsbundling-rails)
"build": "esbuild app/javascript/*.* --bundle --outdir=app/assets/builds"
```

- The bundler writes to `app/assets/builds/`, which Propshaft then serves. Run it
  via `bin/dev` (Procfile.dev runs Rails + `yarn build --watch` + CSS watch).
- Import Stimulus controllers explicitly in the JS entrypoint (no `pin_all_from`).
- Needs Node installed; `bin/setup`/`bin/dev` drive it.

## CSS — within the chosen framework

- **Tailwind** (`tailwindcss-rails`): edit `app/assets/tailwind/application.css`;
  the gem runs a standalone build (no Node) via `bin/dev`. Use utility classes in
  views/components.
- **Bootstrap/Bulma/Sass**: the scaffold wired `dartsass-rails`/`cssbundling-rails`;
  edit the source stylesheet and let the watcher compile into `builds/`.
- **Plain CSS**: hand-written files served by Propshaft.

Run the dev stack with `bin/dev` (Foreman/Procfile.dev) so CSS/JS watchers run
alongside the server — `bin/rails server` alone won't rebuild assets.

## Common Pitfalls

- **Editing built output** in `app/assets/builds/` → overwritten on next build; edit
  the source and rebuild.
- **`bin/rails server` instead of `bin/dev`** with a bundler/Tailwind → stale CSS/JS
  because watchers aren't running.
- **Import-map app expecting npm build features** → won't work; that's a bundler
  app. Don't try to switch pipelines here — that's a `../rails-scaffold/` decision.
- **Stimulus controller not loading** → import maps: check `controllers/index.js`
  and the pin; bundler: check it's imported in the entrypoint.
- **Forgetting `assets:precompile` in production** → unstyled/JS-less app; the
  deploy step handles it (`../rails-deploy/`).

## Verify

```bash
bin/rails assets:precompile >/dev/null 2>&1 && echo "✓ assets compile"
bin/rails server -d && sleep 3
curl -fsS localhost:3000/ -o /tmp/h.html
grep -qE "stylesheet|/assets/" /tmp/h.html && echo "✓ digested CSS linked"
grep -qE "importmap|/assets/.*\.js|builds/" /tmp/h.html && echo "✓ JS served"
kill "$(cat tmp/pids/server.pid)"
```

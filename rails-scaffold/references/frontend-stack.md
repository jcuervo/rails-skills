# Frontend Stack (full-stack apps only)

Three flag-level picks for a full-stack app: **asset pipeline** (`-a`), **CSS**
(`-c`), and **JavaScript** (`-j`), plus the keep/skip on **Hotwire**. This is the
bootstrap layer only — ViewComponent/Phlex, Stimulus controllers, Turbo Frames/
Streams, and Action Cable broadcasting are owned by `../rails-hotwire/SKILL.md`.

**Skip this entire file for API-only apps** — `--api` removes views and assets.

Ask the three as separate `AskUserQuestion` menus (or one multi-question call).

## 3a. Asset pipeline → `-a/--asset-pipeline`

| Option | `-a` value | One-line trade-off |
|---|---|---|
| **Propshaft** *(Recommended)* | `propshaft` | Rails 8 default. Serves digested assets; no transpilation/compilation magic. Pairs with import maps or a JS bundler that writes to `app/assets/builds`. |
| Sprockets | `sprockets` | Legacy pipeline with built-in transpilation/concatenation. Pick only to match an existing app or a gem that hard-depends on it. |
| none | `--skip-asset-pipeline` | No pipeline at all; you own asset serving. Rare for full-stack. |

Decision: **Propshaft unless** you're matching an existing Sprockets app or need a
Sprockets-only gem. Propshaft + (import maps or esbuild) is the modern path.

## 3b. CSS → `-c/--css`

| Option | `-c` value | One-line trade-off |
|---|---|---|
| **Tailwind** *(Recommended)* | `tailwind` | Utility-first; `tailwindcss-rails` wires a standalone build, no Node required. Fast to ship consistent UI. |
| Bootstrap | `bootstrap` | Component-first; familiar, batteries-included; pulls in a JS bundler for its components. |
| Bulma | `bulma` | Lightweight, class-based, no JS dependency. |
| PostCSS | `postcss` | Bring-your-own modern CSS via PostCSS plugins. |
| Sass | `sass` | Sass/SCSS via `dartsass-rails`. Pick when you have existing SCSS. |
| plain CSS | *(omit `-c`)* | Just Propshaft + hand-written CSS. Zero framework weight. |

Decision: **Tailwind** for greenfield UI velocity without Node. Choose **Bootstrap**
if the team already knows it or wants prebuilt components; **plain CSS** for tiny
apps or when a designer owns the stylesheets. Note: passing `-c bootstrap`/`-c
sass` may pull in a JS bundler — coordinate with 3c.

## 3c. JavaScript → `-j/--javascript`

| Option | `-j` value | One-line trade-off |
|---|---|---|
| **Import maps** *(Recommended)* | `importmap` | Rails 8 default. No Node, no build step — pin packages, ship ES modules straight to the browser. Best for Hotwire-centric apps. |
| esbuild | `esbuild` | Fast bundling when you need npm packages that don't work via import maps. Adds Node + a build step. |
| Bun | `bun` | esbuild-class bundling/runtime via Bun; fast, single binary. |
| Rollup | `rollup` | Bundler focused on smaller output; less common for Rails. |
| Webpack | `webpack` | Heaviest; pick only to match an existing Webpacker-era setup. |

Decision: **Import maps** unless you depend on npm packages that need bundling
(heavy client libs, packages shipping JSX/TS) — then **esbuild** (or Bun). More on
Vite below.

> **Vite:** not a built-in `-j` value. If the user wants Vite, scaffold with
> `-j esbuild` (or `--skip-javascript`) and let `../rails-hotwire/` install
> `vite_rails` afterward. Record "Vite (post-generate)" in `STACK.md`.

## 3d. Hotwire → keep or `--skip-hotwire`

| Option | Flag | When |
|---|---|---|
| **Keep Turbo + Stimulus** *(Recommended)* | *(none)* | Default full-stack experience; interactive UI with minimal JS. |
| Skip Hotwire | `--skip-hotwire` | You're bringing a different frontend framework into the asset pipeline, or want a no-JS server-rendered app. |

Decision: **Keep it** for nearly all full-stack apps — it's the reason to go
full-stack. Skip only when deliberately replacing it.

## Coherent combinations

| Goal | Flags |
|---|---|
| Modern Hotwire default | *(propshaft default)* `-c tailwind` *(importmap default)* |
| Need npm packages | `-c tailwind -j esbuild` |
| Match a legacy app | `-a sprockets -j webpack` |
| Minimal server-rendered, no framework | `--skip-hotwire` *(omit `-c`)* |

## Pitfalls

- **Import maps + a package that needs a build step** → it won't work via pinning.
  Switch to esbuild/Bun rather than fighting import maps.
- **Two sources of truth for CSS** — choosing Tailwind *and* hand-writing a second
  framework's classes causes bloat and conflicts. Pick one CSS strategy.
- A bundler (`esbuild`/`bun`/`webpack`) requires **Node** installed; on a bare
  machine `bin/setup` will fail at `yarn/npm install`. Verify Node is present or
  prefer import maps.
- Picking `-c bootstrap` silently adds a JS build for Bootstrap's components —
  don't also force `-j importmap` and expect Bootstrap JS to "just work."

## Verify

```bash
# Asset pipeline
bundle list | grep -E "propshaft|sprockets"
# CSS (example: tailwind)
test -f app/assets/tailwind/application.css && echo "tailwind wired"
# JS
test -f config/importmap.rb && echo "import maps"      # OR:
test -f package.json && grep -q '"build"' package.json && echo "bundler"
# Hotwire present
bundle list | grep -E "turbo-rails|stimulus-rails"
# Everything compiles + boots
bin/rails assets:precompile && bin/rails server -d && sleep 3 \
  && curl -fsS localhost:3000/ >/dev/null && echo "root renders" \
  && kill "$(cat tmp/pids/server.pid)"
```

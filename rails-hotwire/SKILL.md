---
name: rails-hotwire
description: Build the full-stack frontend of a Rails 8.1 app with Hotwire — Turbo Drive/Frames/Streams (including Turbo 8 morphing page refreshes), Stimulus controllers, the view layer (ERB partials, ViewComponent, or Phlex), forms, assets, and rich-text editing with Action Text (the bundled Trix editor) — plus real-time features via Action Cable and Turbo Stream broadcasting over Solid Cable or Redis. Menu-driven: presents the view-layer and Cable-adapter options as vetted menus with a Recommended default, detects the app's already-chosen CSS/JS stack (owned by rails-scaffold) and existing Hotwire wiring first, and verifies pages render and live updates broadcast. Apply when building interactive UI, wiring Turbo Frames/Streams, writing Stimulus controllers, choosing a view-component approach, adding a rich-text editor, or adding live/broadcast updates. Web apps only — not API-only.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[feature-or-view]"
---

# rails-hotwire

## Purpose

Owns the **full-stack interactive frontend** of a Web app and its **real-time**
behavior (the merged real-time topic lives in `references/real-time.md` and stays
distinctly invocable). Hotwire — Turbo + Stimulus — delivers SPA-like
interactivity with server-rendered HTML and minimal custom JavaScript. This skill
covers Turbo Drive/Frames/Streams, Stimulus, the view/component layer, forms, and
asset wiring, then extends to Action Cable + Turbo Stream broadcasting for live
updates. It builds the **HTML response body**; routing, actions, and status codes
are `../rails-controllers/`.

## When to Apply

Use this skill when the task is:

- Building interactive UI with Turbo Frames / Turbo Streams
- Writing Stimulus controllers for client-side behavior
- Choosing/structuring the view layer (partials vs ViewComponent vs Phlex)
- Building forms with `form_with` and Turbo-correct responses
- Rich-text editing (Action Text / Trix) for user-authored formatted content
- Live/real-time updates (broadcasting changes to connected clients)
- Wiring assets/CSS/JS within an already-chosen stack

Do **not** use this skill when the task is:

- **API-only apps** or JSON responses → read `../rails-api/SKILL.md` (this skill is Web-only)
- Routes, controllers, actions, status codes → read `../rails-controllers/SKILL.md`
- Choosing the CSS/JS bundler **flag** for a new app → read `../rails-scaffold/SKILL.md` (it owns the `-c`/`-j` bootstrap pick; this skill works *within* that choice)
- The data/models behind the views → read `../rails-models/SKILL.md`
- Login/forms-for-auth mechanism → read `../rails-auth/SKILL.md`
- File upload UI internals (Active Storage) → read `../rails-storage/SKILL.md`
- System/integration tests for the UI → read `../rails-testing/SKILL.md`
- Fragment/Russian-doll caching of views → read `../rails-performance/SKILL.md`

## Detect Before You Generate

```bash
grep -nE "api_only" config/application.rb            # if api_only=true, STOP — this skill is Web-only
bundle list | grep -E "turbo-rails|stimulus-rails|view_component|phlex"   # Hotwire + view layer installed?
cat config/cable.yml 2>/dev/null                     # current Cable adapter (solid_cable / redis / async)
ls app/components 2>/dev/null; find app/views -name '_*.html.erb' -print -quit 2>/dev/null   # view style: components vs partials
test -f config/importmap.rb && echo "importmap" ; test -f package.json && echo "bundler (esbuild/bun/vite)"
grep -E "tailwind|bootstrap|bulma|sass" Gemfile.lock # the CSS stack scaffold chose
grep -rln "has_rich_text" app/models 2>/dev/null; grep -rn "trix" config/importmap.rb package.json 2>/dev/null   # Action Text already wired?
```

- **`config.api_only == true` → do not use this skill.** Route to `../rails-api/`.
- The **CSS/JS stack was chosen at scaffold time** (`-c`/`-j`). Detect it and work
  within it — don't re-ask or switch it here. If `STACK.md` exists, read those
  picks.
- If ViewComponent/Phlex is already installed, match it silently.
- Detect the Cable adapter from `config/cable.yml` before adding real-time.

## Menu

Two menued decisions, each via `AskUserQuestion`, Rails-default marked
**Recommended**; ask unless already installed/detected. (CSS and JS are **not**
menued here — they were chosen in `../rails-scaffold/`; this skill consumes them.)

### View layer
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **ERB partials + helpers** *(Recommended)* | Rails default; zero deps, lowest ceremony. Can sprawl into logic-in-views on big apps. | [view-layer.md](references/view-layer.md) |
| ViewComponent | Encapsulated, testable, reusable components (Ruby class + template). Best for design systems / large UIs. | [view-layer.md](references/view-layer.md) |
| Phlex | Views as pure Ruby (no template files); fast, composable, refactorable. Steeper shift from ERB. | [view-layer.md](references/view-layer.md) |

### Real-time / Cable adapter → `config/cable.yml`
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Solid Cable** *(Recommended)* | Rails 8 default; DB-backed pub/sub, no Redis to run. Great for most apps. | [real-time.md](references/real-time.md) |
| Redis | Battle-tested, lowest-latency pub/sub at high fan-out; one more service to operate. | [real-time.md](references/real-time.md) |
| Async (dev/test only) | In-process adapter for local/test; never for production multi-process. | [real-time.md](references/real-time.md) |

## Decision Flow

- **View layer:** start with **ERB partials** — they're the default and enough for
  most apps. Adopt **ViewComponent** when UI is reused across many views and you
  want unit-testable components / a design system. Choose **Phlex** when the team
  prefers Ruby-all-the-way and wants maximum composability — it's a bigger mental
  shift. Don't introduce a component framework before view sprawl is a real problem.
- **Turbo first, Stimulus second.** Reach for Turbo Frames/Streams (server-rendered
  HTML) before writing JavaScript. Add a Stimulus controller only for genuinely
  client-side behavior (toggles, drag, third-party JS) — see
  [turbo.md](references/turbo.md) then [stimulus.md](references/stimulus.md).
- **Morphing vs replacing.** Turbo 8 page-refresh morphing
  ([turbo.md](references/turbo.md)) preserves scroll/focus for "refresh the whole
  page" updates; targeted Turbo Streams for surgical updates. Use morphing for
  broadcast-driven full-page freshness, streams for specific DOM changes.
- **Real-time adapter:** Solid Cable unless you already run Redis or need its
  fan-out/latency. Keep `async` for dev/test only.

## Problem → Reference

| Task | Read |
|---|---|
| Turbo Drive/Frames/Streams, morphing page refreshes | [references/turbo.md](references/turbo.md) |
| Stimulus controllers: targets, actions, values, lifecycle | [references/stimulus.md](references/stimulus.md) |
| Choose + structure the view layer (partials / ViewComponent / Phlex) | [references/view-layer.md](references/view-layer.md) |
| Forms with `form_with`, Turbo-correct responses (422 on invalid) | [references/forms-and-responses.md](references/forms-and-responses.md) |
| Rich-text editor + content (Action Text / Trix, `has_rich_text`) | [references/rich-text.md](references/rich-text.md) |
| Real-time: Action Cable, Turbo Stream broadcasting, Solid Cable vs Redis | [references/real-time.md](references/real-time.md) |
| Assets, Propshaft, working within the chosen CSS/JS stack | [references/assets-and-css.md](references/assets-and-css.md) |

## Verify

A frontend change is unfinished until the page renders and (for real-time) an
update actually broadcasts:

```bash
bin/rails server -d && sleep 3
curl -fsS localhost:3000/ -o /dev/null -w "root=%{http_code}\n"                       # page renders (200)
curl -fsS localhost:3000/posts -o /tmp/p.html && grep -qE "turbo-frame|turbo-stream|data-controller" /tmp/p.html && echo "✓ Hotwire markup present"
bin/rails assets:precompile >/dev/null 2>&1 && echo "✓ assets compile"
kill "$(cat tmp/pids/server.pid)"
```

> **CSRF caveat for the `curl` POST checks** in this skill's references: a real Web
> app enables `protect_from_forgery`, so a raw `curl` POST without a CSRF token may
> return 422/redirect for the *wrong* reason (forgery, not your logic). Run these
> against dev with a valid token, or assert form behavior via a system test
> (`../rails-testing/`). Real-time broadcasting is verified in
> [references/real-time.md](references/real-time.md).

Then route to `../rails-testing/` for system tests, `../rails-performance/` for
fragment caching, and record the view-layer/Cable picks in `STACK.md`.

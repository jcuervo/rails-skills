# Stimulus

Stimulus adds **just enough** JavaScript behavior to server-rendered HTML, wired
through `data-` attributes. Reach for it only after Turbo can't do the job —
client-only behavior (toggles, drag, clipboard, third-party JS widgets).

## Quick Pattern

```bash
bin/rails g stimulus toggle      # creates app/javascript/controllers/toggle_controller.js
```
```js
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"
export default class extends Controller {
  static targets = ["body"]
  static values  = { open: Boolean }
  toggle() { this.openValue = !this.openValue }
  openValueChanged() { this.bodyTarget.hidden = !this.openValue }
}
```
```erb
<div data-controller="toggle" data-toggle-open-value="false">
  <button data-action="click->toggle#toggle">Toggle</button>
  <div data-toggle-target="body">…</div>
</div>
```

With import maps (default), controllers auto-register via
`app/javascript/controllers/index.js` (Stimulus `eager`/`lazy` loaders). With a
bundler (esbuild/Bun/Vite), they're imported there — see
[assets-and-css.md](assets-and-css.md).

## The building blocks

- **Controllers** — `data-controller="name"` attaches behavior to an element.
- **Targets** — `static targets = [...]` + `data-name-target="x"` → `this.xTarget`.
- **Actions** — `data-action="event->name#method"` wires DOM events to methods.
- **Values** — `static values = {...}` + `data-name-key-value` → reactive
  `this.keyValue` with a `keyValueChanged()` callback.
- **Lifecycle** — `connect()` / `disconnect()` run when the controller attaches/
  detaches (important with Turbo — see Pitfalls).

## When to pick / not pick

- **Pick Stimulus when** behavior is inherently client-side: toggling UI, debouncing
  input, integrating a JS library (charts, maps, date pickers), copy-to-clipboard,
  keyboard shortcuts.
- **Don't pick Stimulus when** Turbo can do it server-side: navigation, form
  submission, inline edit, list updates → Frames/Streams ([turbo.md](turbo.md)) are
  simpler and keep state on the server.

## Deep Dive

- **Turbo + lifecycle:** Turbo swaps DOM without full reloads, so `connect()` may
  run many times and `disconnect()` is where you clean up (remove listeners, destroy
  widgets) to avoid leaks/duplicates.
- **`connect()` idempotency:** initialize for the element's current state; don't
  assume a fresh page.
- **Outlets** (`static outlets`) let one controller reference another's controllers
  — use sparingly for cross-component coordination.
- **Don't reach into other controllers' DOM**; communicate via events
  (`this.dispatch("opened")`) so components stay decoupled.

## Common Pitfalls

- **Memory leaks / double-binding** from not cleaning up in `disconnect()` when
  Turbo re-renders → destroy third-party widgets and remove manual listeners there.
- **Writing JS for something Turbo already does** → unnecessary client state; prefer
  server-rendered Frames/Streams.
- **Controller not registering** → with a bundler, you forgot to import it; with
  import maps, check `controllers/index.js` and the pin. See
  [assets-and-css.md](assets-and-css.md).
- **Querying the DOM manually** (`document.querySelector`) instead of targets →
  brittle; use `static targets`.
- **Business logic in JS** → keep rules on the server; Stimulus is for interaction.

## Verify

```bash
ls app/javascript/controllers/                          # controller file exists
bin/rails server -d && sleep 3
curl -fsS localhost:3000/ -o /tmp/h.html && grep -qE 'data-controller=' /tmp/h.html && echo "✓ controller attached in markup"
# Import maps: the controller is pinned/served
test -f config/importmap.rb && grep -q "controllers" app/javascript/controllers/index.js 2>/dev/null && echo "✓ registered (importmap)"
bin/rails assets:precompile >/dev/null 2>&1 && echo "✓ JS compiles"
kill "$(cat tmp/pids/server.pid)"
```

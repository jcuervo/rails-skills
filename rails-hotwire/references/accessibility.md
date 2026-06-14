# Accessibility (WCAG / Section 508)

Making the Hotwire frontend usable with a keyboard and a screen reader. Most of this
is **semantic HTML and labeled forms** — which Rails' own helpers already get right
if you let them — plus the **Hotwire-specific** parts that vanilla WCAG guides miss:
Turbo navigation and Turbo Stream updates break focus and screen-reader
announcements by default, and you have to put them back.

> **Scope guard.** This builds the *mechanics*. The **conformance target** — WCAG 2.1
> AA vs 2.2 AA, which Section 508 / ADA / EN 301 549 obligations apply — is a
> policy/legal decision, not a code default. Pick the level deliberately (AA is the
> common bar); this reference gives you the patterns and a test gate, not a
> compliance verdict. Web apps only — API-only apps have no UI to make accessible.

## When this applies (the switch)

Accessibility is **baseline quality for any full-stack app** — this reference is
always reachable when you build views. Additionally, if the `STACK.md` `Compliance`
row (set by `../rails-scaffold/`) lists `wcag` or `section508`, **escalate**:
proactively offer an accessibility pass as you build each view and wire the axe test
gate below into CI. `none`/absent → still apply the baseline, just don't gate.

## Quick Pattern — the baseline every page needs

```erb
<%# app/views/layouts/application.html.erb %>
<html lang="<%= I18n.locale %>">            <%# real language for screen readers / i18n → ../rails-controllers/ %>
  <body>
    <a href="#main" class="sr-only-focusable">Skip to main content</a>   <%# skip link %>
    <header><nav aria-label="Primary"><%= render "shared/nav" %></nav></header>
    <main id="main">                          <%# one <main>, focus target after navigation %>
      <%= yield %>
    </main>
  </body>
</html>
```

- **Use real elements.** `<%= button_to %>` / `<%= link_to %>` / `<button>` — not a
  `<div onclick>`. Rails' helpers emit semantic, focusable, keyboard-operable HTML;
  a clickable `div` is none of those. One `<h1>` per page, headings in order.
- **Landmarks**: `<header>`/`<nav aria-label>`/`<main>`/`<footer>` so AT users can
  jump. **Skip link** first in the body to bypass the nav.

## Forms — label everything, link the errors

```erb
<%= form_with model: @post do |f| %>
  <%= f.label :title %>                                  <%# real <label for> — never a bare placeholder %>
  <%= f.text_field :title,
        aria: { invalid: @post.errors[:title].any?, describedby: ("title-error" if @post.errors[:title].any?) } %>
  <% if @post.errors[:title].any? %>
    <p id="title-error" class="error"><%= @post.errors[:title].to_sentence %></p>
  <% end %>
<% end %>
```

- `f.label` ties the label to the field — required for screen readers and bigger
  click targets. A `placeholder` is **not** a label.
- On an invalid submit, render an **error summary** at the top of the form, move
  focus to it, and link each field to its message with `aria-describedby` +
  `aria-invalid`. Turbo-correct invalid responses (422 + re-render) are owned by
  [forms-and-responses.md](forms-and-responses.md); this is the a11y layer on them.

## Hotwire-specific — focus & announcements (the part that's easy to miss)

Turbo replaces page content with JavaScript, so the browser's normal "new page"
focus reset and screen-reader announcement **don't fire**. You restore them:

- **Turbo Drive navigation** doesn't move focus or announce the new page. After a
  visit, move focus to the `<main>`/`<h1>` and/or announce the title via a live
  region, so a screen-reader user knows the page changed:
  ```js
  // app/javascript/controllers/focus_main_controller.js (or a turbo:load listener)
  document.addEventListener("turbo:load", () => {
    const main = document.getElementById("main")
    if (main) { main.setAttribute("tabindex", "-1"); main.focus() }
  })
  ```
- **Turbo Streams insert/replace DOM silently** — AT announces nothing. Put
  user-facing updates (flash/toast, "item added", validation results) inside an
  `aria-live` region that persists in the layout, so streamed changes are spoken:
  ```erb
  <div id="flash" aria-live="polite" aria-atomic="true"></div>   <%# target streams here; "polite" waits, "assertive" interrupts %>
  ```
- **Turbo Frames** that swap content can strand focus on a now-removed element —
  move focus into the new frame content after it loads.
- **Turbo 8's morphing page refresh is designed to preserve scroll and focus** (verify
  against your Turbo version) — for "refresh the whole page" updates, morphing
  ([turbo.md](turbo.md)) is the more accessible choice than a full replace precisely
  because it keeps the user's place.

## Stimulus & custom widgets

- **Keyboard parity.** Anything you wire with a Stimulus `click` action must also work
  from the keyboard. Prefer a real `<button>` (Enter/Space free); if you must use a
  non-interactive element, add `tabindex="0"`, a `role`, and a `keydown` handler.
- **Custom widgets need ARIA roles/state** (`role`, `aria-expanded`, `aria-selected`,
  `aria-controls`) kept in sync as the controller toggles — follow the WAI-ARIA
  Authoring Practices pattern for the widget. See [stimulus.md](stimulus.md).
- **Respect `prefers-reduced-motion`** for any controller-driven animation/auto-scroll.

## Color, contrast & focus visibility (CSS stack)

- **Contrast:** body text needs ≥ 4.5:1 (WCAG AA), large text ≥ 3:1. This is a
  design-token decision in the chosen CSS stack — Tailwind/Bootstrap palettes still
  need checking. See [assets-and-css.md](assets-and-css.md).
- **Never kill focus outlines** (`outline: none`) without a visible replacement —
  keyboard users navigate by it. Use `:focus-visible` styles.
- **Don't encode meaning in color alone** (red/green status needs an icon or text too).

## Images & rich text

- `image_tag "x.png", alt: "..."` — meaningful `alt`, or `alt: ""` for purely
  decorative images (empty, not missing). Active Storage images need alt from your
  model/UI.
- Action Text/Trix content: provide an alt-text affordance for embedded images and
  ensure the rendered output is semantic — see [rich-text.md](rich-text.md).

## Common Pitfalls

- **`<div>`/`<span>` with a click handler** instead of `<button>` → not focusable, no
  keyboard, no role. The single most common failure; Rails helpers avoid it for free.
- **Placeholder-as-label** → empty accessible name; screen reader reads nothing.
- **Turbo Stream toast with no live region** → sighted users see it, AT users never
  hear it. Stream into an `aria-live` container.
- **Focus lost after a Turbo navigation/frame swap** → keyboard user is dumped at the
  top of `<body>` or on a detached node. Manage focus explicitly.
- **`outline: none`** in the CSS reset with no `:focus-visible` replacement.
- **Icon-only buttons** with no `aria-label` → "button" is all that's announced.
- **Auto-playing/looping motion** ignoring `prefers-reduced-motion`.

## Verify

Automated checks catch ~30–40% of issues — **also do a manual keyboard + screen-reader
pass.** Wire axe into system tests (`../rails-testing/`):

```ruby
# Gemfile (test group) — RSpec system specs:
#   gem "axe-core-rspec" and gem "axe-core-capybara"   (dequelabs)
# or framework-agnostic auto-audit (works with Minitest system tests):
#   gem "capybara_accessibility_audit" (thoughtbot)

# RSpec system spec:
it "home page has no WCAG AA violations" do
  visit root_path
  expect(page).to be_axe_clean.according_to(:wcag2aa)   # set the target level deliberately (policy)
end
```

```bash
# the gem resolves on this app (don't assume a version):
bundle add axe-core-rspec axe-core-capybara --group test && bundle exec rspec spec/system
# baseline structure is present in the rendered HTML:
bin/rails server -d && sleep 3
curl -fsS localhost:3000/ -o /tmp/a.html
grep -q '<main' /tmp/a.html && echo "✓ <main> landmark"
grep -qE 'lang="[a-z]' /tmp/a.html && echo "✓ html lang set"
grep -qiE 'skip to main|sr-only' /tmp/a.html && echo "✓ skip link"
kill "$(cat tmp/pids/server.pid)"
```

Manual gate (no tool replaces it): **Tab through the page** — every interactive
element reachable, in order, with a visible focus ring; **operate it with the
keyboard only**; and run one flow under VoiceOver/NVDA confirming navigations and
streamed updates are announced. Record the chosen WCAG target in `STACK.md`.

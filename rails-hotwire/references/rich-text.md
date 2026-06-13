# Rich Text (Action Text)

WYSIWYG rich-text content with the bundled **Trix** editor, stored in its own
`action_text_rich_texts` table and associated to any model via `has_rich_text`.
Embedded images and files attach through **Active Storage** (`../rails-storage/`).
This is a Web-only view-layer concern — API-only apps don't render Trix. `Article` is
this reference's running example (the rest of the skill uses `Post`).

## Quick Pattern

```bash
# Installer wiring drifts across releases (importmap vs package.json, whether it also runs
# active_storage:install) — run `bin/rails action_text:install --help` to confirm for your version.
bin/rails action_text:install   # adds Trix + trix CSS, creates the rich_text migration
bin/rails db:migrate            # action_text_rich_texts table
# Active Storage is required for embeds — the installer wires it; on an existing app:
# bin/rails active_storage:install && bin/rails db:migrate
```
```ruby
# app/models/article.rb
class Article < ApplicationRecord
  has_rich_text :content        # no DB column needed; stored in action_text_rich_texts
end
```
```erb
<%# the form — rich_text_area renders the Trix editor + hidden input %>
<%= form_with model: @article do |f| %>
  <%= f.rich_text_area :content %>
  <%= f.submit %>
<% end %>
```
```ruby
# controller strong params — permit it like any attribute (../rails-controllers/)
params.expect(article: [:title, :content])
```
```erb
<%# render it — output is already sanitized HTML %>
<%= @article.content %>
```

## When to pick / not pick

- **Pick** Action Text when users author formatted prose — comments, articles,
  descriptions, notes — with bold/lists/quotes/links and inline image uploads, and
  you want it stored and rendered the Rails-native way.
- **Don't pick** it for: a Markdown-authored site (store Markdown text, render with a
  Markdown gem); a structured block editor (Editor.js/ProseMirror via a Stimulus
  controller — [stimulus.md](stimulus.md)); plain text (a normal `text` column + `text_area`); or
  **API-only** apps (no view layer — return the stored HTML/plain text instead).

## Deep Dive

- **Storage model:** `has_rich_text :content` creates a polymorphic
  `ActionText::RichText` record — no column on the host table. Add another named
  field with a second `has_rich_text :summary` + `rich_text_area :summary`.
- **Querying/eager-load:** the association is `rich_text_content`; `with_rich_text_content`
  (and `..._and_embeds`) avoids N+1 when listing records that render their bodies
  (`../rails-performance/`).
- **Attachments:** images dropped into Trix upload via Active Storage and render as
  attachments. Use `variant`-friendly processing (`../rails-storage/`). Galleries and
  remote content render through Action Text's attachment partials, which you can
  override in `app/views/action_text/`.
- **Plain text + search:** `article.content.to_plain_text` strips markup — index that
  for search (`../rails-models/` search reference), not the HTML.
- **Rendering safely:** `<%= article.content %>` sanitizes on output; the editor and
  the renderer share an allowlist. The allowlist is customizable, but the exact config
  surface (`ActionText::ContentHelper.allowed_tags` vs `config.action_text.*`) has
  moved between Rails versions — confirm against the
  [Action Text guide](https://guides.rubyonrails.org/action_text_overview.html) for yours.
- **Styling Trix:** the editor is plain HTML; style `trix-editor` / `.trix-content`
  in your chosen CSS stack ([assets-and-css.md](assets-and-css.md)). Toolbar
  customization is done by overriding the Trix toolbar partial.

## Common Pitfalls

- **Active Storage not installed** → image embeds raise on upload; ensure it's set up
  (`../rails-storage/`).
- **Forgetting to permit `:content`** → the field silently doesn't save; permit it in
  strong params (`../rails-controllers/`).
- **N+1 when listing bodies** → use `with_rich_text_content_and_embeds`.
- **Trusting raw HTML** → don't `raw`/`html_safe` Action Text output or bypass the
  sanitizer; render the attribute directly so it's sanitized.
- **Indexing HTML for search** → index `to_plain_text`, not the markup.
- **API-only app** → there's no Trix; this reference doesn't apply (return stored
  content as data via `../rails-api/`).

## Verify

```bash
# Model wiring: the rich-text association exists and round-trips.
bin/rails runner "a = Article.new(content: '<div>Hi <strong>there</strong></div>'); p a.content.to_plain_text; p a.content.body.is_a?(ActionText::Content)" 2>/dev/null

# The editor renders in the form, and the show page renders sanitized HTML:
bin/rails server -d && sleep 3
curl -fsS localhost:3000/articles/new -o /tmp/f.html && grep -q "trix-editor" /tmp/f.html && echo "✓ Trix editor present"
bin/rails assets:precompile >/dev/null 2>&1 && echo "✓ assets (incl. trix css) compile"
kill "$(cat tmp/pids/server.pid)"
```

Then route to `../rails-storage/` (embedded uploads + variants), `../rails-performance/`
(eager-load bodies), and `../rails-testing/` (system test the editor with Capybara).

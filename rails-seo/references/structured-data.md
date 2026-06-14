# Structured Data — JSON-LD (Schema.org)

Structured data tells search engines *what a page is about* in a machine-readable
form, unlocking **rich results** (article bylines, product price/ratings, breadcrumb
trails, FAQ accordions). Google's recommended format is **JSON-LD** in a
`<script type="application/ld+json">` tag — no gem required, no microdata woven into
markup. A **Web** concern. This is a practice (not a pick-one menu): add the
`@type`s that match the content you actually have.

## Quick Pattern

Build the hash in a helper and render it as JSON in the layout's `:head` slot:

```ruby
# app/helpers/structured_data_helper.rb
module StructuredDataHelper
  def json_ld(hash)
    tag.script(
      raw(JSON.generate(hash.merge("@context" => "https://schema.org"))),
      type: "application/ld+json"
    )
  end
end
```

```erb
<%# app/views/posts/show.html.erb %>
<% content_for :head do %>
  <%= json_ld(
    "@type" => "Article",
    "headline" => @post.title,
    "datePublished" => @post.published_at.iso8601,
    "dateModified" => @post.updated_at.iso8601,
    "author" => { "@type" => "Person", "name" => @post.author.name },
    "image" => @post.cover_url,
    "mainEntityOfPage" => post_url(@post)
  ) %>
<% end %>
```

A site-wide `Organization` (logo, name, sameAs social links) belongs in the layout;
per-page types (`Article`, `Product`, `BreadcrumbList`, `FAQPage`, `Event`) belong in
the view. Multiple `<script type="application/ld+json">` blocks on one page are fine.

## When to use which `@type`

- **Article / BlogPosting / NewsArticle** — editorial content; enables byline + date.
- **Product** (+ nested `Offer`, `AggregateRating`) — store pages; price/availability/
  stars. Only mark up ratings you actually display.
- **BreadcrumbList** — your breadcrumb trail, so Google shows the hierarchy in the SERP.
- **Organization / WebSite** — site-wide identity; `WebSite` + `SearchAction` can
  surface a sitelinks search box.
- **FAQPage / HowTo / Event / LocalBusiness / Recipe** — only when the page genuinely
  is that thing.

Don't add a type just to chase a rich result — Google penalizes structured data that
doesn't match visible page content.

## Deep Dive

- **JSON-LD only.** Microdata/RDFa works but mixes data into markup; JSON-LD keeps it
  in one block, easy to generate from Ruby and to escape correctly.
- **Use `JSON.generate` + `raw`**, not string interpolation — it escapes values so a
  title with a quote or `</script>` can't break the page or inject markup. Never
  interpolate user content into the `<script>` by hand.
- **Match the visible page.** Marked-up price/rating/date must equal what the user
  sees. Mismatches are a manual-action risk.
- **Absolute URLs** for `image`, `url`, `logo` (`*_url` helpers).
- **i18n:** generate localized strings via `I18n` (`../rails-controllers/references/i18n.md`).
- **Idempotent:** it's a helper + `content_for` — safe to add incrementally, one
  `@type` at a time.

## Common Pitfalls

- **Hand-built JSON string** → a quote/newline in a title produces invalid JSON or an
  injection. Always `JSON.generate`.
- **Structured data that doesn't match visible content** (phantom reviews, hidden
  prices) → manual action / lost rich results.
- **Relative image/url values** → ignored; use absolute URLs.
- **Marking up a type you don't render** (e.g. `FAQPage` with no visible FAQ) →
  ineligible or penalized.
- **Forgetting `@context`** → not parsed as Schema.org; the helper above adds it once.

## Verify

```bash
bin/rails server -d && sleep 3
curl -s localhost:3000/posts/1 | \
  ruby -e 'require "json"; html=STDIN.read;
    blocks=html.scan(/<script type="application\/ld\+json">(.+?)<\/script>/m).flatten
    abort "no JSON-LD found" if blocks.empty?
    blocks.each { |b| d=JSON.parse(b); puts "✓ #{d["@type"]} (@context=#{d["@context"]})" }'
kill "$(cat tmp/pids/server.pid)"
```

Then validate the live URL with Google's **Rich Results Test** and the **Schema
Markup Validator** (validator.schema.org) — they catch missing required properties
the JSON parse can't.

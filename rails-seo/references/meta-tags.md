# Meta Tags — title, description, canonical, Open Graph, Twitter/X cards

The per-page `<head>` metadata search engines and link unfurlers read: a unique
`<title>` and meta `description`, a `canonical` link, and Open Graph + Twitter Card
tags so shared links render a title/description/image. A **Web** concern — for an
API-only app the rendering frontend owns this. Two ways to do it: a native helper
(Recommended) or the `meta-tags` gem.

## Option A — Native `content_for` + helper *(Recommended)*

### Quick Pattern

```erb
<%# app/views/layouts/application.html.erb — inside <head> %>
<title><%= content_for?(:title) ? yield(:title) : "Acme" %></title>
<meta name="description" content="<%= content_for?(:description) ? yield(:description) : "Acme — the default tagline." %>">
<link rel="canonical" href="<%= canonical_url %>">
<%= yield :head %>   <%# pages append OG/Twitter/JSON-LD here %>
```

```ruby
# app/helpers/seo_helper.rb
module SeoHelper
  # Self-referencing canonical: current URL minus tracking params, https, no trailing junk.
  def canonical_url
    url_for(only_path: false, host: request.host, protocol: "https").split("?").first
  end

  # Renders OG + Twitter tags from a hash; call from a view via content_for(:head).
  def social_tags(title:, description:, image:, type: "website")
    safe_join([
      tag.meta(property: "og:title", content: title),
      tag.meta(property: "og:description", content: description),
      tag.meta(property: "og:type", content: type),
      tag.meta(property: "og:image", content: image),
      tag.meta(property: "og:url", content: canonical_url),
      tag.meta(name: "twitter:card", content: "summary_large_image"),
      tag.meta(name: "twitter:title", content: title),
      tag.meta(name: "twitter:image", content: image)
    ], "\n")
  end
end
```

```erb
<%# app/views/posts/show.html.erb %>
<% content_for :title, "#{@post.title} — Acme" %>
<% content_for :description, @post.summary.truncate(160) %>
<% content_for :head do %>
  <%= social_tags(title: @post.title, description: @post.summary.truncate(160),
                  image: @post.cover_url) %>
<% end %>
```

### When to pick / not pick

- **Pick** for full control, zero dependencies, and when the layout already uses
  `content_for`. The default for most apps.
- **Don't pick** when dozens of page types each need the full OG/Twitter set and the
  per-view `content_for` boilerplate becomes repetitive — the gem centralizes it.

## Option B — `meta-tags` gem

### Quick Pattern

```ruby
# Gemfile
gem "meta-tags"
```

```erb
<%# layout <head> %>
<%= display_meta_tags(site: "Acme", reverse: true, separator: "—",
      og: { type: "website" }, twitter: { card: "summary_large_image" }) %>
```

```ruby
# a controller action
def show
  @post = Post.find(params[:id])
  set_meta_tags title: @post.title,
                description: @post.summary.truncate(160),
                canonical: post_url(@post),
                og:      { title: @post.title, image: @post.cover_url },
                twitter: { image: @post.cover_url }
end
```

### When to pick / not pick

- **Pick** for many page types, site-wide defaults (site name, separator, default OG
  image), and `set_meta_tags` from the controller without touching the view.
- **Don't pick** if you want zero dependencies or the native helper already covers you.

## Deep Dive

- **Every indexable page needs a unique title (~50–60 chars) and description
  (~150–160 chars).** Duplicate or missing titles are the most common on-page SEO
  defect. Truncate model-derived descriptions; never dump raw HTML body.
- **One canonical per page, self-referencing**, absolute `https` URL, no tracking
  params (`?utm_*`, `?ref=`). This is how you collapse duplicate URLs (pagination,
  filters, trailing-slash variants) into one indexed page.
- **Open Graph vs Twitter:** OG (`og:*`) covers Facebook/LinkedIn/Slack/iMessage;
  Twitter/X adds `twitter:card` (`summary_large_image` for a big image). Provide an
  absolute `og:image` URL (~1200×630). Generating that image asset is
  `../rails-storage/` (variants) — this skill just references its URL.
- **i18n:** translate title/description through `I18n`
  (`../rails-controllers/references/i18n.md`) and emit `hreflang` alternates (see
  slugs-and-urls.md).
- **Idempotent:** both approaches are layout/helper edits — safe to add to an
  existing app. Adding the gem alongside an existing native helper is the thing to
  avoid; pick one.

## Common Pitfalls

- **Same `<title>` on every page** → Google treats pages as duplicates. Set a unique
  one per action.
- **Description longer than ~160 chars or empty** → truncated or auto-generated
  snippet. Truncate deliberately.
- **Relative `og:image`/`canonical`** → unfurlers and Google need absolute URLs with
  host + scheme. Build them with `*_url` helpers, not `*_path`.
- **Leaving tracking params in the canonical** → splits link equity across URLs.
- **Running both the native helper and the gem** → two `<title>` tags. Detect and use
  one.

## Verify

```bash
bin/rails server -d && sleep 3
PAGE=/posts/1                                  # a content page
curl -s localhost:3000$PAGE > /tmp/page.html
grep -c "<title>" /tmp/page.html                                   # expect exactly 1
grep -oE "<title>[^<]*</title>" /tmp/page.html
grep -oE "<meta name=\"description\" content=\"[^\"]{1,160}\"" /tmp/page.html && echo "✓ description ≤160"
grep -oE "<link rel=\"canonical\" href=\"https://[^\"]+\"" /tmp/page.html && echo "✓ absolute canonical"
grep -oE "og:(title|image|url)" /tmp/page.html | sort -u           # expect og:title/image/url
grep -q "twitter:card" /tmp/page.html && echo "✓ twitter card"
kill "$(cat tmp/pids/server.pid)"
```

Paste the URL into the OG validators (Facebook Sharing Debugger, X Card Validator) to
confirm the preview renders.

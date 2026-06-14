# Sitemaps & robots.txt

An XML **sitemap** tells search engines which URLs exist and when they changed; a
`robots.txt` tells crawlers what to fetch and where the sitemap is. Both are **Web**
concerns, but even an API-only app can host them for a separate frontend. Two ways to
build the sitemap: dynamic Rails rendering (Recommended) or the `sitemap_generator`
gem.

## Option A — Dynamic Rails-rendered sitemap *(Recommended)*

### Quick Pattern

```ruby
# config/routes.rb
get "sitemap.xml", to: "sitemaps#show", defaults: { format: "xml" }
```

```ruby
# app/controllers/sitemaps_controller.rb
class SitemapsController < ApplicationController
  def show
    @posts = Post.published.select(:id, :slug, :updated_at)
    expires_in 6.hours, public: true     # cache at the edge/browser; see ../rails-performance/
    fresh_when(@posts)                    # 304 when nothing changed
  end
end
```

```ruby
# app/views/sitemaps/show.xml.builder
xml.instruct!
xml.urlset xmlns: "http://www.sitemaps.org/schemas/sitemap/0.9" do
  xml.url do
    xml.loc root_url
    xml.changefreq "daily"
  end
  @posts.each do |post|
    xml.url do
      xml.loc post_url(post)
      xml.lastmod post.updated_at.iso8601
      xml.changefreq "weekly"
    end
  end
end
```

### When to pick / not pick

- **Pick** for most apps: always current (no regeneration step), no gem, and you
  cache it with HTTP caching / a fragment so rendering cost is amortized.
- **Don't pick** once you cross a single sitemap's limits (**50,000 URLs / 50 MB
  uncompressed**) or rendering the query gets slow — you then need a sitemap *index*
  splitting into multiple files, which is exactly what the gem automates.

## Option B — `sitemap_generator` gem

### Quick Pattern

```ruby
# Gemfile
gem "sitemap_generator"
```

```ruby
# config/sitemap.rb
SitemapGenerator::Sitemap.default_host = "https://acme.com"
SitemapGenerator::Sitemap.create do
  add "/", changefreq: "daily"
  Post.published.find_each { |p| add post_path(p), lastmod: p.updated_at }
end
```

```bash
bin/rails -T sitemap                 # confirm the gem's tasks/names for the installed version
bin/rails sitemap:refresh            # builds public/sitemap.xml(.gz) + index, pings engines
# (or sitemap:refresh:no_ping). Run it from a recurring job — see ../rails-jobs/ for scheduling.
```

> `sitemap:refresh*` are tasks the **gem** adds (not Rails core); run `bin/rails -T sitemap`
> to confirm the exact names for the version in your `Gemfile.lock`.

### When to pick / not pick

- **Pick** for large catalogs: it auto-splits into a sitemap index, gzips, writes to
  `public/` (or a storage service), and pings search engines.
- **Don't pick** for a small site where a dynamic route is simpler and never goes stale.

## robots.txt (a practice, not a menu)

Rails ships a static `public/robots.txt`. Make it **environment-aware** so staging
isn't indexed, and point it at the sitemap. Render it dynamically:

```ruby
# config/routes.rb
get "robots.txt", to: "robots#show", defaults: { format: "text" }
```

```erb
<%# app/views/robots/show.text.erb  —  RobotsController#show needs no body; the text format skips the HTML layout %>
<% if Rails.env.production? %>
User-agent: *
Disallow: /admin
Sitemap: <%= sitemap_url %>
<% else %>
User-agent: *
Disallow: /
<% end %>
```

- **Blocking a page in `robots.txt` does not remove it from the index** — it stops
  crawling, but a blocked URL can still appear (without a snippet). To *de-index*, let
  it be crawled and send `noindex` (a `<meta name="robots" content="noindex">` or the
  `X-Robots-Tag` header). Don't block-and-noindex the same URL — the crawler can't see
  the `noindex` if it can't fetch the page.

## Deep Dive

- **Only include canonical, indexable, 200-OK URLs** in the sitemap — no redirects,
  no `noindex` pages, no params. Mismatched canonical vs sitemap URLs confuse
  crawlers.
- **`lastmod` must be honest** (`updated_at.iso8601`). Faking "now" everywhere trains
  crawlers to ignore it.
- **Submit once** in Google Search Console / Bing Webmaster Tools, and reference it in
  `robots.txt`; you don't need to resubmit on every change.
- **Refresh cadence (gem):** drive `sitemap:refresh` from a recurring job
  (`../rails-jobs/`), not a request.
- **Idempotent:** adding a route + controller is safe on an existing app; if a static
  `public/robots.txt` exists, replace it with the dynamic route (remove the file so it
  doesn't shadow the route).

## Common Pitfalls

- **`public/robots.txt` shadowing a `robots.txt` route** → Rails serves the static
  file first. Delete the static file when you switch to dynamic.
- **Staging indexed** → forgot the environment guard; duplicate content competes with
  production.
- **Sitemap lists non-canonical/redirecting URLs** → crawl budget wasted, indexing
  signals muddled.
- **One giant sitemap past 50k URLs** → silently ignored beyond the limit; split via
  an index (the gem).
- **Expecting `Disallow` to de-index** → it only blocks crawling; use `noindex`.

## Verify

```bash
bin/rails server -d && sleep 3
curl -s -o /dev/null -w "sitemap %{http_code}\n" localhost:3000/sitemap.xml         # expect 200
curl -s localhost:3000/sitemap.xml | head -3 | grep -q "<urlset\|<sitemapindex" && echo "✓ well-formed"
curl -s localhost:3000/sitemap.xml | ruby -rrexml/document -e 'REXML::Document.new(STDIN.read); puts "✓ valid XML"'
curl -s localhost:3000/robots.txt                                                   # inspect: Sitemap: line + env-correct Disallow
kill "$(cat tmp/pids/server.pid)"
# Gem path:
grep -q sitemap_generator Gemfile && bin/rails sitemap:refresh:no_ping && ls -l public/sitemap*.xml*
```

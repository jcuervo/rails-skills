---
name: rails-seo
description: Make a Rails 8.1 app discoverable and correctly indexed by search engines and link unfurlers — per-page title/description/canonical meta tags, Open Graph + Twitter/X cards, JSON-LD structured data (Schema.org), XML sitemaps, robots.txt, and human/SEO-friendly URLs (slugs). Menu-driven for the genuine choices (meta-tag approach, sitemap approach, slug strategy) with a Rails-idiomatic Recommended default; detects what's already installed first, branches on config.api_only (a JSON API has no HTML to index — the rendering frontend owns its SEO), and verifies every tag/sitemap/slug actually renders and is crawlable. Apply when adding meta tags, social-share previews, structured data, a sitemap or robots.txt, or pretty URLs. Page-speed/Core Web Vitals is rails-performance; routing mechanics is rails-controllers; the view layer is rails-hotwire.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[seo-area]"
---

# rails-seo

## Purpose

Make the pages a Rails app **renders as HTML** discoverable, correctly indexed, and
nicely previewed when shared: per-page `<title>` and meta description, canonical
URLs, Open Graph + Twitter/X cards, JSON-LD **structured data** (Schema.org), an XML
**sitemap**, a `robots.txt`, and **SEO-friendly URLs** (slugs). This is the
on-page/technical-SEO layer Rails doesn't ship by default — it's all menu-driven
with a Rails-idiomatic, zero-dependency default marked **Recommended**, and the
heavier gem offered as the alternative for scale. It is a **Web** concern: it owns
the *SEO lens* on the view layer (`../rails-hotwire/`) and routing
(`../rails-controllers/`) without re-documenting how those render or route.

## When to Apply

Use this skill when the task is:

- Adding per-page `<title>` / meta description / canonical tags
- Making links unfurl with a title, description, and image (Open Graph, Twitter/X cards)
- Adding JSON-LD structured data (Article, Product, BreadcrumbList, Organization, …)
- Generating an XML sitemap and/or a `robots.txt`
- Giving records human-readable, stable, SEO-friendly URLs (slugs)
- Fixing crawlability/indexing issues (duplicate content, `noindex`, redirects)

Do **not** use this skill when the task is:

- **API-only / headless** with no HTML rendered by Rails → the JS/native frontend owns its SEO; this skill is largely N/A (it can still serve a sitemap/robots — see below)
- The view layer / `content_for` / Turbo rendering mechanics → read `../rails-hotwire/SKILL.md` (this skill is the SEO lens on it)
- Routing, `to_param`, redirects, route constraints → read `../rails-controllers/SKILL.md` (slugs here build on it)
- Per-request locale / translating tags → read `../rails-controllers/references/i18n.md`; this skill covers `hreflang` wiring
- **Page speed / Core Web Vitals / caching** (a real ranking factor) → read `../rails-performance/SKILL.md`
- Generating/serving the OG **image** asset itself (variants, signed URLs) → read `../rails-storage/SKILL.md`
- Canonical host, `force_ssl`, CDN, www↔apex redirect at the edge → read `../rails-deploy/SKILL.md`

## Detect Before You Generate

```bash
grep -nE "api_only" config/application.rb                                     # API-only? HTML SEO mostly N/A
ls app/views/layouts/application.html.erb 2>/dev/null                         # does Rails render HTML at all?
grep -E "^    (meta-tags|sitemap_generator|friendly_id)" Gemfile.lock         # SEO gems already installed?
ls public/robots.txt config/sitemap.rb 2>/dev/null                            # robots / sitemap_generator config present?
grep -rnE "display_meta_tags|set_meta_tags|content_for\(?:title|og:|ld\+json|extend FriendlyId" app/ config/ 2>/dev/null
grep -rnE "to_param" app/models 2>/dev/null                                   # native slug override already in place?
```

- If a meta-tag/sitemap/slug approach is **already installed**, extend it — don't
  re-ask or introduce a second mechanism alongside it.
- `config.api_only == true` and no `application.html.erb` → there is no HTML for a
  crawler to index; **stop and confirm scope.** Rails can still host a `sitemap.xml`
  and `robots.txt` for a separate frontend, but per-page meta/OG tags belong to
  whatever renders the pages.
- Honor any SEO picks recorded in `STACK.md`.

## Menu

Three genuine choices (meta-tag approach, sitemap approach, slug strategy). The rest
— structured data, `robots.txt`, canonical/`hreflang` — are practices, not pick-one
menus. Ask each via `AskUserQuestion`; the Rails-idiomatic, zero-dependency option is
**Recommended**; ask unless the project already constrains the pick.

### Meta-tag approach
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Native `content_for` + helper** *(Recommended)* | Zero-dependency, fully Rails-idiomatic: a `meta_tags` helper + `content_for :title`/`:description` in the layout. Total control, a little boilerplate. | [meta-tags.md](references/meta-tags.md) |
| `meta-tags` gem | DSL (`set_meta_tags`/`display_meta_tags`) that emits title/description/canonical/OG/Twitter from a hash, with site-wide defaults. Less boilerplate across many pages. | [meta-tags.md](references/meta-tags.md) |

### Sitemap approach
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Dynamic Rails-rendered `sitemap.xml`** *(Recommended)* | A route + controller rendering XML (`.xml.builder`), always fresh, no gem; cache it. Fine into the low tens of thousands of URLs. | [sitemaps-and-robots.md](references/sitemaps-and-robots.md) |
| `sitemap_generator` gem | Precompiles static sitemap files + a sitemap index, handles the 50k-URL/50MB split, pings search engines. Pick for large catalogs. | [sitemaps-and-robots.md](references/sitemaps-and-robots.md) |

### Slug strategy (SEO-friendly URLs)
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **`to_param` override** *(Recommended)* | Rails-native `"#{id}-#{title.parameterize}"`; keeps fast id lookups, zero-dependency. The slug is cosmetic; renames are safe. | [slugs-and-urls.md](references/slugs-and-urls.md) |
| `friendly_id` gem | Pure id-free string slugs with **slug history** (old URLs keep resolving after a rename), scoped/reserved slugs. Pick when URLs must be id-free and stable. | [slugs-and-urls.md](references/slugs-and-urls.md) |

## Decision Flow

- **Meta tags:** default to the **native helper** — no dependency, and the layout
  already owns `content_for`. Reach for the **`meta-tags` gem** when you have many
  page types and want a single hash + site-wide defaults instead of per-view
  `content_for` calls. Either way: a title and description on **every** indexable
  page, one canonical, OG + Twitter for sharing ([meta-tags.md](references/meta-tags.md)).
- **Sitemap:** **dynamic rendering** is the unopinionated default — always current,
  nothing to regenerate, and you cache it (`../rails-performance/`). Switch to
  **`sitemap_generator`** when the URL count pushes past a single 50k file or dynamic
  rendering gets slow, so you get a sitemap index and search-engine pinging
  ([sitemaps-and-robots.md](references/sitemaps-and-robots.md)).
- **Slugs:** **`to_param`** by default — simplest, keeps `find` fast, and a rename
  never 404s the old URL (the id still matches). Use **`friendly_id`** when the URL
  must contain no id and must survive renames (slug history → 301s)
  ([slugs-and-urls.md](references/slugs-and-urls.md)).
- **Structured data:** add JSON-LD for the content types you have (Article, Product,
  Organization, BreadcrumbList); it's plain `<script type="application/ld+json">`, no
  gem needed ([structured-data.md](references/structured-data.md)).
- **Crawl control:** ship an environment-aware `robots.txt` (block crawlers in
  staging), set `noindex` on thin/duplicate pages, and emit one self-referencing
  canonical per page ([sitemaps-and-robots.md](references/sitemaps-and-robots.md)).

## Problem → Reference

| Task | Read |
|---|---|
| Title / description / canonical, Open Graph + Twitter cards (native vs `meta-tags` gem) | [references/meta-tags.md](references/meta-tags.md) |
| XML sitemap (dynamic vs `sitemap_generator`) + environment-aware `robots.txt` + canonical | [references/sitemaps-and-robots.md](references/sitemaps-and-robots.md) |
| JSON-LD structured data (Schema.org: Article/Product/Organization/BreadcrumbList) | [references/structured-data.md](references/structured-data.md) |
| SEO-friendly URLs / slugs (`to_param` vs `friendly_id`), `hreflang`, redirects | [references/slugs-and-urls.md](references/slugs-and-urls.md) |

## Verify

SEO work is unfinished until the tags actually render and the routes are crawlable:

```bash
bin/rails server -d && sleep 3
BASE=localhost:3000
PAGE=/                                  # swap for a content page
# Title + description + canonical + social tags present:
curl -s $BASE$PAGE | grep -ioE "<title>[^<]+</title>|<meta name=.description.|<link rel=.canonical.|og:title|og:image|twitter:card" | sort -u
# Structured data parses as valid JSON:
curl -s $BASE$PAGE | ruby -e 'require "json"; html=STDIN.read; html.scan(/<script type="application\/ld\+json">(.+?)<\/script>/m){|m| JSON.parse(m[0]); puts "✓ valid JSON-LD"}'
# Sitemap + robots respond and are well-formed:
curl -s -o /dev/null -w "sitemap %{http_code}\n" $BASE/sitemap.xml
curl -s $BASE/sitemap.xml | head -2 | grep -q "<urlset\|<sitemapindex" && echo "✓ sitemap XML"
curl -s $BASE/robots.txt | grep -i "sitemap:" && echo "✓ robots points at sitemap"
kill "$(cat tmp/pids/server.pid)"
# Friendly URL resolves (if friendly_id chosen):
grep -q friendly_id Gemfile && bin/rails runner 'p Post.friendly.find(Post.first.slug)' 2>/dev/null
```

Validate the rendered structured data with Google's **Rich Results Test** and the
page with the **Mobile-Friendly** check (external tools), then route to
`../rails-performance/` for Core Web Vitals and `../rails-deploy/` for the canonical
host / `force_ssl` redirect. Record the meta-tag, sitemap, and slug picks in `STACK.md`.

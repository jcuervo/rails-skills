# SEO-friendly URLs — slugs, canonical, hreflang, redirects

Human-readable URLs (`/posts/my-first-post` instead of `/posts/417`) are a minor
ranking signal and a real usability/shareability win. This file owns the **SEO lens**
on URLs; the routing mechanics (`to_param`, route constraints, `redirect`) are
`../rails-controllers/`. Two ways to make slugs: a `to_param` override (Recommended)
or the `friendly_id` gem.

## Option A — `to_param` override *(Recommended)*

### Quick Pattern

```ruby
# app/models/post.rb
def to_param
  "#{id}-#{title.parameterize}"        # => "417-my-first-post"
end
```

`Post.find(params[:id])` still works unchanged — Ruby's `"417-my-first-post".to_i`
is `417`, so the leading id drives the lookup and the rest is cosmetic.

### When to pick / not pick

- **Pick** for the common case: fast id-based `find`, zero dependencies, and renames
  never 404 (the id still matches even if the title changed).
- **Don't pick** when the URL must contain **no id at all**, or must stay byte-stable
  and 301 cleanly across renames — that's history tracking, which `friendly_id` gives.

## Option B — `friendly_id` gem

### Quick Pattern

```ruby
# Gemfile
gem "friendly_id"
```

```bash
bin/rails generate --help | grep -i friendly   # confirm the generator name for the installed version
bin/rails generate friendly_id          # creates the friendly_id_slugs (history) table migration
bin/rails generate migration AddSlugToPosts slug:string:uniq
bin/rails db:migrate
```

The `friendly_id_slugs` history table is only used when the model opts in with
`use: :history` (below); confirm the generator's behavior against the `friendly_id`
version resolved in your `Gemfile.lock`.

```ruby
# app/models/post.rb
extend FriendlyId
friendly_id :title, use: %i[slugged history]   # history → old slugs still resolve
```

```ruby
# controller
@post = Post.friendly.find(params[:id])         # resolves current OR historical slug
```

### When to pick / not pick

- **Pick** for id-free URLs (`/posts/my-first-post`), **slug history** (old links 301
  to the new slug after a rename), and scoped/reserved slugs.
- **Don't pick** if a `to_param` override already satisfies you — it's a dependency
  plus a slugs table to maintain.

## Canonical, hreflang & redirects (practices)

- **Canonical** — one self-referencing `<link rel="canonical">` per page collapses
  duplicate URLs (id-slug vs slug, trailing slash, `?utm_*`) into one indexed page.
  See [meta-tags.md](meta-tags.md) for the helper.
- **hreflang** — for multi-locale sites, emit `<link rel="alternate" hreflang="…">`
  for each locale of the page plus an `x-default`, so Google serves the right language
  and doesn't treat them as duplicates. Locale selection itself is
  `../rails-controllers/references/i18n.md`:

  ```erb
  <% I18n.available_locales.each do |loc| %>
    <link rel="alternate" hreflang="<%= loc %>" href="<%= post_url(@post, locale: loc) %>">
  <% end %>
  <link rel="alternate" hreflang="x-default" href="<%= post_url(@post, locale: I18n.default_locale) %>">
  ```

- **301, not 302, for permanent moves.** When a slug or path changes for good, use a
  permanent redirect so link equity transfers; `friendly_id`'s history makes this
  automatic. The redirect mechanics live in `../rails-controllers/references/routing.md`.
- **Pick one host + scheme.** www↔apex and http→https should 301 to a single
  canonical host — that lives at the edge / `force_ssl` in `../rails-deploy/`.

## Deep Dive

- **`parameterize` transliterates** accented characters to ASCII; for non-Latin
  scripts pass a locale or accept Unicode slugs deliberately.
- **Uniqueness:** with the `to_param` approach the id guarantees uniqueness; with
  `friendly_id`, add a unique index on `slug` (and use `scoped` if slugs repeat across
  a parent).
- **Don't change URLs casually.** Every change costs ranking unless you 301. Stable
  URLs > pretty URLs.
- **Idempotent:** `to_param` is a one-method add. Adding `friendly_id` to existing
  records needs a backfill — `Post.find_each(&:save)` to populate slugs after the
  migration.

## Common Pitfalls

- **Switching to `friendly_id` without backfilling** existing rows → `nil` slugs →
  broken URLs. Backfill with `find_each(&:save)`.
- **Changing slugs with no redirect** → mass 404s and lost rankings. 301 (or use
  history).
- **Slug derived from a mutable field with no history** → old shared links die; that's
  the case for `friendly_id … use: :history`.
- **Missing unique index on `slug`** → race-condition duplicates.
- **hreflang without reciprocity / without `x-default`** → Google ignores the cluster.

## Verify

```bash
bin/rails server -d && sleep 3
# Swap `Post` / the `/posts/...` paths below for a slugged model in your app.
# Friendly URL resolves and is what links render:
grep -q friendly_id Gemfile \
  && bin/rails runner 'p Post.friendly.find(Post.first.slug)' \
  || bin/rails runner 'p Post.first.to_param'        # native: expect "<id>-<slug>"
# A renamed/old path 301s (friendly_id history), not 404s:
curl -s -o /dev/null -w "%{http_code}\n" localhost:3000/posts/an-old-slug   # expect 301 (history) or 200
# hreflang alternates present on a localized page:
curl -s localhost:3000/posts/1 | grep -oE "rel=\"alternate\" hreflang=\"[^\"]+\"" | sort -u
kill "$(cat tmp/pids/server.pid)"
```

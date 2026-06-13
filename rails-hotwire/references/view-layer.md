# View Layer — Partials, ViewComponent & Phlex

The menu deep dive for *how you structure rendered HTML*. Three approaches,
escalating in encapsulation and up-front ceremony. **Start with ERB partials**;
adopt a component framework when view sprawl is a real, felt problem.

## Menu recap

| Option | Deps | Best for |
|---|---|---|
| **ERB partials + helpers** *(Recommended)* | none | Default; most apps, most views |
| ViewComponent | one gem | Reused components, design systems, testable UI |
| Phlex | one gem | Views as pure Ruby, maximum composability |

## ERB partials + helpers (Recommended)

```erb
<!-- app/views/posts/_post.html.erb -->
<article id="<%= dom_id(post) %>">
  <h2><%= post.title %></h2>
  <%= render "posts/byline", author: post.author %>
</article>
```
```erb
<%= render partial: "posts/post", collection: @posts %>   <!-- renders _post per item -->
```

- **Pick when** … almost always to start. Zero deps, every Rails dev reads it,
  pairs directly with Turbo (`dom_id` + partials = stream targets).
- **Don't pick when** partials accumulate logic and `_helpers` sprawl — locals get
  passed five levels deep, conditionals pile up. That's the signal to extract
  components.
- Keep logic out of partials: use helpers or presenters; a partial with heavy `if`
  branches is a component waiting to happen.

## ViewComponent

```ruby
# app/components/post_component.rb
class PostComponent < ViewComponent::Base
  def initialize(post:) = @post = post
  def byline = "by #{@post.author.name}"
end
```
```erb
<!-- app/components/post_component.html.erb -->
<article id="<%= dom_id(@post) %>"><h2><%= @post.title %></h2><p><%= byline %></p></article>
```
```erb
<%= render(PostComponent.new(post: @post)) %>
```

- **Pick when** UI is reused across many views, you want **unit-testable** components
  (`render_inline(PostComponent.new(...))`), previews (`lookbook`), and a design
  system with clear component boundaries.
- **Don't pick when** the app is small or views aren't reused — the class+template
  pair is overhead for one-off markup.
- Components are Ruby objects: encapsulated state, slots, testable in isolation,
  no view-context leakage.

## Phlex

```ruby
# app/views/post_view.rb
class PostView < Phlex::HTML
  def initialize(post:) = @post = post
  def view_template
    article(id: @post.to_gid_param) do
      h2 { @post.title }
      p { "by #{@post.author.name}" }
    end
  end
end
render PostView.new(post: @post)
```

- **Pick when** the team wants **views as pure Ruby** — no template files,
  refactor/compose with Ruby methods, very fast rendering.
- **Don't pick when** designers edit templates or the team is deep in ERB muscle
  memory — it's the biggest mental shift of the three.

## Choosing & mixing

- Default partials; introduce **one** component framework when needed. You *can*
  mix (components for the design system, partials for page glue), but don't run
  ViewComponent *and* Phlex — pick one component story.
- Component frameworks compose cleanly with Turbo: render a component inside a
  `turbo_frame_tag` or from a `.turbo_stream.erb` just like a partial.

## Common Pitfalls

- **Logic-heavy partials** (`if/else` thickets, DB queries in the view) → extract a
  component or presenter; views should render, not decide.
- **Adopting components prematurely** → ceremony with no reuse payoff; wait for the
  sprawl signal.
- **Two component frameworks at once** → cognitive split; standardize.
- **Querying in the view** (N+1) → eager-load in the controller/model
  (`../rails-models/`, `../rails-performance/`).

## Verify

```bash
bundle list | grep -E "view_component|phlex" || echo "partials only (no component gem)"
bin/rails server -d && sleep 3
curl -fsS localhost:3000/posts -o /dev/null -w "index=%{http_code}\n"      # view renders
kill "$(cat tmp/pids/server.pid)"
# Component unit test (ViewComponent), if used:
bin/rails test test/components 2>/dev/null || bundle exec rspec spec/components 2>/dev/null || echo "(no component specs)"
```

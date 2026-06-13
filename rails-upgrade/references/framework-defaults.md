# Framework Defaults — `new_framework_defaults_*.rb` & `load_defaults`

Each Rails version ships new default behaviors. To keep an upgrade safe, you adopt
them **incrementally** rather than all at once. This is the single most important
risk-control in a Rails upgrade.

## How it works

```ruby
# config/application.rb
config.load_defaults 6.1          # ← still on the OLD version's defaults after bumping to 7.0
```

- After bumping the Rails version, **`config.load_defaults` stays at the old value**,
  so the app keeps the **old behavior** even though the new Rails is installed. This
  is intentional — it decouples "install the new version" from "adopt its new
  behaviors."
- `bin/rails app:update` generates
  **`config/initializers/new_framework_defaults_7_0.rb`** — a file of commented-out
  (or per-setting) new defaults you can **flip one at a time**.

```ruby
# config/initializers/new_framework_defaults_7_0.rb (generated; adopt incrementally)
Rails.application.config.action_controller.raise_on_open_redirects = true
# Rails.application.config.active_support.cache_format_version = 7.0
# ... each line is one new default; uncomment/enable one, test, repeat
```

## The safe sequence

```
1. Bump Rails version → load_defaults UNCHANGED (old behavior) → get the suite green.
2. Enable ONE setting in new_framework_defaults_7_0.rb → run tests → fix → commit.
3. Repeat until every new default is enabled and green.
4. Then: set config.load_defaults 7.0  AND delete new_framework_defaults_7_0.rb
   (load_defaults now provides them all).
5. Move to the next version.
```

- **Flip one default at a time** so when something breaks you know *exactly* which
  behavior change caused it. Flipping `load_defaults` straight to the new version
  enables them all at once — a debugging nightmare.
- **`load_defaults` and the defaults file are redundant once you flip** — when you set
  `config.load_defaults 7.0`, delete `new_framework_defaults_7_0.rb` (it would
  double-set / drift).

## Notable defaults that bite

- **Cache format / serializer version** — flipping can invalidate or misread existing
  cache entries; coordinate with `../rails-performance/` (clear/rotate the cache).
- **Cookie/CSRF serializer & encryption changes** — can invalidate existing sessions
  (users logged out) — schedule deliberately.
- **`raise_on_open_redirects`, stricter parameter handling** — surface latent bugs as
  errors; fix them.
- Each of these is *why* you flip incrementally — a surprise in production is far
  worse than a failing test you flip-and-fix locally.

## Common Pitfalls

- **Setting `config.load_defaults` to the new version immediately** → every new
  default at once; can't isolate failures. Use the incremental file.
- **Keeping both `load_defaults <new>` and `new_framework_defaults_<new>.rb`** →
  redundant/conflicting; delete the file once `load_defaults` is flipped.
- **Flipping cache/cookie format defaults without a plan** → mass cache misses or
  logged-out users in production; coordinate the rollout.
- **Never finishing the flip** → app stuck on old defaults indefinitely; the upgrade
  isn't "done" until `load_defaults` is at the current version and the file is gone.

## Verify

```bash
grep -n "load_defaults" config/application.rb                       # the active defaults level
ls config/initializers/new_framework_defaults_*.rb 2>/dev/null      # present = flip still in progress
bin/rails test 2>/dev/null || bundle exec rspec 2>/dev/null         # green after each flip
# When the step is truly done: load_defaults == current version AND no new_framework_defaults file:
test -z "$(ls config/initializers/new_framework_defaults_*.rb 2>/dev/null)" && echo "✓ defaults fully adopted for this step"
```

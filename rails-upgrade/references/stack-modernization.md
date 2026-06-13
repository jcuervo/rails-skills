# Stack Modernization — Sequencing the Omakase Swaps

Once the app is **on the current Rails version and green**, adopt the modern omakase
stack. **This skill owns the sequencing; each sibling owns the destination menu and
the how-to.** Do these **one at a time, after the version upgrade** — never tangle a
stack swap with a version bump.

## The golden rule

> **Version first, stack second.** Get to Rails 8.1 with the *old* stack still
> working (Sprockets, Webpacker, Sidekiq, secrets.yml all run fine on Rails 8). Then
> modernize each piece as its own change. Mixing them makes a failure impossible to
> attribute.

## The swaps (route to the owning sibling)

| Legacy → Modern | Owning sibling (destination + how-to) | Sequencing note |
|---|---|---|
| classic → **Zeitwerk** | [zeitwerk.md](zeitwerk.md) (here) | **Required** before Rails 7; do it first, on 6.x. |
| Sprockets → **Propshaft** | [`../rails-hotwire/`](../../rails-hotwire/references/assets-and-css.md), [`../rails-scaffold/`](../../rails-scaffold/references/frontend-stack.md) | After version is current; re-check asset references/digests. |
| Webpacker → **import maps / jsbundling / Vite** | [`../rails-hotwire/`](../../rails-hotwire/references/assets-and-css.md) | Retire Webpacker (unmaintained); pick the JS approach there. |
| Sidekiq+Redis → **Solid Queue** | [`../rails-jobs/`](../../rails-jobs/SKILL.md) | Jobs are Active Job — swap the adapter; drain queues first. |
| Redis cache → **Solid Cache** | [`../rails-performance/`](../../rails-performance/references/cache-store.md) | Swap `cache_store`; plan for a cold cache. |
| Redis Cable → **Solid Cable** | [`../rails-hotwire/`](../../rails-hotwire/references/real-time.md) | Swap the `config/cable.yml` adapter. |
| `secrets.yml` → **encrypted credentials** | [`../rails-security/`](../../rails-security/references/secrets-and-credentials.md) | Migrate secrets into `credentials.yml.enc`; rotate. |
| (none) → **Kamal / Thruster / CI** | [`../rails-deploy/`](../../rails-deploy/SKILL.md) | Adopt modern deploy once running on 8.1. |

## Why route instead of duplicate

The sibling skills already define the **menus, trade-offs, and Verify** for each
destination state (e.g. import maps vs esbuild vs Vite lives in `../rails-hotwire/`).
Re-documenting them here would duplicate and drift. This skill adds the one thing the
siblings don't: **when** to do each swap in an upgrade and **what order**, so a
brownfield app modernizes without breaking.

## Suggested order (after you're on 8.1)

1. **Zeitwerk** — already done (it was required to reach 7).
2. **Assets** — Sprockets → Propshaft, Webpacker → import maps/jsbundling
   (`../rails-hotwire/`). Asset breakage is visible and self-contained.
3. **Jobs** — Sidekiq → Solid Queue (`../rails-jobs/`): drain the old queue, switch
   the adapter, run the Solid Queue worker.
4. **Cache / Cable** — Redis → Solid Cache/Cable (`../rails-performance/`,
   `../rails-hotwire/`): expect a cold cache; cable adapter swap is low-risk.
5. **Secrets** — secrets.yml → encrypted credentials (`../rails-security/`): migrate
   values, rotate, remove the old file.
6. **Deploy** — adopt Kamal/Thruster/CI (`../rails-deploy/`).

Each step: change one thing, run the suite (`../rails-testing/`), deploy, then the
next.

## Common Pitfalls

- **Stack swap during the version bump** → can't tell whether the version or the swap
  broke it; separate them.
- **Dropping Redis before moving *all* of jobs+cache+cable off it** → something still
  needs Redis; migrate each, then remove Redis.
- **Solid backends without their databases** → Solid Queue/Cache/Cable use separate
  DBs ([`../rails-models/` multi-database.md](../../rails-models/references/multi-database.md));
  `db:prepare` must create them.
- **Removing `secrets.yml` before migrating its values** → missing config at boot;
  copy into credentials first.
- **Modernizing before the suite is green on 8.1** → you're changing two things on a
  shaky base; stabilize the version first.

## Verify

```bash
# Confirm the version is current BEFORE modernizing:
grep -E "^    rails \(" Gemfile.lock | head -1                 # 8.1.x
bin/rails test 2>/dev/null || bundle exec rspec 2>/dev/null    # green on 8.1 first
# After each swap, the legacy gem is gone and the modern one is in:
bundle list | grep -E "propshaft|solid_queue|solid_cache|solid_cable|importmap" 
bundle list | grep -E "webpacker|sprockets|sidekiq" && echo "⚠ legacy still present — finish the swap" || echo "✓ legacy removed"
```

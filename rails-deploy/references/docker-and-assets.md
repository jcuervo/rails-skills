# Production Docker Image, Thruster & Assets

Rails 8 generates a **production-ready multi-stage `Dockerfile`** that runs
**Thruster** in front of Puma. This reference is about understanding and tuning that
image — not replacing it. All orchestration targets ([kamal.md](kamal.md),
[orchestration-alternatives.md](orchestration-alternatives.md)) deploy this same
image.

## The generated Dockerfile (what it already does)

Rails 8's default `Dockerfile`:

- **Multi-stage build** — a build stage (gems, asset precompile, native deps) and a
  slim runtime stage, so the final image is small.
- **Precompiles assets** at build time (`bin/rails assets:precompile`) with a dummy
  `SECRET_KEY_BASE` so it doesn't need real secrets to build.
- **Runs Thruster** as the entrypoint proxy: `CMD ["./bin/thrust", "./bin/rails",
  "server"]` (confirm the exact line in your generated Dockerfile).
- Runs as a **non-root** user, exposes the Thruster port.

Don't hand-roll a Dockerfile unless you must — extend the generated one.

## Thruster (the proxy in the image)

Thruster sits in front of Puma inside the container and provides:

- **X-Sendfile acceleration** (efficient static/file serving),
- **asset compression + caching** (gzip/long-cache headers for precompiled assets),
- **HTTP/2**.

It means **no separate Nginx** is needed in front of the app for these. `bin/thrust`
wraps the Rails server. TLS termination is kamal-proxy's job
([kamal.md](kamal.md)), not Thruster's.

## Assets

- **Precompiled at image-build time** — `assets:precompile` runs in the Dockerfile,
  so the running container serves digested assets immediately (no precompile on
  boot). Propshaft + the chosen CSS/JS (`../rails-hotwire/` assets-and-css.md) feed
  this.
- **CSS/JS bundlers** (esbuild/Bun/Tailwind) must run their build during
  `assets:precompile` — the generated Dockerfile installs Node when the app needs it.
  Import-map apps need no Node.
- Serve assets with far-future cache headers (Thruster + Propshaft digests handle
  this); a CDN in front is optional (point it at the app/asset host).

## Tuning the image

- **`.dockerignore`** (generated) keeps `log/`, `tmp/`, `.git`, test files out of the
  build context — keep it tight for fast builds.
- **Layer caching:** `Gemfile`/`package.json` are copied and installed before app
  code so dependency layers cache across builds.
- **Native deps** (libvips for `../rails-storage/`, a DB client lib) must be installed
  in the image — a missing system lib is a top runtime failure.
- **Image size:** the multi-stage build already drops build tools; avoid adding heavy
  packages to the runtime stage.

## Common Pitfalls

- **Precompiling on boot instead of build** → slow/failed container starts; keep
  `assets:precompile` in the Dockerfile (it already is).
- **Missing native lib** (libvips/ImageMagick, DB client) → runtime errors despite a
  green build; install it in the image.
- **Bloated build context** → slow builds / huge images; keep `.dockerignore` tight.
- **Real secrets needed at build** → the Dockerfile uses a dummy `SECRET_KEY_BASE`
  for precompile; don't bake real secrets into a layer (`../rails-security/`).
- **Bundler installing dev/test gems in prod image** → larger image; the build sets
  `BUNDLE_WITHOUT` for prod.

## Verify

```bash
docker build -t app . && echo "✓ image builds"
# It boots and serves /up + a precompiled asset:
docker run --rm -e RAILS_MASTER_KEY=$(cat config/master.key) -p 3000:3000 app & sleep 8
curl -fsS -o /dev/null -w "up=%{http_code}\n" localhost:3000/up                      # 200
curl -fsS -o /dev/null -w "asset=%{http_code}\n" localhost:3000/assets 2>/dev/null    # assets dir served
docker stop $(docker ps -q --filter ancestor=app) 2>/dev/null
docker images app --format "{{.Size}}"                                               # sanity-check image size
```

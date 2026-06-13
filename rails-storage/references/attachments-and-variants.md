# Attachments & Variants

Attaching files to models and transforming images. The image-processor menu pick is
**libvips** (`:vips`, the Rails default — fast, low memory); MiniMagick is the
ImageMagick alternative.

## Quick Pattern — attach

```ruby
class User < ApplicationRecord
  has_one_attached  :avatar
  has_many_attached :documents
end
```
```ruby
user.avatar.attach(params[:avatar])              # from an uploaded file (strong params: ../rails-controllers/)
user.avatar.attached?                            # => true/false
user.avatar.attach(io: File.open(path), filename: "a.png", content_type: "image/png")
user.documents.attach(params[:documents])        # many
url_for(user.avatar)                             # a URL to serve it (redirects to the service)
```

`has_one_attached`/`has_many_attached` add the association — no column on the model;
blobs live in the Active Storage tables ([storage-services.md](storage-services.md)).

## Variants (image transforms)

```ruby
# Gemfile: gem "image_processing"   (required for variants; pulls ruby-vips or mini_magick)
user.avatar.variant(resize_to_limit: [200, 200])                  # ad-hoc
```
```ruby
# Named variants on the model (preferred — declared once, reused):
class User < ApplicationRecord
  has_one_attached :avatar do |attachable|
    attachable.variant :thumb,  resize_to_limit: [100, 100]
    attachable.variant :medium, resize_to_limit: [500, 500]
  end
end
user.avatar.variant(:thumb)
```

- Variants are generated **lazily** on first access and cached as their own blobs.
  For hot images, pre-process eagerly in a job (`../rails-jobs/`) so the first request
  isn't slow: `user.avatar.variant(:thumb).processed`.
- `resize_to_limit` (fit within, no upscale), `resize_to_fill` (crop to fill),
  `resize_and_pad` are the common transforms.

## Image processor → vips vs MiniMagick

```ruby
# config/application.rb (default is :vips on modern Rails)
config.active_storage.variant_processor = :vips        # or :mini_magick
```

- **libvips (`:vips`)** *(Recommended)* — the default: faster, far lower memory, and
  a smaller security surface than ImageMagick. Needs the **libvips** system library
  installed (`vips` on the host / in the Docker image).
- **MiniMagick** — wraps ImageMagick; broader format/feature coverage but heavier,
  slower, and historically more CVEs. Pick only when you need an ImageMagick-specific
  capability.
- Either way the **system library must be installed** (libvips or ImageMagick) — a
  missing native lib is the #1 "variants don't work" cause.

## Previews (non-image files)

```ruby
document.preview(resize_to_limit: [400, 400])    # PDF/video first-frame preview
```
Active Storage previews PDFs/videos via `poppler`/`mupdf`/`ffmpeg` (system deps).
Like variants, previews generate on demand and need the native tools present.

## Metadata, analysis & validation

- Active Storage **analyzes** blobs (dimensions for images, duration for video) in a
  background job automatically; `blob.metadata` holds the results.
- **Validate** content type and size before accepting — Active Storage doesn't by
  default. Use a gem (`active_storage_validations`) or validate in the model/
  controller; treat content-type as untrusted (`../rails-security/`).

## Common Pitfalls

- **`image_processing` gem missing** → `variant` raises; it's required for any
  transform.
- **System lib (libvips/ImageMagick) not installed** → variants fail at runtime even
  with the gem; install it on the host / in the image (`../rails-deploy/`).
- **Generating variants inline on a hot path** → slow first request; pre-process in a
  job.
- **No content-type/size validation** → users upload anything (huge files, scripts
  mislabeled as images); validate (`../rails-security/`).
- **`url_for` a not-yet-attached blob** → error; guard with `attached?`.

## Verify

```bash
bin/rails runner "p Rails.application.config.active_storage.variant_processor"   # :vips (or :mini_magick)
bundle list | grep -E "image_processing|ruby-vips|mini_magick" || echo "no image_processing — variants unavailable"
# Attach + variant round-trip (model needs has_one_attached :avatar):
bin/rails runner "u=User.first; u.avatar.attach(io: StringIO.new(File.read('test/fixtures/files/1x1.png')) rescue StringIO.new('x'), filename:'a.png', content_type:'image/png'); p u.reload.avatar.attached?" 2>/dev/null
bin/rails runner "u=User.first; p (u.avatar.variant(resize_to_limit:[10,10]).processed.key rescue 'install libvips/ImageMagick + image_processing')" 2>/dev/null
```

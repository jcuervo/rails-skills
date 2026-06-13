---
name: rails-storage
description: Handle file uploads and attachments in a Rails 8.1 app with Active Storage — attach files to models (has_one_attached / has_many_attached), generate image variants (via image_processing with libvips or MiniMagick), serve files, and upload directly from the browser to a storage service. Menu-driven service and image-processor selection with a Recommended default; detects the configured storage service and processor first, branches on config.api_only (direct-upload JS is a Web concern), and verifies an attachment round-trips and a variant generates. Apply when adding file/image uploads, configuring a storage backend (local/S3/GCS/Azure), creating image variants, or wiring direct uploads.
metadata:
  owner: rails-skills
  status: stable
user-invocable: true
argument-hint: "[model-or-attachment]"
---

# rails-storage

## Purpose

Owns **file uploads and attachments** via Active Storage: attaching files to models,
generating image variants, choosing and configuring the storage service
(local/S3/GCS/Azure), and direct browser-to-cloud uploads. Active Storage is the
framework layer; this skill picks the service and image processor beneath it and
wires attachments idempotently. Applies to API-only apps (attach via API, return
URLs) and full-stack apps (direct uploads, variants in views).

## When to Apply

Use this skill when the task is:

- Attaching files/images to a model (`has_one_attached` / `has_many_attached`)
- Configuring a storage service (local disk, S3, GCS, Azure)
- Generating image variants/thumbnails or previews
- Uploading directly from the browser to the storage service
- Serving/streaming attached files, signed URLs

Do **not** use this skill when the task is:

- The model the attachment hangs off (validations, associations) → read `../rails-models/SKILL.md`
- The upload **form** markup / Stimulus for the file field → read `../rails-hotwire/SKILL.md` (this skill wires the direct-upload mechanics; that one styles the form)
- Returning attachment URLs in a JSON API response shape → read `../rails-api/SKILL.md`
- Background processing of a large file (transcode, OCR) → read `../rails-jobs/SKILL.md` (Active Storage analysis/variants already run in jobs)
- Attaching an Active Storage file to an outgoing email → read `../rails-mailers/SKILL.md`
- Access-control/authorization of who can see a file → read `../rails-auth/SKILL.md`
- Securing public URLs, content-type validation as a security control → read `../rails-security/SKILL.md`
- CDN, production bucket/IAM, mirroring strategy in ops → read `../rails-deploy/SKILL.md`

## Detect Before You Generate

```bash
cat config/storage.yml 2>/dev/null                            # configured services (local/s3/gcs/azure)
grep -rnE "active_storage.service|variant_processor" config/environments/*.rb config/application.rb
grep -E "^    (aws-sdk-s3|google-cloud-storage|azure-storage-blob|image_processing|ruby-vips|mini_magick)" Gemfile.lock
grep -rn "has_one_attached\|has_many_attached" app/models 2>/dev/null
ls db/migrate/*active_storage* 2>/dev/null                    # active_storage tables installed?
```

- Active Storage tables come from `bin/rails active_storage:install` +
  `db:migrate`. If `active_storage_blobs`/`_attachments` exist, it's set up.
- `config.active_storage.service` per environment names the live service (local in
  dev, a cloud in prod is typical) — don't re-ask if configured.
- The **variant processor** default is **libvips** (`:vips`); detect
  `config.active_storage.variant_processor` before assuming.
- Honor any storage service recorded in `STACK.md`.

## Menu

Two menus. Each via `AskUserQuestion`, Rails-default marked **Recommended**; ask
unless configured.

### Storage service → `config/storage.yml` + `config.active_storage.service`
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **Local disk** *(Recommended for dev)* | Rails default; files on the server's disk. Zero setup; **not** for multi-host/ephemeral prod. | [storage-services.md](references/storage-services.md) |
| Amazon S3 *(production default)* | The ubiquitous production choice; durable, CDN-friendly. Needs `aws-sdk-s3` + IAM. | [storage-services.md](references/storage-services.md) |
| Google Cloud Storage | GCS equivalent; pick on GCP. Needs `google-cloud-storage`. | [storage-services.md](references/storage-services.md) |
| Azure Blob Storage | Azure equivalent; pick on Azure. Needs `azure-storage-blob`. | [storage-services.md](references/storage-services.md) |

### Image processor → `config.active_storage.variant_processor`
| Option | One-line trade-off | Deep dive |
|---|---|---|
| **libvips (`:vips`)** *(Recommended)* | Rails default; fast, low memory, fewer CVEs. Needs the libvips system lib. | [attachments-and-variants.md](references/attachments-and-variants.md) |
| MiniMagick (ImageMagick) | Ubiquitous, more format coverage; heavier, slower, larger attack surface. | [attachments-and-variants.md](references/attachments-and-variants.md) |

## Decision Flow

- **Service:** **local disk** in development (default, zero setup). For **production**
  pick a cloud (**S3** unless you're standardized on GCP→GCS or Azure→Azure Blob) —
  local disk doesn't survive multi-host or ephemeral/containerized deploys, and the
  files must persist across releases. The dev/prod split lives in `config/storage.yml`
  + per-environment `service`.
- **Image processor:** **libvips** (default — fast, low memory) unless you need an
  ImageMagick-only format/feature, then MiniMagick. Either needs `image_processing`
  + the system library.
- **Variants:** define **named variants** on the model and let them generate lazily
  (on first request) or eagerly via a job (`../rails-jobs/`) for hot images.
- **Direct uploads** (browser → service, bypassing the app) for large files / to
  offload the server — Web concern, needs the Active Storage JS
  ([direct-uploads.md](references/direct-uploads.md)).
- **Access control:** public vs signed/expiring URLs is an auth/security decision —
  `../rails-auth/`, `../rails-security/`.

## Problem → Reference

| Task | Read |
|---|---|
| Configure the storage service (local/S3/GCS/Azure), credentials, mirroring | [references/storage-services.md](references/storage-services.md) |
| Attach files (has_one/has_many_attached), variants, processors, previews | [references/attachments-and-variants.md](references/attachments-and-variants.md) |
| Direct browser-to-service uploads, signed URLs | [references/direct-uploads.md](references/direct-uploads.md) |

## Verify

A storage change is unfinished until a file attaches, persists, and (for images) a
variant generates:

```bash
bin/rails active_storage:install >/dev/null 2>&1; bin/rails db:prepare    # tables exist
bin/rails runner "p ActiveStorage::Blob.service.class.name; p Rails.application.config.active_storage.variant_processor"  # service + processor
# Round-trip an attachment (model must have e.g. has_one_attached :avatar):
bin/rails runner "u=User.first; u.avatar.attach(io: StringIO.new('x'), filename: 't.txt', content_type: 'text/plain'); p u.reload.avatar.attached?" 2>/dev/null
# A variant processes (image + image_processing installed):
bin/rails runner "u=User.first; p (u.avatar.variant(resize_to_limit: [50,50]).processed.key rescue 'needs image + processor')" 2>/dev/null
```

Then route to `../rails-models/` (the host model), `../rails-hotwire/` (the upload
form), `../rails-jobs/` (eager variant processing), and record service + processor in
`STACK.md`.

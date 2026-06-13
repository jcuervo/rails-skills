# Storage Services

Where the bytes live. Local disk for development (Rails default); a cloud service for
production. Configured in `config/storage.yml` (named services) and selected per
environment with `config.active_storage.service`. Credentials live in encrypted
credentials (`../rails-security/`).

## Quick Pattern — install + select

```bash
bin/rails active_storage:install && bin/rails db:migrate    # creates the Active Storage tables
# (blobs + attachments; plus variant_records when track_variants is enabled — verify the
#  exact migration for your Rails version)
```
```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

amazon:
  service: S3
  access_key_id:     <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: us-east-1
  bucket: my-app-<%= Rails.env %>
```
```ruby
# config/environments/development.rb → config.active_storage.service = :local
# config/environments/production.rb  → config.active_storage.service = :amazon
```

## The services

| Service | `service:` | Gem | Pick when |
|---|---|---|---|
| **Local disk** *(dev default)* | `Disk` | — | Development; single-host toy/prototype |
| Amazon S3 | `S3` | `aws-sdk-s3` | Production default; durable, CDN-friendly |
| Google Cloud Storage | `GCS` | `google-cloud-storage` | You're on GCP |
| Azure Blob | `AzureStorage` | `azure-storage-blob` | You're on Azure |

> Service-class keys and gem names drift between Rails versions (Azure's adapter/gem
> support especially) — confirm the `service:` value and gem for each cloud against
> the Rails 8.1 Active Storage guide before wiring it.

- **Local disk is not production-safe** for multi-host or ephemeral/containerized
  deploys: each host has its own disk, and containers lose files on redeploy. Use a
  cloud service (and a persistent volume only for genuinely single-host setups —
  `../rails-deploy/`).
- Each cloud service needs its gem (uncomment in the Gemfile) and credentials.

## Mirroring & migration

```yaml
# Mirror writes to multiple services (e.g. while migrating local → S3):
production:
  service: Mirror
  primary: amazon
  mirrors: [amazon_backup]
```

- **Mirror** service writes to a primary + mirrors — useful for zero-downtime
  migration between providers or cross-region redundancy.
- Migrating existing files between services means re-uploading blobs; do it with a
  background task (`../rails-jobs/`), not inline.

## When to pick / not pick

- **Local disk** — dev always; prod only on a single persistent host.
- **S3** — the safe production default unless your cloud dictates otherwise.
- **GCS / Azure** — match your platform; functionally equivalent to S3 for Active
  Storage's purposes.

## Common Pitfalls

- **Local disk in containerized production** → uploads vanish on each deploy; use a
  cloud service.
- **Credentials in `storage.yml` as literals** → leaked keys; use
  `Rails.application.credentials`.
- **Bucket not created / wrong IAM** → uploads fail at runtime; provision the bucket
  and least-privilege IAM (`../rails-deploy/`, `../rails-security/`).
- **Same bucket across environments** → staging clobbers prod files; suffix the
  bucket with `Rails.env`.
- **Public bucket by default** → anyone can read uploads; decide public vs private +
  signed URLs deliberately ([direct-uploads.md](direct-uploads.md), `../rails-security/`).

## Verify

```bash
bin/rails runner "p ActiveStorage::Blob.service.class.name"       # Disk/S3/GCS/AzureStorage
bin/rails runner "p Rails.application.config.active_storage.service"
# A blob round-trips to the configured service:
bin/rails runner "b=ActiveStorage::Blob.create_and_upload!(io: StringIO.new('hi'), filename: 't.txt'); p b.download" 2>/dev/null   # => "hi"
bundle list | grep -E "aws-sdk-s3|google-cloud-storage|azure-storage-blob" || echo "local disk (no cloud gem)"
```

# Direct Uploads (browser → service)

Upload large files **straight from the browser to the storage service**, bypassing
the Rails server, using Active Storage's JavaScript. This is a **Web** concern (needs
the Active Storage JS); API clients implement the same handshake themselves. The
upload *form markup/styling* is [`../rails-hotwire/`](../../rails-hotwire/SKILL.md) —
this file owns the direct-upload mechanics.

## How it works (the handshake)

1. Browser asks Rails to create a blob record → Rails returns a **signed, short-lived
   upload URL** to the service.
2. Browser **PUTs the file directly** to S3/GCS/Azure/local (not through your app).
3. On form submit, only the blob's **signed id** is sent to Rails, which attaches it.

The server never streams the file bytes — it scales and frees up request workers.

## Quick Pattern

```js
// app/javascript — pin/import the Active Storage JS, then enable it:
import * as ActiveStorage from "@rails/activestorage"
ActiveStorage.start()
```
```erb
<%= form_with model: @user do |f| %>
  <%= f.file_field :avatar, direct_upload: true %>   <!-- the only change vs a normal upload -->
  <%= f.submit %>
<% end %>
```

`direct_upload: true` + `ActiveStorage.start()` is the whole switch. The JS handles
the signed-URL request, the PUT, progress events (`direct-upload:progress`), and
substituting the signed id into the form.

- **Import maps:** `pin "@rails/activestorage"` and import it
  ([`../rails-hotwire/`](../../rails-hotwire/references/assets-and-css.md)).
- **CORS on the bucket:** the browser PUTs cross-origin to S3/GCS/Azure, so the
  **bucket's CORS policy** must allow your origin + the `PUT` method. This is bucket
  config, not Rails CORS (`../rails-api/` cors.md is for the API, not the bucket).

## API clients (no Active Storage JS)

An API/mobile client replicates the handshake manually:

```
POST /rails/active_storage/direct_uploads   → returns { direct_upload: { url, headers }, signed_id }
PUT  <url> (file bytes, with returned headers)
# then send signed_id as the attachment param to your endpoint
```

The endpoint that accepts the final `signed_id` is a normal controller action
(`../rails-controllers/`); attach with `record.avatar.attach(params[:signed_id])`.

## Serving files & access control

- `url_for(blob)` / `rails_blob_path` issue a **redirect** to the service (or stream
  from disk for local). For private buckets, Active Storage signs a short-lived URL.
- **Public vs private** is a deliberate decision: a public bucket means anyone with
  the URL can read; private + signed expiring URLs gate access. Who is *allowed* to
  request a file is `../rails-auth/`; URL/security hardening is `../rails-security/`.

## Common Pitfalls

- **Bucket CORS not configured** → direct PUT fails with a CORS error in the browser;
  add the origin + `PUT`/`POST` to the bucket policy.
- **`ActiveStorage.start()` not called / JS not imported** → `direct_upload: true`
  silently falls back to a normal (through-server) upload.
- **Confusing app CORS with bucket CORS** → `rack-cors` (`../rails-api/`) governs your
  API; the **service bucket** has its own CORS policy for direct PUTs.
- **Orphaned blobs** → a created blob whose form is abandoned isn't attached; purge
  unattached blobs periodically (`ActiveStorage::Blob.unattached`, via a job).
- **Trusting client-supplied content-type** → validate server-side
  (`../rails-security/`).

## Verify

```bash
bin/rails server -d && sleep 3
# The direct-upload endpoint exists and mints a signed URL:
curl -s -o /dev/null -w "direct_uploads=%{http_code}\n" -X POST localhost:3000/rails/active_storage/direct_uploads \
  -H "Content-Type: application/json" \
  -d '{"blob":{"filename":"a.png","byte_size":3,"checksum":"x","content_type":"image/png"}}'   # 200/201 (or 422 on checksum)
# The form field renders the direct-upload data attribute:
curl -s localhost:3000/users/new -o /tmp/f.html 2>/dev/null && grep -q "direct-upload\|direct_upload" /tmp/f.html && echo "✓ direct_upload field present"
kill "$(cat tmp/pids/server.pid)"
bundle list | grep -E "@rails/activestorage" 2>/dev/null || echo "ensure @rails/activestorage is pinned/imported (../rails-hotwire/)"
```

# Personal Data: Protection, Consent, Access & Erasure

The data-layer primitives behind GDPR/CCPA-style obligations — **encrypt and
minimize** what you store, **record consent**, **export** a subject's data on
request (access/portability), and **erase or anonymize** it on request (right to be
forgotten).

> **Scope guard.** This reference covers the *technical mechanics*. Whether wiring
> them up satisfies a given regulation (GDPR, CCPA, HIPAA…) is a **legal** question
> outside this skill — lawful basis, retention periods, and which fields count as
> "personal data" are determined by counsel, not by code. We build the levers; the
> policy is yours. For a retrospective review use `rails-audit`; for protecting the
> data in transit/at the edge see `../rails-security/` and `../rails-deploy/`.

Four primitives, each Rails-native (no gem required). The async halves run in
`../rails-jobs/`; export artifacts land in `../rails-storage/`; consent records reuse
the [audit-trail.md](audit-trail.md) primitive.

## Detect first

```bash
bin/rails credentials:show 2>/dev/null | grep -q active_record_encryption && echo "AR Encryption keys present"  # configured? (or grep config for an explicit encryption.* = ENV[...] wiring — Rails does not auto-read ACTIVE_RECORD_ENCRYPTION_*)
grep -rnE "encrypts |normalizes " app/models 2>/dev/null                               # existing encrypted/normalized fields
grep -rnE "dependent:" app/models 2>/dev/null | grep -iE "destroy|nullify|delete"      # cascade behavior already declared
ls app/models/consent*.rb app/models/*consent*.rb 2>/dev/null                          # existing consent record
```

Reuse what's there — extend an existing erase path or consent model rather than
adding a parallel one. Record any new conventions in `STACK.md`.

A `STACK.md` `Compliance` row listing `gdpr`, `ccpa`, or `hipaa` is the opt-in
signal to **proactively offer** these primitives (encryption at rest, consent,
export, erasure) as you model personal data — instead of waiting to be asked.
`none`/absent → apply them only when the task explicitly requests one. Offering is
still asking: nothing here auto-wires, and none of it asserts the app *is* compliant.

---

## 1. Minimize & encrypt at rest (Art. 5/32)

### Quick Pattern
Active Record Encryption — generate keys once, then declare encrypted columns.
```bash
bin/rails db:encryption:init        # prints primary_key, deterministic_key, key_derivation_salt
bin/rails credentials:edit          # paste them under active_record_encryption:
```
Credentials is the default store. To source the keys from ENV instead, **wire it
explicitly** — Rails does **not** auto-read the `ACTIVE_RECORD_ENCRYPTION_*` names; set
them in config (verified on Rails 8.1):
```ruby
# config/application.rb (or an initializer)
config.active_record.encryption.primary_key         = ENV["ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY"]
config.active_record.encryption.deterministic_key   = ENV["ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY"]
config.active_record.encryption.key_derivation_salt = ENV["ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT"]
```
See `../rails-security/` for secrets handling.
```ruby
class User < ApplicationRecord
  encrypts :ssn                          # non-deterministic: most secure, NOT queryable
  encrypts :email, deterministic: true   # same input → same ciphertext, so find_by works
  normalizes :email, with: ->(e) { e.strip.downcase }   # normalize BEFORE encryption
end
```

### Deep Dive
- **Deterministic vs not.** Non-deterministic (default) uses a random IV — strongest,
  but you can't `where`/`find_by` on it or index it usefully. Deterministic derives
  the IV from the plaintext so equal values yield equal ciphertext — required to look
  a record up by that column (e.g. login by email), at the cost of leaking equality.
  Encrypt the **minimum** deterministically; keep the rest non-deterministic.
- **Minimization** is a modeling decision, not a library: don't add the column if you
  don't need the data. Pseudonymize where you can — store a surrogate id and keep the
  PII↔id mapping in one narrow, encrypted place.
- **Crypto-shredding** (advanced erasure lever): with a per-subject encryption key,
  destroying that key renders the subject's encrypted data permanently unreadable —
  an erasure strategy when hard `DELETE` is impractical (backups, append-only stores).
  Native AR Encryption uses app-wide keys, so this needs a custom key-per-record
  scheme; note it as an option, don't assume it.

### Common Pitfalls
- **Deterministic-encrypting a high-cardinality unique field then expecting an index
  to enforce uniqueness** — it works, but the ciphertext is what's indexed; changing
  the key/salt breaks lookups. Plan for Rails' encryption key-rotation config (verify
  the current key name against the Active Record Encryption guide) before you need it.
- **Normalization must reach the plaintext.** `normalizes` runs on assignment, so the
  normalized value is what gets encrypted *and* what `find_by(email:)` re-normalizes
  to match — the Verify below proves the lookup succeeds on your Rails version. If a
  lookup misses, confirm normalization isn't being bypassed (`update_column`, raw SQL).

## 2. Record consent (Art. 6/7)

### Quick Pattern
Consent is an **append-only event log**, never a mutable boolean — you must prove
*what* was agreed, *when*, and *which version*.
```ruby
# db/migrate/xxxx_create_consents.rb
create_table :consents do |t|
  t.references :user, null: false
  t.string   :purpose,       null: false   # "marketing_email", "analytics", ...
  t.string   :terms_version, null: false   # which policy text they saw
  t.boolean  :granted,       null: false    # true = grant, false = withdrawal
  t.timestamps
end
```
```ruby
class Consent < ApplicationRecord
  belongs_to :user
  scope :latest_for, ->(purpose) { where(purpose:).order(created_at: :desc) }
end

# grant and withdrawal are both new rows — history is preserved
user.consents.create!(purpose: "marketing_email", terms_version: "2026-05", granted: true)
def consented?(user, purpose) = user.consents.latest_for(purpose).first&.granted
```

### Deep Dive / Pitfalls
- **Don't `update` a consent row** — that destroys the audit value. Withdrawal is a
  new `granted: false` row. The current state is "the latest row per purpose."
- If you already run the [audit-trail.md](audit-trail.md) primitive over `User`, a
  consent toggle is captured there too — but a purpose-built `consents` table is the
  queryable source of truth; the audit log is the forensic backup.
- Capturing consent in a request? The controller wiring (params, CSRF) is
  `../rails-controllers/`; cookie/tracking consent UI is frontend (`../rails-hotwire/`).

## 3. Access & portability — export a subject's data (Art. 15/20)

### Quick Pattern
Assemble the subject's personal data across associations into a portable file, in a
**job** (it can be large and slow), then hand back a download.
```ruby
# app/models/user.rb — alongside `has_one_attached :data_export` (Active Storage)
def personal_data_export
  {
    profile:  attributes.slice("id", "email", "created_at"),
    orders:   orders.map { |o| o.attributes.slice("id", "total", "created_at") },
    consents: consents.map { |c| c.attributes.slice("purpose", "granted", "created_at") }
  }
end
```
```ruby
# app/jobs/data_export_job.rb — runs in ../rails-jobs/
class DataExportJob < ApplicationJob
  def perform(user)
    json = JSON.pretty_generate(user.personal_data_export)
    user.data_export.attach(                          # needs `has_one_attached :data_export` — ../rails-storage/
      io: StringIO.new(json), filename: "export-#{user.id}.json", content_type: "application/json")
    # then notify the user with a time-limited, authorized download link
  end
end
```

### Deep Dive / Pitfalls
- **Portable format**: JSON or CSV the subject can actually reuse — not a DB dump.
- **Authorize the download** like any sensitive file: short-lived signed URL, scoped
  to that user — see `../rails-storage/` for signed URLs and `../rails-auth/` for the
  authorization check. An export link is a data-leak vector if it's guessable.
- **Don't leak others' data**: export only the requesting subject's rows. Watch
  shared/joined records (a shared order, another user's message) — slice fields
  deliberately, allow-list rather than dump `attributes`.

## 4. Erasure — delete or anonymize (Art. 17)

### Quick Pattern
Decide per association: **hard-delete** vs **anonymize/pseudonymize** (keep the row
for integrity/analytics/legal-retention, scrub the PII). Run it in a job.
```ruby
# app/models/user.rb
def erase_personal_data!
  transaction do
    messages.delete_all                                   # hard delete: no value retained
    orders.update_all(customer_name: nil, email: nil)     # anonymize: keep order for accounting
    update!(email: "erased-#{id}@example.invalid", name: "[erased]",
            ssn: nil, erased_at: Time.current)
  end
end
```
```ruby
# app/jobs/data_erasure_job.rb — ../rails-jobs/
class DataErasureJob < ApplicationJob
  def perform(user) = user.erase_personal_data!
end
```

### Decision: delete vs anonymize
| Strategy | Use when | Cost |
|---|---|---|
| **Hard delete** (`destroy`/`delete_all`) | The record has no value without the person (messages, drafts) | `dependent:` must cascade correctly or you orphan rows / FK-error |
| **Anonymize / pseudonymize** | The row must survive for integrity, accounting, or aggregate analytics (orders, invoices) | Must scrub *every* PII column + free-text that embeds it |
| **Crypto-shred** | Hard delete is impractical (immutable/append-only/backup) and you encrypt per-subject | Requires a key-per-subject scheme (see §1) |

### Deep Dive / Pitfalls
- **`dependent:` correctness is the whole game.** An erase that misses a child
  association leaves PII behind (a silent failure) or raises an FK error. Audit the
  association graph before trusting a cascade — `dependent: :destroy` runs callbacks
  (slow, safe), `:delete_all` skips them (fast, no cascade past one level). See
  [associations.md](associations.md).
- **Anonymize means *all* of it** — including free-text columns (a `notes` field with
  a name in it), Active Storage blobs (`../rails-storage/`), search indexes
  (`search.md`), cached copies, and **the audit log itself** if it captured the PII.
  Anonymization that leaves a copy somewhere isn't erasure.
- **Do it async and idempotent.** Erasure can span many tables; a half-run that can't
  safely re-run is worse than slow. Make `erase_personal_data!` re-runnable (the
  `update!` above is idempotent; guard `delete_all` with existence). Background-job
  patterns/retries live in `../rails-jobs/`; the **scheduled** variant — purging on a
  retention window rather than on request — is
  [`../../rails-jobs/references/data-retention.md`](../../rails-jobs/references/data-retention.md).
- **Foreign keys with `ON DELETE`**: a DB-level `dependent` (FK `on_delete: :cascade`)
  and an AR `dependent:` can both fire or fight — pick one path per association and
  know which (see [migrations.md](migrations.md)).

## Verify

```bash
# 1. encryption at rest — ciphertext in the DB, plaintext via the model
bin/rails runner 'u=User.create!(email:"a@b.com", ssn:"123"); p User.connection.select_value("select ssn from users where id=#{u.id}") != "123"'  # => true (stored encrypted; id is a trusted integer — never interpolate user input into SQL)
bin/rails runner 'p User.find_by(email:"a@b.com").present?'                          # deterministic field is queryable

# 2. consent — withdrawal is a new row, history intact
bin/rails runner 'u=User.first; u.consents.create!(purpose:"mkt",terms_version:"1",granted:true); u.consents.create!(purpose:"mkt",terms_version:"1",granted:false); p u.consents.latest_for("mkt").first.granted'  # => false, 2 rows

# 3. export — produces a portable file scoped to the subject
bin/rails runner 'p JSON.parse(JSON.generate(User.first.personal_data_export)).keys'  # => ["profile","orders","consents"]

# 4. erasure — PII gone, structural rows decided per strategy, re-runnable
bin/rails runner 'u=User.create!(email:"e@x.com",name:"Real"); u.erase_personal_data!; u.reload; p [u.name, u.email =~ /erased/]'  # => ["[erased]", 0]
bin/rails runner 'u=User.create!(email:"e2@x.com",name:"Real"); 2.times { u.erase_personal_data! }; p "idempotent re-run ok"'  # second run must not raise
```

Each step proves the *effect* (ciphertext stored, history preserved, file scoped, PII
actually gone and re-runnable) — not merely that a method exists. Then record the
erase/anonymize strategy per model in `STACK.md` so the next change keeps the cascade
correct.

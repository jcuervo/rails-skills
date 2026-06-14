# Audit Trail / Change Logging

A **who-changed-what-and-when** record over your models — the technical primitive
behind "audit trail" requirements (SOC 2, HIPAA, internal forensics) and behind
undo/version features. This is the *recording* mechanism only; whether a given
regulation is satisfied is a legal question this skill does not answer. For a
retrospective security review use the upstream `rails-audit` skill; for protecting
the audit records themselves (tamper-resistance, log redaction) cross-link
`../rails-security/`.

> **No Rails-default omakase pick here** — Rails ships no built-in model-audit
> feature, so this menu's **Recommended** is the most common purpose-built choice,
> not a framework default. Detect what the app already uses before adding anything.

## Menu

| Option | One-line trade-off |
|---|---|
| **`audited`** *(Recommended)* | Purpose-built audit log: one polymorphic `audits` table, automatic actor from `current_user`, stores the diff per change. Lowest friction for "who changed what." |
| Rails-native (custom `Audit` model) | Zero dependencies — an `after_commit` hook records `saved_changes` into your own table with `Current.user`. Pick when needs are simple and you want no gem. |
| `paper_trail` | Versioning/undo focus: full-object snapshots in a `versions` table, `reify` to restore a past state. Pick when you need to *roll back*, not just log. |
| `logidze` | Postgres-only, DB-trigger + JSONB log column *on the row itself* — fastest, no extra table. Pick for high write volume on Postgres; costs you trigger migrations. |

## Detect first

```bash
grep -E "^    (audited|paper_trail|logidze) " Gemfile.lock          # already chosen?
ls db/migrate/*audit* db/migrate/*version* 2>/dev/null              # existing audit/version tables
grep -rnE "audited|has_paper_trail|has_logidze" app/models 2>/dev/null
```

If one is already installed, **extend it** — add the macro to the new model, don't
introduce a second mechanism. Record the pick in `STACK.md`.

A `STACK.md` `Compliance` row listing `audit-trail`, `soc2`, or `hipaa` is the
opt-in signal to **proactively offer** this menu when modeling a relevant record
(rather than waiting to be asked). `none`/absent → add an audit trail only when the
task explicitly calls for one. Offering is still asking: the menu below runs either
way, nothing auto-wires, and recording a trail never asserts the app *is* compliant.

---

## `audited` (Recommended)

### Quick Pattern
```ruby
# Gemfile
gem "audited"
```
```bash
bundle install
bin/rails generate audited:install   # creates the audits table migration
bin/rails db:migrate
```
```ruby
class Patient < ApplicationRecord
  audited                                  # log create/update/destroy
  # audited only: [:status], except: [:ssn]   # scope which columns are logged
end
```
The actor is picked up automatically: Audited reads `current_user` in controllers.
Outside a request (jobs, console, rake) set it explicitly:
```ruby
Audited.audit_class.as_user(user) { patient.update!(status: "discharged") }
patient.audits.last.audited_changes   # => {"status"=>["admitted","discharged"]}
```

### When to pick / not pick
- **Pick** when the requirement is literally an audit log — append-only history of
  who changed which fields. It is the most direct fit.
- **Don't pick** when you need to *restore* an old version (paper_trail reifies; an
  audit diff doesn't reconstruct the whole object cleanly), or when audit-write
  overhead on a hot path matters more than convenience (logidze is faster).

### Deep Dive
Audited stores one row per change in a single polymorphic `audits` table
(`auditable_type`/`auditable_id`, `user_type`/`user_id`, `action`,
`audited_changes`, `version`, `created_at`). It hooks the create/update/destroy
lifecycle to write those rows (see the gem source for the exact callbacks in your
installed version). Associated audits (`associated_with:`) let a child's changes
roll up to a parent for a combined timeline.

> **Verify Rails 8.1 compatibility at install time.** The `audited` README's stated
> support matrix has trailed the newest Rails release in the past. Pin a current
> version and confirm the suite boots — don't assume:
> ```bash
> bundle add audited && bin/rails runner "puts Audited::VERSION"
> grep -E "^    rails \(" Gemfile.lock | head -1     # cross-check your Rails line
> ```
> If `bundle` resolves a Rails conflict, check the gem's latest release notes for an
> 8.1-compatible version before forcing it.

## Rails-native custom `Audit`

### Quick Pattern
No gem. A plain model + a concern that records committed changes.
```ruby
# db/migrate/xxxx_create_audits.rb — JSONB on Postgres, json on SQLite/MySQL
create_table :audits do |t|
  t.references :auditable, polymorphic: true, null: false
  t.references :user, null: true
  t.string  :action, null: false
  t.jsonb   :audited_changes, null: false, default: {}   # NOT `changes` — that shadows AR's dirty-tracking method
  t.timestamps
end
```
```ruby
# app/models/concerns/auditable.rb
module Auditable
  extend ActiveSupport::Concern
  included do
    after_create_commit  { record_audit("create") }
    after_update_commit  { record_audit("update") if saved_changes.except("updated_at").any? }
    after_destroy_commit { record_audit("destroy") }
  end
  private
  def record_audit(action)
    Audit.create!(auditable: self, user: Current.user, action:,
                  audited_changes: saved_changes.except("updated_at"))
  end
end
```
`Current.user` is a Rails `ActiveSupport::CurrentAttributes` value — set it in a
controller `before_action` and in jobs (see `../rails-controllers/` and
`../rails-jobs/`).

### When to pick / not pick
- **Pick** for simple, app-specific needs where adding a dependency isn't worth it,
  or when you want full control of the schema and what "a change" means.
- **Don't pick** when you'll reinvent associated-audit rollups, reification, or
  actor plumbing the gems already solved — at that point use one.

### Common Pitfalls (native)
- Recording in `after_save` instead of `after_*_commit` → audits written for changes
  that later **roll back**. Use the commit callbacks.
- Forgetting to set `Current.user` outside requests → null actor in jobs/console.
  Wrap with an explicit assignment.

## `paper_trail`

### Quick Pattern
```ruby
gem "paper_trail"
```
```bash
bundle add paper_trail && bin/rails runner 'puts PaperTrail::VERSION::STRING'  # confirm it resolves on your Rails
bin/rails generate paper_trail:install && bin/rails db:migrate
```
Recent paper_trail releases support ActiveRecord Encryption; confirm the resolved
version against its CHANGELOG rather than pinning a number that drifts.
```ruby
class Article < ApplicationRecord
  has_paper_trail
end

article.versions                 # ordered history
article.paper_trail.previous_version
prior = article.versions.last.reify   # reconstruct the object as it was
```
Actor: set `PaperTrail.request.whodunnit = current_user.id` (auto via the
controller mixin) and `whodunnit` is stored on each version.

> **Serialization gotcha on modern Ruby (verified Ruby 4.0 / Rails 8.1).** With the
> default YAML `object` column, `reify` raises
> `Psych::DisallowedClass: ActiveSupport::TimeWithZone` because Psych safe-loads. Fix
> one of two ways: permit the classes —
> `config.active_record.yaml_column_permitted_classes |= [ActiveSupport::TimeWithZone, ActiveSupport::TimeZone, Time, Date, Symbol, BigDecimal, ActiveSupport::HashWithIndifferentAccess]`
> — or switch paper_trail to JSON columns / `PaperTrail.serializer = PaperTrail::Serializers::JSON`. Without this, the Quick Pattern boots but `reify` fails at runtime.

### When to pick / not pick
- **Pick** when the feature is *undo / restore / "view this record as of last week"* —
  reification is the differentiator.
- **Don't pick** when you only need a field-level change log; the full-object
  snapshots are heavier than `audited`'s diffs.

## `logidze`

### Quick Pattern
**Postgres only.** Log lives in a `log_data` JSONB column on the table, written by a
DB trigger — no separate audits table.
```ruby
gem "logidze"
```
```bash
bin/rails generate logidze:install && bin/rails db:migrate   # installs the pg function
bin/rails generate logidze:model Comment --backfill && bin/rails db:migrate
```
```ruby
class Comment < ApplicationRecord
  has_logidze
end

comment.at(version: 2)              # time-travel without a join
comment.diff_from(version: 1)
Logidze.with_responsible(user.id) { comment.update!(body: "edited") }   # actor
```

### When to pick / not pick
- **Pick** for high write volume on Postgres, or when you want history captured even
  for changes made outside Rails (the trigger fires on raw SQL too).
- **Don't pick** on MySQL/SQLite (unsupported), or when the team is uneasy with
  trigger-based logic in migrations and `structure.sql` (DB triggers force
  `structure.sql` — see [migrations.md](migrations.md)).

---

## Cross-cutting pitfalls (all options)

- **Logging sensitive columns into the audit store.** Passwords, tokens, full SSNs,
  card data should be excluded (`audited except:`, paper_trail `skip`, or omit from
  your native `changes`). An audit table is still a data store under the same
  privacy rules — coordinate with `../rails-security/`.
- **Treating the audit log as tamper-proof.** App-level logs are mutable by anyone
  with DB access. If integrity matters, restrict writes/grants and consider an
  append-only/WORM destination — that's a `../rails-deploy/` + `../rails-security/`
  concern, not a model setting.
- **Auditing inside the save transaction on a hot path.** Diff capture adds write
  cost per change; for very hot tables prefer logidze (trigger) or move heavy audit
  side-effects to a job (`../rails-jobs/`).
- **No index on the lookup path.** Querying history by `auditable`/actor/time needs
  indexes on those columns (the generators add the polymorphic one; add actor/time
  if you filter by them). See [migrations.md](migrations.md).

## Verify

```bash
# audited
bin/rails runner 'p Audited::Audit.column_names'                       # audits table present
bin/rails runner 'a=Patient.create!(name:"x"); a.update!(name:"y"); p a.audits.last.audited_changes'

# rails-native
bin/rails runner 'a=Article.create!(title:"x"); a.update!(title:"y"); p Audit.last.audited_changes'

# paper_trail
bin/rails runner 'a=Article.create!(title:"x"); a.update!(title:"y"); p a.versions.last.reify.title'  # => "x" (needs the yaml_column_permitted_classes / JSON-serializer fix from the reify pitfall above)

# logidze (Postgres)
bin/rails runner 'c=Comment.create!(body:"x"); c.update!(body:"y"); p c.at(version:1).body'           # => "x"
```

Each should print a change record / prior state — proving the trail captures
*before* and *after*, not just that the column changed.

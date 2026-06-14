# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This repo **authors a suite of Claude Code Agent Skills for building Ruby on Rails
apps** — it is *not* itself a Rails application. The deliverable is markdown skill
files (`SKILL.md` + `references/`), not running code. There is no Gemfile, build
step, linter, or test harness at the repo level.

**`project-brief.md` is the source of truth.** Read it before doing anything. It
holds the locked scope, the skill list, the house style, and the rationale behind
every decision. Do not re-litigate decisions recorded there — they were settled
with the user across a planning discussion.

## Current status

Planning is complete; the brief is locked. **All 14 `rails-*` skills are built,
reviewed, and findings-closed**, and the repo deliverables (`README.md`,
`INSTALL.md`, `LICENSE`, `.gitmodules`, `CONTRIBUTING.md`) are in place. Whole-suite
link integrity has been verified (no dead local links, no orphans, all `../../`
cross-skill deep-links resolve). The suite is ready to publish.

**Release prep is complete:** the repo is initialized with `origin`
`https://github.com/jcuervo/rails-skills.git`, the `rails-audit` submodule is
materialized from `jcuervo/rails-audit-claude-skill`, and the repo slug is filled in
throughout `README.md`/`INSTALL.md` (no placeholders remain). Further work is
maintenance: extend a skill's menus, add a skill, or refresh version-specific facts —
each via the authoring + review loop below.

**Compliance primitives (added 2026-06-14; reviewed + runtime-validated):** four
technical-primitive references, all gated behind a single **opt-in switch** — a
`STACK.md` `Compliance` row written by `rails-scaffold`'s interview (step 7, default
`none`, recorded explicitly). Set tags make the host skill **proactively offer** the
primitive (it still asks — nothing auto-wires, and nothing asserts the app *is*
legally compliant; conformance/retention/WCAG *levels* are user policy, not code
defaults). Tag → reference map (the contract every consumer reads in its
detect-before-generate step):
- `gdpr`/`ccpa`/`hipaa` → `rails-models/references/data-protection.md` (AR Encryption at rest, consent, export, erasure/anonymization)
- `audit-trail`/`soc2`/`hipaa` → `rails-models/references/audit-trail.md` (audited / native / paper_trail / logidze; logidze is Postgres-only)
- `gdpr`/`ccpa`/`hipaa` → `rails-jobs/references/data-retention.md` (scheduled purge/anonymize past a retention window)
- `wcag`/`section508` → `rails-hotwire/references/accessibility.md` (always baseline for full-stack; the tag *escalates* to a per-view pass + axe CI gate)

`hipaa` fans out to both data-protection and audit-trail; `none`/absent → reactive
(applied only when explicitly asked). All four passed rubric review, a two-axis
consolidated review, **and** Verify-snippet execution against a real Ruby 4.0.1 /
Rails 8.1.3 scratch app — which caught two accuracy fixes now closed: AR Encryption
does **not** auto-read `ACTIVE_RECORD_ENCRYPTION_*` env vars (wire them in config),
and paper_trail `reify` needs `yaml_column_permitted_classes` (or the JSON serializer)
on modern Ruby/Psych.

## Locked decisions (do not re-ask)

- **Targets:** Ruby **4.0.x** (note: Ruby went 3.4 → 4.0, there is no 3.5), Rails **8.1.x**.
- **Scope:** Both API-only and full-stack Hotwire web apps; full lifecycle (dev → deploy/ops).
- **Style:** **Menu-driven and maximally unopinionated** — every concern is asked via `AskUserQuestion`, including the Rails 8 omakase choices. The Rails 8 default is marked **Recommended** within each menu, but the agent still asks unless a pick is already installed in the target project.
- **14 authored skills** (`rails-*`), each one directory with `SKILL.md` + `references/`.
- **Greenfield vs brownfield entry points:** `rails-scaffold` owns *new* apps (`rails new` + guided interview); `rails-upgrade` owns *existing* apps moving to the latest Ruby/Rails (framework version upgrades). Sibling skills own the *destination state* (latest everything); `rails-upgrade` owns the *journey* there and routes to siblings for the omakase stack swaps. Build `rails-upgrade` **last** — it depends on the menus the other skills define.
- **`rails-audit` is referenced, not authored** — it stays upstream ([jcuervo/rails-audit-claude-skill](https://github.com/jcuervo/rails-audit-claude-skill)), pulled in as a git submodule. Our build-time skills cross-link to it and reuse its `security-checklist.md` / `coding-standards-reference.md` as their rubric rather than duplicating them.
- **Distribution:** public GitHub repo, MIT license. Needs README, INSTALL, LICENSE, `.gitmodules`.

## The 14 skills (and merged topics)

`rails-scaffold` (guided new-app interview + router) · `rails-upgrade` (brownfield framework-version upgrades; built last) · `rails-models` (*+persistence/DB*) · `rails-controllers` (*+i18n*) · `rails-api` (*+GraphQL*) · `rails-hotwire` (*+real-time/Action Cable, +rich text/Action Text*) · `rails-auth` (unified authn+authz) · `rails-jobs` · `rails-mailers` (*+inbound/Action Mailbox*) · `rails-storage` · `rails-testing` · `rails-security` (proactive; complements `rails-audit`) · `rails-performance` · `rails-deploy` (*+observability*).

Merged topics live in their host skill's `references/` and must stay **distinctly
invocable**: database → `rails-models`; real-time → `rails-hotwire`; observability →
`rails-deploy`; **inbound email (Action Mailbox) → `rails-mailers`**; **rich text
(Action Text) → `rails-hotwire`**; **i18n/localization → `rails-controllers`**;
**GraphQL → `rails-api`** (as an alternative API style, REST stays Recommended).

The **compliance primitives** follow the same host-reference pattern, gated by the
opt-in `STACK.md` `Compliance` switch (see Current status): **audit trail + GDPR/CCPA
data rights → `rails-models`**; **data retention/purge → `rails-jobs`**;
**accessibility (WCAG/Section 508) → `rails-hotwire`**.

## House style for authoring a skill

Mirror the existing `rn-*` skills (`~/.claude/skills/rn-scaffold-expo/`, etc.) — that
is the reference implementation for this format. Each `SKILL.md`:

1. **Frontmatter** — `name`, `description`, `metadata`; add `user-invocable` + `argument-hint` where it makes sense.
2. **Purpose** — one paragraph.
3. **When to Apply / When NOT to Apply** — cross-link siblings by relative path (e.g. "a test → read `../rails-testing/SKILL.md`"); never duplicate a sibling's coverage.
4. **Menu** — table of vetted options, one-line trade-off each, one marked **Recommended** (the Rails 8 default).
5. **Decision Flow** — selection criteria, not opinions.
6. **Per-option deep dives** in `references/`: Quick Pattern (install + minimum wiring) · When to pick / not pick · Deep Dive · Common Pitfalls · **Verify** (runnable steps).
7. **Problem → Reference table** mapping tasks to files.

## Conventions every skill must follow

- **Detect before generate.** Read `Gemfile`/`Gemfile.lock`/`config/` to learn the target app's Ruby/Rails version and existing choices before acting; if a menu's pick is already installed, use it silently instead of asking.
- **API vs Web awareness.** Shared skills branch on `config.api_only` (API-only skips Hotwire; Web skips serializers).
- **Idempotent.** Patterns must be safe to apply to an existing app, not only a fresh `rails new`.
- **Verify, don't just write.** Every skill ends with runnable verification (boot the app / run the generator / hit the endpoint / run the relevant tests) confirming the wiring works.

## Authoring workflow (the build loop — follow for every skill)

Every skill goes through the same loop. Do not consider a skill done until it has
been reviewed and its findings are closed.

1. **Author** `SKILL.md` + `references/` per the house style above.
2. **Verify Rails facts while authoring** — confirm version-specific flags, gem
   names, generator behavior, and defaults via `rails new --help`/`--help` and web
   search. Prefer baking in a detect/`--help` step over asserting a fact that
   drifts between releases.
3. **Review** — run the `rails-skill-reviewer` subagent on the skill directory
   (`.claude/agents/rails-skill-reviewer.md`). It checks rubric conformance, not
   Rails runtime correctness.
   - *Session caveat:* the subagent registers at Claude Code startup. If it isn't
     yet an available `subagent_type` this session, run the same rubric via a
     `general-purpose` agent told to read `.claude/agents/rails-skill-reviewer.md`
     as its instructions. Same result; the subagent file is the source of truth.
4. **Close findings** — **Blockers and Should-fixes are mandatory.** Apply **Nits**
   too unless explicitly waived (note the waiver). Resolve **Rails-accuracy flags**
   by verifying the claim or adding a `--help`/detect step — never leave one
   asserted-but-unverified. Then state the close-out (e.g. "0 blockers, 2/2
   should-fix, 3/3 nits").
5. **Re-review if blockers existed.** Nit-only passes don't require a re-run.
6. **Record + advance** — note the picks/conventions the skill establishes (e.g.
   the `STACK.md` contract) and move to the next skill in the brief's §7 order.

## Note on "tests" and "running" in this repo

There is nothing to build or run *here*. The runnable verification lives **inside**
the skills (Rails commands the agent runs against a target app), not as a repo test
suite. To sanity-check an authored skill, install it (copy into `~/.claude/skills/`)
and invoke it against a scratch Rails app.

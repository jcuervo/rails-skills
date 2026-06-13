# Contributing

Thanks for improving the Rails Skills suite. This repo **authors Claude Code Agent
Skills** for Ruby on Rails — the deliverable is markdown (`SKILL.md` + `references/`),
not running code. `project-brief.md` is the locked source of truth for scope and
conventions; `CLAUDE.md` is the operational guide. Read both before contributing.

## Ground rules

- **Targets:** Ruby **4.0.x**, Rails **8.1.x**. Lead menus with the Rails 8 omakase
  default marked **Recommended**, then offer alternatives.
- **Menu-driven, unopinionated.** Present choices via `AskUserQuestion`; mark exactly
  **one Recommended** per menu; ask unless the project already constrains the pick.
- **Detect before generate.** Read `Gemfile`/`Gemfile.lock`/`config/` first; use an
  installed pick silently.
- **Verify, don't just write.** Every skill and every reference ends in **runnable**
  verification.
- **Cross-link, don't duplicate.** Reference siblings by relative path; never
  re-document another skill's concern.

## Adding a new menu option to an existing skill

1. Add a row to the skill's **Menu** table in `SKILL.md` — a concise one-line
   trade-off, not a sales pitch. Keep exactly one **Recommended**.
2. Add (or extend) the per-option deep dive in `references/`, following the house
   shape: **Quick Pattern** (install + minimum wiring) · **When to pick / not pick** ·
   **Deep Dive** · **Common Pitfalls** · **Verify** (runnable steps).
3. Update the **Problem → Reference** table if you added a file.
4. **Verify Rails facts.** Confirm version-specific flags/gem names/defaults via
   `rails new --help` / `--help` / the official guides — or bake in a detect step
   rather than asserting a fact that drifts between releases.

## Adding a new skill

1. Mirror the house style of an existing `rails-*` skill. `SKILL.md` carries:
   frontmatter (`name`, `description`, `metadata`; `user-invocable` +
   `argument-hint` where it helps) → Purpose → When to Apply / When NOT (cross-link
   siblings by relative path) → Menu → Decision Flow → Problem → Reference → Verify.
2. Keep `argument-hint` a **quoted string** (`"[x]"`), never a bare `[x]` (bare
   brackets parse as a YAML array and fail validation).
3. Honor the conventions in `CLAUDE.md` §"Conventions every skill must follow"
   (detect-before-generate, API-vs-Web branching on `config.api_only`, idempotency,
   verify-don't-just-write).

## Review before you submit

Every skill goes through the project's review loop (see `CLAUDE.md` →
"Authoring workflow"):

1. **Author** the `SKILL.md` + `references/`.
2. **Review** against the rubric in `.claude/agents/rails-skill-reviewer.md` (run the
   `rails-skill-reviewer` subagent on the skill directory).
3. **Close findings** — Blockers and Should-fixes are mandatory; apply Nits unless
   explicitly waived (note the waiver); resolve Rails-accuracy flags by verifying the
   claim or adding a `--help`/detect step.
4. State the close-out (e.g. "0 blockers, 2/2 should-fix, 3/3 nits") in your PR.

## House conventions checklist (quick self-review)

- [ ] Exactly one **Recommended** per menu.
- [ ] No dead cross-links; no orphan reference files; references one level deep use
      `../../rails-X/...` depth for clickable sibling links.
- [ ] No leaked absolute machine paths (`/Users/...`) in skill text.
- [ ] Every reference ends in a runnable **Verify**.
- [ ] Version-specific claims are verified or driven off a detect/`--help` step.

## The `rails-audit` submodule

`rails-audit` is **upstream** ([jcuervo/rails-audit-claude-skill](https://github.com/jcuervo/rails-audit-claude-skill)),
included via git submodule. Don't edit it here — contribute fixes upstream. Our
quality skills cross-link to it and reuse its checklist as the rubric they build
toward.

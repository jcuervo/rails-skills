# Installing the Rails Skills suite

These are **Claude Code Agent Skills** (`SKILL.md` + `references/` per skill). To use
them, Claude Code must be able to discover them. There are three install paths —
pick the one that matches how you want to scope them.

## Prerequisites

- **Claude Code** installed (`claude` CLI, the desktop/web app, or an IDE extension).
- **git** (to clone, and to pull the `rails-audit` submodule).
- The skills *target* a Rails project; to actually run their verification steps
  you'll need **Ruby 4.0.x** and **Rails 8.1.x** available in that project.

## Option A — Personal skills (all projects)

Copy (or symlink) the skills into your user-level skills directory so they're
available in every project:

```bash
git clone --recurse-submodules https://github.com/jcuervo/rails-skills.git
cd rails-skills

# Copy all 14 authored skills into the personal scope:
cp -R rails-* ~/.claude/skills/

# …or symlink, so `git pull` keeps them current:
for d in rails-*/; do ln -sfn "$PWD/${d%/}" ~/.claude/skills/"${d%/}"; done
```

Each `rails-*` directory must land directly under `~/.claude/skills/` (so the path is
`~/.claude/skills/rails-scaffold/SKILL.md`, etc.).

## Option B — Project skills (one repo, committed to the team)

Vendor the skills into a specific project so your whole team gets them:

```bash
cd your-rails-app
mkdir -p .claude/skills
cp -R /path/to/rails-skills/rails-* .claude/skills/
git add .claude/skills && git commit -m "Add Rails Skills suite"
```

Project-scoped skills live under `<project>/.claude/skills/` and are picked up when
Claude Code runs in that project.

## Option C — Individual skills

You don't need all fourteen. Copy just the ones you want — each skill is
self-contained except for **relative cross-links** to siblings (e.g. `rails-api`
points at `../rails-auth/`). Those links degrade gracefully (they're guidance, not
hard dependencies), but for the smoothest experience install a skill together with
the siblings it routes to:

```bash
cp -R rails-skills/rails-models rails-skills/rails-controllers ~/.claude/skills/
```

## The `rails-audit` submodule

`rails-audit` is **referenced, not authored here** — it's the separately-maintained
[jcuervo/rails-audit-claude-skill](https://github.com/jcuervo/rails-audit-claude-skill),
bundled as a git submodule for the *retrospective* audit complement to
`rails-security`.

If you cloned **without** `--recurse-submodules`, initialize it:

```bash
git submodule update --init --recursive
```

To install it alongside the others:

```bash
cp -R rails-audit ~/.claude/skills/rails-audit     # after the submodule is initialized
```

> Maintainers initializing the repo for the first time: `git submodule add
> https://github.com/jcuervo/rails-audit-claude-skill.git rails-audit` creates the
> link recorded in [`.gitmodules`](.gitmodules).

## Verify the install

```bash
ls ~/.claude/skills/ | grep rails        # the rails-* skills are present
```

Then, in Claude Code on a Rails project:

- Type `/rails-scaffold` (or another skill) to invoke it directly, **or**
- Just describe the task ("set up background jobs", "add an upgrade path to Rails
  8.1") — Claude routes to the matching skill.

## Updating

```bash
cd rails-skills && git pull --recurse-submodules
# If you copied (Option A/B) rather than symlinked, re-copy:
cp -R rails-* ~/.claude/skills/
```

## Uninstall

```bash
rm -rf ~/.claude/skills/rails-*          # personal scope
# or remove <project>/.claude/skills/rails-* for project scope
```

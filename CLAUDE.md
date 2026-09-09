# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **Claude Code plugin marketplace** of agent skills. There is no application
and no build system — the content is Markdown (`SKILL.md` + reference files) and
JSON manifests. Users consume it with `/plugin marketplace add
violetbuse/violets-skills`.

## Layout & the pieces that must agree

```
.claude-plugin/marketplace.json          # marketplace manifest; one entry per plugin, keyed by `source` dir
plugins/<name>/
  .claude-plugin/plugin.json             # plugin manifest
  skills/<skill>/SKILL.md                 # the skill: YAML frontmatter (name, description) + body
  skills/<skill>/reference/*.md           # optional deep-dive files the SKILL.md points to
vendor/<name>/                            # git submodule: upstream source, AUTHORING REFERENCE ONLY
```

When adding or renaming a skill, keep these in sync:
- `marketplace.json` `plugins[].name` / `source` / `description` / `version`
- `plugins/<name>/.claude-plugin/plugin.json` `name` / `description` / `version`
- `SKILL.md` frontmatter `name` (matches the skill directory) and `description`
- the plugin table in `README.md`

## Critical constraint: skills must be self-contained

A plugin ships only its `source` directory (`plugins/<name>/`). Anything under
`vendor/` is **not** delivered when the plugin is installed on another machine.
So:
- Never write a skill that instructs the reader to open a `vendor/celld/...`
  path — that path exists only in this repo. Point instead to `<tool> --help`,
  public docs, or the upstream GitHub repo (which Claude can `WebFetch`).
- `vendor/` submodules exist purely so skill content can be written and verified
  against real upstream source. Pin them to a released tag and state that
  version in the skill (e.g. the `celld` skill tracks celld **v0.4.1**,
  `vendor/celld`). Re-verify version-sensitive claims when bumping.
- When upstream has no tags (a live docs site), pin to a dated commit and say so
  in the skill. The `drizzle-orm` skill tracks **`drizzle-orm` 0.45.2**
  (`vendor/drizzle-orm`) and **`drizzle-orm-docs` @ `11695b7`** (2025-11-19, the
  last commit before the repo started merging v1 content) — chosen so both
  checkouts describe the same stable release, not the v1 RC.
- The `planetscale-postgres-serverless` skill tracks **`@neondatabase/serverless`
  v1.1.0** (`vendor/neon-serverless`); its `CONFIG.md` is slightly stale, so
  verify `neonConfig` defaults against `src/shims/net/index.ts`
  (`Socket.defaults`).

## Working with submodules

```
git clone --recurse-submodules <url>     # or, after a plain clone:
git submodule update --init
```

To add a new reference submodule: `git submodule add <url> vendor/<name>`, then
`cd vendor/<name> && git checkout <released-tag> && cd - && git add vendor/<name>`.

## Validate before committing

No linter is configured. At minimum:
- Both `*.json` manifests must parse (`python3 -m json.tool <file>`).
- `SKILL.md` must start with a `---` YAML frontmatter block containing `name:`
  and `description:`. The `description` is what Claude matches on to decide when
  to load the skill — make it list concrete trigger situations, not a summary.

## Test a plugin locally

```
/plugin marketplace add /home/violet/violets-skills
/plugin install celld@violets-skills
```

## Repo notes

- GitHub remote `origin` → https://github.com/violetbuse/violets-skills
  (public). GitHub's default branch is `master`; local git config names `main`
  as the main branch — confirm the intended branch before opening a PR.
- Commit trailers for this repo are set by the session; follow whatever the
  active attribution guidance says.

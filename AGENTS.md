# Repository Guidelines

Source of truth for personal agent skills. Skills only — no application code.

## Layout

- `jj-atomic-workflow/` — change-boundary decisions while working in a jj-backed repo
- `jj-local-help/` — jj command syntax, flags, keyword topics, concepts, troubleshooting
- `jj-workspaces/` — when to use a second working copy, how to avoid `@` collisions, and how to clean up
- `nix-package-lookup/` — find the Nix package behind a missing command/library/file, and run it temporarily
- `nix-pkg-npm-cli/` — package an npm/Node-distributed CLI or MCP server as a Nix package in the sysadmin flake

Grouping is by topic, not by agent: a skill lives here because it belongs to a subject
area (jujutsu, nix), never because it targets a particular client.

The dividing line is transferable method versus one program's API surface. A skill that is
the manual for a single tool belongs in that tool's repository, where a renamed flag and the
skill edit land in the same commit — `cosmic-desktop-ops` lives in `cosmic-desktop-mcp` for
exactly that reason. What belongs here is knowledge that outlives any one binary.

Each skill is a directory holding `SKILL.md` plus optional `agents/`, `references/`, and
`scripts/` subdirectories. The `description` in front matter is what agents match on — keep
it accurate and trigger-oriented.

Write skills in **English**. They are read by models that reason in English and are
shared publicly; mixing languages inside one file costs tokens and invites drift.
Conversation with the user stays Chinese, the artifacts do not.

## Source of truth

- Edit skills here. This repository is the only place they are authored.
- Never edit deployed copies directly: `~/.agents/skills/`, `~/.config/magpie/library/skills/`,
  `~/.workbuddy/skills/`, `~/.codex/skills/`. Those are user-managed installs and are
  overwritten on the next sync. Change the source here, then let the user re-sync.

## Commit Guidelines

- Subject: `<topic>: <summary>`, for example `skills: document track-editing`. Topics in use:
  `skills`, `docs`, `agents`. Do not use Conventional Commits.
- Include rationale in commit messages, not just what changed. One coherent intent per commit.
- Author: `myul <hdx0729@gmail.com>`.

## Version Control Workflow

- Use the `jj-atomic-workflow` skill for change-boundary decisions and `jj-local-help` for
  command syntax, flags, and installed-version behavior. Only parallelize read-only `jj`
  commands; treat all other `jj` commands as serial, and refresh repo state after
  stale/concurrent/reconcile/divergent output.

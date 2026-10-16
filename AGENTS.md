# Repository Guidelines

Source of truth for the `jj-*` agent skills. Skills only — no application code.

## Layout

- `jj-atomic-workflow/` — change-boundary decisions while working in a jj-backed repo
- `jj-local-help/` — jj command syntax, flags, keyword topics, concepts, troubleshooting

Each skill is a directory holding `SKILL.md` plus optional `agents/`, `references/`, and
`scripts/` subdirectories. The `description` in front matter is what agents match on — keep
it accurate and trigger-oriented.

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

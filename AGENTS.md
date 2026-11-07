# Repository Guidelines

Source of truth for personal agent skills. Skills only — no application code.

## Layout

- `jj-atomic-workflow/` — change-boundary decisions while working in a jj-backed repo
- `jj-local-help/` — jj command syntax, flags, keyword topics, concepts, troubleshooting
- `jj-workspaces/` — when to use a second working copy, how to avoid `@` collisions, and how to clean up
- `nix-temporary-tools/` — run a command or app for one task without installing it, and decide when to persist it
- `nixos-system-model/` — how NixOS differs from a conventional Linux system, and how to make changes survive a rebuild
- `nix-package-lookup/` — find the Nix package behind a missing command/library/file, and run it temporarily
- `sync-debt/` — keep a shrinking ledger of couplings that fail silently, and turn each entry into an automated check

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

## Cross-agent compatibility

Skills here are deployed to more than one agent. Treat `SKILL.md` as the shared
contract: keep its instructions and `name` / `description` usable by every
client, and put client-specific metadata or invocation policy in that client's
supported location. Do not assume a field understood by one agent has meaning
in another.

In particular, Codex reads UI metadata and invocation policy from
`agents/openai.yaml`, while Claude Code reads invocation controls such as
`disable-model-invocation` from `SKILL.md` frontmatter. A skill that must be
user-invoked across agents needs both:

```yaml
# Claude Code, SKILL.md frontmatter
disable-model-invocation: true
```

```yaml
# Codex, agents/openai.yaml
policy:
  allow_implicit_invocation: false
```

Default to model invocation unless the skill is genuinely useful only when the
human asks for it.

Upstream specifications, when checking whether a field or front matter rule
exists and what it means:

- Agent Skills format: <https://agentskills.io/specification>
- Codex skills: <https://developers.openai.com/codex/skills>
- Codex skill authoring: <https://developers.openai.com/codex/skills/create-skill>
- Claude Code skills: <https://code.claude.com/docs/en/skills>

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

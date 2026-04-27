# JJ Skills

This directory contains general-purpose Jujutsu skills for agents working in
jj-backed repositories.

## Setup Note

These skills assume the repository is already managed by jj. For an existing Git
repository, initialize jj once with `jj git init`; this is a one-time setup step,
not part of the normal agent workflow.

## Skills

### `jj-atomic-workflow`

Use this skill to decide what to do with the current `@` while working:

- start by inspecting the current change with `jj status`
- keep `@` aligned with one coherent intent
- decide when to update descriptions
- decide when to start a fresh change
- decide when a mixed change should be split
- spot when boundary drift is already visible
- prefer `jj new` after completing coherent work so later edits do not pollute
  the finished change
- handle interactive or path-based split/commit decisions carefully
- treat `jj commit` as a low-frequency convenience command rather than the center of the workflow

### `jj-local-help`

Use this skill for runtime command, concept, and troubleshooting help:

- confirm exact `jj` command syntax
- confirm flags and subcommands
- look up keyword topics with `jj help -k <topic>`
- explain stable `jj` concepts with local help as the source of truth
- answer version-specific behavior questions from local `jj help`
- troubleshoot command usage without hardcoding unstable docs

## Recommended Use

- Use `jj-atomic-workflow` for change-boundary decisions, current `@`
  questions, and done-state behavior.
- Use `jj-local-help` for command syntax, flags, keyword topics, concepts, and
  installed-version behavior.
- Use both when a workflow decision depends on exact command semantics.

## Recommended AGENTS.md Guidance

To guide agents toward these skills, repository `AGENTS.md` files can include a
short note such as:

```markdown
When working in this jj repository, use the `jj-atomic-workflow` skill for change-boundary decisions and the `jj-local-help` skill for command syntax, flags, installed-version behavior, and safe handling of concurrent/stale/reconciled repo state; only parallelize read-only `jj` commands, treat all other `jj` commands as serial, and refresh repo state after stale/concurrent/reconcile/divergent output.
```

## Portability Note

These skills provide general-purpose guidance, not a complete repository policy.
Repository `AGENTS.md` files and local contributor policies should override
skill defaults where they conflict.

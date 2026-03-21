---
name: jj-atomic-workflow
description: Keep Jujutsu changes atomic while working in repositories that expect small, coherent changes. Use when Codex needs to decide how to manage change boundaries during work, especially when choosing among `jj desc`, `jj new`, `jj split`, and `jj commit`, or when the current `@` change may have drifted away from a single intent.
---

# Jj Atomic Workflow

## Overview

Maintain one clear unit of intent per change while work is in progress.
Treat change management as a continuous workflow concern, not as a final cleanup step.

Use this skill to decide change-boundary strategy: when to rename, split, finish, or start changes.
Use `jj-local-help` to confirm exact `jj` command syntax, flags, and current-version behavior when needed.

## Core Model

- Treat `@` as exactly one atomic change with one clear purpose.
- Keep the description aligned with that purpose from the start of the task.
- Repair boundary drift as soon as it appears; do not wait until the end.
- Prefer simple, explicit state transitions over clever command combinations.

## Command Roles

### `jj desc`

- Use to keep the current change description accurate.
- Run at the start of a task when `@` already exists.
- Run again whenever the scope of the current change materially changes.
- Treat this as the default metadata maintenance command, not as a final polish step.

### `jj new`

- Use to start a fresh empty change when the current one is coherent and the next work item should not mix into it.
- Prefer this over `jj commit` when the current description is already correct.
- Treat this as the normal way to establish a hard boundary before starting unrelated work.

### `jj split`

- Use to repair an already-mixed change.
- Prefer this when one change contains multiple intents and needs to be separated into atomic pieces.
- Prefer this over `jj commit -i` when the main problem is that an existing change boundary is wrong.
- Use this immediately after noticing mixed intent instead of postponing cleanup.

### `jj commit`

- Treat as a low-frequency convenience command, not the center of the workflow.
- Without interactive selection or path arguments, it is effectively `jj desc` followed by `jj new`.
- Use when one command is genuinely clearer than separate `desc` and `new`.
- Use `-i` only when sealing the current `@` while pushing leftover content into the next working-copy change is the clearest move.
- Do not default to this command just because it sounds like the normal way to finish work.

## Decision Rules

Use this order:

1. Check whether `@` still represents one coherent intent.
2. If only the description is wrong, use `jj desc`.
3. If the change is coherent and the next work item is different, use `jj new`.
4. If the change is not coherent, use `jj split`.
5. Use `jj commit` only when it is the clearest shorthand for the exact transition you want.

## Default Preferences

- Prefer `jj desc` as ongoing maintenance.
- Prefer `jj new` to begin the next atomic change.
- Prefer `jj split` to repair mixed content.
- Prefer `jj commit` rarely.

## Guardrails

- Do not let unrelated edits accumulate in `@` once the boundary problem is visible.
- Do not prescribe a rigid sequence of `desc`, `new`, `split`, and `commit`; choose based on the repository state.
- Do not use `jj commit` as a reflexive "finish work" command.
- When exact flags or semantics matter, query live help instead of relying on memory.

## Quick Examples

- The current change is correct but the description is stale: use `jj desc`.
- The current change is complete and the next task is unrelated: use `jj new`.
- The current change contains refactor plus bugfix: use `jj split`.
- You want a one-step "finalize current change and keep working on top": use `jj commit` if that is clearer than `jj desc` plus `jj new`.

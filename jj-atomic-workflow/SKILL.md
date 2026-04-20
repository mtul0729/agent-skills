---
name: jj-atomic-workflow
description: Keep Jujutsu changes atomic while working in repositories that expect small, coherent changes. Use when Codex needs to decide how to manage change boundaries during work, especially when choosing among `jj desc`, `jj new`, `jj split`, and `jj commit`, or when the current `@` change may have drifted away from a single intent.
---

# Jj Atomic Workflow

## Overview

Maintain one clear unit of intent per change while work is in progress.
Treat change management as a continuous workflow concern, not as a final cleanup step.

Use this skill to decide change-boundary strategy: what to do with the current `@`, when to rename, when to split, when to finish, and when to start a fresh change.
Use `jj-local-help` to confirm exact `jj` command syntax, flags, and current-version behavior when needed.

## Core Model

- Treat `@` as exactly one atomic change with one clear purpose.
- Keep the description aligned with that purpose from the start of the task.
- Repair boundary drift as soon as it appears; do not wait until the end.
- Prefer simple, explicit state transitions over clever command combinations.

## Starting-State Triage

At the start of work, inspect the current `@` before choosing a command.

- Always start with `jj status`.
- If `@` has content or the intent is unclear, inspect `jj diff --stat`.
- If the description, parent, or current graph position affects the boundary
  decision, inspect `jj log -r @`.
- If `@` still represents one coherent intent but the description is stale, use `jj desc`.
- If `@` is coherent and the next task is different, use `jj new` before mixing in unrelated work.
- If `@` already contains multiple intents, identify the separate intents immediately.
  Use `jj split` once the split boundary is clear.
- Use `jj commit` only when describing the current change and starting the next one are naturally one step.

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
- Identify mixed intent immediately instead of postponing cleanup.
- If the split boundary is unclear, report the candidate intents and pause for
  clarification before running an interactive or path-based split.

### `jj commit`

- Treat as a low-frequency convenience command, not the center of the workflow.
- Without interactive selection or path arguments, it is effectively `jj desc` followed by `jj new`.
- Use it when two actions are naturally bound together: describe the current change and start the next working-copy change.
- Use `-i` only when sealing the current `@` while pushing leftover content into the next working-copy change is the clearest move.
- Prefer `jj split` instead when the real problem is that the existing change boundary is wrong.
- Do not default to this command just because it sounds like the normal way to finish work.

## Decision Rules

Use this order:

1. Check whether `@` still represents one coherent intent.
2. If only the description is wrong, use `jj desc`.
3. If the change is coherent and the next work item is different, use `jj new`.
4. If the change is not coherent, identify the split boundary, then use `jj split`.
5. Use `jj commit` only when it is the clearest shorthand for the exact transition you want.

## Boundary Drift Signals

Act before continuing to pile edits into `@` when any of these become true:

- The edits need different change descriptions.
- The edits would be reviewed independently.
- One part could be reverted without the other.
- The edits need different tests, rationale, or release notes.

When one of these signals appears, either start a new change with `jj new` or
identify the boundary and repair the mixed change with `jj split`.

## Default Preferences

- Prefer `jj desc` as ongoing maintenance.
- Prefer `jj new` to begin the next atomic change.
- Prefer `jj split` to repair mixed content.
- Prefer `jj commit` rarely.

## Guardrails

- Do not let unrelated edits accumulate in `@` once the boundary problem is visible.
- Do not prescribe a rigid sequence of `desc`, `new`, `split`, and `commit`; choose based on the repository state.
- Do not use `jj commit` as a reflexive "finish work" command.
- Do not treat this skill as the complete repository JJ workflow.
- Follow repository-specific workflow policy first. In this repository, commands
  such as `jj rebase`, `jj squash`, and `jj abandon` are high-risk and require
  confirmation before execution.
- Use `jj desc` and `jj new` as normal boundary-maintenance commands.
- Use `jj split -i`, `jj commit -i`, or path/fileset-based `jj commit` only
  when the intended selection is clear and the environment supports the
  required interaction. Otherwise, ask or use a clear non-interactive boundary.
- When exact flags or semantics matter, query live help instead of relying on memory.

## Done State

When a coherent unit of work is complete, prefer creating a fresh working-copy
change with `jj new`. This protects the completed change from accidental edits
during follow-up work.

- The completed change should contain one intent and have an accurate description.
- The new `@` should be empty and ready for the next unrelated task.
- Skip the final `jj new` only if the user explicitly asks to stay on the
  completed change or the next action continues the same intent.
- Verify the final state with `jj status`.

## Quick Examples

- You start work and `@` already matches the task, but the description still names yesterday's scope: run `jj desc`.
- You finished one coherent change and are about to start a different task: run `jj new` before making the next edit.
- You notice `@` now contains a refactor and an unrelated doc cleanup: identify the two intents immediately, then run `jj split` once the boundary is clear.
- You want one command that both finalizes a coherent current change and leaves you on the next working-copy change: use `jj commit` only if that is clearer than `jj desc` plus `jj new`.
- You complete a coherent change and no further same-intent edits are planned:
  run `jj new` so later work starts from a fresh empty `@`.

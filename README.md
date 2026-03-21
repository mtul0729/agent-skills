# JJ Skills

This directory contains two complementary Jujutsu skills.

## Skills

### `jj-atomic-workflow`

Use this skill for change-boundary strategy while working:

- keep `@` aligned with one coherent intent
- decide when to update descriptions
- decide when to start a new change
- decide when a mixed change should be split
- treat `jj commit` as a low-frequency convenience command rather than the center of the workflow

### `jj-local-help`

Use this skill for runtime command, concept, and troubleshooting help:

- confirm exact `jj` command syntax
- confirm flags and subcommands
- explain stable `jj` concepts with local help as the source of truth
- answer version-specific behavior questions from local `jj help`
- troubleshoot command usage without hardcoding unstable docs

## Recommended Use

Use `jj-atomic-workflow` when the question is "how should I manage changes while I work?"

Use `jj-atomic-workflow` when the question is specifically about choosing among
`jj desc`, `jj new`, `jj split`, and `jj commit`.

Use `jj-local-help` when the question is "what does this `jj` command or concept
mean in the installed version?"

Use both together when workflow decisions and exact command semantics both matter.

## Workflow Note

These skills do not try to define the complete `jj` workflow for this repository.

For now:

- `jj-atomic-workflow` covers atomic change management
- `jj-local-help` covers runtime command help

A fuller repository workflow may be documented later, but it is intentionally
not part of either skill today.

The rough shape of that fuller workflow is:

- keep stacking small atomic changes during normal work
- keep `@` described accurately as scope evolves
- start a fresh change before unrelated work mixes into the current one
- split mixed changes promptly when boundaries drift
- use live local help when exact command behavior matters
- clean up history before publish when needed, but do not rely on late cleanup as the default way to maintain atomicity

# `jj-local-help`

`jj-local-help` is a runtime help skill for Jujutsu (`jj`).
It is used when users ask about `jj` commands, flags, errors, or workflows.
The key behavior is to resolve command details from local `jj help` / `jj --help` output instead of relying on hardcoded docs.

## Core Advantage

This skill is designed to query local `jj help` / `jj --help` at runtime instead of hardcoding command details in `SKILL.md`.
It is not bound to a specific `jj` version.
Because of that, when `jj` is upgraded, the skill usually does not need to be updated just to support new flags or subcommands.

## Usage Recommendation

To keep change management explicit and safe, add the following section to
`AGENTS.md`:

```md
## Commit & Change Management

- Use `jj commit` for normal submission flow.
- Treat `jj rebase`, `jj squash`, `jj abandon`, and other history-rewriting commands as high risk; confirm before running them.
- Keep work accumulated on `@` during normal iteration. Use `jj desc` at the start of a task and keep it accurate whenever the scope changes.
- If `@` already has changes, update the description before continuing.
- Use `jj new` only when the current task is complete enough to split from the next one.
```

## Disclaimer

Using a coding agent together with this skill will not automatically turn you from a beginner into an expert.
This skill is recommended when you already have basic hands-on experience with `jj`.

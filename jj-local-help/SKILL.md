---
name: jj-local-help
description: Resolve Jujutsu (`jj`) command usage, flags, errors, keyword help, and version-specific semantics from the locally installed `jj` help system instead of hardcoding unstable docs. Use when Codex needs current `jj` command syntax, subcommands, troubleshooting guidance, or installed-version behavior.
---

## Reliability Policy (Important)

`jj` evolves quickly. Do not hardcode option/flag details in this file.

For command behavior:
1. Prefer live local help text from `jj`.
2. Use this skill's concept notes only for stable mental models.
3. If concept notes conflict with live help, trust live help and state that notes may lag behind the installed version.
4. For keyword topics (`jj help -k ...`), treat keyword help as authoritative over `references/core-concepts.md`.

## Help System Bootstrap Workflow (Always First)

Run these before answering command or concept questions:

```bash
jj help
jj help help
```

Use the first command to confirm top-level subcommands for the installed version.
Use the second command to confirm help semantics and keyword topics (`-k/--keyword`).

Then read `references/core-concepts.md` early to refresh the stable `jj` mental
model before routing the specific query. Treat it as orientation, not as the
source of exact syntax or version-specific behavior.

## Query Routing Workflow

### 1) Classify the user request

- Command-oriented: "How do I run X?"
- Concept-oriented: "What does X mean in jj?"
- Troubleshooting-oriented: "Why did command X fail?"

Before going deeper, skim `references/core-concepts.md` if the question depends
on `jj` mental models such as `@`, change vs commit, bookmarks, revsets,
operation log, or immutable revisions.

### 2) Route command-oriented requests

Resolve the command path, then fetch live help:

```bash
jj rebase --help
jj git push --help
jj operation log --help
```

Default source order for command questions:

- Exact command syntax, flags, subcommands -> `jj <command> --help`
- Keyword-driven docs topics -> `jj help -k <topic>`
- Extra explanation of stable concepts -> `references/core-concepts.md`

### 3) Route concept-oriented requests

Use `references/core-concepts.md` first to frame the answer, then confirm the
installed version's terminology and details with keyword help:

```bash
jj help -k tutorial
jj help -k glossary
jj help -k config
jj help -k revsets
jj help -k templates
jj help -k filesets
```

Treat `jj help -k` output as the primary source of truth for those topics.
Use `references/core-concepts.md` to:
- surface the right mental model early
- organize and simplify explanations
- connect concepts to practical command workflows
- cover concepts not directly exposed by `-k` topics

When both are available, cite and follow `jj help -k` first, then add a concise synthesis from `core-concepts.md`.

Default source order for concept questions:

- Stable mental model and Git-difference framing -> `references/core-concepts.md`
- Installed terminology and detailed topic docs -> `jj help -k <topic>`
- Exact command syntax mentioned in the explanation -> `jj <command> --help`

### 4) Route troubleshooting requests

- Check command help for expected flags/arguments.
- Point out likely mismatch with current version/configuration.
- Suggest exact follow-up command(s) to validate state.

Default source order for troubleshooting:

- Expected syntax and flags -> `jj <command> --help`
- Related topic behavior or policy -> `jj help -k <topic>`
- Stable concept framing when needed -> `references/core-concepts.md`

## Concept Priority and Mapping

Use this mapping for concept-first questions:

- Getting started and learning path -> `tutorial`
- Term meaning or ambiguous wording -> `glossary`
- Behavior differences or policy configuration -> `config`
- Revision selection expressions -> `revsets`
- Output customization -> `templates`
- Path/file selection expressions -> `filesets`

## Scope Boundaries (Required)

Use this skill for:

- installed-version command syntax and flags
- subcommand discovery
- keyword topic lookup from local help
- troubleshooting command usage or behavior mismatches
- explaining stable `jj` concepts with local help as the source of truth

Do not use this skill for:

- deciding change-boundary strategy such as when to `desc`, `new`, `split`, or
  `commit`; use `jj-atomic-workflow`
- repository-specific workflow policy; use the repository `AGENTS.md`
- inventing command syntax from memory when local help is available

## Response Contract (Required)

For every response, use this order:
1. Direct answer in one sentence.
2. Source line: installed version (`jj --version`) and help path used.
3. 1-3 runnable commands.
4. Caveat or pitfall note if relevant.

Prefer the smallest correct command example first.
Do not list multiple equivalent command variants unless the distinction matters
for correctness or the user explicitly asks for alternatives.

For concept answers, default to four layers:
1. What it is.
2. How it differs from common Git assumptions.
3. Why it matters.
4. Common pitfall or boundary.

Do not dump full docs. Extract only the parts needed for the specific question.

## Rewrite Safety Guardrails (Required)

Execution model:
- Do not run mutating `jj` commands in parallel. Commands such as `jj commit`,
  `jj new`, `jj desc`, `jj rebase`, `jj squash`, `jj abandon`, `jj resolve`,
  and `jj git push` must be executed one at a time.
- If a workflow requires multiple mutating `jj` commands, run them serially in a
  single shell session and stop on failure. Prefer a single command string with
  `&&` over parallel tool calls that can race with each other.
- Only read-only discovery commands such as `jj --version`, `jj status`, and
  `jj help ...` may be parallelized.

Flag usage principle:
- Prefer default behavior or the minimal argument set.
- Do not add flags unless they are required by the user request or necessary to preserve exact semantics.
- If multiple flag combinations are possible, confirm before adding any non-essential flag.

Before any history-rewriting command (`rebase`, `squash`, `abandon`):
1. Echo the exact command to be run and the intended effect in one line.
2. If command shape is ambiguous (for example `-b` vs `-r`), ask and resolve it before execution.
3. Do not auto-resolve conflicts (`:ours`/`:theirs`) unless the user explicitly authorizes that strategy.

On conflict:
- Pause after reporting conflicted paths and choices.
- Wait for user direction before applying a conflict resolution tool or strategy.

## Fallback Behavior

- If local `jj` is unavailable, say that live help cannot be queried and provide best-effort guidance with an explicit uncertainty note.
- If command lookup fails, ask a focused follow-up or suggest discovery:
  - `jj --help`
  - `jj help help`
  - `jj help <command>`
  - `jj help -k <keyword>` (for docs topics such as `tutorial`, `glossary`, `config`, `revsets`, `templates`, `filesets`)

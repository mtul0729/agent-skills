---
name: jj-local-help
description: Resolve Jujutsu (`jj`) command usage, flags, errors, and workflow questions from the locally installed `jj` help system instead of hardcoding unstable docs. Use when Codex needs current `jj` command syntax, keyword help, troubleshooting guidance, or concrete `jj` workflow advice.
---

## Reliability Policy (Important)

`jj` evolves quickly. Do not hardcode option/flag details in this file.

For command behavior:
1. Prefer live local help text from `jj`.
2. Use this skill's concept notes only for stable mental models.
3. If concept notes conflict with live help, trust live help and state that notes may lag behind the installed version.
4. For keyword topics (`jj help -k ...`), treat keyword help as authoritative over `references/core-concepts.md`.

## Help System Bootstrap Workflow (Always First)

Resolve every relative path in this skill against the skill directory, not the
current workspace root. Change into `skills/jj-local-help/` before invoking
bundled scripts, or use an absolute path. Do not assume that a workspace has a
top-level `skills/` directory unless it is actually present.

Run these before answering command or concept questions:

```bash
./scripts/jj_help.sh help
./scripts/jj_help.sh help help
```

Use the first command to confirm top-level subcommands for the installed version.
Use the second command to confirm help semantics and keyword topics (`-k/--keyword`).

## Query Routing Workflow

### 1) Classify the user request

- Command-oriented: "How do I run X?"
- Concept-oriented: "What does X mean in jj?"
- Troubleshooting-oriented: "Why did command X fail?"

### 2) Route command-oriented requests

Resolve the command path, then fetch live help:

```bash
./scripts/jj_help.sh rebase
./scripts/jj_help.sh git push
./scripts/jj_help.sh operation log
```

### 3) Route concept-oriented requests

First try keyword help:

```bash
./scripts/jj_help.sh help -k tutorial
./scripts/jj_help.sh help -k glossary
./scripts/jj_help.sh help -k config
./scripts/jj_help.sh help -k revsets
./scripts/jj_help.sh help -k templates
./scripts/jj_help.sh help -k filesets
```

Treat `jj help -k` output as the primary source of truth for those topics.
Use `references/core-concepts.md` only to:
- organize and simplify explanations
- connect concepts to practical command workflows
- cover concepts not directly exposed by `-k` topics

When both are available, cite and follow `jj help -k` first, then add a concise synthesis from `core-concepts.md`.

### 4) Route troubleshooting requests

- Check command help for expected flags/arguments.
- Point out likely mismatch with current version/configuration.
- Suggest exact follow-up command(s) to validate state.

## Concept Priority and Mapping

Use this mapping for concept-first questions:

- Getting started and learning path -> `tutorial`
- Term meaning or ambiguous wording -> `glossary`
- Behavior differences or policy configuration -> `config`
- Revision selection expressions -> `revsets`
- Output customization -> `templates`
- Path/file selection expressions -> `filesets`

## Response Contract (Required)

For every response, use this order:
1. Direct answer in one sentence.
2. Source line: installed version (`jj --version`) and help path used.
3. 1-3 runnable commands.
4. Caveat or pitfall note if relevant.

For concept answers, default to three layers:
1. What it is.
2. Why it matters.
3. Common pitfall or boundary.

Do not dump full docs. Extract only the parts needed for the specific question.

## Rewrite Safety Guardrails (Required)

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

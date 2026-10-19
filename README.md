# Agent Skills

Personal agent skills, authored here and deployed to the agents on this machine
through magpie. Each top-level directory is one skill; grouping is by topic, not
by client.

## Skills

### Jujutsu

#### `jj-atomic-workflow`

Use this skill to decide what to do with the current `@` while working:

- start by inspecting the current change with `jj status`
- keep `@` aligned with one coherent intent
- decide when to update descriptions
- decide when to start a fresh change
- decide when a mixed change should be split
- spot when boundary drift is already visible
- prefer `jj new` after completing coherent work so later edits do not pollute
  the finished change
- start unrelated work from `@-` when the current `@` is someone else's unfinished
  change
- handle interactive or path-based split/commit decisions carefully
- treat `jj commit` as a low-frequency convenience command rather than the center of the workflow

#### `jj-local-help`

Use this skill for runtime command, concept, and troubleshooting help:

- confirm exact `jj` command syntax
- confirm flags and subcommands
- look up keyword topics with `jj help -k <topic>`
- explain stable `jj` concepts with local help as the source of truth
- answer version-specific behavior questions from local `jj help`
- troubleshoot command usage without hardcoding unstable docs

#### Setup Note

These skills assume the repository is already managed by jj. For an existing Git
repository, initialize jj once with `jj git init`; this is a one-time setup step,
not part of the normal agent workflow.

### Nix

#### `nix-package-lookup`

Use this skill when a command, library, or file is missing and you need to know
which Nix package provides it:

- locate the providing package with `nix-locate` before guessing a name
- run a command temporarily with an explicit `nix shell nixpkgs#<pkg> -c ...`
- decide when a tool should move into NixOS or Home Manager configuration instead
- avoid `nix-channel --update` workflows on this flakes + `nix-index-database` system

#### `nix-pkg-npm-cli`

Use this skill when packaging an npm/Node-distributed CLI, MCP server, or agent
tool as a Nix package in the sysadmin flake (`systems/nixos/pkgs/*`):

- decide between `buildNpmPackage` and a prebuilt-tarball assembly
- compute the `src` hash and `npmDepsHash` without network surprises
- follow the repository's `default.nix` + `update.py` contract
- audit the dependency tree for install-script side effects before building
- verify with a real build rather than static checks alone

## Recommended Use

- Use `jj-atomic-workflow` for change-boundary decisions, current `@`
  questions, and done-state behavior.
- Use `jj-local-help` for command syntax, flags, keyword topics, concepts, and
  installed-version behavior.
- Use both when a workflow decision depends on exact command semantics.
- Use `nix-package-lookup` for command-not-found and package-provision questions.
- Use `nix-pkg-npm-cli` when adding a new package under `systems/nixos/pkgs/`.

## Recommended AGENTS.md Guidance

To guide agents toward the jj skills, repository `AGENTS.md` files can include a
short note such as:

```markdown
When working in this jj repository, use the `jj-atomic-workflow` skill for change-boundary decisions and the `jj-local-help` skill for command syntax, flags, installed-version behavior, and safe handling of concurrent/stale/reconciled repo state; only parallelize read-only `jj` commands, treat all other `jj` commands as serial, and refresh repo state after stale/concurrent/reconcile/divergent output.
```

## Portability Note

These skills provide general-purpose guidance, not a complete repository policy.
Repository `AGENTS.md` files and local contributor policies should override
skill defaults where they conflict.

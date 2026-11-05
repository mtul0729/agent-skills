---
name: nix-temporary-tools
description: Use when an agent needs a command, tool, or app for one task without installing it — `nix shell`, `nix run`, `nix develop` — or must decide whether to keep it in `home.packages` or `environment.systemPackages`. Covers `-c`, `--`, pipelines, multiple packages, registry vs flake, and the temporary/persistent boundary.
---

# Nix Temporary Tools

## The rule: choose the smallest scope

Agents often need a tool for one command. Use the smallest scope that solves
the task; promote it to the config only when it is needed again.
These are Nix CLI commands; they work on any Nix installation, not just NixOS.

| Need | Use | Persists? |
| --- | --- | --- |
| One command | `nix shell nixpkgs#pkg -c cmd` | no |
| Several tools, pipeline | `nix shell nixpkgs#bash nixpkgs#a nixpkgs#b -c bash -c '...'` | no |
| One app | `nix run nixpkgs#app -- args` | no |
| Project dev environment | `nix develop .#devShell` | no |
| Keep a tool for the user | `home.packages` (Home Manager) | yes, declarative |
| Keep a tool for the system | `environment.systemPackages` (NixOS) | yes, declarative |
| Build, run manually | `nix build .#pkg`, then `./result/bin/...` | no |

None of the temporary forms need root. They do not edit the system, the user
profile, or the config.

## `nix shell`

Put one or more packages on `PATH` for one command:

```bash
nix shell nixpkgs#jq -c jq --version
nix shell nixpkgs#jq -c jq . file.json
nix shell nixpkgs#python3 -c python3 -m http.server 8000
nix shell nixpkgs#bash nixpkgs#curl nixpkgs#jq -c bash -c 'curl -sSL URL | jq .'
```

- `-c` (`--command`) runs the command directly; everything after it is argv.
- `nix shell ... -c` does **not** start a shell. For pipes, redirection,
  `&&`, or variables, wrap in a shell: `-c bash -c '...'` (or `sh -c`).
- Without `-c`, it opens an interactive shell with the packages on `PATH`;
  use that for exploration, not for an agent's one-shot command.
- All binaries in the package's `bin` are on `PATH`; the command name does not
  have to equal the attribute name (`nixpkgs#python3Packages.foo -c foo`).
- List multiple packages before `-c`.
- Packages disappear from `PATH` when the command exits; the store paths
  remain cached.

## `nix run`

Run a package's main program:

```bash
nix run nixpkgs#hello
nix run nixpkgs#hello -- --help
nix run .#myApp -- arg1 arg2
```

- `--` separates Nix's options from the program's arguments. Without it,
  `--help` may be consumed by `nix run`.
- It runs one program. For a derivation, it picks the binary named by
  `meta.mainProgram`, then `pname`, then `name`. If the binary you need is not
  the main one, use `nix shell ... -c <binary>`.
- It does not put the package's other binaries on `PATH`; a program that
  shells out to a sibling binary may need `nix shell`.
- For a flake, `.#app` resolves `apps.<system>.<name>` or a package with
  `meta.mainProgram`.
- It is temporary: no profile or system change.

## `nix develop`

- `nix develop .#devShell` enters a project's dev shell (dependencies,
  environment variables, `shellHook`).
- `nix develop .#devShell -c cmd` runs one command inside it.
- Use it when the project defines the environment; use `nix shell` for ad-hoc
  packages.

## Temporary vs persistent

- `nix shell` / `nix run` / `nix develop` / `nix build`: no profile or system
  change.
- `environment.systemPackages` (NixOS) or `home.packages` (Home Manager):
  declarative persistence; the right home for a tool used repeatedly.
- Imperative profile commands: do not use them to persist a tool on a managed
  system.
- `apt`, `pip install`, `npm -g`, `curl | bash`: do not use for a one-off;
  they write outside the Nix store and may not survive.

## Registry vs flake

- `nixpkgs#foo` uses the registry's nixpkgs, which may differ from the
  project's pinned nixpkgs.
- `.#foo` uses the current flake's pinned nixpkgs; prefer it inside a project
  when the version matters.
- `github:owner/repo#foo` uses a remote flake.
- `nix shell` / `nix run` do not update pins; they use the lock or registry
  as-is. Use `nix flake update` only when the task is to move pins.

## Gotchas

- `nix shell ... -c` is not a shell; pipelines need `-c bash -c '...'`.
- `nix run` only runs one program; sibling binaries need `nix shell`.
- `--` is required to pass options like `--help` to the program.
- The first invocation may download or build; later invocations reuse the
  store.
- `nix shell` / `nix run` do not persist. Persist a tool in `home.packages`
  or `environment.systemPackages` only when it is needed repeatedly;
  imperative profile commands sit outside the config.
- `nix shell` is the current CLI; `nix-shell -p` is legacy but also temporary.
- `nix build` only builds; run `./result/bin/...` or use `nix run`.
- If a command needs shell features, use `-c bash -c '...'` and quote
  carefully.

## Examples

```bash
# JSON inspection
nix shell nixpkgs#jq -c jq . file.json

# HTTP without installing curl
nix shell nixpkgs#curl -c curl -sSL https://example.com

# Pipeline: the inner shell owns the pipe
nix shell nixpkgs#bash nixpkgs#curl nixpkgs#jq -c bash -c 'curl -sSL URL | jq .'

# Run an app once
nix run nixpkgs#hello -- --help

# Run a project app
nix run .#myApp -- --help
```

# The `nix` CLI

## Command families

`nix` is the package, store, and flake CLI. It builds and inspects; it does
not activate a NixOS system.

| Family | What it owns | Examples |
| --- | --- | --- |
| `nix` | packages, store, flakes, dev shells | `nix build`, `nix eval`, `nix shell`, `nix flake` |
| `nixos-*` | the running system's lifecycle and option tree | `nixos-rebuild`, `nixos-option`, `nixos-version` |
| `home-manager` | a standalone user environment | `home-manager switch`, `home-manager generations` |
| `nix-*` legacy | older commands still common in docs | `nix-env`, `nix-build`, `nix-shell`, `nix-channel` |
| wrappers | repository-specific deployment | `nh`, `deploy-rs`, `colmena`, `nixos-rebuild --target-host` |

Integrated Home Manager is part of the NixOS system: use `nixos-rebuild`, not
`home-manager switch`.

## Installables

Most `nix` commands take an **installable**:

```text
nixpkgs#hello                 # registry entry + attribute
.#hello                       # current directory's flake + attribute
github:owner/repo#hello       # remote flake + attribute
/nix/store/...                # an existing store path
```

`nixpkgs` alone is the whole flake, not a package. `--` separates the
installable from the program's arguments:

```bash
nix run nixpkgs#hello -- --help
nix shell nixpkgs#jq -c jq --version
```

## Temporary commands

See the `nix-temporary-tools` skill for `nix shell`, `nix run`, `nix develop`,
pipelines, and the temporary/persistent boundary. Short version:
`nix shell nixpkgs#<pkg> -c <cmd>` for a tool,
`nix run nixpkgs#<app> -- <args>` for an app; both are temporary and need no
root.

## Task map

| Want | Command |
| --- | --- |
| List flake outputs | `nix flake show .` |
| Inspect inputs and lock | `nix flake metadata .` |
| Check flake outputs without building | `nix flake check --no-build` |
| Update one input | `nix flake update <input>` |
| Evaluate a value | `nix eval .#attr` (`--json`, `--raw`) |
| Build an output | `nix build .#attr` (leaves `./result`) |
| Build without a result link | `nix build --no-link --print-out-paths .#attr` |
| Run a package | `nix run nixpkgs#hello -- args` |
| Temporary shell with a package | `nix shell nixpkgs#jq -c jq ...` |
| Enter a dev shell | `nix develop .#devShell` |
| Interactive evaluation | `nix repl`, then `:lf .` |
| Search packages | `nix search nixpkgs <name>` |
| Show store info | `nix path-info -rsh /run/current-system` |
| Compare closures | `nix store diff-closures <old> <new>` |
| Find why a path is in a closure | `nix why-depends /run/current-system /nix/store/<path>` |
| Garbage-collect unreachable store paths | `nix store gc` |
| Delete old generations | `nix-collect-garbage -d` |

## State changes

Inspection (does not change the system; may download or evaluate):

```text
nix flake show / metadata
nix flake check --no-build
nix eval
nix search
nix path-info
nix why-depends
nix store diff-closures
nix repl
```

These build or download but do not change the system:

```text
nix flake check
nix build
nix run
nix shell
nix develop
```

Imperative profile commands change a profile. Avoid them on a managed system;
persist tools in the config instead.

These change the NixOS system:

```text
nixos-rebuild test / switch / boot
```

Destructive:

```text
nix store gc                      # unreachable store paths only
nix-collect-garbage -d            # old generations, then collect
nix store delete                  # delete a store path (dangerous)
```

`nix store gc` does not delete generations; `nix-collect-garbage -d` does.
Old generations are the rollback path.

## Persistence

On a NixOS or Home Manager system, persist a tool in the config, not in an
imperative profile:

```nix
home.packages = [ pkgs.foo ];              # Home Manager
environment.systemPackages = [ pkgs.foo ]; # NixOS
```

Imperative profile commands sit outside the config and are not reproducible;
do not use them as the persistence path on a managed system.

## Building a NixOS system without activating it

`nix build` can build the system closure directly:

```bash
nix build .#nixosConfigurations.<host>.config.system.build.toplevel
```

That builds and leaves `./result`, but does not activate.
`nixos-rebuild build --flake .#host` is the same idea with NixOS-aware
defaults.

## Legacy to modern

| Legacy | Modern |
| --- | --- |
| `nix-build` | `nix build` |
| `nix-shell -p foo` | `nix shell nixpkgs#foo` |
| `nix-shell` (dev shell) | `nix develop` |
| `nix-instantiate --eval` | `nix eval` |
| `nix-env -iA nixpkgs.foo` | `home.packages` (Home Manager) or `environment.systemPackages` (NixOS) |
| `nix-channel --update` | `nix flake update` (flake-based systems) |
| `nix-collect-garbage -d` | still the way to delete old generations; `nix store gc` only collects unreachable paths |

Legacy commands still appear in blog posts and older manuals. Translate them
before copying a recipe. Do not mix `nix-channel` with a flake-managed
system.

## Gotchas

- `nix` commands can write `flake.lock`. For read-only evaluation, pass
  `--no-write-lock-file`.
- A flake in a git-backed tree ignores untracked files; stage a new file
  before it can be evaluated.
- Persist a user tool in `home.packages`, or a system tool in
  `environment.systemPackages`; imperative profile commands sit outside the
  config.
- `nix build .#...` only builds. Activation is `nixos-rebuild` or
  `home-manager switch`.
- `nix search nixpkgs` uses the registry, not necessarily the flake's pinned
  nixpkgs.
- `nix store gc` and `nix-collect-garbage -d` are not interchangeable: the
  first collects unreachable paths, the second also deletes old generations.

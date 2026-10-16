---
name: nix-package-lookup
description: Use on NixOS flakes systems when Codex needs to find which Nix package provides a missing command, executable, library, or file; diagnose command-not-found or package lookup issues; or temporarily run a command without adding it permanently to NixOS or Home Manager configuration. Prefer this over guessing package names or using nix-channel workflows.
---

# Nix Package Lookup

This system uses flakes with `nix-index-database`. Do not suggest `nix-channel --update` for command lookup problems.

## Lookup Policy

Use `nix-locate` before guessing package names.

For a missing executable command:

```bash
nix-locate --at-root --whole-name --type x /bin/<command>
```

For a library or arbitrary file:

```bash
nix-locate lib/libssl.so
nix-locate share/applications/foo.desktop
```

If multiple packages provide a command, report the candidates and choose only when the surrounding task makes the right package clear.

## Temporary Execution

Prefer explicit `nix shell` for agent-run temporary commands:

```bash
nix shell nixpkgs#<package> -c <command> [args...]
```

Examples:

```bash
nix shell nixpkgs#jq -c jq --version
nix shell nixpkgs#dnsutils -c dig example.com
```

Use `comma` only as an interactive convenience when ambiguity is low:

```bash
, jq --version
```

Do not use `comma` as the default automation path when an explicit `nix shell nixpkgs#... -c ...` command is available.

## Permanent Installation

If the tool is needed long term, add it to the relevant NixOS or Home Manager configuration instead of relying on temporary execution.

In the sysadmin repository, follow its NixOS workflow and use `just switch` from `systems/nixos` after configuration changes.

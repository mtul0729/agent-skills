---
name: nixos-system-model
description: Use when working on a NixOS machine or flake — installing packages, enabling services, changing users/network/boot/mounts, debugging a failed rebuild or service, or when a task would otherwise be done the traditional Linux way (apt, editing /etc, systemctl enable, crontab, fstab, iptables). Covers generated state, option lookup, the `nix` CLI, and making changes survive a rebuild.
---

# NixOS System Model

## The system is a build output

NixOS builds a **closure** from a Nix expression and activates it.
`/run/current-system` is a symlink to that closure. The managed parts of
`/etc`, systemd units, users and groups, firewall rules, mounts, and boot/kernel
settings are **generated or applied** by the build; they are outputs, not the
source of truth.

Three consequences shape every task:

- The authoritative copy of a generated file or unit is the Nix expression
  that produced it. A manual edit changes the running machine, but it is not
  part of the build; the next activation replaces managed entries and can
  silently discard the change. Mutable-user records are the exception only
  while `users.mutableUsers = true`.
- The generation switch is atomic. A rebuild produces a new **generation**;
  the previous generation stays available for rollback. Activation itself is
  a sequence of unit changes, so a failed activation can leave a partially
  switched system.
- Packages are not upgraded in place. The closure is rebuilt against one
  pinned package set; `nix flake update` (or a channel update) moves that set,
  and every dependent package moves with it. There is no partial-upgrade
  state to repair. Because inputs are pinned, the same expression and lock
  should produce the same closure; this is what makes rollback and
  reproduction meaningful.

State outside this model — `/nix/store` (immutable), `/var/lib`, `/home`,
databases, application data — is not migrated by a rebuild. A rebuild can
change the package without migrating its data. Treat schema migrations,
backups, and state as separate work.

If the task is "make this persist", the answer is almost always "find the
option that owns the concern and set it in the module".

The config is the source of truth; the running system is one build of it.
Before running a command, ask whether it changes the running system or the
expression. If it only changes the running system, find the expression.

## Workflow

1. **Locate the source of truth.** Find the flake or channel the machine was
   built from, the host attribute if it is a flake, and the running closure.
   Completion: you can name all three and point to the `.nix` file that owns
   the concern.
2. **Find the option.** Use the live option tree, not memory. Completion: you
   have the option's type, default, description, and the files that declare
   and define it.
3. **Edit the module.** Change the imported `.nix` file, or add a module and
   import it. Completion: the change is in the source of truth, not in `/etc`,
   a unit file, or a store path.
4. **Evaluate and build.** Completion: evaluation succeeds and the new
   closure builds. This phase needs no root.
5. **Activate.** Completion: the new generation is current and no unit newly
   failed.
6. **Verify or roll back.** Completion: the intended behavior is observed, or
   the previous generation is active again.

Do not skip from step 3 to step 5. A build failure is cheap; a half-activated
system is not.

## Look up the option

Option names, defaults, package names, and module paths change between
releases. Query the installed system or the pinned flake.

```bash
nixos-version
readlink -f /run/current-system

# Type, default, description, and declaring/defining files.
nixos-option -F /path/to/flake#host services.nginx.enable
nixos-option -F /path/to/flake#host -r services.openssh

# All option documentation offline. Search it; it is large.
man configuration.nix

# Final merged value after every module and overlay.
nix eval --no-write-lock-file \
  .#nixosConfigurations.<host>.config.<dotted.attr>

# Package search, and which package provides a command or file.
nix search nixpkgs <name>
nix-locate --at-root --whole-name --type x /bin/<command>
```

- `nixos-option` falls back to `/etc/nixos`; when the configuration lives
  elsewhere, pass `-F <flake>#<host>`. `Declared by` and `Defined by` in its
  output point at the module files.
- `nixos-option` describes the option; `nix eval` shows what this build will
  actually use. Use both when a value looks wrong.
- `-r` can fail on an option subtree whose values its renderer cannot print;
  fall back to a specific leaf or `nix eval`.
- Home Manager options are a separate tree; integrated options appear under
  `home-manager.users.<user>` and work with `nixos-option -F ...`.
- A flake copies only tracked files from a git-backed tree. If a new module
  seems invisible, check that the file is tracked/staged; in jj, run
  `jj status` to snapshot it, and `git add <path>` if the tool reads the git
  index.
- `nix search nixpkgs` follows the registry, which may not be this flake's
  pinned nixpkgs. For a forked or pinned package set, inspect the flake's
  `legacyPackages` / `packages` outputs with `nix repl` instead.
- `nix repl` is the interactive escape: `:lf /path/to/flake`, then inspect
  `nixosConfigurations.<host>.config` or `legacyPackages.<system>`.
- If the session exposes an MCP server such as `mcp-nixos` (package and option
  discovery; sometimes called nixos-mcp) or `nix-agent` (build, diff, switch,
  eval against this flake), use it for the covered question. `mcp-nixos` is
  good at discovery (NixOS/Home Manager options and packages, the wiki,
  nix.dev); it answers what exists upstream, while `nix eval` answers what
  this machine resolves. If neither is callable, the local `nix` commands are
  the fallback. Never invent an option or package name to fill a gap.

## The `nix` CLI

`nix` is the package, store, and flake CLI. It builds and inspects; it does
not activate a NixOS system. The system lifecycle lives in `nixos-rebuild` and
`nixos-option`; the user environment lives in `home-manager` for a standalone
flake, or inside the system for integrated Home Manager.

Most commands take an installable: `nixpkgs#hello` (registry), `.#attr`
(current flake), `github:owner/repo#attr`, or a store path. `--` separates the
installable from the program's arguments in `nix run`.

For one-off tools and apps, use the `nix-temporary-tools` skill; the short
rule is `nix shell nixpkgs#<pkg> -c <cmd>` or
`nix run nixpkgs#<app> -- <args>`. Neither changes the system or profile.

For the command map, state-changing commands, legacy equivalents, and gotchas,
read `references/nix-cli.md`.

## Translate the request

| Traditional action | NixOS move |
| --- | --- |
| `apt install foo` / `dnf install foo` | `environment.systemPackages = [ pkgs.foo ];`; `home.packages = [ pkgs.foo ];` for a user; `nix shell nixpkgs#foo` for one command |
| Edit `/etc/foo.conf` | The service's `settings` / `extraConfig` option; otherwise `environment.etc."foo.conf".text = "...";` |
| `systemctl enable foo` | `services.foo.enable = true;` or `systemd.services.foo.wantedBy = [ "multi-user.target" ];` |
| `systemctl --user enable foo` | `systemd.user.services.foo.wantedBy = [ "default.target" ];` (usually Home Manager) |
| `systemctl edit foo` | `systemd.services.foo.serviceConfig = { ... };` — the unit is generated |
| `crontab -e` | `systemd.timers.<name>` plus `systemd.services.<name>`, or `services.cron.systemCronJobs` |
| `/etc/fstab` | `fileSystems."/mnt/data" = { device = "..."; fsType = "..."; };` |
| Edit or symlink `~/.config/foo` | `home.file.".config/foo".source/text` or `xdg.configFile."foo".source` in Home Manager |
| `iptables` / `ufw` | `networking.firewall.*` or `networking.nftables.*` |
| `useradd` / `passwd` | `users.users.<name> = { isNormalUser = true; ... };` and a hash/file source |
| `hostnamectl set-hostname` | `networking.hostName = "...";` |
| `timedatectl set-timezone` | `time.timeZone = "...";` |
| `locale-gen` | `i18n.defaultLocale` / `i18n.extraLocaleSettings` |
| `modprobe foo` at boot | `boot.kernelModules = [ "foo" ];` |
| `update-grub` | `boot.loader.*`; activation regenerates boot entries |
| Edit `/etc/environment` | `environment.variables` / `environment.sessionVariables`; `home.sessionVariables` for one user |
| `pip install` / `npm -g` system-wide | Package it, or use `nix shell`; a global imperative install is not part of the system |
| A script with `#!/bin/bash` | `#!/usr/bin/env bash`, or `patchShebangs` in the derivation |
| A prebuilt binary needing `/usr/lib` | `programs.nix-ld`, `buildFHSEnv`, or `patchelf`; NixOS has no FHS |
| `docker run ...` | `virtualisation.oci-containers` (declarative) or a NixOS service module |

The pattern is constant: find the **option** that owns the concern, set it,
rebuild. An imperative command is a diagnostic tool, not the persistence
mechanism.

If an action is not listed here, find the module option that owns the
concern; the pattern still applies.

`environment.systemPackages` lands in the system profile
(`/run/current-system/sw`) and is available to every user. `home.packages`
lands in the user profile and only that user's `PATH`. Seeing
`/nix/store/.../bin/foo` in `which foo` is normal; executables are exposed
through profiles, not copied into `/usr/bin`.

## Module semantics

A configuration is a set of modules. Each option has a type; every module
defining it contributes, and the type merges the contributions.
Last-write-wins is not the model.

- `imports = [ ./foo.nix ];` adds a module.
- `lib.mkDefault value` lowers priority; `lib.mkForce value` raises it.
- `lib.mkIf condition value` contributes only when true.
- `lib.mkMerge [ ... ]` merges a list of attribute sets.
- `lib.mkBefore` / `lib.mkAfter` order list or string contributions.
- `options.<name> = lib.mkOption { type = ...; default = ...; };` declares an
  option; `config` reads the merged result.

List options concatenate, attribute-set options merge, and scalar options
conflict unless one definition has higher priority (`mkDefault` / `mkForce`)
or the type defines a merge function.

Home Manager has two shapes: **integrated**
(`home-manager.users.<user>` inside the NixOS configuration; a system switch
activates it) and **standalone** (`homeConfigurations`; applied with
`home-manager switch`). Identify the shape before choosing a command.
Integrated HM rolls back with the system generation; standalone HM has its
own generations (`home-manager generations`,
`home-manager switch --rollback`).

When definitions conflict, evaluation fails and names both files. Read that
as "these modules disagree", then decide which definition is stale. `mkForce`
is for a real priority decision, not for hiding a conflict you have not
understood.

An **overlay** changes packages, not options:

```nix
nixpkgs.overlays = [
  (final: prev: {
    foo = prev.foo.overrideAttrs (old: { ... });
  })
];
```

## Build and activation are separate

1. **Evaluate** catches type errors, missing options, and recursion.
2. **Build** realises the closure in `/nix/store`; no root.
3. **Activate** runs the activation script and `switch-to-configuration`: it
   writes generated files and stops/starts units; `nixos-rebuild` also
   updates the boot entry. Root.

```bash
nix flake check --no-build
nixos-rebuild dry-build --flake .#host
nixos-rebuild build --flake .#host
nixos-rebuild dry-activate --flake .#host
nixos-rebuild test --flake .#host     # activate now, no boot entry
nixos-rebuild switch --flake .#host   # activate now + boot entry
nixos-rebuild boot --flake .#host     # boot entry only, no activation
```

`test` is the safe choice for a risky service change on a machine you can
reboot; `switch` is the normal path; `boot` stages the change for the next
boot. A failed build never touches the running system. A failed activation
generally leaves the previous generation as the default, but a partially
applied activation can leave units inconsistent — verify, then roll back if
needed.

A kernel, initrd, or bootloader change only takes effect on reboot. `switch`
stages it; it cannot replace the running kernel.

`nixos-rebuild switch` uses `flake.lock`; it does not update inputs.
`nix flake update` moves pins; the next rebuild may then rebuild much of the
closure, so update one input at a time when that is the intent. For read-only
`nix eval` or `nix flake` commands, pass `--no-write-lock-file`. Identify
whether the machine is flake-based or channel-based before reaching for
`nix-channel`; do not mix the two models.

Activation can restart the service carrying your session (SSH, a display
manager). Prefer `test` or `boot`, or a maintenance window, for those changes.

## Verify and roll back

```bash
systemctl --failed
systemctl status <unit>
journalctl -u <unit> -b --no-pager
systemctl cat <unit>          # shows the store path behind the unit

nix store diff-closures /run/current-system ./result
nixos-rebuild list-generations
nix-env -p /nix/var/nix/profiles/system --list-generations
```

`nix-env` against the system profile may need root even to list generations;
build and evaluation do not.

Rollback options:

- reboot and select the previous generation in the boot menu;
- `nixos-rebuild switch --rollback` to activate the previous generation now;
- `nix-env -p /nix/var/nix/profiles/system --set <store-path>` followed by
  `<store-path>/bin/switch-to-configuration switch` only when you have a
  recorded target.

`nix-env --rollback` rolls back a user profile, not the system. The system
profile is `/nix/var/nix/profiles/system`.

Old generations are the rollback path. `nix-collect-garbage -d` deletes them;
do not run it casually on a machine you may need to restore.

## State, secrets, and the edge of the model

- `/nix/store` is immutable and shared. Never edit a store path; rebuild the
  expression. Do not hardcode store paths in scripts or config — refer to
  `pkgs.foo` or `${pkgs.foo}/bin/foo`.
- Managed `/etc` entries are replaced on activation. Unmanaged files may
  persist, but they are not reproducible; encode the change in a module.
- `/var/lib`, `/home`, databases, and application data are mutable state.
  The module can create the service and its directories; it does not migrate
  the data. Back up and plan the migration before an upgrade that changes a
  schema.
- Secrets belong in sops-nix/agenix or a hashed-password file, not in the Nix
  store or the repo. Reference the secret; never paste the value into a
  module.
- `users.mutableUsers = true` (the default) lets `passwd` work, but the result
  is mutable state. Set it to `false` when users must be fully declarative.
- Persist a tool for a user in `home.packages`, or for the system in
  `environment.systemPackages`; imperative profile commands are not the
  persistence path on a managed system.

## Debugging map

- **Evaluation error**: read the file:line; check option types, missing
  `imports`, and recursive `config` references. Add `--show-trace` only when
  the default trace is insufficient.
- **Build error**: `nix log <drv>` or `nix build .#... -L`; the failing
  derivation is named.
- **Activation error**: read `switch-to-configuration` output, then
  `systemctl status` and `journalctl`. A unit that fails after activation is
  a runtime problem, not necessarily a build problem.
- **A unit does not match its documentation**: `systemctl cat <unit>` and
  follow the store path back to the module; edit the option, not the unit.
- **Closure unexpectedly large**: `nix why-depends /run/current-system
  /nix/store/<path>`; find what pulls the dependency in.
- **Unsure what sets an option**: `nixos-option -F ... -r <prefix>`;
  `Declared by` and `Defined by` name the files.

## On an unfamiliar machine

1. `nixos-version`; `readlink -f /run/current-system`.
2. Find the config source: `/etc/nixos/flake.nix`,
   `/etc/nixos/configuration.nix`, or the path the deployment tooling uses.
   Do not assume `/etc/nixos`; a flake-managed machine may keep it elsewhere,
   and `/etc/nixos/configuration.nix` may be a symlink into the store.
3. If it is a flake, `nix flake metadata` in that directory and note the host
   attribute; if it is channel-based, inspect `nix-channel --list` and
   `NIX_PATH`.
4. Read the repository's own README/AGENTS and deployment wrapper before
   rebuilding; they may add gates this skill does not know about.
5. Change one option, build, then activate.

## Repository gates

This skill describes the NixOS model, not a deployment policy. When a
repository wraps rebuilds (a Justfile, a deploy script, an `AGENTS.md`), those
gates win: they may pin the host, require a dry-run, take a snapshot, or leave
activation to a human. A wrapper's build phase can succeed while activation
is blocked by a sandbox that forbids setuid; treat that as an environment
limit, not a configuration failure.

## Sources

- NixOS manual: <https://nixos.org/manual/nixos/stable/>
- Option and package search: <https://search.nixos.org>
- NixOS Wiki: <https://wiki.nixos.org>
- `mcp-nixos`: <https://github.com/utensils/mcp-nixos>
- `nix-agent`: <https://github.com/JEFF7712/nix-agent>

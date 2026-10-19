---
name: jj-workspaces
description: Use when work needs a second working copy in a Jujutsu repository — parallel agents, a long build that must not block edits, or a dirty default workspace — and when cleaning up the ones that accumulate. Covers `jj workspace add`/`forget`, why `@` collides when two processes share a workspace, what leftover workspaces cost, and the git plumbing to use when `jj rebase` refuses to rewrite an immutable commit.
---

# Jj Workspaces

## Overview

A jj workspace is a second checkout of the same repository, with its own `@` but a
shared object store and operation log. Use one whenever two pieces of work must not
share a working copy; do not use one as a substitute for `jj new`.

Use `jj-atomic-workflow` for change boundaries inside a workspace and `jj-local-help`
for exact command syntax.

## When a Workspace Is the Right Tool

- A second agent or session will edit the same repository while you work.
- A build or test run holds files still while you need to keep editing.
- You must check out an old revision to reproduce something without moving `@`.
- The default workspace is dirty with someone else's unfinished change.

A workspace is **not** needed to start a new unit of work in the same checkout;
that is `jj new`.

## The Collision This Prevents

`@` is per workspace, but the default workspace is shared by every process that runs
in the repository root. Two agents there do not get two commits: each `jj` command
snapshots the working copy into the same `@`, so one agent's files land in the other
agent's description.

Observed failure: an agent ran `jj new` and `jj describe` for a package bump while a
second agent's files were still in the working copy; the result was a commit whose
message described one package and whose tree contained another. Nothing was lost, but
the commit was unusable and had to be rebuilt.

If it happens, read the operation log — it records the user, host, and workspace:

```bash
jj op log --no-graph --limit 10
jj op show <op-id>
```

## Creating

```bash
jj workspace add ../fix-<topic> -r 'pkg/<name>@origin'
```

- `-r` accepts anything: a bookmark, a remote bookmark, `@-`, a commit id.
- Each workspace is a **full checkout**. A nixpkgs tree is ~55k files and ~336 MB;
  create it for the task and remove it when the task ends.
- Name it after the task, not after the agent, and keep it outside the repository
  directory so it cannot be committed by accident.
- A workspace's `@` is a normal commit: push from inside it with `git push <sha>:<ref>`
  or `jj git push -b <bookmark>`.

## Finishing and Cleaning Up

```bash
# from the main workspace
jj workspace forget fix-<topic>
rm -rf ../fix-<topic>
git worktree prune
```

Leftover workspaces are not free: they hold a full checkout each, appear in
`jj workspace list`, and their `@` keeps a commit alive that a revset may pick up.
Check for them before reporting a task done:

```bash
jj workspace list
```

## When `jj rebase` Refuses

Remote bookmarks are immutable, so rebasing a commit that descends from one fails:

```
Hint: This operation would rewrite 10 or more immutable commits.
```

Three ways out, in order of preference:

1. **Do not rewrite it.** Start from the intended base with `jj new <base>` and re-apply
   the change; a fresh commit on the right parent is almost always cheaper.
2. **Build the commit with git plumbing** when the tree already exists and only the
   parent must change. This is the documented recovery for a branch based on the wrong
   commit:

   ```bash
   export GIT_INDEX_FILE=/tmp/idx-rebuild
   rm -f "$GIT_INDEX_FILE"
   git read-tree <new-base>
   blob=$(git rev-parse <old-commit>:<path>)
   git update-index --add --cacheinfo "100644,$blob,<path>"
   tree=$(git write-tree)
   new=$(git commit-tree "$tree" -p <new-base> -F message.txt)
   git push --force-with-lease=<ref>:<old-sha> <url> "$new:<ref>"
   ```

   Verify before pushing: `git diff --stat <new-base> $new` must show exactly the
   intended files, and `git merge-base $new <base>` must equal `<base>`.
3. **Relax immutability** only if rewriting the bookmark is genuinely intended:
   `jj config set --repo revsets.immutable-heads 'tags()'`. This makes every remote
   bookmark rewritable locally, so a mistyped `jj edit` can now rewrite shared history.

## New Files and Nix Flakes

A file created in a workspace is invisible to `nix flake` until it reaches the git
index; the build fails with `add it to git`. Force a snapshot first:

```bash
jj status            # snapshots the working copy
git add <path>       # if a tool reads the index directly
```

## Reclaiming Space

`jj util gc` is the safe collector: it respects the operation log and only prunes what
is already unreachable under the configured expiry (two weeks by default). On a ~5 GB
nixpkgs clone it took ~7 minutes and reduced `.git` from 4.5 GB to 3.2 GB by packing
loose objects.

Do not reach for `git gc --prune=now` in a jj repository: commits that only the
operation log references are not reachable from a git ref, and pruning them breaks
`jj op restore` and `jj op show`.

Stray remote refs outlive the remote that created them. If `jj bookmark list` shows a
`name@remote` for a remote that no longer exists, delete the ref directly:

```bash
git for-each-ref refs/remotes
git update-ref -d refs/remotes/<remote>/<branch>
```

## Keeping Commands Fast in Large Repositories

- Narrow the default log: `jj config set --repo revsets.log 'ancestors(@, 4) | master@origin | remote_bookmarks(glob:"pkg/*")'`.
- `name@remote` only works for one concrete name; use `remote_bookmarks(glob:'...')` for a pattern.
- Restrict what a fetch imports: `jj config set --repo remotes.upstream.fetch-bookmarks nixos-unstable`
  and list the default remotes in `git.fetch`. A plain fetch of a large upstream otherwise
  imports hundreds of branches.

## Checklist

- [ ] Is another process going to touch this repository? If yes, use a workspace.
- [ ] Did you create the workspace at the right base revision?
- [ ] Is the workspace outside the repository directory?
- [ ] Before pushing, does the diff contain only the intended files?
- [ ] After finishing, did you `jj workspace forget`, delete the directory, and prune?
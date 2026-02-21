# jj Core Concepts (Stable Notes)

Use this file only for stable mental models. For options, flags, and exact behavior, query live help.

## Help-system-first navigation

### What it is
- `jj` has a built-in command help system and keyword topic help.

### Why it matters
- It is the most reliable source for the currently installed version.

### Common pitfalls
- Skipping `jj help help` and missing how `-k/--keyword` works.

### Validate with commands
- `jj help`
- `jj help help`
- `jj help -k glossary`

## Working copy as commit (`@`)

### What it is
- The working copy is represented by a real commit (`@`) instead of a staging area.

### Why it matters
- Editing and history operations become part of normal daily workflow.

### Common pitfalls
- Expecting Git-style staged/unstaged flow and misreading what `@` represents.

### Validate with commands
- `jj status`
- `jj log -r @`
- `jj show @`

## Change vs commit evolution

### What it is
- A logical change can evolve through multiple rewritten commits over time.

### Why it matters
- Rewrite operations preserve intent while refining history.

### Common pitfalls
- Assuming rewritten commits are always equivalent to brand-new unrelated work.

### Validate with commands
- `jj evolog`
- `jj log`

## Rewrite-first workflow

### What it is
- Commands like `new`, `edit`, `describe`, `squash`, `split`, and `rebase` are primary tools.

### Why it matters
- Iterative cleanup before publish is a core workflow, not an edge case.

### Common pitfalls
- Treating history rewrite as exceptional and avoiding essential cleanup.

### Validate with commands
- `jj new --help`
- `jj rebase --help`
- `jj squash --help`
- `jj split --help`

## Operation log and recovery

### What it is
- `jj` tracks operations and supports moving backward/forward through operation history.

### Why it matters
- Many mistakes are recoverable with operation-level undo/redo.

### Common pitfalls
- Trying to recover only via commit-level reasoning and ignoring operation history.

### Validate with commands
- `jj operation log`
- `jj undo --help`
- `jj redo --help`

## Bookmarks vs Git branches

### What it is
- Bookmarks are movable names for revisions and serve branch-like collaboration roles.

### Why it matters
- Remote push/fetch workflow in `jj` is centered around bookmarks.

### Common pitfalls
- Assuming one-to-one mental mapping with every Git branch workflow detail.

### Validate with commands
- `jj bookmark list`
- `jj git push --help`
- `jj git fetch --help`

## Revsets and selection language

### What it is
- Revsets are expressions for selecting revision sets.

### Why it matters
- Most history commands depend on revset selection quality.

### Common pitfalls
- Using command defaults blindly instead of precise revset filters.

### Validate with commands
- `jj help -k revsets`
- `jj log -r ::`

## Templates and output customization

### What it is
- Templates control how command output is rendered.

### Why it matters
- Better output improves review and debugging workflows.

### Common pitfalls
- Editing templates without checking built-in keywords and template functions first.

### Validate with commands
- `jj help -k templates`
- `jj log -T`

## Filesets and path selection language

### What it is
- Filesets are expressions for selecting sets of files/paths.

### Why it matters
- Path-based filtering is key for targeted operations and queries.

### Common pitfalls
- Treating filesets as simple globbing and missing expression semantics.

### Validate with commands
- `jj help -k filesets`
- `jj file list --help`

## Immutable revisions and policy constraints

### What it is
- Some revisions are configured as immutable and protected from rewrite by default.

### Why it matters
- Rewrite commands can fail due to policy, not syntax.

### Common pitfalls
- Debugging as a command failure without checking immutable policy/config.

### Validate with commands
- `jj --help`
- `jj config list --help`
- `jj rebase --help`

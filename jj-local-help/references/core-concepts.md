# jj Core Concepts (Stable Notes)

Use this file for stable mental models only. For exact flags, option names,
keyword topics, and installed-version behavior, query live local help.

## How to use this file

- Read this early when a question depends on `jj` mental models instead of only
  command syntax.
- Use it to explain concepts, connect related ideas, and correct Git-shaped
  assumptions.
- If this file and live help appear to disagree, trust live help and mention
  that these notes may lag behind the installed version.

## Concept map

`jj` treats your working copy as a real revision (`@`), not as a staging area.
A logical change can be rewritten through many commits over time. Bookmarks are
movable names for revisions used for collaboration. The operation log tracks the
history of repository operations, which makes many mistakes recoverable. Revsets
and filesets are the query languages you use to select revisions and paths.

## Help-system-first navigation

### What it is

`jj` ships with built-in command help and keyword help.

### Git-shaped assumption to drop

Do not assume that external docs or memory are more accurate than local help for
the installed version.

### Why it matters in daily use

Many `jj` questions are really version or syntax questions. Local help answers
those more reliably than a static skill can.

### Common misread

People often run `jj help <thing>` without first learning how keyword help works
and then miss useful topics behind `jj help -k <topic>`.

### Smallest useful commands

- `jj help`
- `jj help help`
- `jj help -k glossary`

## Working copy as commit (`@`)

### What it is

Your working copy lives in a real revision named `@`. It is not a Git-style
index plus working tree split.

### Git-shaped assumption to drop

Do not look for a staged versus unstaged boundary by default. In `jj`, the
working state itself is already represented in history.

### Why it matters in daily use

Commands like `desc`, `new`, `split`, `show`, and `log` all make more sense
once you understand that `@` is a revision you are actively editing.

### Common misread

People often treat `@` as "my dirty workspace" instead of "the current working
copy commit". That leads to confusion about why history-editing commands feel so
normal in `jj`.

### Smallest useful commands

- `jj status`
- `jj log -r @`
- `jj show @`

## Change vs commit

### What it is

A change is the logical unit of intent. A commit is one concrete revision in the
history of that change. The same change can be rewritten into new commits over
time.

### Git-shaped assumption to drop

Do not assume that a new commit ID always means brand-new unrelated work. In
`jj`, rewrites often preserve the same logical change while refining its shape.

### Why it matters in daily use

This is why `jj` workflows can keep improving a change's description, contents,
or parentage without treating every rewrite as a separate conceptual task.

### Common misread

People often collapse "change" and "commit" into one concept and then get lost
when rebasing, splitting, or rewriting produces new commit IDs.

### Smallest useful commands

- `jj log`
- `jj evolog`

## Operation log and recovery

### What it is

`jj` records operations as well as revisions. This gives you a second recovery
layer above ordinary commit history.

### Git-shaped assumption to drop

Do not reason only at the commit level. A bad rebase, undoable rewrite, or
accidental history edit may be easiest to recover from through operation
history, not by manually reconstructing commits.

### Why it matters in daily use

This makes `jj` much more forgiving during history editing. Recovery is often a
normal workflow step rather than a panic path.

### Common misread

People sometimes search the visible revision graph for lost work when the faster
answer is in the operation log.

### Smallest useful commands

- `jj operation log`
- `jj undo --help`
- `jj redo --help`

## Bookmarks vs Git branches

### What it is

Bookmarks are movable names attached to revisions. They fill the branch-like
role in collaboration and remote synchronization.

### Git-shaped assumption to drop

Do not expect every Git branch habit to transfer directly. A bookmark is not
just a renamed Git branch concept.

### Why it matters in daily use

Push and fetch workflows in `jj` revolve around bookmarks, so understanding them
is essential for collaboration and remote state tracking.

### Common misread

People often assume bookmarks are a cosmetic alias over Git branches and then
miss how `jj git push` and remote bookmark state are modeled.

### Smallest useful commands

- `jj bookmark list`
- `jj help -k bookmarks`
- `jj git push --help`
- `jj git fetch --help`

## Immutable revisions and policy constraints

### What it is

Some revisions are protected from rewrite by repository policy or configuration.

### Git-shaped assumption to drop

Do not assume a failed rewrite means the command syntax was wrong. The command
may be fine while the target revision is intentionally protected.

### Why it matters in daily use

When `rebase`, `squash`, or similar commands fail, policy constraints can be the
real explanation. That changes troubleshooting from "what flag did I miss?" to
"what is allowed to move here?"

### Common misread

People debug immutable-revision failures as command mistakes and never inspect
the repository's rewrite policy.

### Smallest useful commands

- `jj config list --help`
- `jj rebase --help`

## Revsets and selection language

### What it is

Revsets are expressions for selecting revisions.

### Why it matters

Many history commands become safer and clearer when you choose revisions
precisely instead of relying on broad defaults.

### Common misread

Treating revsets like ad hoc shell filters instead of a query language.

### Smallest useful commands

- `jj help -k revsets`
- `jj log -r '::@'`

## Filesets and path selection language

### What it is

Filesets are expressions for selecting files and paths.

### Why it matters

They enable precise path-based filtering without reducing everything to simple
glob syntax.

### Common misread

Expecting plain globbing semantics and missing fileset expression rules.

### Smallest useful commands

- `jj help -k filesets`
- `jj file list --help`

## Templates and output customization

### What it is

Templates control how command output is rendered.

### Why it matters

A better template often makes review and debugging faster than post-processing
default output.

### Common misread

Editing templates before checking built-in keywords and template functions.
Running bare `jj log -T` can print a useful alias hint in current `jj`, but it
does so after reporting that the required template argument is missing.

### Smallest useful commands

- `jj help -k templates`
- `jj log -T builtin_log_oneline`

---
name: sync-debt
description: Maintain a shrinking ledger of places that must change together but fail silently, then convert each entry into an automated check. Use when designing or evolving a multi-part system (schema ↔ code, prompt ↔ template, docs ↔ config, two implementations of one rule, serialized state), when the user asks what must change together, or when drift should be caught without human review.
---

# Sync Debt

A **sync point** is a place where changing one artifact requires changing another
and nothing errors when you forget. The build passes, the job runs, the numbers
are just quietly wrong. These are the expensive bugs in a system with many
parts, and they are invisible in proportion to how well everything else works.

Two artifacts, in this order:

1. A **ledger** of the sync points still depending on human memory.
2. **Checks** that take entries off the ledger.

The ledger is debt, not documentation. Its length is the count of unguarded
couplings, so **shorter is strictly better**. A ledger nobody shortens is a
wishlist.

## Hunt

Five patterns produce most sync points. Walk them explicitly instead of waiting
for a bug:

| Pattern | Ask |
| --- | --- |
| Producer / consumer | Which type, schema, field list, or file format is defined here and parsed there? |
| One rule, two implementations | Where do TS and Python, server and client, or real and fake each re-derive the same decision? `parity` tests usually cover only the easy half. |
| Chained invariant | Which hash, id, signature, or version threads through several stages, so breaking one link leaves the rest stale but still valid-looking? |
| One fact, several statements | Which design doc, roadmap, diagram, and deck restate the same conclusion? Only conclusions must agree; details may differ by audience. |
| Serialized state | Which state files, caches, journals, or manifests have a format that readers assume? Readers of a drifted format see empty values, not errors. |

For each candidate, write down who produces it, who consumes it, and what the
drift looks like from outside: which number is wrong, which write silently
no-ops, which page shows a blank.

## The ledger

One table. Columns: **sync point | drift symptom | current guard | replacement
check | what blocks the check**.

- An entry exists only while no automated check covers it. On automation, delete
  the row and keep a one-line index under "automated". The index is the audit
  trail; the table is the work.
- Every entry must name the check that will replace it. Failure to name one
  means the coupling is not understood yet, not that it is unautomatable.
- **Say what blocks the check.** Usually it is not scripting effort but a
  missing single source of truth: a field list, an exported schema, one
  designated writer. Name the artifact that would unblock it, so entry becomes
  a concrete prerequisite instead of a permanent excuse.
- Human-only entries sit outside the table: name who, at which step, how often.
  Shrink them by removing duplication (define once, reference elsewhere), not by
  asking for more review.

## Turning an entry into a check

Prefer the check that can be written today, even if it covers one direction.

1. **Anchor on machine-readable data**: a type declaration, a fenced JSON
   example, an exported schema, a generated table dump.
2. **Compare names, keys, or sets — never prose.** Scanning backticked
   identifiers in a document needs an allowlist, and the allowlist becomes its
   own sync point.
3. **Go one-directional when the consumer may lag**: flag names that no longer
   exist, not names that have not been added yet.
4. **Run it before fixing the drift.** A check that has never failed is
   unverified; expect the first run to catch something real.
5. **Keep it tens of lines, in the existing test suite**, next to the code it
   guards. If it needs configuration or a new runner, it is not a check yet.
6. **Fix what it found, then delete the row** and add the index line.

## Anti-patterns

- The ledger grows with every design conversation. A new row must either remove
  another row or name the artifact that unblocks it.
- A check derives its expectation from the implementation, so it can never
  disagree with it.
- A check compares two hand-maintained copies that drift together.
- The same rule is documented in three places "for safety". That is the sync
  point, not the guard.
- An automated entry stays in the main table "for visibility". The index is the
  visibility.

## Example

A skill's JSON example versus the plan type it claims to describe. Anchor: the
fenced JSON block. Check: parse it, collect keys, assert each exists in the
type. The first run failed on `rows`, `evidence`, `unresolved`, and
`line_notes` — the example had drifted from the real shape for weeks while
reading perfectly plausible to every human who opened it.

---
name: hsi-trim
description: Trim the codebase — find repetition, missing abstractions, and dead code, propose simplifications with their payoff, and apply approved ones without changing behavior.
---

Invoke the `hybrid-spec-intent` skill in trim mode.

This command reads across the whole codebase by design: repetition and dead code are only visible
from everywhere at once. It is one of the two sanctioned exceptions to the skill's context budget,
and it runs only when the user invokes it.

## Phase 1 — Find candidates

- **Repetition** — identical error handling, duplicated parsing or serialization, copy-pasted
  structural layouts that differ only in a value or two.
- **Copy-paste features** — separate near-identical implementations of one behavior that could
  become a single data-driven engine reading from a config model or table.
- **Missing seams** — logic that cannot be tested in isolation because it is welded to I/O, the
  UI layer, or a global.
- **Dead code** — orphaned modules, unreferenced exports, functions and variables never called.
- **Unused imports and dependencies** — imports nothing uses; manifest dependencies nothing
  imports.

**Dead code that carries a `@spec` annotation is not dead weight.** It means a requirement's
implementation is unreachable: either the requirement is no longer wanted — a REMOVED entry in a
delta, which is DELTA work — or something that should call it no longer does, which is a bug.
Report these separately and never delete them as a trim.

## Phase 2 — Proposal

Present, for each candidate:

1. **Target** — what is redundant or unused, with file:line references.
2. **Vision** — the shape that replaces it, or confirmation that it simply goes.
3. **Payoff** — estimated lines removed, dependencies dropped, and whether it opens a testing
   seam.
4. **Classification** — the spec tier and code radius this change would carry.

That last point matters. A change that preserves behavior is a **PATCH** no matter how many files
it touches — no delta spec is needed. Its code radius is what determines the real cost: CONTAINED
is nearly free, WIDE means Gate 2 stops and a full back-port follows.

If a proposal would change any observable behavior, it is **not** a trim. Say so, classify it as
DELTA, and it takes the full flow with a delta spec.

## Phase 3 — Execution

Wait for explicit approval, then apply only the approved items.

- **Move `@spec` annotations with the entry points they mark.** When behavior relocates, its
  annotation relocates in the same edit. Leaving one on a now-hollow wrapper is the most common
  source of annotation rot, and the test suite cannot detect it — the change is
  behavior-preserving, so everything stays green while the linkage silently breaks.
- **Before deleting "unused" code, rule out references a static search misses** — reflection,
  string-keyed lookups, route tables, serialization names, platform entry points. When in doubt,
  report it rather than delete it.
- Run the full suite using the command declared in the map; after removing a dependency, run the
  build too. A trim that changes test results changed behavior, which means it was misclassified —
  stop and reclassify.
- Update the map only if a capability boundary or a declared contract actually moved. Trims
  usually touch neither.
- Append a ledger line.

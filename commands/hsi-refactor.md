---
name: hsi-refactor
description: Find repetition and missing abstractions, propose a simplification with its payoff, and apply it through the tier system once approved.
---

Invoke the `hybrid-spec-intent` skill in refactor mode.

## Phase 1 — Pattern recognition

Scan for:

- **Repetition** — identical error handling, duplicated parsing or serialization, copy-pasted
  structural layouts that differ only in a value or two.
- **Copy-paste features** — separate near-identical implementations of one behavior that could
  become a single data-driven engine reading from a config model or table.
- **Missing seams** — logic that cannot be tested in isolation because it is welded to I/O, the
  UI layer, or a global.

## Phase 2 — Proposal

Present, for each candidate:

1. **Target** — what is redundant, with file:line references.
2. **Vision** — the shape that replaces it.
3. **Payoff** — estimated lines removed, and whether it opens a testing seam.
4. **Classification** — the spec tier and code radius this refactor would carry.

That last point matters. A refactor that preserves behavior is a **PATCH** no matter how many
files it touches — no delta spec is needed. Its code radius is what determines the real cost:
CONTAINED is nearly free, WIDE means Gate 2 stops and a full back-port follows.

If a proposed refactor would change any observable behavior, it is **not** a refactor. Say so,
classify it as DELTA, and it takes the full flow with a delta spec.

## Phase 3 — Execution

Wait for explicit approval, then apply only the approved items.

- **Move `@spec` annotations with the entry points they mark.** When behavior relocates, its
  annotation relocates in the same edit. Leaving one on a now-hollow wrapper is the single most
  common source of annotation rot, and the test suite cannot detect it — a refactor is
  behavior-preserving by definition, so everything stays green while the linkage silently breaks.
- Run the full suite using the command declared in the map. A refactor that changes test results
  changed behavior, which means it was misclassified — stop and reclassify.
- Update the map only if a capability boundary or a declared contract actually moved. Refactors
  usually touch neither.
- Append a ledger line.

---
name: hsi-reconcile
description: Apply completed change deltas into their capability specs, archive them, append the ledger, and run the coherence checks.
---

Invoke the `hybrid-spec-intent` skill in reconcile mode.

Operate on the change named in the argument, or on every change in `docs/hsi/changes/` that is
not yet archived. Process them one at a time, in the order they were started.

For each change, follow the skill's reconcile procedure:

1. **Refuse if `tasks.md` has unticked entries** — including shim resolution. Report which, and
   move on to the next change rather than stopping the batch.
2. **Refuse if the full suite is not green.** Run it using the suite command declared in the
   map's *Stack & Commands* section. Never hardcode a test command.
3. Apply ADDED / MODIFIED / REMOVED into the target capability spec.
4. Assign stable IDs to ADDED requirements, continuing that capability's sequence. Never reuse a
   retired ID.
5. Delete REMOVED requirements outright — git carries the history.
6. Move the change directory to `changes/archive/<YYYY-MM-DD>-<slug>/`.
7. Append one line to `changes/archive/index.md`.

Then run all six coherence checks and report the results. Checks 1–4 and 6 are structural and
soft-block; check 5 is advisory. Surface every failure plainly; the user may override any of
them.

Finish with a summary: which changes reconciled, which were refused and why, and any coherence
findings outstanding.

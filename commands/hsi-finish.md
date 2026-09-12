---
name: hsi-finish
description: Finish a DELTA or ARCHITECTURAL change — review the files it touched, merge its delta into the capability spec, archive it, and run the coherence checks.
---

Invoke the `hybrid-spec-intent` skill in finish mode. This runs the skill's **Finish** procedure:
a review scoped to the change, then reconcile.

## Pick what to finish

**With a slug:** look for `docs/hsi/changes/<slug>/`.

- If it exists, finish that change only.
- If it is already under `changes/archive/`, say so and stop — it was finished before.
- If it does not exist anywhere, say so and stop. **Do not guess a similar name, and do not fall
  back to finishing other changes.** List the open changes so the user can pick the right one.

**Without a slug:** do not start processing. List every open change in `docs/hsi/changes/`
(excluding `archive/`) with its tier and whether its `tasks.md` is fully ticked, then **ask which
to finish.** Finish only what the user picks. Running this command "for one change" must never
quietly finish unrelated ones.

**If there are no open changes,** say so and stop.

### Nothing to finish for PATCH changes

PATCH changes — refactors, renames, typo fixes, behavior-preserving tweaks — never create a
change directory, so there is nothing for this command to act on. Their ledger line is written by
the flow when the change completes. If the user asks to finish a refactor or other PATCH work,
explain this rather than searching for something to process.

## Procedure

Process the chosen changes one at a time, in the order they were started. For each, follow the
skill's Finish section exactly:

1. **Refuse if `tasks.md` has unticked entries** — including shim resolution and **preview
   approval**. Preview approval is ticked only on the user's explicit STOP#2 approval; passing
   tests do not count. Report which entries are open, and move on to the next chosen change
   rather than stopping the batch.
2. **Refuse if the full suite is not green.** Run it using the suite command declared in the
   map's *Stack & Commands* section. Never hardcode a test command.
3. **Change-scoped review** — only the files this change touched. Continue without stopping if
   it finds nothing; stop and present findings if it does.
4. **Reconcile** — merge the delta into the capability spec, assign IDs, archive, append the
   ledger line, run the coherence checks.

Finish with a summary: which changes finished, which were refused and why, review findings the
user chose to carry forward, and any coherence findings outstanding.

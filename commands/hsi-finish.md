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

## Finishing each change

Read `${CLAUDE_PLUGIN_ROOT}/skills/hybrid-spec-intent/references/finish.md` and follow it for every
chosen change — it holds the blockers, the change-scoped review, reconcile, and the coherence
checks. Do not work from memory or from this file.

What this command adds on top:

- Process the chosen changes **one at a time, in the order they were started**.
- When a change is refused — unticked tasks, a red suite — report why and **move on to the next
  chosen change** rather than stopping the batch.

Finish with a summary: which changes finished, which were refused and why, review findings the
user chose to carry forward, and any coherence findings outstanding.

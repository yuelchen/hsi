---
name: hsi-change
description: Start an HSI change with an explicit slug — classifies blast radius, then runs the gates and the flow its tier requires.
---

Invoke the `hybrid-spec-intent` skill in change mode. Argument: a short hyphenated slug
(`add-sso`, `expire-idle-sessions`). If none is given, propose one from the user's description
and confirm it.

Run the workflow from Gate 1, exactly as the skill defines it:

1. **Gate 1** — read only `docs/hsi/project_context_map.md` and assign the spec tier. State the
   tier and the reasoning in one line. Unclassifiable means ARCHITECTURAL.
2. **Gate 2** — contract pre-flight; assess code radius by finding call sites. Stop if anything
   breaks.
3. Follow the flow for that tier × radius combination.

Use this command when you want the workflow engaged deliberately — the skill also triggers on its
own for any prompt that could change code.

If `docs/hsi/project_context_map.md` does not exist, stop and tell the user to run `/hsi-setup`.

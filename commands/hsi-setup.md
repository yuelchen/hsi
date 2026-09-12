---
name: hsi-setup
description: Initialize HSI in this project — writes the capped context map and, for existing codebases, the capability index and skeleton capability specs.
---

Invoke the `hybrid-spec-intent` skill in setup mode.

## Already configured?

Check for `docs/hsi/project_context_map.md` **before doing anything else.**

If it exists, **stop. Do not rewrite, regenerate, or overwrite any file under `docs/hsi/`.** The
map and capability specs are curated by hand over time, and a second setup run would silently
replace that work with a fresh inference.

Tell the user the project is already set up and offer a **review** instead:

- Is the map still under its line cap?
- Does the Capability Index still match the code — any directories no capability claims, or
  capabilities whose path glob matches nothing?
- Do Stack & Commands still run?

Report the findings and change nothing unless the user picks specific fixes. If they genuinely
want to start over, they must say so explicitly; even then, confirm by naming the files that
will be replaced before touching them.

## Not configured

Create `docs/hsi/` with `project_context_map.md`, `capabilities/`, `changes/`, and
`changes/archive/index.md`. Use `references/map-template.md` for the map's structure and its
hard line cap.

**Detect which path applies** by whether the directory contains source beyond scaffolding.

## Greenfield

Ask the user for, in one round:

1. Language, framework, and version
2. State management, storage, and any other load-bearing architectural choices
3. Non-negotiable constraints (offline-first, single binary, regulatory limits)
4. Test suite command **and single-file test command**, lint command, build command

Write the map from their answers. Leave the Capability Index with only the capabilities they
actually named, and `Contracts` empty.

**Do not generate a speculative spec set.** Capability specs grow from deltas. Writing
requirements for features that do not exist yet produces specs that get redesigned before they
are ever built. Finish by telling the user to run `/hsi-change <slug>` for their first feature.

## Brownfield

1. Scan the directory structure, entry points, and dependency manifest. Do not read every file —
   read enough to identify boundaries.
2. Derive a **proposed** capability index: name, path glob, one-line purpose. Present it and ask
   the user to correct it before writing anything. Capability boundaries are the load-bearing
   decision in HSI and inferring them silently is how the map goes wrong.
3. On approval, write the map: architecture rules inferred from what the code actually does,
   commands read from the project's own config, contracts identified from cross-capability
   imports.
4. Write **skeleton** capability specs — one file per capability with its purpose and an empty
   `## Requirements` section. Do not reverse-engineer requirements from code; they would be
   descriptions of the implementation rather than statements of intent, and they would be wrong
   in ways nobody could see. Requirements accrue as changes touch each capability.
5. Report which capabilities have no spec content yet, so the user knows where the map is thin.

Verify the map is under its line cap before finishing. If it is not, promote detail into the
capability specs rather than raising the cap.

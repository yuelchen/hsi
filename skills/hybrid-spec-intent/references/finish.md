# Finish

Ends every DELTA and ARCHITECTURAL change, whether reached at the end of the flow or through
`/hsi-finish` in a later session. Runs after the full suite is green. Not optional — an
unreconciled delta means the capability spec is lying about current behavior.

1. Verify no `tasks.md` entry is unticked, **including shim resolution and preview approval**.
   Block if any are.
2. Run the **change-scoped review** (below).
3. **Reconcile:** apply the delta's ADDED / MODIFIED / REMOVED sections into
   `docs/hsi/capabilities/<cap>.md`.
4. Assign stable IDs to ADDED requirements, continuing the capability's sequence. Never reuse a
   retired ID.
5. Delete REMOVED requirements outright — git carries the history.
6. Move `changes/<slug>/` to `changes/archive/<YYYY-MM-DD>-<slug>/`.
7. Append one line to `changes/archive/index.md`.
8. Run the coherence checks.

## Change-scoped review

The judgment checks no grep can do, limited to **only the files this change touched**. Never widen
it to the rest of the codebase — the project-wide sweep is `/hsi-audit`.

**Touched files** are the union of:

- files changed in commits since `docs/hsi/changes/<slug>/delta.md` was first committed
  (`git log --diff-filter=A --format=%H -- <that path>` gives the starting commit)
- uncommitted changes in the working tree
- files named in the change's `tasks.md`

If the change directory was never committed, use the working tree plus `tasks.md`.

Within those files, check:

- **Hollow wrappers** — an annotated entry point that now only delegates.
- **Semantic staleness** — for each requirement the delta ADDED or MODIFIED, read the annotated
  code against its scenarios.
- **Unresolved shims** — anything marked TEMPORARY that this change introduced.
- **Security** — hardcoded secrets, unencrypted sensitive data, unsafe deserialization,
  injection-prone string-built queries.

**If nothing is found, continue to reconcile without stopping.** If anything is found, stop and
present the findings with file:line, most severe first. For each, the user chooses:

- **Fix now** — add a `tasks.md` entry, fix it within this change, re-run the suite, and review
  again.
- **Carry forward** — proceed; the finding is listed in the finish summary.

Never fix a finding automatically. A fix that changes behavior beyond what the delta describes is
an escalation: it needs a MODIFIED entry in this delta, or a change of its own.

## Coherence checks

**Structural — soft-block:**

1. All tests pass.
2. Every `@spec` annotation in changed files resolves to a requirement ID that exists.
3. Every ADDED or MODIFIED requirement has at least one **test** citing it.
4. No file — test **or source** — references a REMOVED ID.
5. *(advisory)* Every requirement has at least one **code** annotation. A requirement that is
   tested but whose implementation cannot be located is a finding, not a failure.
6. No requirement ID carries more than one non-test annotation. Catches copy-paste propagation
   and un-followed splits.

**Soft-block** means the change is not reported complete until these pass, and failures are
surfaced plainly. The user can override. HSI makes the cost visible; it is not a linter and not a
CI gate.

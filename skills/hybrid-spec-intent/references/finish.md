# Finish

Ends every DELTA and ARCHITECTURAL change, whether reached at the end of the flow or through
`/hsi-finish` in a later session. Runs after the full suite is green. Not optional — an
unreconciled delta means the capability spec is lying about current behavior.

1. Verify no `tasks.md` entry is unticked, **including shim resolution and preview approval**.
   Block if any are, naming each one — never tick an entry during finish, even when the work
   looks done. Preview approval is ticked only on the user's explicit STOP#2 approval;
   passing tests do not count.
2. Confirm the full suite is green, running it with the suite command declared in the map's
   *Stack & Commands* section — never a hardcoded command. Block if it is not.
3. Run the **change-scoped review** (below).
4. **Reconcile:** apply the delta's ADDED / MODIFIED / REMOVED sections into
   `docs/hsi/capabilities/<cap>.md`.
5. Confirm each ADDED requirement's reserved ID is still free — another change may have claimed it
   since this delta was drafted. If it was taken, take the next free ID and update every `@spec`
   tag that cites the old one. Never reuse a retired ID.
6. Delete REMOVED requirements outright — git carries the history.
7. Move `changes/<slug>/` to `changes/archive/<YYYY-MM-DD>-<slug>/`.
8. Append one row to the table in `changes/archive/index.md`:
   `| date | slug | tier | capabilities | summary |`, where the summary lists requirement IDs added
   (`+`), modified (`~`), and removed (`-`).
9. Run the coherence checks.

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

---
name: hybrid-spec-intent
description: Fail-fast spec-driven development workflow (HSI). Consult for ALL code changes. Classifies each change by blast radius and routes it to the cheapest safe path — trivial changes skip specs entirely, behavioral changes write a delta spec first, architectural changes stop for review. Enforces tests-before-code, breaking-change pre-flight, and @spec linkage between requirements, tests, and code.
---

# Hybrid Spec-Intent Development

HSI exists to hold two costs down at once: the **ceremony** a change must walk, and the
**context** the agent must load to walk it.

*Ceremony scales with blast radius. Context load scales with the surface touched — not with the
size of the project.*

It borrows OpenSpec's artifact economics (write a delta, not a spec set; reconcile and archive)
and LID's linkage (stable requirement IDs, `@spec` annotations, coherence verification). It drops
OpenSpec's unlinked specs and LID's stop-at-every-phase.

## Context budget — read this before reading anything else

The budget is the point of the workflow. Violating it defeats HSI even if every other rule is
followed.

| artifact | when it is loaded |
|---|---|
| `docs/hsi/project_context_map.md` | **always** — it is line-capped so this stays flat forever |
| `docs/hsi/capabilities/<cap>.md` | **only the capability being touched** |
| `docs/hsi/changes/<slug>/` | only while that change is in flight |
| `docs/hsi/changes/archive/` | **never by default** — grep only, when asked about history |

Do not read the whole `capabilities/` directory "for context." Do not read the archive to
understand current behavior — the capability spec *is* current behavior. If the map alone is not
enough to classify a change, that is a signal the change is ARCHITECTURAL, not a signal to read
more files.

**Two user-invoked commands are deliberate exceptions.** `/hsi-trim` and `/hsi-audit` read across
the whole project because their jobs cannot be done any other way — dead code, repetition, and
drift across capabilities are only visible from everywhere at once. They run only when the user
invokes them, never as part of a change. Nothing else widens the budget.

## Not configured yet?

If `docs/hsi/project_context_map.md` does not exist, this project has not been set up.

**In setup mode** (invoked by `/hsi-setup`), that is the expected state — create the artifacts.

**In every other mode**, tell the user to run `/hsi-setup` and stop. Do not improvise the map or
a capability spec mid-change: boundaries invented under pressure to finish a feature are the ones
that turn out wrong, and everything downstream inherits them.

## The three gates

Each gate sits at the cheapest point where the change can be shown wrong.

### Gate 1 — Classify

Read **only** the map. Assign a spec tier and provisionally note the likely code radius.

**A change that cannot be confidently classified is ARCHITECTURAL.** Fail toward ceremony.

### Gate 2 — Contract pre-flight

Before writing any code, determine whether the change alters:

- a public / exported symbol signature
- a persisted schema, migration, or stored format
- a wire format or API response shape
- a config key, environment variable, or CLI flag
- a contract another capability consumes (see the map's Contracts section)

If **none** apply, the code radius is CONTAINED — proceed.

If **any** apply, find the existing call sites, assess the code radius, and **STOP**:

```
⚠️  BREAKING CHANGE

  AuthService.login(String email, String password)
    → login(Credentials creds)

  Call sites: 3 files across 2 capabilities  →  code radius WIDE
    lib/ui/login_page.dart:42
    lib/ui/signup_page.dart:88
    lib/sync/background_auth.dart:17

  Proceed? Reply with confirmation, or say "shim" to isolate
  the new logic and defer the callers.
```

Wait for text confirmation. Do not proceed on silence, and do not route around the break on your
own initiative — see *Shims* below.

### Gate 3 — Red test

Write the test before the code, and **run it**. It must fail *for the expected reason*.

A test that passes before the implementation exists does not exercise the change. That is a
failure signal about the test, not a shortcut — report it and rewrite the test.

## Two axes

Ceremony is set by two independent measurements. Spec radius decides how much *specification*
work the change needs; code radius decides how much *code* work it triggers.

**Spec radius** — assigned at Gate 1, reading only the map:

| tier | trigger | delta.md | stops | reconcile |
|---|---|---|---|---|
| **PATCH** | no behavior asserted by any existing scenario changes, and at most one capability is touched | no | none | ledger line only |
| **DELTA** | new or changed behavior within one capability | yes | STOP#1, STOP#2 | into that capability |
| **ARCHITECTURAL** | crosses ≥2 capabilities, adds/removes/renames a capability, changes a consumed contract, changes stack or deployment shape, or changes anything asserted in the map | yes + map diff | STOP#1, STOP#2 | per capability |

**Code radius** — assessed at Gate 2, by finding call sites:

| radius | trigger | shims | back-port | full suite |
|---|---|---|---|---|
| **CONTAINED** | nothing existing breaks | n/a | not needed | at the end |
| **LOCAL** | breaks callers inside one capability | rarely justified | same pass | at the end |
| **WIDE** | breaks callers across capabilities | offered on approval | explicit stage | after back-port |

The axes are independent, and the divergent cases are the reason for two of them:

| change | spec radius | code radius |
|---|---|---|
| speed up an internal sort | PATCH | CONTAINED |
| rename a public method, no behavior change | PATCH | WIDE |
| add an optional parameter with a default | DELTA | CONTAINED |
| add session expiry | DELTA | LOCAL |
| change the auth token format | ARCHITECTURAL | WIDE |

Row 2 is why a single ladder does not work: a pure rename changes **no** specification, yet
breaks every caller. It needs no delta file and full back-port handling.

See `${CLAUDE_PLUGIN_ROOT}/skills/hybrid-spec-intent/references/tier-rules.md` for the full trigger list and worked examples.

## Escalation

**Escalation is free.** Discovering mid-flight that a PATCH touches a second capability makes it
DELTA immediately; re-enter at whichever gate the new tier requires. Say so in one line and
continue — no permission needed.

**De-escalation requires the user to say so explicitly.** Never lower a tier on your own
judgment. Without this rule every change drifts into PATCH and the workflow evaporates.

## Flows

### PATCH × CONTAINED
Gate 2 (passes) → red test if the change is behavior-bearing → code → run the suite → append
ledger line. No delta file, no stop.

### PATCH × LOCAL or WIDE
Gate 2 **STOP** → on approval, optionally shims → code → back-port callers → full suite →
ledger line. Still no delta file: nothing in the specification changed.

### DELTA
1. Write `docs/hsi/changes/<slug>/delta.md` and `tasks.md`.
2. **STOP#1** — present the delta. *Is this the right feature?* Cheap to redo here.
3. Gate 3: write tests into a **discrete, isolated test file** (e.g.
   `test/features/<slug>_test.dart`) and confirm they fail correctly.
4. Implement.
5. Iterate using the map's **single-file** test command — not the full suite. This is the
   fail-fast loop; keep it tight.
6. **STOP#2** — Feature Preview Approval. Present the working behavior and the diff. *Is this
   the right feel?* A spec cannot answer this question. Tick the `preview approved` entry in
   `tasks.md` **only** when the user explicitly approves — passing tests are not approval.
7. Back-port: expand context to the files with failing tests, fix them, resolve any shims.
8. Run the **full suite**. Everything green.
9. Finish: the change-scoped review, then reconcile (see *Finish*, below). `/hsi-finish` runs the
   same steps for a change picked up in a later session.

### ARCHITECTURAL
Write `delta.md`, a proposed map diff, and the list of affected capabilities → **STOP#1** → then
run the DELTA flow once per affected capability, pausing at each.

## Shims

A shim is glue that translates an old interface into a new implementation so existing callers
survive a contract change untouched.

**Never generate one on your own initiative.** Gate 2 stops first; shims exist only after the
user approves them. Silently routing around a break consumes the signal the gate exists to
deliver.

When approved, every shim gets a tracked pair of entries in `tasks.md`:

```markdown
- [ ] shim: LegacyAuthAdapter wraps AuthService.login for 3 legacy callers — TEMPORARY
- [ ] resolve shim: LegacyAuthAdapter — delete, or promote to a specified adapter
```

**Reconcile blocks while any shim is unresolved.** Resolution is one of two things:

- **Delete it** — the callers were fixed during back-port. The usual outcome.
- **Promote it** — the adapter is genuinely the right permanent answer (a public API version you
  cannot break, a deliberate anti-corruption layer at a module boundary). Then it stops being
  debt and becomes architecture: give it a requirement in the capability spec, annotate it with
  `@spec`, and delete the TEMPORARY marker.

The rule is *resolved*, not *removed* — a shim that earns its place is allowed to stay, but it
must be specified rather than merely tolerated.

## Requirements and scenarios

Capability specs hold requirements with **stable IDs** and GIVEN/WHEN/THEN scenarios:

```markdown
### AUTH-003 — Session expiry

#### Scenario: Idle session expires
- **GIVEN** an authenticated session with no remember-me flag
- **WHEN** the session has been idle for 30 minutes
- **THEN** the next request returns 401 and the session record is deleted
```

- **IDs are stable.** Revisions mutate text, not IDs. Deleted IDs are never reused.
- **One scenario, one behavior.** Multiple THEN bullets only when they are facets of a single
  outcome. A requirement bundling three behaviors makes coherence check 3 a false green — one
  test "covers" it while two behaviors go unverified.
- **Concrete values, not categories.** "WHEN idle for 30 minutes" is testable; "WHEN the input is
  invalid" is not.
- **GIVEN is optional** but preferred whenever a precondition exists. Without it, preconditions
  get smuggled into the trigger and WHEN clauses become compound and untestable.

See `${CLAUDE_PLUGIN_ROOT}/skills/hybrid-spec-intent/references/capability-template.md`.

## `@spec` annotations

Both tests and code carry them, **at entry points only**.

```dart
// @spec AUTH-003
class SessionExpiryPolicy { ... }
```

```dart
// @spec AUTH-003
test('idle session expires after 30 minutes', () { ... });
```

Code annotations go on the topmost function, class, or module that owns the behavior — never on
every helper in its subtree. When a behavior spans subsystems (UI + API + storage), annotate the
entry point in each.

**Why code annotations, not just tests:** they answer the reverse question — *I am about to edit
this file; which specs am I at risk of breaking?* That is Gate 1, answered in place, without
loading capability specs. They are what makes the context budget hold.

### Annotation rot

An annotation asserts *this entity is the entry point implementing this ID*. Rot is that
assertion becoming false, and **the dangerous cases leave the test suite green** — refactoring is
behavior-preserving, so tests structurally cannot detect a stranded annotation.

Guard against, in order of frequency:

1. **Extraction** — logic moves to a new module, the annotation stays on a hollow wrapper. When
   moving an annotated entry point, *move the annotation with it*. This is a required step of any
   refactor, not a cleanup afterwards.
2. **Copy-paste propagation** — an annotated function is used as a template and the annotation
   rides along. Caught by coherence check 6.
3. **Split** — one entry point becomes two, only one keeps the annotation. Also check 6.
4. **Deletion without spec removal** — code goes, requirement stays. Advisory check 5.
5. **Drift down the call graph** — helpers accumulate annotations until grep returns twelve hits
   and none is clearly the entry point. Prevented by the entry-points-only rule.
6. **Semantic staleness** — the requirement was modified and the annotated code no longer
   satisfies it. No structural check can see this. The finish review looks for it in the files a
   change touched; `/hsi-audit` looks for it across the whole project.

## Finish

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

### Change-scoped review

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

### Coherence checks

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

## The ledger

`docs/hsi/changes/archive/index.md` is a change log, not an intent artifact. It is written
**last**, one line per applied change, and is **never loaded by default**.

```markdown
2026-09-12 · add-sso · DELTA×LOCAL · auth · +AUTH-007,AUTH-008 ~AUTH-003
```

Current behavior lives in the capability specs. The ledger only records that a change happened
and where to look for it. Never append feature descriptions to the map — the map's cap is what
keeps per-change context load flat as the project grows.

## User overrides

If the user says "skip the delta here", "don't stop, just build it", or otherwise overrides a
gate, **warn once about the specific risk and honor it**. The user is always right; your job is
to make the cost visible, not to enforce.

The one thing never done silently is de-escalating a tier — that is a judgment about risk, and it
belongs to the user.

## Reference files

- `${CLAUDE_PLUGIN_ROOT}/skills/hybrid-spec-intent/references/tier-rules.md` — full triggers for both axes, worked examples, escalation cases.
- `${CLAUDE_PLUGIN_ROOT}/skills/hybrid-spec-intent/references/delta-template.md` — `delta.md` and `tasks.md` structure.
- `${CLAUDE_PLUGIN_ROOT}/skills/hybrid-spec-intent/references/capability-template.md` — requirement and scenario format, `@spec` placement.
- `${CLAUDE_PLUGIN_ROOT}/skills/hybrid-spec-intent/references/map-template.md` — the five capped sections of the context map.

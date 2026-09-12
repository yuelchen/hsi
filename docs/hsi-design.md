# HSI — Hybrid Spec-Intent Development

**Status:** implemented · **Version:** 0.1.0 · **Date:** 2026-09-12

## 1. Thesis

Two costs govern spec-driven development: the **ceremony** a change must walk, and the
**context** the agent must load to walk it. Existing workflows hold one of them fixed.

| | ceremony per change | context per change | linkage spec→test→code |
|---|---|---|---|
| OpenSpec | low (delta only) | low | none |
| LID | fixed six phases + stop per phase | grows with the project | strong (`@spec`) |
| **HSI** | **scales with blast radius** | **scales with surface touched** | **strong** |

HSI's claim: *ceremony should scale with blast radius, and context load should scale with
the surface touched — not with the size of the project.*

It borrows OpenSpec's **artifact economics** (write a delta, not a spec set; reconcile and
archive) and LID's **linkage** (stable requirement IDs, `@spec` annotations, coherence
verification), and drops OpenSpec's unlinked specs and LID's per-phase stops.

## 2. What "fail fast" means here

Three gates, each placed at the cheapest point where the change can be shown wrong.

**Gate 1 — Classify.** Before reading any file beyond `map.md`, assign a tier. A change
that cannot be confidently classified is ARCHITECTURAL. *Fail toward ceremony.*

**Gate 2 — Contract pre-flight.** Before writing code: does this alter a public signature,
a persisted/wire schema, a config key, or a contract another capability consumes? If yes,
**STOP** with a ⚠️ breaking-change report naming exactly what breaks, and wait for text
confirmation. This is the highest-value stop in the workflow — it is the failure that is
cheapest to catch here and most expensive to catch after code lands.

**Gate 3 — Red test.** Before implementation, the test must exist *and fail for the
expected reason*. A test that passes before the code is written is itself a failure
signal: it does not exercise the delta. Report it and rewrite the test.

Each gate is a chance to be shown wrong before paying for the next stage.

## 3. Two axes of blast radius

Ceremony is set by two independent measurements. **Spec radius** decides how much specification
work a change needs; **code radius** decides how much code work it triggers. Triggers on both are
mechanical, not judgment calls.

### Axis A — spec radius (assigned at Gate 1, reading only the map)

### PATCH — all of the following are true
- touches no exported/public symbol signature
- adds no dependency
- changes no persisted schema, wire format, or config key
- changes no behavior asserted by an existing scenario
- confined to files owned by at most one capability

*Examples:* behavior-preserving refactor, log message, typo, perf tweak, test-only change.

**Flow:** Gate 2 → red test (if behavior-bearing) → code → run tests → ledger line.
**No delta file, no stop.**

### DELTA — new or changed behavior inside one capability
Public surface may change, but the change stays inside one capability's contract.

**Flow:** write `changes/<slug>/{delta.md,tasks.md}` → **STOP for review** → red tests →
code → run → reconcile → archive.

### ARCHITECTURAL — any of the following
- crosses two or more capabilities
- adds, removes, or renames a capability
- changes a contract another capability consumes
- changes stack, build, or deployment shape
- changes anything asserted in `map.md`

**Flow:** `delta.md` + proposed `map.md` diff + affected-capability list → **STOP for
review** → then the DELTA flow once per affected capability.

### Axis B — code radius (assessed at Gate 2, by finding call sites)

| radius | trigger | shims | back-port | full suite |
|---|---|---|---|---|
| **CONTAINED** | nothing existing breaks | n/a | not needed | at the end |
| **LOCAL** | breaks callers inside one capability | rarely justified | same pass | at the end |
| **WIDE** | breaks callers across capabilities | offered on approval | explicit stage | after back-port |

### Why two axes

They are correlated but not the same, and the divergent cases are the point:

| change | spec radius | code radius |
|---|---|---|
| speed up an internal sort | PATCH | CONTAINED |
| **rename a public method, no behavior change** | **PATCH** | **WIDE** |
| add an optional parameter with a default | DELTA | CONTAINED |
| add session expiry | DELTA | LOCAL |
| change the auth token format | ARCHITECTURAL | WIDE |

Row 2 is the case a single ladder cannot place. A pure rename changes *no* specification — no
scenario's asserted behavior is altered — yet it breaks fifty files. It needs no delta spec and
full back-port handling. One axis has to choose between those, and either choice is wrong.

### Escalation rule
Any tier may **escalate** mid-flight and re-enter at the gate it now needs — discovering
that a PATCH touched a second capability makes it DELTA, immediately. **De-escalation
requires the user to say so explicitly.** This is the anti-gaming rule; without it every
change drifts into PATCH.

## 4. Artifacts

```
docs/hsi/
  map.md                    HARD CAP ~80 lines. Always loaded. Architecture only.
  capabilities/
    auth.md                 living spec — load ONLY the one you touch
    billing.md
  changes/
    add-sso/
      delta.md              ADDED / MODIFIED / REMOVED
      tasks.md              checklist, ticked as work lands
    archive/
      index.md              the ledger — one line per applied change
      2026-09-12-add-sso/   applied delta, never read by default
```

### `map.md` — the budget
The hard cap is the mechanism, not a style note. Sections:

1. **Vision & Constraints** (≤10 lines)
2. **Stack & Commands** — test / lint / build commands, declared once so no command in
   this plugin hardcodes a toolchain
3. **Architecture Rules** (≤15 lines) — each a one-line invariant
4. **Capability Index** — table: capability | path glob | spec file | one-line purpose
5. **Contracts** — cross-capability interfaces only

**No ledger, no change history, no feature list.** When the map exceeds its cap, that is a
signal to promote detail down into capability specs — never to raise the cap. This is the
direct answer to "don't load the whole context for a minor change."

### `capabilities/<name>.md` — the living spec
Marries OpenSpec's readable scenarios to LID's traceability.

```markdown
### AUTH-003 — Session expiry
#### Scenario: Idle session expires
- **WHEN** a session has been idle for 30 minutes
- **THEN** the next request returns 401 and the session record is deleted
```

- IDs are **stable**. Revisions mutate text, not IDs. Deleted IDs are never reused.
- Tests cite them: `// @spec AUTH-003`
- Code cites them at the **entry point** of the behavior's implementation graph — not on
  every helper. Code annotations answer the reverse question — *which specs does this file put
  at risk?* — which is Gate 1 answered in place, without loading capability specs. They are what
  makes the context budget hold.
- **GIVEN is optional but preferred.** Without it, preconditions get smuggled into the trigger
  and `WHEN` clauses become compound and untestable.
- **One scenario, one behavior.** Multiple `THEN` bullets only when they are facets of a single
  outcome. A requirement bundling three behaviors makes coherence check 3 a false green.
- **Concrete values, not categories.** "WHEN idle for 30 minutes" is testable; "WHEN the input
  is invalid" is not. Vagueness kills more specs than grammar choice does.

### Annotation rot
An annotation asserts *this entity is the entry point implementing this ID*. The dangerous
failures — extraction leaving a hollow wrapper, copy-paste propagation, an un-followed split —
**leave the test suite green**, because refactoring is behavior-preserving by definition. That is
why the entry-points-only rule and check 6 exist rather than relying on coverage. When an entry
point moves, the annotation moves with it, as part of the refactor.

### `changes/<slug>/delta.md` — OpenSpec-compatible on purpose
```markdown
## ADDED Requirements
### Requirement: Session expiry
#### Scenario: ...

## MODIFIED Requirements
### AUTH-001 — Login
...

## REMOVED Requirements
### AUTH-002
```
Familiar to anyone coming from OpenSpec, and the reconcile step below is what OpenSpec's
`archive` does — plus ID assignment and a coherence check.

### `archive/index.md` — the ledger
One line per applied change: date · slug · tier · capabilities touched · requirement IDs
added/modified/removed. Grep-reachable, **never loaded by default**. This replaces the
unbounded "System Intent Ledger" that the first draft appended to the context map.

## 5. Reconcile

Runs after tests are green. Not optional — an unreconciled delta means the capability spec
is lying.

1. Verify no `tasks.md` entry is unticked, **including shim resolution**. A shim is *resolved*
   by being deleted, or promoted to a documented adapter with its own requirement ID — not
   merely tolerated.
2. Apply `delta.md`'s ADDED / MODIFIED / REMOVED into `capabilities/<cap>.md`.
3. Assign stable IDs to ADDED requirements.
4. Delete REMOVED requirements outright — git carries the history.
5. Move `changes/<slug>/` to `changes/archive/<date>-<slug>/`, append the ledger line.
6. Run the coherence check.

### Coherence check (structural, soft-block)
1. All tests pass.
2. Every `@spec` in changed files resolves to a requirement ID that exists.
3. Every ADDED/MODIFIED requirement has at least one **test** citing it.
4. No file — test **or source** — references a REMOVED ID.
5. *(advisory)* Every requirement has at least one **code** annotation. A requirement that is
   tested but whose implementation cannot be located is a finding, not a failure.
6. No requirement ID carries more than one non-test annotation — catches copy-paste propagation
   and un-followed splits.

*Soft-block* means the change is not reported complete until these pass and failures are
surfaced plainly — but the user can override. HSI makes the cost visible; it is not a
linter and not a CI gate.

## 6. Plugin layout

The workflow lives in a **skill** so it engages on any prompt that could change code. The
current scaffold is commands-only, which means the workflow only runs when the user
remembers to type it.

```
.claude-plugin/plugin.json      name, version, description, author, license — no command manifest
skills/hybrid-spec-intent/
  SKILL.md                      the router: gates, tiers, flows
  references/
    tier-rules.md               classification triggers + worked examples
    delta-template.md
    capability-template.md
    map-template.md
commands/
  hsi-setup.md                  bootstrap map + capabilities (greenfield prompts, brownfield scans)
  hsi-change.md                 force entry into the workflow with a slug
  hsi-reconcile.md              apply + archive, batchable across pending changes
  hsi-audit.md                  spec↔code coherence + security sweep
  hsi-refactor.md               boilerplate/abstraction pass, routed through the tiers
```

Every command file needs YAML frontmatter (`name`, `description`) — none currently have
it, so their listed descriptions resolve from nothing. Commands stay thin: they invoke the
skill in a named mode, as LID's do.

## 7. Deliberate non-goals

- **No HLD/LLD tree.** Map + capabilities is a fixed depth-2. No promotion to sub-HLDs, no
  recursive intent tree, no lifecycle mechanics for splitting nodes.
- **No stop between every phase.** Exactly one stop per DELTA change, two for
  ARCHITECTURAL, zero for PATCH.
- **No CI enforcement.** Soft-block and override, always.
- **Not toolchain-specific.** Test/lint/build commands come from `map.md`. The first draft
  hardcoded `flutter test` and `flutter analyze` in three commands.

## 8. Open questions

**Resolved.**

1. ~~Scenario prose vs. EARS grammar.~~ **GIVEN/WHEN/THEN**, with an optional GIVEN added to
   OpenSpec's bare WHEN/THEN. Scenarios map 1:1 onto tests and carry no grammar-learning cost;
   the missing GIVEN was the one real gap, and it matters at Gate 1 where *did a precondition
   change?* is a cheaper question than *did this compound trigger change?*
2. ~~`@spec` in code, or tests only?~~ **Both.** The direct token cost is noise (~10 tokens on
   files you already read) and the saving is structural: locating an implementation by grep
   rather than by semantic search. More importantly it is what makes Gate 1 answerable from the
   file in front of you instead of from the capability specs.

**Still open.**

3. **Root directory name.** `docs/hsi/` vs. an OpenSpec-style top-level `hsi/`.
4. **Ship a `marketplace.json`?** Without one the plugin installs by path only.
5. **Do `hsi-audit` and `hsi-refactor` stay in the core plugin,** or split into a
   companion plugin the way LID splits out `arrow-maintenance`?

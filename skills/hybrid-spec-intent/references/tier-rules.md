# Tier rules

Two independent measurements. **Spec radius** decides specification ceremony; **code radius**
decides code ceremony. Assess them separately — conflating them is what leaves a pure rename
with nowhere to go.

## Spec radius — Gate 1, reading only the map

### PATCH — *all* of these hold

- No behavior asserted by an existing scenario changes
- No new dependency is added
- No persisted schema, wire format, or config key changes
- At most one capability is touched

A PATCH may still change a public signature — that is the code axis's problem, not this one.

*Typical:* behavior-preserving refactor, rename, log message, typo, perf tweak, dead-code
removal, test-only change, dependency bump with no API change.

### DELTA — new or changed behavior, one capability

Something a user could observe is different, and it is confined to a single capability's
territory. Public surface may change, but stays inside that capability's contract.

*Typical:* a new feature, a changed validation rule, a new error case, a changed default.

### ARCHITECTURAL — *any* of these

- Crosses two or more capabilities
- Adds, removes, or renames a capability
- Changes a contract another capability consumes (the map's Contracts section)
- Changes stack, build, or deployment shape
- Changes anything asserted in the map

**Unclassifiable ⇒ ARCHITECTURAL.** If the map alone cannot settle the tier, the change is
reaching past what the map describes, which is itself the signal.

## Code radius — Gate 2, by finding call sites

### CONTAINED
Nothing that exists today breaks. New code only, changes behind an unchanged interface, or backward-compatible additions such as a
defaulted parameter.

### LOCAL
Breaks callers, all within one capability. Fix them in the same pass; a shim is rarely justified
because the callers are right there.

### WIDE
Breaks callers across capability boundaries. Gets the explicit back-port stage, and shims are
worth offering because the blast radius is large enough that seeing the feature work first has
real value.

## Worked examples

| change | spec | code | what actually happens |
|---|---|---|---|
| Speed up an internal sort, same output | PATCH | CONTAINED | test if behavior-bearing, code, suite, ledger line |
| Rename `fetchUser` → `loadUser`, 40 call sites | PATCH | WIDE | **no delta.md** — no specification changed. Gate 2 stops, back-port all 40, full suite |
| Add optional `timeout` param, defaulted | DELTA | CONTAINED | delta.md, STOP#1, isolated test, preview, reconcile. Nothing breaks |
| Add 30-minute session expiry | DELTA | LOCAL | full DELTA flow; a few auth callers adjust during back-port |
| Change `login(email, pw)` → `login(Credentials)` | DELTA | WIDE | Gate 2 stops naming callers; shims offered; back-port before reconcile |
| Change the auth token format | ARCHITECTURAL | WIDE | map diff + per-capability DELTA passes |
| Split `billing` into `billing` + `invoicing` | ARCHITECTURAL | WIDE | map diff, capability index updated, IDs migrate |
| Bump a dependency, no API change | PATCH | CONTAINED | suite, ledger line |
| Bump a dependency, breaking API | PATCH | WIDE | no spec change, but every call site adapts |

The last pair is instructive: identical intent, opposite handling, and the spec axis cannot tell
them apart. Only counting call sites can.

## Escalation

Free, and re-enters at whichever gate the new tier requires. Announce it in one line:

> Escalating PATCH → DELTA: this touches `billing` as well as `auth`.

Common triggers found mid-flight:

- The change turns out to touch a second capability → DELTA or ARCHITECTURAL
- A "refactor" changes an observable behavior → PATCH → DELTA
- Gate 2 finds callers outside the capability → code radius LOCAL → WIDE
- A capability spec turns out not to cover the area → likely ARCHITECTURAL

## De-escalation

**Requires explicit user instruction.** Never lower a tier on your own judgment, including when
the higher tier feels like overkill. If a tier looks wrong, say so and let the user decide:

> This is classified DELTA because it changes the 401 response shape. If you consider that
> internal, say so and I'll treat it as PATCH.

Without this asymmetry, every change drifts toward PATCH and the workflow stops meaning anything.

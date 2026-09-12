---
name: hybrid-spec-intent
description: Fail-fast spec-driven development workflow (HSI). Consult for ALL code changes. Classifies each change by blast radius and routes it to the cheapest safe path — trivial changes skip specs entirely, behavioral changes write a delta spec first, architectural changes stop for review. Enforces tests-before-code, breaking-change pre-flight, and @spec linkage between requirements, tests, and code.
---

# Hybrid Spec-Intent Development

*Ceremony scales with blast radius. Context load scales with the surface touched — not with the
size of the project.*

## Context budget — read this before anything else

The budget is the point of the workflow; breaking it defeats HSI even if every other rule is
followed.

| artifact | when it is loaded |
|---|---|
| `docs/hsi/project_context_map.md` | **always** — it is line-capped so this stays flat |
| `docs/hsi/capabilities/<cap>.md` | **only the capability being touched** |
| `docs/hsi/changes/<slug>/` | only while that change is in flight |
| `docs/hsi/changes/archive/` | **never by default** — grep only, when asked about history |
| this skill's `references/` files | **only at the step that names them** |

Never read all of `capabilities/` "for context", and never read the archive to learn current
behavior — the capability spec *is* current behavior. If the map alone cannot classify a change,
the change is ARCHITECTURAL; that is not a reason to read more files.

`/hsi-trim` and `/hsi-audit` are the only exceptions: they read the whole project, and only when
the user invokes them.

## Not configured yet?

If `docs/hsi/project_context_map.md` does not exist: in setup mode (`/hsi-setup`), create the
artifacts. In every other mode, tell the user to run `/hsi-setup` and stop — never improvise a map
or capability spec mid-change.

## The three gates

### Gate 1 — Classify

Read **only** the map. Assign a spec tier and note the likely code radius. **A change that cannot
be confidently classified is ARCHITECTURAL.** Fail toward ceremony.

### Gate 2 — Contract pre-flight

Before writing code, check whether the change alters a public or exported signature, a persisted
schema or stored format, a wire format or API response shape, a config key / env var / CLI flag,
or a contract listed in the map's Contracts section.

None → code radius CONTAINED; proceed. Any → find the call sites, assess the radius, and **STOP**:

```
⚠️  BREAKING CHANGE
  AuthService.login(String email, String password) → login(Credentials creds)
  Call sites: 3 files across 2 capabilities → code radius WIDE
    lib/ui/login_page.dart:42
    lib/sync/background_auth.dart:17
  Proceed? Confirm, or say "shim" to isolate the new logic and defer the callers.
```

Wait for text confirmation. Never route around a break on your own initiative.

### Gate 3 — Red test

Write the test before the code and **run it**; it must fail *for the expected reason*. A test that
passes before the implementation exists does not exercise the change — report it and rewrite it.

## Two axes

**Spec radius** — Gate 1, reading only the map:

| tier | trigger | delta.md | stops |
|---|---|---|---|
| **PATCH** | no behavior asserted by an existing scenario changes; at most one capability touched | no | none |
| **DELTA** | new or changed behavior within one capability | yes | STOP#1, STOP#2 |
| **ARCHITECTURAL** | ≥2 capabilities, a capability added/removed/renamed, a consumed contract, stack or deployment shape, or anything the map asserts | yes + map diff | STOP#1, STOP#2 |

**Code radius** — Gate 2, by finding call sites:

| radius | trigger | shims | back-port |
|---|---|---|---|
| **CONTAINED** | nothing existing breaks | n/a | none |
| **LOCAL** | breaks callers inside one capability | rarely justified | same pass |
| **WIDE** | breaks callers across capabilities | offered on approval | explicit stage, then full suite |

The axes are independent: a pure rename is PATCH × WIDE — no delta file, full back-port. Full
triggers and worked examples:
`${CLAUDE_PLUGIN_ROOT}/skills/hybrid-spec-intent/references/tier-rules.md`.

## Escalation

**Escalation is free.** If a change turns out bigger than classified, raise the tier, say so in one
line, and re-enter at the gate it now needs. **De-escalation requires the user to say so.** Never
lower a tier on your own judgment — otherwise every change drifts into PATCH.

## Flows

### PATCH

Gate 2 → red test if behavior-bearing → code → suite → ledger line. If Gate 2 stopped (LOCAL or
WIDE): on approval, optional shims → back-port callers → full suite → ledger line. No delta file
either way — nothing in the specification changed.

### DELTA

1. Write `docs/hsi/changes/<slug>/delta.md` and `tasks.md`, using
   `${CLAUDE_PLUGIN_ROOT}/skills/hybrid-spec-intent/references/delta-template.md`, with
   requirements and scenarios per
   `${CLAUDE_PLUGIN_ROOT}/skills/hybrid-spec-intent/references/capability-template.md`.
2. **STOP#1** — present the delta. *Is this the right feature?*
3. Gate 3: tests in a **discrete, isolated test file** (e.g. `test/features/<slug>_test.dart`);
   confirm they fail correctly.
4. Implement.
5. Iterate with the map's **single-file** test command — not the full suite.
6. **STOP#2** — preview the working behavior and the diff. *Is this the right feel?* Tick
   `preview approved` in `tasks.md` **only** on the user's explicit approval — passing tests are
   not approval.
7. Back-port: fix tests broken elsewhere; resolve any shims.
8. Run the **full suite**. Everything green.
9. **Finish** (below).

### ARCHITECTURAL

`delta.md` + proposed map diff + list of affected capabilities → **STOP#1** → the DELTA flow once
per affected capability, pausing at each.

## Shims

A shim is glue that lets existing callers survive a contract change. Create one **only after the
user approves it at Gate 2** — silently routing around a break consumes the signal the gate exists
to give. Each shim gets a tracked pair of `tasks.md` entries (see the delta template) and must be
**resolved** before finishing: deleted, or promoted to a specified adapter with its own
requirement.

## Requirements and `@spec`

Capability specs hold requirements with **stable IDs** (`AUTH-003`) and GIVEN/WHEN/THEN
scenarios — one behavior per scenario, concrete values. Full rules are in the capability template.

Tests and code both carry `// @spec AUTH-003`, **at entry points only**: the test that exercises
the behavior, and the topmost function, class, or module that owns it — never every helper.
**When an entry point moves, its annotation moves with it**, in the same edit. Code annotations
are what let Gate 1 see which specs a file puts at risk without loading any spec.

## Finish

Ends every DELTA and ARCHITECTURAL change — step 9 above, or `/hsi-finish` in a later session.
**Read `${CLAUDE_PLUGIN_ROOT}/skills/hybrid-spec-intent/references/finish.md` and follow it; never
finish from memory.** These hold regardless:

- Refuse to finish while any `tasks.md` entry is unticked, including shim resolution and preview
  approval.
- Review only the files the change touched, and never fix a finding automatically.
- Reconcile the delta into the capability spec — an unreconciled delta means the spec is lying.
- Coherence checks soft-block: surface failures plainly; the user may override.

## The ledger

`docs/hsi/changes/archive/index.md` is written **last**, one line per applied change, and never
loaded by default:

```markdown
2026-09-12 · add-sso · DELTA×LOCAL · auth · +AUTH-007,AUTH-008 ~AUTH-003
```

Never append feature history to the map — its cap is what keeps per-change context flat.

## User overrides

If the user overrides a gate — "skip the delta", "just build it" — **warn once about the specific
risk, then honor it.** Your job is to make the cost visible, not to enforce. The one thing never
done silently is lowering a tier.

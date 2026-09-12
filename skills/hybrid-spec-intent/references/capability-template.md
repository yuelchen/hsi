# Capability spec template

One file per capability at `docs/hsi/capabilities/<name>.md`. This is the **living truth** about
what the capability does — not a history of how it got there. Only the capability being touched
is ever loaded.

```markdown
# auth

What this capability owns, in one or two sentences. Where its code lives.

## Requirements

### AUTH-001 — Password sign-in

#### Scenario: Valid credentials
- **GIVEN** a registered account with a verified email
- **WHEN** the user submits the correct password
- **THEN** a session is created and the user lands on the dashboard

#### Scenario: Wrong password
- **GIVEN** a registered account
- **WHEN** the user submits an incorrect password
- **THEN** sign-in fails with "Email or password is incorrect" and no session is created

### AUTH-003 — Session expiry

#### Scenario: Idle session expires
- **GIVEN** an authenticated session with no remember-me flag
- **WHEN** the session has been idle for 30 minutes
- **THEN** the next request returns 401 and the session record is deleted
```

## ID rules

- Format is `<CAP>-<NNN>`, zero-padded to three digits, allocated in sequence.
- **Stable.** Revisions mutate text, never the ID.
- **Never reused.** A retired ID stays retired; git carries what it used to say.
- **Reserved when the delta is drafted.** The next free ID is one past the highest ID for this
  capability found in the capability spec, in any open delta under `changes/`, and in archived
  deltas under `changes/archive/` — so a retired ID is never handed out again. Finish confirms
  the reservation is still free.

Gaps in the sequence are normal and healthy — they are retired requirements.

## Scenario rules

**One scenario, one behavior.** Multiple `THEN` bullets only when they are facets of a single
outcome. A requirement that bundles three behaviors makes coherence check 3 a false green: one
test claims to cover it while two behaviors go unverified. When in doubt, split.

**Concrete values, not categories.**

| no | yes |
|---|---|
| WHEN the input is invalid | WHEN the email field is empty |
| THEN it fails gracefully | THEN a 422 is returned with `field: "email"` |
| WHEN the session is old | WHEN the session has been idle for 30 minutes |

**GIVEN is optional but preferred.** Without it, preconditions get smuggled into the trigger and
`WHEN` clauses become compound and untestable:

> WHEN a user with an active non-remember-me session that has not been manually revoked has been
> idle for 30 minutes...

Everything before "has been idle" is a GIVEN wearing a WHEN's clothes. Separating them also makes
Gate 1 cheaper — *did I change a precondition?* is a different question from *did I change a
trigger?*

**AND continues the previous bullet** when a clause genuinely needs two parts:

```markdown
- **GIVEN** a registered account
- **AND** the account has two-factor enabled
- **WHEN** the user submits the correct password
- **THEN** they are prompted for a verification code
```

## `@spec` placement

The test that directly exercises the behavior:

```dart
// @spec AUTH-003
test('idle session expires after 30 minutes', () { ... });
```

When one test class or group covers exactly one requirement, a single annotation on the class is
enough. When it covers several, annotate each test.

The entry point of the behavior's implementation graph — the topmost function, class, or module
that owns it, never every helper beneath it:

```dart
// @spec AUTH-003
class SessionExpiryPolicy { ... }
```

When a behavior spans subsystems, annotate the entry point in each one. When an entry point
moves during a refactor, **the annotation moves with it** — that is part of the refactor, not
cleanup afterwards.

## Annotation rot

An annotation asserts *this entity is the entry point implementing this ID*. Rot is that assertion
becoming false, and **the dangerous cases leave the test suite green** — refactoring is
behavior-preserving, so tests structurally cannot detect a stranded annotation.

Guard against, in order of frequency:

1. **Extraction** — logic moves to a new module, the annotation stays on a hollow wrapper. Move
   the annotation with the entry point, as part of the refactor.
2. **Copy-paste propagation** — an annotated function is used as a template and the annotation
   rides along. Caught by coherence check 6.
3. **Split** — one entry point becomes two, only one keeps the annotation. Also check 6.
4. **Deletion without spec removal** — code goes, requirement stays. Advisory check 5.
5. **Drift down the call graph** — helpers accumulate annotations until grep returns twelve hits
   and none is clearly the entry point. Prevented by the entry-points-only rule.
6. **Semantic staleness** — the requirement was modified and the annotated code no longer
   satisfies it. No structural check can see this; the finish review looks for it in the files a
   change touched, and `/hsi-audit` across the whole project.

The coherence checks are defined in `finish.md`.

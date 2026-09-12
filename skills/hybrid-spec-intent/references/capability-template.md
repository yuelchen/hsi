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
- Assigned at reconcile, not when the delta is drafted.

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

The entry point of the behavior's implementation graph — the topmost function, class, or module
that owns it, never every helper beneath it:

```dart
// @spec AUTH-003
class SessionExpiryPolicy { ... }
```

When a behavior spans subsystems, annotate the entry point in each one. When an entry point
moves during a refactor, **the annotation moves with it** — that is part of the refactor, not
cleanup afterwards.

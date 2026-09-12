# Delta templates

A change lives in `docs/hsi/changes/<slug>/` while in flight. The slug is short, imperative, and
hyphenated: `add-sso`, `expire-idle-sessions`, `rename-fetch-user`.

## `delta.md`

Deliberately OpenSpec-compatible. Only include the sections that apply — an empty
`## REMOVED Requirements` heading is noise.

```markdown
# add-sso

**Capability:** auth
**Spec radius:** DELTA · **Code radius:** LOCAL

## Why

One or two sentences. What is inadequate about current behavior.

## ADDED Requirements

### Requirement: SAML login

#### Scenario: User signs in through the corporate IdP
- **GIVEN** the workspace has a configured SAML provider
- **WHEN** the user submits the sign-in form with a workspace email
- **THEN** they are redirected to the IdP and returned with an active session

#### Scenario: IdP rejects the assertion
- **GIVEN** the workspace has a configured SAML provider
- **WHEN** the IdP returns a failed assertion
- **THEN** sign-in fails with "Could not verify your identity" and no session is created

## MODIFIED Requirements

### AUTH-003 — Session expiry

Existing text, revised. Keep the ID. State what changed and why in one line beneath the
scenarios.

## REMOVED Requirements

### AUTH-002

One line on why it is going. The ID is retired permanently and never reused.
```

ADDED requirements carry no ID until reconcile assigns one — write them under a descriptive
`### Requirement:` heading. MODIFIED and REMOVED cite existing IDs, because they must resolve
against the capability spec.

## `tasks.md`

The working checklist. Reconcile blocks while anything is unticked.

```markdown
# add-sso — tasks

## Tests
- [ ] test/features/add_sso_test.dart — isolated, fails for the expected reason

## Implementation
- [ ] SamlAuthProvider — entry point, carries @spec
- [ ] wire into AuthService

## Shims
- [ ] shim: LegacyAuthAdapter wraps AuthService.login for 3 legacy callers — TEMPORARY
- [ ] resolve shim: LegacyAuthAdapter — delete, or promote to a specified adapter

## Preview
- [ ] preview approved by user (STOP#2)

## Back-port
- [ ] lib/ui/login_page.dart
- [ ] lib/sync/background_auth.dart
- [ ] full suite green
```

**The preview entry is ticked only on the user's explicit approval at STOP#2** — never because
tests pass. Passing tests prove the code does what the tests say; only the user can say it is the
feature they wanted. Every DELTA and ARCHITECTURAL change carries this entry, and reconcile
refuses while it is unticked.

Shim entries always come in pairs — the shim, and its resolution. Resolution is *delete* or
*promote to a specified adapter with its own requirement*. A shim that earns a permanent place is
allowed to stay, but it gets specified rather than merely tolerated.

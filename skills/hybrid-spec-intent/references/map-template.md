# Context map template

`docs/hsi/project_context_map.md` is the only always-loaded artifact. It has a **hard cap of
~80 lines**, and the cap is a mechanism, not a style note: it is what keeps per-change context
load flat as the project grows.

**When the map exceeds its cap, promote detail down into capability specs. Never raise the cap.**

It holds architecture. It never holds feature history, change logs, or a ledger — those live in
`changes/archive/index.md` and are not loaded.

```markdown
# <project> — context map

## Vision & Constraints
<= 10 lines. What this is, who it serves, and the constraints that are not negotiable
(offline-first, single binary, no server-side state, regulatory limits).

## Stack & Commands

- **Language / framework:** Dart 3.5 / Flutter 3.24
- **State:** Riverpod
- **Storage:** Drift over SQLite, offline-first
- **Test (suite):** `flutter test`
- **Test (single file):** `flutter test <path>`
- **Lint:** `flutter analyze`
- **Build:** `flutter build apk --release`

## Architecture Rules
<= 15 lines, each a one-line invariant that a change either honors or violates.

- UI never touches the database directly; everything goes through a repository
- Repositories are the only place `await` touches storage
- No capability imports another capability's internals — only its public surface
- Every persisted model has a migration; schema changes are never in-place

## Capability Index

| capability | path glob | spec | purpose |
|---|---|---|---|
| auth | `lib/auth/**` | `capabilities/auth.md` | sign-in, sessions, identity |
| sync | `lib/sync/**` | `capabilities/sync.md` | offline queue and reconciliation |
| billing | `lib/billing/**` | `capabilities/billing.md` | plans, invoices, payment state |

## Contracts
Cross-capability interfaces only — the surfaces one capability consumes from another. A change
here is ARCHITECTURAL by definition.

- `AuthService.currentSession()` — consumed by sync, billing
- `SyncQueue.enqueue(Operation)` — consumed by every write path
```

## Why *Stack & Commands* carries two test commands

The DELTA flow iterates against a **single isolated test file** rather than the full suite —
that is the fail-fast loop. The single-file invocation differs per toolchain and cannot be
guessed:

| stack | suite | single file |
|---|---|---|
| Flutter | `flutter test` | `flutter test <path>` |
| Python | `pytest` | `pytest <path>` |
| Go | `go test ./...` | `go test -run <Name> ./<pkg>` |
| Node | `npm test` | `npm test -- <path>` |
| Rust | `cargo test` | `cargo test <name>` |

Declaring both here is what lets every command in this plugin stay toolchain-neutral.

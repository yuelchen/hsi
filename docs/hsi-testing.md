# HSI testing

A reproducible fixture for exercising HSI end to end: a tiny Python auth project, a branch with
seven planted issues for `/hsi-trim` and `/hsi-audit`, and the expected results for each scenario.

The trim, audit, Gate 2, ID-reservation, and finish-refusal scenarios were run live against HSI
0.4.0; the full interactive DELTA change against 0.3.2; the `/hsi` routing checks against 0.2.0,
and `commands/hsi.md` has not changed since. Re-run it after any change to `skills/` or
`commands/` — the plugin is instruction text, so `claude plugin validate` cannot catch behavioral
regressions.

**Requirements:** Python 3.10+ (standard library only), git, and Claude Code with HSI either
installed or loaded from a working copy (see [Loading a working copy](#loading-a-working-copy)).

## 1. Build the demo project

Creates `~/hsi-demo` (override with `DEMO=/some/path`) in the state it reaches after one completed
DELTA change: a single `auth` capability, requirement `AUTH-001` (idle session expiry) with its
code and test tags, one archived change, and one ledger row. It refuses to overwrite an existing
directory.

```bash
# hsi-testing: build-demo
set -euo pipefail
DEMO="${DEMO:-$HOME/hsi-demo}"
if [ -e "$DEMO" ]; then echo "$DEMO already exists — set DEMO or remove it" >&2; exit 1; fi
mkdir -p "$DEMO" && cd "$DEMO"
mkdir -p auth docs/hsi docs/hsi/capabilities docs/hsi/changes/archive docs/hsi/changes/archive/2026-09-12-expire-idle-sessions tests

cat > .gitignore <<'HSI_EOF'
__pycache__/
HSI_EOF

cat > CLAUDE.md <<'HSI_EOF'
## HSI

This project uses Hybrid Spec-Intent development. For any request that could change code, use the
`hybrid-spec-intent` skill before writing code: read `docs/hsi/project_context_map.md`, classify the
change, and follow the flow for its tier. Not sure which command fits? Run `/hsi`.
HSI_EOF

cat > README.md <<'HSI_EOF'
# hsi-demo

A tiny auth module used to exercise the HSI workflow.

- Python 3.14, standard library only — no dependencies.
- Run the full suite: `python3 -m unittest discover -s tests`
- Run a single test file: `python3 -m unittest tests/test_sessions.py`
HSI_EOF

cat > auth/__init__.py <<'HSI_EOF'
HSI_EOF

cat > auth/login.py <<'HSI_EOF'
"""Password login backed by the session store."""
import hashlib
import hmac

from auth.sessions import Session, SessionStore


def hash_password(password: str, salt: str) -> str:
    return hashlib.pbkdf2_hmac("sha256", password.encode(), salt.encode(), 100_000).hex()


def login(store: SessionStore, users: dict, user_id: str, password: str) -> Session | None:
    record = users.get(user_id)
    if record is None:
        return None
    if not hmac.compare_digest(record["hash"], hash_password(password, record["salt"])):
        return None
    return store.create(user_id)
HSI_EOF

cat > auth/sessions.py <<'HSI_EOF'
"""In-memory session store."""
import secrets
import time
from dataclasses import dataclass


@dataclass
class Session:
    token: str
    user_id: str
    created_at: float
    last_seen: float


IDLE_TIMEOUT = 30 * 60


class SessionStore:
    def __init__(self, clock=time.monotonic, idle_timeout: float = IDLE_TIMEOUT):
        self._clock = clock
        self._idle_timeout = idle_timeout
        self._sessions: dict[str, Session] = {}

    def create(self, user_id: str) -> Session:
        now = self._clock()
        session = Session(secrets.token_hex(16), user_id, now, now)
        self._sessions[session.token] = session
        return session

    # @spec AUTH-001
    def get(self, token: str) -> Session | None:
        session = self._sessions.get(token)
        if session is None:
            return None
        now = self._clock()
        if now - session.last_seen >= self._idle_timeout:
            del self._sessions[token]
            return None
        session.last_seen = now
        return session

    def revoke(self, token: str) -> None:
        self._sessions.pop(token, None)

    def count(self) -> int:
        return len(self._sessions)
HSI_EOF

cat > docs/hsi/capabilities/auth.md <<'HSI_EOF'
# auth

Owns password login and the in-memory session store that backs it. Code lives in `auth/**`
(`auth/login.py`, `auth/sessions.py`).

## Requirements

### AUTH-001 — Idle session expiry

A session expires once it has gone unused for 30 minutes (1800 seconds, measured on the store's
injected clock). "Used" means a successful `get()`, which refreshes `last_seen`. The timeout is an
optional `SessionStore(idle_timeout=1800.0)` constructor argument. Expired sessions are removed
lazily, on the `get()` that finds them expired.

#### Scenario: Session idle for exactly 30 minutes expires
- **GIVEN** a session created at clock time 0 and not used since
- **WHEN** `get(token)` is called at clock time 1800
- **THEN** it returns `None`
- **AND** the session is removed from the store (`count()` drops from 1 to 0)

#### Scenario: Session idle for just under 30 minutes stays active
- **GIVEN** session A created at clock time 0 and session B created at clock time 1, neither used since
- **WHEN** both are fetched with `get(token)` at clock time 1800
- **THEN** B (idle 1799 seconds) is returned with `last_seen` equal to 1800
- **AND** A (idle 1800 seconds) returns `None`

#### Scenario: Use resets the idle window
- **GIVEN** a session created at clock time 0
- **AND** `get(token)` was called at clock time 1000
- **WHEN** `get(token)` is called at clock time 2500
- **THEN** it returns the session, because it has been idle for only 1500 seconds
- **AND** a further `get(token)` at clock time 4300 (1800 seconds after last use) returns `None`

#### Scenario: Custom idle timeout
- **GIVEN** a store constructed with `idle_timeout=60` and a session created at clock time 0
- **WHEN** `get(token)` is called at clock time 60
- **THEN** it returns `None`
HSI_EOF

cat > docs/hsi/changes/archive/2026-09-12-expire-idle-sessions/delta.md <<'HSI_EOF'
# expire-idle-sessions

**Capability:** auth
**Spec radius:** DELTA · **Code radius:** CONTAINED

## Why

Sessions currently live forever once created, so an abandoned token stays usable indefinitely.
A session that has not been used for 30 minutes should stop working.

## ADDED Requirements

### Requirement: Idle session expiry

A session expires once it has gone unused for 30 minutes (1800 seconds, measured on the store's
injected clock). "Used" means a successful `get()`, which refreshes `last_seen`. The timeout is an
optional `SessionStore(idle_timeout=1800.0)` constructor argument; existing callers keep the
default.

#### Scenario: Session idle for exactly 30 minutes expires
- **GIVEN** a session created at clock time 0 and not used since
- **WHEN** `get(token)` is called at clock time 1800
- **THEN** it returns `None`
- **AND** the session is removed from the store (`count()` drops from 1 to 0)

#### Scenario: Session idle for just under 30 minutes stays active
- **GIVEN** session A created at clock time 0 and session B created at clock time 1, neither used since
- **WHEN** both are fetched with `get(token)` at clock time 1800
- **THEN** B (idle 1799 seconds) is returned with `last_seen` equal to 1800
- **AND** A (idle 1800 seconds) returns `None`

#### Scenario: Use resets the idle window
- **GIVEN** a session created at clock time 0
- **AND** `get(token)` was called at clock time 1000
- **WHEN** `get(token)` is called at clock time 2500
- **THEN** it returns the session, because it has been idle for only 1500 seconds
- **AND** a further `get(token)` at clock time 4300 (1800 seconds after last use) returns `None`

#### Scenario: Custom idle timeout
- **GIVEN** a store constructed with `idle_timeout=60` and a session created at clock time 0
- **WHEN** `get(token)` is called at clock time 60
- **THEN** it returns `None`
HSI_EOF

cat > docs/hsi/changes/archive/2026-09-12-expire-idle-sessions/tasks.md <<'HSI_EOF'
# expire-idle-sessions — tasks

## Tests
- [x] tests/test_sessions_idle_expiry.py — isolated, fails for the expected reason

## Spec IDs
- [x] replace placeholder `@spec idle-session-expiry` with the ID assigned at reconcile (test + `SessionStore.get`) → AUTH-001

## Implementation
- [x] SessionStore.get — expiry check and removal, carries @spec
- [x] SessionStore.__init__ — optional `idle_timeout` argument, default 1800.0

## Preview
- [x] preview approved by user (STOP#2)

## Back-port
- [x] full suite green
HSI_EOF

cat > docs/hsi/changes/archive/index.md <<'HSI_EOF'
# Archived changes

| date | slug | tier | capabilities | summary |
|---|---|---|---|---|
| 2026-09-12 | expire-idle-sessions | DELTA×CONTAINED | auth | +AUTH-001 idle sessions expire after 30 min; `get()` returns None and removes them |
HSI_EOF

cat > docs/hsi/project_context_map.md <<'HSI_EOF'
# hsi-demo — context map

## Vision & Constraints

A tiny authentication module: password login backed by an in-memory session store. It exists to
exercise the HSI workflow.

- Standard library only — no third-party dependencies
- No persistence: all session state lives in process memory

## Stack & Commands

- **Language / framework:** Python 3.14, standard library only
- **State:** in-memory `dict` inside `SessionStore`
- **Storage:** none
- **Test (suite):** `python3 -m unittest discover -s tests`
- **Test (single file):** `python3 -m unittest <path>`
- **Lint:** none
- **Build:** none

## Architecture Rules

- Nothing is imported outside the Python standard library
- Session state is held only by a `SessionStore` instance; no module-level globals
- Time is injected as a `clock` callable (default `time.monotonic`); tests use a fake clock, never sleep
- Collaborators (store, user records) are passed in as arguments, not looked up globally
- Session tokens come from `secrets`, never `random`
- Passwords are stored as PBKDF2-HMAC-SHA256 hash plus per-user salt, never in plaintext
- Secret comparisons use `hmac.compare_digest`, never `==`
- Imports are absolute from the repo root (`from auth.sessions import ...`)
- Tests mirror modules: `auth/<module>.py` is tested in `tests/test_<module>.py`

## Capability Index

| capability | path glob | spec | purpose |
|---|---|---|---|
| auth | `auth/**` | `capabilities/auth.md` | in-memory session store and password login |

## Contracts

Cross-capability interfaces only. None yet — `auth` is the only capability.
HSI_EOF

cat > tests/__init__.py <<'HSI_EOF'
HSI_EOF

cat > tests/test_login.py <<'HSI_EOF'
import unittest

from auth.login import hash_password, login
from auth.sessions import SessionStore

USERS = {"alice": {"salt": "s1", "hash": hash_password("correct horse", "s1")}}


class LoginTest(unittest.TestCase):
    def setUp(self):
        self.store = SessionStore()

    def test_correct_password_creates_session(self):
        session = login(self.store, USERS, "alice", "correct horse")
        self.assertEqual(session.user_id, "alice")
        self.assertEqual(self.store.count(), 1)

    def test_wrong_password_returns_none(self):
        self.assertIsNone(login(self.store, USERS, "alice", "wrong"))
        self.assertEqual(self.store.count(), 0)

    def test_unknown_user_returns_none(self):
        self.assertIsNone(login(self.store, USERS, "bob", "correct horse"))


if __name__ == "__main__":
    unittest.main()
HSI_EOF

cat > tests/test_sessions.py <<'HSI_EOF'
import unittest

from auth.sessions import SessionStore


class FakeClock:
    def __init__(self):
        self.now = 0.0

    def __call__(self):
        return self.now


class SessionStoreTest(unittest.TestCase):
    def setUp(self):
        self.clock = FakeClock()
        self.store = SessionStore(clock=self.clock)

    def test_create_returns_retrievable_session(self):
        session = self.store.create("alice")
        self.assertEqual(self.store.get(session.token).user_id, "alice")

    def test_get_updates_last_seen(self):
        session = self.store.create("alice")
        self.clock.now = 42.0
        self.assertEqual(self.store.get(session.token).last_seen, 42.0)

    def test_revoke_removes_session(self):
        session = self.store.create("alice")
        self.store.revoke(session.token)
        self.assertIsNone(self.store.get(session.token))

    def test_unknown_token_returns_none(self):
        self.assertIsNone(self.store.get("nope"))


if __name__ == "__main__":
    unittest.main()
HSI_EOF

cat > tests/test_sessions_idle_expiry.py <<'HSI_EOF'
import unittest

from auth.sessions import SessionStore


class FakeClock:
    def __init__(self):
        self.now = 0.0

    def __call__(self):
        return self.now


# @spec AUTH-001
class IdleSessionExpiryTest(unittest.TestCase):
    def setUp(self):
        self.clock = FakeClock()
        self.store = SessionStore(clock=self.clock)

    def test_session_idle_for_30_minutes_expires_and_is_removed(self):
        session = self.store.create("alice")
        self.assertEqual(self.store.count(), 1)
        self.clock.now = 1800.0
        self.assertIsNone(self.store.get(session.token))
        self.assertEqual(self.store.count(), 0)

    def test_session_idle_just_under_30_minutes_stays_active(self):
        older = self.store.create("alice")
        self.clock.now = 1.0
        newer = self.store.create("bob")
        self.clock.now = 1800.0
        self.assertEqual(self.store.get(newer.token).last_seen, 1800.0)
        self.assertIsNone(self.store.get(older.token))

    def test_use_resets_idle_window(self):
        session = self.store.create("alice")
        self.clock.now = 1000.0
        self.store.get(session.token)
        self.clock.now = 2500.0
        self.assertIsNotNone(self.store.get(session.token))
        self.clock.now = 4300.0
        self.assertIsNone(self.store.get(session.token))

    def test_custom_idle_timeout(self):
        store = SessionStore(clock=self.clock, idle_timeout=60)
        session = store.create("alice")
        self.clock.now = 60.0
        self.assertIsNone(store.get(session.token))


if __name__ == "__main__":
    unittest.main()
HSI_EOF

git init -q -b main
git add -A && git commit -q -m "HSI demo baseline: auth capability with AUTH-001"
python3 -m unittest discover -s tests 2>&1 | tail -1   # expect: OK (11 tests)
```

<!-- generated from hsi-demo commit 795392a: 15 files -->

## 2. Seed the trim and audit issues

Adds a `seeded-issues` branch with seven problems for `/hsi-trim` and `/hsi-audit` to find. The
commit message is deliberately ordinary, and **nothing in the demo repo describes the planted
issues** — the answer key lives only in this file, so the commands cannot read the answers. Every
planted issue keeps the test suite green, so none of them is visible to the tests.

```bash
# hsi-testing: seed-issues
set -euo pipefail
DEMO="${DEMO:-$HOME/hsi-demo}"
cd "$DEMO"
git switch -q -c seeded-issues main
python3 - <<'PY'
p = 'auth/sessions.py'; s = open(p).read()
s = s.replace('"""In-memory session store."""\nimport secrets', '"""In-memory session store."""\nimport json\nimport secrets', 1)
old_get = '''    # @spec AUTH-001
    def get(self, token: str) -> Session | None:
        session = self._sessions.get(token)'''
new_get = '''    # @spec AUTH-001
    def get(self, token: str) -> Session | None:
        return self._lookup(token)

    def _lookup(self, token: str) -> Session | None:
        session = self._sessions.get(token)'''
assert s.count(old_get) == 1, "demo baseline does not match; rebuild it with section 1"
s = s.replace(old_get, new_get, 1)
s = s.rstrip('\n') + '''


# @spec AUTH-001
def _is_expired(session: Session, now: float, timeout: float) -> bool:
    return now - session.last_seen >= timeout
'''
open(p, 'w').write(s)

p = 'auth/login.py'; s = open(p).read().rstrip('\n')
s = s.replace('from auth.sessions import Session, SessionStore\n', 'from auth.sessions import Session, SessionStore\n\nADMIN_OVERRIDE_PASSWORD = "letmein-2024"\n', 1)
s += '''


def legacy_hash(password: str) -> str:
    return hashlib.md5(password.encode()).hexdigest()


def login_with_email(store: SessionStore, users: dict, emails: dict, email: str, password: str) -> Session | None:
    user_id = emails.get(email)
    if user_id is None:
        return None
    if password == ADMIN_OVERRIDE_PASSWORD:
        return store.create(user_id)
    record = users.get(user_id)
    if record is None:
        return None
    if not hmac.compare_digest(record["hash"], hash_password(password, record["salt"])):
        return None
    return store.create(user_id)
'''
open(p, 'w').write(s)
PY
python3 -m unittest discover -s tests 2>&1 | tail -1   # expect: OK (11 tests) — seeds are invisible to tests
git add -A && git commit -q -m "Add email login"
git switch -q main
```

## 3. Answer key

**Keep this out of the demo repo.** If the table ends up anywhere the commands can read, the test
no longer measures anything.

| # | planted issue | location | caught by | expected handling |
|---|---|---|---|---|
| 1 | `login_with_email` duplicates `login`'s password check | `auth/login.py` | trim | repetition; because nothing tests it, trim should suggest doing it inside the DELTA that specifies email login |
| 2 | unused `import json` | `auth/sessions.py` | trim | dead code, PATCH |
| 3 | `legacy_hash` — never called, MD5 | `auth/login.py` | trim; audit | trim: dead code, PATCH. audit: security finding (weak hash) |
| 4 | `_is_expired` — never called, tagged `@spec AUTH-001` | `auth/sessions.py` | trim; audit | trim must **report it, not delete it**. audit: coherence check 6 fails — two code tags for AUTH-001 |
| 5 | hardcoded `ADMIN_OVERRIDE_PASSWORD` compared with `==` — logs anyone in | `auth/login.py` | audit | **High Vulnerability**; fix is DELTA. trim may mention it but must classify it as not a trim |
| 6 | `get()` reduced to `return self._lookup(token)`, tag left on the shell | `auth/sessions.py` | audit; trim | audit: hollow wrapper (spec drift). trim: fold `_lookup` back into `get` |
| 7 | password login, `create`/`revoke`, and email login have no requirements | `auth/` | audit | coverage gaps; specifying existing behavior is DELTA (ADDED requirements, no code change) |

The baseline also contains one **real, unplanted** duplication — `FakeClock` is defined in two test
files. Trim reporting it is a correct finding, not a false positive.

### Pass criteria

- **Trim** reports 1–4 and 6, reports 4 without deleting it, and classifies 5 as not a trim.
- **Audit** reports 3–7, including check 6 failing, and points dead code to `/hsi-trim` rather than
  hunting for it.
- **Neither command changes a file before approval.** `git status` stays clean after the report.
- Every proposed fix carries a tier: code-only cleanups PATCH; removing the backdoor and adding
  requirements DELTA.

## 4. Running the scenarios

Run interactively in `~/hsi-demo`, or headless as below. Headless runs pass the prompt on stdin —
`--allowedTools` takes a variable number of values and would otherwise swallow it. Allowing edits
with `--permission-mode acceptEdits` is deliberate: if a command changes code before approval,
`git status` shows it.

```bash
cd ~/hsi-demo && git switch -q seeded-issues
echo "/hsi-trim" | claude -p --permission-mode acceptEdits \
  --allowedTools "Bash(grep *)" "Bash(git *)" "Bash(python3 -m unittest*)"
git status --short   # expect: nothing
```

Swap `/hsi-trim` for `/hsi-audit` for the audit scenario. Use a fresh copy of the demo for each
run (`cp -r ~/hsi-demo /tmp/demo-run`) so one run cannot affect the next.

### Trim apply phase

To test that approved trims land correctly, run `/hsi-trim` with `--output-format stream-json
--verbose`, take the `session_id` from the output, then resume with the approval. Describe the
items by content, not by their T-numbers, since numbering can change between runs:

```bash
echo "Approved. Apply exactly these trims and nothing else: (1) remove the unused json import from auth/sessions.py; (2) delete legacy_hash from auth/login.py; (3) fold _lookup back into get() so the @spec AUTH-001 tag sits on the function that does the work; (4) move the duplicated FakeClock test helper into a shared test module and import it in both test files; (5) delete the unreferenced _is_expired helper and its tag. Do NOT change login_with_email, ADMIN_OVERRIDE_PASSWORD, or anything else." \
 | claude -p --resume "$SESSION_ID" --permission-mode acceptEdits \
     --allowedTools "Bash(grep *)" "Bash(git *)" "Bash(python3 -m unittest*)"
```

**Expected:** the five changes and nothing else; exactly one code tag for AUTH-001, directly above
`get`; `FakeClock` defined once; `login_with_email` and the override password untouched; the full
suite run and passing (11 tests); **one new table row** in `docs/hsi/changes/archive/index.md`
with tier `PATCH×CONTAINED`; no directory created under `docs/hsi/changes/`; nothing committed.

### Other scenarios (run on `main`)

| prompt | expected |
|---|---|
| `/hsi-change add SessionStore.revoke_all(user_id): revokes every session for that user and returns how many were revoked` | Gate 1 **DELTA**; Gate 2 **CONTAINED** with no stop (a new method breaks nothing); **AUTH-002 reserved** after checking the spec, open changes, and the archive; no code or tests written before STOP#1 |
| `/hsi-change rename SessionStore.get to SessionStore.lookup` | **PATCH × LOCAL**; Gate 2 **stops** with a breaking-change report listing the call sites in both session test files; `auth/login.py`'s `users.get` correctly recognized as a dict call; no files edited |
| `/hsi-finish <slug>` on a change whose `tasks.md` has any unticked entry | **refuses**, names the entry, and does **not** tick it — even when the work is visibly done |
| `/hsi what does trim do?` | answers directly and routes nothing |
| `/hsi clean up auth` | asks one question offering `/hsi-change`, `/hsi-trim`, and `/hsi-audit` |
| `/hsi audit the project for security issues` | names `/hsi-audit`, explains it reads the whole project, and asks before running it |

A full DELTA change is best run interactively, answering both stops yourself. Watch for: delta
written with GIVEN/WHEN/THEN scenarios and a reserved ID; tests written and **failing** before the
code; the single-file test command used while iterating; `tasks.md` ticked as work lands; STOP#2
listing any delta edits; finish reading `finish.md`, merging the requirement into `auth.md`,
archiving the change, and appending a ledger **table row**.

### Loading a working copy

To test uncommitted plugin changes instead of the installed release, add these flags, which load
the working copy and switch the installed copy off for that run so the two cannot mix:

```bash
--plugin-dir /path/to/hsi \
--settings '{"enabledPlugins":{"hybrid-spec-intent@hsi":false}}' \
--allowedTools "Read(//path/to/hsi/**)"
```

The `Read` rule is needed because the plugin's templates live outside the demo project.

## 5. Cleanup

```bash
rm -rf ~/hsi-demo
```

Or keep it: `main` is the clean baseline and `seeded-issues` is ready for the next run.

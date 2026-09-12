---
name: hsi-audit
description: Adversarial sweep for spec-to-code drift, annotation rot, dead weight, and security flaws. Reports findings by severity and fixes only what the user selects.
---

Invoke the `hybrid-spec-intent` skill in audit mode.

This is the judgment-level counterpart to reconcile's structural checks. Reconcile catches what
a grep can prove; this catches what only reading can see. Read commands from the map's
*Stack & Commands* section — never hardcode a toolchain.

## Phase 1 — Annotation rot

Structural checks 2, 4, 5 and 6 run at reconcile and are assumed clean. Hunt the cases they
cannot see:

- **Hollow wrappers** — an annotated entry point that only delegates. The behavior moved during
  a refactor and the annotation was left behind, so the annotation now points at a shell. The
  test suite stays green through this, which is why it needs a human eye.
- **Semantic staleness** — the annotation cites a real ID, but the requirement was modified and
  the code no longer satisfies it. Read each changed requirement against its annotated code.
- **Drift down the call graph** — a single ID with annotations scattered across helpers, so no
  single entry point is identifiable.
- **Unresolved shims** — anything still marked TEMPORARY in the codebase or in a `tasks.md`
  under `changes/`. A shim that outlived its change is either debt or an undocumented adapter.

## Phase 2 — Spec coverage

- Requirements with no code annotation (advisory check 5) — tested, but the implementation
  cannot be located.
- Capabilities whose spec is still a skeleton while their code has grown substantially.
- Behavior visible in the code that no requirement describes. Report these; do not
  reverse-engineer requirements to fill the gap.

## Phase 3 — Dead weight & decay

- Orphaned modules, unreferenced exports, functions and variables never called.
- Imports with no remaining use; dependencies in the manifest nothing imports.

## Phase 4 — Security

- Run the project's own static analysis and dependency audit, per the map.
- Hardcoded credentials or API keys; secrets in source or config committed to the repo.
- Unencrypted storage of sensitive data; unsafe deserialization; injection-prone string-built
  queries.

## Phase 5 — Interactive resolution

**Do not modify any code automatically.**

Present a **🛡️ HSI AUDIT REPORT** grouped by severity — High Vulnerability, Spec Drift, Code
Decay, Dead Weight — with a file:line for every finding.

Ask which findings to resolve. On confirmation, fix only those, and route each fix through the
workflow: a fix that changes behavior is a DELTA and needs a delta spec, not a silent edit.
Annotation repairs and dead-code removal are PATCH.

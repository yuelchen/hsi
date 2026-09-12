---
name: hsi-audit
description: Full-project audit — spec-to-code drift across every capability, spec coverage gaps, and a project-wide security sweep. Reports by severity and changes nothing until the user picks findings.
---

Invoke the `hybrid-spec-intent` skill in audit mode.

## Scope

**The whole project, every time:** every capability spec, all source and test code, and the
dependency manifest. There is no narrowed mode. Reviewing a single change's files is
`/hsi-finish`'s job; dead code and repetition are `/hsi-trim`'s.

This is one of the two sanctioned exceptions to the skill's context budget — it reads everything,
because drift across capabilities is only visible from everywhere at once. Run it periodically
(before a release, after a long stretch of changes, when taking over a codebase), never per
change.

Read commands from the map's *Stack & Commands* section — never hardcode a toolchain.

## Phase 1 — Structural checks, project-wide

**Do not assume these are clean.** `/hsi-finish` runs them only over one change's files, so code
that never went through a finish has never been checked — including everything that existed
before `/hsi-setup` in a brownfield project.

Run across the entire project:

- **Check 2** — every `@spec` annotation resolves to a requirement ID that exists. At project
  scope this also covers check 4: a reference to a retired ID is simply one that does not
  resolve.
- **Check 3** — every requirement has at least one test citing it.
- **Check 5** — every requirement has at least one code annotation.
- **Check 6** — no requirement ID carries more than one non-test annotation.

## Phase 2 — Spec drift

Examine **every requirement in every capability spec**. There is no "changed since" reference
point — a project-wide audit reads all of them.

- **Semantic staleness** — read each requirement's scenarios against its annotated code. The
  annotation cites a real ID, but does the code still do what the scenarios say?
- **Hollow wrappers** — an annotated entry point that only delegates; the behavior moved and the
  annotation stayed behind.
- **Drift down the call graph** — one ID with annotations scattered across helpers, so no single
  entry point is identifiable.
- **Unresolved shims** — anything marked TEMPORARY in the codebase, and shim entries in any open
  change's `tasks.md`. A shim that outlived its change is either debt or an undocumented adapter.

## Phase 3 — Spec coverage

- Capabilities whose spec is still a skeleton while their code has grown substantially.
- Behavior visible in the code that no requirement describes. Report these; do not
  reverse-engineer requirements to fill the gap.
- Code under no capability's path glob in the map's Capability Index.

## Phase 4 — Security

- Run the project's own static analysis and dependency audit, per the map.
- Hardcoded credentials or API keys; secrets in source or committed config.
- Unencrypted storage of sensitive data; unsafe deserialization; injection-prone string-built
  queries.

## Phase 5 — Report and resolution

**Do not modify anything automatically.**

Present a **🛡️ HSI AUDIT REPORT** grouped by severity — High Vulnerability, Spec Drift,
Structural, Coverage Gaps — with a file:line for every finding. If dead code is noticed in
passing, mention it once and point to `/hsi-trim`; do not hunt for it.

Ask which findings to resolve. On confirmation, route each through the workflow:

- A fix that changes behavior is **DELTA** and needs a delta spec, not a silent edit.
- Annotation repairs are **PATCH**.
- Specifying behavior that already exists goes through a delta as an ADDED requirement, so it
  gets a stable ID and a test — even though no code changes.

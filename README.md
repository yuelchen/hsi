# Hybrid Spec-Intent (HSI)

A fail-fast spec-driven development plugin for Claude Code.

**Ceremony scales with blast radius. Context load scales with the surface touched — not with the
size of the project.**

## Why

Spec-driven workflows charge two costs: the **ceremony** a change must walk, and the **context**
the agent must load to walk it. Existing approaches hold one of them fixed.

| | ceremony per change | context per change | spec→test→code linkage |
|---|---|---|---|
| [OpenSpec](https://github.com/Fission-AI/OpenSpec) | low — write a delta, not a spec set | low | none |
| [LID](https://github.com/jszmajda/lid) | fixed six phases, a stop at each | grows with the project | strong (`@spec`) |
| **HSI** | **scales with blast radius** | **scales with surface touched** | **strong** |

HSI takes OpenSpec's artifact economics — a small delta per change, reconciled and archived — and
LID's linkage — stable requirement IDs, `@spec` annotations, coherence verification. It drops
OpenSpec's unlinked specs and LID's stop-at-every-phase.

A typo fix writes no spec and stops for nothing. A new feature writes a ~20-line delta, stops
once before any code exists and once when it works. An architectural change stops harder. You
never load the whole project to change one line of it.

## Three gates

Each sits at the cheapest point where the change can be shown wrong.

1. **Classify** — assign a tier reading *only* the context map. Unclassifiable means
   architectural: fail *toward* ceremony.
2. **Contract pre-flight** — does this break a public signature, schema, wire format, config key,
   or a contract another capability consumes? **Stop**, name the broken callers, and wait. This
   is the failure that is cheapest to catch here and most expensive to catch later.
3. **Red test** — the test must exist *and fail for the expected reason*. A test that passes
   before the code is written doesn't exercise the change; that's a finding about the test.

## Two axes

Ceremony comes from two independent measurements:

**Spec radius** — how much specification changes.

| tier | trigger | delta spec | stops |
|---|---|---|---|
| `PATCH` | no asserted behavior changes, ≤1 capability | no | none |
| `DELTA` | new or changed behavior, one capability | yes | 2 |
| `ARCHITECTURAL` | crosses capabilities, or changes the map | yes + map diff | 2 |

**Code radius** — how many existing call sites break: `CONTAINED`, `LOCAL`, `WIDE`. This decides
shims, back-port scope, and when the full suite runs.

They diverge, and that's the point. A pure rename changes **zero** specification but breaks fifty
files — `PATCH × WIDE`. No delta spec, full back-port. One axis can't express that.

Escalation is free and happens automatically. **De-escalation requires you to say so** — without
that asymmetry, everything drifts into `PATCH` and the workflow stops meaning anything.

## Artifacts

```
docs/hsi/
  project_context_map.md      ~80 lines, HARD CAP. Always loaded. Architecture only.
  capabilities/
    auth.md                   living spec — only the one you touch is ever loaded
    billing.md
  changes/
    add-sso/
      delta.md                ADDED / MODIFIED / REMOVED
      tasks.md                checklist, ticked as work lands; finish blocks while unticked
    archive/
      index.md                the ledger — one table row per change, grep only
      2026-09-12-add-sso/
```

The map's line cap is a mechanism, not a style note: it's what keeps per-change context load flat
as the project grows. When it overflows, detail gets promoted down into capability specs. The cap
never rises.

## Requirements

```markdown
### AUTH-003 — Session expiry

#### Scenario: Idle session expires
- **GIVEN** an authenticated session with no remember-me flag
- **WHEN** the session has been idle for 30 minutes
- **THEN** the next request returns 401 and the session record is deleted
```

IDs are stable and never reused. Tests and code both carry `// @spec AUTH-003`, at entry points
only — so *which specs does this file put at risk?* is answerable from the file in front of you,
without loading any spec.

## Commands

| | |
|---|---|
| `/hsi [request]` | **start here** — ask a question, or describe what you want and get routed to the right command |
| `/hsi-setup` | initialize — writes the map; for existing code, derives the capability index |
| `/hsi-change <slug>` | start a change deliberately |
| `/hsi-finish` | finish a change: review the files it touched, merge its delta into the capability spec, archive, run coherence checks |
| `/hsi-trim` | find repetition, missing abstractions, and dead code; apply approved simplifications without changing behavior |
| `/hsi-audit` | full-project audit: spec drift across every capability, coverage gaps, security |

The workflow is a **skill**, so it engages on any prompt that could change code — the commands
are for invoking it deliberately. Not sure which one you need? `/hsi` with a description announces
which command fits, and asks before any that write project files or read the whole project.

## Install

From inside a Claude Code session:

```
/plugin marketplace add yuelchen/hsi
/plugin install hybrid-spec-intent@hsi
```

Or from your shell:

```bash
claude plugin marketplace add yuelchen/hsi
claude plugin install hybrid-spec-intent@hsi
```

`marketplace add` also accepts a URL or a local path, so a clone works too:

```bash
claude plugin marketplace add ./path/to/hsi
```

**Let HSI read its own templates.** The plugin keeps its templates in its install folder, which is
outside your project, so Claude Code asks permission before reading them — and the read fails
anywhere it can't ask. Add this rule once to `~/.claude/settings.json`, merging it into an
existing `permissions.allow` list if you have one:

```json
{
  "permissions": {
    "allow": ["Read(~/.claude/plugins/cache/hsi/**)"]
  }
}
```

The `**` covers every installed version, so the rule keeps working across updates. Without it,
HSI still runs, but you'll be asked to approve each template read.

Then, in the project you want to use it on:

```
/hsi-setup
```

That writes `docs/hsi/`, the context map, and a short `## HSI` section in the project's
`CLAUDE.md`, so the workflow engages in every session. After that, just describe the change you
want — or use `/hsi` when you want a command deliberately.

To pull a newer version later:

```bash
claude plugin marketplace update hsi     # refresh the marketplace
claude plugin update hybrid-spec-intent  # restart Claude Code to apply
```

## Design

`docs/hsi-design.md` carries the full rationale — why each gate sits where it does, why the tier
axes split, and what HSI deliberately does not do.

## License

MIT

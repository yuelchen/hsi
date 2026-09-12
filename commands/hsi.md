---
name: hsi
description: Start here — ask a question about HSI, or describe what you want to do and get routed to the right HSI command.
argument-hint: [question, or what you want to do]
---

The user's request: $ARGUMENTS

This command is HSI's front door. It does exactly one of three things: **shows the menu**,
**answers a question about HSI**, or **routes the request to one HSI command**. It holds no
workflow rules of its own — gates, tiers, and procedures live in the `hybrid-spec-intent` skill
and the individual commands. Keep it that way.

## No request

If the request is empty, show this menu and stop:

| command | use it to |
|---|---|
| `/hsi-setup` | set HSI up in a project (once per project) |
| `/hsi-change <what>` | build, fix, or change something specific |
| `/hsi-finish [slug]` | finish an in-progress change: review it, merge its spec, archive it |
| `/hsi-trim` | find repetition and dead code, and simplify without changing behavior |
| `/hsi-audit` | audit the whole project: spec drift, coverage gaps, security |

Or describe what you want — `/hsi add session expiry` — and it will be routed.

## A question about HSI

If the request asks what HSI is, how it works, or what a command, tier, gate, or artifact does,
**answer it directly and do not route.** Draw on the `hybrid-spec-intent` skill for accuracy. Keep
the answer short, and end by naming the command that fits, if one does.

## Routing

| the request is about | route to | confirm first? |
|---|---|---|
| building, fixing, or changing something specific — a feature, a bug fix, a refactor the user has already chosen | `/hsi-change` | no |
| completing or wrapping up a change already in progress | `/hsi-finish` | no |
| setting up HSI in this project | `/hsi-setup` | **yes** |
| finding repetition, simplifying, removing dead or unused code | `/hsi-trim` | **yes** |
| auditing the project — spec drift, coverage, security | `/hsi-audit` | **yes** |

A request that is neither about HSI nor a change to the code — explaining a function, a general
question — is not HSI's business. Handle it normally without routing.

## Rules

1. **Announce the route before acting.** One line: *Routing to `/hsi-change` — this adds new
   behavior to auth.* The user learns the real command, and a wrong route is visible
   immediately.
2. **Confirm before the costly or sensitive routes.** `/hsi-setup` writes project files;
   `/hsi-trim` and `/hsi-audit` read the entire project. Say which command and why it costs more,
   and proceed only on a yes. `/hsi-change` and `/hsi-finish` go straight through — they carry
   their own safeguards.
3. **When the request is ambiguous, ask one question — never guess.** Offer two or three
   candidate commands, each with a one-line consequence. *"Clean up auth"* could be a behavior
   change (`/hsi-change`), a simplification (`/hsi-trim`), or a check for drift and security
   issues (`/hsi-audit`).
4. **Route to exactly one command.** If the request spans several — *"add SSO, then audit"* —
   route to the first and tell the user which command to run next. Never chain.
5. **Hand off completely.** Invoke the target through the Skill tool by its full name
   (`hybrid-spec-intent:hsi-change`, `hybrid-spec-intent:hsi-finish`, and so on), passing the
   user's request along as its arguments. From then on, follow that command entirely; `/hsi`
   adds nothing further.
6. **Stay thin.** Never restate gate, tier, or procedure logic here. If unsure how a command
   behaves, route to it rather than paraphrase it.

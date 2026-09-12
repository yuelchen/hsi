## 🤝 Contribution Guidelines

Contributions are welcomed! 

Please follow this workflow to submit your adjustments or additions.

### 1. Codebase Architecture Constraints
* **Manifest Control:** `.claude-plugin/plugin.json` carries core metadata only. Commands are
  auto-discovered from `commands/*.md` — do not add a command manifest to it. New plugin entries
  belong in `.claude-plugin/marketplace.json`.
* **The Skill Owns the Workflow:** gates, tier rules, flows, and the reconcile procedure live in
  `skills/hybrid-spec-intent/SKILL.md`. Long-form formats and templates go in its `references/`
  directory so they load only when needed.
* **Commands Stay Thin:** files in `commands/` invoke the skill in a named mode. They must not
  restate workflow logic — duplicated rules drift apart. Every command needs YAML frontmatter
  with `name` and `description`.
* **Toolchain Neutrality:** never hardcode a test, lint, or build command. Read them from the
  project's context map. Toolchain names may appear only in illustrative examples.
* **The Context Budget Is Load-Bearing:** changes that cause more to be loaded per change — a
  raised map cap, a new always-read file, reading the archive by default — defeat the plugin's
  purpose and need explicit justification in the PR.

### 2. Submission Steps
1. **Fork** this repository and pull it down to your machine.
2. Create a feature-focused branch detailing your upgrade (`git checkout -b feature/hsi-custom-command`).
3. Implement your changes within `skills/`, `commands/`, or `.claude-plugin/`.
4. Run through Local Integration Testing and Verification below to confirm the plugin loads and the workflow behaves.
5. Push your feature branch to your fork and submit a **Pull Request** detailing the token
   savings impact or security sweep improvement your changes provide.

### 3. Local Integration Testing

Install your working copy as a marketplace so edits are testable end to end. The path given to
`marketplace add` must contain `.claude-plugin/marketplace.json`, so run it from the repo root or
pass the repo path explicitly:

```bash
claude plugin marketplace add .          # from the repo root
```

Then install from the project you want to test *in* — not from this repo:

```bash
cd ~/some-scratch-project
claude plugin install hybrid-spec-intent@hsi -s local
```

**`-s local` matters.** Both `marketplace add` and `install` default to `user` scope, which makes
the plugin live in every project you open. `-s local` pins it to the project you run it from, so
a work-in-progress skill cannot follow you into unrelated work. The marketplace itself can stay
at user scope — it is only a source list.

Test in a scratch project rather than in this repo. HSI's skill description says *consult for ALL
code changes*, and so does LID's; with both enabled they compete on any prompt that touches code,
which makes it hard to tell whose behavior you are observing.

**Plugin files do not hot-reload.** After editing `SKILL.md`, a reference, or a command, pick
the working copy back up before testing:

```bash
claude plugin marketplace update hsi
claude plugin update hybrid-spec-intent
```

Then **restart Claude Code** — the update is not applied to a running session.

Confirm what actually loaded, rather than assuming:

```bash
claude plugin list                        # is it installed and enabled?
claude plugin details hybrid-spec-intent  # component inventory + projected token cost
```

`details` is the one to watch. It reports the projected token cost of what the plugin puts into
context, and HSI's entire thesis is that per-change context load stays flat. A PR that grows
`SKILL.md` substantially should say why in its description.

To remove a working copy when you are done:

```bash
claude plugin uninstall hybrid-spec-intent
claude plugin marketplace remove hsi
```

### 4. Verification

Before opening a PR:

```bash
claude plugin validate .                         # manifests, skills, agents, commands
head -1 commands/*.md skills/*/SKILL.md          # every file opens with ---
grep -rniE 'flutter|pytest|cargo|npm' commands/  # must return nothing
```

Then walk at least one change end to end at the tier your change affects. The DELTA path is the
most informative single walk — it is the only one that exercises both stops, the isolated
single-file test loop, and reconcile in one pass.

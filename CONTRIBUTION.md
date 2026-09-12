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

### 5. Releasing

`claude plugin update` compares **version numbers, not commits**. If `main` gets new commits but
the version stays the same, every installed copy reports `already at the latest version` and keeps
running the old code. **Every change to `skills/`, `commands/`, or the `.claude-plugin/` manifests
must bump the version before it reaches `main`.**

Changes only to `README.md`, `CONTRIBUTION.md`, or `docs/` do not change what the plugin does, so
they go to `main` without a version bump or a tag.

1. Bump `version` in **both** `.claude-plugin/plugin.json` and this plugin's entry in
   `.claude-plugin/marketplace.json`. While below 1.0:
   - **minor** (`0.2.0` → `0.3.0`) when a command is added, removed, or renamed, or when behavior
     changes in a way users will notice
   - **patch** (`0.2.0` → `0.2.1`) for fixes and wording changes
2. Commit, then confirm the two files agree — this refuses when they differ:
   ```bash
   claude plugin tag --dry-run
   ```
3. Merge to `main` and push.
4. Tag the release from `main` as `v<version>` and push the tag:
   ```bash
   git tag -a v<version> -m "hybrid-spec-intent <version>"
   git push origin v<version>
   ```
   Use plain `git tag` here, not `claude plugin tag --push` — that command always names tags
   `hybrid-spec-intent--v<version>`. Its `--dry-run` in step 2 is still the check that the two
   manifests agree.

Users then pick up the release with the update commands in the README, followed by a restart.

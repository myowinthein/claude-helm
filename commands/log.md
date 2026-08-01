---
description: Update CLAUDE.md with a full scan or gap update since last review
---

# log

## Before starting

`/helm:log` rewrites CLAUDE.md from the project's state, so it needs the full merged, stable state — never an in-progress feature branch. Behavior depends on `git-strategy` in CLAUDE.md's Project Config (absence defaults to GitHub Flow, per git.md).

**Solo Mode** (`git-strategy: solo`):
- Only proceed if on `main` or `master`. If on any other branch, stop: "log must be run on main or master in Solo Mode. Current branch is {branch} — switch and re-run."

**GitHub Flow** (`git-strategy: github-flow`, or absent):
- Record the current branch as `{original_branch}` — the workflow returns here at the end, whatever it was.
- Checkout a fresh branch from main's current tip: update local main (`git pull` if a remote exists), then `git checkout -b docs/log-{YYYYMMDD}` from it. This happens unconditionally — CLAUDE.md is always scanned from main's own state, never from a feature branch's unmerged work. If a feature branch changes something CLAUDE.md should reflect, re-run `/helm:log` after that branch merges to main.
- Call this branch `{branch}` for the rest of this command.
- If this command exits at any point without writing anything — the user selects Skip at any prompt, or CLAUDE.md is already up to date — delete `{branch}` and return to `{original_branch}` before exiting. This applies everywhere in this command, not just at the specific case called out later: never leave the user stranded on an empty temporary branch it created. (Gap Update's Outcome A does not qualify — it still bumps the last-reviewed hash, a real change that needs the normal commit path in Step 5.)

## Scope

CLAUDE.md = descriptive project knowledge (orientation layer).
.claude/rules/ = prescriptive rules (architecture, safety, git, testing).
Keep them consistent. Project-specific safety rules live in CLAUDE.md's `## Hard Safety Rules` section (written here, loaded with CLAUDE.md) — proposed, not auto-written. Never auto-edit the adopt-managed `.claude/rules` files (`git.md`, `safety.md`); they are overwritten by `/helm:adopt`, so propose any needed change to those instead.
Whenever writing to CLAUDE.md, in any step: use em-dashes sparingly — only when no other punctuation (comma, semicolon, colon, or a new sentence) works as well. When in doubt, restructure the sentence instead.

## CLAUDE.md Schema

CLAUDE.md uses these eight sections, in order. Both a full scan and a gap update work only within them — never add a new heading, and a finding that fits none of them does not belong in CLAUDE.md.

1. **Project Identity** — name, stack, purpose, blast radius.
2. **Project Config** — flag-style declarations read by `.claude/rules` (e.g. `git-strategy: solo`, `git-auto-commit: true`); keep the heading even if empty.
3. **Dev Commands** — install, run, test a single file, migrate, logs.
4. **Architecture Pointers** — key files and modules with a one-line why, not summaries.
5. **Domain Rules** — non-obvious business, lifecycle, and permission constraints a change could violate; one line each; write "None" if there are none.
6. **Behavior Rules** — development conventions, autonomy model, confirmation gates, test requirements.
7. **Hard Safety Rules** — this project's concrete never-do list: the real operational risks it carries (what `git push` deploys, which scripts are destructive, which environment is shared). Keep it concise; write "None" if there are no project-specific hazards.
8. **Known Traps** — gotchas; initially empty or inferred from README warnings.

A `## Rules` section, if present, is **not** part of this schema — it is written and managed by `/helm:adopt` and points at the adopted rule files (`.claude/rules/git.md`, `safety.md`), distinct from the inline Domain / Behavior / Hard Safety Rules above. Preserve it exactly as-is: a full scan or gap update must never write, move, or remove it.

## Noise commits

"Noise" means commits that carry no durable knowledge: bug fixes, styling, dependency updates, and routine CRUD. Both the gap assessment and the gap review skip them.

---

## Step 1 — Assessment

Check CLAUDE.md:
- Does it exist?
- Does it have content?
- Is there a saved commit hash? (look for `<!-- last-reviewed: {hash} -->`)
- If hash exists, run `git log {hash}..HEAD --oneline` to see the gap
- How significant is the gap? (ignore noise commits — see Noise commits above)
- If the file exists and has content, check whether all eight sections from the CLAUDE.md Schema (above) are present.

Schema is intact if all eight headings are found. Schema is broken if any are missing.

Based on current state, use AskUserQuestion (single-select) to present options:

If CLAUDE.md is absent, empty, or has no saved commit hash:
  AskUserQuestion:
    question: "{one sentence status, e.g. 'CLAUDE.md not found — a full scan is required.'}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Rewrite CLAUDE.md from a complete project scan"
      - label: "Skip"
        description: "No update needed"

If CLAUDE.md exists with a saved commit hash and schema is intact, and there are no meaningful commits since the hash (`git log {hash}..HEAD` is empty, or only noise commits):
  CLAUDE.md is already current. Inform the user: "CLAUDE.md is up to date — no meaningful commits since last review." Do not present any prompt. Proceed to Step 5 — it writes nothing but still needs to run its GitHub Flow cleanup (delete `{branch}`, return to `{original_branch}`) if a temporary branch was created.

If CLAUDE.md exists with a saved commit hash, and either schema is broken or there are meaningful commits since the hash:
  Determine recommendation:
  - If schema is broken (any required section missing): recommend Full scan regardless of gap size — state which sections are missing.
  - Otherwise (schema intact, meaningful commits exist): recommend Gap update for a small or moderate gap, or Full scan for a large or significant gap.
  List the recommended option first and append "(Recommended)" to it.

  AskUserQuestion:
    question: "{one sentence status and recommendation}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Gap update"
        description: "Update only sections affected by commits since last review{ — missing sections will not be added, if schema is broken}"
      - label: "Full scan"
        description: "Rewrite CLAUDE.md from a complete project scan{ to restore missing sections, if schema is broken}"
      - label: "Skip"
        description: "No update needed"

---

## Step 2 — Project Config Check

**Full scan:** always ask all applicable questions below.
**Gap update:** ask only the question(s) for flags missing from the existing Project Config section.
  If all applicable flags are already present, skip this step silently.

**Question 1 — Branching strategy** (ask if `git-strategy` is absent, or on full scan):

  AskUserQuestion:
    question: "Which git branching strategy should this project use?"
    header:   "Branching strategy"
    multiSelect: false
    options:
      - label: "Solo Mode"
        description: "Commit directly to main — no feature branches, no PRs. Intended for solo work with no collaborators."
      - label: "GitHub Flow"
        description: "Branch off main for every change, open a PR to merge back. Intended for teams or projects with peer review."

  Solo Mode selected → write `git-strategy: solo` under Project Config.
  GitHub Flow selected → write `git-strategy: github-flow` under Project Config.

**Question 2 — Auto-commit** (ask if `git-auto-commit` is absent, or on full scan):

  AskUserQuestion:
    question: "Should Claude commit automatically after completing each task?"
    header:   "Auto-commit"
    multiSelect: false
    options:
      - label: "Yes — commit automatically"
        description: "Claude commits after each task without prompting. Push still requires confirmation."
      - label: "No — ask before committing"
        description: "Claude asks for confirmation before every commit."

  Yes selected → write `git-auto-commit: true` under Project Config.
  No selected → omit `git-auto-commit` from Project Config.

**Question 3 — Merge strategy** (ask only if GitHub Flow is active or was just selected; ask if `git-merge-strategy` is absent, or on full scan):

  AskUserQuestion:
    question: "Which merge strategy should be used when merging pull requests into main?"
    header:   "Merge strategy"
    multiSelect: false
    options:
      - label: "Squash (Recommended)"
        description: "Squash all commits into one with a Conventional Commit message. Keeps main history clean and linear."
      - label: "Rebase"
        description: "Replay branch commits onto main without a merge commit. Linear history, preserves individual commits."
      - label: "Merge commit"
        description: "Create a merge commit. Preserves full branch topology and commit authorship."

  Squash selected → write `git-merge-strategy: squash` under Project Config.
  Rebase selected → write `git-merge-strategy: rebase` under Project Config.
  Merge commit selected → write `git-merge-strategy: merge` under Project Config.

---

## Step 3 — Full Project Scan

Before writing anything, investigate in this order to build the understanding for the eight schema sections:
1. Understand the business purpose of the application
2. Identify major modules and workflows
3. Identify technology stack and important versions
4. Identify major architectural patterns (from implementation, not folder names)
5. Identify high-level development conventions
6. Identify domain rules and business constraints
7. Identify operational context and common development workflows
8. Review existing docs, README, .claude/rules

Then write CLAUDE.md using the CLAUDE.md Schema above — all eight sections, in order.

Before writing the Project Config section, run Project Config Check.

CLAUDE.md is not: a README, a file listing, a code walkthrough,
a technical spec, or a changelog. Only include information a future
session would struggle to discover quickly from the codebase alone.

Do not create or modify .claude/rules during full scan unless explicitly
requested. Safety findings should be proposed, not auto-written.

At the end of the file, append:
`<!-- last-reviewed: {current HEAD commit hash} -->`

Be concise. Target under 150 lines. Do not pad.
Write directly — no approval needed.

---

## Step 4 — Gap Update

Before reviewing commits, run Project Config Check (Step 2) — skip silently if all flags are already present.

Read commit messages first to get the shape of what changed.
Then read file changes only for the significant commits — skip noise commits.

Focus on: architectural changes, new modules, new conventions, domain
rule changes, new operational knowledge, newly discovered traps.

**Stay within the CLAUDE.md Schema (above).** Every change must land in one of the eight existing sections — the schema says what each holds. Never add a new heading or section; if a finding fits none of the eight, it does not belong in CLAUDE.md — leave it out.

Before adding anything, apply the three-question filter:
1. Will a future session struggle to find this from the codebase?
2. Would knowing it improve future development decisions?
3. Will it stay true for weeks or months?

Only update if all three are yes. Durable knowledge includes:
architecture decisions, development conventions, domain knowledge
(business rules, lifecycle rules, permissions), operational knowledge,
and project traps. Does not include noise commits, plus refactors,
completed tasks, and temporary workarounds.

Only record what is supported by evidence — code, config, docs, or
repository structure. No assumptions, preferences, or speculation.

Prefer improving existing content over adding new content.
Merge overlapping entries, remove outdated ones, improve clarity first.

If a `.claude/rules` file looks out of date, **propose** the change — do not edit it directly (see Scope for why, and where safety findings go).

Update the saved commit hash at the end of the file to current HEAD.

Then provide one of two outcomes:

Outcome A — no update required
Brief explanation why no durable knowledge was introduced.

Outcome B — update required
Describe what was learned, why it belongs in project memory,
and the exact changes to make. Propose as a diff per section.
Ask for confirmation before writing.

---

## Step 5 — Commit and finalize

If nothing was written (the "CLAUDE.md is already up to date" case in Step 1, or the user selected Skip anywhere): skip commit, merge, and environment promotion below — there is nothing to act on. Under GitHub Flow, still delete `{branch}` and return to `{original_branch}` (per Before starting's cleanup rule) — do not skip that part. Either way, proceed to Step 6 to report the outcome.

Otherwise, commit per git.md's Auto-Commit rule: silent if `git-auto-commit: true`, otherwise ask for confirmation before committing, merging, and pushing together. Either way, push itself always requires its own confirmation regardless of `git-auto-commit` — git.md's Auto-Commit rule states this as an explicit exception, and rules/safety.md lists `git push` as always requiring confirmation with no exceptions. Environment promotion's branch-selection prompt below already serves as that confirmation for environment-branch pushes; under GitHub Flow, confirm separately before step 2's `git push origin main`, even when the commit above was silent.

**Environment promotion** — shared by both modes below. If environment branches exist (discover via `git branch -r`, filter for known environment names, same detection as ship.md), ask which should also receive this update. (This block is intentionally duplicated across every content-sync command — see rules/git.md's Environment Branches section for the canonical shape; keep all copies in sync if it changes.)

  AskUserQuestion:
    question: "main will be updated. Which environment branches should also receive this CLAUDE.md update?"
    header:   "Promote to environments"
    multiSelect: true
    options: one entry per discovered environment branch, e.g.:
      - label: "staging"
        description: "Merge main into staging"
      - label: "production"
        description: "Merge main into production"

  AskUserQuestion caps at 4 options. If more than 4 branches qualify, offer only the first 4 (recognized tier names first — staging/stage/uat/preprod, then production/prod, then any others alphabetically) and note in the question that remaining branches need a follow-up run.

  If exactly one environment branch qualifies, a multi-select with one option isn't valid — AskUserQuestion requires at least 2. Ask a direct yes/no confirmation instead:

    AskUserQuestion:
      question: "main will be updated. Promote to {branch} as well?"
      header:   "Promote to environment"
      multiSelect: false
      options:
        - label: "Yes — promote to {branch} (Recommended)"
          description: "Merge main into {branch}"
        - label: "Skip"
          description: "Leave {branch} as-is for now"

    "Yes" is equivalent to selecting {branch} in the multi-select above — proceed the same way.

  For each selected environment:
  - git checkout {environment}
  - git merge main --no-ff -m "chore(deploy): promote main to {environment} for CLAUDE.md update"
  - git push origin {environment}
  - git checkout main

  If no environment branches exist, or the user selects none, skip silently.

**Solo Mode:** commit directly to main. Before pushing, confirm:

  AskUserQuestion:
    question: "Push main now? This publishes the commit to origin."
    header:   "Push"
    multiSelect: false
    options:
      - label: "Push (Recommended)"
        description: "git push origin main"
      - label: "Cancel"
        description: "Leave main committed locally but unpushed — push manually when ready"

If Cancel selected → stop here. Do not push or promote. Proceed to Step 6 to report the outcome, noting main is committed locally but unpushed.

If Push selected: `git push origin main`, then run Environment promotion above.

**GitHub Flow:**
1. Commit on `{branch}`.
2. Merge into main: `git checkout main`, `git merge {branch} --no-ff -m "{same message as the commit above}"`. Before pushing, confirm:

   AskUserQuestion:
     question: "Push main now? This publishes the merged commit to origin."
     header:   "Push"
     multiSelect: false
     options:
       - label: "Push (Recommended)"
         description: "git push origin main"
       - label: "Cancel"
         description: "Leave main merged locally but unpushed — push manually when ready"

   If Cancel selected → stop here. Do not push, promote, or clean up. Proceed to Step 6 to report the outcome, noting main is merged locally but unpushed.

   If Push selected: `git push origin main`.
3. Run Environment promotion above.
4. Delete `{branch}`: `git branch -d {branch}` locally, and `git push origin --delete {branch}` if it was ever pushed.
5. Return to where you started: `git checkout {original_branch}`.

---

## Step 6 — Confirm completion

Report:

- Outcome: {up to date / full scan written / gap update written / skipped}
- Last-reviewed commit: {new hash, or unchanged if nothing written}
- Sections updated: {list, or none}
- Environments promoted: {list or none}
- Rule files worth revisiting: {list any `.claude/rules` files whose content looks stale relative to what this scan found, or none}
- If GitHub Flow: temporary branch's fate ({merged and deleted / left as-is}) and which branch you were returned to.


---
description: Update CLAUDE.md with a full scan or gap update since last review
---

# log

## Step 1 — Branch check

Before doing anything, check current branch.
Only proceed if on `main` or `master`.
If on any other branch, stop and inform the user:

"log must be run on main or master.
Current branch is {branch}. Please switch and re-run."

---

## Step 2 — Assessment

Check CLAUDE.md:
- Does it exist?
- Does it have content?
- Is there a saved commit hash? (look for `<!-- last-reviewed: {hash} -->`)
- If hash exists, run `git log {hash}..HEAD --oneline` to see the gap
- How significant is the gap? (ignore: bug fixes, styling, dependency updates, routine CRUD)
- If the file exists and has content, check whether all seven required sections are present:
  `## Project Identity`, `## Project Config`, `## Dev Commands`,
  `## Architecture Pointers`, `## Behavior Rules`, `## Hard Safety Rules`, `## Known Traps`

Schema is intact if all seven headings are found. Schema is broken if any are missing.

Based on current state, use AskUserQuestion (single-select) to present options:

If CLAUDE.md is absent or empty:
  AskUserQuestion:
    question: "{one sentence status, e.g. 'CLAUDE.md not found — a full scan is required.'}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Rewrite CLAUDE.md from a complete project scan"
      - label: "Skip"
        description: "No update needed"

If CLAUDE.md exists but has no saved commit hash:
  AskUserQuestion:
    question: "{one sentence status and recommendation}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Rewrite CLAUDE.md from a complete project scan"
      - label: "Skip"
        description: "No update needed"

If CLAUDE.md exists with a saved commit hash and schema is broken (any required section missing):
  Regardless of gap size, recommend Full scan. State which sections are missing.
  AskUserQuestion:
    question: "{one sentence stating which sections are missing and why full scan is recommended}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Rewrite CLAUDE.md from a complete project scan to restore missing sections"
      - label: "Gap update"
        description: "Update only commits since last review — missing sections will not be added"
      - label: "Skip"
        description: "No update needed"

If CLAUDE.md exists with a saved commit hash and schema is intact:
  Form a recommendation (Full or Gap) based on gap significance.
  Put the recommended option first.

  Gap is the recommendation (small or moderate gap):
  AskUserQuestion:
    question: "{one sentence status and recommendation}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Gap update (Recommended)"
        description: "Update only sections affected by commits since last review"
      - label: "Full scan"
        description: "Rewrite CLAUDE.md from a complete project scan"
      - label: "Skip"
        description: "No update needed"

  Full is the recommendation (large or significant gap):
  AskUserQuestion:
    question: "{one sentence status and recommendation}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Rewrite CLAUDE.md from a complete project scan"
      - label: "Gap update"
        description: "Update only sections affected by commits since last review"
      - label: "Skip"
        description: "No update needed"

---

## Step 3 — Project Config Check

**Full scan:** always ask both questions below.
**Gap update:** ask only the question(s) for flags missing from the existing Project Config section.
  If both flags are already present, skip this step silently.

**Question 1 — Branching strategy** (ask if `git-solo` is absent, or on full scan):

  AskUserQuestion:
    question: "Which git branching strategy should this project use?"
    header:   "Branching strategy"
    multiSelect: false
    options:
      - label: "Solo Mode"
        description: "Commit directly to main — no feature branches, no PRs. Intended for solo work with no collaborators."
      - label: "GitHub Flow"
        description: "Branch off main for every change, open a PR to merge back. Intended for teams or projects with peer review."

  Solo Mode selected → write `git-solo: true` under Project Config.
  GitHub Flow selected → omit `git-solo` from Project Config (absence means GitHub Flow).

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

---

## Step 4 — Full Project Scan

Before writing anything, investigate in this order:
1. Understand the business purpose of the application
2. Identify major modules and workflows
3. Identify technology stack and important versions
4. Identify major architectural patterns (from implementation, not folder names)
5. Identify high-level development conventions
6. Identify domain rules and business constraints
7. Identify operational context and common development workflows
8. Review existing docs, README, .claude/rules

Then write CLAUDE.md using the seven-section schema:
1. Project Identity (name, stack, purpose, blast radius)
2. Project Config (flag-style declarations read by .claude/rules, e.g. `git-solo: true`, `git-auto-commit: true`; keep heading even if empty)
3. Dev Commands (install, run, test single file, migrate, logs)
4. Architecture Pointers (key files with one-line why, not summaries)
5. Behavior Rules (autonomy model, confirmation gates, test requirements)
6. Hard Safety Rules (invariants, never-do list — keep brief, detail in .claude/rules/safety.md)
7. Known Traps (initially empty or inferred from README warnings)

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

## Step 5 — Gap Update

Before reviewing commits, run Project Config Check (Step 3) — skip silently if all flags are already present.

Read commit messages first to get the shape of what changed.
Then read file changes only for significant commits — skip: bug fixes,
styling, dependency updates, routine CRUD.

Focus on: architectural changes, new modules, new conventions, domain
rule changes, new operational knowledge, newly discovered traps.

Before adding anything, apply the three-question filter:
1. Will a future session struggle to find this from the codebase?
2. Would knowing it improve future development decisions?
3. Will it stay true for weeks or months?

Only update if all three are yes. Durable knowledge includes:
architecture decisions, development conventions, domain knowledge
(business rules, lifecycle rules, permissions), operational knowledge,
and project traps. Does not include: bug fixes, refactors, styling,
dependency updates, routine CRUD, completed tasks, temporary workarounds.

Only record what is supported by evidence — code, config, docs, or
repository structure. No assumptions, preferences, or speculation.

Prefer improving existing content over adding new content.
Merge overlapping entries, remove outdated ones, improve clarity first.
Apply the same review to .claude/rules files.

Update the saved commit hash at the end of the file to current HEAD.

Then provide one of two outcomes:

Outcome A — no update required
Brief explanation why no durable knowledge was introduced.

Outcome B — update required
Describe what was learned, why it belongs in project memory,
and the exact changes to make. Propose as a diff per section.
Ask for confirmation before writing.

---

## Scope

CLAUDE.md = descriptive project knowledge (orientation layer).
.claude/rules/ = prescriptive rules (architecture, safety, git, testing).
Keep them consistent. Update rule files when conventions change.


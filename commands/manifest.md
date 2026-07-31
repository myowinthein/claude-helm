---
description: Update README.md with a full scan or gap update since last review
---

# manifest

## Before starting

`/helm:manifest` rewrites README.md from the project's state, so it needs the full merged, stable state — never an in-progress feature branch. Behavior depends on `git-strategy` in CLAUDE.md's Project Config (absence defaults to GitHub Flow, per git.md).

**Solo Mode** (`git-strategy: solo`):
- Only proceed if on `main` or `master`. If on any other branch, stop: "manifest must be run on main or master in Solo Mode. Current branch is {branch} — switch and re-run."

**GitHub Flow** (`git-strategy: github-flow`, or absent):
- Record the current branch as `{original_branch}` — the workflow returns here at the end, whatever it was.
- Checkout a fresh branch from main's current tip: update local main (`git pull` if a remote exists), then `git checkout -b docs/manifest-{YYYYMMDD}` from it. This happens unconditionally — README.md is always scanned from main's own state, never from a feature branch's unmerged work. If a feature branch changes something README.md should reflect, re-run `/helm:manifest` after that branch merges to main.
- Call this branch `{branch}` for the rest of this command.

---

## Scope

README.md = human-facing documentation (contributors, GitHub visitors, new users).
Not a changelog. Not a technical spec. Not a deployment manual.
Audience is humans, not future Claude sessions — keep it clear and scannable.
Whenever writing to README.md, in any step: use em-dashes sparingly — only when no other punctuation (comma, semicolon, colon, or a new sentence) works as well. When in doubt, restructure the sentence instead.

---

## Step 1 — Assessment

**Determine readme-style:**

Check CLAUDE.md Project Config for `readme-style`.

If `readme-style` is absent:
  Check whether README.md exists and has content — the question below depends on it.

  If README.md exists with content:
    AskUserQuestion:
      question: "README.md exists with no known style. Which style should this project use?"
      header:   "README style"
      multiSelect: false
      options:
        - label: "Standard Readme spec"
          description: "Enforce the Standard Readme spec structure when writing or updating."
        - label: "Custom style"
          description: "Follow the existing README structure — never rewrite into the spec."

  If README.md is absent or empty (nothing to be "custom" relative to yet):
    AskUserQuestion:
      question: "No README.md yet. Which style should this project use?"
      header:   "README style"
      multiSelect: false
      options:
        - label: "Standard Readme spec"
          description: "Write README.md following the Standard Readme spec structure."
        - label: "Custom style"
          description: "No existing structure to follow yet — you write README.md yourself in whatever structure you prefer; once it exists, future runs preserve it rather than rewrite it."

  Write the chosen value (`readme-style: standard` or `readme-style: custom`) to CLAUDE.md Project Config before continuing.

If `readme-style` is already set → use it as-is. Do not ask again.

---

**Assess README.md:**

Check README.md:
- Does it exist?
- Does it have content?
- Is there a saved commit hash? (look for `<!-- last-reviewed: {hash} -->`)
- If hash exists, run `git log {hash}..HEAD --oneline` to see the gap
- How significant is the gap? (ignore: bug fixes, styling, dependency updates, routine CRUD)
- If `readme-style: standard` and the file exists and has content, check whether all mandatory sections are present:
  `Title`, `Short Description`, `Install`, `Usage`, `Contributing`, `License`

  Structure is intact if all mandatory sections are found. Structure is broken if any are missing.

Based on current state, use AskUserQuestion (single-select) to present options:

If README.md is absent or empty:
  AskUserQuestion:
    question: "{one sentence status, e.g. 'README.md not found — a full scan is required.'}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Write README.md from a complete project scan"
      - label: "Skip"
        description: "No update needed"

If README.md exists but has no saved commit hash:
  AskUserQuestion:
    question: "{one sentence status and recommendation}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Rewrite README.md from a complete project scan"
      - label: "Skip"
        description: "No update needed"

If README.md exists with a saved commit hash:
  Determine recommendation:
  - If `readme-style: standard` and structure is broken (any mandatory section missing): recommend Full scan regardless of gap size — state which sections are missing.
  - Otherwise: recommend Gap update for a small or moderate gap, or Full scan for a large or significant gap.
  List the recommended option first and append "(Recommended)" to it.

  AskUserQuestion:
    question: "{one sentence status and recommendation}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Gap update"
        description: "Update only sections affected by commits since last review{ — missing sections will not be added, if structure is broken}"
      - label: "Full scan"
        description: "Rewrite README.md from a complete project scan{ to restore missing sections, if structure is broken}"
      - label: "Skip"
        description: "No update needed"

---

## Step 2 — Full Project Scan

If `readme-style: custom` and README.md is absent or empty: skip investigation entirely — there is nothing to write. Do not write anything — custom mode preserves structure, it does not author it. Report that there is no README.md yet, so there is nothing to be "custom" relative to, and ask the developer to write one in whatever structure they prefer. A future run will then preserve that structure. Stop here — do not proceed to Step 4.

Otherwise, before writing anything, investigate:
- Business purpose and target audience
- Technology stack and major dependencies
- Installation requirements and steps
- Core usage patterns and CLI commands
- Public API surface (if any)
- License, contributing model, maintainers
- Project motivation or history (if non-obvious) and any meaningful limitations worth surfacing

If `readme-style: standard`:
  Write README.md following the Standard Readme spec section order.
  Include only sections relevant to this project — do not force all sections.

  Mandatory sections (always include):
  - Title (must match repo/package name)
  - Short Description (under 120 chars, matches package.json description field)
  - Table of Contents (if README will exceed 100 lines)
  - Install
  - Usage
  - Contributing
  - License (must be last section before the comment tag)

  Include when relevant:
  - Badges (CI, version — keep minimal)
  - Long Description (only if short description is insufficient)
  - Background (useful if the "why" behind the project is non-obvious)
  - API (if the project has a public interface)
  - Limitations (only if the project has meaningful limitations worth surfacing to users evaluating adoption)

  Skip unless specifically needed:
  - Banner, Security, Thanks, Maintainers, Extra Sections

  Sections must appear in the order listed by the spec.
  Do not invent sections outside the spec.

If `readme-style: custom` (README.md already exists with content — the only way to reach this point in custom mode, per the early exit above):
  Read the existing README structure before writing.
  Preserve the existing section order and naming.
  Rewrite content within each section based on the project scan.
  Do not add, remove, or rename sections without explicit approval.

At the end of the file, append:
`<!-- last-reviewed: {current HEAD commit hash} -->`

Write directly — no approval needed.

---

## Step 3 — Gap Update

Read commit messages first to get the shape of what changed.
Then read file changes only for significant commits — skip: bug fixes,
styling, dependency updates, routine CRUD.

Focus on: new features, API changes, new install steps, changed usage,
changed CLI commands, new env vars, removed functionality.

For each significant change, identify which README section is affected.
Update only those sections. Do not rewrite unaffected sections.

If `readme-style: standard`:
  Sections must remain in spec order after updates.
  If a significant change makes an optional section newly relevant (per Step 2's "include when relevant" list — e.g. a first public API, a newly meaningful limitation), add it in its spec-ordered position. If a change removes what justified an existing optional section (e.g. the public API is removed), remove that section. Do not invent sections outside the spec.
If `readme-style: custom`:
  Preserve existing section order and naming.
  Do not add, remove, or rename sections without explicit approval — matching Step 2.
  If a significant change does not fit any existing section, ask whether to add a new section (and where) or fold it into the closest existing one; do not decide silently.

Update the saved commit hash at the end of the file to current HEAD.

Propose changes per affected section. Ask for confirmation before writing.

---

## Step 4 — Commit and finalize

Skip this step entirely if Step 2 wrote nothing (the custom-mode, absent-README case) — there is nothing to commit, merge, or promote.

Commit per git.md's Auto-Commit rule — this also governs whether the sequence below needs confirmation before proceeding: silent if `git-auto-commit: true`, otherwise one confirmation covers commit, merge, promotion, and cleanup together.

**Environment promotion** — shared by both modes below. If environment branches exist (discover via `git branch -r`, filter for known environment names, same detection as ship.md), ask which should also receive this update:

  AskUserQuestion:
    question: "main will be updated. Which environment branches should also receive this README.md update?"
    header:   "Promote to environments"
    multiSelect: true
    options: one entry per discovered environment branch, e.g.:
      - label: "staging"
        description: "Merge main into staging"
      - label: "production"
        description: "Merge main into production"

  For each selected environment:
  - git checkout {environment}
  - git merge main --no-ff -m "chore(deploy): promote main to {environment} for README.md update"
  - git push origin {environment}
  - git checkout main

  If no environment branches exist, or the user selects none, skip silently.

**Solo Mode:** commit directly to main, then run Environment promotion above.

**GitHub Flow:**
1. Commit on `{branch}`.
2. Merge into main: `git checkout main`, `git merge {branch} --no-ff -m "{same message as the commit above}"`, `git push origin main`.
3. Run Environment promotion above.
4. Delete `{branch}`: `git branch -d {branch}` locally, and `git push origin --delete {branch}` if it was ever pushed.
5. Return to where you started: `git checkout {original_branch}`.

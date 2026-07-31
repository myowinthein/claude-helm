---
description: Update README.md with a full scan or gap update since last review
---

# manifest

## Before starting

`/helm:manifest` rewrites README.md from the project's state, so it needs the full merged state — otherwise it captures a partial branch and collides with other branches' updates at merge:
- On `main` or `master` → proceed.
- On another branch → proceed only if it is up-to-date with main (main is an ancestor of `HEAD`, so no merged work is missing). Check `git merge-base --is-ancestor <main> HEAD` (use `origin/main` when a remote exists).
- If the branch is behind main → stop: "manifest needs the current main state. {branch} is behind main — merge or rebase main in first, then re-run."

---

## Step 1 — Assessment

**Determine readme-style:**

Check CLAUDE.md Project Config for `readme-style`.

If `readme-style` is absent:
  AskUserQuestion:
    question: "README.md exists with no known style. Which style should this project use?"
    header:   "README style"
    multiSelect: false
    options:
      - label: "Standard Readme spec"
        description: "Enforce the Standard Readme spec structure when writing or updating."
      - label: "Custom style"
        description: "Follow the existing README structure — never rewrite into the spec."

  Write the chosen value (`readme-style: standard` or `readme-style: custom`) to CLAUDE.md Project Config before continuing.

If `readme-style` is already set → use it as-is. Do not ask again.

---

**Assess README.md:**

Check README.md:
- Does it exist?
- Does it have content?
- Is there a saved commit hash? (look for `<!-- last-reviewed: {hash} -->`)
- If hash exists, verify it still resolves (`git cat-file -e {hash}^{commit}`) — a squash-merged branch's hash becomes unreachable once the branch is deleted. If it resolves, run `git log {hash}..HEAD --oneline` to see the gap. If it doesn't, treat README.md as having no saved hash.
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

If README.md exists with a saved commit hash and `readme-style: standard` and structure is broken (any mandatory section missing):
  Regardless of gap size, recommend Full scan. State which sections are missing.
  AskUserQuestion:
    question: "{one sentence stating which sections are missing and why full scan is recommended}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Rewrite README.md from a complete project scan to restore missing sections"
      - label: "Gap update"
        description: "Update only commits since last review — missing sections will not be added"
      - label: "Skip"
        description: "No update needed"

If README.md exists with a saved commit hash and (structure is intact or `readme-style: custom`):
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
        description: "Rewrite README.md from a complete project scan"
      - label: "Skip"
        description: "No update needed"

  Full is the recommendation (large or significant gap):
  AskUserQuestion:
    question: "{one sentence status and recommendation}"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Rewrite README.md from a complete project scan"
      - label: "Gap update"
        description: "Update only sections affected by commits since last review"
      - label: "Skip"
        description: "No update needed"

---

## Step 2 — Full Project Scan

Before writing anything, investigate:
- Business purpose and target audience
- Technology stack and major dependencies
- Installation requirements and steps
- Core usage patterns and CLI commands
- Public API surface (if any)
- License, contributing model, maintainers

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

If `readme-style: custom`:
  Read the existing README structure before writing.
  Preserve the existing section order and naming.
  Rewrite content within each section based on the project scan.
  Do not add, remove, or rename sections without explicit approval.

At the end of the file, append:
`<!-- last-reviewed: {current HEAD commit hash} -->`

Use em-dashes sparingly — only when no other punctuation (comma, semicolon, colon, or a new sentence) works as well. When in doubt, restructure the sentence instead.
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

If `readme-style: standard`: sections must remain in spec order after updates.
If `readme-style: custom`: preserve existing section order and naming.

Update the saved commit hash at the end of the file to current HEAD.

Propose changes per affected section. Ask for confirmation before writing.

---

## Scope

README.md = human-facing documentation (contributors, GitHub visitors, new users).
Not a changelog. Not a technical spec. Not a deployment manual.
Audience is humans, not future Claude sessions — keep it clear and scannable.

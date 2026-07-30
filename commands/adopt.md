---
description: Install or update helm rule files (git.md, safety.md) into the current project
---

# adopt

Install or update the helm rule files (`git.md`, `safety.md`) into the current project. Setup helper, not a workflow command.

## Step 1 — Sanity check

Check whether the current directory looks like a project — any of:
- `.git/`
- `CLAUDE.md`
- `package.json`, `composer.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`

Decide based on what is present:
- **Project markers found** → proceed.
- **Empty directory** (no project markers and no other files) → proceed silently. This is a plausible fresh-project setup with nothing to clobber.
- **Non-empty but no project markers** → this is the suspicious case (a populated folder that isn't a project — possibly the wrong directory). Confirm before writing:

  AskUserQuestion:
    question: "This directory has files but does not look like a project (no .git, no CLAUDE.md, no recognised manifest). Continue anyway?"
    header:   "Confirm"
    multiSelect: false
    options:
      - label: "Continue"
        description: "Adopt rules here anyway"
      - label: "Cancel"
        description: "Exit without changes"

  If Cancel → exit.

## Step 2 — Scan existing rules

First detect the CLAUDE.md reference type for each rule, one of:
- **marketplace ref** — a line with `marketplaces/claude-helm/rules/{name}` is present
- **local ref** — a line with `.claude/rules/{name}` is present
- **no ref** — neither is present

Then classify each of `.claude/rules/git.md` and `.claude/rules/safety.md` as one of:
- **Helm-marked** — file exists and starts with `<!-- helm-rule: claude-helm@v{X.Y.Z} -->`; record version
- **Foreign** — file exists but does not contain the helm marker
- **Referenced** — file does not exist and a marketplace ref is present
- **Absent** — file does not exist and no marketplace ref is present (a stray local ref does not block this — Step 5 rewrites it)

Classification is driven by the file first; the reference type only disambiguates the file-absent cases (Referenced vs Absent) and is reconciled in Step 5.

Compute the overall state:
- All Absent → state = FRESH
- Any Referenced, none Helm-marked, none Foreign → state = REFERENCED
- Any Helm-marked, none Foreign → state = UPDATE
- Any Foreign → state = CONFLICT

Read the currently installed helm version from `~/.claude/plugins/marketplaces/claude-helm/.claude-plugin/plugin.json`. Record as `current_version`.

## Step 3 — Show the scan summary

Display:

```
.claude/rules/git.md     {Absent / Helm v{X.Y.Z} / Foreign / Referenced in CLAUDE.md}
.claude/rules/safety.md  {Absent / Helm v{X.Y.Z} / Foreign / Referenced in CLAUDE.md}
Installed helm version:  v{current_version}
```

## Step 4 — Choose install mode

Choose the question and labels based on state.

If state = REFERENCED:
  AskUserQuestion:
    question: "Helm rules are already referenced in CLAUDE.md (reference mode). What would you like to do?"
    header:   "Already installed"
    multiSelect: false
    options:
      - label: "Keep references (no change)"
        description: "References point to the marketplaces path and stay current after /plugin update."
      - label: "Switch to copy mode"
        description: "Copy versioned rule files into .claude/rules/ and repoint the CLAUDE.md references at them."
      - label: "Cancel"
        description: "Exit without changes."

If state = FRESH:
  AskUserQuestion:
    question: "How should helm install the rules into this project?"
    header:   "Install mode"
    multiSelect: false
    options:
      - label: "Copy into .claude/rules/ (Recommended)"
        description: "Self-contained. Rules get committed with your project and travel with it."
      - label: "Reference from CLAUDE.md"
        description: "Lighter footprint. Pulls latest after /plugin update, but references are machine-local."
      - label: "Cancel"
        description: "Exit without changes."

If state = UPDATE:
  AskUserQuestion:
    question: "Update the helm rules in this project to v{current_version}?"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Update rules in .claude/rules/ (Recommended)"
        description: "Overwrite helm-marked files with the version from the installed plugin."
      - label: "Switch to reference mode"
        description: "Delete the copied files and reference the installed plugin path from CLAUDE.md instead."
      - label: "Cancel"
        description: "Exit without changes."

If state = CONFLICT:
  AskUserQuestion:
    question: "Existing rule files without the helm marker were found. How should helm proceed?"
    header:   "Conflict"
    multiSelect: false
    options:
      - label: "Review per file (Recommended)"
        description: "For each foreign file, ask whether to overwrite, skip, or show the diff."
      - label: "Reference from CLAUDE.md instead"
        description: "Leave existing files alone, point CLAUDE.md at the installed plugin path."
      - label: "Cancel"
        description: "Exit without changes."

Wait for response.

## Step 5 — Execute

### Copy or Update path

- Create `.claude/rules/` if it does not exist.
- For each of `git.md`, `safety.md`:
  - Read the source from `~/.claude/plugins/marketplaces/claude-helm/rules/{name}`.
  - Prepend a marker line: `<!-- helm-rule: claude-helm@v{current_version} -->`
  - Write to `.claude/rules/{name}`.
- Point CLAUDE.md at the copied rules so they load as context. Detect an existing helm `## Rules` section — entries pointing at `.claude/rules/` or the marketplaces path — and set its entries to the local paths below; create the section if absent. This replaces any marketplace-path or stale entries (e.g. left by reference mode) rather than appending alongside them:
  ```
  ## Rules

  This project follows the rules shipped in claude-helm:
  - .claude/rules/git.md
  - .claude/rules/safety.md

  At the start of every session, read the rule files above and follow them.
  ```
  If CLAUDE.md does not exist, create it with just this section.
- For the CONFLICT/Review path: for each Foreign file, use AskUserQuestion with options: Overwrite, Skip, Show diff. Loop the prompt after Show diff so the user can still pick Overwrite or Skip.

### Reference path

- Use the marketplaces install path: `~/.claude/plugins/marketplaces/claude-helm/rules/`. This path always reflects the latest installed version and updates automatically after `/plugin update helm@claude-helm`.
- If `CLAUDE.md` exists:
  - Detect an existing helm `## Rules` section — entries pointing at `.claude/rules/` or the marketplaces path — and set its entries to the marketplace paths below; create the section if absent. This replaces any local-path entries left by copy mode rather than appending alongside them.
    ```
    ## Rules

    This project follows the rules shipped in claude-helm:
    - ~/.claude/plugins/marketplaces/claude-helm/rules/git.md
    - ~/.claude/plugins/marketplaces/claude-helm/rules/safety.md

    At the start of every session, read the rule files at the paths above and follow them.
    If either path is missing, inform the user: "helm rules are referenced in CLAUDE.md but the
    plugin is not installed on this machine. Install it with: /plugin install claude-helm"
    ```
  - Ask for confirmation before writing.
- If `CLAUDE.md` does not exist:
  - Ask for confirmation before creating it:
    AskUserQuestion:
      question: "CLAUDE.md does not exist. Create it with just the helm rule references?"
      header:   "Create CLAUDE.md"
      multiSelect: false
      options:
        - label: "Create CLAUDE.md (Recommended)"
          description: "Creates a minimal CLAUDE.md containing only the helm rules section."
        - label: "Skip"
          description: "Print the snippet to chat instead so you can place it manually."
  - If Create: write a new `CLAUDE.md` containing only the rules snippet:
    ```
    ## Rules

    This project follows the rules shipped in claude-helm:
    - ~/.claude/plugins/marketplaces/claude-helm/rules/git.md
    - ~/.claude/plugins/marketplaces/claude-helm/rules/safety.md

    At the start of every session, read the rule files at the paths above and follow them.
    If either path is missing, inform the user: "helm rules are referenced in CLAUDE.md but the
    plugin is not installed on this machine. Install it with: /plugin install claude-helm"
    ```
  - If Skip: print the snippet to the chat so the user can place it manually.

### Cancel path

- Exit silently. No files written.

## Step 6 — Report

For Copy/Update, verify the written files before reporting: read `.claude/rules/git.md` and `.claude/rules/safety.md` and confirm each file exists and contains the `<!-- helm-rule: claude-helm@v{current_version} -->` marker. If a file is missing or the marker is absent, report the error instead of ADOPT COMPLETE.

Print outcome.

For Copy/Update:
```
ADOPT COMPLETE
- git.md    {written / updated / skipped} → .claude/rules/git.md (v{current_version})
- safety.md {written / updated / skipped} → .claude/rules/safety.md (v{current_version})
- CLAUDE.md ## Rules section points at the copied files
```

For Reference:
```
ADOPT COMPLETE
- CLAUDE.md updated with references to claude-helm rules at v{current_version}
```

## Notes

- The marker line at the top of each copied file is required for future `/helm:adopt` runs to detect helm-managed files vs. user-managed files. Do not strip it.
- This command never writes outside `.claude/rules/` or `CLAUDE.md` in the current project.
- This command never touches anything in the user's `~/.claude/` directory.


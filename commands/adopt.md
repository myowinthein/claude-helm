---
description: Install or update helm rule files (git.md, safety.md) into the current project
---

# adopt

Install or update the helm rule files (`git.md`, `safety.md`) into the current project. Setup helper, not a workflow command.

## Step 1 — Sanity check

Confirm the current directory looks like a project. Check for any of:
- `.git/`
- `CLAUDE.md`
- `package.json`, `composer.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`

If none found, use AskUserQuestion:
  AskUserQuestion:
    question: "This directory does not look like a project (no .git, no CLAUDE.md, no recognised manifest). Continue anyway?"
    header:   "Confirm"
    multiSelect: false
    options:
      - label: "Continue"
        description: "Adopt rules here anyway"
      - label: "Cancel"
        description: "Exit without changes"

If Cancel → exit.

## Step 2 — Scan existing rules

For each of `.claude/rules/git.md` and `.claude/rules/safety.md`, record one of:
- **Absent** — file does not exist and not referenced in CLAUDE.md
- **Helm-marked** — file exists and starts with `<!-- helm-rule: claude-helm@v{X.Y.Z} -->`; record version
- **Foreign** — file exists but does not contain the helm marker
- **Referenced** — file does not exist but a matching `~/.claude/plugins/marketplaces/claude-helm/rules/{name}` path is present in CLAUDE.md

To detect Referenced: check whether CLAUDE.md contains a line with `marketplaces/claude-helm/rules/git.md` (or `safety.md`).

Compute the overall state:
- All Absent → state = FRESH
- Any Referenced, none Helm-marked, none Foreign → state = REFERENCED
- Any Helm-marked, none Foreign → state = UPDATE
- Any Foreign → state = CONFLICT

Read the currently installed helm version from `~/.claude/plugins/claude-helm/.claude-plugin/plugin.json` (or wherever the plugin install path resolves to). Record as `current_version`.

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
        description: "Copy versioned rule files into .claude/rules/ and remove the CLAUDE.md references."
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
- If switching from reference mode (REFERENCED → copy): also remove the helm reference lines from CLAUDE.md. If the `## Rules` section becomes empty after removal, remove the section heading too.
- For the CONFLICT/Review path: for each Foreign file, use AskUserQuestion with options: Overwrite, Skip, Show diff. Loop the prompt after Show diff so the user can still pick Overwrite or Skip.

### Reference path

- Use the marketplaces install path: `~/.claude/plugins/marketplaces/claude-helm/rules/`. This path always reflects the latest installed version and updates automatically after `/plugin update helm@claude-helm`.
- If `CLAUDE.md` exists:
  - Detect or create a `## Rules` section.
  - Append a list of absolute paths:
    ```
    ## Rules

    This project follows the rules shipped in claude-helm:
    - ~/.claude/plugins/marketplaces/claude-helm/rules/git.md
    - ~/.claude/plugins/marketplaces/claude-helm/rules/safety.md
    ```
  - Ask for confirmation before writing.
- If `CLAUDE.md` does not exist:
  - Print the snippet to the chat so the user can place it manually.
  - Inform the user that helm references work best with a CLAUDE.md present.

### Cancel path

- Exit silently. No files written.

## Step 6 — Report

Print outcome.

For Copy/Update:
```
ADOPT COMPLETE
- git.md    {written / updated / skipped} → .claude/rules/git.md (v{current_version})
- safety.md {written / updated / skipped} → .claude/rules/safety.md (v{current_version})
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

**Writing style**
- Use em-dashes sparingly. Only use one when no other punctuation
  (comma, semicolon, colon, or a new sentence) works as well.
  When in doubt, restructure the sentence instead.

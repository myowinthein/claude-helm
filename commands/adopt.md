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
- **Absent** — file does not exist and no marketplace ref is present (a stray local ref does not block this — Step 4 rewrites it)

Classification is driven by the file first; the reference type only disambiguates the file-absent cases (Referenced vs Absent) and is reconciled in Step 4.

Read the currently installed helm version from `~/.claude/plugins/marketplaces/claude-helm/.claude-plugin/plugin.json`. Record as `current_version`.

Compute the overall state:
- All Absent → state = FRESH
- Any Referenced, none Helm-marked, none Foreign → state = REFERENCED
- Any Helm-marked, none Foreign → state = UPDATE
- Any Foreign → state = CONFLICT

For UPDATE state, compare each Helm-marked file's marker version against `current_version` (semver) and record the relationship:
- **behind** — any marker version is lower than `current_version` (a real update is available)
- **in sync** — every marker version equals `current_version` (nothing to reconcile)
- **ahead** — any marker version is higher than `current_version` (project is newer than the installed plugin — unusual, e.g. the install was downgraded)

## Step 3 — Choose install mode

First display the scan summary so the user sees the per-file state before deciding:

```
.claude/rules/git.md     {Absent / Helm v{X.Y.Z} / Foreign / Referenced in CLAUDE.md}
.claude/rules/safety.md  {Absent / Helm v{X.Y.Z} / Foreign / Referenced in CLAUDE.md}
Installed helm version:  v{current_version}
```

Then choose the question and labels based on state.

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

If state = UPDATE, branch on the version relationship:

If **in sync** (marker versions all equal `current_version`):
  Inform the user: "Helm rules are already up to date (v{current_version})." Do not offer an overwrite.
  AskUserQuestion:
    question: "Rules are already at v{current_version}. Anything to change?"
    header:   "Up to date"
    multiSelect: false
    options:
      - label: "Nothing (Recommended)"
        description: "Leave everything as-is."
      - label: "Switch to reference mode"
        description: "Delete the copied files and reference the installed plugin path from CLAUDE.md instead."

If **behind** (a marker version is lower than `current_version`):
  AskUserQuestion:
    question: "Update the helm rules in this project from v{marker_version} to v{current_version}?"
    header:   "Update mode"
    multiSelect: false
    options:
      - label: "Update rules in .claude/rules/ (Recommended)"
        description: "Overwrite helm-marked files with v{current_version} from the installed plugin."
      - label: "Switch to reference mode"
        description: "Delete the copied files and reference the installed plugin path from CLAUDE.md instead."
      - label: "Cancel"
        description: "Exit without changes."

If **ahead** (a marker version is higher than `current_version`):
  Warn the user: "This project's rules (v{marker_version}) are newer than the installed plugin (v{current_version}) — the installed plugin may have been downgraded. Overwriting would roll the rules back."
  AskUserQuestion:
    question: "Project rules (v{marker_version}) are ahead of the installed plugin (v{current_version}). How would you like to proceed?"
    header:   "Project ahead"
    multiSelect: false
    options:
      - label: "Keep project rules (Recommended)"
        description: "Leave the newer files in place — update the installed plugin instead if you want to move forward."
      - label: "Roll back to v{current_version}"
        description: "Overwrite with the older installed-plugin version anyway."
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

## Step 4 — Execute

### Copy or Update path

- Create `.claude/rules/` if it does not exist.
- For each of `git.md`, `safety.md`:
  - Read the source from `~/.claude/plugins/marketplaces/claude-helm/rules/{name}`.
  - Prepend a marker line: `<!-- helm-rule: claude-helm@v{current_version} -->`
  - Write to `.claude/rules/{name}`.
- Point CLAUDE.md at the copied rules so they load as context. Detect the helm entries in any `## Rules` section — lines pointing at `.claude/rules/` or the marketplaces path — and set them to the local paths below, replacing any marketplace-path or stale entries rather than appending alongside them. If a `## Rules` section already exists (even a user-authored one), add or replace only the helm entries and preserve the user's other entries; create a new `## Rules` section only if none exists — never add a second `## Rules` heading:
  ```
  ## Rules

  This project follows the rules shipped in claude-helm:
  - .claude/rules/git.md
  - .claude/rules/safety.md

  At the start of every session, read the rule files above and follow them.
  ```
  If CLAUDE.md does not exist, create it with just this section.

### Review per file path

- For each Foreign file, use AskUserQuestion with options: Overwrite, Skip, Show diff. Loop the prompt after Show diff so the user can still pick Overwrite or Skip.
- Overwrite writes the file via the Copy or Update path (marker + local `## Rules` entry); Skip leaves the user's file untouched.

### Reference path

- If Helm-marked local files exist at `.claude/rules/{name}` (switching from copy/update), delete them so the next scan classifies as REFERENCED, not UPDATE. Leave Foreign (unmarked) files untouched — the CONFLICT → reference path points CLAUDE.md at the plugin without removing user-authored files.
- Use the marketplaces install path: `~/.claude/plugins/marketplaces/claude-helm/rules/`. This path always reflects the latest installed version and updates automatically after `/plugin update helm@claude-helm`.
- If `CLAUDE.md` exists:
  - Detect the helm entries in any `## Rules` section — lines pointing at `.claude/rules/` or the marketplaces path — and set them to the marketplace paths below, replacing any local-path entries left by copy mode rather than appending alongside them. If a `## Rules` section already exists (even a user-authored one), add or replace only the helm entries and preserve the user's other entries; create a new `## Rules` section only if none exists — never add a second `## Rules` heading.
    ```
    ## Rules

    This project follows the rules shipped in claude-helm:
    - ~/.claude/plugins/marketplaces/claude-helm/rules/git.md
    - ~/.claude/plugins/marketplaces/claude-helm/rules/safety.md

    At the start of every session, read the rule files at the paths above and follow them.
    If either path is missing, inform the user: "helm rules are referenced in CLAUDE.md but the
    plugin is not installed on this machine. Install it with: /plugin install helm@claude-helm"
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
    plugin is not installed on this machine. Install it with: /plugin install helm@claude-helm"
    ```
  - If Skip: print the snippet to the chat so the user can place it manually.

### No-change path

- For "Nothing" (in sync), "Keep project rules" (ahead), or "Keep references" (already in reference mode): make no changes and report the status (already up to date, project ahead of the installed plugin, or references kept). No files written.

### Cancel path

- Exit silently. No files written.

### Overwrite mapping

- "Update rules" (behind) and "Roll back to v{current_version}" (ahead) both run the Copy or Update path.
- "Switch to reference mode" runs the Reference path.

## Step 5 — Report

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

For No-change (in sync or project ahead):
```
ADOPT COMPLETE — no changes
- Rules already at v{current_version} (in sync)         # in-sync case
- Project rules v{marker_version} are ahead of installed v{current_version}; left as-is   # ahead case
```

## Notes

- The marker line at the top of each copied file is required for future `/helm:adopt` runs to detect helm-managed files vs. user-managed files. Do not strip it.
- This command never writes outside `.claude/rules/` or `CLAUDE.md` in the current project.
- This command never touches anything in the user's `~/.claude/` directory.


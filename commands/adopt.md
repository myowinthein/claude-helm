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
        description: "For each foreign file, ask whether to overwrite it or skip it."
      - label: "Reference from CLAUDE.md instead"
        description: "Point CLAUDE.md at the installed plugin path. You'll be asked per foreign file whether to keep or delete it, since a local copy would coexist with the referenced rules."
      - label: "Cancel"
        description: "Exit without changes."

Wait for response.

## Step 4 — Execute

### Updating CLAUDE.md's `## Rules` section

The Copy or Update and Reference paths both update CLAUDE.md the same way — only the snippet content differs. Wherever a path says "update the `## Rules` section with {snippet}", do this:

- Detect the helm entries in any existing `## Rules` section (lines pointing at `.claude/rules/` or the marketplaces path) and set them to the snippet's entries. Preserve any user-authored entries — add or replace only the helm entries. Create a new `## Rules` section only if none exists — never add a second `## Rules` heading.
- If CLAUDE.md exists: write silently — the install mode was already chosen in Step 3.
- If CLAUDE.md does not exist: create it silently with just the `## Rules` section. Do not prompt and do not offer a manual-placement path — the pointer is required for the rules to load, so anything short of writing it leaves the install non-functional. The Report step notes that CLAUDE.md was created.

### Copy or Update path

- Create `.claude/rules/` if it does not exist.
- For each of `git.md`, `safety.md`:
  - Read the source from `~/.claude/plugins/marketplaces/claude-helm/rules/{name}`.
  - Prepend a marker line: `<!-- helm-rule: claude-helm@v{current_version} -->`
  - Write to `.claude/rules/{name}`.
- Update CLAUDE.md's `## Rules` section (see *Updating CLAUDE.md's `## Rules` section* above) with this snippet, so the copied rules load as context:
  ```
  ## Rules

  This project follows the rules shipped in claude-helm:
  - .claude/rules/git.md
  - .claude/rules/safety.md

  At the start of every session, read the rule files at the paths above and follow them.
  ```

### Review per file path

For each Foreign file:

AskUserQuestion:
  question: "{file} already exists and was not created by helm. Overwrite it with the plugin's version?"
  header:   "Foreign file"
  multiSelect: false
  options:
    - label: "Overwrite"
      description: "Replace it with the helm version — adds the marker and the local ## Rules entry"
    - label: "Skip"
      description: "Leave your file untouched"

- On Overwrite → write the file via the Copy or Update path.
- On Skip → leave the user's file as-is. (To compare first, the developer can diff their file against the plugin source manually.)

### Reference path

- Delete the Helm-marked local files at `.claude/rules/git.md` and `.claude/rules/safety.md` — no need to ask, the user already chose reference mode. This also makes the next scan classify as REFERENCED, not UPDATE.
- If `.claude/rules/git.md` or `.claude/rules/safety.md` is Foreign (unmarked) — the user's own file sitting at helm's path — ask before removing it, since it would coexist with and conflict with the referenced rule of the same name. Only these two names; leave any other file in `.claude/rules/` untouched.

  AskUserQuestion:
    question: "{file} is your own rule file. In reference mode it would sit alongside the plugin's referenced {name} and could conflict. Delete it?"
    header:   "Local rule file"
    multiSelect: false
    options:
      - label: "Delete it"
        description: "Remove your file so only the referenced plugin rules apply."
      - label: "Skip"
        description: "Keep your file — it stays active alongside the referenced rules."

  On Skip, note the kept file in the Report as a possible conflict.
- Update CLAUDE.md's `## Rules` section (see *Updating CLAUDE.md's `## Rules` section* above) with this snippet:
  ```
  ## Rules

  This project follows the rules shipped in claude-helm:
  - ~/.claude/plugins/marketplaces/claude-helm/rules/git.md
  - ~/.claude/plugins/marketplaces/claude-helm/rules/safety.md

  At the start of every session, read the rule files at the paths above and follow them.
  If either path is missing, inform the user: "helm rules are referenced in CLAUDE.md but the
  plugin is not installed on this machine. Install it with: /plugin install helm@claude-helm"
  ```

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
- CLAUDE.md {created / updated}: ## Rules section points at the copied files
```

For Reference:
```
ADOPT COMPLETE
- CLAUDE.md {created / updated}: ## Rules section references claude-helm rules at v{current_version}
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


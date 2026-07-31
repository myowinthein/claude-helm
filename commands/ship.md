---
description: Bump version, run tests, tag release, and promote to deployment environments
---

# ship

## Before starting

Check current branch.

If on main or master:
  Proceed normally — full ship flow including version tagging.

If on environment branch (staging, production, etc):
  Proceed with promotion only — no version tagging.
  Inform user:
  "On {branch}. Promoting to next environment only — no version tagging."

If on feature or any other branch:
  Stop and inform user:
  "Feature branches must be merged to main via PR before releasing.
  Please merge your branch, switch to main or master, and run /ship
  from there."

---

## Step 1 — Select deployment targets (main branch only)

Skip this step if on environment branch — proceed directly to Step 5.

Discover environment branches via: git branch -r
Filter for known environment names (staging, production, or similar).

If no environment branches exist:
  Skip this step. Tag and push main only.

If environment branches exist, use AskUserQuestion:
  AskUserQuestion:
    question: "main/master will always be tagged and pushed. Which additional environments should be promoted?"
    header:   "Deploy targets"
    multiSelect: true
    options: one entry per discovered environment branch, e.g.:
      - label: "staging"
        description: "Merge and push main to the staging branch"
      - label: "production"
        description: "Merge and push main to the production branch"

  If user selects none → deploy to main/master only.
  Wait for response before proceeding.

---

## Step 2 — Calculate and propose version (main branch only)

Skip this step if on environment branch — proceed directly to Step 5.

Detect version file by scanning for package.json, composer.json, VERSION file.
Read current version from detected file.

Run: git describe --tags --abbrev=0
If no tag exists:
  Read current version from detected version file.
  If version found in file → use it as base version.
  If not found → use 0.1.0 as base version.
  Inform human: "No tags found. Using {base_version} as base version."

**Detect commit style and calculate version**
Run: git log {last_tag}..HEAD --oneline

Use this single output for both detection and version calculation:
- If feat:/fix:/chore:/feat!: patterns are present → Conventional Commits detected
- If no Conventional Commits patterns → not detected

If Conventional Commits detected:
  Read commit messages and scan file changes.

  Triage by commit type:
  - feat              → minor bump candidate
  - fix               → patch bump candidate
  - feat! or BREAKING CHANGE → major bump candidate
  - chore, docs, style, ci, build → ignore for version calculation

  Calculate next version based on highest-priority commit type:
  - Any BREAKING CHANGE or feat! → major bump
  - Any feat (no breaking change) → minor bump
  - Only fix/patch types → patch bump

  Present to human:

  Current version: v{last_tag}
  Next version:    v{proposed}

  Commits included:
  - {list of feat and fix commits, skip chore/docs/style}

  Deployment targets:
  - {selected environments from Step 1}

  If current version is 0.x.x:
    AskUserQuestion:
      question: "Ship v{proposed}? Confirming will commit, tag, push, and promote to any environments selected in Step 1. Current: v{last_tag} → Proposed: v{proposed}"
      header:   "Version"
      multiSelect: false
      options:
        - label: "Confirm v{proposed} (Recommended)"
          description: "Commit, tag, push, and promote v{proposed} — this is the release go-ahead"
        - label: "Bump to v1.0.0"
          description: "This release completes a usable set of features that solves a real problem end-to-end"
        - label: "Enter custom version"
          description: "Specify a different version number"

  If current version is 1.0.0 or above:
    AskUserQuestion:
      question: "Ship v{proposed}? Confirming will commit, tag, push, and promote to any environments selected in Step 1. Current: v{last_tag} → Proposed: v{proposed}"
      header:   "Version"
      multiSelect: false
      options:
        - label: "Confirm v{proposed} (Recommended)"
          description: "Commit, tag, push, and promote v{proposed} — this is the release go-ahead"
        - label: "Enter custom version"
          description: "Specify a different version number"

  If "Enter custom version" selected → ask human to type the version before proceeding.
  Wait for response before proceeding.

If Conventional Commits not detected:
  Inform human:
  "Conventional Commits not detected in this repo.
  Version cannot be calculated automatically.
  Consider adopting Conventional Commits for future auto-versioning.
  Current version: v{last_tag}
  What should the next version be?"

  Wait for human to input version before proceeding.

---

## Step 3 — Run code quality checks

Run the git.md Code Quality gate — lint + tests — before releasing. Two release-specific overrides:
- Run the full test suite, not just changed-file tests — a release warrants it.
- If no test framework is detected, don't skip silently — ask "No test framework detected — releasing without test validation. Continue?" and wait.

Do not proceed until lint passes and tests are green.

---

## Step 4 — Execute release on main

Only run this step if on main or master.

Bump version in detected version file to {version}.

Scan README.md for version references (badges, inline mentions).
If found, update to {version}. Skip silently if none found.

Commit, tag, and push:
- git add {version_file}
- git add README.md  (only if README was updated in the step above)
- git add {any files the Step 3 linter/formatter modified}  (per git.md: fold formatting changes into the last commit; never git add -A)
- git commit -m "chore(release): bump version to {version}"
- git tag -a v{version} -m "Release v{version}"
- git push origin HEAD
- git push origin v{version}

For each selected environment branch:
- git checkout {environment}
- git merge main --no-ff -m "chore(deploy): promote main to {environment} for v{version}"
- git push origin {environment}
- git checkout main

Then check whether this repo is hosted on GitHub:
- Run: git remote get-url origin
- If the URL does not contain `github.com` → skip the rest of this step silently

If hosted on GitHub, use AskUserQuestion:
  AskUserQuestion:
    question: "Create a GitHub Release for v{version}?"
    header:   "GitHub Release"
    multiSelect: false
    options:
      - label: "Create release (Recommended)"
        description: "Run gh release create with auto-generated notes"
      - label: "Skip"
        description: "No GitHub Release for this version"

  If Skip selected → skip silently.

  If Create release selected:
  - Build release notes from git log:
    git log v{last_tag}..HEAD --pretty=format:"- %s" | grep -E "^- (feat|fix)"
  - Run: gh release create v{version} --title "v{version}" --notes "{extracted_notes}"
  - If the command fails with an auth error, inform human:
    "gh is not authenticated. Run: gh auth login
    Then re-run /helm:ship or create the release manually."

---

## Step 5 — Execute promotion on environment branch

Only run this step if on environment branch.

Determine next environment in promotion chain:
  staging    → production
  production → no further promotion, inform human and exit

If next environment exists:
  git push origin {current_branch}
  Inform human:
  "Pushed {current_branch}. CI/CD will deploy to corresponding server."

If no next environment (already on production):
  Stop and inform human:
  "Already on production. Nothing to promote further."

---

## Step 6 — Confirm completion

If on main or master:
  Report:
  - Version tagged:        v{version}
  - Tag pushed:            yes
  - README updated:        yes/no
  - Environments promoted: {list or none}
  - GitHub Release:        created / skipped / not GitHub
  - Deployment triggered:  yes/no (based on CI/CD presence)

If on environment branch:
  Report:
  - Branch pushed:         {branch}
  - Promoted to:           {next environment}
  - Deployment triggered:  yes/no (based on CI/CD presence)
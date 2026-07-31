---
description: Bump version, run tests, tag release, and promote to deployment environments
---

# ship

## Before starting

Check current branch.

If on main or master:
  Proceed normally — full ship flow including version tagging.

If on environment branch (staging, production, etc):
  Proceed with promotion only — Steps 1-4 (version bump, commit, tag, code quality checks, GitHub Release) are all skipped; only Step 5's promotion runs.
  Inform user:
  "On {branch}. Promoting to next environment only — no version bump, commit, tag, code quality checks, or GitHub Release. Those only run from main or master."

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

Check `environment-promotion` in CLAUDE.md Project Config (see git.md's Environment Branches section; absence defaults to `fan-out`):
- **fan-out**: offer every discovered environment branch below.
- **chain**: offer only first-tier branches (staging, stage, uat, preprod) — second-tier branches (production, prod) are never a direct deploy target from main in chain mode; they're only reached later via Step 5's promotion from the first tier. If only second-tier branches exist (no first tier to promote through), offer none and inform the human why: "production-tier branches require a pre-production tier to promote through in chain mode; none found."

If no environment branches exist (after filtering):
  Skip this step. Tag and push main only.

If environment branches exist (after filtering), use AskUserQuestion:
  AskUserQuestion:
    question: "main/master will always be tagged and pushed. Which additional environments should be promoted?"
    header:   "Deploy targets"
    multiSelect: true
    options: one entry per offered environment branch, e.g.:
      - label: "staging"
        description: "Merge and push main to the staging branch"
      - label: "production"
        description: "Merge and push main to the production branch"  (fan-out mode only)

  If user selects none → deploy to main/master only.
  Wait for response before proceeding.

---

## Step 2 — Calculate and propose version (main branch only)

Skip this step if on environment branch — proceed directly to Step 5.

Detect version file by scanning for `.claude-plugin/plugin.json` (Claude Code plugin format), package.json, composer.json, or VERSION file, in that priority order if more than one exists — a `.claude-plugin/plugin.json` is a strong, deliberate signal that this repo's canonical version lives there, not in an incidental package.json. Needed later in Step 4 regardless of whether a tag exists.

Run: git describe --tags --abbrev=0
If no tag exists:
  Read current version from the detected file.
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

  Triage by commit type (covers every type in git.md's Conventional Commits list):
  - feat              → minor bump candidate
  - fix, perf         → patch bump candidate — perf is a real improvement to shipped behavior, not a no-op
  - feat! or BREAKING CHANGE → major bump candidate
  - chore, docs, style, ci, build, refactor, test, revert → ignore for version calculation — no behavior change from the user's perspective

  Calculate next version based on highest-priority commit type:
  - Any BREAKING CHANGE or feat! → major bump
  - Any feat (no breaking change) → minor bump
  - Only fix/perf commits (the patch-bump-triggering types) → patch bump

  Present to human:

  Current version: v{last_tag}
  Next version:    v{proposed}

  Commits included:
  - {list of feat and fix commits, skip chore/docs/style}

  Deployment targets:
  - {selected environments from Step 1}

  AskUserQuestion:
    question: "Ship v{proposed}? Confirming will commit, tag, push, and promote to any environments selected in Step 1. Current: v{last_tag} → Proposed: v{proposed}"
    header:   "Version"
    multiSelect: false
    options:
      - label: "Confirm v{proposed} (Recommended)"
        description: "Commit, tag, push, and promote v{proposed} — this is the release go-ahead"
      - label: "Bump to v1.0.0"  (only include if current version is 0.x.x)
        description: "This release completes a usable set of features that solves a real problem end-to-end"
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

Whichever path supplied {version} manually (custom version entry, or the no-CC direct-entry path), validate it matches Semantic Versioning format (MAJOR.MINOR.PATCH, optionally with a `-prerelease` or `+build` suffix, e.g. `2.1.0` or `2.1.0-beta.1`) before proceeding to Step 3. If it doesn't match, inform the human and ask again — this value gets written into the version file and baked into a permanent tag (git.md: tags are never rewritten), so it isn't safe to accept unchecked. {proposed} from the auto-calculated path doesn't need this check — it's derived from a known-good base version, always well-formed by construction.

---

## Step 3 — Run code quality checks

Only reached on the main-branch path — Step 1's redirect sends the environment-branch path straight to Step 5, skipping this step entirely.

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

If the detected version file is `.claude-plugin/plugin.json`, also check `.claude-plugin/marketplace.json` for a `plugins[]` entry whose `name` matches plugin.json's own `name` field, and bump that entry's `version` to {version} too — marketplace metadata drifting from the actual plugin version is exactly the kind of staleness this step exists to prevent. Skip silently if `marketplace.json` doesn't exist or has no matching entry.

Commit, tag, and push:
- git add {version_file}
- git add README.md  (only if README was updated in the step above)
- git add .claude-plugin/marketplace.json  (only if it was updated in the step above)
- git add {any files the Step 3 linter/formatter modified}  (per git.md: fold formatting changes into the last commit; never git add -A)
- git commit -m "chore(release): bump version to {version}"
- git tag -a v{version} -m "Release v{version}"
- git push origin HEAD
- git push origin v{version}

Capture the commit SHA (`git rev-parse HEAD`) as {commit_sha} for the Step 6 report.

Before promoting to any selected environment, check CI status for the commit just pushed to main — per git.md's Environment Branches rule ("if CI is configured, it must pass before promoting"):
- If this repo isn't hosted on GitHub, or no workflow files exist under `.github/workflows/`, skip this check silently — proceed straight to the promotion loop below.
- Otherwise: using {commit_sha} captured above, run `gh run list --branch main --limit 20 --json headSha,status,conclusion` and filter to runs whose `headSha` matches {commit_sha}. A single push can trigger multiple workflow runs, and querying only the single most recent run risks catching a stale run from a previous commit before GitHub has registered this one — matching on `headSha` avoids both problems.
  - If no matching runs found → treat as "none yet" (CI hasn't registered this commit).
  - If every matching run has conclusion "success" → proceed to the promotion loop below silently.
  - Otherwise (any matching run queued, in progress, failed, or none found yet):
      AskUserQuestion:
        question: "CI for this release hasn't reported success yet ({summary of matching run states, or 'no runs found yet' if none}). Promote to the selected environments anyway?"
        header:   "CI status"
        multiSelect: false
        options:
          - label: "Skip promotion for now"
            description: "main stays tagged and pushed; no environment branches are touched this run"
          - label: "Promote anyway"
            description: "Proceed without waiting for CI"
      If "Skip promotion for now" selected → skip the promotion loop below entirely, note it in the Step 6 report.

For each selected environment branch (unless skipped above):
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
  - On success, `gh release create` prints the release URL to stdout — capture it as {release_url} for the Step 6 report.
  - If the command fails with an auth error, inform human:
    "gh is not authenticated. Run: gh auth login
    Then re-run /helm:ship or create the release manually."

---

## Step 5 — Execute promotion on environment branch

Only run this step if on environment branch.

Check `environment-promotion` in CLAUDE.md Project Config (see git.md's Environment Branches section; absence defaults to `fan-out`).

### Fan-out mode

There is no chain to advance — environment branches only ever receive code directly from main (Step 1), never from each other. Just push the current branch:
- git push origin {current_branch}
- Inform human:
  "Pushed {current_branch}. CI/CD will deploy to corresponding server."

### Chain mode

Determine next environment in promotion chain. git.md recognizes these environment names; treat names within the same tier as equivalent, matching {current_branch} against whichever it is:
  Tier 1 (pre-production): staging, stage, uat, preprod
  Tier 2 (production):     production, prod

If {current_branch} matches Tier 1: the next environment is whichever Tier 2 branch actually exists on the remote (discover via `git branch -r`, same detection as Step 1). If no Tier 2 branch exists, treat as no further promotion.
If {current_branch} matches Tier 2: no further promotion, inform human and exit.
If {current_branch} matches neither tier (a long-lived branch that still qualifies as an environment branch per git.md, just outside the recognized name list): ask the human which branch it should promote to, rather than guessing.

If next environment exists, first check CI status for {current_branch} — per git.md's Environment Branches rule ("if CI is configured, it must pass before promoting"):
- If this repo isn't hosted on GitHub, or no workflow files exist under `.github/workflows/`, skip this check — proceed to the confirmation below with no CI line in the question.
- Otherwise: capture the branch tip's SHA (`git rev-parse {current_branch}`), then run `gh run list --branch {current_branch} --limit 20 --json headSha,status,conclusion` and filter to runs whose `headSha` matches — a single commit can trigger multiple workflow runs, so checking only the most recent entry could miss one still failing or in progress. Fold the result (all matching runs succeeded / not yet / none found) into the confirmation question below rather than asking separately.

Confirm before mutating anything (per safety.md: never push without confirmation):
  AskUserQuestion:
    question: "Promote {current_branch} to {next_environment}? This merges {current_branch} into {next_environment} and pushes both branches. {CI: {conclusion}, if checked above}"
    header:   "Promote"
    multiSelect: false
    options:
      - label: "Promote to {next_environment} (Recommended if CI passed)"
        description: "Merge {current_branch} into {next_environment} and push"
      - label: "Cancel"
        description: "Exit without promoting"

  If Cancel → exit silently.

  If confirmed:
  - git push origin {current_branch}
  - git checkout {next_environment}
  - git merge {current_branch} --no-ff -m "chore(deploy): promote {current_branch} to {next_environment}"
    - If the merge conflicts: run `git merge --abort` to restore a clean state, then `git checkout {current_branch}` to return to the starting branch. Inform the human: "Merge conflict promoting {current_branch} to {next_environment}. Aborted — {next_environment} is unchanged. Resolve the conflict manually (e.g. merge {current_branch} into {next_environment} in a local checkout, fix conflicts, push), then re-run." Stop here — do not proceed to the push/checkout below.
  - git push origin {next_environment}
  - git checkout {current_branch}
  - Inform human:
    "Promoted {current_branch} → {next_environment}. CI/CD will deploy to corresponding server."

If no next environment (already on the last tier):
  Stop and inform human:
  "Already on the last environment in the chain. Nothing to promote further."

---

## Step 6 — Confirm completion

If on main or master:
  Report:
  - Commit:                {commit_sha} "chore(release): bump version to {version}"
  - Version tagged:        v{version}
  - Tag pushed:            yes
  - README updated:        yes/no
  - Environments promoted: {list, or "none" if none selected, or "none — skipped" if skipped for CI reasons}
  - GitHub Release:        {release_url, if created} / skipped / not GitHub
  - Deployment triggered:  yes/no (based on CI/CD presence)

  If any environment was promoted after choosing "Promote anyway" despite CI not reporting success: add a line — "Note: promoted to {list of those environments} without a passing CI run (manually overridden)."
  If promotion was skipped because CI hadn't passed: add a line — "To promote later: checkout the environment branch, merge main in manually, and push — re-running /helm:ship would cut a new version rather than retry this promotion."

If on environment branch, fan-out mode:
  Report:
  - Branch pushed:         {current_branch}
  - Deployment triggered:  yes/no (based on CI/CD presence)

If on environment branch, chain mode:
  Report:
  - Promoted:              {current_branch} → {next_environment}
  - Branches pushed:       {current_branch}, {next_environment}
  - Deployment triggered:  yes/no (based on CI/CD presence)

  If the CI check ran and did not find success (regardless of which run states it was) before the user confirmed promotion: add a line — "Note: promoted without a passing CI run (manually confirmed anyway)."
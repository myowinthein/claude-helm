---
description: Detect test framework and write missing tests for recent changes or full project
---

# test

## Before starting

Check `git-strategy` in CLAUDE.md's Project Config (absence defaults to GitHub Flow, per git.md).

**Solo Mode** (`git-strategy: solo`):
- Only proceed if on `main` or `master`. If on any other branch, stop: "test must be run on main or master in Solo Mode. Current branch is {branch} — switch and re-run."
- No branch is created — every commit in Steps 4–6 lands directly on main as it happens.

**GitHub Flow** (`git-strategy: github-flow`, or absent):
- Record the current branch as `{original_branch}` — the workflow returns here at the end, whatever it was.
- Checkout a fresh branch from main's current tip: update local main (`git pull` if a remote exists), then `git checkout -b test/{YYYYMMDD}` from it.
- Call this branch `{branch}` for the rest of this command. Every commit in Steps 4–6 lands on `{branch}`, mirroring `/helm:refactor`'s single-branch-per-run model — Step 7 merges it back to main once at the end, rather than each cluster's commit pushing to main individually.
- If this command exits at any point without writing anything, delete `{branch}` and return to `{original_branch}` before exiting — never leave the user stranded on an empty temporary branch it created.

## Scope

Tests must reflect actual proven behavior — not speculative edge cases.
Follow existing test conventions, naming, and file structure.
Never push with failing tests.

---

## Step 1 — Detect test framework

Scan for test framework configuration files and dependencies.
Also detect the project's coverage tool (coverage.py, nyc, jest --coverage, etc.) — record it for use in Step 5.

If framework detected → proceed to Step 2.

If no framework detected, form the recommendation from the project actually detected — language, package manager (from the lockfile), existing build tooling and module system, monorepo structure if any — reusing what the scan above already found rather than defaulting to generic knowledge.
  AskUserQuestion:
    question: "No test framework detected. Which would you like to set up? (based on detected stack)"
    header:   "Framework"
    multiSelect: false
    options:
      - label: "{recommended framework} (Recommended)"
        description: "{why it fits this stack}"
      - label: "{alternative}"
        description: "{brief reason}"
      - label: "{alternative}"
        description: "{brief reason}"
      - label: "Skip"
        description: "I'll set up testing manually"

If Skip selected → exit and inform user:
  "Configure a test framework and re-run /test."

If framework selected:
- Detect the project's package manager from its lockfile (e.g. `package-lock.json` → npm, `yarn.lock` → yarn, `pnpm-lock.yaml` → pnpm, `composer.lock` → composer, `poetry.lock` → poetry, `requirements.txt` → pip, `Gemfile.lock` → bundler) and run the matching install command for the chosen framework as a dev dependency (e.g. `npm install --save-dev {framework}`).
- If the install fails, stop and inform the user with the error — do not proceed to Step 2 with a framework that isn't actually installed.
- If it succeeds, proceed to Step 2.

---

## Step 2 — Load ledger

Look for `.claude/helm/test-log.json`. If it exists, load it.

If not found, check the legacy path `.claude/test-log.json` (this plugin's ledgers used to live flat in `.claude/`, which risks colliding with another plugin's own files of the same generic name). If a legacy file exists, load it from there — this is a one-time **path** migration, distinct from the schema-shape migration below: the next time the ledger is written (Step 6), it moves to the new `.claude/helm/` path and the old file is removed as part of the same commit.

If neither path has a file, proceed as if the ledger is empty — a missing ledger is not an error.

Schema:
```json
{
  "schema_version": 2,
  "last_run_commit": "abc1234",
  "full_scan_ever_run": true,
  "consecutive_catchup_count": 0,
  "files": [
    {
      "file": "path/to/file.ts",
      "priority": "high" | "medium" | "low",
      "last_judged_commit": "abc1234",
      "user_status": "skipped-by-user" | "ambiguous",
      "note": "short reason, e.g. what was unclear or why user skipped",
      "recorded_commit": "abc1234",
      "recorded_date": "2026-07-09"
    }
  ]
}
```

`schema_version` — bumped only if this ledger's structure changes in a future release. Lets a future version of this command detect an older-shaped file and handle it explicitly instead of misreading it.

If `schema_version` is `1` or absent (a ledger written before this shape existed): convert on load rather than misreading it. The old shape had `last_test_run_commit`, `last_full_scan_commit`, a `findings` array (`status`, `note`, `recorded_commit`, `recorded_date`), and a separate `full_scan_findings` array (`priority`, `last_judged_commit`).

Convert as follows:
- Map `last_test_run_commit` → `last_run_commit` directly.
- `full_scan_ever_run` is `true` only if `last_full_scan_commit` holds an actual commit SHA — **not** merely if the key is present. The old field could legitimately be `null` (a repo where a full scan has never run yet, `findings` and `full_scan_findings` both empty) while still being present in the JSON; treating key-presence alone as "a full scan happened" gets this case wrong.
- Merge `findings` and `full_scan_findings` into one `files` array keyed by file path. A file present in both old arrays becomes one record carrying both sets of fields — this is common, not an edge case (e.g. a file already flagged `ambiguous` by a human is also a real target for the next Full Scan's priority judgment).
- Set `consecutive_catchup_count` to `0` — the old shape never tracked it, and there's no way to reconstruct a meaningful count retroactively.
- Every commit SHA carried over (`last_run_commit`, and each entry's `last_judged_commit`/`recorded_commit`) should already be a valid, reachable commit in this repo, since the old command only ever wrote real SHAs there — no fallback or repair needed for those.

Treat the result as `schema_version: 2` for the rest of this run, and it will be persisted in that shape at Step 6.

Each `files` entry represents everything currently known about one file. `priority` and `last_judged_commit` come from sub-agent scan judgment (Full Scan only) and are independent of `user_status`, `note`, `recorded_commit`, and `recorded_date`, which come from an actual human decision (Behavior Clarity Check, or the Catch Up/Full Scan skip option) — a file can have either set of fields, both, or neither, and updating one set must never clobber the other.

`full_scan_ever_run` and `consecutive_catchup_count` exist for Step 3's scenario decision and Full Scan recommendation — see below.

---

## Step 3 — Assessment

Check recent git activity and existing test coverage:
- Identify recently changed files using the same commit-range diff as Step 4:
  - If `last_run_commit` is in the ledger: run `git diff {last_run_commit}..HEAD --name-only`
  - If absent: fall back to `git diff HEAD~1..HEAD --name-only`, or the working tree diff if uncommitted changes exist
- Count `files` entries with `user_status: ambiguous`, regardless of whether they appear in the diff above — these are outstanding from a previous run, not just files that happen to have changed again.
- Scan for existing test files
- Estimate coverage gaps in recently changed code
- Estimate overall project test coverage

Based on current state, pick one of four scenarios:

**Scenario 1 — No existing tests found:**
  AskUserQuestion:
    question: "No existing tests found — a full scan is recommended."
    header:   "Test scope"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Scan entire project for missing tests"
      - label: "Skip"
        description: "No tests needed"

**Scenario 2 — Tests exist, full scan was never run (`full_scan_ever_run` is `false` or absent):**
  AskUserQuestion:
    question: "Tests exist but a full scan has never been run — coverage gaps may exist."
    header:   "Test scope"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Scan entire project for pre-existing coverage gaps"
      - label: "Skip"
        description: "No tests needed"

**Scenario 3 — Tests exist, full scan done, no changes since last run, no outstanding ambiguous findings:**
  Inform the user: "No changes since last run — tests are up to date." Do not present any AskUserQuestion. Proceed to Step 7 to report the outcome — nothing was written, but this is still an outcome worth confirming.

**Scenario 4 — Tests exist, and either recent changes are detected or ambiguous findings are outstanding:**
  Recommend Full Scan if `consecutive_catchup_count >= 5` (many Catch Ups have piled up since the last full look) OR the diff identified above touches 50+ files (this one batch is already big enough to warrant a full check). Otherwise recommend Catch Up. Outstanding ambiguous findings alone (no other changes) still route here — Catch Up will pick them up via Step 4's ambiguous-entry union, giving the user another pass through the Behavior Clarity Check.
  Put recommended option first.

  Catch Up is the recommendation:
  AskUserQuestion:
    question: "{one sentence describing coverage status and recommendation}"
    header:   "Test scope"
    multiSelect: false
    options:
      - label: "Catch Up (Recommended)"
        description: "Write tests for files changed since the last /test run. {reason, e.g. 'Only 6 files changed and 2 catch-ups since the last full scan — a targeted check should cover it.'}"
      - label: "Full scan"
        description: "Scan entire project for missing tests"
      - label: "Skip"
        description: "No tests needed"

  Full is the recommendation:
  AskUserQuestion:
    question: "{one sentence describing coverage status and recommendation}"
    header:   "Test scope"
    multiSelect: false
    options:
      - label: "Full scan (Recommended)"
        description: "Scan entire project for missing tests. {reason, e.g. '58 files changed since the last run — too broad for a focused check.' or '5 catch-ups in a row since the last full scan — due for a full look.'}"
      - label: "Catch Up"
        description: "Write tests for files changed since the last /test run"
      - label: "Skip"
        description: "No tests needed"

If Skip selected in any of Scenarios 1, 2, or 4 → nothing to write. Proceed to Step 7 to report the outcome.

---

## Step 4 — Catch Up

Identify changed files using the ledger's `last_run_commit`:
- If `last_run_commit` is present: run `git diff {last_run_commit}..HEAD` to get all files changed since the last run.
- If `last_run_commit` is absent (first run or missing ledger): fall back to `git diff HEAD~1..HEAD`, or the working tree diff if uncommitted changes exist.

Focus only on files that were added or modified.

Union in any `files` entries with `user_status: ambiguous`, even if they don't appear in the diff above — they're outstanding from a previous run and get another pass through the Behavior Clarity Check below, rather than sitting unresolved indefinitely until the file happens to change again.

Cross-check the resulting file list against `files` entries with `user_status: skipped-by-user`. Drop those files from the plan unless the file has been modified since its `recorded_commit` — if it has changed, re-include it. This does not apply to `ambiguous` entries, which are always re-included regardless of whether they changed.

Group the remaining files into clusters by module/domain — same folder, or a direct dependency (e.g. a controller and its service) — mirroring `/helm:refactor`'s clustering. Files in unrelated areas never share a cluster: git.md requires one commit per logical unit, and a single commit covering tests for two unrelated features would violate that.

Apply the **Behavior Clarity Check** (see below) before writing any test.

Before writing, present test plan:

  AskUserQuestion:
    question: "I will write tests for:\n- {file}: {what will be tested}\n- {file}: {what will be tested}\n\nProceed?"
    header:   "Confirm"
    multiSelect: false
    options:
      - label: "Write tests (Recommended)"
        description: "Proceed with the test plan above"
      - label: "Cancel"
        description: "Exit without writing tests"

If Cancel selected → nothing to write. Proceed to Step 7 to report the outcome.

Wait for response before proceeding.

Write tests that reflect actual proven behavior — not speculative edge cases.
Follow existing test conventions and file structure in the project.
Place test files according to project's existing test organization.

Write cluster by cluster. For each cluster:
- Run tests after writing — fix if failing before committing.
- Commit, with scope inferred from this cluster's files per git.md's Scope inference convention:
  test({scope}): add tests for {feature}

---

## Behavior Clarity Check

Applied in Step 5 and Step 6 before writing a test for any piece of code.

Assess whether the expected behavior is clear from the code, docs, or existing tests:

- **Clear**: write the test directly. No extra prompt.
- **Ambiguous** (undocumented edge case, unclear intended behavior, behavior contradicts docs):
  stop and confirm with the user:
  AskUserQuestion:
    question: "Behavior is unclear for {file} — {what is ambiguous}. How should this be handled?"
    header:   "Ambiguous behavior"
    multiSelect: false
    options:
      - label: "Clarify and proceed"
        description: "Provide clarification, then write the test"
      - label: "Skip this case"
        description: "Record as ambiguous in the ledger and move on"

  If the user clarifies → write the test using the clarified behavior.
  If the user skips → set `user_status: "ambiguous"` and a short note on that file's `files` entry (creating one if it doesn't exist yet, without touching any `priority` fields already there). Do not guess and encode a guess as a test.

---

## Step 5 — Full Scan

### Coverage check

Run the project's coverage tool (detected in Step 1) across the full project. This is always full and current — never partial, never skipped, regardless of ledger state.

### Priority judgment

Determine which untested areas are high, medium, or low priority.

Split the project into folder/module chunks (keeping related files together). Spawn one sub-agent per chunk, capped at 8 sub-agents — same chunking approach as Deep Mode in `/helm:refactor`. Each sub-agent reads its chunk and returns a priority judgment for untested areas.

Before spawning sub-agents, cross-check each file against `files` in the ledger:
- File has a `files` entry with `priority` set and unchanged since its `last_judged_commit`: carry the stored priority forward — do not re-run sub-agent judgment for this file.
- File has no `priority` set yet, or it's changed since `last_judged_commit`: include it in the sub-agent run; set its `priority` and `last_judged_commit` to the new judgment and current HEAD — creating a new `files` entry if none exists, or adding these fields to an existing entry without touching any `user_status`/`note`/`recorded_commit`/`recorded_date` fields already on it.
- File has a `files` entry (with `priority`, `user_status`, or both) but no longer exists in the repo: remove the entry entirely — neither the priority judgment nor any lingering user decision applies to a file that's gone.

Present findings before writing:

─────────────────────────────────
TEST COVERAGE REPORT
─────────────────────────────────

HIGH PRIORITY        {X untested}
─────────────────────────────────
{file}: {what is untested and why it matters}

MEDIUM PRIORITY      {X untested}
─────────────────────────────────
...

LOW PRIORITY         {X untested}
─────────────────────────────────
...

─────────────────────────────────
TOTAL: {N} untested areas found
─────────────────────────────────

Then use AskUserQuestion for priority selection:
  AskUserQuestion:
    question: "Which priorities to cover?"
    header:   "Priorities"
    multiSelect: true
    options:
      - label: "High Priority"
        description: "{N} areas — payment, auth, core business logic"
      - label: "Medium Priority"
        description: "{N} areas — API endpoints, data transformation"
      - label: "Low Priority"
        description: "{N} areas — everything else"

  Selecting none = skip the Writing tests section below — no tests get written, but the scan itself already ran and still needs to be recorded: proceed to Step 6 to update the ledger (the coverage check and priority judgment happened regardless of what's selected here), then Step 7. Do not add an explicit All or Skip option.

Wait for response before proceeding.

### Writing tests

Apply the **Behavior Clarity Check** (see above) before writing each test.

Write tests priority by priority, single-agent and sequential. Do not parallelize writing across sub-agents.

Within each priority, group files into clusters by module/domain — same folder, or a direct dependency — same reasoning as Step 4's Catch Up clustering. A priority tier commonly spans multiple unrelated areas (e.g. "high priority" including both `payment.ts` and `core-engine.ts`); a single commit must never bundle them together just because they share a priority label.

For each priority, cluster by cluster:
- Write tests reflecting actual proven behavior
- If a cluster is large enough to strain one agent's context, batch it sequentially (write a chunk, commit, continue) — no planning or dependency system needed
- Run tests — stop and inform if failing
- Fix before proceeding to the next cluster
- Commit, with scope inferred from this cluster's files per git.md's Scope inference convention (not the priority label itself):
  test({scope}): add missing {priority}-priority tests for {brief summary of this cluster}

---

## Step 6 — Update ledger

After tests are written, run, and committed:

- Set `schema_version` to `2` if not already present — including when converting an old-shaped ledger per Step 2's migration note.
- **Catch Up run**: set `last_run_commit` to current HEAD. Increment `consecutive_catchup_count` by 1.
- **Full Scan run**: set `last_run_commit` to current HEAD, set `full_scan_ever_run` to `true`, reset `consecutive_catchup_count` to `0`. Persist all `priority`/`last_judged_commit` changes from Step 5 into `files` (new entries, updated priorities, entries removed for files that no longer exist).
- For files recorded during the Behavior Clarity Check as ambiguous: set `user_status: "ambiguous"` (plus `note`, `recorded_commit`, `recorded_date`) on that file's `files` entry — creating one if it doesn't exist, without touching any `priority`/`last_judged_commit` fields already there. `skipped-by-user` has no writer in the current flow — Step 4's plan confirmation and Step 5's priority selection are both all-or-nothing/tier-level, not per-file — but Step 4 still checks for and honors any pre-existing `skipped-by-user` entries (e.g. from a schema migration or manual edit).
- For files resolved this run (test written successfully, or ambiguity clarified): clear `user_status`/`note`/`recorded_commit`/`recorded_date` from that file's entry. Remove the entry entirely only if it has no `priority` left either — otherwise keep it for the scan-cache data.

Write it to `.claude/helm/test-log.json` (creating `.claude/helm/` if it doesn't exist yet). Commit the ledger — if this run migrated a legacy `.claude/test-log.json` (Step 2), remove it in the same commit:
```
git add .claude/helm/test-log.json
git rm .claude/test-log.json   # only if migrating this run
git commit -m "test(log): update test ledger after {catch-up / full-scan}"
```

---

## Step 7 — Merge and cleanup

Only relevant under GitHub Flow — Solo Mode made every commit directly on main as it happened, so there is nothing to merge.

If nothing was committed this run (up to date, or Skip/Cancel selected anywhere before any test was written): under GitHub Flow, delete `{branch}` and return to `{original_branch}` — do not skip that part. Under Solo Mode there is nothing further to do. Either way, proceed to Step 8 to report the outcome.

**Solo Mode:** before pushing, confirm — push always requires its own confirmation regardless of `git-auto-commit`, per git.md's Auto-Commit rule and rules/safety.md's unconditional push-confirmation requirement:

  AskUserQuestion:
    question: "Push main now? This publishes {N} commit(s) made this run to origin."
    header:   "Push"
    multiSelect: false
    options:
      - label: "Push (Recommended)"
        description: "git push origin main"
      - label: "Cancel"
        description: "Leave main committed locally but unpushed — push manually when ready"

If Cancel selected → stop here. Do not push or promote. Proceed to Step 8 to report the outcome, noting main is committed locally but unpushed.

If Push selected: `git push origin main`, then run Environment promotion below.

**GitHub Flow:**
1. Merge into main: `git checkout main`, `git merge {branch} --no-ff -m "test(project): write tests {catch-up / full-scan}"`. Before pushing, confirm:

   AskUserQuestion:
     question: "Push main now? This publishes the merged commit to origin."
     header:   "Push"
     multiSelect: false
     options:
       - label: "Push (Recommended)"
         description: "git push origin main"
       - label: "Cancel"
         description: "Leave main merged locally but unpushed — push manually when ready"

   If Cancel selected → stop here. Do not push, promote, or clean up. Proceed to Step 8 to report the outcome, noting main is merged locally but unpushed.

   If Push selected: `git push origin main`.
2. Run Environment promotion below.
3. Delete `{branch}`: `git branch -d {branch}` locally, and `git push origin --delete {branch}` if it was ever pushed.
4. Return to where you started: `git checkout {original_branch}`.

**Environment promotion** — shared by both modes. If environment branches exist (discover via `git branch -r`, filter for known environment names, same detection as ship.md), ask which should also receive these tests:

  AskUserQuestion:
    question: "main has been updated. Which environment branches should also receive these tests?"
    header:   "Promote to environments"
    multiSelect: true
    options: one entry per discovered environment branch, e.g.:
      - label: "staging"
        description: "Merge main into staging"
      - label: "production"
        description: "Merge main into production"

  For each selected environment:
  - git checkout {environment}
  - git merge main --no-ff -m "chore(deploy): promote main to {environment} for test updates"
  - git push origin {environment}
  - git checkout main

  If no environment branches exist, or the user selects none, skip silently.

---

## Step 8 — Confirm completion

Report:

- Outcome: {up to date / catch up written / full scan written / skipped}
- Mode: {Catch Up / Full Scan / N/A}
- Tests written: {N}, covering {list of files or clusters}
- Commits made: {N}
- Tests passing: {yes/no, or N/A if nothing was run}
- Ledger: {N} skipped-by-user, {N} ambiguous, {N} resolved this run
- Pushed: {yes / no — push manually when ready / N/A, nothing committed}
- Environments promoted: {list or none}
- If GitHub Flow: temporary branch's fate ({merged and deleted / left as-is}) and which branch you were returned to.

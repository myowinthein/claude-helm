---
description: Detect test framework and write missing tests for recent changes or full project
---

# test

## Step 1 — Detect test framework

Scan for test framework configuration files and dependencies.
Also detect the project's coverage tool (coverage.py, nyc, jest --coverage, etc.) — record it for use in Step 5.

If framework detected → proceed to Step 2.

If no framework detected, use AskUserQuestion:
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

Look for `.claude/test-log.json`. If it exists, load it. If it does not exist, proceed as if it is empty — a missing ledger is not an error.

Schema:
```json
{
  "last_test_run_commit": "abc1234",
  "last_full_scan_commit": "def5678",
  "findings": [
    {
      "file": "path/to/file.ts",
      "status": "skipped-by-user" | "ambiguous",
      "note": "short reason, e.g. what was unclear or why user skipped",
      "recorded_commit": "abc1234",
      "recorded_date": "2026-07-09"
    }
  ],
  "full_scan_findings": [
    {
      "file": "path/to/file.ts",
      "priority": "high" | "medium" | "low",
      "last_judged_commit": "abc1234"
    }
  ]
}
```

`findings` tracks user decisions only: files skipped by the user or flagged as ambiguous. It does not track coverage state.

`full_scan_findings` tracks sub-agent priority judgments from the last Full Scan. Used in Step 6 to avoid re-running judgment on unchanged files. Kept separate from `findings` because it is scan-result data, not a user decision.

---

## Step 3 — Assessment

Check recent git activity and existing test coverage:
- Identify recently changed files using the same commit-range diff as Step 4:
  - If `last_test_run_commit` is in the ledger: run `git diff {last_test_run_commit}..HEAD --name-only`
  - If absent: fall back to `git diff HEAD~1..HEAD --name-only`, or the working tree diff if uncommitted changes exist
- Count ledger `findings` entries with `status: ambiguous`, regardless of whether they appear in the diff above — these are outstanding from a previous run, not just files that happen to have changed again.
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

**Scenario 2 — Tests exist, full scan was never run (`last_full_scan_commit` absent):**
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
  Exit cleanly. Inform the user:
  "No changes since last run — tests are up to date."
  Do not present any AskUserQuestion.

**Scenario 4 — Tests exist, and either recent changes are detected or ambiguous findings are outstanding:**
  Form recommendation (Catch Up or Full) based on gap significance. Outstanding ambiguous findings alone (no other changes) still route here — Catch Up will pick them up via Step 4's ambiguous-entry union, giving the user another pass through the Behavior Clarity Check.
  Put recommended option first.

  Catch Up is the recommendation:
  AskUserQuestion:
    question: "{one sentence describing coverage status and recommendation}"
    header:   "Test scope"
    multiSelect: false
    options:
      - label: "Catch Up (Recommended)"
        description: "Write tests for files changed since the last /test run"
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
        description: "Scan entire project for missing tests"
      - label: "Catch Up"
        description: "Write tests for files changed since the last /test run"
      - label: "Skip"
        description: "No tests needed"

---

## Step 4 — Catch Up

Identify changed files using the ledger's `last_test_run_commit`:
- If `last_test_run_commit` is present: run `git diff {last_test_run_commit}..HEAD` to get all files changed since the last run.
- If `last_test_run_commit` is absent (first run or missing ledger): fall back to `git diff HEAD~1..HEAD`, or the working tree diff if uncommitted changes exist.

Focus only on files that were added or modified.

Union in any ledger `findings` entries with `status: ambiguous`, even if they don't appear in the diff above — they're outstanding from a previous run and get another pass through the Behavior Clarity Check below, rather than sitting unresolved indefinitely until the file happens to change again.

Cross-check the resulting file list against ledger entries with `status: skipped-by-user`. Drop those files from the plan unless the file has been modified since its `recorded_commit` — if it has changed, re-include it. This does not apply to `ambiguous` entries, which are always re-included regardless of whether they changed.

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

Wait for response before proceeding.

Write tests that reflect actual proven behavior — not speculative edge cases.
Follow existing test conventions and file structure in the project.
Place test files according to project's existing test organization.

Write cluster by cluster. For each cluster:
- Run tests after writing — fix if failing before committing.
- Commit, with scope inferred from the cluster's primary module, folder, or domain — per git.md's Conventional Commits convention (the actual code touched, e.g. `orders`, `auth`), not a generic label. If the cluster spans multiple areas, use the dominant one; if truly cross-cutting, use `project` or `core`:
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
  If the user skips → record in the ledger with `status: "ambiguous"` and a short note. Do not guess and encode a guess as a test.

---

## Step 5 — Full Scan

### Coverage check

Run the project's coverage tool (detected in Step 1) across the full project. This is always full and current — never partial, never skipped, regardless of ledger state.

### Priority judgment

Determine which untested areas are high, medium, or low priority.

Split the project into folder/module chunks (keeping related files together). Spawn one sub-agent per chunk, capped at 8 sub-agents — same chunking approach as Deep Mode in `/helm:refactor`. Each sub-agent reads its chunk and returns a priority judgment for untested areas.

Before spawning sub-agents, cross-check each file against `full_scan_findings` in the ledger:
- File present in `full_scan_findings` and unchanged since its `last_judged_commit`: carry the stored priority forward — do not re-run sub-agent judgment for this file.
- File present but changed since `last_judged_commit`, or not present: include it in the sub-agent run; update its `full_scan_findings` entry (or add one) with the new priority and `last_judged_commit` set to current HEAD.
- File present in `full_scan_findings` but no longer exists in the repo: remove its entry.

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

  Selecting none = skip. Do not add an explicit All or Skip option.

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
- Commit, with scope inferred from the cluster's primary module, folder, or domain — same convention as Step 4 (not the priority label itself):
  test({scope}): add missing {priority}-priority tests for {brief summary of this cluster}

---

## Step 6 — Update ledger

After tests are written, run, and committed:

- **Catch Up run**: set `last_test_run_commit` to current HEAD.
- **Full Scan run**: set both `last_full_scan_commit` and `last_test_run_commit` to current HEAD. Persist all `full_scan_findings` changes from Step 5 (new entries, updated priorities, removed stale entries).
- Add new `skipped-by-user` entries for files the user chose to skip in the confirmation step.
- Add new `ambiguous` entries recorded during the Behavior Clarity Check.
- Remove or update entries for files resolved this run (test written successfully, or ambiguity clarified).

Commit the ledger:
  test(log): update test ledger after {catch-up / full-scan}

---

## Scope

Tests must reflect actual proven behavior — not speculative edge cases.
Follow existing test conventions, naming, and file structure.
Never push with failing tests.

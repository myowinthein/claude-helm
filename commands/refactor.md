---
description: Scan codebase for refactoring opportunities and apply selected categories
---

# refactor

Full project scan to identify and apply refactoring opportunities.
Run periodically on mature codebases to maintain code quality.

---

## Step 1 — Branch check

Only proceed if on main or master.
If on any other branch, stop and inform user:

"refactor must be run on main or master.
Current branch is {branch}. Please switch and re-run."

---

## Step 2 — Create refactor branch

Create a dedicated branch for refactoring changes:
  refactor/{YYYYMMDD-HHMMSS}

If an existing refactor/* branch is detected, use AskUserQuestion:
  AskUserQuestion:
    question: "An existing refactor branch was found: {branch}. Continue on it or start fresh?"
    header:   "Branch"
    multiSelect: false
    options:
      - label: "Continue on {branch} (Recommended)"
        description: "Switch to the existing branch and continue refactoring"
      - label: "Create new branch"
        description: "Start a new refactor/{YYYYMMDD-HHMMSS} branch"

On selection, create or switch to refactor branch and proceed.

---

## Step 3 — Scan boundaries

Scan all project files except:
- vendor/
- node_modules/
- public/
- storage/
- __pycache__/, .venv/, dist/, build/  (Python)
- tmp/, log/                            (Ruby)
- bin/                                  (Go)
- migration files
- .env files
- generated or compiled files
- .claude/refactor-log.json (the memory ledger itself — never treat it as source code)

Include test files — test quality degrades fastest and matters most.

---

## Step 4 — Load history and choose mode

### 4.1 Load the ledger

Look for `.claude/refactor-log.json`. This file is the command's memory across runs.

Schema:
```json
{
  "last_scanned_commit": "abc1234",
  "last_mode": "deep",
  "last_scan_date": "2026-06-01",
  "consecutive_quick_count": 0,
  "findings": [
    {
      "id": "f-0012",
      "category": "architecture",
      "priority": "high",
      "risk": "needs-review",
      "cluster_id": "c-004",
      "depends_on": [],
      "file": "app/Http/Controllers/OrderController.php",
      "line": 42,
      "description": "Business logic embedded in controller action",
      "status": "open",
      "first_found_commit": "9f1a2bc",
      "first_found_date": "2026-05-01",
      "resolved_commit": null,
      "resolved_date": null
    }
  ]
}
```

`status` values: `open`, `fixed`, `skipped-by-user`, `auto-resolved`.

`risk` values:
- `safe` — mechanical, low-ambiguity fix (rename, dead code removal, add missing index, deduplicate identical blocks). Safe to auto-apply.
- `needs-review` — requires judgment about business intent (e.g. an N+1 fix that depends on what the query is actually for). Must be confirmed with the user before touching code.

`cluster_id` — findings that touch the same file, or files with a direct dependency (e.g. controller + its service), share a cluster id. All findings in a cluster are applied together by a single agent — never split across parallel sub-agents, to avoid conflicting edits.

`depends_on` — list of finding ids that must be applied before this one (e.g. "extract shared helper" must land before "deduplicate the 3 call sites that use it").

`consecutive_quick_count` — number of consecutive Quick Mode scans since the last Deep Mode scan. Increment by 1 each time Quick Mode runs; reset to 0 when Deep Mode runs.

### 4.2 First run — no ledger found

If `.claude/refactor-log.json` does not exist, this is the first run. Do not ask which mode to use.
Inform user:

"No refactor history found — running Deep Mode to build a baseline."

Proceed directly to Step 5, Deep Mode.

### 4.3 Later runs — ledger found

Compute, since `last_scanned_commit`:
- number of commits on the current branch
- number of days since `last_scan_date`

Decide which mode to recommend using this logic:
- Recommend **Deep Mode** if: 40+ commits since last scan, OR 60+ days since last scan, OR `consecutive_quick_count` is 2 or more (old untouched code is overdue for a fresh look).
- Otherwise recommend **Quick Mode**.

Ask with AskUserQuestion, filling in the reason in the description:

  AskUserQuestion:
    question: "Which scan mode would you like to run?"
    header:   "Scan mode"
    multiSelect: false
    options:
      - label: "Deep Mode{recommended tag if applicable}"
        description: "Full codebase re-scan using multiple agents. {reason, e.g. '45 commits and 70 days since last full scan — old code may have drifted.'}"
      - label: "Quick Mode{recommended tag if applicable}"
        description: "Only checks files changed since the last scan, plus re-validates past findings. {reason, e.g. 'Only 8 commits since last scan — a quick check should cover it.'}"

Wait for response before proceeding to Step 5.

---

## Step 5 — Scan

### 5A. Deep Mode

1. List all project files within scan boundaries (Step 3). Group them by folder/module rather than raw line count, so related code (e.g. a controller and its service layer) stays in the same chunk — this avoids missing issues that span two files in different chunks.
2. Estimate a comfortable read budget per sub-agent, and compute how many chunks/sub-agents are needed based on total project size. Cap at 8 sub-agents maximum — if more chunks are needed, merge the smallest adjacent folders until within the cap.
3. Spawn one sub-agent per chunk. Each sub-agent reads its full chunk once and checks **all** categories in that single pass (architecture, code quality, performance, tests, dependencies) — do not run a second category-only pass over the same files; that doubles cost for no real benefit.
4. Each sub-agent returns structured findings: category, priority, file, line, description.
5. Main agent consolidation:
   - Merge all sub-agent reports.
   - Look across chunk boundaries for cross-cutting issues (e.g. the same duplicated pattern appearing in two different chunks) that individual sub-agents couldn't see on their own.
   - Cross-check against the ledger's existing `open` findings — do not duplicate an entry that already exists; keep it as-is.
   - Any `open` ledger entry whose file no longer exists, or has been rewritten enough that the finding no longer applies, gets marked `auto-resolved`.
   - Any genuinely new issue gets added as a new `open` entry.
   - Assign `risk` (`safe` or `needs-review`) to every new finding.
   - Group findings that touch the same or directly-related files into a shared `cluster_id`.
   - Where one finding's fix is a precondition for another (e.g. extract-before-dedupe), record it in `depends_on`.
6. Update `.claude/refactor-log.json`: set `last_scanned_commit` to current HEAD, `last_mode` to `deep`, `last_scan_date` to today, reset `consecutive_quick_count` to 0, and save the merged findings list.
7. Commit the ledger immediately:
   ```
   git add .claude/refactor-log.json
   git commit -m "chore(refactor): update ledger after deep scan"
   ```

### 5B. Quick Mode

1. Read `last_scanned_commit` from the ledger.
2. Run `git diff --name-only {last_scanned_commit}..HEAD` to get the list of changed files (respecting Step 3 exclusions).
3. If the changed file list exceeds 50 files, stop and inform the user:
   "Quick Mode found {N} changed files — that's too broad for a focused check. Consider switching to Deep Mode for a full re-scan."
   Ask with AskUserQuestion:
     question: "How would you like to proceed?"
     header:   "Scope too large"
     multiSelect: false
     options:
       - label: "Switch to Deep Mode (Recommended)"
         description: "Run a full re-scan with multiple agents"
       - label: "Continue with Quick Mode anyway"
         description: "Scan all {N} changed files in a single pass"
4. Re-validate every `open` ledger entry:
   - File deleted or heavily rewritten → mark `auto-resolved`.
   - File untouched → carry forward unchanged, still `open`.
5. Scan only the changed files (single thread — the list is small, no sub-agents needed) across all categories for new issues. Assign `risk`, `cluster_id`, and `depends_on` to any new findings the same way Deep Mode does.
6. Update the ledger: add new findings as `open`, keep carried-forward entries as-is, mark auto-resolved ones. Set `last_scanned_commit` to current HEAD, `last_mode` to `quick`, `last_scan_date` to today, increment `consecutive_quick_count` by 1.
7. Commit the ledger immediately:
   ```
   git add .claude/refactor-log.json
   git commit -m "chore(refactor): update ledger after quick scan"
   ```

---

## Step 6 — Present findings

Group by category, prioritize within each. Split into three sections so the user can see real progress instead of a flat repeated list:

- **New** — found for the first time this run
- **Still Open** — found in a previous run, not yet fixed, still valid
- **Auto-Resolved** — found in a previous run, file since deleted/rewritten, no longer applicable

Priority levels:
- High   — architectural issues, security concerns, major code smells
- Medium — redundant code, missing abstractions, inconsistent patterns
- Low    — minor improvements, style inconsistencies, small optimizations

Present in this format:

─────────────────────────────────
REFACTORING REPORT   (mode: {Deep/Quick})
─────────────────────────────────
NEW                    {X issues}
STILL OPEN             {X issues carried over}
AUTO-RESOLVED          {X issues closed automatically}
─────────────────────────────────

ARCHITECTURE          {X issues — High: N, Medium: N, Low: N}
─────────────────────────────────
[New][High][Safe] Brief description
      File: path/to/file.php
      Why: one sentence explanation

[Still Open][Medium][Needs Review] ...

CODE QUALITY          {X issues — High: N, Medium: N, Low: N}
─────────────────────────────────
...

PERFORMANCE           {X issues}
─────────────────────────────────
...

TESTS                 {X issues}
─────────────────────────────────
...

DEPENDENCIES          {X issues}
─────────────────────────────────
...

─────────────────────────────────
TOTAL: {N} issues found ({N} new, {N} still open)
─────────────────────────────────

Then present multi-select category selection using the AskUserQuestion tool:

  Question:    "Which categories to apply?"
  multiSelect: true
  Options (include only categories that have at least one `new` or `still open` finding; max 4):
    - Architecture   — "{N} issues: {one-line summary}"
    - Code Quality   — "{N} issues: {one-line summary}"
    - Tests          — "{N} issues: {one-line summary}"
    - Dependencies   — "{N} issues: {one-line summary}"

  If more than 4 categories qualify, merge the smallest two into
  one option (e.g. "Tests & Dependencies").

  Selecting no options = skip. Do not add an explicit All or Skip option.

Wait for response before proceeding.

---

## Step 7 — Apply refactoring

Apply selected categories one at a time. Complete steps 7.1–7.4 fully for each category before starting the next. Do not batch changes from multiple categories into a single commit.

### 7.1 Order the work

- Take all findings in this category (across all clusters they belong to).
- Sort by `depends_on`: a finding cannot be applied before every finding in its `depends_on` list has already been applied.
- Findings sharing a `cluster_id` are handled together, in one pass, by a single agent — never split a cluster across parallel sub-agents, since they may touch the same file.
- Clusters with no shared files can be worked through one after another; there's no need to parallelize fixing the way scanning was parallelized — the discovery work is already done, and applying fixes safely matters more than applying them fast.

### 7.2 Apply safe findings

- For each cluster, apply every finding tagged `risk: safe` in dependency order.
- No need to pause for user input on these — proceed automatically.

### 7.3 Confirm needs-review findings

- For each finding tagged `risk: needs-review`, present it to the user individually (or batched if several are related) via AskUserQuestion before touching any code:

  AskUserQuestion:
    question: "This fix needs a judgment call: {finding description}. Apply it or skip it?"
    header:   "Needs review"
    multiSelect: false
    options:
      - label: "Apply as suggested"
        description: "{brief description of the fix}"
      - label: "Skip"
        description: "Leave this finding open, don't change this code"

- If skipped, set the finding's ledger `status` to `skipped-by-user` (not `open`) so it stops resurfacing every run. Only re-surface a `skipped-by-user` finding if the surrounding code has changed significantly — the original context has shifted enough that the user may want to reconsider.

### 7.4 Test, lint, commit

After all clusters in this category are applied:
- Run tests — stop and inform if tests fail. Do not proceed to the next category if tests fail; inform the human and wait for resolution before continuing.
- Run lint and formatter.
- Commit everything from this category in one commit:
  refactor({category}): {brief summary of changes}
- Update the corresponding ledger entries: `status` to `fixed` (or `skipped-by-user` per 7.3), with `resolved_commit` and `resolved_date` set for fixed ones.
- Then begin the next selected category from step 7.1.

For any finding in this category the user did not select this run at the category level (Step 6), leave its ledger `status` as `open`.

Example commits:
  refactor(architecture): extract business logic from controllers
  refactor(quality): remove duplicate helper methods
  refactor(tests): update outdated assertions

### 7.5 Scoped verification pass

After all selected categories have been applied and committed, run one lightweight check — reusing the Quick Mode mechanics (Step 5B), but scoped only to the files touched during this Step 7, not the whole repo:

- Re-check each `fixed` finding's file to confirm the issue is actually gone, not just that an edit was made.
- Scan the same touched files once more for anything new the fixes themselves may have introduced.
- Update the ledger accordingly: confirmed fixes stay `fixed`; anything not actually resolved reverts to `open` with a note; any newly introduced issue is added as a new `open` finding.

This step is cheap — it only covers files already touched this session, not a fresh full scan — and it's what tells the user honestly whether this run's fixes fully landed or quietly created new small issues.

---

## Step 8 — Merge and cleanup

Use AskUserQuestion:
  AskUserQuestion:
    question: "Refactoring applied. What would you like to do next?"
    header:   "Next step"
    multiSelect: false
    options:
      - label: "Merge to main (Recommended)"
        description: "Merge refactor branch into main automatically and delete the branch"
      - label: "Open PR"
        description: "Push branch and open a pull request for review"
      - label: "Leave branch"
        description: "Leave the refactor branch as-is for now"

Wait for response before proceeding.

**Option 1 — Auto merge:**
- Switch to main
- Merge refactor/{timestamp} into main --no-ff
  with message: refactor(project): apply refactoring {timestamp}
- Delete refactor branch locally and remotely
- Push main

**Option 2 — PR:**
- Push refactor/{timestamp} to remote (including the updated ledger)
- Inform user:
  "Branch pushed. Open a PR to merge into main when ready."

**Option 3 — Leave as-is:**
- Inform user:
  "Branch refactor/{timestamp} left intact locally."

---

## Step 9 — Confirm completion

Report:

─────────────────────────────────
REFACTORING COMPLETE
─────────────────────────────────
Branch:   refactor/{timestamp}
Mode:     {Deep/Quick}
Applied:
- Architecture: {N} changes
- Code Quality: {N} changes
- Performance:  {N} changes
- Tests:        {N} changes
- Dependencies: {N} changes

Ledger:         {N} still open, {N} auto-resolved, {N} newly fixed, {N} skipped by user
Verification:   {N} fixes confirmed resolved, {N} newly introduced issues found
Commits made:   {N}
Tests passing:  yes/no
Outcome:        {merged to main / PR pending / branch left intact}
─────────────────────────────────

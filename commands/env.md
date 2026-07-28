---
description: Audit and fix .env files and .gitignore — sync, format, detect missing vars, flag hardcoded values
---

# env

Scan `.env` files, `.gitignore`, and source code to find and fix environment configuration issues.

---

## Step 1 — Branch check

Only proceed if on `main` or `master`.
If on any other branch, stop and inform the user:

"env must be run on main or master.
Current branch is {branch}. Please switch and re-run."

---

## Step 2 — Scan

Read-only. Do not modify anything in this step.

### 2.1 — Detect env files

Find all `.env*` files in the project. For each file found, infer its purpose from the filename and context. If purpose cannot be determined, label as unknown.

### 2.2 — Compare entries across env files

For each env file, extract all key names (ignore values).

Build a matrix of all keys across all files. Find mismatches:
- Keys present in one file but missing from others
- For each mismatch, note which files have the key and which don't

### 2.3 — Scan source code for env var references

Detect the project's language and framework, then scan source files for env var access patterns appropriate for the detected stack. Collect every unique key referenced in source code.

### 2.4 — Cross-reference code vs env files

Using findings from 2.2 and 2.3:

**Missing from env files:** Keys referenced in source code that do not appear in any env file. These would silently fail or use undefined values at runtime.

**Never referenced in code:** Keys present in env files that do not appear anywhere in source code. These may be stale or only used at the infrastructure level — flag but do not auto-remove.

### 2.5 — Check secret vs placeholder correctness

Two complementary checks:

**`.env.example` — no real secrets allowed**
If no `.env.example` exists, skip this check silently.
Flag any value that looks like a real credential. Redact flagged values in the report — show only the first 4 and last 4 characters (e.g. `sk-a...xyz9`).

**All other env files — no placeholders allowed**
Flag any key whose value looks like an unfilled placeholder. These would cause silent failures at runtime.

### 2.6 — Scan for hardcoded values in source code

Scan source files for values that should be env vars. For each finding, suggest a descriptive env var name.

Exclude test fixtures, mock data, and constants that are genuinely environment-agnostic.

### 2.7 — Check env file formatting

Check each env file for formatting issues. Also check whether comment-based category groupings exist and whether all keys belong to the right group.

### 2.8 — Scan .gitignore

Check `.gitignore` for formatting issues, missing stack-appropriate entries, and entries that should not be ignored (overly broad or harmful patterns).

Run `git ls-files` and cross-check against `.gitignore` patterns — flag any tracked file that matches, since gitignore will not protect already-tracked files.

---

## Step 3 — Report

Present all findings grouped by category. Use this format:

```
─────────────────────────────────
ENV AUDIT REPORT
─────────────────────────────────
ENV FILES FOUND
─────────────────────────────────
.env              local development
.env.example      onboarding template
.env.staging      staging deployment
.env.production   production deployment

─────────────────────────────────
ENV SYNC          {X issues}
─────────────────────────────────
KEY_NAME          missing from: .env.staging, .env.production
KEY_NAME          missing from: .env.production

─────────────────────────────────
MISSING FROM ENV  {X issues}
─────────────────────────────────
Keys referenced in code but not in any env file:
- KEY_NAME        src/config.js:14
- KEY_NAME        src/api/client.ts:8

─────────────────────────────────
NEVER REFERENCED  {X issues}
─────────────────────────────────
Keys in env files with no code reference (may be infrastructure-only):
- KEY_NAME        .env, .env.staging

─────────────────────────────────
SECRET/PLACEHOLDER ISSUES        {X found}
─────────────────────────────────
.env.example — real secrets found (should be placeholders):
- KEY_NAME        sk-a...xyz9
- KEY_NAME        ghp_A...B3k

.env, .env.staging — placeholders found (should be real values):
- KEY_NAME        .env            your-key-here
- KEY_NAME        .env.staging    REPLACE_ME

─────────────────────────────────
HARDCODED VALUES  {X issues}
─────────────────────────────────
[High] API key hardcoded as string literal
      File: src/api/stripe.js:23
      Value: pk_live_A...xyz (redacted)
      Suggested env var: STRIPE_PUBLISHABLE_KEY

[Medium] Environment-specific URL hardcoded
      File: src/config.js:5
      Value: https://api.production.com
      Suggested env var: API_BASE_URL

─────────────────────────────────
ENV FORMATTING    {X issues}
─────────────────────────────────
.env:
- 2 duplicate keys
- Inconsistent spacing around = on 3 lines
.env.staging:
- Trailing whitespace on 1 line

─────────────────────────────────
GITIGNORE         {X issues}
─────────────────────────────────
Missing entries (stack: Node.js):
- node_modules/
- .env.local
- dist/

Tracked files matching gitignore patterns:
- .env.local      already tracked by git — run git rm --cached .env.local to untrack

Entries that should not be ignored:
- *.js             overly broad — would exclude source files
- config/          matches committed application code — likely added by mistake

Formatting:
- 3 duplicate entries found
- Missing category groupings

─────────────────────────────────
TOTAL: {N} issues across {N} categories
─────────────────────────────────
```

Then ask which categories to fix:

AskUserQuestion:
  question: "Which categories would you like to fix?"
  header:   "Fix categories"
  multiSelect: true
  options (include only if the category has findings):
    - label: "Env sync"
      description: "{N} keys missing from one or more env files"
    - label: "Missing from env"
      description: "{N} keys referenced in code but not in any env file"
    - label: "Hardcoded values"
      description: "{N} values in source code that should be env vars"
    - label: "Cleanup"
      description: "Formatting and grouping fixes for env files and .gitignore"

Never referenced and Secret/placeholder issues are always reported but never auto-fixed:
- Never referenced: flagged for manual review — do not auto-remove, as they may be infrastructure-only
- Secret/placeholder issues: warn the user to act manually — do not modify values

Wait for response before proceeding.

---

## Step 4 — Apply fixes

Apply selected categories one at a time.

### Env sync

Add missing keys with placeholder values. Do not overwrite existing values.

### Missing from env

Add missing keys to `.env.example` with placeholders. Create `.env.example` if it does not exist.

### Hardcoded values

Ask for confirmation before touching each finding:

AskUserQuestion:
  question: "Replace hardcoded value in {file}:{line} with env var {SUGGESTED_NAME}?"
  header:   "Hardcoded value"
  multiSelect: false
  options:
    - label: "Replace"
      description: "Substitute with the appropriate env var access pattern for this stack"
    - label: "Skip"
      description: "Leave this value as-is"

If Replace selected:
- Replace the hardcoded value in source code
- Add the key with the original value to `.env` if absent
- Add the key with a placeholder to `.env.example` and all other env files if absent
- Inform the user which files were updated and that real values need to be filled in per environment

### Cleanup

For env files: fix formatting, remove duplicate keys (keep last occurrence), add category groupings if absent. Preserve existing group order — do not reorder entries within a group.

For `.gitignore`: fix formatting, remove duplicates, add missing stack-appropriate entries, remove overly broad or harmful entries, add category groupings if absent. Preserve existing group order.

For tracked files matching gitignore patterns, ask for confirmation before untracking:

AskUserQuestion:
  question: "{file} is tracked by git but matches a .gitignore pattern. Untrack it with git rm --cached?"
  header:   "Untrack file"
  multiSelect: false
  options:
    - label: "Untrack"
      description: "Run git rm --cached {file} so .gitignore takes effect"
    - label: "Skip"
      description: "Leave it tracked — handle manually later"

If Untrack selected → run `git rm --cached {file}`.
If Skip selected → note it in the Step 6 report under manual action.

---

## Step 5 — Commit

After all fixes are applied, ask:

AskUserQuestion:
  question: "Fixes applied. Commit now?"
  header:   "Commit"
  multiSelect: false
  options:
    - label: "Commit (Recommended)"
      description: "chore(env): audit and fix env configuration"
    - label: "Cancel"
      description: "Leave changes uncommitted — commit manually when ready"

If Commit selected:
  Stage only the files modified during this command. Never use `git add -A`.
  git commit -m "chore(env): audit and fix env configuration"

---

## Step 6 — Confirm completion

Report:

```
ENV COMPLETE
─────────────────────────────────
Env sync:         {N} keys added across {N} files
Missing from env: {N} keys added to .env.example
Hardcoded values: {N} replaced, {N} skipped
Cleanup:          {N} env files cleaned, .gitignore: {N} entries added, {N} removed, {N} duplicates removed, groupings added: yes/no
─────────────────────────────────
Needs manual action:
- Secret/placeholder issues: {N} flagged — replace real secrets in .env.example with placeholders; fill in real values where placeholders remain in other env files
- Never referenced: {N} flagged — review and remove if stale
- Tracked files:  {list} — run git rm --cached to untrack
─────────────────────────────────
Committed: yes / no
```

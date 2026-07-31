---
description: Audit and fix .env files and .gitignore — sync, format, detect missing vars, flag hardcoded values
---

# env

Scan `.env` files, `.gitignore`, and source code to find and fix environment configuration issues.

---

## Step 1 — Scan

Read-only. Do not modify anything in this step.

Scan all env files and source files completely before proceeding. Never stop early — a partial scan produces an incomplete report and missed findings will silently persist.

### 1.1 — Detect env files

Find all `.env*` files in the project. For each file found, infer its purpose from the filename and context. If purpose cannot be determined, label as unknown.

### 1.2 — Compare entries across env files

For each env file, extract all key names (ignore values).

Build a matrix of all keys across all files. Find mismatches:
- Keys present in one file but missing from others
- For each mismatch, note which files have the key and which don't

### 1.3 — Scan source code for env var references

Detect the project's language and framework, then scan source files for env var access patterns appropriate for the detected stack. Collect every unique key referenced in source code.

### 1.4 — Cross-reference code vs env files

Using findings from 2.2 and 2.3:

**Missing from env files:** Keys referenced in source code that do not appear in any env file (completely missing, not just missing from some). Keys missing from some but not all env files are covered by the Env sync check. These would silently fail or use undefined values at runtime.

**Never referenced in code:** Keys present in env files that do not appear anywhere in source code. These may be stale or only used at the infrastructure level — flag but do not auto-remove.

### 1.5 — Check secret vs placeholder correctness

Two complementary checks:

**`.env.example` — no real secrets allowed**
If no `.env.example` exists, skip this check silently.
Flag any value that looks like a real credential. Redact flagged values in the report — show only the first 4 and last 4 characters (e.g. `sk-a...xyz9`).

**All other env files — no placeholders allowed**
Flag any key whose value looks like an unfilled placeholder. These would cause silent failures at runtime.

### 1.6 — Scan for hardcoded values in source code

Scan source files for values that should be env vars. For each finding, suggest a descriptive env var name.

Exclude test fixtures, mock data, and constants that are genuinely environment-agnostic.

### 1.7 — Check env file formatting

Check each env file for formatting issues. Also check whether comment-based category groupings exist and whether all keys belong to the right group.

### 1.8 — Scan .gitignore

Check `.gitignore` for formatting issues, missing stack-appropriate entries, and entries that should not be ignored (overly broad or harmful patterns).

Run `git ls-files` and cross-check against `.gitignore` patterns — flag any tracked file that matches, since gitignore will not protect already-tracked files.

---

## Step 2 — Report

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

Then ask which targets to work through:

AskUserQuestion:
  question: "Which findings would you like to work through?"
  header:   "Fix scope"
  multiSelect: true
  options (include only if that target has findings):
    - label: "Env files"
      description: "All env-file findings: sync, missing keys, secrets/placeholders, hardcoded values, unreferenced keys, formatting"
    - label: ".gitignore"
      description: "All .gitignore findings: missing entries, tracked files, overly-broad entries, formatting"

Selecting a target walks through each of its findings in Step 3 — every sub-flow shows what it found and asks, so you can skip any you do not want. Selecting neither exits without changes (the report above already lists everything).

Wait for response before proceeding.

---

## Step 3 — Apply fixes

Run the sub-flows for each selected target: the **Env files** sub-flows if "Env files" was selected, the **.gitignore** sub-flows if ".gitignore" was selected. Within a selected target, run a sub-flow only if it has findings — each shows its findings and asks, with a Skip.

Apply each sub-flow one at a time. Complete each fully before moving to the next. Never mark the command done if a sub-flow was only partially applied — partial fixes leave the project in an inconsistent state.

---

**Env files sub-flows** — run these only if "Env files" was selected.

### Secret cleanup (.env.example)

Only run this section if real secrets were found in `.env.example`.

Display the flagged keys with their planned replacement:

```
.env.example — real secrets found. Planned replacements:

KEY_NAME        sk-a...xyz9    →  <YOUR_KEY_NAME>
GITHUB_TOKEN    ghp_A...B3k    →  <YOUR_GITHUB_TOKEN>
```

Then ask:

AskUserQuestion:
  question: "Replace real secrets in .env.example with placeholders?"
  header:   "Secret cleanup"
  multiSelect: false
  options:
    - label: "Replace all"
      description: "Replace all {N} flagged values with the placeholders shown above"
    - label: "Skip"
      description: "Leave .env.example unchanged — handle manually"

If Replace all selected: replace each flagged key's value with `<YOUR_{KEY_NAME}>`.
If user selects Other and types instructions: follow their instructions exactly. Apply only what was explicitly requested.
If Skip selected: note in Step 5 report under manual action.

Do not touch values in any other env file during this section.

### Placeholder cleanup (other env files)

Only run this section if placeholders were found in non-example env files.

Display findings grouped by file:

```
Other env files — placeholders found (should be real values):

.env
  KEY_NAME        your-key-here
  DB_PASSWORD     <YOUR_DB_PASSWORD>

.env.staging
  API_SECRET      REPLACE_ME
  KEY_NAME        your-key-here

Fill in the real values in each file, then mark done.
```

Then ask:

AskUserQuestion:
  question: "Have you filled in the placeholder values?"
  header:   "Placeholders"
  multiSelect: false
  options:
    - label: "Done — move to next step"
      description: "Continue to the next sub-flow"
    - label: "Skip"
      description: "Leave these placeholders as-is and move on"

Do not recheck or validate what was filled in.

---

### Env sync

Process one env file at a time. For each file with missing keys:

1. Display the missing keys and where they exist:

```
.env.staging — 3 keys missing

KEY_NAME       exists in: .env, .env.production
DB_URL         exists in: .env only
API_SECRET     exists in: .env, .env.production
```

2. Let the developer add the keys manually to the file.

3. Ask to proceed:

AskUserQuestion:
  question: "Have you finished updating .env.staging?"
  header:   "Env sync"
  multiSelect: false
  options:
    - label: "Done — move to next file"
      description: "Continue to the next env file with missing keys"
    - label: "Skip this file"
      description: "Leave .env.staging as-is and move on"

4. Move to the next file. Do not recheck or validate what was added.

### Missing from env

Display keys that are completely missing from all env files with their code reference locations:

```
Keys referenced in code but missing from all env files:

KEY_NAME        src/config.js:14
KEY_NAME        src/api/client.ts:8

Add these to .env and all relevant env files manually.
```

Then wait:

AskUserQuestion:
  question: "Have you finished adding the missing keys to your env files?"
  header:   "Missing from env"
  multiSelect: false
  options:
    - label: "Done — move to next step"
      description: "Continue to the next sub-flow"
    - label: "Skip"
      description: "Leave these keys unresolved and move on"

### Hardcoded values

Display all findings:

```
Hardcoded values found — should be moved to env vars:

[High] src/api/stripe.js:23
  Value: pk_live_A...xyz (redacted)
  Suggested env var: STRIPE_PUBLISHABLE_KEY

[Medium] src/config.js:5
  Value: https://api.production.com
  Suggested env var: API_BASE_URL
```

Then ask:

AskUserQuestion:
  question: "Move hardcoded values to env vars?"
  header:   "Hardcoded values"
  multiSelect: false
  options:
    - label: "Replace all"
      description: "Substitute in source + add to all env files using the suggested names above"
    - label: "Skip"
      description: "Leave all hardcoded values as-is"

If Replace all selected:
- If there are more than 10 findings, process one source file at a time and commit after each file before moving to the next.
- For each finding:
  - Replace the hardcoded value in source with the env var access pattern for the detected stack
  - Add the key + real value to `.env` and all other non-example env files if absent
  - Add the key + placeholder (`<YOUR_{KEY_NAME}>`) to `.env.example` if absent
- Inform the dev which source files and env files were updated

If user selects Other and types instructions: follow their instructions exactly. Apply only what was explicitly requested.
If Skip selected: no changes.

### Never referenced

Display keys that exist in env files but are never referenced in source code, grouped by file, with a note on whether each looks infrastructure-only or stale:

```
Keys in env files with no code reference:

KEY_NAME        .env, .env.staging     (looks infrastructure-only)
OLD_API_KEY     .env                   (looks stale — similar to STRIPE_API_KEY)
DEBUG_TOKEN     .env.staging           (looks stale)

What would you like to do? You can say things like:
- "delete OLD_API_KEY and DEBUG_TOKEN, keep KEY_NAME"
- "replace config/database.js line 12 with KEY_NAME"
- "delete all" / "keep all"
```

Read the developer's free-text response and act accordingly. Apply only what was explicitly requested.

### Env formatting

Only run this section if env formatting issues were found.

AskUserQuestion:
  question: "Fix formatting across all env files?"
  header:   "Env formatting"
  multiSelect: false
  options:
    - label: "Proceed"
      description: "Fix formatting, remove duplicates, and add category groupings consistently across all env files"
    - label: "Skip"
      description: "Leave env file formatting as-is"

If Proceed selected:
- Apply consistent formatting rules across all env files: spacing around `=`, line endings, no trailing whitespace
- Remove duplicate keys (keep last occurrence) in each file
- Add category groupings if absent — use the same grouping structure across all env files
- Preserve existing group order — do not reorder entries within a group

If Skip selected: no changes to env files.

---

**.gitignore sub-flows** — run these only if ".gitignore" was selected.

### Missing entries

Only run this section if missing entries were found in Step 1.8.

Display all missing entries with a reason:

```
Missing .gitignore entries (stack: Node.js):

node_modules/    dependency directory — never committed
.env.local       local override file — contains machine-specific values
dist/            build output — regenerated on every build
```

Then ask:

AskUserQuestion:
  question: "Add missing entries to .gitignore?"
  header:   "Missing entries"
  multiSelect: false
  options:
    - label: "Add all"
      description: "Add all {N} missing entries to .gitignore"
    - label: "Skip"
      description: "Leave .gitignore unchanged"

If Add all selected: add all entries to .gitignore, then proceed immediately to tracked files check below.
If Skip selected: skip to next section.

### Tracked files

Run this section in two cases:
- Automatically after Missing entries if new entries were added — check for tracked files matching those new patterns
- If pre-existing tracked files were found in Step 1.8 — handle those too

For each tracked file, ask:

AskUserQuestion:
  question: "{file} is tracked by git but matches a .gitignore pattern. Untrack it?"
  header:   "Untrack file"
  multiSelect: false
  options:
    - label: "Untrack"
      description: "Run git rm --cached {file} so .gitignore takes effect"
    - label: "Skip"
      description: "Leave it tracked — handle manually later"

If Untrack selected → run `git rm --cached {file}`.
If Skip selected → note it in the Step 5 report under manual action.

### Entries to remove

Only run this section if overly broad or harmful entries were found in Step 1.8.

Display all entries to remove with a reason:

```
Entries that should not be ignored:

*.js             overly broad — would exclude source files
config/          matches committed application code — likely added by mistake
```

Then ask:

AskUserQuestion:
  question: "Remove these entries from .gitignore?"
  header:   "Entries to remove"
  multiSelect: false
  options:
    - label: "Remove all"
      description: "Remove all {N} flagged entries from .gitignore"
    - label: "Skip"
      description: "Leave .gitignore unchanged"

If Remove all selected: remove all flagged entries from .gitignore.
If Skip selected: no changes.

### Gitignore formatting

Only run this section if formatting issues were found in Step 1.8.

AskUserQuestion:
  question: "Fix .gitignore formatting?"
  header:   "Gitignore formatting"
  multiSelect: false
  options:
    - label: "Proceed"
      description: "Fix formatting, remove duplicate entries, and add category groupings"
    - label: "Skip"
      description: "Leave .gitignore formatting as-is"

If Proceed selected:
- Fix formatting and remove duplicate entries
- Add category groupings if absent
- Preserve existing group order — do not reorder entries within a group

If Skip selected: no changes.

---

## Step 4 — Commit

After all fixes are applied:
  git commit -m "chore(env): audit and fix env configuration"

---

## Step 5 — Confirm completion

Report:

```
ENV COMPLETE
─────────────────────────────────
Env files
  Secret cleanup:   .env.example — {N} replaced, {N} skipped
  Placeholder:      {N} keys flagged across {N} files — done / skipped
  Env sync:         {N} keys added across {N} files
  Missing from env: {N} keys identified — added manually
  Hardcoded values: {N} replaced, {N} skipped
  Never referenced: {N} deleted, {N} kept
  Env formatting:   {N} files fixed
.gitignore
  Entries:          {N} missing added, {N} overly-broad removed
  Tracked files:    {N} untracked, {N} skipped
  Formatting:       {N} duplicates removed, groupings added: yes/no
─────────────────────────────────
Needs manual action:
- Tracked files skipped: {list} — run git rm --cached to untrack manually
─────────────────────────────────
Committed: yes / no
```

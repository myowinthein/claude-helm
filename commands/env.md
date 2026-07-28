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

### 2.5 — Detect real secrets vs placeholders

For each key-value pair across all env files, classify the value:

- **Placeholder** — looks like a template value: `your-key-here`, `REPLACE_ME`, `xxx`, empty string, `changeme`, `secret`, `password`, etc.
- **Real secret** — looks like an actual credential: long random strings, tokens starting with known prefixes (`sk-`, `pk_`, `ghp_`, `xoxb-`, `ya29.`, etc.), Base64-encoded blobs, connection strings with passwords.

Flag any real secrets found, noting the file and key name. Redact values in the report — show only the first 4 and last 4 characters (e.g. `sk-a...xyz9`).

### 2.6 — Scan for hardcoded values in source code

Scan source files for values that should be env vars:

- API keys, tokens, or credentials hardcoded as string literals
- Base URLs or hostnames tied to a specific environment (e.g. `https://api.production.com`, `localhost:5432`)
- Database connection strings
- Third-party service IDs (Stripe publishable keys, Sentry DSNs, Firebase config objects, etc.)
- Magic strings that vary between environments (port numbers, feature flags, timeout values that differ per env)

For each finding, suggest a descriptive env var name (e.g. `STRIPE_PUBLISHABLE_KEY`, `DATABASE_URL`, `API_BASE_URL`).

Exclude: test fixtures, mock data, constants that are genuinely environment-agnostic.

### 2.7 — Check env file formatting

For each env file, check for formatting issues:
- Duplicate keys
- Inconsistent spacing around `=`
- Keys not in `UPPER_SNAKE_CASE`
- Trailing whitespace or Windows-style line endings
- Missing category groupings — if the file has no comment-based groups, flag it; if groups exist, check whether all keys belong to the right group

### 2.8 — Scan .gitignore

Check `.gitignore`:
- **Formatting issues** — inconsistent spacing, duplicate entries, missing category groupings
- **Missing stack-appropriate entries** — based on detected language and framework, identify files or folders that should be gitignored but are not (e.g. `.env`, `node_modules/`, `__pycache__/`, `.DS_Store`, `*.log`, `dist/`, `.venv/`)
- **Entries matching tracked files** — run `git ls-files` and cross-check against `.gitignore` patterns; flag any tracked file that matches a gitignore pattern, since gitignore will not protect already-tracked files
- **Entries that should not be ignored** — flag patterns that are overly broad or likely harmful: patterns that would exclude source files, patterns that match nothing in the project, or entries that appear to have been added by mistake (e.g. ignoring a directory that contains committed application code). Suggest removing them.

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
REAL SECRETS      {X found}
─────────────────────────────────
[WARNING] These look like real credentials, not placeholders:
- KEY_NAME        .env          sk-a...xyz9
- KEY_NAME        .env.staging  ghp_A...B3k

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
- .env.local      already tracked by git — gitignore will not protect it

Formatting:
- 3 duplicate entries found
- Missing section groupings

─────────────────────────────────
TOTAL: {N} issues across {N} categories
─────────────────────────────────
```

Then ask which categories to fix:

AskUserQuestion:
  question: "Which categories would you like to fix?"
  header:   "Fix categories"
  multiSelect: true
  options (include only categories with findings, max 4):
    - label: "Env sync"
      description: "{N} keys missing from one or more env files"
    - label: "Missing from env"
      description: "{N} keys referenced in code but not in any env file"
    - label: "Hardcoded values"
      description: "{N} values in source code that should be env vars"
    - label: "Formatting"
      description: "{N} formatting issues across env files and .gitignore"

If more than 4 categories qualify, merge the smallest into one option.

Never referenced and Real secrets are always reported but never auto-fixed:
- Never referenced: flagged for manual review — do not auto-remove, as they may be infrastructure-only
- Real secrets: warn the user to rotate and replace with placeholders — do not modify values

Wait for response before proceeding.

---

## Step 4 — Apply fixes

Apply selected categories one at a time.

### Env sync

For each key missing from an env file: add it with a placeholder value.
Use the existing placeholder style found in `.env.example` if present, otherwise use `your-{key-name}-here`.
Add a comment above new entries: `# Added by /helm:env — set the real value for this environment`.
Do not overwrite existing values.

### Missing from env

Add each missing key to `.env.example` with a placeholder value and a comment indicating where it is referenced.
If no `.env.example` exists, create one.
Inform the user: "These keys have been added to `.env.example`. Add real values to `.env` and environment-specific files manually."

### Hardcoded values

For each finding, present the specific change needed and ask for confirmation before touching code:

AskUserQuestion:
  question: "Replace hardcoded value in {file}:{line} with env var {SUGGESTED_NAME}?"
  header:   "Hardcoded value"
  multiSelect: false
  options:
    - label: "Replace"
      description: "Substitute the hardcoded value with process.env.{SUGGESTED_NAME} (or equivalent for this stack)"
    - label: "Skip"
      description: "Leave this value as-is"

If Replace selected:
- Substitute the hardcoded value in source code with the appropriate env var access pattern for the detected language
- Add the key with a placeholder to `.env.example`
- Add the key with the original value to `.env` if it does not already exist

### Formatting

For each env file with formatting issues:
- Remove duplicate keys (keep the last occurrence)
- Normalise spacing to `KEY=value` with no spaces around `=`
- Convert keys to `UPPER_SNAKE_CASE`
- Strip trailing whitespace and normalise to Unix line endings
- If the file already has comment-based category groups: preserve them, correct any misplaced keys, and do not reorder entries within a group
- If the file has no groups: infer categories from key names (e.g. `DB_*` → Database, `STRIPE_*` → Stripe, `JWT_*` → Auth) and add comment headers; do not reorder keys within an inferred group

For `.gitignore`:
- Remove duplicate entries
- If the file already has comment-based category groups: preserve them and do not reorder entries within a group
- If the file has no groups: infer categories (Dependencies, Environment, Build output, OS, Editor, etc.) and add comment headers
- Strip trailing whitespace

### .gitignore

Apply all three sub-fixes together:
- Add missing stack-appropriate entries, grouped by category with a comment header
- Remove duplicate entries
- For tracked files matching gitignore patterns, do not modify `.gitignore` — inform the user:
  "{file} is already tracked by git. Run `git rm --cached {file}` to untrack it, then gitignore will take effect."

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
Formatting:       {N} env files cleaned, .gitignore cleaned
.gitignore:       {N} entries added, {N} duplicates removed
─────────────────────────────────
Needs manual action:
- Real secrets:   {N} flagged — rotate and replace with placeholders
- Never referenced: {N} flagged — review and remove if stale
- Tracked files:  {list} — run git rm --cached to untrack
─────────────────────────────────
Committed: yes / no
```

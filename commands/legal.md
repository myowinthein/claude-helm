---
description: Scan project and generate GDPR-compliant legal documents based on what it does
---

# legal

Scan the project and generate legal documents based on what the
project actually does. Only generate documents that apply.

## Before starting

`/helm:legal` generates canonical, public-facing legal documents from the project's data profile, so it needs the full merged state and a clean landing path to main. Behavior depends on `git-strategy` in CLAUDE.md's Project Config (absence defaults to GitHub Flow, per git.md).

**Solo Mode** (`git-strategy: solo`):
- Only proceed if on `main` or `master`. If on any other branch, stop: "legal must be run on main or master in Solo Mode. Current branch is {branch} — switch and re-run."

**GitHub Flow** (`git-strategy: github-flow`, or absent):
- Record the current branch as `{original_branch}` — the workflow returns here at the end, whatever it was.
- Checkout a fresh branch from main's current tip: update local main (`git pull` if a remote exists), then `git checkout -b docs/legal-{YYYYMMDD}` from it. This happens unconditionally, regardless of what `{original_branch}` was — the generated documents are always scanned from main's own content, never from whatever the starting branch happened to contain, so there is nothing to check or reject about the starting branch itself.
- Call this branch `{branch}` for the rest of this command.
- If this command exits without writing anything at all (Step 2's "nothing selected" case), delete `{branch}` and return to `{original_branch}` before exiting — never leave the user stranded on an empty temporary branch it created. This does **not** apply once documents have been written but left uncommitted (Step 4's Cancel option) — that branch is deliberately preserved so the user can review or finish committing later; do not delete real drafted work.

## Step 1 — Project scan

Scan the codebase to understand the project's legal profile:

**App type**
- Chrome extension, web app, mobile app, desktop app, open source library
- Infer from manifest.json, package.json, composer.json, folder structure

**Data collection**
- Forms, auth systems, user accounts
- Analytics tools (Google Analytics, Mixpanel, Hotjar, GTM)
- Error tracking (Sentry, Bugsnag)
- Any personally identifiable information (PII) collected or stored

**Sensitive data (GDPR Art. 9 special category)**
- Health, medical, or fitness data; biometric or genetic data
- Race or ethnicity, religion, political views, trade-union membership, sexual orientation
- Detect from data fields, model schemas, or integrations that handle such data
- If any is present, the privacy policy must call it out — special-category data needs explicit consent and a stricter legal basis

**Children and minors**
- Is the service directed at, or knowingly accessible to, children?
- Age gates, minimum-age checks (13+ / 16+), parental-consent flows, kid-directed content or framing
- Drives the Children's Privacy section (GDPR Art. 8, and COPPA where US minors are involved)

**Third party integrations**
- Payment processors (Stripe, PayPal, Paddle)
- Social auth (Google, GitHub, Facebook OAuth)
- Cloud services (AWS, GCP, Firebase)
- Marketing or tracking tools

**Email and marketing communications**
- Email service providers (Mailchimp, SendGrid, Postmark, Klaviyo, Resend, and similar)
- Newsletter or marketing-list signup forms
- Distinguish transactional email (account/receipts) from marketing email — marketing requires consent under GDPR and the ePrivacy rules, and belongs in the privacy policy

**Monetization**
- Payment processing, subscription logic, pricing pages
- Free vs paid tiers

**Content model**
- Does the app let users generate or upload content?
- Does the app provide advice (financial, health, legal, AI-generated)?

**AI features**
- Does the app use AI to generate recommendations presented as facts?
- Does it use a BYOK model (user provides their own API key)?

**Framework and output detection**

Scan the project to determine which scenario applies. Use config files, package
manifests, folder structure, and any other available signals to decide:

  Static site generator (Jekyll, Hugo, Eleventy, and similar):
  - Format: `.md`
  - Output path: the generator's content or docs directory, under a `legal/` subfolder
  - Detect the correct content directory from the generator's config

  JS framework with SSR or SSG (Next.js, Astro, Nuxt, SvelteKit, and similar):
  - Format: `.mdx`
  - Output path: the framework's pages or content directory, under a `legal/` subfolder
  - Detect the correct pages directory from the framework's config and folder structure

  SPA or plain HTML site (no server-side rendering, no static site generator):
  - Format: `.html` with subfolder structure (`legal/privacy-policy/index.html`, etc.)
  - Output path: the directory that is served as the web root (e.g. `public/`, `dist/`, or root)

  No web project detected (pure library, CLI tool, or non-web repo):
  AskUserQuestion:
    question: "No web framework detected. Where should legal documents be written, and in what format?"
    header:   "Output"
    multiSelect: false
    options:
      - label: "legal/ as Markdown (Recommended)"
        description: "Standard location for repos without a web front end"
      - label: "legal/ as HTML"
        description: "Bare semantic HTML — suitable for plain HTML sites"
      - label: "Custom path"
        description: "Specify a path and format manually"

  This question is itself the output-location choice for non-web repos — skip the "Confirm output location" step below when it was asked.

**CSS style detection (HTML output only)**

When the resolved format is `.html`, scan the project to determine whether styles
are applied globally or scoped:

  Global CSS (element and class selectors that apply site-wide):
  → Output bare semantic HTML. The site's stylesheet handles rendering.

  Scoped or utility-first CSS (styles tied to components or utility classes,
  where unstyled semantic HTML receives no visual treatment):
  → Find the prose or layout container used by existing pages and wrap the
  generated semantic HTML in the same element and classes inside `<body>`.

  Cannot determine:
  → Wrap the generated semantic HTML in `<main>` as a safe default.

Record the resolved format and output path before proceeding.

**Confirm output location**

Run this only for the auto-detected scenarios (SSG, JS framework, SPA/plain HTML), where the path and format were resolved without asking. Skip it for the no-web case — the "No web project detected" question already served as the location choice.

After resolving the format and path, confirm with the user before continuing:

  AskUserQuestion:
    question: "Legal docs will be written to {resolved-path} as {Format} files. Confirm or choose a different location."
    header:   "Output location"
    multiSelect: false
    options:
      - label: "Use {resolved-path} (Recommended)"
        description: "{Format} — detected from {framework/scenario name}"
      - label: "Custom path"
        description: "Specify a different output path and format"

  If Custom path selected → ask the user to type the path and preferred format before proceeding.
  If confirmed → proceed with the resolved path and format.

**Contact point**

Legal documents must give users a real way to reach the owner (mandatory under GDPR Art. 13). Resolve it now — do not generate anything until a valid contact point is confirmed. Never fall back to a bare placeholder.

Infer candidates from the project:
- Existing contact page — a contact / contact-us page in the project (e.g. `contact.html`, `contact.md`, a `/contact` route, `pages/contact.*`, `src/pages/contact.*`). Use its site URL or path.
- Repo issues URL — from the git remote, `package.json`, or `composer.json`.
- Support/contact email — from `package.json` (`bugs.email`, `author.email`), `composer.json` (`authors.email`), or a `SECURITY.md` / `CONTACT` file.

Ask which to use. Build the option list from the candidates found (include one option per candidate, in the order above), keeping the question at 2–4 options; if fewer than two candidates were found, add an "Enter a contact manually" option so there are at least two:

  AskUserQuestion:
    question: "How should users contact the owner in the legal documents? Required — a legal document needs a real contact."
    header:   "Contact point"
    multiSelect: false
    options:
      - label: "Contact page"                ← include only if a contact page was found; show its URL/path in the description
        description: "Point users to {contact page URL}"
      - label: "Open a repo issue"           ← include only if an issues URL was found; show the URL in the description
        description: "Users open an issue at {issues URL}"
      - label: "Email"                        ← include only if an email was found; show it in the description
        description: "Users email {email}"

  The built-in Other option also lets the developer type any contact. Whatever is chosen or typed must be a valid contact — an email address, a URL, or a postal address. If the response is empty or not a recognisable contact, re-ask; do not proceed without one.

Record the resolved contact and its type (email / issue URL / page / postal) for the Contact section in Step 3.

**Existing legal documents**
- Check whether the resolved output path exists.
- List which documents are already present using the resolved file extension.
- Read `.claude/legal-manifest.json` if it exists — it records which documents a previous run generated and the commit each was generated at. Classify each present document:
  - **Helm-generated** — its path is listed in the manifest. Safe to regenerate. Check whether it is stale: run `git log {generated_at_commit}..HEAD` and look for changes that could shift the legal profile scanned above in this step — new analytics, payment processors, auth providers, third-party integrations, or data collection (dependency manifests, config, integration code). If any are found, mark the document **may be outdated**. Do not edit it in place — regeneration is always a full rewrite.
  - **Foreign** — present on disk but not in the manifest. Likely hand-written or lawyer-reviewed; must not be overwritten without explicit confirmation.
- The published legal documents carry no marker of their own — this manifest is the only record of what helm generated, so end-user-facing files stay clean.

Record all findings, including each document's classification and staleness, before proceeding.

---

## Step 2 — Determine which documents to generate

Based on scan findings, mark each document as Recommended if it applies:

- privacy-policy.md — always Recommended
- terms.md — always Recommended
- cookie-policy.md — Recommended if non-essential cookies or analytics detected
- refund-policy.md — Recommended if payment processing detected
- eula.md — Recommended if Chrome extension, desktop app, or downloadable software detected
- disclaimer.md — Recommended if financial, health, legal advice or AI recommendations detected

AskUserQuestion supports a maximum of 4 options. Split into two questions.

Labeling applies to both questions below:
- Append `(Recommended)` per each document's recommendation rule (noted inline per option).
- For an existing document, append its classification suffix from Step 1: `(exists)` for an up-to-date Helm-generated file, `(exists — may be outdated)` for a Helm-generated file flagged stale (also mark it Recommended so the user regenerates it), or `(exists — not helm-generated)` for a Foreign file. This lets the user tell an auto-generated file from a hand-edited one, and a current file from a stale one.

Question 1 — core documents:
  AskUserQuestion:
    question: "Which core legal documents should be generated? (Jurisdiction: GDPR · Tone: plain English)"
    header:   "Core documents"
    multiSelect: true
    options:
      - label: "privacy-policy.md (Recommended)"
        description: "Explains what data is collected and how it is handled"
      - label: "terms.md (Recommended)"
        description: "Acceptable use, IP ownership, liability, governing law"
      - label: "eula.md"                            ← append "(Recommended)" if downloadable software
        description: "License grant, restrictions, and liability for installable software"
      - label: "disclaimer.md"                      ← append "(Recommended)" if AI content or advice
        description: "No-professional-advice notice and AI-generated content warning"

Question 2 — conditional documents (only ask if applicable based on scan findings):
  AskUserQuestion:
    question: "Any additional legal documents needed?"
    header:   "Additional documents"
    multiSelect: true
    options:
      - label: "cookie-policy.md"                   ← append "(Recommended)" if analytics detected
        description: "Required if non-essential cookies or analytics tools are present"
      - label: "refund-policy.md"                   ← append "(Recommended)" if payments detected
        description: "Required if payment processing is present"

  Skip Question 2 entirely if neither cookie-policy nor refund-policy is applicable
  based on scan findings (no analytics, no payments detected).

Generate only the documents selected across both questions.
If nothing is selected, exit without generating anything — proceed to Step 4, which writes nothing but still needs to run its GitHub Flow cleanup (delete `{branch}`, return to `{original_branch}`) if a temporary branch was created.
Selected Helm-generated documents that already exist are overwritten. Selected Foreign documents are confirmed individually before overwriting (see Step 3).

Wait for both responses before proceeding.

---

## Step 3 — Generate documents

Generate each selected document in plain English, GDPR compliant.
Use em-dashes sparingly — only when no other punctuation (comma, semicolon, colon, or a new sentence) works as well. When in doubt, restructure the sentence instead.
Write to the resolved output path using the resolved format.

**Before writing a document that already exists:**
- If it is **Helm-generated** (in the manifest) → overwrite silently.
- If it is **Foreign** (not in the manifest) → confirm first, since it may be hand-written or lawyer-reviewed:

  AskUserQuestion:
    question: "{file} already exists and was not generated by helm — it may be customized or lawyer-reviewed. Overwrite it?"
    header:   "Overwrite existing"
    multiSelect: false
    options:
      - label: "Overwrite"
        description: "Replace it with the freshly generated document"
      - label: "Skip"
        description: "Leave the existing file untouched"

  If Skip → do not write that document; note it in the completion report.

Do not write any marker into the documents themselves — they are published to end users and must stay clean.

**After generating**, write/update `.claude/legal-manifest.json` so future runs can tell helm-generated documents from hand-edited ones and detect staleness. Record each document actually written, with its path, the current date, and the current HEAD commit:

```
{
  "documents": [
    { "file": "legal/privacy-policy.md", "generated_at": "YYYY-MM-DD", "generated_at_commit": "abc1234" }
  ]
}
```

Keep entries for documents that still exist; add or refresh entries for the ones written this run (the refreshed `generated_at_commit` resets their staleness baseline).

**Format rules**

For Markdown (`.md`) in an SSG content directory:
- Use standard markdown headings (`#`, `##`)
- No YAML frontmatter
- No HTML wrapper — the site's template handles rendering

For MDX (`.mdx`) in a JS framework's pages directory:
- Use standard markdown headings (`#`, `##`)
- No HTML wrapper — the framework's layout handles rendering

For HTML (`.html`):
- Write each document to its own subfolder as `index.html` for clean URLs:
  `{output-path}/privacy-policy/index.html`, `{output-path}/terms/index.html`, etc.
- Write a complete standalone page: `<!DOCTYPE html>`, `<html>`, `<head>`, `<body>`
- Use semantic HTML elements: `<h1>`, `<h2>`, `<p>`, `<ul>`, `<li>`
- No inline styles, no `<style>` blocks, no JavaScript — the site's stylesheet handles rendering
- Include `<meta charset="UTF-8">` and `<meta name="viewport" content="width=device-width, initial-scale=1.0">`
- Set `<title>` to the document title (e.g. "Privacy Policy")

**Last updated (all documents):** Begin every document with a `**Last updated:** {current date}` line directly below the title, before the first section. Use the current date in `YYYY-MM-DD` format (e.g. `2026-07-07`).

**Contact section (all documents):** End every document with a `## Contact` section, using the contact point resolved in Step 1 and wording that matches its type:

  Email → "For questions about this {Document Title}, contact us at {email}."
  Contact page → "For questions about this {Document Title}, visit our contact page at {url}."
  Repo issue URL → "For questions about this {Document Title}, open an issue at {issues URL}."
  Other URL → "For questions about this {Document Title}, reach us at {url}."
  Postal address → "For questions about this {Document Title}, write to us at {address}."

  Never emit a placeholder here — Step 1 has already guaranteed a valid contact.

---

### privacy-policy.md

Title: `# Privacy Policy`

Required sections in order:

1. `## Who We Are` — one paragraph identifying the project, its purpose, and its owner
2. `## What Data We Collect` — list every category of data collected; if none, state that explicitly
3. `## How We Use Your Data` — for each purpose, state the GDPR legal basis (Art. 6); mandatory
4. `## Data Sharing and Third Parties` — name every third party that receives data; if none, state that
5. `## International Data Transfers` — state whether data leaves the EEA and under what safeguards; omit only if no data is collected at all
6. `## Data Retention` — how long each category of data is kept and why
7. `## Your Rights` — cover all GDPR Art. 15–21 rights: access, rectification, erasure, restriction, portability, objection; mandatory
8. `## Cookies` — include if cookies are used; omit if not applicable
9. `## Children's Privacy` — state whether the service is directed at children and any age restrictions
10. `## Supervisory Authority` — inform users of their right to lodge a complaint with a data protection authority (GDPR Art. 77); mandatory
11. `## Changes to This Policy` — how and when users are notified of updates
12. `## Contact` — standard contact wording

---

### terms.md

Title: `# Terms of Service`

Required sections in order:

1. `## Acceptance of Terms` — by using the service, users accept these terms
2. `## Description of Service` — what the service is and what it does
3. `## User Accounts and Eligibility` — age requirements, account responsibilities; omit if no accounts
4. `## Acceptable Use` — what users may do
5. `## Prohibited Conduct` — what users may not do; be specific
6. `## Intellectual Property` — who owns the software; who owns user-generated content if any
7. `## Disclaimer of Warranties` — service provided as-is; no guarantees
8. `## Limitation of Liability` — cap on damages; exclude consequential damages
9. `## Indemnification` — user agrees to hold the owner harmless for their misuse
10. `## Termination` — conditions under which access can be revoked
11. `## Governing Law and Dispute Resolution` — jurisdiction and how disputes are handled
12. `## Changes to These Terms` — how users are notified of updates
13. `## Contact` — standard contact wording

---

### cookie-policy.md

Title: `# Cookie Policy`

Required sections in order:

1. `## What Are Cookies` — brief plain-English explanation
2. `## How We Use Cookies` — overview of cookie usage on this site/app
3. `## Categories of Cookies` — list each category with its legal basis under GDPR/ePrivacy:
   - Necessary (no consent required)
   - Functional (consent required)
   - Analytics (consent required)
   - Marketing (consent required)
   Only include categories that actually apply; omit the rest.
4. `## Third-Party Cookies` — name each third party that sets cookies; link to their policies
5. `## Your Consent and How to Withdraw It` — how consent was obtained and how to withdraw it; withdrawal must be as easy as giving consent; mandatory under ePrivacy Directive
6. `## Managing Cookies in Your Browser` — link or instructions for browser cookie controls
7. `## Contact` — standard contact wording

---

### refund-policy.md

Title: `# Refund Policy`

Required sections in order:

1. `## Overview` — what this policy covers and which products or plans it applies to
2. `## Eligibility for Refunds` — conditions that must be met to qualify
3. `## Refund Window` — how many days after purchase a refund can be requested
4. `## How to Request a Refund` — step-by-step process
5. `## How Refunds Are Issued` — method (original payment method, credit, etc.) and timeline
6. `## Non-Refundable Items` — what is explicitly excluded
7. `## Contact` — standard contact wording

---

### eula.md

Title: `# End User License Agreement`

Required sections in order:

1. `## Grant of License` — what rights the user is granted (scope, number of devices, personal vs commercial)
2. `## License Restrictions` — what the user may not do (reverse engineer, redistribute, sublicense, etc.)
3. `## Intellectual Property Ownership` — all IP belongs to the owner; no transfer of ownership
4. `## Updates and Modifications` — owner may update the software; user's continued use implies acceptance
5. `## No Warranty` — software provided as-is; all implied warranties disclaimed
6. `## Limitation of Liability` — cap on damages; exclude consequential and indirect damages
7. `## Termination` — conditions for termination; what happens on termination (cease use, delete copies)
8. `## Governing Law` — jurisdiction
9. `## Contact` — standard contact wording

---

### disclaimer.md

Title: `# Disclaimer`

Required sections in order:

1. `## No Professional Advice` — nothing in the output constitutes legal, financial, or medical advice; name all three categories explicitly
2. `## AI-Generated Content` — output is produced by an AI model; AI can make mistakes, omit details, or misread context; always review before use
3. `## Accuracy and Completeness` — information may be incomplete, outdated, or incorrect; no guarantee of accuracy
4. `## External Links` — if the project links to third-party sites, disclaim responsibility for their content; omit if not applicable
5. `## Limitation of Liability` — owner is not liable for decisions made based on the output
6. `## Changes to This Disclaimer` — owner may update this disclaimer; continued use implies acceptance
7. `## Contact` — standard contact wording

---

## Step 4 — Commit and finalize

If Step 2 selected nothing (no documents to generate): skip the commit confirmation and environment promotion below — there is nothing to act on. Under GitHub Flow, still delete `{branch}` and return to `{original_branch}` (per Before starting's cleanup rule) — do not skip that part. Under Solo Mode there is nothing further to do, since no branch was created.

Otherwise, always confirm before committing, even when `git-auto-commit: true` is set — these are public, legally-binding documents, so this is a deliberate exception to the normal auto-commit flow (same category as the boundaries in safety.md's Agent Execution Boundaries: never skip review just because autonomy is high).

**Environment promotion** — shared by both modes below. If environment branches exist (discover via `git branch -r`, filter for known environment names, same detection as ship.md), ask which should also receive these documents:

  AskUserQuestion:
    question: "main will be updated. Which environment branches should also receive these documents?"
    header:   "Promote to environments"
    multiSelect: true
    options: one entry per discovered environment branch, e.g.:
      - label: "staging"
        description: "Merge main into staging"
      - label: "production"
        description: "Merge main into production"

  For each selected environment:
  - git checkout {environment}
  - git merge main --no-ff -m "chore(deploy): promote main to {environment} for legal document updates"
  - git push origin {environment}
  - git checkout main

  If no environment branches exist, or the user selects none, skip silently.

**Solo Mode:**

Present the list of documents written and ask for confirmation:

  AskUserQuestion:
    question: "The following documents were written to {resolved-path}: {list}. Commit them now?"
    header:   "Commit"
    multiSelect: false
    options:
      - label: "Commit (Recommended)"
        description: "docs(legal): generate legal documents"
      - label: "Cancel"
        description: "Leave documents written but uncommitted — commit manually when ready"

If Cancel selected → leave the documents written but uncommitted, then proceed to Step 5 (records that nothing was committed; skip environment promotion).

If Commit selected:
- Stage the generated documents and the updated `.claude/legal-manifest.json`; do not use `git add -A`.
- Commit: `docs(legal): generate legal documents` (include the resolved output path and format in the body if they differ from the default).
- Push: `git push origin main`.
- Run Environment promotion above.

**GitHub Flow:**

Present the list of documents written and ask for confirmation, stating plainly what confirming will trigger — this covers the whole sequence, not just the commit:

  AskUserQuestion:
    question: "The following documents were written to {resolved-path}: {list}. Confirming will commit on {branch}, merge it into main, promote to any environment branches you select next, delete {branch}, and return you to {original_branch}."
    header:   "Commit"
    multiSelect: false
    options:
      - label: "Commit and finish (Recommended)"
        description: "docs(legal): generate legal documents"
      - label: "Cancel"
        description: "Leave documents written but uncommitted on {branch} — finish manually when ready"

If Cancel selected → leave the documents written but uncommitted on `{branch}`. Do not merge, delete, or switch branches. Proceed to Step 5 (records that nothing was committed and that `{branch}` was left in place).

If Commit and finish selected:
1. Stage the generated documents and the updated `.claude/legal-manifest.json` on `{branch}`; do not use `git add -A`. Commit: `docs(legal): generate legal documents` (include the resolved output path and format in the body if they differ from the default).
2. Merge into main: `git checkout main`, `git merge {branch} --no-ff -m "docs(legal): generate legal documents"`, `git push origin main`.
3. Run Environment promotion above.
4. Delete `{branch}`: `git branch -d {branch}` locally, and `git push origin --delete {branch}` if it was ever pushed.
5. Return to where you started: `git checkout {original_branch}`.

---

## Step 5 — Confirm completion

Report:

LEGAL COMPLETE
Generated:
- {list of documents generated}

Location:     {resolved output path}
Format:       {Markdown / MDX / HTML}
Jurisdiction: GDPR
Tone:         plain English
Committed:    yes / no
Environments promoted: {list or none}

GitHub Flow only:
Branch:       {branch} — merged to main and deleted / left in place, uncommitted (if cancelled)
Returned to:  {original_branch}

Note: These documents are AI-generated starting points.
Review before publishing. Consult a lawyer for high-stakes products.

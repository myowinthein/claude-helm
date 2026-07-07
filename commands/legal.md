---
description: Scan project and generate GDPR-compliant legal documents based on what it does
---

# legal

Scan the project and generate legal documents based on what the
project actually does. Only generate documents that apply.

---

## Step 1 — Branch check

Only proceed if on main or master.
If on any other branch, stop and inform user:

"legal must be run on main or master.
Current branch is {branch}. Please switch and re-run."

---

## Step 2 — Project scan

Scan the codebase to understand the project's legal profile:

**App type**
- Chrome extension, web app, mobile app, desktop app, open source library
- Infer from manifest.json, package.json, composer.json, folder structure

**Data collection**
- Forms, auth systems, user accounts
- Analytics tools (Google Analytics, Mixpanel, Hotjar, GTM)
- Error tracking (Sentry, Bugsnag)
- Any personally identifiable information (PII) collected or stored

**Third party integrations**
- Payment processors (Stripe, PayPal, Paddle)
- Social auth (Google, GitHub, Facebook OAuth)
- Cloud services (AWS, GCP, Firebase)
- Marketing or tracking tools

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
  - Output path: `legal/` at the repo root
  - Rationale: SSGs are typically used for documentation or blogs. Legal docs are
    secondary and do not need to be rendered as site pages. Root `legal/` keeps them
    out of the site's content directory; GitHub renders the markdown natively and
    they can be linked to directly.

  JS framework with SSR or SSG (Next.js, Astro, Nuxt, SvelteKit, and similar):
  - Format: `.mdx`
  - Output path: the framework's pages or content directory, under a `legal/` subfolder
  - Detect the correct pages directory from the framework's config and folder structure

  SPA or plain HTML site (no server-side rendering, no static site generator):
  - Format: `.html` with subfolder structure (`legal/privacy-policy/index.html`, etc.)
  - Output path: the directory that is served as the web root (e.g. `public/`, `dist/`, or root)

  No web project detected (pure library, CLI tool, or non-web repo):
  AskUserQuestion:
    question: "No web framework detected. Where should legal documents be written?"
    header:   "Output"
    multiSelect: false
    options:
      - label: "legal/ as Markdown (Recommended)"
        description: "Standard location for repos without a web front end"
      - label: "legal/ as HTML"
        description: "Bare semantic HTML — suitable for plain HTML sites"
      - label: "Custom path"
        description: "Specify a path manually"

**CSS style detection (HTML output only)**

When the resolved format is `.html`, scan the project to determine whether styles
are applied globally or scoped:

  Global CSS (element and class selectors that apply site-wide):
  → Output bare semantic HTML. The site's stylesheet handles rendering.

  Scoped or utility-first CSS (styles tied to components or utility classes,
  where unstyled semantic HTML receives no visual treatment):
  → Output bare semantic HTML AND prepend this comment inside `<body>`:

    <!-- TODO: wrap this content in your site's prose or layout container.
         Styles are scoped or utility-first — bare semantic HTML will not
         inherit your site's visual design without a wrapper. -->

  Cannot determine:
  → Add the comment as a safe default.

Record the resolved format and output path before proceeding.

**Confirm output location**

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

**Existing legal documents**
- Check whether the resolved output path exists
- List which documents are already present using the resolved file extension
- Note existing files — the user will choose whether to regenerate them

Record all findings before proceeding.

---

## Step 3 — Determine which documents to generate

Based on scan findings, mark each document as Recommended if it applies:

- privacy-policy.md — always Recommended
- terms.md — always Recommended
- cookie-policy.md — Recommended if non-essential cookies or analytics detected
- refund-policy.md — Recommended if payment processing detected
- eula.md — Recommended if Chrome extension, desktop app, or downloadable software detected
- disclaimer.md — Recommended if financial, health, legal advice or AI recommendations detected

AskUserQuestion supports a maximum of 4 options. Split into two questions:

Question 1 — core documents:
  AskUserQuestion:
    question: "Which core legal documents should be generated? (Jurisdiction: GDPR · Tone: plain English)"
    header:   "Core documents"
    multiSelect: true
    options:
      - label: "privacy-policy.md (Recommended)"    ← append "(exists)" if file already present
        description: "Explains what data is collected and how it is handled"
      - label: "terms.md (Recommended)"             ← append "(exists)" if file already present
        description: "Acceptable use, IP ownership, liability, governing law"
      - label: "eula.md"                            ← append "(Recommended)" if downloadable software; "(exists)" if file present
        description: "License grant, restrictions, and liability for installable software"
      - label: "disclaimer.md"                      ← append "(Recommended)" if AI content or advice; "(exists)" if file present
        description: "No-professional-advice notice and AI-generated content warning"

Question 2 — conditional documents (only ask if applicable based on scan findings):
  AskUserQuestion:
    question: "Any additional legal documents needed?"
    header:   "Additional documents"
    multiSelect: true
    options:
      - label: "cookie-policy.md"                   ← append "(Recommended)" if analytics detected; "(exists)" if file present
        description: "Required if non-essential cookies or analytics tools are present"
      - label: "refund-policy.md"                   ← append "(Recommended)" if payments detected; "(exists)" if file present
        description: "Required if payment processing is present"

  Skip Question 2 entirely if neither cookie-policy nor refund-policy is applicable
  based on scan findings (no analytics, no payments detected).

Generate only the documents selected across both questions.
If nothing is selected, exit without generating anything.
Selected documents that already exist will be overwritten.

Wait for both responses before proceeding.

---

## Step 4 — Generate documents

Generate each selected document in plain English, GDPR compliant.
Write to the resolved output path using the resolved format. Overwrite if the file already exists.

**Format rules**

For Markdown (`.md`) at repo root (SSG projects):
- Use standard markdown headings (`#`, `##`)
- No YAML frontmatter — these files are not site pages
- No HTML wrapper

For MDX (`.mdx`) in a JS framework's pages directory:
- Use standard markdown headings (`#`, `##`)
- No HTML wrapper
- The framework's layout handles rendering

For HTML (`.html`):
- Write each document to its own subfolder as `index.html` for clean URLs:
  `{output-path}/privacy-policy/index.html`, `{output-path}/terms/index.html`, etc.
- Write a complete standalone page: `<!DOCTYPE html>`, `<html>`, `<head>`, `<body>`
- Use semantic HTML elements: `<h1>`, `<h2>`, `<p>`, `<ul>`, `<li>`
- No inline styles, no `<style>` blocks, no JavaScript — the site's stylesheet handles rendering
- Include `<meta charset="UTF-8">` and `<meta name="viewport" content="width=device-width, initial-scale=1.0">`
- Set `<title>` to the document title (e.g. "Privacy Policy")

**Writing style**
- Use em-dashes sparingly. Only use one when no other punctuation
  (comma, semicolon, colon, or a new sentence) works as well.
  When in doubt, restructure the sentence instead.

**Date format:** Use `YYYY-MM-DD` (e.g. `2026-07-07`) for all "Last updated" dates.

**Contact section (all documents):** End every document with a `## Contact` section
using this exact wording:

  For questions about this {Document Title}, open an issue at {repo issues URL}.

  Infer the repo issues URL from `package.json`, `composer.json`, or git remote.
  If not found, use a placeholder and note it.

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
13. `**Last updated:** YYYY-MM-DD` — at the very top, below the title

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
14. `**Last updated:** YYYY-MM-DD` — at the very top, below the title

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
8. `**Last updated:** YYYY-MM-DD` — at the very top, below the title

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
8. `**Last updated:** YYYY-MM-DD` — at the very top, below the title

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
10. `**Last updated:** YYYY-MM-DD` — at the very top, below the title

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
8. `**Last updated:** YYYY-MM-DD` — at the very top, below the title

---

## Step 5 — Commit

Commit all generated documents:
  docs(legal): generate legal documents for v{version}

Include the resolved output path and format in the commit body if they differ from
the default (`legal/` Markdown), so future runs have context.

---

## Step 6 — Confirm completion

Report:

LEGAL COMPLETE
Generated:
- {list of documents generated}

Location:     {resolved output path}
Format:       {Markdown / MDX / HTML}
Jurisdiction: GDPR
Tone:         plain English
Committed:    yes

Note: These documents are AI-generated starting points.
Review before publishing. Consult a lawyer for high-stakes products.

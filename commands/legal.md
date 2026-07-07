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

Record findings before proceeding to document generation.

---

## Step 3 — Determine which documents to generate

Based on scan findings:

Always generate:
- privacy-policy.md
- terms.md

Generate if non-essential cookies or analytics detected:
- cookie-policy.md

Generate if payment processing detected:
- refund-policy.md

Generate if Chrome extension, desktop app, or downloadable software:
- eula.md

Generate if financial, health, legal advice or AI recommendations detected:
- disclaimer.md

Use AskUserQuestion (multi-select) listing every possible document. Mark each one
that matches the scan findings with "(Recommended)". Always mark privacy-policy.md
and terms.md as Recommended:

  AskUserQuestion:
    question: "Which legal documents should be generated? (Jurisdiction: GDPR · Tone: plain English)"
    header:   "Documents"
    multiSelect: true
    options:
      - label: "privacy-policy.md (Recommended)"
        description: "Always required — explains what data is collected and how it is handled"
      - label: "terms.md (Recommended)"
        description: "Always required — acceptable use, IP ownership, liability, governing law"
      - label: "cookie-policy.md (Recommended if analytics detected)"
        description: "Required if non-essential cookies or analytics tools are present"
      - label: "refund-policy.md (Recommended if payments detected)"
        description: "Required if payment processing is present"
      - label: "eula.md (Recommended if downloadable software)"
        description: "Required for plugins, desktop apps, Chrome extensions, or any installable software"
      - label: "disclaimer.md (Recommended if AI content or advice detected)"
        description: "Required if the project generates AI content, legal docs, financial or health advice"

  Only include "(Recommended)" on labels that match scan findings.
  The user may deselect any document or add ones the scan did not flag.
  If nothing is selected, exit without generating anything.

Wait for response before proceeding.

---

## Step 4 — Generate documents

Generate each document in plain English, GDPR compliant.
Write to legal/ folder.

**Writing style**
- Use em-dashes sparingly. Only use one when no other punctuation
  (comma, semicolon, colon, or a new sentence) works as well.
  When in doubt, restructure the sentence instead.

**Date format:** Use `YYYY-MM-DD` (e.g. `2026-07-07`) for all "Last updated" dates.

**Contact section (all documents):** End every document with a `## Contact` section using this exact wording:

  For questions about this {Document Title}, open an issue at {repo issues URL}.

  Where {Document Title} is the document's heading title (e.g. "Privacy Policy", "Terms of Use",
  "End User License Agreement", "Disclaimer"). Infer the repo issues URL from the project's
  `package.json`, `composer.json`, or git remote. If not found, use a placeholder and note it.

**privacy-policy.md must cover:**
- What data is collected and why
- How data is stored and protected
- Whether data is shared with third parties (name them)
- User rights under GDPR (access, rectification, erasure, portability)
- Data retention periods
- Cookie usage (if applicable)
- Last updated date (YYYY-MM-DD)
- Contact section

**terms.md must cover:**
- What the service is and what it does
- Acceptable use — what users can and cannot do
- IP ownership — who owns the content and the software
- Liability limitations
- Termination conditions
- Governing law
- Last updated date (YYYY-MM-DD)
- Contact section

**cookie-policy.md must cover:**
- What cookies are used and why
- Which are essential vs non-essential
- How users can control cookies
- Third party cookies (name them)
- Last updated date (YYYY-MM-DD)
- Contact section

**refund-policy.md must cover:**
- Refund eligibility conditions
- Refund request process and timeframe
- Non-refundable items or conditions
- Last updated date (YYYY-MM-DD)
- Contact section

**eula.md must cover:**
- License grant — what users are permitted to do
- Restrictions — what users cannot do
- IP ownership
- Disclaimer of warranties
- Limitation of liability
- Termination conditions
- Last updated date (YYYY-MM-DD)
- Contact section

**disclaimer.md must cover:**
- Nature of the information provided
- No professional advice claim
- Accuracy limitations
- User responsibility for decisions made
- AI-generated content disclaimer if applicable
- Last updated date (YYYY-MM-DD)
- Contact section

---

## Step 5 — Commit

Commit all generated documents:
  docs(legal): generate legal documents for v{version}

---

## Step 6 — Confirm completion

Report:

LEGAL COMPLETE
Generated:
- {list of documents generated}

Location:     legal/
Jurisdiction: GDPR
Tone:         plain English
Committed:    yes

Note: These documents are AI-generated starting points.
Review before publishing. Consult a lawyer for high-stakes products.
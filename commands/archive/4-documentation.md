# Step 4 — Documentation

Create project documentation for long-term archive recovery and future demonstration.

Use all relevant outputs from previous steps as context.

Goal: Make future me able to understand, restore, run, troubleshoot, and demonstrate this project years later.

---

## Rules

- Read all existing documentation before making any changes.
- Preserve useful historical information. Clearly distinguish it from verified current information.
- Only document verified information. Label anything unverified explicitly.
- Do not invent setup procedures, demo workflows, runtime requirements, or API behavior.
- Do not include secrets, credentials, API keys, or private/production URLs.
- Keep README.md concise — link to docs/ rather than repeating content.
- Do not commit or push.

---

## Consolidation

For every existing documentation file, decide: Keep, Update, Merge, Archive, or Remove. Provide a reason for each.

Use this to guide the decision:

| Existing doc type | Action |
|---|---|
| Accurate, has recovery or demonstration value | Keep — link from README |
| Accurate but duplicates docs/setup.md content | Merge into docs/setup.md, remove original |
| Outdated instructions that affect restoration | Update |
| Historical context, domain knowledge, business background | Move to docs/historical-notes.md |
| Contributor guides, pull request templates, code of conduct | Remove — irrelevant for private archive |
| Changelogs and release notes | Move to docs/historical-notes.md if they provide context; otherwise remove |
| CI/CD documentation, deployment runbooks | Remove — no recovery value |
| Auto-generated documentation | Remove — regenerate if needed |
| Outdated and no recovery or historical value | Remove |

Consolidate duplicates into a single authoritative source. Do not leave orphaned files — every active document must be reachable through README.md or another active document.

---

## Required Files

### README.md
Always write a fresh archive-focused README regardless of whether one already exists.

Before writing: read the existing README if present. Extract any historically valuable content — business context, domain knowledge, why the project was built — and move it to `docs/historical-notes.md`. Do not carry outdated instructions, contributor guides, or public-facing content into the new README.

Write a clean README containing: project name, one-paragraph purpose, tech stack summary, restoration complexity rating from Step 1, and links to `docs/setup.md` and `docs/archive-metadata.md`. Keep it short — this is an index for future-you, not a public-facing document.

### docs/setup.md

The single reference for everything needed to restore, run, and demonstrate this project. Write the following sections, skipping any that are not applicable to this project type:

**Environment**
Original runtime requirements. Verified restoration environment from Step 2 — note differences from original if any. Environment variables required.

**Restore & Run**
Step-by-step procedure to restore the project locally. Use exact commands from Step 2:
```
docker load < recovery/docker/{service}.tar.gz
docker-compose up
```
Access points — URLs and ports for web projects, run commands for CLI tools, build steps for mobile or desktop.

**Recovery Priority**
Classify each dependency:
- Required (Docker tarballs, database dump, environment variables)
- Optional (email service, third-party integrations)
- Not required (production services, analytics)

**Demo** *(skip for CLI tools, libraries, and projects with no interactive surface)*
Demo accounts from Step 2. Happy path walkthrough. Key screens or features. Link to Postman collection at `recovery/postman/` if Step 3 generated one.

**API** *(include only if Step 3 generated a Postman collection)*
Authentication method. Collection location: `recovery/postman/{project-name}.postman_collection.json`. How to import and use.

**Restoration Fixes**
Every fix applied in Step 2: what was changed, why it was necessary, and whether it may need revisiting on future recovery.

**Known Limitations**
Anything that could not be restored, features that require unavailable services, or constraints the developer should know before demonstrating.

### docs/archive-metadata.md
Structured snapshot for quick inventory across archived projects:
- Repository name and archive date
- Project purpose (one line)
- Technology stack
- Restoration outcome from Step 2
- Runtime versions used
- Database source used
- Docker images exported and tarball paths
- Postman collection generated: yes / no
- Recovery complexity rating from Step 1
- Known limitations (brief)

---

## Optional File

**docs/historical-notes.md** — create only if significant historical documentation was found that has archival value but would clutter current docs. Clearly label all content as historical.

---

## Tasks

1. Read all existing documentation
2. Decide Keep / Update / Merge / Archive / Remove for each existing file
3. Present the consolidation plan and wait for approval before making any changes:

   ```
   CONSOLIDATION PLAN

   {filename}    → {action} ({reason})
   {filename}    → {action} ({reason})
   ...
   ```

   Do not proceed until the developer explicitly approves.

4. Extract historically valuable content from existing README into docs/historical-notes.md
5. Write a fresh README.md
6. Write or update docs/setup.md with all applicable sections
7. Write docs/archive-metadata.md
8. Verify every active document is reachable from README.md

---

Output the report in this format:

## Consolidation Summary
For every existing documentation file reviewed: action taken (Keep / Update / Merge / Archive / Remove) and reason.

## Files Created
New files written and what each covers.

## Files Updated
Existing files updated and what changed.

## Files Removed
Files removed and why.

## Navigation
Confirm every active file is reachable from README.md.

## Gaps
Anything that could not be documented due to missing or unverified information from previous steps.

---

Do not commit or push. Output the report and stop.

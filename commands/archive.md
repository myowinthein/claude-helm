---
description: Explore, restore, containerize, document, and seal a project for long-term archive recovery
---

# archive

Run `ls -la` to confirm the current directory, then immediately begin Step 1 below without waiting for further instruction.

---

# Step 1 — Explore

Analyze this project in read-only mode.

Goal: Understand the project well enough to make an informed decision about archiving it, and to give every later step the context it needs.

Rules:

- Read-only only — do not modify, create, delete, or run anything.
- Do not contact external services.
- Do not print full credential values. Redact — show only file path, variable name, and a short redacted preview (e.g. `API_KEY=abc...xyz`).

Determine:

- Project purpose and business domain
- Technology stack — framework, language, runtime versions, package manager, database type
- Architecture — key modules, features, and how the pieces fit together
- Authentication and authorization approach
- External integrations and third-party service dependencies
- Environment variables required
- Database situation — type, credentials location, and available restoration sources (dumps, seeders, migrations)
- Demo accounts or demo data availability
- Existing documentation files — what exists, what each covers, whether it appears current or outdated
- Existing Postman or API collection files
- Archive-worthy assets — screenshots, videos, sample exports, demo files
- Large or binary files that could affect Git storage
- CI/CD configuration — any pipelines that could auto-deploy on push
- Git remotes, branches, and repository safety concerns
- Active credentials — API keys, tokens, secrets, and passwords across all env files (.env, .env.staging, .env.production, .env.*.local, config files, Docker files). Flag anything that appears still-active. Redact values.
- Restoration risks and unknowns
- Recommended restoration approach — local or Docker

Guidance:

- Do not rely only on README files — read source code, config, and manifests.
- Separate confirmed facts from guesses. Label guesses explicitly.
- Mark anything that cannot be safely determined in read-only mode as unknown.

Output the report in this format:

## Overview
Project name, purpose, and business domain. One short paragraph.

## Technology Stack
Framework, language, runtime versions, package manager, database type.

## Architecture
Key modules, layers, and how the project is structured. Keep it brief.

## Data & Database
Database type and where credentials live. Available restoration sources in priority order (dumps → seeders → migrations → none). Demo accounts or demo data if found.

## External Integrations
Third-party services the project depends on. Note which are required for basic operation vs optional.

## Existing Documentation
List documentation files found. For each: what it covers and whether it appears current or outdated.

## Git & Repository Safety
Remotes and their URLs. Branch list. Any safety concerns (production remotes, CI/CD that auto-deploys on push, sensitive history).

## Setup & Restoration Approach
Runtime versions required. Package manager. Whether Docker is needed. Recommended restoration strategy and why.

## Archive Assets
Postman files, screenshots, videos, sample exports, or other assets worth preserving. Large or binary files worth noting.

## Credentials Risk
All env files scanned (.env, .env.staging, .env.production, config files, Docker files). For each credential found: file path, variable name, redacted value, and whether it appears still-active or likely expired. If none found, state that explicitly.

## Risks & Unknowns
Blockers, missing dependencies, unclear areas. Be specific.

## Restoration Complexity
Rate as: Easy / Medium / Complex / Blocked — one sentence explaining why.

## Recommended Next Step
What to watch out for or resolve before proceeding to Step 2.

Do not make any changes. Output the report and stop. Wait for approval before continuing to Step 2.

---

# Step 2 — Restore and Freeze

Restore this project and freeze its environment for long-term archive recovery.

Use all relevant outputs from previous steps as context.

Goal: Get the project running locally, containerize it with pinned image versions, and export the built images to tarballs so the environment can be recovered years later without depending on external registries.

## Rules

- Preserve the original project as much as possible. Prefer compatibility fixes over upgrades, restoration over modernization.
- Record every file created, modified, or deleted with the reason.
- Use placeholder values for external services.
- Do not connect to production systems, live databases, or external services.
- Do not send emails, SMS, notifications, webhooks, or invoke payment gateways without explicit approval.
- Do not change Git state — no commit, push, pull, fetch, checkout, reset, rebase, merge, or history rewrite.
- Work only within the current repository. If another repository appears required, report it and stop for approval.

## Restoration Philosophy

The primary goal is not just to run the project today — it is to freeze the environment so it can be recovered 5–10 years from now, regardless of what runtimes, package versions, or registries are available at that time.

Docker is the preferred environment freeze mechanism. Base image tags and registries can disappear over time, so after a successful build, export all images to gzipped tarballs stored in `recovery/docker/`. Recovery then requires only Docker and the tarballs — no internet access, no registry dependency.

For project types where Docker does not apply, document exact runtime versions, SDK versions, and toolchain versions in `docs/setup.md` as the freeze mechanism instead.

## Restoration Strategy

Use Step 1's findings to select the approach. Docker is the primary choice for most project types.

**Containerize with Docker (primary for most projects):**

If a Docker or Docker Compose setup already exists: use it, but verify all image tags are pinned to specific versions (e.g. `node:18.17.1-alpine`, not `node:18` or `node:latest`). Update any unpinned tags before building.

If no Docker setup exists: create a `Dockerfile` and `docker-compose.yml` appropriate for the project's stack and runtime. Pin all base image versions specifically.

By project type:

- **Web backend / API** (any language or framework): Docker Compose with app + database + any required services
- **CMS** (WordPress, Drupal, etc.): Docker Compose with web server + database
- **Web frontend** (React, Vue, Angular, Svelte, etc.): Docker for the build environment and a simple static server for the output
- **Static site** (Jekyll, Hugo, Gatsby, Astro, etc.): Docker for the build environment; output is static files
- **Full-stack / monorepo**: Docker Compose for all required services; restore minimum components for a working demonstration
- **Data science / ML** (Jupyter, Python scripts, R, etc.): Docker with pinned base image and all dependencies installed; verify notebooks or scripts execute inside the container
- **CLI tool or script** (with non-trivial runtime): Docker with pinned runtime image

**Document versions instead of containerizing (where Docker does not apply):**

- **Mobile** (React Native, Flutter, iOS, Android): Docker cannot meaningfully freeze a mobile build environment. Document exact SDK version, toolchain version, target platform version, and any required IDE version in `docs/setup.md`.
- **Desktop** (Electron, Tauri, etc.): prefer Docker for the build step if possible; otherwise document exact Node or Rust version.
- **Browser extension**: document exact browser version and extension API versions used.
- **Library / package**: Docker for the test environment; document the published package version.

## Data Restoration

Discover all available data sources in the repository. Assess each for completeness and suitability.

Use the best available source in this order:

1. Repository-contained database dumps or backup files (SQL, MongoDB, SQLite, Redis snapshots, etc.)
2. Seed scripts, fixture files, or sample datasets
3. Schema migrations — reconstruct from scratch
4. File-based data sources (CSV, JSON, XML data files used by the application)
5. Documented external exports
6. Existing localhost database — only if no repository-contained source exists

If no data source exists and the project requires one: report the gap and explain the impact on the demonstration.

If restoration requires a network, VPN-accessible, or cloud database: report why and stop for approval.

## Image Export

After the project is verified as running:

Export all Docker images to `recovery/docker/` as gzipped tarballs:

```
docker save {image_name} | gzip > recovery/docker/{service}.tar.gz
```

Export one tarball per service (web, database, cache, etc.).

Add `recovery/docker/` to `.gitignore` — tarballs are stored locally alongside the repo, not committed.

Document the recovery commands in `docs/setup.md`:

```
docker load < recovery/docker/web.tar.gz
docker load < recovery/docker/db.tar.gz
docker-compose up
```

## Tasks

1. Select restoration strategy using Step 1 context
2. Configure the local environment
3. Create or update Docker setup with pinned image versions
4. Install dependencies
5. Restore available data
6. Build and start the containers
7. Verify the project works
8. Export all images to `recovery/docker/` as gzipped tarballs
9. Add `recovery/docker/` to `.gitignore`
10. Record all changes made

Output the report in this format:

## Restoration Strategy
Chosen approach and why. Whether Docker was created or already existed. What was tried first if the first choice failed.

## Docker Setup
Base images used with exact pinned versions. Services defined. Whether Dockerfile and docker-compose.yml were created or already existed.

## Image Exports
Images exported to `recovery/docker/`. Size of each tarball. Recovery commands.

## Data & Database
Data source selected and why. How data was restored. Demo accounts or sample data available if any.

## Changes Made
Every file created or modified: path, what changed, and why it was necessary.

## Environment Variables
Any variables added or changed from the original.

## Commands Used
Exact commands run in order.

## Access Points
How to access or run the project locally (URLs and ports for web projects, commands for CLI tools, entry points for scripts or notebooks). State "not applicable" if the project type has no access point.

## Verification Results
What was tested and what the results were. Any critical errors found.

## Restoration Outcome
One of: Fully Restored / Partially Restored / Buildable But Not Runnable / Blocked / Cannot Be Restored Without Missing Dependencies — one sentence explaining why.

## Remaining Blockers
Anything that prevented full restoration or needs attention before proceeding to Step 3.

Do not commit or push. Output the report and stop.

If Restoration Outcome is Fully Restored: wait for approval before continuing to Step 3.

If Restoration Outcome is anything other than Fully Restored: clearly state that proceeding to Step 3 is not recommended until blockers are resolved. List what needs to be fixed. Wait for explicit approval before continuing.

---

# Step 3 — Postman Collection

Generate a Postman collection for this project if it exposes an API.

Use all relevant outputs from previous steps as context.

Goal: Create reusable Postman files that future me can use to understand, test, troubleshoot, and demonstrate the project's API years later.

## Applicability

Generate a collection if the project exposes an API it owns — REST, GraphQL, or similar.

Skip if:
- The project has no API surface (pure frontend, static site, CLI tool, library, etc.)
- The only APIs present belong to third-party services (Stripe, SendGrid, etc.)

If skipping: explain why, output the report, and stop.

If an existing Postman collection is found (from Step 1): use it as the starting point — update, verify, and expand rather than generating from scratch.

## Rules

- Discover endpoints from source code, route definitions, or OpenAPI/Swagger specs. Do not speculate.
- Avoid deprecated, disabled, or commented-out endpoints.
- Use demo accounts, local tokens, or placeholder values only. No real credentials, secrets, or production URLs.
- If the local application is running (from Step 2): use it to verify requests.
- If the local application is not running: fall back to source-code discovery only and mark all requests as unverified.
- Do not commit or push.

## Verification

Verify representative requests where safe — GET requests, health endpoints, authentication flows, and local demo workflows.

Do not verify endpoints that may delete data, send emails or SMS, trigger webhooks, process payments, or perform bulk actions without explicit approval. Mark those as unverified with a note explaining why.

## Output Files

Use the project name from Step 1 for the filenames:

- `recovery/postman/{project-name}.postman_collection.json`
- `recovery/postman/{project-name}.postman_environment.json`

Organize requests by functional area (Authentication, Users, Admin, Products, Orders, etc.). Use collection variables, environment variables, request descriptions, and authentication notes. Use localhost URLs and placeholder values. Avoid duplicate endpoints.

## Tasks

1. Determine whether a Postman collection is applicable for this project
2. Check for existing Postman files from Step 1 — use as starting point if found
3. Discover all API endpoints from source code, routes, or OpenAPI specs
4. Determine authentication method and required environment variables
5. Generate or update the collection and environment files
6. Organize requests by functional area
7. Verify representative requests where safe
8. Mark unverified requests with a reason

Output the report in this format:

## Applicability
Whether a collection was generated or skipped and why.

## Existing Collection
Whether an existing Postman collection was found and how it was used.

## Endpoints Discovered
Total endpoints found. How they were discovered (routes, OpenAPI spec, source code). Any endpoints excluded and why.

## Authentication
Auth method used. How it is configured in the collection.

## Files Created or Updated
File paths and what changed if updating an existing collection.

## Verified Requests
Requests that were tested against the running app and their results.

## Unverified Requests
Requests that could not be safely verified. Reason for each.

## Known Limitations
Endpoints that could not be covered, external dependencies, or gaps in coverage.

Do not commit or push. Output the report and stop. Wait for approval before continuing to Step 4.

---

# Step 4 — Documentation

Create project documentation for long-term archive recovery and future demonstration.

Use all relevant outputs from previous steps as context.

Goal: Make future me able to understand, restore, run, troubleshoot, and demonstrate this project years later.

## Rules

- Read all existing documentation before making any changes.
- Preserve useful historical information. Clearly distinguish it from verified current information.
- Only document verified information. Label anything unverified explicitly.
- Do not invent setup procedures, demo workflows, runtime requirements, or API behavior.
- Do not include secrets, credentials, API keys, or private/production URLs.
- Keep README.md concise — link to docs/ rather than repeating content.
- Do not commit or push.

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

## Optional File

**docs/historical-notes.md** — create only if significant historical documentation was found that has archival value but would clutter current docs. Clearly label all content as historical.

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

Do not commit or push. Output the report and stop. Wait for approval before continuing to Step 5.

---

# Step 5 — Finalize

Prepare the repository for final archival, clean up remotes and branches, and push to the private archive remote.

Use all relevant outputs from previous steps as context.

Goal: Seal the archive — ensure the repository is clean, safe, and pushed to the correct private remote.

## Rules

- Present a plan and wait for explicit approval before any destructive operation.
- Do not delete branches without explicit approval.
- Do not push until origin is confirmed as the private archive remote.
- Do not commit until all cleanup tasks are complete.

## Task 1 — Remote Verification

List all configured remotes and their URLs. Present them to the developer:

```
REMOTES

origin      https://github.com/company/old-project.git
upstream    https://gitlab.com/client/project.git
```

Ask:
- Which of these point to company, client, or external repositories?
- What is the private GitLab URL for this archive? (The repository must already exist on GitLab before proceeding — remind the developer to create it if needed.)

Remove all non-personal remotes. Set origin to the provided private GitLab URL.

Do not proceed to any other task until origin is confirmed as the private archive remote.

## Task 2 — Branch Cleanup

List all local and remote branches. Identify main or master as the single branch to keep.

Present the deletion plan and wait for explicit approval:

```
BRANCH CLEANUP

Keep:     main
Delete:   feature/user-auth   (local + remote)
Delete:   fix/payment-bug     (local + remote)
Delete:   chore/cleanup       (local + remote)
```

Once approved:
- Delete all branches except main/master locally and remotely
- Verify local main/master is in sync with remote
- If local and remote have diverged: report the divergence and stop for approval before resolving

## Task 3 — Git LFS for Large Files

Scan for files larger than 100 MB — both tracked and untracked.

For each file found: report path, size, type, and whether it is currently tracked by Git.

If large files exist:
- Set up Git LFS if not already configured
- Track large files with appropriate LFS patterns in `.gitattributes`
- Stage `.gitattributes` changes

If no large files exist: skip silently.

## Task 4 — Asset Consolidation

Identify archive-worthy assets scattered across the project: screenshots, videos, audio files, sample exports, database dumps, demo files, and other binary or media assets that are not part of the application source code.

For each asset found:
1. Check whether any source file, template, stylesheet, or documentation references its current path
2. Classify as:
   - **Safe to move** — not referenced anywhere in the codebase or documentation
   - **Referenced** — one or more files reference the current path; moving would break them

For safe assets: move to `recovery/assets/` and update any documentation references.

For referenced assets: report the file, what references it, and why moving would break things. Do not move — present findings and wait for the developer to decide.

If no assets worth consolidating are found: skip silently.

## Task 5 — Stop Services

Identify all running services started by this project: Docker containers, Node dev servers, PHP built-in servers, Python servers, or any other processes tied to this project.

Present the list and wait for approval before stopping anything:

```
RUNNING SERVICES

docker-compose (project: vote-for-change) — containers: web, db, redis
node (port 3000) — npm run dev
```

Once approved, stop all listed services.

If no running services are found: skip silently.

## Task 6 — Final Commit and Push

Show git status — all staged and unstaged changes accumulated across the full workflow.

Commit all changes:
```
git add -A
git commit -m "chore(archive): seal project archive"
```

Push to origin using the branch identified in Task 2:
```
git push origin {main_or_master}
```

Confirm the push was successful.

Output the report in this format:

## Remote Changes
Previous remote(s) removed. New origin confirmed as private archive remote.

## Branch Cleanup
Branches deleted locally and remotely. Confirmation that local and remote main/master are in sync.

## Git LFS
Files tracked with LFS and `.gitattributes` changes. If none: state no large files were found.

## Asset Consolidation
Assets moved to `recovery/assets/`. Assets that could not be moved due to code references and why. If none: state no assets were consolidated.

## Services Stopped
Services identified and stopped. If none: state no running services were found.

## Final Commit
Commit hash and summary of what was committed.

## Push
Remote URL. Confirmation of success.

## Archive Complete

Archive date, GitLab URL, and restoration complexity from Step 1.

To restore this project:

```
git clone {gitlab_url}
```

If Docker was used:
```
docker load < recovery/docker/{service}.tar.gz
docker-compose up
```

If Docker was not used: refer to the version and toolchain documentation in `docs/setup.md`.

Full recovery instructions: `docs/setup.md`

Stop. Archive complete.

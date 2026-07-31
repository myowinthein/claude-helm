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
- Compare the current branch's last commit date and content against every other local and remote branch. If another branch has more recent activity, or otherwise looks more complete than the current branch, flag it — the developer may be archiving from the wrong branch. This is informational only; do not block on it.
- Active credentials — API keys, tokens, secrets, and passwords across all env files (.env, .env.staging, .env.production, .env.*.local, config files, Docker files). Flag anything that appears still-active. Redact values. Before completing the Credentials Risk section, confirm you have searched every file matching these patterns and list the files searched explicitly.
- Restoration risks and unknowns
- Recommended restoration approach — local or Docker

Guidance:

- Do not rely only on README files — read source code, config, and manifests.
- Separate confirmed facts from guesses. Label guesses explicitly.
- Mark anything that cannot be safely determined in read-only mode as unknown.

---

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
Archiving from: {current branch name} — Step 5's branch cleanup reads this field directly rather than inferring it from prose, since a resumable, multi-session workflow could otherwise lose track of which branch was original.
Remotes and their URLs. Branch list. Any safety concerns (production remotes, CI/CD that auto-deploys on push, sensitive history). Whether any other branch looks more recently active or complete than the current one — state "none" if the current branch is clearly the most current.

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

---

Output the report and stop. Wait for explicit approval before proceeding to Step 2.

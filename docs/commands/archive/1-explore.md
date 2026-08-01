---
title: "Step 1 — Explore"
parent: /helm:archive
grand_parent: Commands
nav_order: 1
---

# Step 1 — Explore

Read-only reconnaissance. Scans every corner of the project — stack, data sources, integrations, credentials, Git state — and produces a structured report with a restoration complexity rating. Nothing is modified. The developer decides whether to proceed based on the report.

## Flow

```mermaid
flowchart TD
  Start([Begin Step 1]) --> ReadCode[Read source code, config,\nmanifests, docs, env files]

  ReadCode --> Gather["Determine:\n— Stack and runtime versions\n— Architecture and key modules\n— Database type and data sources\n— External integrations\n— Existing documentation\n— Postman/API collection files\n— Archive-worthy assets\n— Large or binary files\n— CI/CD pipelines\n— Git remotes and branches\n— Active credentials (redacted)"]

  Gather --> Complexity{Restoration\ncomplexity?}
  Complexity -->|Easy| Report
  Complexity -->|Medium| Report
  Complexity -->|Complex| Report
  Complexity -->|Blocked| Report

  Report[Output structured report] --> Stop([Stop — wait for approval])
```

## What it scans

The step reads source code, configuration files, manifests, env files, and any existing documentation. It does not rely only on README files. It separates confirmed facts from guesses and labels guesses explicitly.

**Project profile**
- Records the project's name explicitly as "Project name:" in its own field, rather than leaving it to be extracted from the Overview prose — Step 3 (Postman collection filenames) and Step 4 (README content) both read this field directly
- Purpose and business domain
- Technology stack: framework, language, runtime versions, package manager, database type
- Architecture: key modules, layers, and how the pieces fit together
- Authentication and authorization approach

**Data and database**
- Database type and where credentials live
- Available restoration sources in priority order: dumps, seeders, migrations, or none
- Demo accounts or demo data if found

**External integrations**
- Third-party services the project depends on
- Which are required for basic operation vs optional

**Setup and restoration approach**
- Runtime versions required, package manager, whether Docker is needed
- Recommended restoration strategy (local or Docker) and why — this is a project-wide setup judgment, not specific to the database

**Credentials risk**
- Scans all env files: `.env`, `.env.staging`, `.env.production`, config files, Docker files
- Redacts values — shows only file path, variable name, and a short preview (`API_KEY=abc...xyz`)
- Flags anything that appears still-active
- Before completing this section, confirms every file matching those env-file patterns was actually searched, and lists the files searched explicitly

**Git and repository safety**
- All remotes and their URLs
- Branch list
- Records the current branch explicitly as "Archiving from:" in its own field — Step 5's branch cleanup reads this directly rather than inferring it from prose, since a resumed multi-session archive could otherwise lose track of which branch was original
- Compares the current branch against every other local and remote branch; flags it (informational only, never blocking) if another branch looks more recently active or complete — archive candidates are often neglected, so the checked-out branch isn't always the real latest state
- CI/CD pipelines that could auto-deploy on push

**Existing documentation**
- Documentation files found in the project, what each one covers, and whether it appears current or outdated

**Archive assets**
- Existing Postman or API collection files
- Screenshots, videos, sample exports, demo files
- Large or binary files that could affect Git storage

## Complexity rating

The report closes with one of four ratings:

| Rating | Meaning |
|---|---|
| Easy | Project is well-documented, dependencies are available, restoration is straightforward |
| Medium | Some gaps or unknowns, but restoration is likely achievable |
| Complex | Significant unknowns or missing dependencies; expect manual work |
| Blocked | A hard blocker exists that prevents restoration without external action |

## Rules

- Read-only only. No files created, modified, deleted, or executed.
- No external service contact.
- Credential values are always redacted.

## Output

A structured report covering all categories above, ending with the complexity rating and recommended next steps for Step 2. The command stops after the report and waits for explicit approval before Step 2 begins.

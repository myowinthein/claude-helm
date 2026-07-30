# Operational Safety Rules

Loaded as context so agents respect the same operational guardrails throughout the project. It is not a checklist to execute top to bottom — the safety scan runs only when triggered (see When to Run a Safety Scan).

## Agent Execution Boundaries

These actions always require explicit confirmation regardless of context:
- git push, force-push, tag creation
- any deployment command
- migrations, database imports, database resets
- infrastructure changes, DNS changes, secret rotation
- destructive operations (resets, drops, clears, environment recreation)

Document any project-specific additions to this list.

## When to Run a Safety Scan

Perform this scan at project bootstrap, and update it when deployment,
infrastructure, or environment configuration changes.

/helm:log Full scan mode may surface and propose safety findings for CLAUDE.md's
Hard Safety Rules section, but it does not run this scan or write
.claude/rules automatically — findings are proposed, never auto-written.

Only document risks supported by evidence in the repository — not assumptions.

**Output:** Write documented findings — the Risk/Instruction pairs, environment
classifications, and any project-specific boundary additions — into
`.claude/rules/safety.local.md`, a project-owned file adopt never overwrites so
findings survive plugin updates. Keep only a brief never-do summary in
CLAUDE.md's `## Hard Safety Rules` section, and link it to `safety.local.md` for
the detail.

## Scan Targets

Review for operational risks:
- CLAUDE.md, .claude/rules, README files, project documentation
- CI/CD pipelines, GitHub Actions, GitLab CI, deployment scripts
- Composer scripts, package scripts, Makefiles, shell scripts
- Docker, Dev Containers, Podman, Lando, Laravel Sail configuration
- Git remotes, branch structure, protected branches
- .env.example, credential files, service account files, private keys
- Queue config, scheduler config, background job definitions
- Deployment platforms (Fly.io, Railway, Render, Heroku, or similar)
- Cloud/infrastructure config (AWS, GCP, Azure, DO, Cloudflare, Supabase, Vercel, Neon, Elasticsearch, Redis, etc)

## Risk Categories

For each category, scan for evidence and document findings as:
  Risk: one-line description
  Instruction: direct rule for future agents

Avoid vague warnings. Every instruction must be actionable.

**Deployment**
Which branches auto-deploy. Whether automatic or approval-gated.
Whether tests run before deployment. Which envs are affected.

**Git**
Protected branches. Multiple remotes. Force-push exposure.
Mirror repositories. Company-owned or production remotes.

**Database**
Migrations, seeders, bulk scripts, data repair scripts, sync/import jobs
that could affect shared environments.

**Queue & Jobs**
Operations that could duplicate, lose, or disrupt background work.
Scheduler commands that affect shared state.

**Infrastructure**
Cloud storage, CDN, search indexes, caching systems —
operations that should never be performed automatically.

**Secrets & Credentials**
Where they live (env files, credential files, service accounts, keys, certs,
deployment tokens). Never print, commit, copy into docs, or expose in output.
Only identify locations and risks — never reproduce values.

**Destructive Operations**
Commands or scripts that delete data, reset environments, recreate databases,
destroy infrastructure, or clear storage.

**Project-Specific**
Generated code, vendor-managed code, legacy systems, sync processes,
unusual deployment requirements, operational dependencies unique to this repo.

## Environment Classification

For each known environment (local/staging/UAT/production/preview/etc) document:
- purpose
- ownership
- whether shared
- whether persistent (or ephemeral)
- whether safe for testing
- any access restrictions or environment-specific rules

## Minimum Evidence Checklist

Infer from actual CI/hooks/pipelines — only checks the project enforces.
Examples: unit tests, feature tests, linting, static analysis, frontend build,
generated file verification, pre-commit hooks.

Before completing this checklist, verify you have inspected: CI config files (`.github/workflows/`, `.gitlab-ci.yml`, etc.), pre-commit hook configs (`.husky/`, `.pre-commit-config.yaml`), and test/lint scripts in `package.json`, `Makefile`, or `composer.json`. List the files that informed each check.

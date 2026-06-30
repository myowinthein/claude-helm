# Step 2 — Restore and Freeze

Restore this project and freeze its environment for long-term archive recovery.

Use all relevant outputs from previous steps as context.

Goal: Get the project running locally, containerize it with pinned image versions, and export the built images to tarballs so the environment can be recovered years later without depending on external registries.

---

## Rules

- Preserve the original project as much as possible. Prefer compatibility fixes over upgrades, restoration over modernization.
- Record every file created, modified, or deleted with the reason.
- Use placeholder values for external services.
- Do not connect to production systems, live databases, or external services.
- Do not send emails, SMS, notifications, webhooks, or invoke payment gateways without explicit approval.
- Do not change Git state — no commit, push, pull, fetch, checkout, reset, rebase, merge, or history rewrite.
- Work only within the current repository. If another repository appears required, report it and stop for approval.

---

## Restoration Philosophy

The primary goal is not just to run the project today — it is to freeze the environment so it can be recovered 5–10 years from now, regardless of what runtimes, package versions, or registries are available at that time.

Docker is the preferred environment freeze mechanism. Base image tags and registries can disappear over time, so after a successful build, export all images to gzipped tarballs stored in `recovery/docker/`. Recovery then requires only Docker and the tarballs — no internet access, no registry dependency.

For project types where Docker does not apply, document exact runtime versions, SDK versions, and toolchain versions in `docs/setup.md` as the freeze mechanism instead.

---

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

- **Mobile** (React Native, Flutter, iOS, Android): Docker cannot meaningfully freeze a mobile build environment. Document exact SDK version, toolchain version, target platform version, and any required IDE version in `docs/recovery-notes.md`.
- **Desktop** (Electron, Tauri, etc.): prefer Docker for the build step if possible; otherwise document exact Node or Rust version.
- **Browser extension**: document exact browser version and extension API versions used.
- **Library / package**: Docker for the test environment; document the published package version.

---

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

---

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

---

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

---

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

---

Do not commit or push. Output the report and stop.

If Restoration Outcome is Fully Restored: proceed to Step 3 when approved.

If Restoration Outcome is anything other than Fully Restored: clearly state that proceeding to Step 3 is not recommended until blockers are resolved. List what needs to be fixed. Wait for explicit approval before continuing.

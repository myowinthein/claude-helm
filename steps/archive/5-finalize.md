# Step 5 — Finalize

Prepare the repository for final archival, clean up remotes and branches, and push to the private archive remote.

Use all relevant outputs from previous steps as context.

Goal: Seal the archive — ensure the repository is clean, safe, and pushed to the correct private remote.

---

## Rules

- Present a plan and wait for explicit approval before any destructive operation.
- Do not delete branches without explicit approval.
- Do not push until origin is confirmed as the private archive remote.
- Do not commit until all cleanup tasks are complete.

---

## Task 1 — Remote Verification

List all configured remotes and their URLs. Present them to the developer:

```
REMOTES

origin      https://github.com/company/old-project.git
upstream    https://gitlab.com/client/project.git
```

Ask:
- Which of these point to company, client, or external repositories?
- What is the private archive remote URL? This can be any Git host — GitHub, GitLab, Gitea, Forgejo, or self-hosted. The repository must already exist before proceeding; remind the developer to create it if needed.

Remove all non-personal remotes. Set origin to the provided private archive remote URL.

Do not proceed to any other task until origin is confirmed as the private archive remote.

---

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

---

## Task 3 — Git LFS for Large Files

Scan for files larger than 100 MB — both tracked and untracked.

For each file found: report path, size, type, and whether it is currently tracked by Git.

If large files exist:
- Set up Git LFS if not already configured
- Track large files with appropriate LFS patterns in `.gitattributes`
- Stage `.gitattributes` changes

If no large files exist: skip silently.

---

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

---

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

---

## Task 6 — Final Commit and Push

Show git status — all staged and unstaged changes accumulated across the full workflow.

Commit all changes. Stage all tracked and new files explicitly — do not use `git add -A` blindly. Check `git status` first and stage only the files accumulated during this workflow:
```
git add -u                                  # stage all tracked modifications
git add recovery/ docs/ README.md .gitignore  # stage new files created this session
git commit -m "chore(archive): seal project archive"
```

Push to origin using the branch identified in Task 2:
```
git push origin {main_or_master}
```

Confirm the push was successful.

---

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

Archive date, archive remote URL, and restoration complexity from Step 1.

To restore this project:

```
git clone {archive_remote_url}
```

If Docker was used:
```
docker load < recovery/docker/{service}.tar.gz
docker-compose up
```

If Docker was not used: refer to the version and toolchain documentation in `docs/setup.md`.

Full recovery instructions: `docs/setup.md`

---

Stop. Archive complete.

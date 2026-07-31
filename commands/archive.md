---
description: Explore, restore, containerize, document, and seal a project for long-term archive recovery
---

# archive

## Before starting

Only proceed if on `main` or `master`.
If on any other branch, stop and inform the user:

"archive must be run on main or master.
Current branch is {branch}. Please switch and re-run."

## State tracking

Progress and each step's report are persisted under `.archive/` so the workflow can resume across the approval gates and survive context loss between sessions.

At the start, check for `.archive/state.json`:
- If it exists and `last_completed_step` is 5, all steps are done — tell me the archive already completed and that I can delete `.archive/` to run again. Do nothing else.
- If it exists with `last_completed_step` below 5, read it, tell me which step you are resuming from, and continue from the next step. Do not re-run completed steps.
- If it does not exist, this is a fresh run — start from Step 1. Do not create `.archive/` yet: Step 1 is read-only and nothing may be modified before its approval.

State file format:
```
{
  "last_completed_step": 0,
  "updated_at": ""
}
```

After I approve each step:
- On the first write (after Step 1's approval), create `.archive/` and add `.archive/` to `.gitignore` — it is scratch, never committed to the archive.
- Write that step's report — verbatim, exactly as the step produced it — to `.archive/step{N}-{name}.md`, matching the step file name (e.g. `.archive/step1-explore.md`). Do not summarize or reformat it.
- Set `last_completed_step` to N and `updated_at` to now in `.archive/state.json`.

Later steps read these report files by name rather than relying on conversation memory.

## Sequence

1. Read prompts/archive/step1-explore.md and follow it exactly.
2. Read prompts/archive/step2-restore-and-freeze.md and follow it exactly.
3. Read prompts/archive/step3-postman.md and follow it exactly.
4. Read prompts/archive/step4-documentation.md and follow it exactly.
5. Read prompts/archive/step5-finalize.md and follow it exactly.

## Important

- Each step file will tell you when to stop and wait for approval. Always stop and wait — never proceed to the next step without explicit confirmation.
- Do not skip, combine, or summarize steps.

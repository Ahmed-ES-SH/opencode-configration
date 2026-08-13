---
description: Plan a structural refactor of part of the Laravel codebase into domain modules (read-only — plan agent)
agent: plan
---

You are in the **planning phase** of a structural-only Laravel refactor. You must NOT move, edit, or create any files in this run — this is inspection and proposal only.

The complete rules, target architecture, and hard constraints for this task are defined in the project's AGENTS.md — read it fully before doing anything else:

@AGENTS.md

## Target for this run

Plan the refactor for: **$ARGUMENTS**

## Steps (AGENTS.md §49, steps 1–4 only)

1. **Inspect** — find every file related to "$ARGUMENTS" in the *actual current repository* (models, controllers, requests, resources, services, DTOs, events, listeners, jobs, mail, routes, tests, policies, factories). Verify against the real tree — don't assume the example structure in AGENTS.md matches this project.
2. **Map** — identify the domain boundary: which classes clearly belong to "$ARGUMENTS", and which are shared/global infrastructure that must stay where it is.
3. **Propose** — lay out the exact target `app/Modules/<Domain>/...` structure: every file's current path → new path, and the resulting namespace for each.
4. **Validate** — for every file you propose to move, list what references it (imports, route files, service provider bindings, tests, factories, OpenAPI annotations) and confirm the move won't break any of them. Flag anything ambiguous instead of guessing.

## Output format

End your response with:
- A file-by-file move table (old path → new path → new namespace)
- Anything you're uncertain belongs in this module, called out explicitly
- Any pre-existing bugs or code-quality issues noticed in "$ARGUMENTS" (report only, do not fix)
- This exact line: **"Plan only — no files were changed. If this looks correct, run `/laravel-refactor-apply $ARGUMENTS` to execute it with the build agent."**

## Hard rules

- Do not move, edit, or create files.
- Do not run composer/artisan commands that change state.
- If "$ARGUMENTS" doesn't map cleanly to a single domain in the current codebase, say so and ask which specific controllers/models/services should be included, instead of guessing.

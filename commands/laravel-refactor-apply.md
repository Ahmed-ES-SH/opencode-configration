---
description: Execute a previously approved Laravel refactor plan (move files, verify, review diff — build agent)
agent: build
---

You are in the **execution phase** of a structural-only Laravel refactor. A plan for this module was already proposed earlier in this conversation by the plan agent (file-by-file move table, references, validation). Do not re-plan from scratch and do not second-guess the approved structure — execute it.

The complete rules and hard constraints are in AGENTS.md — re-check them before making changes:

@AGENTS.md

## Target for this run

Execute the approved plan for: **$ARGUMENTS**

## Before you touch anything

Look back through this conversation for the move table produced by the plan-phase command for "$ARGUMENTS". If you cannot find a matching approved plan in this conversation, **stop immediately** and tell the user to run `/laravel-refactor $ARGUMENTS` first — do not invent a plan.

If you find a plan but it's for a different target than "$ARGUMENTS", stop and flag the mismatch instead of proceeding.

## Steps (AGENTS.md §49, steps 5–8)

5. **Refactor** — move exactly the files listed in the approved plan to their approved new paths. Update namespaces, `use` statements, and route/binding references only as required by the move. Do not touch logic, queries, validation rules, response shapes, middleware, or any other behavior.
6. **Verify** — run:
   - `composer dump-autoload`
   - `php artisan optimize:clear`
   - `php artisan route:list`
   - `php artisan test`
7. **Review diff** — run `git diff` and `git status`. The diff must show only file moves, namespace/import changes, and required reference updates. Revert anything that looks like a behavioral change.
8. **Report** — summarize what moved, confirm verification results, list any pre-existing bugs or code-quality issues noticed (report only, do not fix), and flag anything skipped because it was ambiguous.

## Hard rules

- Move code, don't rewrite it. Reorganize, don't optimize.
- No bug fixes, no query/validation/response changes, no renamed business concepts (tables, columns, API fields, route names), no new packages, repositories, interfaces, or architectural patterns.
- No opportunistic cleanup, reformatting, or import re-sorting beyond what the move requires.
- Do not commit or push. Leave changes staged/unstaged for review.
- When it's unclear whether a change is structural or behavioral: **do not make it.** Stop and report it instead of guessing.

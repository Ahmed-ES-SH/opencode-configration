---
description: Implement a task/phase on the current worktree branch — no commit, no push
agent: build
---

Current git context:
!`git branch --show-current`
!`git worktree list`
!`git status --short`

Task: $ARGUMENTS

## Step 0 — always ask first: new work or continuing?

Before touching any code, ask the user one question (every single time this command runs, regardless of what was decided last time):

> "Is this a continuation of the current phase/task on this branch, or are you starting something new (a different order, or a new plan starting at phase 1)? If it's new, should I create a new git worktree/branch for it, and if so what should the branch/worktree be named and off which base branch?"

Do not guess the answer from context, and do not skip this question even if the task text looks similar to a past one. Wait for the user's answer before proceeding.

- **If continuing:** stay on the current branch/worktree exactly as-is (see Rule 1 below).
- **If new AND the user confirms a new worktree/branch:** create it now, following git worktree best practices — e.g. `git worktree add <path> -b <branch-name> <base-branch>` — using the name/base the user gave you (ask for whichever of those two they didn't specify). Then do all implementation work inside that new worktree only. Do not touch the original worktree's files.
- **If new AND the user says to stay on the current branch:** proceed on the current branch as in the "continuing" case.

## Rules — follow strictly

1. **Stay in whichever branch/worktree was established in Step 0 — and only that one.** Once Step 0 is resolved, do not create, switch, or remove any *other* branches or worktrees, and do not run further `git checkout`, `git switch`, or `git worktree add/remove` beyond the single setup step above.
2. **Never commit and never push.** Do not run `git commit`, `git add` (staging is fine only if truly needed for a tool you're using, but do not finalize a commit), `git push`, or open a PR. Leave all changes as uncommitted working-tree edits so the user can review and commit them manually.
3. **Ask before assuming.** If the task, scope, or an implementation decision is ambiguous (e.g. which phase "phase 1" refers to, which spec/plan file to follow, naming conventions, which files are in scope), stop and ask a specific question rather than guessing. Only proceed once you have what you need.
4. **Work the task to completion.** Once scope is clear, implement it fully within this turn/session — don't stop halfway or hand back a partial implementation unless you hit a genuine blocker (missing info, failing dependency, conflicting requirement). If you hit a blocker, explain it and ask rather than silently skipping it.
5. **Respect existing project conventions.** Match the codebase's existing patterns (framework, folder structure, naming, styling) rather than introducing new ones. If a spec-kit / plan / phases document exists in the repo (e.g. `specs/`, `plan.md`, `tasks.md`), read it first and implement exactly the scope requested (e.g. "phase 1 only" means stop at the end of phase 1's tasks — do not start phase 2).
6. **Review every single run with `clean-code-guard`.** After finishing the implementation for *this* order/phase — not at the end of the whole multi-phase conversation, every time this command completes a task — invoke the `clean-code-guard` skill to review the changes just made in this run. Apply any fixes it flags before summarizing. Do not skip this because a previous run was already reviewed; each order gets its own review.
7. **Use `test-guard` whenever tests are needed.** If the task requires writing or updating tests (unit, integration, etc.), use the `test-guard` skill to write/validate them rather than writing tests ad hoc.
8. **Summarize at the end.** After implementing and running `clean-code-guard` (and `test-guard` if used), give a short summary of: what was changed, which files were touched, what `clean-code-guard` flagged and how it was resolved, anything left ambiguous or deferred, and what the suggested commit message would be (for the user to commit manually — do not commit it yourself).

## Scope discipline

- If $ARGUMENTS specifies a phase/subset (e.g. "phase 1 only"), treat everything outside that phase as out of scope for this run, even if related work is visible nearby.
- Do not refactor, "clean up", or touch unrelated files/code outside the task's scope.
- Do not modify CI/CD config, `.git` internals, or repo-level settings unless the task explicitly asks for it.

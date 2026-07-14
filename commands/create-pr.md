---
description: Analyze current changes, push to a new branch, and open a PR
agent: build
subtask: false
---

Current git state:
!`git status --short`

Current branch:
!`git branch --show-current`

Diff of unstaged + staged changes:
!`git diff HEAD`

Recent commit history (for context/style):
!`git log --oneline -10`

---

## Task

You are preparing the current working-tree changes for review. Follow these steps **in order** and stop to report back if any step fails — do not force-push or overwrite existing branches.

1. **Analyze the changes above.**
   - Summarize what changed and why, grouped by concern (feature, fix, refactor, config, etc.).
   - Flag anything risky: missing validation, security concerns, breaking API changes, secrets/env leaks, or missing tests.

2. **Create a new branch** (never commit directly to `main`/`master`/the current shared branch):
   - Branch name: use `$ARGUMENTS` if provided (slugify it to kebab-case), otherwise derive a short kebab-case name from the change summary, prefixed by type — e.g. `feat/notification-service-no-redis`, `fix/refresh-token-rotation`.
   - Command: `git checkout -b <branch-name>`

3. **Commit the changes** on the new branch:
   - Stage relevant files (`git add`), excluding anything that looks like a secret, `.env`, or generated/build artifact unless it was already tracked.
   - Write a clear, conventional commit message (`type(scope): summary`) based on the actual diff — do not invent changes that aren't there.

4. **Push the new branch** to `origin`:
   - `git push -u origin <branch-name>`

5. **Open a Pull Request** (use `gh pr create` if the GitHub CLI is available; otherwise output the PR title/description and the compare URL for the user to open manually):
   - Base branch: the repo's default branch (detect via `git remote show origin` or `gh repo view`), not necessarily `main`.
   - Title: concise, conventional-commit style.
   - Description: include a "What changed" summary, a "Why" section, and a "Notes" section calling out anything from step 1 that needs reviewer attention (security, validation, performance, breaking changes).

## Output

Report back with:
- The branch name created
- The commit(s) made
- The PR URL (or the manual compare URL + suggested title/description if `gh` isn't available)
- The risk/notes list from step 1

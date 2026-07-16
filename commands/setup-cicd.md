---
description: Detect the project's real stack, then use the github-actions-cicd skill to set up GitHub Actions CI/CD
agent: build
---

Optional scope hint from the user (may be empty — e.g. "backend only", "apps/api"): $ARGUMENTS

## Step 1 — Detect the actual stack before touching the skill

Don't guess the framework from the folder name or from what the user says in passing — confirm it from the files below. A repo can have more than one app (a Next.js frontend next to a NestJS or Laravel API), and each needs different CI steps.

App-signal files found (package.json / composer.json / artisan / nest-cli.json), up to 3 levels deep, excluding vendor/node_modules:
!`find . -maxdepth 3 \( -name "package.json" -o -name "composer.json" -o -name "artisan" -o -name "nest-cli.json" \) -not -path "*/node_modules/*" -not -path "*/vendor/*" -not -path "*/.git/*" 2>/dev/null`

Lockfiles found (tells you the package manager):
!`find . -maxdepth 3 \( -name "package-lock.json" -o -name "yarn.lock" -o -name "pnpm-lock.yaml" -o -name "composer.lock" \) -not -path "*/node_modules/*" -not -path "*/vendor/*" 2>/dev/null`

Framework signals in dependencies:
!`grep -l '"next"' package.json */package.json 2>/dev/null`
!`grep -l '"@nestjs/core"' package.json */package.json 2>/dev/null`
!`grep -l 'laravel/framework' composer.json */composer.json 2>/dev/null`

Existing workflows (don't blindly overwrite these — check what's already there first):
!`ls -la .github/workflows 2>/dev/null || echo "none yet"`

For every app-signal file found above, open it and actually read the dependencies/`require` block to confirm which framework it is — `package.json` alone doesn't tell you Next.js vs NestJS vs a plain Express app, and a Laravel repo without `artisan` visible at this depth (e.g. nested one level deeper) is easy to miss. If the scope hint above names a specific app or folder, focus there; otherwise cover every app you find.

## Step 2 — Now use the github-actions-cicd skill for everything else

With the stack confirmed, follow the `github-actions-cicd` skill's process as written — don't shortcut it. In particular:

- **Ask which CD target fits before generating anything** (build-and-push only, VPS over SSH, or a platform like Vercel/Render/Railway) — this is a direct question to the user, not an assumption, and it's asked per app if the repo has both a frontend and a backend with different needs.
- Confirm whether Docker or a native runtime applies, only where the repo doesn't already make that obvious.
- Default to staging + production as separate environments with separate triggers unless the user says otherwise.
- Pull the actual CI steps and CD YAML from the skill's `references/` files for the frameworks and target you confirmed, and start from the matching starter file in `assets/workflows/` (and `assets/dockerfiles/` if Docker is in play) rather than writing workflows from a blank page.
- Finish with the secrets checklist and the sanity checklist from the skill — tell the user exactly which secrets to add and where, and flag if production isn't gated behind a review step.

Write the resulting files to `.github/workflows/`, and don't present a wall of clarifying questions up front — ask only what Step 1's detection couldn't already answer, then generate.

---
name: github-actions-cicd
description: Sets up GitHub Actions CI/CD pipelines (.github/workflows) for Next.js, NestJS, and Laravel projects — lint/test/build CI, plus CD that can build-and-push an artifact, deploy to a VPS over SSH, or deploy to a platform (Vercel/Render/Railway). Use this whenever the user asks to set up CI/CD, add GitHub Actions, automate deployment, create a pipeline, add a workflow file, or mentions .github/workflows — even if they only name one framework or don't specify how deployment should work yet. Also use it when the user asks to review, fix, or extend an existing workflow file, add a staging/production split, containerize a build with Docker for CI, or deploy over SSH/rsync to a server. Trigger this even for casual phrasing like "can you make my repo auto-deploy" or "I want tests to run on every push."
---

# GitHub Actions CI/CD Setup

## What this skill produces

A `.github/workflows/` folder with separate, focused workflow files rather than one giant pipeline:

- `ci.yml` — runs on every push and pull request: install, lint, type-check, test, build. This never deploys anything; it's a gate.
- `cd-staging.yml` — runs after CI passes on the staging branch, deploys to staging.
- `cd-production.yml` — runs after CI passes on the production branch (or a version tag), deploys to production.

Splitting CI from CD like this matters for a practical reason: CI should run on every PR from any contributor, but CD touches real secrets and real servers, so it should only run in contexts GitHub trusts (pushes to protected branches, not arbitrary PRs). Mixing them into one file makes it easy to accidentally leak deploy credentials to a fork's PR run.

## Step 1 — Work out what's actually in the repo

Don't guess the stack from the user's phrasing alone — check the repo (or ask if no repo is attached):

- `package.json` with `next` in dependencies → Next.js
- `package.json` with `@nestjs/core` in dependencies, or a `nest-cli.json` → NestJS
- `composer.json` with `laravel/framework`, plus an `artisan` file → Laravel
- Lockfile tells you the package manager: `package-lock.json` (npm), `yarn.lock` (yarn), `pnpm-lock.yaml` (pnpm), `composer.lock` (composer, always for Laravel)
- A monorepo with `apps/frontend` + `apps/backend` (or similar) needs two CI jobs and, usually, path filters so a backend-only change doesn't rebuild the frontend

If nothing is attached and the user is describing a project verbally, ask what's there rather than assuming a default stack — a NestJS API and a Laravel API need genuinely different CI steps (npm vs composer, Jest vs Pest/PHPUnit), not just a find-and-replace.

## Step 2 — Ask how deployment should actually work (don't assume this)

This is the one decision that changes the shape of the whole CD file, so it's worth a direct question rather than a default. Ask the user which of these fits, in plain terms:

1. **Build & push only** — CI produces a build artifact or Docker image (pushed to GHCR/Docker Hub) and stops. Nothing gets deployed automatically; the user (or another system) takes it from there. Good fit when the user already has their own deploy mechanism, or isn't ready to hand a server's SSH key to GitHub yet.
2. **VPS over SSH** — the workflow connects to a server the user owns (DigitalOcean, Hetzner, a VPS, etc.) and pulls the new code or image, then restarts the process. Needs an SSH key stored as a GitHub secret.
3. **Platform deploy** — Vercel for Next.js, or Render/Railway for NestJS/Laravel. No server to manage; the workflow calls the platform's CLI or a deploy hook.

A repo can mix these — e.g. Next.js frontend to Vercel, NestJS backend to a VPS. If the user has a frontend and a backend, ask about each independently rather than assuming they want the same target for both.

Also confirm, only if it's not obvious from the repo: does the backend run in **Docker**, or **natively** (`npm start` behind PM2, or `php-fpm` + queue workers)? Docker is the natural fit for build-and-push and for VPS deploys where you want the server to just run `docker compose pull && up`; native is common when the platform (Render, a shared host) manages the runtime for you. Skip this question when the target already implies the answer — Vercel deploys don't need Docker.

## Step 3 — Confirm environments

Default to **staging + production as separate environments with separate triggers**, since that's the shape that scales without rework later:

- Push to `develop` (or `staging`) → deploy to staging automatically, every time.
- Push to `main` (or a `v*.*.*` tag) → deploy to production. For production specifically, wire it through a [GitHub Environment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) with required reviewers, so a merge to main doesn't instantly hit production without anyone confirming it.

If the user says they only need one environment, collapse `cd-staging.yml` and `cd-production.yml` into a single `cd.yml` triggered off `main` — don't force the two-file structure on a solo side project that doesn't need it.

See `references/environments-and-secrets.md` for how to actually set up GitHub Environments (required reviewers, per-environment secrets) and for the branching convention this skill assumes.

## Step 4 — Generate the workflow files

Once the stack, deploy target, and environment shape are settled, write the files using the CI steps for that framework and the CD steps for that target:

| Framework CI details | Read |
|---|---|
| Next.js | `references/nextjs.md` |
| NestJS | `references/nestjs.md` |
| Laravel | `references/laravel.md` |

| CD target details | Read |
|---|---|
| Build & push (Docker image to a registry) | `references/deploy-docker.md` |
| VPS over SSH | `references/deploy-vps.md` |
| Vercel / Render / Railway | `references/deploy-platforms.md` |

Each reference file has copy-ready YAML with the current action versions (as of mid-2026) and explains the non-obvious parts — caching, service containers for test databases, build-once-deploy-many patterns, and so on. `assets/workflows/` has full starter files for the common combinations (e.g. `nextjs-ci.yml` + `nextjs-deploy-vercel.yml`, `nestjs-ci.yml` + `nestjs-deploy-docker-vps.yml`, `laravel-ci.yml` + `laravel-deploy-vps.yml`) — read the relevant one, copy it into the user's repo, then adjust names, branches, and app-specific commands (build script, migration command, port) rather than writing every workflow from scratch. If Docker was chosen, `assets/dockerfiles/` has multi-stage Dockerfiles for Next.js and NestJS to copy in alongside the workflow.

Keep each job's steps in a sensible order: checkout → set up runtime (Node/PHP) with dependency caching → install → lint → test → build. Fail fast on lint/test before spending time on a build that won't matter if the tests are red.

## Step 5 — Secrets and GitHub Environments checklist

Tell the user exactly what to add, since this is the step people usually get stuck on. The full list per CD target lives in the relevant reference file, but as a baseline point them to **Settings → Secrets and variables → Actions** for repo-level secrets, and **Settings → Environments** to create `staging` and `production` environments (this is also where per-environment secrets and required reviewers live, so a production secret never leaks into a staging run by accident).

Common ones across most setups:
- `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID` for Vercel
- A registry token (`GITHUB_TOKEN` is enough for GHCR; a PAT or Docker Hub token otherwise)
- `SSH_HOST`, `SSH_USER`, `SSH_PRIVATE_KEY` for VPS deploys
- Framework secrets the app itself needs at build/runtime — `DATABASE_URL`, `APP_KEY` (Laravel), `JWT_SECRET`, etc. — these belong in the environment the app runs in, not hardcoded in the workflow

## Step 6 — Sanity checklist before handing it back

- Does `ci.yml` actually fail if a test fails? (Don't let a misconfigured step swallow a non-zero exit code with `|| true` unless that's genuinely intended.)
- Are deploy secrets scoped to a GitHub Environment rather than plain repo secrets, at least for production?
- Does the production workflow require something more than "someone pushed to main" — a reviewer, a tag, a manual `workflow_dispatch` approval — appropriate to how careful this project needs to be?
- If it's a monorepo, do path filters (`paths:` in the trigger, or `dorny/paths-filter`) stop a frontend-only change from redeploying the backend?
- Mention that adding [Dependabot](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuring-dependabot-version-updates) for `github-actions` keeps action versions (`actions/checkout@v6` etc.) patched automatically, since pinned majors still get superseded over time.

Don't run through this checklist out loud as a wall of questions — weave it into the explanation of what was generated and flag anything that genuinely needs the user's input (mainly: secrets they need to go create, and whether production should require manual approval).

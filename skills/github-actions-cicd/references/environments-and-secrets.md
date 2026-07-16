# GitHub Environments, branching, and secrets

## Why use Environments instead of plain repo secrets

A plain repo secret (Settings → Secrets and variables → Actions → Repository secrets) is available to *every* workflow run, including one triggered from a pull request opened by someone else. A GitHub Environment (Settings → Environments → New environment) scopes secrets to only the jobs that explicitly declare `environment: <name>` — so a `PRODUCTION_DATABASE_URL` stored under the `production` environment simply isn't visible to a CI job that doesn't reference it, even if that job runs in the same repo.

Environments also unlock **required reviewers**: a job can be configured to pause and wait for a specific person (or team) to click approve before it proceeds, which is the mechanism to use for "don't let a push to main instantly hit production" without needing a separate approval bot or manual process.

## Setup

1. Settings → Environments → New environment, create `staging` and `production`.
2. On `production`, add required reviewers if deploys should be gated by a human — for a solo project this can just be the same person, functioning as a deliberate "are you sure" pause rather than a second reviewer.
3. Add environment-scoped secrets under each environment's own secrets section — `SSH_HOST`, `DATABASE_URL`, etc. can differ between staging and production, which is the point.
4. Reference the environment in the job:
   ```yaml
   jobs:
     deploy:
       environment: production
       # ...
   ```

## Branch strategy this skill assumes

- `develop` (or `staging`) → auto-deploys to staging on every push, no approval gate. This is meant to be low-friction so staging always reflects the latest work.
- `main` → deploys to production, gated by the `production` Environment's review requirement if configured.
- Feature branches → no CD at all, only `ci.yml` runs.

This isn't the only valid strategy — trunk-based development with tag-triggered production releases (`v*.*.*` tags instead of watching `main` directly) is a reasonable alternative for a project doing versioned releases rather than continuous deploys. Ask if the user's team already has a convention before assuming `develop`/`main` — some shops use `staging`/`production` branch names instead, which is just a rename, not a structural change.

## Secret naming when one workflow serves both environments

If staging and production share a single `cd.yml` (parameterized by which environment triggered it) rather than two separate files, prefix secrets by environment to avoid collisions: `STAGING_DATABASE_URL` / `PRODUCTION_DATABASE_URL`, referenced via a step that picks the right one based on `github.ref`. In practice, two separate files (`cd-staging.yml`, `cd-production.yml`) each referencing their own Environment's un-prefixed secrets (`DATABASE_URL` scoped per-Environment) is simpler to read and less error-prone — prefer that unless the user specifically wants a single parameterized file.

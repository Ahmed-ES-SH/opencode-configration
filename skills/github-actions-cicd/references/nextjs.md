# Next.js CI

## Core CI job

```yaml
name: CI

on:
  push:
    branches: ['**']
  pull_request:
    branches: [main, develop]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: 'npm' # use 'yarn' or 'pnpm' to match the project's lockfile

      - run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npx tsc --noEmit
        # Skip this step if the project is plain JavaScript

      - name: Test
        run: npm test -- --ci

      - name: Build
        run: npm run build
        env:
          # Build-time public env vars go here (NEXT_PUBLIC_*), pulled from
          # a GitHub Environment or repo secret, not hardcoded.
          NEXT_PUBLIC_API_URL: ${{ vars.NEXT_PUBLIC_API_URL }}
```

Why the trigger looks like this: `push: branches: ['**']` runs CI on every branch, which catches problems before a PR even opens. If the project already has a strict "always PR" workflow, narrowing this to `pull_request` only is fine — but running CI on push as well is cheap and catches broken commits on feature branches early.

## Notes specific to Next.js

- **`output: 'standalone'`** in `next.config.js` is worth setting whenever the deploy target is Docker or a VPS — it produces a minimal `.next/standalone` folder with only the files needed to run, instead of requiring `node_modules` on the server. Not needed for Vercel, which handles this itself.
- **Env vars split into two kinds**: `NEXT_PUBLIC_*` vars get baked into the client bundle at build time, so they must be present during `npm run build` in CI. Server-only vars (API keys, DB URLs) are only needed at runtime and shouldn't be passed to the build step at all.
- **Testing**: if the project uses Playwright for e2e rather than just Jest/Vitest unit tests, that's usually a separate job (or separate workflow) since it needs a running server and browser binaries — don't fold `npx playwright test` into the same job as the unit tests without `npx playwright install --with-deps` first.
- **Turborepo/monorepo**: if Next.js lives in a monorepo (`apps/web`), add `paths: ['apps/web/**']` to the trigger so backend-only commits don't re-run the frontend CI, and set `working-directory: apps/web` as a job default:
  ```yaml
  defaults:
    run:
      working-directory: apps/web
  ```

## Matching Ahmed's typical stack

For a Next.js + Tailwind + TypeScript frontend talking to a separate NestJS or Laravel API, the CI job above is normally sufficient as-is — there's no database to spin up for the frontend's own tests, since API calls should be mocked in unit tests. If the project has integration tests that hit a real (test) backend, add that backend as a `services:` container (see `references/nestjs.md` or `references/laravel.md` for the pattern) or run it via `docker compose up -d` before the test step.

For the deploy side, see:
- `references/deploy-platforms.md` for Vercel (the default choice for Next.js)
- `references/deploy-docker.md` + `references/deploy-vps.md` for self-hosting Next.js on a server

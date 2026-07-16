# NestJS CI

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

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U test"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: 'npm'

      - run: npm ci

      - name: Lint
        run: npm run lint

      - name: Unit tests
        run: npm run test

      - name: E2E tests
        run: npm run test:e2e
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
          JWT_SECRET: test-secret-not-for-production

      - name: Build
        run: npm run build
```

Why the `services:` block: NestJS e2e tests almost always hit a real database (through TypeORM/Prisma), and spinning up a throwaway Postgres container that lives only for the job is far more representative than mocking the DB layer — and it's what GitHub Actions' service containers exist for. Swap `postgres:16-alpine` for `mysql:8` if the project uses MySQL; the pattern is identical, just change the health-check command to `mysqladmin ping`.

## Notes specific to NestJS

- **No Redis needed for CI** — this mirrors the production architecture choice of removing Redis for the Render free tier; the same in-memory or DB-backed approach used in production should be what's tested here too, so CI stays representative.
- **Prisma projects** need a migrate step before e2e tests: `npx prisma migrate deploy` (not `migrate dev`, which prompts interactively) right after the database is healthy and before running tests.
- **Coverage**: `npm run test:cov` if the project wants a coverage report; upload it as a build artifact with `actions/upload-artifact@v4` if it needs to be inspected, or pipe it to Codecov if that's already set up.
- **Monorepo (NestJS as `apps/api`)**: same `paths:` filter and `working-directory` default pattern as the Next.js file — see `references/nextjs.md`.

## Matching Ahmed's typical setup (Render free tier, no Redis)

If the NestJS app is the one built for Render's free tier — JWT auth with refresh rotation, real-time notifications without Redis — keep CI's env vars minimal and never point CI at the real Render database. The `services:` Postgres container above is the right substitute: fast, disposable, and doesn't risk touching production data.

For the deploy side:
- `references/deploy-platforms.md` for Render/Railway (the platform-managed path, matches the free-tier setup)
- `references/deploy-docker.md` + `references/deploy-vps.md` if moving to a self-managed VPS

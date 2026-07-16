# CD: Deploy to a platform (Vercel / Render / Railway)

No server to manage — the workflow calls the platform's CLI or hits a deploy hook. This is usually the fastest path to a working pipeline and the right default when the user doesn't have (or want) their own server.

## Vercel (Next.js)

Requires three secrets from the Vercel dashboard (Settings → Tokens for the token; `vercel link` locally, then read `.vercel/project.json`, for the org/project IDs):

| Secret | Where it comes from |
|---|---|
| `VERCEL_TOKEN` | vercel.com/account/tokens |
| `VERCEL_ORG_ID` | `.vercel/project.json` after running `vercel link` locally |
| `VERCEL_PROJECT_ID` | same file |

```yaml
name: Deploy to Vercel (production)

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    env:
      VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
      VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
    steps:
      - uses: actions/checkout@v6
      - run: npm install --global vercel@latest
      - run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}
      - run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}
      - run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
```

For staging, drop `--prod` everywhere and use `--environment=preview` on the `pull` step instead — this produces a preview deployment URL rather than promoting to the production domain.

Note that if the repo is already connected to Vercel's own GitHub integration (the default when importing a project through the Vercel dashboard), Vercel deploys automatically on every push without any of this — this workflow is only needed when the user wants deploys gated by GitHub Actions' own CI first (lint/test must pass before Vercel even attempts a build), or wants deploys to not fire from Vercel's side at all. Ask which is the case before adding this, since running both at once means double deploys.

## Render (NestJS / Laravel)

Render deploy hooks are the simplest option: each service has a secret URL (Service → Settings → Deploy Hook) that triggers a deploy on request, no CLI or auth token needed beyond keeping the URL secret.

```yaml
name: Deploy to Render (production)

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Trigger Render deploy
        run: curl -fsS "${{ secrets.RENDER_DEPLOY_HOOK_URL }}"
```

Store the hook URL as a secret (`RENDER_DEPLOY_HOOK_URL`) — it's a bearer credential even though it looks like a plain link. To deploy a specific commit rather than whatever Render would otherwise pick up, append `?ref=<sha>`: `curl -fsS "${{ secrets.RENDER_DEPLOY_HOOK_URL }}&ref=${{ github.sha }}"`.

This only triggers the deploy; it doesn't wait for it to finish or fail the job if the deploy fails, since Render's hook response only confirms the trigger was accepted. That's usually fine for a simple pipeline — mention it to the user so they know the green checkmark means "deploy started," not "deploy succeeded."

## Railway (NestJS / Laravel)

Two options depending on whether the user wants GitHub Actions driving the deploy, or Railway's own GitHub integration doing it automatically:

**Railway CLI from Actions** (explicit control, fits the same CI-gates-deploy pattern as the others):

```yaml
name: Deploy to Railway (production)

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v6
      - run: npm install -g @railway/cli
      - run: railway up --service <service-name>
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
```

Use a **Project Token** (scoped to one environment, deployment-only permissions) rather than a personal account token for `RAILWAY_TOKEN` — generate it from the Railway project's settings.

**Railway's own GitHub integration** deploys automatically on push with zero workflow needed, similar to Vercel's default behavior — worth mentioning as the simpler alternative if the user doesn't specifically need GitHub Actions to gate the deploy behind CI passing first.

## Staging vs production for platform deploys

Same branch-split pattern as the VPS examples: point the staging version of any of these at `develop` with a Vercel preview / a separate Render staging service's hook / a Railway staging environment, and the production version at `main` wrapped in a GitHub Environment named `production`.

# CD: Deploy to a VPS over SSH

Two flavors, depending on the Docker decision from Step 2 of `SKILL.md`: **Docker-based** (server just pulls and restarts a container) or **native** (server pulls source and runs `npm ci`/`composer install` itself). Docker-based is simpler to reason about and matches what CI already tested; native avoids running Docker on a small VPS at all, which matters on very memory-constrained boxes.

## Required secrets

| Secret | Value |
|---|---|
| `SSH_HOST` | server IP or hostname |
| `SSH_USER` | deploy user (use a non-root user scoped to the app directory, not `root`) |
| `SSH_PRIVATE_KEY` | private key content, generated specifically for CI — don't reuse a personal key |
| `SSH_PORT` | usually 22, only needed if non-default |

Generate a dedicated key pair for this (`ssh-keygen -t ed25519 -C "github-actions-deploy"`), add the public half to the server's `~/.ssh/authorized_keys` for the deploy user, and store only the private half as `SSH_PRIVATE_KEY`.

## Docker-based deploy

```yaml
name: Deploy to VPS (staging)

on:
  push:
    branches: [develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy over SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT }}
          script: |
            cd /var/www/app
            docker compose pull
            docker compose up -d
            docker image prune -f
```

This assumes a `docker-compose.yml` already lives on the server referencing `ghcr.io/<owner>/<repo>:latest` (or a specific tag/SHA for more controlled rollouts — swap `:latest` for `${{ github.sha }}` in both the build workflow's tags and the compose file if reproducible deploys matter more than convenience). `docker image prune -f` keeps old layers from slowly filling the disk.

## Native (no Docker) deploy — Node apps (NestJS/Next.js self-hosted)

```yaml
name: Deploy to VPS (production)

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy over SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT }}
          script: |
            cd /var/www/app
            git pull origin main
            npm ci --omit=dev
            npm run build
            pm2 reload app --update-env
```

`pm2 reload` (not `restart`) does a near-zero-downtime reload for cluster-mode apps. If the app isn't under PM2 yet, that's worth setting up on the server once, separately from this workflow — a raw `node dist/main.js &` won't survive a server reboot or crash.

## Native (no Docker) deploy — Laravel

```yaml
name: Deploy to VPS (production)

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy over SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT }}
          script: |
            cd /var/www/app
            git pull origin main
            composer install --no-dev --optimize-autoloader --no-interaction
            php artisan migrate --force
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache
            php artisan queue:restart
```

`queue:restart` matters even if nothing about the queue changed — Laravel's queue workers cache the application code they loaded at boot, so without this, a deployed code change won't actually take effect for queued jobs until the worker process is restarted some other way.

## Zero-downtime note

Both native examples above have a brief gap between `git pull` and the app being fully ready again. For anything more traffic-sensitive than a side project, a symlink-swap release strategy (releases in timestamped folders, `current` symlinked to the active one, swap the symlink only after the new release is fully prepared) removes that gap — worth mentioning to the user as a next step rather than building it into the default template, since it adds real complexity that most projects don't need yet.

## Staging vs production trigger split

Notice the two examples above differ only in `branches:` and `environment:`. That's intentional — copy the pattern, not just the YAML: staging watches `develop` and deploys unattended, production watches `main` and runs inside a GitHub Environment named `production` (set up under Settings → Environments) so required reviewers can gate it if the user wants a manual approval step before anything touches production.

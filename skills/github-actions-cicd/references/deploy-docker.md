# CD: Build & push a Docker image (no server deploy)

This is the right choice when the workflow's job is to produce a deployable artifact and stop — the user (or another system, or a separate VPS-deploy workflow) takes it from there. It's also the first half of both the "Docker + VPS" combo (see `deploy-vps.md`) and a platform's image-backed deploy (see `deploy-platforms.md`).

## Workflow

```yaml
name: Build & Push Image

on:
  push:
    branches: [main]
    tags: ['v*.*.*']

permissions:
  contents: read
  packages: write

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    needs: [] # point this at the CI job's ID if calling from the same workflow
    steps:
      - uses: actions/checkout@v6

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=
            type=semver,pattern={{version}}
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Why `type=gha` caching matters: without it, every build re-downloads and re-installs every dependency layer from scratch. `cache-to: type=gha,mode=max` caches every layer (not just the final one), which is what actually speeds up rebuilds when only application code — not dependencies — changed.

`GITHUB_TOKEN` is enough to push to GHCR (`ghcr.io`) with no extra secret setup. For Docker Hub instead, swap the registry to (leave blank for Docker Hub's default) and use `vars.DOCKERHUB_USERNAME` / `secrets.DOCKERHUB_TOKEN`.

## Dockerfiles

Use `assets/dockerfiles/Dockerfile.nextjs` or `assets/dockerfiles/Dockerfile.nestjs` as a starting point — both are multi-stage builds that keep the final image small (deps → build → slim runtime, not shipping devDependencies or build tooling in the final layer). For Laravel, a PHP-FPM + Nginx combo is more involved; ask the user whether they want a single-container (php-fpm serving via built-in `artisan serve`, fine for low-traffic APIs) or a two-container (php-fpm + nginx, more production-typical) setup before generating one, since it changes the Dockerfile and any `docker-compose.yml` significantly.

## Where this feeds into

- If the next step is deploying to a VPS, see `deploy-vps.md` — it picks up from `ghcr.io/<repo>:latest` and pulls it on the server.
- If the next step is a platform's image-backed service (Render supports deploying a prebuilt image), see `deploy-platforms.md`.

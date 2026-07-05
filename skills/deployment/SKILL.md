---
name: deployment
description: Manages build, release, and hosting workflows for Next.js applications. Use when discussing builds, deployment, Docker, Vercel, Node server, static export, environment setup, CI/CD, or production rollout.
---

# Deployment Skill

## Trigger

Use when the request mentions: **build, release, hosting, Docker, Vercel, Node server, static export, environment setup, platform configuration, or production rollout**.

---

## Scope

- Vercel deployment (recommended)
- Node.js server deployment (standalone)
- Docker containerization
- Static export (fully static sites)
- Environment variable management per environment
- CI/CD with GitHub Actions
- Production build validation
- Pre-deployment checklist

---

## Build Commands

```bash
# Development
npm run dev

# Production build
npm run build    # Outputs to .next/
npm run start    # Starts production server

# Type check without building
npx tsc --noEmit

# Lint
npm run lint
```

---

## Vercel (Recommended)

Vercel is the native platform for Next.js — zero-config for App Router, Edge, ISR, and streaming.

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel

# Production deploy
vercel --prod
```

**next.config.ts for Vercel:**
```typescript
// next.config.ts
import type { NextConfig } from 'next'

const config: NextConfig = {
  // Vercel handles this automatically
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'cdn.yourapp.com' },
    ],
  },
}

export default config
```

**Environment Variables in Vercel:**
- Set via Vercel Dashboard → Project → Settings → Environment Variables
- Or: `vercel env add API_URL production`
- Use separate values for `development`, `preview`, `production`

---

## Node.js Standalone Server

For self-hosted deployments (VPS, EC2, DigitalOcean).

```typescript
// next.config.ts
const config: NextConfig = {
  output: 'standalone', // Creates minimal self-contained server
}
```

```bash
# Build
npm run build

# Output structure after standalone build:
# .next/standalone/          ← copy this
# .next/standalone/server.js ← node server
# .next/static/              ← copy to .next/standalone/.next/static/
# public/                    ← copy to .next/standalone/public/

# Start standalone server
node .next/standalone/server.js
```

**Environment setup:**
```bash
export PORT=3000
export HOSTNAME=0.0.0.0
export NEXTAUTH_URL=https://yourapp.com
export NEXTAUTH_SECRET=your-secret
export API_URL=https://api.yourapp.com
node .next/standalone/server.js
```

---

## Docker

```dockerfile
# Dockerfile
FROM node:20-alpine AS base

# Install dependencies
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Build
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ENV NEXT_TELEMETRY_DISABLED 1
RUN npm run build

# Production image
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

CMD ["node", "server.js"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NEXTAUTH_URL=https://yourapp.com
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
      - API_URL=${API_URL}
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certbot/conf:/etc/letsencrypt:ro
    depends_on:
      - web
```

---

## Static Export (No Node.js Required)

Use only for fully static sites (no server actions, no route handlers, no ISR).

```typescript
// next.config.ts
const config: NextConfig = {
  output: 'export',
  trailingSlash: true, // for static hosting compatibility
  images: {
    unoptimized: true, // next/image needs a server for optimization
  },
}
```

```bash
npm run build
# Output in: out/
# Deploy out/ to S3, Cloudflare Pages, GitHub Pages
```

**Limitations of static export:**
- No server actions
- No route handlers
- No middleware
- No ISR (revalidate)
- No dynamic routes without generateStaticParams

---

## GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npx tsc --noEmit
      - run: npm run lint
      - run: npm test -- --ci --coverage
      - run: npm run build
        env:
          API_URL: ${{ secrets.API_URL }}
          NEXTAUTH_SECRET: ${{ secrets.NEXTAUTH_SECRET }}
          NEXTAUTH_URL: ${{ secrets.NEXTAUTH_URL }}

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
```

---

## Environment Files Strategy

```bash
.env                  # Checked in — default values, no secrets
.env.local            # Never committed — local dev overrides
.env.development      # Dev-specific (checked in if no secrets)
.env.production       # Prod defaults (checked in if no secrets)
.env.example          # Template — always commit this

# .env.example
NEXTAUTH_SECRET=        # Generate with: openssl rand -base64 32
NEXTAUTH_URL=http://localhost:3000
API_URL=https://api.yourapp.com
NEXT_PUBLIC_APP_URL=https://yourapp.com
```

---

## Pre-Deployment Checklist

```
□ npm run build succeeds with no errors
□ npx tsc --noEmit passes with no type errors
□ npm run lint passes with no errors
□ All tests pass (npm test)
□ NEXTAUTH_SECRET is set (minimum 32 chars)
□ NEXTAUTH_URL matches production domain exactly
□ API_URL points to production backend
□ Security headers configured (X-Frame-Options, CSP, etc.)
□ robots.txt configured correctly (noindex for staging)
□ Error tracking set up (Sentry or similar)
□ next.config.ts has output: 'standalone' for Docker
□ .env.example is up to date
□ No console.log or debug code in production
□ next/image remotePatterns includes all CDN domains
□ Bundle analyzed — no unexpected large packages
```

---

## Notes

- **Vercel Edge Runtime** — use `export const runtime = 'edge'` for ultra-low latency middleware and route handlers; note: no native Node.js APIs
- **Health check endpoint** — add `app/api/health/route.ts` returning `{ status: 'ok' }` for load balancer checks
- **Rolling deployments** — ensure your app handles old and new sessions simultaneously during rollout
- **Preview deployments** — Vercel creates these automatically per PR; set `NEXTAUTH_URL` to the preview URL using `VERCEL_URL` env var

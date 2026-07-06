---
name: nestjs-auth-logic
description: Implements authentication and authorization logic for NestJS APIs — Passport local/JWT strategies, access + refresh token flows with rotation and reuse detection, AuthGuard/RolesGuard implementations, @Public()/@Roles()/@CurrentUser() decorators, bcrypt password hashing, password reset flows, session-based auth, and wiring a Next.js frontend (middleware, httpOnly cookies, silent refresh) to a NestJS auth API. Use this whenever the user asks about NestJS login, JWT guards, protecting routes, refresh tokens, RBAC/roles, password hashing, auth middleware, or securing a NestJS backend — even if they just say "add auth to my NestJS API", "protect this route", "add login to my app", "how do I check user roles", or "my JWT keeps expiring". Scope is authentication/authorization logic only — do not use this for unrelated NestJS topics like caching, queues, file uploads, or general CRUD scaffolding.
---

# NestJS Authentication & Authorization Logic

This skill covers one thing well: how a NestJS API authenticates users and authorizes their access to routes. It does not cover general NestJS architecture, unrelated modules, or frontend styling — stay focused on auth logic and pull in other skills/knowledge for the rest.

## Why this is organized as reference files

Auth touches several genuinely separate concerns (token issuance, rotation, role checks, password storage, frontend wiring). Loading all of it at once would bury the part that's actually relevant to the current question. Read only the reference file(s) that match what the user is asking about — don't dump every pattern into one answer.

| File | Read this when the user is asking about... |
|---|---|
| `references/jwt-authentication.md` | Setting up login, Passport local/JWT strategies, issuing access tokens, protecting routes with `@nestjs/passport` |
| `references/refresh-tokens.md` | Refresh tokens, "why does my token expire", silent refresh, logout, token theft/reuse detection |
| `references/guards-and-decorators.md` | Global auth guards, `@Public()` routes, extracting `req.user`, custom param decorators |
| `references/rbac-authorization.md` | Roles, permissions, `@Roles()`, admin-only routes, "how do I check if a user can do X" |
| `references/password-security.md` | Hashing passwords, bcrypt, password reset flows, brute-force protection on login |
| `references/session-based-auth.md` | Cookie/session auth instead of JWT, server-rendered apps, instant revocation needs |
| `references/nextjs-integration.md` | Connecting a Next.js frontend to the NestJS auth API — cookies, middleware, CORS, calling protected endpoints |

## First decision: JWT vs sessions

Ask this before writing any code, since it changes almost everything downstream:

- **Stateless JWT (access + refresh token pair)** — default choice for a NestJS API consumed by a separate Next.js frontend, a mobile app, or third-party clients. Scales horizontally with no shared session store. Go to `jwt-authentication.md` + `refresh-tokens.md`.
- **Server-side sessions** — better when the API and frontend share a domain/cookie jar, you need to instantly kill a session (ban a user, force logout everywhere) without a revocation list, or you're not exposing the API to non-browser clients. Go to `session-based-auth.md`.

If the user hasn't said which and it isn't obvious from their stack, ask — don't default silently, since retrofitting one onto the other later is expensive.

## Non-negotiable security defaults

These apply regardless of which reference file you're pulling from. Call them out in any auth code you write, don't silently skip them to keep an example short:

- Secrets (`JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, session secret) come from environment variables via `@nestjs/config`, never hardcoded, never the same value for access and refresh.
- Access tokens are short-lived (10–15 min). Refresh tokens are long-lived (7–30 days) but rotated on every use — see `refresh-tokens.md`.
- Passwords are hashed with bcrypt (cost factor 10–12) or argon2, never encrypted or stored plain. See `password-security.md`.
- Login/register/refresh/password-reset endpoints are rate-limited (`@nestjs/throttler`) — these are the endpoints brute-force attacks target.
- Auth error responses are generic ("Invalid credentials") — never reveal whether an email exists or whether the password vs. username was wrong.
- Tokens delivered to browsers use httpOnly, `secure`, `sameSite` cookies, not `localStorage` — see `nextjs-integration.md` for why and how.
- Guards fail closed: prefer a global guard with an explicit `@Public()` opt-out (see `guards-and-decorators.md`) over per-route `@UseGuards()` that's easy to forget on a new route.

## Assembled shape of a JWT auth module

This is the skeleton every reference file below plugs into — skim it first, then jump to the file that covers the part you need in depth.

```
auth/
├── auth.module.ts        # wires PassportModule + JwtModule + strategies + guards
├── auth.controller.ts     # POST /auth/login, /auth/refresh, /auth/logout, /auth/register
├── auth.service.ts        # validateUser(), login(), refreshTokens(), logout()
├── strategies/
│   ├── local.strategy.ts
│   ├── jwt.strategy.ts
│   └── jwt-refresh.strategy.ts
├── guards/
│   ├── jwt-auth.guard.ts
│   └── roles.guard.ts
└── decorators/
    ├── public.decorator.ts
    ├── roles.decorator.ts
    └── current-user.decorator.ts
```

When implementing, build in this order: strategies → guards → decorators → service methods → controller routes. Guards depend on strategies existing; the controller depends on the service methods existing. Building in the wrong order means stubbing things out and revisiting them.

## Response structure

Structure answers as: a short explanation of the concept, the NestJS code (plus matching Next.js code when the question touches the frontend), the concrete steps to wire it in, and a short security/performance notes section. Don't restate an entire reference file — pull only the pieces relevant to the question asked.

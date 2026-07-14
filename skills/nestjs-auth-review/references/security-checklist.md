# Core Auth Security Checklist

Check every applicable item below against the actual code found, not against what the person says the code does. Each item states what compliant looks like and what the common failure looks like, so a deviation is easy to recognize either way.

## 1. Secrets & configuration

- **Secrets come from environment variables**, loaded via `@nestjs/config` (or equivalent), never hardcoded in source. Fail: a literal string passed as `secretOrKey`, a fallback default value baked into code (e.g. `config.get('JWT_SECRET', 'devsecret')` shipped as-is), or a secret committed in `.env` checked into version control.
- **Access and refresh tokens use distinct secrets.** Fail: `JWT_ACCESS_SECRET` and `JWT_REFRESH_SECRET` resolving to the same value, or a single shared `JWT_SECRET` used for both.
- **Session secret (if sessions are used) is a long random value from env**, not a short/guessable string.
- Secrets are of adequate length/entropy where this is visible (e.g. an `.env.example` placeholder that's obviously short, like `secret123`, is worth flagging as a template-quality issue even though the real value is presumably different — it invites someone to actually ship the placeholder).

## 2. Token issuance & lifetime

- **Access tokens are short-lived** — 10–15 minutes is the reference range. Fail: hours/days-long access token expiry, or no `expiresIn` set at all (defaults to no expiry in some configurations).
- **Refresh tokens are long-lived but bounded** — days to a few weeks, not indefinite/no-expiry.
- **JWT payload contains no sensitive data** — id, email, roles is the reference shape. Fail: password hash, internal flags, or other sensitive fields inside the signed payload (JWTs are base64, not encrypted — anyone holding the token can read the payload).
- **`expiresIn` is set in exactly one place** (either the module-level `signOptions` or per-call, not both inconsistently) — conflicting expiry configuration is a correctness bug even if not a security one.

## 3. Refresh token rotation & reuse detection

- **Every refresh issues a brand-new access + refresh pair and invalidates the old refresh token.** Fail: the same refresh token remains valid across multiple refresh calls ("no rotation").
- **Refresh tokens are stored hashed, not raw**, in a per-session/per-device row (not a single column on the user, which can't support "log out of all devices" or reuse detection properly).
- **Reuse of an already-rotated refresh token is treated as theft** — presenting a stale/replayed refresh token should revoke all sessions for that user and force re-login, not silently fail or silently succeed.
- **Refresh cookie is scoped to the refresh endpoint's path** (e.g. `/auth/refresh`), not sent on every request as `path: '/'` — this is a defense-in-depth item, not critical on its own, but is cheap to get right.
- There's some mechanism (cron, DB TTL, cleanup job) for removing expired/revoked token rows — an unbounded, ever-growing table is a Low finding, not a security one, but worth noting.

## 4. Guards & route protection

- **The app fails closed**: either every controller/route explicitly applies an auth guard, or (preferred) there's a global guard with an explicit opt-out for the few public routes (login, register, refresh, health checks, webhooks). Fail: protection applied per-route with no global default, especially if any route in the codebase turns out to have no guard and isn't meant to be public — treat any accidentally-unprotected sensitive route as **Critical**, not just a style nit.
- **Guard order is correct where multiple guards compose** — e.g. a `RolesGuard` that reads `req.user` must run after the auth guard has populated it, whether via explicit `@UseGuards(JwtAuthGuard, RolesGuard)` ordering or because the auth guard is already global.
- If the app registers a global guard, confirm `@Public()` (or equivalent) is applied *only* to routes that should genuinely be unauthenticated — flag any route marked `@Public()` that returns or mutates user-specific data.

## 5. RBAC / authorization

- **Role checks happen on the server for every protected action**, not just hidden in the frontend UI. A role/permission check that exists only as conditional rendering in Next.js with no matching backend enforcement is a **Critical** finding — nothing stops a direct API call from bypassing it.
- **Coarse role checks vs. resource ownership are not conflated.** A route that lets any user with role `Editor` edit *any* resource (not just their own) when the intent was clearly "edit your own content" is a finding — check for a missing ownership/policy check (`resource.ownerId !== user.id`) alongside the role check.
- If roles are embedded in the JWT payload for performance, confirm the team is aware role changes/revocations won't take effect until the access token expires — this is an acceptable tradeoff, not automatically a finding, but call it out if the app has a "ban user" or "revoke admin" feature that assumes instant effect and doesn't actually get it.

## 6. Password handling

- **Passwords are hashed with bcrypt (cost 10–12) or argon2** — never MD5/SHA-1/SHA-256 alone (too fast, brute-forceable at scale), never encrypted (reversible), never stored plain. Any of these is **Critical**.
- **Comparison uses the library's constant-time compare** (`bcrypt.compare`), never a manual re-hash-and-`===` check.
- **Registration/login DTOs validate password strength server-side** (`class-validator` or equivalent) — a frontend-only strength meter with no backend enforcement is a finding.
- **Auth error responses don't reveal which check failed** — "Invalid credentials" generically, not "user not found" vs. "wrong password" as distinguishable responses/status codes.
- **Sensitive auth endpoints are rate-limited**: login, register, refresh, request-password-reset, resend-verification. Fail: `@nestjs/throttler` (or equivalent) present in the project but not applied to these specific routes, or absent entirely.

## 7. Session-based auth (only if the app uses sessions instead of/alongside JWT)

- **Session store is not in-memory** in a production configuration — `MemoryStore` doesn't survive restarts or work across multiple instances; look for Redis or another shared store.
- **Session cookie has `httpOnly`, `secure`, and an appropriate `sameSite`** value for the frontend/API origin relationship.
- **CSRF protection exists for state-changing requests** if the frontend and API are on different origins — `sameSite` alone is not sufficient once cross-site, look for a custom header check or double-submit token.
- Only the user id (not the full user object) is stored in the session; the record is re-fetched on deserialize.

## 8. Cookie & transport handling (JWT-over-cookies pattern)

- **Tokens delivered to browsers use httpOnly cookies, not `localStorage`/`sessionStorage`** for anything long-lived (refresh tokens). An access token kept in memory (JS variable/React state) is fine and expected; an access *or* refresh token in `localStorage` is a finding — one XSS anywhere in the app or a dependency means full token theft.
- **`secure` and appropriate `sameSite`** are set on any auth cookie. `sameSite: 'none'` without a clear cross-site necessity, or without `secure: true` alongside it, is a finding.
- **CORS is configured with an exact origin, not `*`**, whenever `credentials: true` is set (required for cookies to work cross-origin, and `*` + credentials is actually rejected by browsers, so if it's present it means the CORS config is broken, not just loose).

## 9. Next.js ↔ NestJS wiring (if the frontend code is in scope for this review)

- Client-side code never receives the raw NestJS API URL/secrets unnecessarily if a Route Handler proxy pattern is in use — check that the proxy doesn't leak the upstream URL or forward auth headers/cookies to unrelated third parties (analytics, error trackers).
- Middleware-level auth checks (cookie presence) are treated as UX only — confirm the actual protected NestJS routes still enforce their own guard rather than trusting that middleware already handled it.

## Summary line format

For each of the 9 categories above, the report's "Checklist Coverage" section gets one line: category name, and either "✅ Reviewed, no issues" / "⚠️ See findings below (N)" / "➖ Not applicable — `<short reason>`".

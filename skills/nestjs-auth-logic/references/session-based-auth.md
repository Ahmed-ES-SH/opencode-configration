# Session-Based Authentication (Alternative to JWT)

Use this instead of `jwt-authentication.md` when: the API and frontend share a domain/cookie jar (not a separately-hosted Next.js app calling a cross-origin API), instant server-side revocation matters more than horizontal statelessness, or there's no need to expose the API to non-browser clients (mobile apps, third-party integrations). If that's not the situation, use the JWT flow instead — don't run both simultaneously without a clear reason.

## Install

```bash
npm i express-session connect-redis passport passport-local
npm i -D @types/express-session @types/passport-local
```

A production session store must persist outside process memory (Redis via `connect-redis` shown here) — the default in-memory `MemoryStore` leaks memory and doesn't survive restarts or work across multiple instances.

## Session middleware setup

```typescript
// main.ts
import * as session from 'express-session';
import { RedisStore } from 'connect-redis';
import { createClient } from 'redis';
import * as passport from 'passport';

const redisClient = createClient({ url: process.env.REDIS_URL });
await redisClient.connect();

app.use(
  session({
    store: new RedisStore({ client: redisClient }),
    secret: process.env.SESSION_SECRET, // long random string, env-only
    resave: false,
    saveUninitialized: false,
    cookie: {
      httpOnly: true,
      secure: true,
      sameSite: 'lax', // 'strict' if frontend and API are same-site
      maxAge: 24 * 60 * 60 * 1000,
    },
  }),
);
app.use(passport.initialize());
app.use(passport.session());
```

## Serialize / deserialize

```typescript
// auth/session.serializer.ts
import { Injectable } from '@nestjs/common';
import { PassportSerializer } from '@nestjs/passport';
import { UsersService } from '../users/users.service';

@Injectable()
export class SessionSerializer extends PassportSerializer {
  constructor(private readonly usersService: UsersService) {
    super();
  }

  serializeUser(user: { id: string }, done: (err: Error, id: string) => void) {
    done(null, user.id); // only the id goes into the session store
  }

  async deserializeUser(id: string, done: (err: Error, user: any) => void) {
    const user = await this.usersService.findById(id);
    done(null, user ?? null);
  }
}
```

Only the user id is stored in the session — the full user record is re-fetched from the DB on each request via `deserializeUser`. That DB hit per request is the main cost trade-off versus JWT (which carries the payload and needs no lookup for basic claims).

## Guard

```typescript
// auth/guards/session-auth.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';

@Injectable()
export class SessionAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    return request.isAuthenticated(); // provided by passport.session()
  }
}
```

## Login / logout

```typescript
@UseGuards(LocalAuthGuard) // passport-local, same strategy as the JWT flow
@Post('login')
login(@Request() req) {
  return req.user; // session is established by passport at this point
}

@Post('logout')
logout(@Request() req, @Res({ passthrough: true }) res: Response) {
  req.logout((err) => { if (err) throw err; });
  req.session.destroy(() => {});
  res.clearCookie('connect.sid');
  return { success: true };
}
```

## CSRF

Cookie-based auth is vulnerable to CSRF in a way bearer-token auth isn't, since browsers attach cookies automatically to cross-site requests. `sameSite: 'strict'` (or `'lax'`, if cross-site GET navigation needs to preserve the session) closes most of this, but for state-changing requests from a separately-hosted frontend, also verify a custom header (e.g., `X-Requested-With`) or a double-submit CSRF token — plain cookies with only `sameSite` protection are not sufficient once the frontend is on a different origin than the API.

## Implementation steps

1. Stand up Redis (or another shared store) — don't ship `MemoryStore` to production.
2. Add session middleware + passport session init in `main.ts`, before route registration.
3. Add `SessionSerializer`, register it as a provider in `AuthModule`.
4. Reuse the same `LocalStrategy` from `jwt-authentication.md` — the credential-checking logic doesn't change, only how the resulting session is tracked.
5. Add `SessionAuthGuard`, register globally the same way as `JwtAuthGuard` in `guards-and-decorators.md` if most routes should require a session by default.
6. Add CSRF protection appropriate to the frontend's origin relationship with the API.

## Security & performance notes

- Instant revocation is the main advantage over JWT: deleting the session row (or flipping a flag checked in `deserializeUser`) ends access immediately, no waiting for a token to expire.
- Every authenticated request costs a session-store round trip (Redis) plus a DB lookup in `deserializeUser` — cache the deserialized user briefly if this becomes a bottleneck under load, but keep the cache TTL short enough that a ban/role-change still takes effect quickly.
- If the API must also serve non-browser clients (mobile app, public API), sessions don't fit well for those clients — consider supporting both a session guard (browser) and a JWT guard (API clients) side by side rather than forcing one model on both.

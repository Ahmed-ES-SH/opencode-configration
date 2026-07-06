# Wiring a Next.js Frontend to the NestJS Auth API

Assumes the JWT flow from `jwt-authentication.md` + `refresh-tokens.md`: NestJS issues a short-lived access token in the response body and a long-lived refresh token as an httpOnly cookie.

## Why not localStorage for tokens

Storing the access token in `localStorage` (or any JS-readable storage) makes it readable by any script running on the page — one XSS vulnerability anywhere in the app (including a compromised third-party dependency) means full token theft. httpOnly cookies aren't readable by JS at all, which is why the refresh token goes there. The access token itself is short-lived enough that keeping it in memory (a React context/store, cleared on refresh) rather than persisted storage is the safer default — it's lost on a hard page reload, which the silent-refresh flow below handles.

## NestJS CORS config (required for cookies to work cross-origin)

```typescript
// main.ts
app.enableCors({
  origin: process.env.FRONTEND_URL, // exact origin, not '*' — required when credentials: true
  credentials: true,
});
```

`credentials: true` on the server and `credentials: 'include'` on every frontend fetch are both required — missing either one means the refresh cookie silently won't be sent/set.

## Login from a Next.js Server Action

```typescript
// app/(auth)/login/actions.ts
'use server';

import { cookies } from 'next/headers';

export async function login(email: string, password: string) {
  const res = await fetch(`${process.env.API_URL}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password }),
    credentials: 'include',
  });

  if (!res.ok) {
    return { error: 'Invalid credentials' };
  }

  const { accessToken } = await res.json();
  // Forward the Set-Cookie header from NestJS to the browser.
  const setCookie = res.headers.get('set-cookie');
  if (setCookie) {
    cookies().set('refresh_token', extractCookieValue(setCookie), {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      path: '/',
    });
  }
  return { accessToken };
}
```

In practice, many teams instead let the Next.js server itself proxy to NestJS (a Route Handler at `/api/auth/login` that calls NestJS and re-sets the cookie under the Next.js domain) so the browser only ever talks to the Next.js origin — simpler CORS story, and the NestJS API URL never reaches client-side code. Reach for that pattern when the two apps are on different domains rather than fighting cross-site cookie rules.

## Middleware: protecting routes

Edge Middleware can't easily verify an HttpOnly-cookie-stored *refresh* token's signature against a Node-only secret store, so the common pattern is a light presence check in middleware (fast, no crypto) and real verification happening server-side on the actual request to NestJS, which will 401 if the token is invalid regardless of what middleware decided:

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const hasSession = request.cookies.has('refresh_token');
  const isAuthRoute = request.nextUrl.pathname.startsWith('/login');

  if (!hasSession && !isAuthRoute) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  if (hasSession && isAuthRoute) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/login'],
};
```

This is a UX optimization (avoid rendering a protected page just to have it immediately fail), not the actual security boundary — the real enforcement is `JwtAuthGuard` on the NestJS side. Never treat middleware's cookie-presence check as sufficient authorization on its own.

## Silent refresh: a fetch wrapper that retries once on 401

Since the access token lives in memory and is lost on reload, the app needs to silently exchange the refresh cookie for a new access token on load and whenever a request 401s mid-session:

```typescript
// lib/api-client.ts
let accessToken: string | null = null;

export function setAccessToken(token: string | null) {
  accessToken = token;
}

async function refreshAccessToken(): Promise<string | null> {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/auth/refresh`, {
    method: 'POST',
    credentials: 'include', // sends the httpOnly refresh cookie
  });
  if (!res.ok) return null;
  const { accessToken: newToken } = await res.json();
  setAccessToken(newToken);
  return newToken;
}

export async function apiFetch(path: string, init: RequestInit = {}) {
  const doFetch = (token: string | null) =>
    fetch(`${process.env.NEXT_PUBLIC_API_URL}${path}`, {
      ...init,
      credentials: 'include',
      headers: {
        ...init.headers,
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
      },
    });

  let res = await doFetch(accessToken);
  if (res.status === 401) {
    const newToken = await refreshAccessToken();
    if (!newToken) throw new Error('Session expired');
    res = await doFetch(newToken);
  }
  return res;
}
```

Call `refreshAccessToken()` once on app mount (e.g., in a root client component's effect) to restore a session after a page reload, before any protected data fetching happens.

## Implementation steps

1. Set `FRONTEND_URL` on the NestJS side and enable CORS with `credentials: true`.
2. Decide same-origin-via-proxy vs. direct cross-origin cookies — same-origin (Next.js Route Handlers proxying to NestJS) is simpler and avoids most `sameSite`/CORS edge cases; default to it unless there's a reason not to.
3. Add `middleware.ts` for the UX-level redirect.
4. Add the `apiFetch` wrapper (or equivalent in whatever data-fetching layer the app uses) so 401s trigger a silent refresh transparently instead of every call site handling it.
5. Call the refresh endpoint once on initial app load to rehydrate the access token after a hard reload.

## Security & performance notes

- Never set `sameSite: 'none'` unless the frontend and API are genuinely on different sites and there's no proxy option — it disables the main CSRF protection cookies provide, and requires `secure: true` regardless (browsers reject `none` without it).
- If proxying through Next.js Route Handlers, make sure the Route Handler doesn't accidentally log or forward the `Authorization` header/cookie to anywhere else (analytics, error trackers) — treat it with the same care as the NestJS side does.
- The access token living only in memory means every hard reload costs one refresh round-trip — acceptable, but don't call `refreshAccessToken()` more than once per mount; guard against duplicate calls from concurrent effects/renders.

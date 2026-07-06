# Refresh Tokens: Rotation, Storage, and Reuse Detection

Builds on `jwt-authentication.md` — read that first if the access-token login flow isn't set up yet.

## Why refresh tokens need more care than access tokens

An access token is short-lived and stateless — if it leaks, it's only useful for a few minutes. A refresh token is long-lived, which means it needs to be revocable (a signature-valid JWT alone can't be revoked before its expiry) and protected against reuse if it's ever stolen. The standard fix is **rotation with reuse detection**:

1. Every time a refresh token is used, issue a brand-new access + refresh pair and invalidate the old refresh token.
2. Store only a hash of the current valid refresh token per session (never the raw token — same reasoning as passwords).
3. If a refresh token is presented that matches a hash you've already rotated past (i.e., someone is replaying an old token), treat it as theft: revoke every active session for that user and force a full re-login.

## Data model

One row per active session/device, not one column on the user:

```typescript
// refresh-token.entity.ts (TypeORM example — same shape works with Prisma)
@Entity('refresh_tokens')
export class RefreshTokenEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  userId: string;

  @Column()
  tokenHash: string; // bcrypt hash of the refresh token, never the raw value

  @Column({ nullable: true })
  replacedByHash?: string; // set when rotated, used to detect reuse

  @Column()
  expiresAt: Date;

  @Column({ default: false })
  revoked: boolean;

  @CreateDateColumn()
  createdAt: Date;
}
```

Per-session rows (rather than a single token on the user) let you support "log out of all devices" and "show active sessions" later without a schema change.

## Refresh JWT strategy

```typescript
// auth/strategies/jwt-refresh.strategy.ts
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Request } from 'express';

@Injectable()
export class JwtRefreshStrategy extends PassportStrategy(Strategy, 'jwt-refresh') {
  constructor(config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromExtractors([
        (req: Request) => req?.cookies?.refresh_token ?? null,
      ]),
      ignoreExpiration: false,
      secretOrKey: config.getOrThrow<string>('JWT_REFRESH_SECRET'),
      passReqToCallback: true,
    });
  }

  async validate(req: Request, payload: { sub: string }) {
    const rawToken = req.cookies?.refresh_token;
    return { userId: payload.sub, rawToken };
  }
}
```

This requires `cookie-parser` (`app.use(cookieParser())` in `main.ts`) since the refresh token travels as an httpOnly cookie rather than a header — see `nextjs-integration.md` for why.

## AuthService: issue, rotate, and detect reuse

```typescript
// auth/auth.service.ts (continued from jwt-authentication.md)
import * as bcrypt from 'bcrypt';
import { randomUUID } from 'crypto';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import { UnauthorizedException } from '@nestjs/common';

async issueTokenPair(user: { id: string; email: string; roles: string[] }) {
  const payload = { sub: user.id, email: user.email, roles: user.roles };
  const accessToken = this.jwtService.sign(payload, {
    secret: this.config.getOrThrow('JWT_ACCESS_SECRET'),
    expiresIn: this.config.get('JWT_ACCESS_EXPIRES_IN', '15m'),
  });
  const refreshToken = this.jwtService.sign({ sub: user.id, jti: randomUUID() }, {
    secret: this.config.getOrThrow('JWT_REFRESH_SECRET'),
    expiresIn: this.config.get('JWT_REFRESH_EXPIRES_IN', '7d'),
  });

  const tokenHash = await bcrypt.hash(refreshToken, 10);
  await this.refreshTokenRepo.save({
    userId: user.id,
    tokenHash,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
  });

  return { accessToken, refreshToken };
}

async refreshTokens(userId: string, presentedToken: string) {
  const candidates = await this.refreshTokenRepo.find({
    where: { userId, revoked: false },
  });

  let matched: RefreshTokenEntity | undefined;
  for (const candidate of candidates) {
    if (await bcrypt.compare(presentedToken, candidate.tokenHash)) {
      matched = candidate;
      break;
    }
  }

  if (!matched || matched.expiresAt < new Date()) {
    // Not found among live tokens — either expired/unknown, or this hash was
    // already rotated past. Either way, don't trust it.
    await this.refreshTokenRepo.update({ userId }, { revoked: true }); // reuse -> kill all sessions
    throw new UnauthorizedException('Session expired, please log in again');
  }

  await this.refreshTokenRepo.update(matched.id, { revoked: true });

  const user = await this.usersService.findById(userId);
  return this.issueTokenPair(user); // new access + refresh pair
}

async logout(userId: string) {
  await this.refreshTokenRepo.update({ userId, revoked: false }, { revoked: true });
}
```

The "not found → revoke everything" branch is the reuse-detection guard: a legitimate client always presents the most recently issued refresh token, so a mismatch means either expiry (fine) or a stale/stolen token being replayed (not fine) — both are handled safely by forcing re-authentication.

## Controller routes

```typescript
// auth/auth.controller.ts (continued)
@Public()
@UseGuards(JwtRefreshAuthGuard) // AuthGuard('jwt-refresh')
@Post('refresh')
async refresh(@Request() req, @Res({ passthrough: true }) res: Response) {
  const { accessToken, refreshToken } = await this.authService.refreshTokens(
    req.user.userId,
    req.user.rawToken,
  );
  res.cookie('refresh_token', refreshToken, {
    httpOnly: true, secure: true, sameSite: 'strict', path: '/auth/refresh',
  });
  return { accessToken };
}

@UseGuards(JwtAuthGuard)
@Post('logout')
async logout(@Request() req, @Res({ passthrough: true }) res: Response) {
  await this.authService.logout(req.user.id);
  res.clearCookie('refresh_token', { path: '/auth/refresh' });
  return { success: true };
}
```

## Implementation steps

1. Add the `refresh_tokens` table/entity and repository.
2. Add `JwtRefreshStrategy` + a thin `JwtRefreshAuthGuard extends AuthGuard('jwt-refresh')`.
3. Replace the plain `login()` from `jwt-authentication.md` with `issueTokenPair()`.
4. Add `/auth/refresh` and `/auth/logout` routes.
5. Set up a periodic cleanup (cron or DB TTL) to delete expired/revoked rows so the table doesn't grow unbounded.

## Security & performance notes

- Scanning all of a user's live refresh-token rows with `bcrypt.compare` in a loop is fine at normal session counts (a handful per user); if you expect very high concurrent sessions per user, store a fast-lookup identifier (e.g., the `jti` claim, unhashed) alongside the hash so you can index-lookup the row before doing the bcrypt comparison.
- Set the refresh cookie's `path` to the refresh endpoint only (`/auth/refresh`), not `/` — it has no reason to be sent on every request, which shrinks the attack surface.
- Rotate on every refresh, no exceptions — skipping rotation "for convenience" reintroduces the exact replay risk this pattern exists to close.

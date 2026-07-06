# JWT Authentication with Passport

Covers: installing the auth stack, the local strategy (login), the JWT strategy (protecting routes), and wiring `AuthModule`. Pair with `refresh-tokens.md` for the refresh flow and `guards-and-decorators.md` for the global-guard + `@Public()` pattern.

## Install

```bash
npm i @nestjs/passport @nestjs/jwt passport passport-jwt passport-local bcrypt
npm i -D @types/passport-jwt @types/passport-local @types/bcrypt
```

## Environment variables

```
JWT_ACCESS_SECRET=          # long random string, distinct from refresh secret
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_SECRET=         # long random string, distinct from access secret
JWT_REFRESH_EXPIRES_IN=7d
```

Never reuse the same secret for access and refresh tokens — if one leaks, you don't want it valid for both token types.

## Local strategy (validates credentials on login)

```typescript
// auth/strategies/local.strategy.ts
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { AuthService } from '../auth.service';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private readonly authService: AuthService) {
    super({ usernameField: 'email' }); // default is 'username', override to match your DTO
  }

  async validate(email: string, password: string) {
    const user = await this.authService.validateUser(email, password);
    if (!user) {
      throw new UnauthorizedException('Invalid credentials');
    }
    return user; // becomes req.user for this request
  }
}
```

`AuthService.validateUser` looks up the user and compares the password with bcrypt (see `password-security.md`) — it never throws on "user not found" vs "wrong password" separately, to avoid leaking which one failed.

## JWT strategy (validates the access token on protected routes)

```typescript
// auth/strategies/jwt.strategy.ts
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

export interface JwtPayload {
  sub: string;      // user id
  email: string;
  roles: string[];
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: config.getOrThrow<string>('JWT_ACCESS_SECRET'),
    });
  }

  async validate(payload: JwtPayload) {
    // Whatever is returned here is attached to req.user.
    // Keep it minimal — id, email, roles. Don't re-hit the DB on every request
    // unless you need fresh data (e.g. to catch a just-revoked account).
    return { id: payload.sub, email: payload.email, roles: payload.roles };
  }
}
```

## Guards

```typescript
// auth/guards/local-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}
```

```typescript
// auth/guards/jwt-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

If you're registering `JwtAuthGuard` globally with a `@Public()` bypass instead of `@UseGuards()` per-route, see `guards-and-decorators.md` — that's the recommended default for anything beyond a handful of routes.

## AuthModule wiring

```typescript
// auth/auth.module.ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { LocalStrategy } from './strategies/local.strategy';
import { JwtStrategy } from './strategies/jwt.strategy';
import { JwtRefreshStrategy } from './strategies/jwt-refresh.strategy';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    ConfigModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.getOrThrow<string>('JWT_ACCESS_SECRET'),
        signOptions: { expiresIn: config.get('JWT_ACCESS_EXPIRES_IN', '15m') },
      }),
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, LocalStrategy, JwtStrategy, JwtRefreshStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

Note `JwtModule` here is configured with the **access** token secret/expiry. Signing refresh tokens needs the refresh secret, which `AuthService` does directly with a second `JwtService` instance or a plain `jsonwebtoken.sign()` call — see `refresh-tokens.md` for the full pattern, since refresh tokens also need to be hashed and stored, not just signed.

## AuthService core methods

```typescript
// auth/auth.service.ts (login/validate portion — refreshTokens()/logout() live in refresh-tokens.md)
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import * as bcrypt from 'bcrypt';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService,
  ) {}

  async validateUser(email: string, password: string) {
    const user = await this.usersService.findByEmail(email);
    if (!user) return null;
    const passwordMatches = await bcrypt.compare(password, user.passwordHash);
    if (!passwordMatches) return null;
    const { passwordHash, ...safeUser } = user;
    return safeUser;
  }

  async login(user: { id: string; email: string; roles: string[] }) {
    const payload = { sub: user.id, email: user.email, roles: user.roles };
    const accessToken = this.jwtService.sign(payload);
    // refreshToken issuance + storage — see refresh-tokens.md
    return { accessToken, user };
  }
}
```

## AuthController

```typescript
// auth/auth.controller.ts
import { Controller, Post, Request, UseGuards, Res } from '@nestjs/common';
import type { Response } from 'express';
import { LocalAuthGuard } from './guards/local-auth.guard';
import { AuthService } from './auth.service';
import { Public } from './decorators/public.decorator';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Public()
  @UseGuards(LocalAuthGuard)
  @Post('login')
  async login(@Request() req, @Res({ passthrough: true }) res: Response) {
    const { accessToken, refreshToken, user } = await this.authService.login(req.user);
    res.cookie('refresh_token', refreshToken, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      path: '/auth/refresh',
      maxAge: 7 * 24 * 60 * 60 * 1000,
    });
    return { accessToken, user };
  }
}
```

`/auth/refresh` and `/auth/logout` routes belong here too — they're covered in `refresh-tokens.md` since they depend on the rotation/storage logic defined there.

## Implementation steps

1. Install packages and set env vars above.
2. Add `UsersModule` (or wherever user lookup lives) as a dependency of `AuthModule`.
3. Create the two strategies, then the two guards, then `@Public()` decorator (`guards-and-decorators.md`).
4. Implement `AuthService.validateUser` and `login`.
5. Wire `AuthModule` as shown, import it in `AppModule`.
6. Add the login route; confirm a valid login returns an access token and sets the refresh cookie.
7. Protect a test route with `JwtAuthGuard` (or the global guard pattern) and confirm it 401s without a token and 200s with one.

## Security & performance notes

- `jwtService.sign()` without an explicit `expiresIn` on the call falls back to the module-level `signOptions` — don't set expiry in two places, it invites drift.
- Don't put anything sensitive (password hash, internal flags) in the JWT payload — it's base64, not encrypted, and readable by anyone holding the token.
- Verifying a JWT is CPU-light; it doesn't need a DB hit per request unless you need to check for revocation/ban status in real time, in which case consider a short-lived in-memory/Redis cache of revoked user IDs rather than a full user fetch on every request.

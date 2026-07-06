# Global Guards, @Public(), and Custom Decorators

Covers making auth apply by default across an app (fail-closed) instead of remembering `@UseGuards()` on every new route, plus the decorators that make controllers read cleanly.

## Why global guard + opt-out beats per-route @UseGuards()

`@UseGuards(JwtAuthGuard)` on every protected controller works, but it's opt-in: a new route added by anyone on the team is public by default unless they remember to add the guard. Registering the guard globally and marking the *few* public routes instead (login, register, health check, webhooks) is fail-closed — the default is "requires auth," which is the safer default for anything beyond a small prototype.

## @Public() decorator

```typescript
// auth/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

## Global JwtAuthGuard that respects @Public()

```typescript
// auth/guards/jwt-auth.guard.ts
import { ExecutionContext, Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private readonly reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;
    return super.canActivate(context);
  }
}
```

## Register it globally

```typescript
// app.module.ts
import { APP_GUARD } from '@nestjs/core';
import { JwtAuthGuard } from './auth/guards/jwt-auth.guard';

@Module({
  // ...
  providers: [
    { provide: APP_GUARD, useClass: JwtAuthGuard },
  ],
})
export class AppModule {}
```

Now every route requires a valid access token unless explicitly marked:

```typescript
@Public()
@Post('login')
login() { /* ... */ }

@Get('profile') // protected by default, no decorator needed
getProfile(@Request() req) {
  return req.user;
}
```

## @CurrentUser() decorator

Cleaner than repeating `@Request() req` and reaching into `req.user` in every handler:

```typescript
// auth/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export interface AuthenticatedUser {
  id: string;
  email: string;
  roles: string[];
}

export const CurrentUser = createParamDecorator(
  (data: keyof AuthenticatedUser | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user: AuthenticatedUser = request.user;
    return data ? user?.[data] : user;
  },
);
```

Usage:

```typescript
@Get('profile')
getProfile(@CurrentUser() user: AuthenticatedUser) {
  return user;
}

@Get('my-email')
getEmail(@CurrentUser('email') email: string) {
  return { email };
}
```

## Implementation steps

1. Add `public.decorator.ts` and update `JwtAuthGuard` to check it via `Reflector`, as shown.
2. Register the guard as `APP_GUARD` in `AppModule` (must be a provider in the *root* module, or wherever guards are meant to apply app-wide).
3. Mark `login`, `register`, `refresh`, and any health/webhook routes with `@Public()`.
4. Add `@CurrentUser()` and start using it instead of `@Request() req` + manual `.user` access in new handlers.
5. Write one quick test per changed route: public routes return 200 with no token, everything else returns 401 with no token.

## Security & performance notes

- `Reflector.getAllAndOverride` checks the handler first, then the controller class — so a `@Public()` on a whole controller can be overridden by a stricter setting on one method if needed, but there's no built-in "require auth" override decorator by default; if you need that, add an explicit `@Auth()` marker and check for it the same way.
- Don't forget `context.getType()` checks if the same guard also runs against GraphQL resolvers or WebSocket gateways — `context.switchToHttp()` throws in those contexts. If the app is HTTP-only, this isn't a concern.
- A global guard runs on *every* request including static assets served by Nest, if any are configured — scope the guard's route matching or exclude those explicitly if that becomes an issue.

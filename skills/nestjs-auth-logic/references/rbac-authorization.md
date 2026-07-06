# Role-Based Authorization

This is authorization (what an authenticated user is *allowed* to do), layered on top of authentication (confirming *who* they are — see `jwt-authentication.md`). `RolesGuard` always runs after the auth guard has already populated `req.user`.

## Roles decorator

```typescript
// auth/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export enum Role {
  User = 'user',
  Editor = 'editor',
  Admin = 'admin',
}

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

## RolesGuard

```typescript
// auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY, Role } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles || requiredRoles.length === 0) {
      return true; // no @Roles() on this route -> auth alone is enough
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user?.roles?.includes(role));
  }
}
```

## Usage on a controller

```typescript
import { Controller, Get, Delete, Param, UseGuards } from '@nestjs/common';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Roles, Role } from '../auth/decorators/roles.decorator';

@Controller('users')
export class UsersController {
  // JwtAuthGuard already applies globally (guards-and-decorators.md) — only
  // add RolesGuard here, it composes with the global guard automatically.

  @UseGuards(RolesGuard)
  @Roles(Role.Admin)
  @Delete(':id')
  remove(@Param('id') id: string) {
    // only Admins reach this point
  }

  @Get('me')
  getOwnProfile(@CurrentUser() user: AuthenticatedUser) {
    // any authenticated user, no @Roles() needed
  }
}
```

If `JwtAuthGuard` is *not* global in your app, add `RolesGuard` after it explicitly: `@UseGuards(JwtAuthGuard, RolesGuard)` — order matters, `req.user` must exist before `RolesGuard` runs.

## When roles aren't enough: resource-level permissions

`@Roles(Role.Admin)` answers "what kind of user is this," not "does this specific user own this specific resource." For checks like "can this user edit *this* post" (not just "is this user an Editor"), a role check isn't sufficient — you need an ownership/policy check inside the service or handler:

```typescript
@Get(':id')
async findOne(@Param('id') id: string, @CurrentUser() user: AuthenticatedUser) {
  const post = await this.postsService.findById(id);
  if (post.authorId !== user.id && !user.roles.includes(Role.Admin)) {
    throw new ForbiddenException();
  }
  return post;
}
```

For apps with many resources and complex permission matrices (not just "own vs not own"), consider a policy library like CASL rather than hand-rolling checks in every handler — that's a bigger architectural decision than this skill's scope, but worth knowing the boundary: `RolesGuard` handles coarse role checks, ownership/policy checks handle the rest.

## Implementation steps

1. Define the `Role` enum to match the app's actual roles (adjust names/count as needed — don't over-generalize to roles the product doesn't have).
2. Add `@Roles()` decorator and `RolesGuard`.
3. Apply `@Roles(...)` + `@UseGuards(RolesGuard)` only on routes that need role restrictions — routes with no `@Roles()` just require authentication (via the global guard), nothing more.
4. For per-resource checks (ownership), add explicit checks in the service/handler — `RolesGuard` doesn't know about individual resources.

## Security & performance notes

- Store roles in the JWT payload at login time (`payload.roles`) so `RolesGuard` doesn't need a DB hit per request. Trade-off: if a role is revoked mid-session, it won't take effect until the access token expires (10–15 min if you followed `jwt-authentication.md`) — acceptable for most apps, but call this out if the user needs instant role revocation, since that requires either a DB check per request or a short-lived revocation cache.
- `some()` in `RolesGuard` means "any of these roles is sufficient" — if a route genuinely needs *all* of several roles, swap to `.every()`, but that's an unusual requirement worth double-checking with the user before assuming it.

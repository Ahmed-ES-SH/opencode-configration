# Password Hashing & Reset Flows

## Hashing on registration

```typescript
import * as bcrypt from 'bcrypt';

const SALT_ROUNDS = 12; // 10-12 is the standard range; higher = slower = more resistant to brute force

async function hashPassword(plain: string): Promise<string> {
  return bcrypt.hash(plain, SALT_ROUNDS);
}
```

`bcrypt.hash` generates its own random salt internally and embeds it in the output string — there's no separate salt column to manage. Never hash without a salt, and never reuse a fixed salt across users.

## Comparing on login

```typescript
const isMatch = await bcrypt.compare(plainPassword, user.passwordHash);
```

Always compare against the stored hash with `bcrypt.compare` — never decrypt (bcrypt is one-way, there's nothing to decrypt) and never compare with `===` against a re-hashed value using a different salt.

## DTO-level validation

```typescript
// register.dto.ts
import { IsEmail, IsString, MinLength, Matches } from 'class-validator';

export class RegisterDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  @Matches(/(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'Password must contain at least one uppercase letter, one lowercase letter, and one number',
  })
  password: string;
}
```

Enforce this with `ValidationPipe` (`app.useGlobalPipes(new ValidationPipe({ whitelist: true }))` in `main.ts`) so invalid payloads never reach the service layer.

## Password reset flow

The reset token needs the same treatment as a refresh token: random, hashed at rest, short-lived, single-use.

```typescript
// password-reset.service.ts
import { randomBytes, createHash } from 'crypto';

async requestReset(email: string) {
  const user = await this.usersService.findByEmail(email);
  // Always return the same generic response whether or not the user exists —
  // do the token creation/email send conditionally, but don't let the response
  // shape or timing reveal which case happened.
  if (user) {
    const rawToken = randomBytes(32).toString('hex');
    const tokenHash = createHash('sha256').update(rawToken).digest('hex');
    await this.resetTokenRepo.save({
      userId: user.id,
      tokenHash,
      expiresAt: new Date(Date.now() + 30 * 60 * 1000), // 30 min
      used: false,
    });
    await this.mailService.sendResetEmail(user.email, rawToken);
  }
  return { message: 'If that email exists, a reset link has been sent.' };
}

async resetPassword(rawToken: string, newPassword: string) {
  const tokenHash = createHash('sha256').update(rawToken).digest('hex');
  const record = await this.resetTokenRepo.findOne({ where: { tokenHash, used: false } });
  if (!record || record.expiresAt < new Date()) {
    throw new BadRequestException('Reset link is invalid or has expired');
  }
  const passwordHash = await bcrypt.hash(newPassword, 12);
  await this.usersService.updatePassword(record.userId, passwordHash);
  await this.resetTokenRepo.update(record.id, { used: true });
  // Also revoke all existing refresh tokens for this user — a password reset
  // should end every other active session (see refresh-tokens.md logout()).
  await this.authService.logout(record.userId);
}
```

Plain SHA-256 (not bcrypt) is fine for the reset token itself since it's a high-entropy random value, not a low-entropy user-chosen password — the attack bcrypt defends against (fast brute-forcing of guessable inputs) doesn't apply to a 32-byte random token. Bcrypt is still the right choice for the actual password.

## Brute-force protection

```typescript
// main.ts or auth module
import { ThrottlerModule } from '@nestjs/throttler';

ThrottlerModule.forRoot([{ ttl: 60_000, limit: 5 }]); // 5 requests/min default

// then on the sensitive routes specifically:
import { Throttle } from '@nestjs/throttler';

@Throttle({ default: { limit: 5, ttl: 60_000 } })
@Public()
@Post('login')
login() { /* ... */ }
```

Apply a tighter, dedicated limit on `login`, `register`, `refresh`, and `request-reset` — these are exactly the endpoints credential-stuffing and brute-force tools target.

## Implementation steps

1. Add `bcrypt` and `class-validator`/`class-transformer` if not already present; enable `ValidationPipe` globally.
2. Hash on registration, compare on login (never store or log plaintext passwords — check logging interceptors don't accidentally dump request bodies).
3. Add a `password_reset_tokens` table matching the refresh-token pattern in `refresh-tokens.md`.
4. Wire `@nestjs/throttler` and apply tighter limits to auth endpoints specifically.
5. Confirm the reset flow revokes existing sessions after a successful reset.

## Security & performance notes

- 12 salt rounds is a reasonable default in 2026 hardware terms; if login latency becomes noticeable under load, drop to 10 before considering anything more drastic like caching hashes (never cache password comparisons).
- Never log the raw password, the raw reset token, or the raw refresh token — log the user id and event type ("password reset requested for user X") instead.
- The generic "if that email exists" response must apply to both the response body *and* the response timing — if the DB lookup for a non-existent user returns near-instantly while an existing user triggers an email send that takes longer, that timing difference itself leaks the answer. Awaiting the same shape of work (or not awaiting the email send inline) avoids this.

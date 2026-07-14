# Email Flow Checklist — Account Verification & Password Reset

These two flows are consistently among the weakest parts of real-world auth implementations, because they're often built quickly after the "main" login/JWT work is done and get less scrutiny. Treat this file as mandatory reading for every review, not an optional add-on.

## A. Account verification email

**Data model**
- A `emailVerified` (boolean) or `emailVerifiedAt` (nullable timestamp) column exists on the user and defaults to false/null on registration. Fail: no such field at all when the app's registration flow implies verification is expected (e.g. it sends a "verify your email" message but has no field tracking whether that happened).

**Token handling**
- The verification token is a **high-entropy random value** (`crypto.randomBytes(32)` or similar), not a predictable/sequential id and not just the user's own database id encoded.
- The token is **stored hashed** (SHA-256 is adequate — same reasoning as reset tokens: it's a high-entropy random value, not a low-entropy guessable secret, so bcrypt's slowness isn't needed here) in its own table/column, not compared in plaintext.
- The token has an **expiry** (24–72 hours is a common reference range for verification, shorter if the product wants tighter security) and is **single-use** — consumed/deleted/flagged on success, not reusable afterward.
- Requesting a **new verification email invalidates the previous outstanding token** rather than leaving multiple valid tokens active for the same user.

**Link design — the GET-burns-the-token problem**
- If verification is implemented as a bare `GET /auth/verify?token=...` that performs the state change directly, flag this as a finding: corporate email scanners, antivirus link-prefetchers, and some email clients automatically follow links in incoming mail, which silently consumes a one-shot GET token before the real user ever clicks it, locking them out with no clear explanation.
- The safer pattern: the emailed link points to a **frontend page** that reads the token from the URL, then the frontend makes a **POST** request to actually consume it and flip `emailVerified`. Recommend this pattern in the fix if the current implementation performs the state-changing action directly on the GET.
- If a bare-GET implementation genuinely can't change (e.g. constraint out of the person's control), a lesser mitigation is making the *first* GET set verified=true but returning the same success response on any subsequent GET with the same token within its expiry window, rather than erroring — reduces lockout risk without fully closing the underlying design issue. Mention this as a fallback, not the primary recommendation.

**Enumeration & rate limiting**
- The resend-verification endpoint returns a **generic response regardless of whether the email exists or is already verified** ("If an account exists for this email, a verification link has been sent") — same principle as password reset. Fail: a response or status code that differs based on account existence.
- Resend-verification is **rate-limited** — an unbounded resend endpoint is both an abuse vector (mail-bombing an inbox) and an enumeration vector (timing/response differences under load).

**Sending & content**
- The email send happens **without blocking the registration response** — either fire-and-forget with error logging, or queued. A mail-provider outage should not turn a normal registration into a failed request. If the current code `await`s the send inline and lets a thrown error propagate to a 500, flag it.
- The email **never contains the user's password** (plaintext or otherwise) — only ever a verification link/token. A "here is your account, here is your password" welcome email is a real anti-pattern still worth explicitly checking for, not just a hypothetical.
- If the app supports **changing email address** post-registration, confirm the new address goes through its own verification (and the account isn't treated as verified for the new address until that completes) — silently trusting an email change with no re-verification is a finding.

**Enforcement**
- If the product actually requires a verified email before login or before certain actions, confirm that's **enforced server-side** (a guard, or a check in the relevant service method) — not just a flag that's set but never checked anywhere. A `@RequireVerified()`-style decorator that exists but isn't applied to any route is functionally the same as not having verification at all; flag it as a finding, not a pass.

## B. Password reset email

This mirrors the pattern documented in `nestjs-auth-logic`'s `password-security.md` — use that as the reference shape when writing the fix snippet.

**Request-reset endpoint**
- Returns the **same generic response** whether or not the email exists — check both the **response body** and, as best as can be judged from the code, the **timing**: if the non-existent-user path returns near-instantly while the existing-user path does a DB write + waits on a mail send inline, that timing gap itself leaks which case occurred. The fix is to not let the response depend on an awaited mail send, or to await equivalent-cost work either way.
- Is **rate-limited** — same reasoning as login/register.

**Token handling**
- Reset token is a **high-entropy random value**, hashed at rest (SHA-256 is fine, per the same reasoning as verification tokens), with a **short expiry** (~15–30 minutes is the reference range — reset tokens should be shorter-lived than verification tokens, since the reset window is a more sensitive operation).
- Token is **single-use** — flagged/deleted after a successful reset.
- Requesting a new reset **invalidates any previously issued outstanding reset token** for that user, so an attacker can't collect and race multiple valid tokens.

**Reset (consume) endpoint**
- The **new password is validated with the same strength rules** as registration (`class-validator` DTO reused or mirrored) — a common gap is a reset endpoint that accepts any string because the strength decorators only got added to `RegisterDto`.
- A successful reset **revokes all existing refresh tokens/sessions** for that user — a password reset should end every other active session, since the whole point is recovering from a suspected compromise.
- The reset email **never contains a new password directly** — it must only ever contain a link/token to let the user set their own new password. Auto-generating and emailing a new plaintext password is a **Critical** finding if found (plaintext password sent over email, often also logged by mail infrastructure along the way).

**Redirect safety**
- If any part of the flow accepts a `redirectUrl` / `returnTo` / similar query parameter (e.g. "redirect back to the page you were on after reset"), confirm it's checked against an **allow-list of known frontend origins/paths** before being used — an unchecked redirect target is an open-redirect vector, useful for phishing since it rides on a legitimate-looking reset link.

## C. General mail hygiene (applies to both flows, and any other transactional email)

- **Mail provider credentials/API keys come from environment variables**, never hardcoded, never logged.
- **Raw tokens and reset/verification links are never logged** in plaintext — check logging interceptors/middleware don't accidentally dump full request bodies or the mail payload itself. Log the user id and event type ("password reset requested for user X"), not the secret.
- **User-controlled values interpolated into email HTML are escaped** by the templating engine — an unescaped name field (e.g. `Hi ${user.name},`) that a user can set to arbitrary HTML is an injection risk in the email content, primarily useful for phishing amplification against whoever receives forwarded copies. Lower severity than app-side XSS but still worth flagging if the templating approach doesn't escape by default.
- **Mail send failures are handled gracefully** — a thrown exception from the mail provider inside a registration/reset/verification handler shouldn't produce an ungraceful 500 that also potentially reveals stack traces; catch, log, and keep the user-facing response as designed (generic, for reset/verification-resend specifically).
- SPF/DKIM/DMARC configuration on the sending domain is out of application-code scope, but if it's easy to tell the domain has no visible mail-auth setup and it's relevant context, a one-line infra note in the report is appropriate — don't turn it into a full finding since it's not something fixed in this codebase.

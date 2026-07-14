---
name: nestjs-auth-review
description: Audits an EXISTING NestJS auth implementation for security/correctness issues and writes a findings report — reviews code, never edits application source files. Use whenever the user asks to review, audit, check, scan, or find bugs/vulnerabilities in NestJS auth logic — login, JWT/refresh-token flows, guards, RBAC, password hashing, or (always in scope) account-verification and password-reset email flows. Trigger on phrasing like "check my auth for security holes", "audit my login system", "review this auth module", "is my refresh token flow secure", "check my verify-email and reset-password flows", "does my auth have vulnerabilities", or any request to inspect auth code someone already wrote. Counterpart to `nestjs-auth-logic` — that one builds new auth code, this one reviews existing auth code and reports back; never implement or fix code directly here, only produce the report.
---

# NestJS Auth Logic Reviewer

This skill reviews an existing NestJS authentication/authorization implementation against a fixed security checklist and writes the findings to a Markdown report. It is a **read-only audit**, not an implementation task — the deliverable is the report file, not edited application code.

If the person wants both a review *and* fixes applied, do the review first, produce the report, then treat applying fixes as a separate, explicitly-confirmed follow-up (ideally using `nestjs-auth-logic` for the actual implementation patterns). Don't silently start rewriting auth files just because a review turned up problems.

## Why this is a separate skill from `nestjs-auth-logic`

`nestjs-auth-logic` is a reference for building auth correctly the first time. This skill is for the much more common real-world situation: auth code already exists — maybe written months ago, maybe by someone else, maybe scaffolded quickly under deadline — and the person wants to know what's actually wrong with it before it ships or before a security review. The two skills share the same underlying idea of "correct," but reviewing existing code is a different task from writing new code: it means locating scattered files, checking each one against a checklist, being honest about severity, and writing that down clearly — not producing new code.

## Step 1 — Locate what to review

Don't ask the person to paste every file if you can find them yourself:

1. If specific files were already shared/uploaded/pasted in the conversation, review those.
2. Otherwise, search the project for the relevant pieces. Auth logic is usually scattered beyond one folder — look for all of these, not just `src/auth/`:
   - The auth module itself: `**/auth/**` (controller, service, module, strategies, guards, decorators)
   - The user entity/model (look for `emailVerified`, `isVerified`, `passwordHash`, roles/permissions columns)
   - Password/token reset and verification tables or entities (`password_reset*`, `verification*`, `refresh_token*`)
   - The mail/notification layer: `**/mail/**`, `**/email/**`, `**/notifications/**`, anywhere `sendMail`, `sendVerificationEmail`, `sendResetPasswordEmail`, or similar is defined or called
   - DTOs used by register/login/verify/reset endpoints (`class-validator` decorators matter for the checklist)
   - `main.ts` (CORS, cookie-parser, global pipes, throttler setup) and `.env`/`.env.example` (secret naming, not values)
3. If a plausible search still turns up nothing (e.g. auth genuinely lives somewhere non-standard, or there's no code to search — just pasted snippets), ask once, briefly, rather than guessing.

Read every file found before judging anything — don't flag an issue based on a filename or a guess about what a function probably does.

## Step 2 — Run the checklist

Two reference files hold the actual audit criteria. Read both in full before starting — don't sample a few items and call it done:

| File | Covers |
|---|---|
| `references/security-checklist.md` | Core auth: token issuance/storage, refresh rotation, guards, RBAC, password hashing, session auth, Next.js wiring |
| `references/email-flows-checklist.md` | Account-verification email and password-reset email flows specifically — token handling, timing/enumeration, link design, mail hygiene |

**The email-flows checklist is always in scope**, even if the person's request only mentions "auth" generally — verification and reset emails are two of the most commonly under-secured parts of an auth system (enumeration leaks, unbounded tokens, tokens burned by email-scanner bots), so don't skip that file just because it wasn't named explicitly.

For each checklist item:
- If the code satisfies it, don't write an issue for it — the report lists problems, not a wall of green checkmarks. A short "reviewed, no issues found" line per category in the summary is enough.
- If it doesn't, or if the relevant code doesn't exist at all (e.g. no rate limiting anywhere), that's a finding.
- If a whole category doesn't apply (e.g. the app uses sessions, so JWT-rotation items don't apply, or the app genuinely has no email-verification concept and that's a deliberate product choice, not an oversight), mark it "Not applicable" with a one-line reason rather than silently omitting it — the person should be able to see that you considered it.

## Step 3 — Rate severity honestly

Use this rubric consistently — don't inflate minor style issues to "Critical," and don't soften a real vulnerability to "Low" because the fix looks like a small diff:

- **Critical** — directly exploitable right now: plaintext/reversibly-encrypted passwords, hardcoded or committed secrets, tokens readable via XSS (e.g. JWT in `localStorage` with no mitigating factors), a sensitive route with no guard at all, auth bypassable by a crafted request.
- **High** — a realistic attacker path exists even if it takes some effort: no rate limiting on login/register/reset/verify, no refresh-token rotation or reuse detection, sessions not revoked after a password reset, missing/weak cookie flags (`httpOnly`, `secure`, `sameSite`), permissive CORS with `credentials: true`.
- **Medium** — weakens defense-in-depth or leaks information without being immediately catastrophic: email enumeration via distinguishable responses or timing, verbose error messages revealing which credential was wrong, an unbounded resend-verification endpoint.
- **Low** — hygiene and best-practice deviations unlikely to be directly exploitable on their own: slightly low bcrypt cost factor, missing cleanup job for expired tokens, inconsistent logging, code duplication across near-identical guards.

Every finding needs a concrete, specific recommended fix — not "improve security here." Reference the actual pattern from the checklist/relevant `nestjs-auth-logic` reference file, and include a short code snippet when the fix isn't obvious from prose alone.

## Step 4 — Write the report

The report is always a single Markdown file, always saved to a folder literally named `reports` (create it if it doesn't exist), at the project root if one is evident, otherwise the current working directory.

**Filename:** `reports/auth-review-<YYYY-MM-DD>.md` (e.g. `reports/auth-review-2026-07-07.md`). If a report with that name already exists from the same day (re-running the review), append `-2`, `-3`, etc. rather than overwriting a prior run silently.

Follow the structure in `assets/report-template.md` exactly — section order and headings matter because the person may be scanning multiple reports over time and expects them in the same shape. Populate every section; don't drop the "Reviewed, no issues found" or "Not applicable" categories even when short.

After writing the file, use `present_files` (or the equivalent file-sharing mechanism available) so the person can open it directly — don't just paste the whole report back into the chat as well, that defeats the point of a saved file they can revisit.

## What NOT to do

- Don't edit the reviewed source files. If a fix is genuinely one line and the person clearly wants it applied immediately, confirm that's what they want before touching anything — the default behavior of this skill is report-only.
- Don't pad the report with restated code the person already has — quote only the minimal snippet needed to locate and fix an issue.
- Don't guess at severity or applicability to make the report look more thorough — an honest "no issues found in this category" is a valid and useful result.

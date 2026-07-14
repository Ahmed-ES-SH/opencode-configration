# NestJS Auth Logic — Security Review Report

**Project:** {{project name or directory}}
**Reviewed on:** {{YYYY-MM-DD}}
**Files reviewed:** {{list of file paths actually read}}

## Summary

- **Total findings:** {{N}} (Critical: {{n}}, High: {{n}}, Medium: {{n}}, Low: {{n}})
- **One-paragraph overview** of the app's current auth approach (JWT vs sessions, whether email verification/reset are implemented at all) and the overall state — one or two sentences is enough, save detail for the findings.

## Checklist coverage

| Category | Status |
|---|---|
| Secrets & configuration | ✅ / ⚠️ (N findings) / ➖ N/A — reason |
| Token issuance & lifetime | ✅ / ⚠️ / ➖ |
| Refresh token rotation & reuse detection | ✅ / ⚠️ / ➖ |
| Guards & route protection | ✅ / ⚠️ / ➖ |
| RBAC / authorization | ✅ / ⚠️ / ➖ |
| Password handling | ✅ / ⚠️ / ➖ |
| Session-based auth | ✅ / ⚠️ / ➖ |
| Cookie & transport handling | ✅ / ⚠️ / ➖ |
| Next.js ↔ NestJS wiring | ✅ / ⚠️ / ➖ |
| Account verification email | ✅ / ⚠️ / ➖ |
| Password reset email | ✅ / ⚠️ / ➖ |
| General mail hygiene | ✅ / ⚠️ / ➖ |

## Findings

List every finding, most severe first. Use this exact shape per finding:

### [{{SEVERITY}}] {{Short issue title}}

- **Category:** {{one of the checklist categories above}}
- **Location:** `{{file path}}{{:line if known}}`
- **Issue:** {{what's wrong, described plainly}}
- **Risk:** {{why it matters — what an attacker or a real scenario would do with this}}
- **Recommended fix:**

  {{concrete steps, plus a short code snippet if the fix isn't obvious from prose alone}}

  ```typescript
  // example fix snippet, only if it clarifies something prose can't
  ```

<!-- repeat the ### block above for every finding -->

## Not applicable

- **{{Category}}** — {{one-line reason it doesn't apply to this app}}

<!-- omit this whole section if nothing was marked N/A -->

## Notes

Anything worth mentioning that isn't a finding against this codebase specifically — e.g. an infra-level observation (SPF/DKIM), a tradeoff the team should be aware of but that isn't wrong per se, or a suggestion for a follow-up review once a planned change lands.

## Context

`identity/registration-and-login` already established the DDD layering
(ADR 0002) and the email+password/JWT-bearer mechanism (ADR 0004) this
change builds directly on top of. ADR 0004 explicitly left password reset
as an open follow-up ("will need the SMTP capability... plus its own
token/expiry mechanics") and separately flagged that no token
revocation/refresh strategy existed. This change resolves the first and
partially resolves the second. See proposal.md - Why.

## Goals / Non-Goals

**Goals:**
- Self-service password change and email-based forgot/reset, following the
  same domain/application/infrastructure/interface layering as the existing
  Identity module.
- Make the first real implementation of the `EmailSender` port ADR 0002
  anticipated (a `src/notifications/` module), rather than inventing
  ad hoc SMTP code inside Identity.
- Close the specific security gap password change/reset creates on its
  own: without some invalidation, a password change or reset is pointless
  against a leaked JWT, since the old token would keep working until it
  expired.

**Non-Goals:**
- General "log out of all sessions" as a user-facing feature — the
  invalidation here is a side effect of a password change, not its own
  capability.
- Rate limiting or abuse protection on `forgot-password` — flagged as a
  follow-up, not built speculatively.
- Full token revocation/refresh-token infrastructure — `tokenVersion` (see
  Decisions) solves the password-change case specifically; a general
  revocation list or refresh-token rotation is still open, same as ADR
  0004 already noted.

## Decisions

### Reset code: opaque random token, only its hash stored
32 random bytes (`node:crypto.randomBytes`), hex-encoded, emailed to the
User as-is. Only a SHA-256 hash of it is persisted
(`PasswordResetToken.tokenHash`, unique) — the same reasoning as password
hashing: a database read can never recover a usable code. SHA-256 (fast,
unsalted) is deliberately not bcrypt here: the code is already
high-entropy (32 random bytes, not a human-guessable secret), so a slow
hash defends against nothing extra and would only add latency.

A `PasswordResetToken` row also carries `expiresAt` and `usedAt`
(nullable). Default TTL is 60 minutes
(`PASSWORD_RESET_TOKEN_TTL_MINUTES`, configurable) — long enough to find
and read the email, short enough that a stale, unused code is low-value to
an attacker. Requesting a new reset deletes any earlier unused codes for
that User first, so only the most recent one can ever succeed.

### Session invalidation: a `tokenVersion` counter, not a timestamp comparison
**This is the one real correctness bug found while building this change,
worth recording:** the first implementation compared the JWT's standard
`iat` (issued-at, integer seconds) claim against a `passwordChangedAt`
timestamp on `User`, rejecting a token if `iat` was earlier. That is
broken by JWT's second-level precision — logging in and changing the
password inside the same wall-clock second (trivially reproducible, not
just a lab edge case) left `iat` and `passwordChangedAt` truncating to the
same second, and the old token kept working.

Fixed by replacing the timestamp comparison with a monotonic integer
`tokenVersion` on `User` (default `0`), incremented atomically in the same
database write that sets the new password hash. Every issued JWT embeds
the `tokenVersion` it was issued under; every authenticated request
compares the token's `tokenVersion` to the User's current one and rejects
on any mismatch. An integer equality check has no timing window to race,
unlike a "before/after" comparison on a clock both sides read
independently.

Change-password and reset-password both issue a fresh token carrying the
*new* `tokenVersion` in their response, so the request that performed the
change keeps working — only tokens issued strictly before it stop being
accepted.

### Forgot-password response never depends on whether the email exists
Enforced at the use-case level, not just the controller: looking up an
unknown email returns silently (no error, no email sent) rather than
throwing, so there is no error-vs-success branch for a controller to leak
through timing or response shape. The controller always returns the same
generic message regardless of what the use case did internally.

### Email content: German, plain text, sent through the new Notifications module
`src/notifications/` provides a generic `EmailSender.send({to, subject,
text, html?})` port with a `nodemailer` adapter reading `MAIL_HOST` /
`MAIL_PORT` / `MAIL_USERNAME` / `MAIL_PWD` (already present in `.env` per
ADR 0002, just not previously consumed by any code). The port stays
content-agnostic; Identity's `ForgotPasswordUseCase` composes the actual
German-language email text, per the project's German-UI/English-code
language policy (user-facing content follows the German UI, not the
English domain/code identifiers).

## Risks / Trade-offs

- **[Risk]** A `PasswordResetToken` row is deleted (superseded codes) or
  marked used, but never garbage-collected once expired-and-unused. →
  **Mitigation**: none yet — the table stays small (one live row per user
  who's mid-reset) and this is cheap to add later (a scheduled delete of
  `expiresAt < now()`) if it ever matters.
- **[Trade-off]** `tokenVersion` invalidates *every* other session on
  password change, not just "sessions other than this device." There's no
  concept of a per-device session to spare one selectively. Accepted:
  matches most products' behavior for a password change, and building
  per-device session tracking wasn't asked for and isn't needed by
  anything else in the system yet.
- **[Risk]** Forgot-password has no rate limiting. A script could spam the
  same email's inbox with reset codes without ever confirming the email
  exists (that part is protected), which is still a mild nuisance/cost
  (SMTP sends) against a known address. → **Mitigation**: none yet; flagged
  as a follow-up, not built speculatively without a concrete abuse case.

## Migration Plan

Additive: new `PasswordResetToken` table, and `User` gains `tokenVersion
Int @default(0)` (backfills existing rows to `0`, which is exactly correct
— every already-issued token was signed without a `tokenVersion` claim
change relative to that baseline). No rollback concerns beyond the
standard `prisma migrate` down path.

## Open Questions

- Whether `forgot-password` needs rate limiting before real users hit it —
  revisit if it becomes a real target, not speculatively now.

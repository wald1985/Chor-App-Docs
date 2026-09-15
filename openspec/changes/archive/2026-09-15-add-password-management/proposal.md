## Why

`identity/registration-and-login` (see its spec) covers creating an account
and signing in, but gave a User no way to change their password once
signed in, or recover access after forgetting it. ADR 0004 explicitly
flagged password reset as an open follow-up needing its own token/expiry
mechanics. This change closes that gap and, since a self-service password
change is unauthenticated-adjacent by nature (a stolen JWT could otherwise
keep working forever), also fixes a real gap the reset flow surfaced: JWTs
issued before a password change had no way to be invalidated.

## What Changes

- `POST /auth/change-password` (authenticated): verifies the caller's
  current password, sets the new one, and returns a fresh access token.
- `POST /auth/forgot-password`: accepts an email, always responds with the
  same generic message regardless of whether the email is registered, and —
  only if it is — emails a single-use, time-limited reset code via the
  Notifications capability (SMTP, ADR 0002).
- `POST /auth/reset-password`: accepts the code and a new password;
  verifies it (unexpired, unused, matches a real request), sets the new
  password, and returns a fresh access token plus the account's Community
  memberships (same shape as login).
- **Cross-cutting addition to the auth mechanism (ADR 0004):** every `User`
  now carries a `tokenVersion` counter. It's bumped on every successful
  password change or reset, embedded in every newly issued JWT, and checked
  on every authenticated request — so changing or resetting a password
  immediately invalidates every other JWT issued for that account, not just
  the one used to make the request. Recorded as an amendment to
  `decisions/0004-auth-mechanism.md` rather than its own ADR, since it
  changes how the existing JWT mechanism validates a token, not a new
  mechanism.

**Not in scope:**
- A user-facing "log out of all other sessions" action — the invalidation
  above is a side effect of changing a password, not its own capability.
- Rate limiting `forgot-password` against abuse/spam — noted as a follow-up,
  not built speculatively here.

## Capabilities

### New Capabilities
- `identity/password-management`: authenticated password change, and
  unauthenticated forgot/reset password by email.

### Modified Capabilities
(none — registration and login's own requirements are unchanged; the
token-invalidation side effect is specified under the new capability, since
it's only observable through change-password/reset-password)

## Impact

- **chor-app-server**: new use cases and endpoints in `src/identity/`; new
  `src/notifications/` module (`EmailSender` port + nodemailer adapter) —
  the first real implementation of the port ADR 0002 anticipated; new
  `PasswordResetToken` Prisma model; `User` gains `tokenVersion`
  (replacing nothing — this is the first session-invalidation mechanism
  built).
- **chor-app-client**: no code yet, but any future session handling must
  expect a call to `/auth/change-password` or a successful
  `/auth/reset-password` to return a *new* token that replaces the one the
  client was using — the old one stops working immediately, including for
  the request that changed it, if reused.
- New env var: `PASSWORD_RESET_TOKEN_TTL_MINUTES` (default 60).

# 0004: Auth mechanism (email + password, JWT bearer token)

**Status:** Accepted (2026-09-15)

## Context
ADR 0003 fixed the tenancy/identity model (Community as tenant root,
registration creates a Community + Administrator User, invites by email)
but explicitly left the concrete auth mechanism and session-carrying
mechanism open ("Auth mechanism itself... not decided"). Implementing
registration and login in `chor-app-server` required deciding both before
any endpoint could be built.

## Decision

### Credentials: email + password
A `User` authenticates with their email and a password. Passwords are
hashed (bcrypt) before storage; the plain password and the hash are never
returned by any endpoint.

### Session: JWT in `Authorization: Bearer <token>`
Registration and login both return a JWT access token. Clients send it as
`Authorization: Bearer <token>` on every subsequent request. The token is
**not** carried as a cookie — `chor-app-server`'s initial scaffold already
had `cookie-parser` and CORS `credentials: true` wired in, but those were
leftovers from initial setup, not a considered choice, and are unrelated to
this decision.

`JWT_SECRET` and `JWT_EXPIRES_IN` are environment/config, never hardcoded.

### Session invalidation on password change: a `tokenVersion` counter
**Added 2026-09-15**, while implementing `identity/password-management`
(change-password / forgot-password / reset-password). Every `User` carries
a `tokenVersion` integer (default `0`), embedded as a claim in every JWT
issued for them. `JwtStrategy` rejects a token whose `tokenVersion` claim
doesn't match the User's current value. Changing or resetting a password
atomically increments `tokenVersion` in the same database write, which
invalidates every token issued before that moment — except the fresh token
that change-password/reset-password itself returns, which carries the new
value.

This replaced a first attempt that compared the JWT's `iat` claim against
a `passwordChangedAt` timestamp on `User`: broken because `iat` only has
second-level precision, so logging in and changing the password inside the
same wall-clock second (easily reproducible) left the old token still
accepted. An integer equality check (`tokenVersion`) has no such timing
window — see `identity/password-management`'s design.md for the full
account.

## Consequences
- `chor-app-client`'s custom fetch-based HTTP utility (ADR 0001) is
  responsible for storing the token and attaching the `Authorization`
  header itself; there is no browser-managed cookie jar to rely on.
- No server-side session store or token revocation exists yet. A
  compromised or leaked token is valid until it expires. Acceptable for now
  given `JWT_EXPIRES_IN` is short-lived and configurable; revisit
  (refresh tokens, a revocation list) only if a real requirement or
  incident calls for it.
- Every future protected endpoint in any bounded context authenticates the
  same way: validate the bearer JWT, resolve the `User` it names. This ADR
  is binding for all of `chor-app-server`, not just Identity & Community.
- Because SMTP is not required for login (only for invites, ADR 0003), this
  auth mechanism does not make every login depend on mail delivery.

## Alternatives considered
- **Passwordless / magic link** — rejected as the default login mechanism:
  would make SMTP a hard dependency of every single login, not just
  invites/resets, for no stated product requirement.
- **OAuth (e.g. Google)** — rejected for now: adds provider-specific account
  linking on top of "Administrator creates a Community and invites members
  by email" (ADR 0003) without a requirement asking for it. Not ruled out
  as an additional method later.
- **httpOnly cookie session** — considered because `cookie-parser` and CORS
  `credentials: true` were already present in the scaffold. Rejected in
  favor of a bearer token so the client owns the token explicitly (ADR
  0001's fetch utility) and to avoid the CSRF-protection design a cookie
  session would require.

## Open follow-ups
- ~~Password reset flow — not designed...~~ Decided 2026-09-15: see
  `identity/password-management` capability (email+code, `tokenVersion`
  invalidation above).
- Token refresh / general revocation strategy beyond the password-change
  case above (e.g. a user-facing "log out of all sessions", or revoking a
  single stolen token without changing the password) — still not built;
  revisit only if needed.
- Community invite mechanics (ADR 0003) still separately open — the
  `tokenVersion` mechanism above doesn't address invite tokens, which are
  a different flow.

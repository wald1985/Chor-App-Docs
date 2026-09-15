## 1. Database

- [x] 1.1 Add `PasswordResetToken` model (`tokenHash` unique, `expiresAt`, `usedAt`) to `prisma/schema.prisma`
- [x] 1.2 Add `tokenVersion Int @default(0)` to `User`
- [x] 1.3 Run and apply both migrations against the real database

## 2. Notifications module (`src/notifications/`)

- [x] 2.1 `EmailSender` port (`domain/ports/email-sender.port.ts`)
- [x] 2.2 `NodemailerEmailSender` implementation reading `MAIL_*` config
- [x] 2.3 `NotificationsModule`, imported by `IdentityModule`

## 3. Domain layer (`src/identity/domain/`)

- [x] 3.1 `PasswordResetToken` entity (`isUsed`/`isExpired`/`isValid`)
- [x] 3.2 `IncorrectCurrentPasswordError`, `InvalidOrExpiredResetTokenError`
- [x] 3.3 `PasswordResetTokenRepository` port (create/findByTokenHash/markUsed/deleteAllForUser)
- [x] 3.4 `ResetTokenGenerator` port (generate raw+hash+expiry, hash-on-verify)
- [x] 3.5 Extend `UserRepository` port: `updatePasswordHash` now bumps and returns the new `tokenVersion`
- [x] 3.6 Extend `AuthTokenPayload` with `tokenVersion`

## 4. Application layer (`src/identity/application/`)

- [x] 4.1 `ChangePasswordUseCase` — verify current password, update, invalidate outstanding reset codes, issue fresh token
- [x] 4.2 `ForgotPasswordUseCase` — silent no-op for unknown email; else supersede old codes, create new one, send email
- [x] 4.3 `ResetPasswordUseCase` — verify code, update password, mark used, issue fresh token + memberships

## 5. Infrastructure layer (`src/identity/infrastructure/`)

- [x] 5.1 `PrismaPasswordResetTokenRepository`
- [x] 5.2 `CryptoResetTokenGenerator` (`node:crypto`, SHA-256, configurable TTL)
- [x] 5.3 `PrismaUserRepository.updatePasswordHash` — atomic `tokenVersion: { increment: 1 }`
- [x] 5.4 `JwtStrategy` — reject on `tokenVersion` mismatch (see design.md for why this replaced an earlier, buggy `iat`-based check)

## 6. Interface layer (`src/identity/interface/`)

- [x] 6.1 `ChangePasswordDto`, `ForgotPasswordDto`, `ResetPasswordDto`
- [x] 6.2 `AuthController`: `POST /auth/change-password` (guarded), `POST /auth/forgot-password`, `POST /auth/reset-password`; map domain errors to `400`

## 7. Verification

- [x] 7.1 `tsc --noEmit`, `nest build`, `eslint` all clean
- [x] 7.2 End-to-end against the real database and real SMTP: change-password (wrong/correct current password, old token invalidated, new token works, login with new password); forgot-password (unknown email still generic 200, real SMTP send succeeds); reset-password (garbage token rejected, valid token succeeds, reused token rejected, login with reset password)
- [x] 7.3 Found and fixed the `iat`-vs-`passwordChangedAt` same-second race by switching to `tokenVersion`; re-verified with a back-to-back login+change-password test
- [x] 7.4 Cleaned up all test data created during verification

## 8. Documentation (this repo)

- [x] 8.1 Write this change's proposal/specs/design
- [x] 8.2 Amend `decisions/0004-auth-mechanism.md` with the `tokenVersion` session-invalidation mechanism
- [x] 8.3 Archive this change into `openspec/specs/identity/password-management/`

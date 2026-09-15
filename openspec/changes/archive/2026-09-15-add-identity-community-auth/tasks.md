## 1. Database

- [x] 1.1 Add Prisma (pinned to stable `7.10.0`) and `@prisma/client` to `chor-app-server`
- [x] 1.2 Define `Community`, `User`, `CommunityMembership` (+ `CommunityRole` enum) in `prisma/schema.prisma`
- [x] 1.3 Add `prisma.config.ts` and `@prisma/adapter-pg`-based `PrismaService`/`PrismaModule`
- [x] 1.4 Wire `DATABASE_URL` from existing `POSTGRES_*` env vars and run the first migration against the real instance

## 2. Domain layer (`src/identity/domain/`)

- [x] 2.1 `User`, `Community`, `CommunityMembership` entities + `CommunityRole` enum
- [x] 2.2 `EmailAlreadyRegisteredError`, `InvalidCredentialsError` domain errors
- [x] 2.3 Repository/service ports: `UserRepository`, `MembershipRepository`, `RegistrationRepository`, `PasswordHasher`, `TokenIssuer`

## 3. Application layer (`src/identity/application/`)

- [x] 3.1 `RegisterUseCase` — checks for an existing email, hashes the password, calls `RegistrationRepository` to create Community + User + Administrator membership atomically
- [x] 3.2 `LoginUseCase` — verifies credentials, loads memberships, issues a token
- [x] 3.3 `GetCurrentUserUseCase` — resolves profile + memberships from a user id

## 4. Infrastructure layer (`src/identity/infrastructure/`)

- [x] 4.1 Prisma-backed `UserRepository`/`MembershipRepository`/`RegistrationRepository` implementations (registration wrapped in one `$transaction`)
- [x] 4.2 `BcryptPasswordHasher`
- [x] 4.3 `JwtTokenIssuer` + `JwtStrategy` (passport-jwt, bearer extraction)

## 5. Interface layer (`src/identity/interface/`)

- [x] 5.1 `RegisterDto`/`LoginDto` with `class-validator` constraints
- [x] 5.2 `JwtAuthGuard`, `@CurrentUser()` decorator
- [x] 5.3 `AuthController`: `POST /auth/register`, `POST /auth/login`, `GET /auth/me`; map domain errors to `409`/`401`

## 6. Wiring & verification

- [x] 6.1 `IdentityModule`, global `ConfigModule` + `ValidationPipe`, mount in `AppModule`
- [x] 6.2 Fix pre-existing broken `main.ts` bootstrap (dead/broken Swagger basic-auth block, wrong `cookie-parser` import) that otherwise crashed startup
- [x] 6.3 Manual end-to-end verification against the real database: register, duplicate-email conflict, wrong-password rejection, successful login, `/auth/me` with and without a token, DTO validation errors — then clean up the test data created during verification
- [x] 6.4 `tsc --noEmit`, `nest build`, and `eslint` all clean

## 7. Documentation (this repo)

- [x] 7.1 Record the auth mechanism decision as `decisions/0004-auth-mechanism.md`
- [x] 7.2 Write this change's proposal/specs/design
- [x] 7.3 Archive this change into `openspec/specs/identity/registration-and-login/`

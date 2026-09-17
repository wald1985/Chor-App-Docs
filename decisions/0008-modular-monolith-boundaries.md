# 0008: Modular monolith — module boundaries per feature/bounded context

**Status:** Accepted (2026-09-17)

## Context
ADR 0002 already says "DDD, one NestJS module per bounded context" with a
fixed internal layering (`domain/application/infrastructure/interface`).
In practice that internal layering is followed, but nothing so far governs
how modules relate to *each other*, and the code has already drifted from
looking like a modular monolith:

- `NotificationsModule` is never imported by `AppModule` — it is only
  imported inside `IdentityModule` (`src/identity/identity.module.ts`), so
  it exists as a dependency of Identity rather than as its own top-level
  feature module.
- `ForgotPasswordUseCase` (`src/identity/application/use-cases/forgot-password.use-case.ts`)
  imports Notifications' `EmailSender` port with a deep relative path
  (`../../../notifications/domain/ports/email-sender.port`) straight into
  another module's `domain/`, instead of depending only on something
  `NotificationsModule` explicitly exports.

Neither breaks anything today, but both blur the line between "modules
that happen to sit in the same repo" and a real modular monolith, and set
a bad precedent as more bounded contexts (People, Repertoire, Rehearsal,
...) are added. The product owner flagged this directly: despite DDD
layering inside each module, the codebase doesn't yet read as a modular
monolith, and every feature must be its own module going forward.

## Decision
Every capability/bounded context is one **top-level** NestJS module,
independently pluggable, on top of the layering ADR 0002 already defines:

1. **Every bounded-context module is registered directly in `AppModule`.**
   A module is never wired in *only* as another feature module's import —
   if `AppModule` doesn't import it, it isn't a real module boundary.
2. **Cross-module access only through what a module's `*.module.ts`
   explicitly `exports`** (a guard, a decorator, a use case, a port/token
   intended for reuse). A module never reaches into another module's
   `domain/`, `application/`, or `infrastructure/` with a relative import
   that walks up and back down into a sibling module's folder. If a use
   case in module A needs something from module B, B exports it (a
   provider, an interface) and A imports B's module and injects it — A
   never imports B's internals directly.
3. **A module owns its own persistence concern.** It may only read/write
   the Prisma models that belong to its bounded context (already implied
   by ADR 0002's "each context's `infrastructure/` only touches its own
   models"); this ADR makes it explicit that this is a *module* boundary,
   not just a documentation convention.
4. **Shared, framework-free code** (cross-cutting value objects, generic
   errors, etc., if any emerge) lives under `src/shared/`, never inside one
   feature module for another to reach into.
5. **New modules follow the same shape as `IdentityModule`/
   `NotificationsModule`**: a `<feature>.module.ts` at the module root,
   `domain/application/infrastructure/interface` inside, and an explicit
   `exports: [...]` array listing exactly what other modules may depend on
   — nothing more.

This does not change ADR 0002's internal layering rules, only how modules
compose with each other.

## Consequences
- `AppModule` becomes the single place that shows the full list of
  features in the system — reading it should be enough to know what
  bounded contexts exist, without having to open every feature module to
  discover a nested one.
- Adding a capability (People, Repertoire, ...) means adding one line to
  `AppModule`'s `imports`, not threading it through an existing module.
- **Known deviation fixed (2026-09-17):** `NotificationsModule` is now
  imported directly by `AppModule`, exposes its public API through
  `src/notifications/index.ts`, and `ForgotPasswordUseCase` depends on
  `EMAIL_SENDER` / `EmailSender` via that public barrel instead of deep
  relative imports into `notifications/domain/`. An ESLint rule
  (`no-restricted-imports`) enforces module boundaries across `src/**`
  to prevent deep cross-module imports into `domain/`, `application/`,
  or `infrastructure/`.
- Code review (and the "review code against design and architecture" step
  in `development-process.md`'s Implementation phase) must check new
  modules against this ADR, not just against ADR 0002's internal layering.

## Alternatives considered
- **Leave it as "one module per bounded context" without composition
  rules** (status quo) — rejected: it already produced the Notifications
  drift above, and gets worse as more modules are added.
- **A stricter enforcement tool** (e.g. `eslint-plugin-boundaries` or
  Nest's own module-graph linting to fail CI on a forbidden cross-module
  import) — not decided against, just not adopted yet; revisit once there
  are enough modules for manual review to become unreliable.

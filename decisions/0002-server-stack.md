# 0002: chor-app-server tech stack & architecture

**Status:** Accepted (2026-09-15)

## Context
`chor-app-server` had no stack decided yet (see its AGENTS.md, previously
marked TBD). Needed before any scaffolding or first backend-facing
spec/tasks.

## Decision

### Framework & persistence
- **NestJS + TypeScript.**
- **Prisma + PostgreSQL** for persistence.

### Architecture: DDD, per bounded context, model isolated from the DB
One NestJS module per **bounded context** (e.g. song collections,
Rehearsal/Performance logging — exact contexts to be confirmed as specs
are
written). Inside each bounded-context module, four sublayers:

- `domain/` — plain TypeScript entities, value objects, domain services,
  and **repository interfaces (ports)**. No NestJS decorators, no Prisma
  imports, no framework dependency of any kind. This is what enforces
  "isolate the model from the database."
- `application/` — use cases / application services that orchestrate
  domain objects via the repository ports. Depends on `domain/` only.
- `infrastructure/` — Prisma-based repository **implementations
  (adapters)** for the domain's repository interfaces, plus mappers
  between Prisma models and domain entities. Depends on `domain/` (to
  implement its ports) and on Prisma.
- `interface/` — NestJS controllers and DTOs (Swagger-decorated, see
  below), request/response mapping. Depends on `application/`. Never
  imports Prisma directly, never puts Nest/Swagger decorators on domain
  entities.

**Dependency rule:** `domain` depends on nothing project-specific;
`application` depends on `domain`; `infrastructure` depends on `domain` (to
implement it) plus Prisma; `interface` depends on `application`. Nothing
outside `domain/` may leak into it.

Prisma needs one `schema.prisma`; it lives centrally (e.g.
`prisma/schema.prisma` at repo root) and is shared across bounded
contexts. Each context's `infrastructure/` layer only touches its own
models in that schema; mappers keep Prisma-shape changes from ever
touching `domain/` code directly.

### Email
Connect to an SMTP mail server via `nodemailer`, hidden behind a
domain-level port (an `EmailSender`-style interface in `domain/`, likely
in a shared/notifications bounded context), with the concrete
SMTP-based implementation in that context's `infrastructure/`. SMTP host,
port and credentials come from environment/config, never hardcoded. This
ADR fixes the **transport** only (SMTP + nodemailer, isolated behind a
port) — which events trigger an email, and their content/templates, belong
in the capability spec that needs them, not here.

### API docs
`@nestjs/swagger`, auto-generated from `@ApiProperty()`-decorated DTOs
that live in each bounded context's `interface/` layer only. Domain
entities are never given Nest/Swagger decorators — that would violate the
isolation rule above. Exact docs route and whether it's public or
protected: TBD at scaffolding time.

## Consequences
- More upfront boilerplate than using Prisma models as the "domain
  model" directly: mappers Prisma↔domain, repository interfaces +
  implementations, DTOs separate from domain entities. Traded for
  domain logic being unit-testable without a database, and the ability to
  change ORM/DB later without touching `domain/`/`application/`.
- One shared `schema.prisma` across bounded contexts needs discipline: a
  migration for one context shouldn't silently couple to another's model
  unless a cross-context relation is deliberately being modeled.
- Email sending is decoupled from any specific trigger/content for now;
  each future spec that needs email defines its own "when/what" against
  this ADR's "how."

## Alternatives considered
- **Prisma-generated types as the domain model directly** — rejected:
  defeats the explicit requirement to isolate the model from the DB.
- **Global layering** (`src/domain`, `src/application`, ... at repo root,
  organized by layer first) instead of per-bounded-context — considered;
  user chose per-bounded-context for better long-term scaling and fit with
  Nest's module system.
- **Third-party transactional email API/SDK** instead of raw SMTP — not
  chosen; the ask was specifically "connection to a mail server," so
  SMTP + nodemailer is the default. Revisit if a specific provider is
  later required.

## Open follow-ups
- Bounded context names/boundaries not yet defined — will emerge from the
  first specs (song collections, Rehearsal/Performance are likely
candidates).
- Swagger route path and auth-protection on the docs endpoint — decide at
  scaffolding time.
- SMTP server details (host, self-hosted vs. relay/provider) — not
  specified yet, needed before implementation.
- Auth/user model is still undecided (see `openspec/config.yaml`) — it
  affects both email use cases (e.g. password reset) and how
  Swagger-documented endpoints get secured.

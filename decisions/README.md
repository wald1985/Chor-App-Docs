# Architecture Decision Records

Cross-cutting technical decisions (stack, tooling, conventions) that aren't
tied to one OpenSpec capability live here, one file per decision:

`NNNN-short-title.md`, numbered sequentially, never renumbered/reused.

Each file: **Status** (Proposed / Accepted / Superseded by NNNN),
**Context**, **Decision**, **Consequences**, and optionally **Alternatives
considered**. Once Accepted, treat a decision as binding for agents working
in the affected repo(s) until a new ADR supersedes it — don't silently
deviate from it.

- `0001-client-stack.md` — chor-app-client tech stack
- `0002-server-stack.md` — chor-app-server tech stack & DDD architecture
- `0003-multi-tenancy-identity.md` — Community/User tenancy model
- `0004-auth-mechanism.md` — email+password auth, JWT bearer session
- `0005-deployment-and-cicd.md` — *Superseded by 0006.* Planned Docker
  builds, CI vs. CD split, deploy triggers for both repos
- `0006-deployment-as-implemented.md` — the working deploy pipeline of
  both repos (host layout, secrets, config, client API URL resolution)
  and its known issues
- `0007-community-scoped-requests-and-permissions.md` — `/communities/:communityId/...`
  routing, membership guard, hardcoded per-member permissions
- `0008-modular-monolith-boundaries.md` — one top-level NestJS module per
  feature/bounded context, registered in `AppModule`; cross-module access
  only through a module's own exports, never deep imports into another
  module's internals
- `0009-repertoire-folders-attachments-themes.md` — Community repertoire =
  attached library books (read-only, live) + own Folders (*Mappe*, several,
  editable, `isNew`); custom Community themes on any song; `SongLookup` port;
  `SongCollectionType`/NewSongs removed
- `0010-public-book-library.md` — global library of printed editions in its
  own module, live-linked into Communities; hybrid numbering (per book or
  continuous series); uploads parsed once, re-upload updates in place;
  library main themes delivered with books; superadmin usage warning via
  `LibraryAdminModule`; `/admin/library` behind a placeholder `SuperAdminGuard`
  *(guard superseded by 0011)*
- `0011-superadmin-identity-and-session.md` — `Superadmin` as a separate
  entity/table from `User`, all superadmins equal (no permission model), JWT
  `aud` claim distinguishes a superadmin session from a user session, seed
  script creates/recovers the first superadmin, replaces ADR 0010's
  placeholder `SuperAdminGuard` with real guards


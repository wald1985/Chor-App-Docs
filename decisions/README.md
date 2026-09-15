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


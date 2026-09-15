# chor-app-docs — agent instructions

## What this repo is
Product specs and planning for **Chor-App** (choir repertoire/rehearsal
tracker), managed with **OpenSpec**. This repo has **no application code**.
Two sibling repos, checked out next to this one on disk, implement what's
specified here:

- `../chor-app-client` — React + TypeScript frontend
- `../chor-app-server` — NestJS backend

These three repos are kept **deliberately separate** (no monorepo). Do not
propose merging them.

## Where things live
- `openspec/config.yaml` — full product/domain context (song collections,
  Performance/Rehearsal/WarmUp logic, open architecture questions).
  **Read this before writing or changing any spec.**
- `glossary.md` — German UI term <-> English domain/code name mapping.
  **The target UI is German; the domain model and all code are English.**
  Read before naming anything in a spec or design.md.
- `openspec/specs/` — current source-of-truth capability specs (empty until
  the first change is archived).
- `openspec/changes/<change-id>/` — in-flight proposals: `proposal.md`,
  `specs/<capability>/spec.md` (delta), `design.md`, `tasks.md`.
- `openspec/changes/archive/` — completed changes.
- `chor-app_1.html`, `chor-app_V2.html`, `README.md`, `README_old.md` — the
  **legacy single-file HTML/Excel app** this project replaces. Treat these
  as the functional source of truth for a capability until it has a spec;
  read them before writing a spec that covers what they already do.
- `domain-model.md` — draft domain entities/value objects by candidate
  bounded context. Living reference, feeds each capability's design.md.


## Workflow
Use the OpenSpec slash commands (`/opsx:propose`, `/opsx:apply`,
`/opsx:archive`, etc. — see `.claude/commands/opsx/`):

1. `/opsx:propose "<what to build>"` — creates the change, including specs,
   design and a task list. Planning only, no code.
2. Implementation of `tasks.md` happens **in the code repos**
   (`chor-app-client` / `chor-app-server`), not here — since this repo has
   no application code. An agent working there should read the relevant
   `openspec/changes/<change-id>/` folder here (directly via the sibling
   path, or via `openspec ... --store chor-app` once the store is
   registered — see below) to get its tasks and specs.
3. Once implemented and merged in the code repo(s), come back here to run
   `/opsx:archive` and fold the change into `openspec/specs/`.

## Cross-repo spec access
This repo is marked as OpenSpec store id `chor-app` (`.openspec-store/`).
To let `chor-app-client`/`chor-app-server` query it without duplicating
`openspec/` there, register it once **from a real terminal** (not a bridged
session, so the registered path stays valid):

```
openspec store register /Users/alex/Desktop/apps/chor_app/chor-app-docs --id chor-app --yes
```

Then `openspec show <name> --store chor-app`, `openspec list --store
chor-app`, etc. work from any repo on this machine.

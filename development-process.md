# Chor-App: development process

**Status:** binding for all development in this project (decided 2026-09-16).

The process is split into four phases. The result of each phase becomes the
context for the next one.

## Phase 1: Research

- **Goal:** narrow the codebase down to the relevant files and functions.
- **How:** an agent (or a group of subagents) scans the project and finds
  relationships, external integrations and patterns.
- **Output:** a document containing only **facts and references** to the
  code as it is (as-is), with no advice or opinions, so that no noise is
  carried into the next phases.

## Phase 2: Design

- **Goal:** create the architectural solution before any coding starts.
- **Tools:** the **C4** model (context, containers, components), data flow
  diagrams (**DFD**) and sequence diagrams.
- **Review matters most here:** the engineer reviews the design carefully by
  eye, discusses it with colleagues and corrects it by hand where needed.
  This is the "golden time" for applying engineering skills.

## Phase 3: Planning

- **Goal:** break the implementation of the feature into concrete, complete
  phases.
- **Principle:** each phase of the plan can be verified, tested and committed
  on its own.
- **Control:** the plan passes a review (the second quality gate) before
  moving on to writing code.

## Phase 4: Implementation

- **Single agent, strictly sequential** (decided 2026-09-16, replaces the
  original multi-agent "team" of developer/reviewer/tester/security
  agents): one agent does all the work, one plan phase at a time, with no
  subagents and no parallel work. The former roles become ordered steps the
  same agent performs within every plan phase:
  1. implement the code and tests of the phase;
  2. check that the tests cover every acceptance criterion, add missing ones;
  3. review the code against the design and architecture (ADR layering);
  4. security check (injections, data leaks, authorization) where relevant;
  5. run all quality gates below, then commit.
  The next plan phase starts only after the previous one is committed.
- **Quality gates:** code is not accepted until all checks pass:
  - successful build and passing tests;
  - compliance with linter rules and complexity metrics;
  - security review (no injections, no data leaks);
  - conformance to the original design and architecture.

## Mapping to this repo (OpenSpec)

- Research output: `openspec/changes/<change-id>/research.md`.
- Design: `design.md` of the change (with C4/DFD/sequence diagrams) plus
  `proposal.md` and `specs/`; quality gate 1 = design review.
- Planning: `tasks.md`, grouped into independently verifiable and
  committable phases; quality gate 2 = plan review.
- Implementation: in `chor-app-server` / `chor-app-client`, following the
  quality gates above.

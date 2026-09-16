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

- **Multi-agent approach:** instead of a single prompt, a "team" of agents
  with separate roles is used: developer, reviewer, tester, security agent.
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

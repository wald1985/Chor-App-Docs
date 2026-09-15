# 0001: chor-app-client tech stack

**Status:** Accepted (2026-09-15)

## Context
`chor-app-client` had no stack decided yet (see its AGENTS.md, previously
marked TBD). Needed before any scaffolding or first UI-facing spec/tasks.

## Decision
- **Framework:** React + TypeScript.
- **Global/client state:** Redux Toolkit (RTK) — for state that is *not*
  server data: UI state (open modals, active tab), in-progress form drafts,
  filters not yet submitted, session/auth info once an auth model exists.
- **Server state / data fetching & caching:** TanStack React Query — owns
  *all* data fetched from `chor-app-server` (song lists, Proben,
  Auswertung, etc.), including caching and invalidation. Do **not** use
  RTK Query — RTK is for client state only, to avoid two competing caches
  for the same data.
- **UI / component library:** `react-bootstrap` (Bootstrap 5 styled React
  components), not plain Bootstrap CSS + JS bundle. Reason: idiomatic React
  components, no jQuery/Bootstrap-JS fighting React over DOM ownership.
- **Styling philosophy: stay as close to default Bootstrap as possible.**
  Use react-bootstrap components with their default styling; reach for
  custom CSS/SCSS only when a screen genuinely needs something Bootstrap
  doesn't offer out of the box — not as the default way of building a
  screen. Don't build a custom design system or override Bootstrap's
  look-and-feel wholesale; any custom style should be the exception on a
  specific component, not the norm.
- **No Tailwind CSS.** Styling goes through `react-bootstrap`'s components
  and Bootstrap's grid/utility classes/SCSS variables — don't add Tailwind
  alongside it. Two utility-class systems fighting over the same elements
  is what this rule avoids; it's not a judgment on Tailwind in general.
- **Layout:** mobile-first. Design and build for the smallest screen first,
  then progressively enhance using Bootstrap's responsive grid/breakpoints
  for larger screens.
- **HTTP layer: no axios.** Team does not trust axios as a dependency.
  Build a small custom HTTP utility on top of native `fetch` instead, with
  axios-equivalent ergonomics: configurable base URL, JSON
  request/response handling, typed responses, normalized error shape,
  timeout/abort support, and a hook point for auth-header injection once
  the auth model is decided. Exact name/location (e.g.
  `src/lib/http/client.ts`) is chosen when the client repo is scaffolded —
  update this ADR or `chor-app-client/AGENTS.md` once that happens.

## Consequences
- Extra upfront work to build and maintain the custom HTTP utility instead
  of using axios off the shelf.
- Mixing RTK and React Query requires discipline: any piece of state must
  live in exactly one of the two, per the split above, or it will drift out
  of sync.
- Mobile-first means the smallest supported viewport (assume phone) is the
  default design target, not an afterthought.

## Alternatives considered
- **axios** — rejected: explicit lack of trust in the dependency.
- **RTK Query instead of React Query** — rejected: team specifically wants
  React Query for server state.
- **Plain Bootstrap CSS + JS bundle instead of react-bootstrap** —
  rejected: worse fit for React, more manual DOM/interactivity handling.
- **Tailwind CSS** — rejected: the styling system is Bootstrap (via
  react-bootstrap); adding Tailwind on top would mean two competing
  utility/styling conventions in the same codebase.

## Open follow-ups
- Custom HTTP utility's exact API/location — decide at scaffolding time.
- Auth-header injection depends on the still-undecided auth model (see
  `openspec/config.yaml` → open architecture decisions).

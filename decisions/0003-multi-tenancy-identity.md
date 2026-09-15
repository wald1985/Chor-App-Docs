# 0003: Multi-tenancy & identity (Community / User)

**Status:** Accepted (2026-09-15)

**Revised (2026-09-15, same day):** the User-Community cardinality below
was changed from single- to multi-community membership after comparing it
against adding a `MusicalGroup` layer to solve the same underlying
problem (a person involved with two ensembles needing two accounts). See
"Alternatives considered" and "Considered and deferred: MusicalGroup".

## Context
The legacy app is single-device/local, with an informal fixed set of known
conductors and no real user accounts or tenant separation. The rebuild
must support multiple independent choirs/groups, each with isolated data,
and real user accounts with roles.

## Decision
- **Community** — entity, tenant root. All domain data from every other
  bounded context (Song, Theme, Rehearsal, Performance, Person, ...) is
  scoped
  to exactly one Community — no cross-community data sharing/visibility.
- **User** — entity: account (auth mechanism TBD). **Can belong to
  multiple Communities**, via a `CommunityMembership` (userId, communityId,
  role) join — not a single `communityId` on User. Each membership carries
  its own Role, independent per Community (e.g. Administrator in one,
  general member in another). The client/session needs an "active
  Community" concept — pick which Community's data you're acting in,
  similar to switching workspaces/orgs in Slack or GitHub — since every
  request is still scoped to exactly one Community's data (see the
  Community bullet above; that part is unchanged).
- **Registration flow:** the first user to register creates both a new
  Community and their own User account in one step, and is automatically
  assigned the **Administrator** role for that Community. There is no
  separate "create a community" step.
- **Adding members:** an Administrator can add/invite further Users into
  their Community — typically conductors, choir wardens ("старосты") per
  the product owner's own framing. **Decided: invite is by email** — the
  Administrator enters the invitee's email address, and the system sends
  an invitation via the SMTP capability from ADR 0002. Token/link format,
  expiry, resend, and what exactly the invitee sets on accepting are left
  to the Identity & Community capability spec, not fixed here.
- **Roles:** at minimum `Administrator` and a general member role for now.
  Finer-grained roles (e.g. a distinct "Conductor" or "Warden" role with
  different permissions) are explicitly **not decided** — don't assume
  beyond Admin vs. member until a capability spec says otherwise.
- **Person ≠ User.** The `Person` entity (pianist/conductor used for
  Auswertung stats — see `domain-model.md`) stays separate from `User`,
  with an **optional** link to a User. This allows guest pianists/
  conductors who are tracked in stats but have no login. `Person` is
  scoped to a Community, same as other repertoire/rehearsal data.

## Consequences
- Every repository/query in every other bounded context must filter by
  `communityId` — a cross-cutting concern touching Repertoire, Rehearsal &
  performance log, and Auswertung. Implement it as one consistent pattern
  (aggregate roots carry `communityId`; application-layer use cases always
  scope by the acting user's community) rather than reinventing it per
  context.
- Multi-community membership needs an "active Community" switcher in the
  client and in every authenticated request/session — there's no single
  implicit Community for a User anymore. This is a well-understood pattern
  (Slack/GitHub-style workspace switching), not a novel one.
- Person/User being separate lets Auswertung and Identity evolve
  independently, but whoever builds "admin adds a User" must remember to
  offer linking to an existing Person (or creating one) so stats don't
  fragment between a User's linked Person and an unrelated guest Person
  with a similar name.

## Alternatives considered
- **Merging Person and User** — rejected: would force every guest
  pianist/conductor to have a login.
- **Single-community-per-user** — the original decision made earlier the
  same day; reversed once compared directly against adding a
  `MusicalGroup` layer as the other way to solve "one person, two
  ensembles, one login." Multi-community membership is a small, contained
  change (a join table + an active-Community selector) that touches only
  Identity & Community, versus `MusicalGroup`, which would cut across
  every bounded context modeled so far (Repertoire, Rehearsal &
  performance log, Reporting) for a need that's still hypothetical. See
  "Considered and deferred: MusicalGroup" below.

## Considered and deferred: MusicalGroup / multi-ensemble

Considered adding a `MusicalGroup` (Ensemble) entity between `Community`
and the Repertoire/Rehearsal data — so one organization could run several
ensembles (e.g. a choir *and* an orchestra) sharing the same
Administrator, and sharing `Person`/`User` records across them (one
conductor or pianist serving both), while each ensemble keeps its own
repertoire.

**Decided not to build this now** (2026-09-15) — asked the user directly:
no concrete case exists today, it's a hypothetical future need. Instead,
the actual driver (someone involved with two ensembles needing two
accounts) is solved by multi-community membership (see Decision above,
revised same day): **one Community = one musical group**; an organization
that runs a second ensemble creates a **second Community**, and thanks to
multi-community membership the same User can belong to both without a
second login. What's still not shared across Communities: `Person`
records (pianists/conductors) and Reporting — each Community's stats
stay separate, same as two separate Slack workspaces don't share a
unified report. That's expected, not a gap: nothing today calls for
merged cross-ensemble reporting.

If this becomes a real requirement later, expect it to be a genuine
migration, not a toggle: `Song`, `Theme`, `Rehearsal`, `Performance`, and
`Person` would move from being scoped directly by `communityId` to being
scoped by a new `MusicalGroupId` nested under `Community`, and `User`'s
role model would need to become per-`MusicalGroup` rather than
per-`Community` (e.g. Administrator of the Community, but only Conductor
of one specific ensemble). Don't build toward this speculatively — revisit
only when a real multi-ensemble organization is a confirmed target.

## Open follow-ups
- ~~Auth mechanism itself (password+email, magic link, OAuth, ...) — not
  decided.~~ Decided 2026-09-15: email+password with a JWT bearer token,
  see `decisions/0004-auth-mechanism.md`.
- Invite email mechanics (token/link format, expiry, resend, what the
  invitee sets on accept) — the channel is decided (email), the mechanics
  are left to the Identity & Community capability spec.
- Role granularity beyond Administrator/member — not decided.
- Whether/how a Community can be renamed, deleted, or have its data
  exported — not addressed.

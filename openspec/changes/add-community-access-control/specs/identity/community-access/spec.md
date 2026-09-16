## Purpose

Makes every Community-scoped request explicit about its target Community,
ensures only members of that Community can reach its data, and lets the
Administrator decide which additional actions each member may perform.

## ADDED Requirements

### Requirement: Community-scoped requests name their Community and require membership
The system SHALL expose Community-scoped endpoints under
`/communities/:communityId/...` and SHALL only process such a request when
the authenticated User has a `CommunityMembership` in that Community.

#### Scenario: Member accesses their Community
- **WHEN** an authenticated User with a membership in Community C calls an
  endpoint under `/communities/C/...`
- **THEN** the request is processed in the context of Community C and that
  membership

#### Scenario: Non-member is rejected
- **WHEN** an authenticated User without a membership in Community C (or C
  does not exist) calls an endpoint under `/communities/C/...`
- **THEN** the system rejects the request with a forbidden error and
  returns no data about Community C

#### Scenario: Unauthenticated request
- **WHEN** a request to a Community-scoped endpoint carries no valid bearer
  token
- **THEN** the system rejects it as unauthorized before any Community check

### Requirement: Permissions are a fixed, code-defined set
The system SHALL define the available Community permissions as a fixed set
in code; the MVP set SHALL contain `PEOPLE_MANAGE`. Permissions SHALL NOT be
creatable or renamable at runtime.

#### Scenario: Unknown permission is rejected
- **WHEN** an Administrator tries to grant a permission value that is not in
  the defined set
- **THEN** the system rejects the request with a validation error and
  changes nothing

### Requirement: Administrator holds every permission implicitly
A membership with role `ADMINISTRATOR` SHALL be treated as holding every
defined permission, regardless of its stored permission set.

#### Scenario: Administrator performs a protected action
- **WHEN** an Administrator of Community C calls an endpoint that requires
  `PEOPLE_MANAGE` in C
- **THEN** the request is allowed without any permission having been granted

### Requirement: Members hold only granted permissions
A membership with role `MEMBER` SHALL hold exactly the permissions granted
to it, and SHALL start with none.

#### Scenario: Member without permission
- **WHEN** a `MEMBER` of Community C without `PEOPLE_MANAGE` calls an
  endpoint that requires it
- **THEN** the system rejects the request with a forbidden error and
  changes nothing

#### Scenario: Member with permission
- **WHEN** a `MEMBER` of Community C who was granted `PEOPLE_MANAGE` calls
  an endpoint that requires it
- **THEN** the request is allowed

#### Scenario: Permission is per Community
- **WHEN** a User is a `MEMBER` with `PEOPLE_MANAGE` in Community A and a
  `MEMBER` without it in Community B
- **THEN** protected People actions are allowed in A and rejected in B

### Requirement: Administrator lists members with permissions
The system SHALL let an Administrator of a Community list its memberships,
each with membership id, User name, User email, role, and effective
permissions. Non-administrators SHALL be rejected.

#### Scenario: Administrator lists members
- **WHEN** an Administrator of Community C requests C's member list
- **THEN** the system returns every membership of C with the fields above

#### Scenario: Member requests member list
- **WHEN** a `MEMBER` of Community C requests C's member list
- **THEN** the system rejects the request with a forbidden error

### Requirement: Administrator sets a member's permissions
The system SHALL let an Administrator of a Community replace the full
permission set of a `MEMBER` membership in that Community. Setting
permissions on an `ADMINISTRATOR` membership SHALL be rejected.

#### Scenario: Grant a permission
- **WHEN** an Administrator of C sets the permissions of a `MEMBER`
  membership in C to `[PEOPLE_MANAGE]`
- **THEN** that membership holds exactly `PEOPLE_MANAGE` from the next
  request on, and the updated membership is returned

#### Scenario: Revoke all permissions
- **WHEN** an Administrator of C sets a `MEMBER` membership's permissions to
  an empty set
- **THEN** that membership holds no permissions from the next request on

#### Scenario: Target is an Administrator
- **WHEN** an Administrator tries to set permissions on an `ADMINISTRATOR`
  membership
- **THEN** the system rejects the request and changes nothing

#### Scenario: Membership of another Community
- **WHEN** an Administrator of C targets a membership id that does not
  belong to C
- **THEN** the system responds as not found and changes nothing

### Requirement: Authenticated user responses include effective permissions
Every membership returned by login and by the current-user endpoint SHALL
include its effective permissions (all defined permissions for
`ADMINISTRATOR`, the granted set for `MEMBER`).

#### Scenario: Login as Administrator
- **WHEN** an Administrator logs in
- **THEN** each of their `ADMINISTRATOR` memberships lists every defined
  permission

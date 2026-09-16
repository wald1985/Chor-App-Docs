## Purpose

Keeps one record per pianist/conductor within a Community, so every other
capability can reference the same human by a stable id regardless of which
roles they hold, and history is never broken by removing someone.

## ADDED Requirements

### Requirement: A Person belongs to one Community and has at least one role
The system SHALL store each Person with a name, the Community it belongs
to, and a non-empty set of roles drawn from the fixed set `PIANIST`,
`CONDUCTOR`. A Person MAY hold both roles.

#### Scenario: Create a Person with both roles
- **WHEN** a user with `PEOPLE_MANAGE` in Community C creates a Person
  named "Alex" with roles `[PIANIST, CONDUCTOR]`
- **THEN** the system creates one active Person in C holding both roles and
  returns its id, name, roles and archived state

#### Scenario: Create a Person without roles
- **WHEN** a user with `PEOPLE_MANAGE` submits a Person with an empty role
  set
- **THEN** the system rejects the request with a validation error and
  creates nothing

#### Scenario: Unknown role
- **WHEN** a request contains a role value outside `PIANIST`, `CONDUCTOR`
- **THEN** the system rejects the request with a validation error

#### Scenario: Duplicate roles are collapsed
- **WHEN** a request contains `[PIANIST, PIANIST]`
- **THEN** the Person holds the role `PIANIST` once

### Requirement: Person names are valid and unique per Community
The system SHALL trim the name, SHALL require 1 to 100 characters after
trimming, and SHALL reject a name that equals (case-insensitively) the name
of any other Person in the same Community, archived or not.

#### Scenario: Duplicate name in the same Community
- **WHEN** Community C already has a Person "Paul" and a user creates or
  renames a Person to "paul "
- **THEN** the system rejects the request with a conflict error

#### Scenario: Duplicate of an archived Person
- **WHEN** Community C has an archived Person "Paul" and a user creates a
  new Person "Paul"
- **THEN** the system rejects the request with a conflict error that
  identifies the archived Person, so it can be restored instead

#### Scenario: Same name in another Community
- **WHEN** Community A has a Person "Paul" and a user creates "Paul" in
  Community B
- **THEN** the Person is created in B

#### Scenario: Blank name
- **WHEN** the submitted name is empty or whitespace only
- **THEN** the system rejects the request with a validation error

### Requirement: Members list and view People of their Community
The system SHALL let any member of a Community list its People sorted by
name (case-insensitive), SHALL exclude archived People unless explicitly
requested, SHALL support filtering by role, and SHALL let any member fetch a
single Person by id, including an archived one.

#### Scenario: List pianists for a dropdown
- **WHEN** a member of C lists People filtered by role `PIANIST`
- **THEN** the system returns the active People of C holding `PIANIST`,
  including those who also hold `CONDUCTOR`, sorted by name

#### Scenario: Include archived
- **WHEN** a member of C lists People with archived included
- **THEN** active and archived People of C are returned, each with its
  archived state

#### Scenario: Fetch an archived Person by id
- **WHEN** a member of C fetches an archived Person of C by id
- **THEN** the system returns it with its archived state

#### Scenario: Person of another Community
- **WHEN** a member of C fetches a Person id that belongs to another
  Community
- **THEN** the system responds as not found

### Requirement: Managing People requires the PEOPLE_MANAGE permission
The system SHALL require `PEOPLE_MANAGE` in the target Community for
creating, editing, archiving and restoring People, and SHALL NOT require it
for listing or viewing.

#### Scenario: Member without permission tries to create
- **WHEN** a `MEMBER` of C without `PEOPLE_MANAGE` creates a Person in C
- **THEN** the system rejects the request with a forbidden error and
  creates nothing

#### Scenario: Member without permission lists People
- **WHEN** a `MEMBER` of C without `PEOPLE_MANAGE` lists People of C
- **THEN** the list is returned

### Requirement: Edit a Person's name and roles
The system SHALL let a user with `PEOPLE_MANAGE` change a Person's name
and/or replace its role set, applying the same validation and uniqueness
rules as creation. Editing an archived Person SHALL be rejected.

#### Scenario: Add the conductor role to a pianist
- **WHEN** a user with `PEOPLE_MANAGE` sets the roles of pianist "Lorina" to
  `[PIANIST, CONDUCTOR]`
- **THEN** "Lorina" holds both roles and appears in both role-filtered lists

#### Scenario: Remove the last role
- **WHEN** a user sets a Person's roles to an empty set
- **THEN** the system rejects the request with a validation error and the
  Person keeps its previous roles

#### Scenario: Edit an archived Person
- **WHEN** a user edits an archived Person
- **THEN** the system rejects the request with a conflict error

### Requirement: Removing a Person archives it instead of deleting
The system SHALL NOT offer hard deletion of People. Archiving SHALL mark a
Person as archived with a timestamp, keep all its data and id, and hide it
from default lists. Restoring SHALL make it active again.

#### Scenario: Archive a Person
- **WHEN** a user with `PEOPLE_MANAGE` archives active Person "Springer"
- **THEN** "Springer" is marked archived, no longer appears in default
  lists, and is still returned when fetched by id

#### Scenario: Restore a Person
- **WHEN** a user with `PEOPLE_MANAGE` restores archived Person "Springer"
- **THEN** "Springer" is active again with its previous name and roles

#### Scenario: Archive twice
- **WHEN** a user archives an already archived Person (or restores an
  active one)
- **THEN** the system leaves the Person unchanged and returns its current
  state

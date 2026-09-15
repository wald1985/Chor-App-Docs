## Purpose

Lets a choir set itself up as a tenant (Community) with an owning account,
and lets that account and any future member authenticate afterward, so
every other capability has an authenticated, Community-scoped user to build
on.

## ADDED Requirements

### Requirement: Registration creates a Community and its Administrator
The system SHALL accept a registration request containing a Community name,
the registrant's name, email, and password, and SHALL atomically create a
new `Community`, a new `User`, and a `CommunityMembership` linking them with
role `ADMINISTRATOR`. No separate "create a Community" step SHALL exist.

#### Scenario: First registration for a new choir
- **WHEN** a visitor submits a Community name, their name, email, and a
  password of at least 8 characters
- **THEN** the system creates a Community with that name, a User with that
  name and email, and a Membership with role `ADMINISTRATOR` linking them,
  and returns the new user id, Community id, Community name, and role

#### Scenario: Registration with an already-registered email
- **WHEN** a visitor submits a registration whose email already belongs to
  an existing User
- **THEN** the system rejects the request, creates nothing, and returns a
  conflict error without revealing any other detail about the existing
  account

#### Scenario: Registration with invalid input
- **WHEN** a visitor submits a registration missing a required field, with
  an invalid email format, or with a password shorter than 8 characters
- **THEN** the system rejects the request without creating a Community,
  User, or Membership, and reports which fields were invalid

### Requirement: Password credentials are never stored or exposed in plain text
The system SHALL store only a salted hash of a User's password, and SHALL
never include the password or its hash in any response.

#### Scenario: Stored credential is not the plain password
- **WHEN** a User registers or is later read back through any endpoint
- **THEN** the plain-text password never appears in storage or in any API
  response, only a one-way hash is persisted

### Requirement: Login authenticates by email and password
The system SHALL accept an email and password, verify them against a
registered User, and on success SHALL issue an access token plus the list
of Communities that User is a member of (Community id, Community name, and
the User's role in it).

#### Scenario: Successful login
- **WHEN** a registered User submits their correct email and password
- **THEN** the system returns an access token, the User's own id/email/name,
  and every CommunityMembership belonging to that User

#### Scenario: Login with wrong password or unknown email
- **WHEN** a login request supplies a password that does not match the
  User's stored credential, or an email with no matching User
- **THEN** the system rejects the request with the same generic
  "invalid email or password" error in both cases, without indicating
  which part was wrong

### Requirement: Authenticated requests resolve the calling User
The system SHALL let a caller present the access token issued at
registration or login to identify themselves on subsequent requests, and
SHALL provide an endpoint that returns the caller's own profile and current
CommunityMemberships from that token alone.

#### Scenario: Fetching the current user with a valid token
- **WHEN** a caller presents a valid, unexpired access token issued by this
  system
- **THEN** the system returns that token's User (id, email, name) and their
  current list of CommunityMemberships

#### Scenario: Fetching the current user without a valid token
- **WHEN** a caller presents no access token, an expired token, or a token
  that does not correspond to any current User
- **THEN** the system rejects the request as unauthorized and returns no
  profile data

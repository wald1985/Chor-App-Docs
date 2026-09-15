## Purpose

Lets a signed-in User change their own password, and lets anyone who has
forgotten theirs regain access via an emailed, single-use reset code, so a
lost or leaked password never permanently locks someone out or leaves a
stale credential valid forever.

## ADDED Requirements

### Requirement: Signed-in User can change their password
The system SHALL let an authenticated User change their password by
supplying their current password and a new one, and SHALL reject the
request if the current password does not match.

#### Scenario: Successful password change
- **WHEN** an authenticated User submits their correct current password and
  a new password of at least 8 characters
- **THEN** the system updates the stored credential to the new password and
  returns a new access token for that User

#### Scenario: Wrong current password
- **WHEN** an authenticated User submits a current password that does not
  match their stored credential
- **THEN** the system rejects the request and leaves the stored password
  unchanged

### Requirement: Forgot-password request never reveals whether an email is registered
The system SHALL accept a forgot-password request containing only an
email, and SHALL respond with the same generic confirmation whether or not
that email belongs to a registered User.

#### Scenario: Email belongs to a registered User
- **WHEN** a visitor submits a forgot-password request for an email that
  matches a User
- **THEN** the system generates a single-use reset code, emails it to that
  address, and responds with a generic confirmation message

#### Scenario: Email does not belong to any User
- **WHEN** a visitor submits a forgot-password request for an email with no
  matching User
- **THEN** the system sends no email and responds with the exact same
  generic confirmation message as the successful case

### Requirement: Reset code is single-use and time-limited
The system SHALL accept a reset code and a new password, and SHALL apply
the new password only if the code corresponds to a still-valid,
not-yet-used, unexpired request.

#### Scenario: Valid, unused, unexpired code
- **WHEN** a visitor submits a reset code that was issued, has not expired,
  and has not already been used, along with a new password of at least 8
  characters
- **THEN** the system sets that account's password to the new one, marks
  the code as used so it cannot be reused, and returns a new access token
  plus the account's Community memberships

#### Scenario: Expired, already-used, or unknown code
- **WHEN** a visitor submits a reset code that has expired, was already
  used, or does not correspond to any issued request
- **THEN** the system rejects the request without changing any password

#### Scenario: A new forgot-password request supersedes a previous one
- **WHEN** a User requests a password reset more than once before using an
  earlier code
- **THEN** only the most recently issued code can succeed; earlier codes
  for that User no longer work

### Requirement: Changing or resetting a password invalidates other active sessions
The system SHALL ensure that once a User's password has been changed
(whether via self-service change or a reset code), every access token
issued before that moment stops being accepted — except that the request
which performed the change or reset itself still receives a valid, newly
issued token to continue with.

#### Scenario: A session's own token still works right after a password change
- **WHEN** a User successfully changes or resets their password
- **THEN** the access token returned by that same request is accepted on
  the very next request

#### Scenario: A different, previously issued token stops working
- **WHEN** a User successfully changes or resets their password, and a
  request is later made with a different access token that was issued
  before that change
- **THEN** the system rejects that request as unauthorized, even though the
  token itself has not expired

## 1. Database

- [ ] 1.1 Add `CommunityPermission` enum (`PEOPLE_MANAGE`) and `permissions CommunityPermission[] @default([])` on `CommunityMembership`
- [ ] 1.2 Generate and apply the migration; confirm existing memberships get `{}`

## 2. Domain (`src/identity/domain/`)

- [ ] 2.1 `CommunityPermission` value object + `ALL_PERMISSIONS`
- [ ] 2.2 Extend `CommunityMembership` with `permissions`, `effectivePermissions()`, `hasPermission()`, `setPermissions()` (throws for ADMINISTRATOR)
- [ ] 2.3 Errors: `NotACommunityMemberError`, `MembershipNotFoundError`, `CannotChangeAdministratorPermissionsError`
- [ ] 2.4 Extend `MembershipRepository` port: `findByUserAndCommunity`, `findById`, `listByCommunityWithUsers`, `save`

## 3. Application (`src/identity/application/`)

- [ ] 3.1 `ResolveMembershipUseCase` (userId, communityId → membership or error)
- [ ] 3.2 `ListCommunityMembersUseCase`
- [ ] 3.3 `SetMemberPermissionsUseCase`
- [ ] 3.4 Include effective permissions in `LoginUseCase` and `GetCurrentUserUseCase` results

## 4. Infrastructure

- [ ] 4.1 Update `PrismaMembershipRepository` (map `permissions`, new queries)

## 5. Interface

- [ ] 5.1 `CommunityMemberGuard`, `@RequirePermission()`, `@RequireRole()`, `@CurrentMembership()`; export from `IdentityModule`
- [ ] 5.2 `MembersController`: `GET /communities/:communityId/members`, `PUT /communities/:communityId/members/:membershipId/permissions` with DTO validation (`IsEnum(each)`)
- [ ] 5.3 Map errors: non-member/insufficient permission → 403, membership not in Community → 404, Administrator target → 409
- [ ] 5.4 Swagger DTOs for the new endpoints and the extended login/me responses

## 6. Verification

- [ ] 6.1 Unit tests: effective permissions, `setPermissions` on Administrator
- [ ] 6.2 E2E: non-member 403, Administrator implicit permission, MEMBER with/without `PEOPLE_MANAGE`, permission per Community, set/revoke, Administrator target 409, foreign membership 404, login/me include permissions
- [ ] 6.3 lint, type-check, build clean

## 7. Documentation (this repo)

- [ ] 7.1 Accept `decisions/0007-community-scoped-requests-and-permissions.md`
- [ ] 7.2 Update `domain-model.md` and `glossary.md` if implementation deviates
- [ ] 7.3 Archive into `openspec/specs/identity/community-access/`

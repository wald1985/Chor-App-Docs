## 0. Prerequisite

- [ ] 0.1 `add-community-access-control` implemented (Community guard, `PEOPLE_MANAGE`)

## 1. Database

- [ ] 1.1 Add `PersonRole` enum and `Person` model (`nameKey`, `roles[]`, `archivedAt`, unique `(communityId, nameKey)`), relation on `Community`
- [ ] 1.2 Generate and apply migration

## 2. Domain (`src/people/domain/`)

- [ ] 2.1 `PersonRole`, `PersonName` (trim, 1–100, key) and `PersonRoles` (non-empty, deduplicated) value objects
- [ ] 2.2 `Person` aggregate: `create`, `rename`, `changeRoles`, `archive`, `restore`, `hasRole`; archived → not editable
- [ ] 2.3 Errors: `PersonNotFoundError`, `PersonNameTakenError` (with existing id/archived), `PersonArchivedError`, `PersonRolesEmptyError`
- [ ] 2.4 `PersonRepository` port: `findById(communityId, id)`, `findByNameKey`, `list(communityId, { role?, includeArchived })`, `save`

## 3. Application (`src/people/application/`)

- [ ] 3.1 `CreatePersonUseCase`
- [ ] 3.2 `UpdatePersonUseCase`
- [ ] 3.3 `ArchivePersonUseCase`, `RestorePersonUseCase`
- [ ] 3.4 `ListPeopleUseCase`, `GetPersonUseCase`

## 4. Infrastructure

- [ ] 4.1 `PrismaPersonRepository` + mapper; map unique violation (P2002) to `PersonNameTakenError`

## 5. Interface

- [ ] 5.1 DTOs: `CreatePersonDto` (`name`, `roles` `@ArrayNotEmpty @IsEnum each`), `UpdatePersonDto`, `ListPeopleQueryDto`, response DTO (Swagger)
- [ ] 5.2 `PeopleController` with endpoints from design.md, `CommunityMemberGuard`, `@RequirePermission(PEOPLE_MANAGE)` on write endpoints
- [ ] 5.3 Error mapping 400/403/404/409
- [ ] 5.4 `PeopleModule` mounted in `AppModule`

## 6. Verification

- [ ] 6.1 Unit tests: `PersonName`, `PersonRoles`, `Person` invariants (archive/restore idempotent, archived not editable)
- [ ] 6.2 E2E: create with both roles, empty/unknown roles, duplicate name (case/whitespace, archived, other Community), role filter includes dual-role Person, includeArchived, foreign Community 404, member without permission 403 on write / 200 on read, archive/restore
- [ ] 6.3 lint, type-check, build clean

## 7. Documentation (this repo)

- [ ] 7.1 Update `domain-model.md` / `glossary.md` if implementation deviates
- [ ] 7.2 Archive into `openspec/specs/people/person-management/`

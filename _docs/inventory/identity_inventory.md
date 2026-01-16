# Implementation Inventory: IdentityContext

## Meta

| Key | Value |
|-----|-------|
| **Context** | Identity |
| **作成日** | 2026-01-16 |
| **最終更新** | 2026-01-16 |
| **担当者** | - |
| **BDA Version** | 3.3.0 |
| **Design Files Hash** | (run: `git log -1 --format=%H -- sample/design/contexts/identity/`) |
| **Last Synced** | 2026-01-16 |

## Design Files Reference

| File | Description |
|------|-------------|
| `sample/design/contexts/identity/main.yaml` | Context 定義 |
| `sample/design/contexts/identity/domain.yaml` | Aggregate, Event, Repository, Domain Service |
| `sample/design/contexts/identity/application.yaml` | UseCase, Query |

## Domain Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| D-001 | Aggregate | User | domain/identity/entity.py | Unit | [ ] |
| D-002 | Value Object | UserId | domain/identity/value_objects.py | Unit | [ ] |
| D-003 | Value Object | Email | domain/identity/value_objects.py | Unit | [ ] |
| D-004 | Domain Event | UserRegistered | domain/identity/events.py | Unit | [ ] |
| D-005 | Domain Event | UserDeleted | domain/identity/events.py | Unit | [ ] |
| D-006 | Repository Interface | IUserRepository | domain/identity/repository.py | - | [ ] |
| D-007 | Domain Service | PasswordHashService | domain/identity/services.py | Unit | [ ] |
| D-008 | Domain Service | EmailUniquenessService | domain/identity/services.py | Unit | [ ] |

## Application Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| A-001 | UseCase | RegisterUserUseCase | application/identity/register_user.py | Unit | [ ] |
| A-002 | UseCase | LoginUseCase | application/identity/login.py | Unit | [ ] |
| A-003 | UseCase | ChangePasswordUseCase | application/identity/change_password.py | Unit | [ ] |
| A-004 | UseCase | DeactivateUserUseCase | application/identity/deactivate_user.py | Unit | [ ] |
| A-005 | Query | GetUserProfileQuery | application/identity/queries/user_profile.py | Unit | [ ] |
| A-006 | Query | GetUserByEmailQuery | application/identity/queries/user_by_email.py | Unit | [ ] |

## Infrastructure Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| I-001 | Repository Impl | UserRepositoryImpl | infrastructure/identity/repository_impl.py | Integration | [ ] |
| I-002 | ORM Model | UserORM | infrastructure/persistence/orm_models.py | Integration | [ ] |
| I-003 | Service Impl | PasswordHashServiceImpl | infrastructure/identity/password_service.py | Unit | [ ] |
| I-004 | Service Impl | JwtService | infrastructure/auth/jwt_service.py | Unit | [ ] |
| I-005 | Event Publisher | EventPublisher | infrastructure/event_publisher.py | Integration | [ ] |

## Interfaces Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| IF-001 | Controller | AuthController | interfaces/api/auth_controller.py | Integration | [ ] |
| IF-002 | DTO | UserDTO | interfaces/api/dto.py | - | [ ] |
| IF-003 | DTO | AuthTokenDTO | interfaces/api/dto.py | - | [ ] |
| IF-004 | DTO | RegisterRequest | interfaces/api/dto.py | - | [ ] |
| IF-005 | DTO | LoginRequest | interfaces/api/dto.py | - | [ ] |
| IF-006 | Middleware | JwtAuthMiddleware | interfaces/api/middleware.py | Integration | [ ] |

## Platform-Dependent Items

| ID | Category | Item | File | Verification Procedure | Status | 確認日 | 確認者 |
|----|----------|------|------|------------------------|--------|--------|--------|
| P-001 | Database | DB Migration (users table) | alembic/versions/*.py | `alembic upgrade head` が成功すること | [ ] | | |
| P-002 | Auth | JWT Token発行 | infrastructure/auth/ | トークンが正しく発行されること | [ ] | | |
| P-003 | Auth | JWT Token検証 | interfaces/api/middleware.py | 認証付きエンドポイントが保護されること | [ ] | | |
| P-004 | Security | Password Hashing | infrastructure/identity/ | bcrypt/argon2でハッシュ化されること | [ ] | | |

## UseCase / Query Coverage

`application.yaml` との対応:

| Name (from YAML) | Type | Implementation File | Test | Status |
|------------------|------|---------------------|------|--------|
| RegisterUser | UseCase | register_user.py | Unit | [ ] |
| Login | UseCase | login.py | Unit | [ ] |
| ChangePassword | UseCase | change_password.py | Unit | [ ] |
| DeactivateUser | UseCase | deactivate_user.py | Unit | [ ] |
| GetUserProfile | Query | queries/user_profile.py | Unit | [ ] |
| GetUserByEmail | Query | queries/user_by_email.py | Unit | [ ] |

## Domain Event Coverage

`domain.yaml` との対応:

| Behavior | Emits | Event Defined | Handler in Other Context | Status |
|----------|-------|---------------|--------------------------|--------|
| User.register | UserRegistered | Yes | - | [ ] |
| User.deactivate | UserDeleted | Yes | TodoContext.HandleUserDeleted | [ ] |

## Summary

| Layer | Total | Completed | Remaining |
|-------|-------|-----------|-----------|
| Domain | 8 | 0 | 8 |
| Application | 6 | 0 | 6 |
| Infrastructure | 5 | 0 | 5 |
| Interfaces | 6 | 0 | 6 |
| Platform-Dependent | 4 | 0 | 4 |
| **Total** | **29** | **0** | **29** |

## Completion Log

| Step | Completed | Date | Notes |
|------|-----------|------|-------|
| Step 2.5 Inventory作成 | [x] | 2026-01-16 | 自動生成 by /bda-inventory |
| Step 3 Skeleton作成 | [ ] | | |
| Step 5 Domain/Application完了 | [ ] | | |
| Step 6 Infrastructure/Interfaces完了 | [ ] | | |
| Step 9 Final確認 | [ ] | | |

## Change History (設計ファイル同期履歴)

| Date | Design Files Hash | Changes | Updated By |
|------|-------------------|---------|------------|
| 2026-01-16 | - | 初回作成（/bda-init + /bda-inventory） | Claude |

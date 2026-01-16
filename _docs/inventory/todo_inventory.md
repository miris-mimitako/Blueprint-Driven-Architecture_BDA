# Implementation Inventory: TodoContext

## Meta

| Key | Value |
|-----|-------|
| **Context** | Todo |
| **作成日** | 2026-01-16 |
| **最終更新** | 2026-01-16 |
| **担当者** | - |
| **BDA Version** | 3.3.0 |
| **Design Files Hash** | (run: `git log -1 --format=%H -- sample/design/contexts/todo/`) |
| **Last Synced** | 2026-01-16 |

## Design Files Reference

| File | Description |
|------|-------------|
| `sample/design/contexts/todo/main.yaml` | Context 定義 |
| `sample/design/contexts/todo/domain.yaml` | Aggregate, Event, Repository |
| `sample/design/contexts/todo/application.yaml` | UseCase, Query |

## Domain Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| D-001 | Aggregate | Todo | domain/todo/entity.py | Unit | [ ] |
| D-002 | Value Object | TodoId | domain/todo/value_objects.py | Unit | [ ] |
| D-003 | Value Object | UserId | domain/todo/value_objects.py | Unit | [ ] |
| D-004 | Enum | TodoStatus (PENDING, COMPLETED) | domain/todo/value_objects.py | Unit | [ ] |
| D-005 | Domain Event | TodoCompleted | domain/todo/events.py | Unit | [ ] |
| D-006 | Repository Interface | ITodoRepository | domain/todo/repository.py | - | [ ] |

## Application Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| A-001 | UseCase | CreateTodoUseCase | application/todo/create_todo.py | Unit | [ ] |
| A-002 | UseCase | CompleteTodoUseCase | application/todo/complete_todo.py | Unit | [ ] |
| A-003 | Query | GetDashboardStatsQuery | application/todo/queries/dashboard_stats.py | Unit | [ ] |

## Infrastructure Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| I-001 | Repository Impl | TodoRepositoryImpl | infrastructure/todo/repository_impl.py | Integration | [ ] |
| I-002 | ORM Model | TodoORM | infrastructure/persistence/orm_models.py | Integration | [ ] |
| I-003 | Event Publisher | EventPublisher | infrastructure/event_publisher.py | Integration | [ ] |

## Interfaces Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| IF-001 | Controller | TodoController | interfaces/api/todo_controller.py | Integration | [ ] |
| IF-002 | DTO | TodoDTO | interfaces/api/dto.py | - | [ ] |
| IF-003 | DTO | TodoCreateRequest | interfaces/api/dto.py | - | [ ] |
| IF-004 | DTO | DashboardStatsDTO | interfaces/api/dto.py | - | [ ] |
| IF-005 | ViewModel | useTodoList | frontend/hooks/useTodoList.ts | Unit | [ ] |
| IF-006 | View | DashboardPage | frontend/app/dashboard/page.tsx | UI | [ ] |

## Platform-Dependent Items

| ID | Category | Item | File | Verification Procedure | Status | 確認日 | 確認者 |
|----|----------|------|------|------------------------|--------|--------|--------|
| P-001 | Database | DB Migration | alembic/versions/*.py | `alembic upgrade head` が成功すること | [ ] | | |

## UseCase / Query Coverage

`application.yaml` との対応:

| Name (from YAML) | Type | Implementation File | Test | Status |
|------------------|------|---------------------|------|--------|
| CreateTodo | UseCase | create_todo.py | Unit | [ ] |
| CompleteTodo | UseCase | complete_todo.py | Unit | [ ] |
| GetDashboardStats | Query | queries/dashboard_stats.py | Unit | [ ] |

## Domain Event Coverage

`domain.yaml` との対応:

| Behavior | Emits | Event Defined | Handler Required | Status |
|----------|-------|---------------|------------------|--------|
| Todo.complete | TodoCompleted | Yes | - | [ ] |

## Summary

| Layer | Total | Completed | Remaining |
|-------|-------|-----------|-----------|
| Domain | 6 | 0 | 6 |
| Application | 3 | 0 | 3 |
| Infrastructure | 3 | 0 | 3 |
| Interfaces | 6 | 0 | 6 |
| Platform-Dependent | 1 | 0 | 1 |
| **Total** | **19** | **0** | **19** |

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
| 2026-01-16 | - | 初回作成（/bda-inventory による自動生成） | Claude |

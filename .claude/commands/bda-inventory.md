# /bda-inventory - Implementation Inventory 自動生成

設計ファイル（domain.yaml, application.yaml）からImplementation Inventoryを自動生成します。

## 引数

```
/bda-inventory {context_name}
```

- `context_name`: 対象のContext名（例: `todo`, `identity`）

## 実行手順

### 1. 設計ファイルの読み込み

```
design/contexts/{context_name}/
├── main.yaml
├── domain.yaml
└── application.yaml
```

上記ファイルを読み込み、以下の情報を抽出します。

### 2. Domain Layer Items の抽出

`domain.yaml` から:

```yaml
# 入力例
aggregates:
  - name: Todo
    root: true
    properties:
      - { name: id, type: TodoId }
      - { name: title, type: String }
    behaviors:
      - { name: complete, emits: TodoCompleted }

events:
  - name: TodoCompleted
    properties: { todoId: TodoId, userId: UserId }

repositories:
  - name: TodoRepository
    methods:
      - { name: save, input: Todo, output: Todo }
```

```markdown
# 出力例
## Domain Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| D-001 | Aggregate | Todo | domain/todo/entity.py | Unit | [ ] |
| D-002 | Value Object | TodoId | domain/todo/value_objects.py | Unit | [ ] |
| D-003 | Domain Event | TodoCompleted | domain/todo/events.py | Unit | [ ] |
| D-004 | Repository Interface | TodoRepository | domain/todo/repository.py | - | [ ] |
```

**抽出ルール:**
- `aggregates[].name` → Aggregate
- `aggregates[].properties[].type` から ValueObject を抽出（型名に Id, Title 等が含まれる場合）
- `events[].name` → Domain Event
- `repositories[].name` → Repository Interface
- `services[].name` → Domain Service（存在する場合）

### 3. Application Layer Items の抽出

`application.yaml` から:

```yaml
# 入力例
useCases:
  - name: CreateTodo
    input: { title: String, userId: UserId }
    output: TodoDTO
    flow: "Factory.create -> Repo.save"

  - name: CompleteTodo
    input: { todoId: TodoId, userId: UserId }
    output: TodoDTO
    flow: "Repo.find -> Todo.complete() -> Repo.save -> Publish"

queries:
  - name: GetDashboardStats
    input: { userId: UserId }
    output: DashboardStatsDTO
```

```markdown
# 出力例
## Application Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| A-001 | UseCase | CreateTodoUseCase | application/todo/create_todo.py | Unit | [ ] |
| A-002 | UseCase | CompleteTodoUseCase | application/todo/complete_todo.py | Unit | [ ] |
| A-003 | Query | GetDashboardStatsQuery | application/todo/queries/dashboard_stats.py | Unit | [ ] |
```

**抽出ルール:**
- `useCases[].name` → UseCase（ファイル名は snake_case に変換）
- `queries[].name` → Query
- `eventHandlers[].name` → Event Handler（存在する場合）

### 4. Infrastructure / Interfaces Layer Items の生成

Domain Layer から推論:

```markdown
## Infrastructure Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| I-001 | Repository Impl | TodoRepositoryImpl | infrastructure/todo/repository_impl.py | Integration | [ ] |
| I-002 | ORM Model | TodoORM | infrastructure/persistence/orm_models.py | Integration | [ ] |

## Interfaces Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| IF-001 | Controller | TodoController | interfaces/api/todo_controller.py | Integration | [ ] |
| IF-002 | DTO | TodoDTO, TodoCreateRequest | interfaces/api/dto.py | - | [ ] |
```

### 5. Inventory ファイルの生成

出力先: `_docs/inventory/{context_name}_inventory.md`

テンプレート（codingStepSpecification.md のセクション7参照）に従って生成。

### 6. Summary の計算

```markdown
## Summary

| Layer | Total | Completed | Remaining |
|-------|-------|-----------|-----------|
| Domain | 4 | 0 | 4 |
| Application | 3 | 0 | 3 |
| Infrastructure | 2 | 0 | 2 |
| Interfaces | 2 | 0 | 2 |
| Platform-Dependent | 0 | 0 | 0 |
| **Total** | **11** | **0** | **11** |
```

---

## 使用例

```
/bda-inventory todo
```

`design/contexts/todo/` の設計ファイルを読み込み、
`_docs/inventory/todo_inventory.md` を生成します。

## 注意事項

- 既存のInventoryファイルがある場合は上書き確認を行う
- `design/` が存在しない場合は `sample/design/` を参照
- 生成後、`git log -1 --format=%H -- design/contexts/{context}/` でハッシュを取得し、Metaに記録

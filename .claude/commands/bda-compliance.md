# /bda-compliance - DDDアーキテクチャ準拠チェック

コードベースがDDD/Clean Architectureの原則に従っているかを検証します。

## 引数

```
/bda-compliance
/bda-compliance {context_name}
/bda-compliance --fix-suggestions
```

## 検証ルール

### 1. レイヤー依存方向 (Dependency Rule)

**原則:** 依存は外側から内側へのみ許可。Domain Layerは他のどのレイヤーにも依存してはならない。

```
Interfaces → Application → Domain ← Infrastructure
     ↓            ↓           ↑           ↑
     └────────────┴───────────┴───────────┘
                 依存方向 (内向き)
```

#### 検出パターン

```python
# ❌ VIOLATION: Domain が Infrastructure に依存
# File: domain/todo/entity.py
from infrastructure.persistence.orm_models import TodoORM  # NG!

# ❌ VIOLATION: Domain が Application に依存
# File: domain/todo/entity.py
from application.todo.dto import TodoDTO  # NG!

# ✅ OK: Application が Domain に依存
# File: application/todo/create_todo.py
from domain.todo.entity import Todo  # OK
from domain.todo.repository import ITodoRepository  # OK
```

#### チェックロジック

```python
LAYER_ORDER = {
    "domain": 0,      # 最内層
    "application": 1,
    "infrastructure": 2,
    "interfaces": 2,  # Infrastructure と同レベル
}

def check_import(file_path: str, import_statement: str) -> Violation | None:
    source_layer = get_layer(file_path)
    target_layer = get_layer(import_statement)

    # 内側のレイヤーが外側に依存していたら違反
    if LAYER_ORDER[source_layer] < LAYER_ORDER[target_layer]:
        return Violation(
            rule="DependencyDirection",
            file=file_path,
            message=f"{source_layer} cannot depend on {target_layer}"
        )
    return None
```

---

### 2. Repository Interface 分離

**原則:** Repository InterfaceはDomain Layer、実装はInfrastructure Layer。

#### 検出パターン

```python
# ❌ VIOLATION: Domain に Repository 実装がある
# File: domain/todo/repository.py
class TodoRepository:  # 実装クラス
    def __init__(self, session: AsyncSession):  # DB依存
        self._session = session

# ✅ OK: Domain には Interface のみ
# File: domain/todo/repository.py
from abc import ABC, abstractmethod

class ITodoRepository(ABC):
    @abstractmethod
    async def save(self, entity: Todo) -> Todo:
        raise NotImplementedError()

# ✅ OK: Infrastructure に実装
# File: infrastructure/todo/repository_impl.py
class TodoRepositoryImpl(ITodoRepository):
    def __init__(self, session: AsyncSession):
        self._session = session
```

#### チェックロジック

```python
def check_repository_separation(file_path: str) -> list[Violation]:
    violations = []

    if "domain" in file_path and "repository" in file_path:
        content = read_file(file_path)

        # ABC を継承していない Repository クラスは違反
        if re.search(r"class \w+Repository[^(]*:", content):
            if "ABC" not in content and "abstractmethod" not in content:
                violations.append(Violation(
                    rule="RepositorySeparation",
                    file=file_path,
                    message="Repository in Domain must be abstract (ABC)"
                ))

        # SQLAlchemy等の import があれば違反
        if "sqlalchemy" in content or "AsyncSession" in content:
            violations.append(Violation(
                rule="RepositorySeparation",
                file=file_path,
                message="Domain Repository cannot import ORM libraries"
            ))

    return violations
```

---

### 3. Aggregate 境界

**原則:** Aggregate間の直接参照は禁止。IDでのみ参照する。

#### 検出パターン

```python
# ❌ VIOLATION: 他のAggregate を直接参照
# File: domain/todo/entity.py
from domain.identity.entity import User  # NG!

@dataclass
class Todo:
    owner: User  # NG! 直接参照

# ✅ OK: ID で参照
from domain.identity.value_objects import UserId

@dataclass
class Todo:
    owner_id: UserId  # OK! ID参照
```

#### チェックロジック

```python
def check_aggregate_boundary(file_path: str) -> list[Violation]:
    violations = []

    if "domain" in file_path and "entity" in file_path:
        content = read_file(file_path)
        context = extract_context(file_path)  # e.g., "todo"

        # 他のContext の entity を import していたら違反
        other_entity_imports = re.findall(
            r"from domain\.(\w+)\.entity import",
            content
        )
        for other_context in other_entity_imports:
            if other_context != context:
                violations.append(Violation(
                    rule="AggregateBoundary",
                    file=file_path,
                    message=f"Cannot import entity from {other_context} context. Use ID reference instead."
                ))

    return violations
```

---

### 4. Domain Event 発行

**原則:** Domain Eventは Aggregate の振る舞いから発行する。Application Layerで直接生成しない。

#### 検出パターン

```python
# ❌ VIOLATION: Application で Event を直接生成
# File: application/todo/complete_todo.py
from domain.todo.events import TodoCompleted

class CompleteTodoUseCase:
    async def execute(self, input):
        todo = await self.repo.find(input.todo_id)
        todo.status = "COMPLETED"  # 直接変更
        event = TodoCompleted(...)  # Application で生成 NG!
        await self.publisher.publish(event)

# ✅ OK: Aggregate の振る舞いから Event を取得
class CompleteTodoUseCase:
    async def execute(self, input):
        todo = await self.repo.find(input.todo_id)
        event = todo.complete()  # Aggregate の振る舞いが Event を返す
        await self.repo.save(todo)
        await self.publisher.publish(event)
```

---

### 5. UseCase 単一責任

**原則:** 1つのUseCaseは1つのユースケースのみを担当する。

#### 検出パターン

```python
# ❌ VIOLATION: 複数の責任を持つ UseCase
# File: application/todo/todo_service.py
class TodoService:  # "Service" という名前は警告
    def create_todo(self, ...): ...
    def complete_todo(self, ...): ...
    def delete_todo(self, ...): ...
    def get_stats(self, ...): ...

# ✅ OK: 単一責任
# File: application/todo/create_todo.py
class CreateTodoUseCase:
    async def execute(self, input: CreateTodoInput) -> TodoDTO: ...

# File: application/todo/complete_todo.py
class CompleteTodoUseCase:
    async def execute(self, input: CompleteTodoInput) -> TodoDTO: ...
```

---

### 6. DTO の配置

**原則:** DTOはApplication Layer または Interfaces Layerに配置。Domain Layerには置かない。

#### 検出パターン

```python
# ❌ VIOLATION: Domain に DTO がある
# File: domain/todo/dto.py  # NG! ファイルの存在自体が違反

# ❌ VIOLATION: Domain Entity に to_dto メソッド
# File: domain/todo/entity.py
class Todo:
    def to_dto(self) -> TodoDTO:  # NG!
        return TodoDTO(...)

# ✅ OK: Application の DTO に変換ロジック
# File: application/todo/dto.py
@dataclass
class TodoDTO:
    @classmethod
    def from_entity(cls, entity: Todo) -> "TodoDTO":
        return cls(id=entity.id, ...)
```

---

## レポート出力

```markdown
# BDA Compliance Report

**Generated:** YYYY-MM-DD HH:MM
**Files Checked:** 45
**Violations:** 3
**Warnings:** 5

---

## 🔴 Violations (Must Fix)

### 1. DependencyDirection

**File:** `domain/todo/entity.py:5`
**Message:** Domain cannot depend on infrastructure
**Code:**
```python
from infrastructure.persistence.orm_models import TodoORM  # ← Remove this
```
**Fix:** Remove import. Domain should not know about ORM.

### 2. RepositorySeparation

**File:** `domain/todo/repository.py:12`
**Message:** Repository in Domain must be abstract (ABC)
**Fix:** Convert to abstract class:
```python
from abc import ABC, abstractmethod

class ITodoRepository(ABC):
    @abstractmethod
    async def save(self, entity: Todo) -> Todo:
        raise NotImplementedError()
```

---

## 🟡 Warnings (Should Fix)

### 1. NamingConvention

**File:** `application/todo/todo_service.py`
**Message:** "Service" suffix suggests multiple responsibilities
**Suggestion:** Split into separate UseCase classes

---

## ✅ Passed Rules

- [x] Aggregate Boundary (no cross-context entity imports)
- [x] Domain Event Publishing (events from aggregate behaviors)
- [x] DTO Placement (no DTOs in domain layer)

---

## Architecture Overview

```
Layers Dependency Graph (should be acyclic, inward only):

    interfaces ──→ application ──→ domain
         │              │              ↑
         │              │              │
         └──────────────┴── infrastructure
                              (implements domain interfaces)

Current Status: ✅ No circular dependencies detected
```
```

---

## オプション

| Option | Description |
|--------|-------------|
| `--fix-suggestions` | 修正提案をコード付きで表示 |
| `--strict` | Warnings も Violations として扱う |
| `--ignore {rule}` | 特定ルールを無視 |
| `--ci` | CI用出力（exit code: 0=pass, 1=violations） |

## 設定ファイル

`.bda-compliance.yaml` で除外設定:

```yaml
ignore:
  - rule: NamingConvention
    path: "legacy/*"

  - rule: DependencyDirection
    path: "infrastructure/migrations/*"

custom_rules:
  max_usecase_methods: 1
  require_docstrings: true
```

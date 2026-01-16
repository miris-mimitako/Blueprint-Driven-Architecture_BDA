# Blueprint-Driven Architecture (BDA) Specification

## Version: 3.3.0 (Modular DDD)

### Core Philosophy Update

BDA 3.3.0では、**「参照の透明性（Reference Transparency）」**を導入します。
巨大な一枚岩の仕様書はメンテナンス性を損ないます。各コンテキストやレイヤーを物理ファイルとして分離し、`$ref` で結合することを強制します。
また、AIがリンク先の中身を予測できるよう、**必ず `note` フィールドで概要を記述しなければなりません。**

### Modularization Rules (New)

1. **Separation**: Context, Layer, Feature 単位でファイルを分割する。
2. **Referencing**: 分割したファイルは `$ref: "./relative/path.yaml"` で読み込む。
3. **Annotation**: `$ref` を使用する際は、**必ず `note: "..."` を併記**し、リンク先のファイルを開かなくても役割がわかるようにする。

---

## 1. Root Blueprint (Entry Point)

プロジェクトのルートに配置されるメインファイルです。ここには詳細を書かず、全体構造とファイルへのポインタのみを記述します。

```yaml
blueprint: "3.3.0"

meta:
  appName: "NextFastTodo"
  version: "1.0.0"
  architecture: "Clean Architecture / DDD"

guidelines:
  # 技術スタック定義も別ファイルへ切り出し
  $ref: "./guidelines/tech_stack.yaml"
  note: "Backend(FastAPI), Frontend(Next.js), Infrastructure rules."

# =========================================================
# 1. BOUNDED CONTEXTS (Domain & Application)
# =========================================================
contexts:
  - $ref: "./contexts/identity/main.yaml"
    note: "【認証・ID管理】ユーザー登録、ログイン、権限管理を行うコンテキスト"

  - $ref: "./contexts/todo/main.yaml"
    note: "【タスク管理】TodoのCRUD、ステータス変更、期限管理を行うコア機能"

# =========================================================
# 2. WORKFLOWS (Sagas)
# =========================================================
workflows:
  - $ref: "./workflows/onboarding_saga.yaml"
    note: "ユーザー登録からウェルカムメール送信、クーポン発行までの長期トランザクション"

# =========================================================
# 3. INTERFACES (Adapters)
# =========================================================
interfaces:
  backend:
    controllers:
      - $ref: "./interfaces/backend/todo_controller.yaml"
        note: "REST API definition for Todo operations"
  
  frontend:
    features:
      - $ref: "./interfaces/frontend/dashboard_feature.yaml"
        note: "Todo Dashboard UI, ViewModel, and Scenarios"

    routing:
      $ref: "./interfaces/frontend/routing.yaml"
      note: "Frontend navigation structure and auth guards"

# =========================================================
# 4. INFRASTRUCTURE & ARTIFACTS
# =========================================================
infrastructure:
  $ref: "./infrastructure/config.yaml"
  note: "DB settings, ACL definitions, and external adapters"

artifacts:
  - type: openapi
    path: "./docs/openapi.yaml"
    source: ["contexts.*", "interfaces.backend"]
    note: "Generate OpenAPI 3.1 spec from distributed context files"

```

---

## 2. Child File Examples

分割されたファイルの記述例です。

### A. Context Definition

File: `/contexts/todo/main.yaml`

```yaml
# Todo Context Definition
name: TodoContext

domain:
  # ドメインモデル定義
  $ref: "./domain.yaml"
  note: "Aggregates(Todo), Events(TodoCompleted), Domain Services"

application:
  # ユースケース定義
  $ref: "./application.yaml"
  note: "UseCases(Create, Complete), Queries(GetStats)"

```

### B. Domain Layer

File: `/contexts/todo/domain.yaml`

```yaml
aggregates:
  - name: Todo
    root: true
    properties:
      - { name: id, type: TodoId }
      - { name: title, type: String }
      - { name: status, type: Enum[PENDING, COMPLETED] }
    behaviors:
      - { name: complete, emits: TodoCompleted, desc: "Mark as done" }

events:
  - name: TodoCompleted
    properties: { todoId: TodoId, occurredAt: DateTime }

repositories:
  - name: TodoRepository
    methods:
      - { name: save, input: Todo, output: Todo }

```

### C. Feature (Frontend)

File: `/interfaces/frontend/dashboard_feature.yaml`

```yaml
id: TodoDashboardFeature
name: DashboardPage
path: "/dashboard"

state:
  - { name: todos, type: "List<TodoDTO>" }

inputs:
  - { name: onAddTodo, args: [{name: title, type: String}] }

scenarios:
  - name: "Initial Load"
    given: { state: { todos: [] } }

```

---

## 3. Revised Directory Structure

`$ref` のパスと一致するディレクトリ構成です。

```md
/design
  ├── blueprint.yaml                # [Root] エントリーポイント
  │
  ├── guidelines/
  │   └── tech_stack.yaml           # 技術スタック定義
  │
  ├── contexts/                     # [Bounded Contexts]
  │   ├── identity/
  │   │   ├── main.yaml             # Identity Context Root
  │   │   ├── domain.yaml
  │   │   └── application.yaml
  │   │
  │   └── todo/
  │       ├── main.yaml             # Todo Context Root
  │       ├── domain.yaml
  │       └── application.yaml
  │
  ├── workflows/                    # [Sagas]
  │   └── onboarding_saga.yaml
  │
  ├── interfaces/                   # [Adapters]
  │   ├── backend/
  │   │   └── todo_controller.yaml
  │   └── frontend/
  │       ├── dashboard_feature.yaml
  │       └── routing.yaml
  │
  └── infrastructure/               # [Infra]
      └── config.yaml

```

### この構成のメリット

1. **AIのトークン節約**: AIは最初に `blueprint.yaml` だけを読み、「Todoのドメインルールを変更したい」という指示があった場合のみ、`note` を手がかりに `/contexts/todo/domain.yaml` を読みに行きます。
2. **人間への可読性**: ファイルを開かなくても `note` を見れば、「ここに何が書いてあるか」が一目瞭然です。
3. **Git競合の回避**: 巨大なYAMLファイルを修正するのではなく、小さなファイルを修正するため、チーム開発時のコンフリクトが激減します。
# コーディング実装規約 (BDA v3.3.0)

**Version:** 3.1.0
**Architecture:** Blueprint-Driven Architecture (Modular DDD)
**Last Updated:** 2026-01-16

### Change Log
| Version | Date | Changes |
|---------|------|---------|
| 3.1.0 | 2026-01-16 | 既存機能連携チェック、UI詳細仕様、非機能要件、Inventory再生成ルール、Deprecation Warning対応、設計レビューチェックリストを追加 |
| 3.0.0 | - | 初版 |

## 1. 概要 (Overview)

本規約は、**「手戻りの完全排除」**と**「AIとの協働効率の最大化」**を目的とした開発標準である。
開発プロセスは厳格な **8ステップ** で構成され、各工程の「完了の定義 (Definition of Done)」を満たさない限り、次の工程へ進むことは許されない。

### Core Philosophy

1. **Blueprint is the App:** `blueprint.yaml` と分割された設計ファイル群こそがアプリケーションの正体（Identity）であり、コードはその出力結果（Artifact）に過ぎない。
2. **Domain First, Infrastructure Second:** ビジネスドメイン（Aggregate, Event, UseCase）を先に確定させ、技術的詳細（DB, API）は後から決定する。
3. **Reference Transparency:** 設計ファイルは `$ref` + `note` パターンで分割し、AIが部分的に読み込んでも全体像を把握できる状態を維持する。
4. **Context-Driven Development:** Bounded Context単位で設計・実装・テストを行い、Context間は疎結合（Event駆動）で連携する。
5. **No Item Left Behind:** テスト困難な項目も含め、全実装項目を **Implementation Inventory** で追跡し、漏れを防ぐ。

---

## 2. アーキテクチャ定義 (Architecture Definition)

BDA 3.3.0 では **Clean Architecture + DDD** を採用し、Bounded Context を中心にアプリケーションを構成する。

### 2.1 レイヤー構成 (DDD Layers)

```
┌─────────────────────────────────────────────────────────────┐
│                      Interfaces Layer                        │
│   (Controllers, ViewModels, API Handlers)                   │
├─────────────────────────────────────────────────────────────┤
│                     Application Layer                        │
│   (Use Cases, Queries, Event Handlers)                      │
├─────────────────────────────────────────────────────────────┤
│                       Domain Layer                           │
│   (Aggregates, Entities, Value Objects, Domain Events,      │
│    Domain Services, Repository Interfaces)                  │
├─────────────────────────────────────────────────────────────┤
│                   Infrastructure Layer                       │
│   (Repository Impl, DB, External APIs, Platform Services)   │
└─────────────────────────────────────────────────────────────┘
         ↑ 依存方向: 外側から内側へ（Domain Layer に依存）
```

| Layer | Component | Responsibility | Testability |
|-------|-----------|----------------|-------------|
| **Domain** | Aggregate, Entity, Value Object, Domain Event, Domain Service | ビジネスルールの表現。技術詳細に依存しない。 | Unit Test |
| **Application** | UseCase, Query, Event Handler | ユースケースの実行、トランザクション境界の管理 | Unit Test |
| **Interfaces** | Controller (REST), ViewModel (Frontend) | 外部との入出力変換、プレゼンテーション | Integration Test |
| **Infrastructure** | Repository Impl, DB Adapter, Platform Service | 永続化、外部API連携、OS固有機能 | Integration / E2E |

### 2.2 Bounded Context について

各 Bounded Context は独立した業務領域を表し、以下の構造を持つ。

```yaml
# contexts/{context_name}/main.yaml
name: TodoContext
description: "タスク管理の中核機能"

domain:
  $ref: "./domain.yaml"
  note: "Aggregates, Events, Domain Services, Repository Interfaces"

application:
  $ref: "./application.yaml"
  note: "UseCases, Queries, Event Handlers"
```

### 2.3 Context 間連携

Context 間は **Domain Event** を介して疎結合に連携する。直接の依存は禁止。

```
IdentityContext                    TodoContext
┌─────────────────┐               ┌─────────────────┐
│  User.delete()  │               │                 │
│       ↓         │               │                 │
│  UserDeleted    │ ─── Event ──→ │ HandleUserDeleted│
│    (publish)    │   (疎結合)    │       ↓         │
└─────────────────┘               │ TodoCleanupService│
                                  └─────────────────┘
```

### 2.4 Platform Layer について

**Infrastructure Layer** 内の **Platform Service** はOS/フレームワーク固有の機能を担当し、**自動テストが困難**な領域である。

| Platform項目 | Android例 | iOS例 | Web例 |
|--------------|-----------|-------|-------|
| バックグラウンド処理 | Foreground Service | Background Tasks | Service Worker |
| 通知 | NotificationManager | UNUserNotificationCenter | Notification API |
| ライフサイクル | ProcessLifecycleOwner | AppDelegate | visibilitychange |
| 権限 | Permission API | Info.plist | Permission API |

**重要:** Platform 項目は **Implementation Inventory** で明示的に追跡し、Step 6 で確実に実装する。

---

## 3. 開発ワークフロー (The 8-Step Workflow)

開発者は必ず以下の順序でタスクを遂行すること。

### [Step 1] 要件定義 (Requirement Definition)

自然言語で機能の目的とスコープを確定させる。**どの Bounded Context に属するか**も決定する。

* **Action:**
  - 自然言語による機能記述
  - ユーザーアクションとシステムレスポンスのリストアップ
  - 対象 Context の特定（新規 or 既存）
* **DoD:** `meta` 情報や既存の `guidelines` と矛盾せず、作るべき機能が言語化されていること。

#### 既存機能との連携チェック（重要）

新機能を追加する際は、**既存のBounded Contextとの連携**を必ず検討すること。

```yaml
# blueprint.yaml に追加すべき例
cross_context_requirements:
  - from: shopping
    to: manufacturer
    integration: "アイテム登録時にメーカーを自動登録"
    direction: "Shopping → Manufacturer"
```

| チェック項目 | 確認内容 |
|-------------|---------|
| データの関連性 | 新Contextのデータは既存Contextと関連するか？ |
| 機能の連携 | 新機能は既存機能と連動すべきか？ |
| マスターデータ | 新しいマスターデータは他機能から参照されるか？ |
| 自動登録 | ユーザー入力を他のマスターに自動反映すべきか？ |

#### Completion Gate 1
- [ ] 機能の目的が1文で説明できる
- [ ] 対象の Bounded Context が特定されている
- [ ] ユーザーアクションが全て列挙されている
- [ ] システムレスポンス（正常系・異常系）が定義されている
- [ ] **既存Contextとの連携要件が明確になっている**

### [Step 2] ドメインモデル確定 (Blueprint Definition)

BDAの核となる工程。**Bounded Context 単位**でYAMLを作成し、仕様をFIXさせる。

* **Action:**
  1. **Domain Layer**: `contexts/{name}/domain.yaml` に Aggregate, Event, Repository Interface を記述
  2. **Application Layer**: `contexts/{name}/application.yaml` に UseCase, Query を記述
  3. **Interfaces Layer**: `interfaces/` に Controller, ViewModel を記述
  4. 各ファイルは `$ref` + `note` で参照を明示

* **DoD:** プログラミング言語を使わずに、YAMLだけで「ドメインの振る舞い」が脳内再生できる状態。

#### 設計ファイル構成例

```
design/
├── blueprint.yaml              # Entry Point
├── guidelines/tech_stack.yaml  # 技術スタック
├── contexts/
│   └── todo/
│       ├── main.yaml           # Context定義（$ref集約）
│       ├── domain.yaml         # Aggregate, Event, Repository
│       └── application.yaml    # UseCase, Query
└── interfaces/
    ├── backend/todo_controller.yaml
    └── frontend/dashboard.yaml
```

#### UI/UX 詳細仕様の記載（重要）

`interfaces/` の設計ファイルには、UIコンポーネントの**振る舞いの詳細**を記載すること。
曖昧な仕様は実装時の手戻りの原因となる。

```yaml
# interfaces/mobile/add_item_dialog.yaml の例
components:
  manufacturer_input:
    type: AutoCompleteTextField
    behavior:
      filter: "部分一致（大文字小文字無視）"
      max_suggestions: 10
      show_on_focus: true
      empty_state: "候補なし"
    validation:
      required: false
      max_length: 100
```

| 記載すべき項目 | 例 |
|---------------|-----|
| 入力フィールドの種類 | TextField, AutoCompleteTextField, Dropdown |
| フィルタリング方法 | 前方一致、部分一致、大文字小文字の扱い |
| 表示件数上限 | max_suggestions: 10 |
| 空状態の表示 | "候補がありません" |
| バリデーションルール | required, max_length, pattern |

#### Completion Gate 2
- [ ] `domain.yaml` に Aggregate（properties, behaviors）が定義されている
- [ ] `domain.yaml` に Domain Event（properties）が定義されている
- [ ] `domain.yaml` に Repository Interface（methods）が定義されている
- [ ] `application.yaml` に UseCase（input, output, flow）が定義されている
- [ ] `application.yaml` に Query（input, output, strategy）が定義されている（読み取り専用操作がある場合）
- [ ] `interfaces/` に Controller または ViewModel が定義されている
- [ ] **`interfaces/` にUI振る舞いの詳細が記載されている**
- [ ] **非機能要件が検討・記載されている**
- [ ] 全ての `$ref` に `note` が付与されている

#### 非機能要件の考慮（重要）

設計ファイルには**非機能要件**も明記すること。実装後に発覚すると大きな手戻りとなる。

```yaml
# contexts/{context}/main.yaml または guidelines/ に記載
non_functional_requirements:
  performance:
    - "メーカー候補は最大100件まで表示"
    - "サジェスト表示は100ms以内"
  offline:
    - "オフライン時はローカルキャッシュから候補を表示"
  scalability:
    - "メーカーマスターは10,000件まで対応"
  security:
    - "入力値は必ずサニタイズ"
```

| カテゴリ | 検討項目 |
|---------|---------|
| Performance | 大量データ時のレスポンス、ページネーション |
| Offline | オフライン時の挙動、キャッシュ戦略 |
| Scalability | データ量の上限、将来の拡張性 |
| Security | 入力値検証、認証・認可 |
| Usability | エラーメッセージ、ローディング表示 |

### [Step 2.5] Implementation Inventory 作成 (実装項目の棚卸し)

**設計ファイル群から全実装項目を抽出し、Context 単位で追跡可能な形式で整理する。**

* **Action:**
  1. 対象 Context の `domain.yaml`, `application.yaml` から実装項目を抽出
  2. 各項目を **DDD Layer 別** に分類（Domain / Application / Infrastructure / Interfaces）
  3. `_docs/inventory/{context}_inventory.md` に Inventory ファイルを作成
  4. 設計ファイルのバージョン/ハッシュを Inventory に記録

* **Output:** 以下の形式で `_docs/inventory/{context}_inventory.md` を作成

```markdown
# Implementation Inventory: {Context名}

## Meta
- **Context:** {ContextName}
- **Blueprint Version:** 3.3.0
- **Design Files Hash:** {git hash}
- **Last Synced:** YYYY-MM-DD HH:MM

## Domain Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| D-001 | Aggregate | Todo | todo/entity.py | Unit | [ ] |
| D-002 | Value Object | TodoId, TodoTitle | todo/value_objects.py | Unit | [ ] |
| D-003 | Domain Event | TodoCompleted | todo/events.py | Unit | [ ] |
| D-004 | Repository Interface | ITodoRepository | todo/repository.py | - | [ ] |
| D-005 | Domain Service | TodoCleanupService | todo/services.py | Unit | [ ] |

## Application Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| A-001 | UseCase | CreateTodoUseCase | usecases/create_todo.py | Unit | [ ] |
| A-002 | UseCase | CompleteTodoUseCase | usecases/complete_todo.py | Unit | [ ] |
| A-003 | Query | GetDashboardStatsQuery | queries/dashboard_stats.py | Unit | [ ] |
| A-004 | Event Handler | HandleUserDeletedHandler | handlers/user_deleted.py | Unit | [ ] |

## Infrastructure Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| I-001 | Repository Impl | TodoRepositoryImpl | infra/todo_repository.py | Integration | [ ] |
| I-002 | DB Mapping | Todo <-> todos table | infra/orm_models.py | Integration | [ ] |

## Interfaces Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| IF-001 | Controller | TodoController | api/todo_controller.py | Integration | [ ] |
| IF-002 | ViewModel | useTodoList | hooks/useTodoList.ts | Unit | [ ] |
| IF-003 | View | DashboardPage | pages/dashboard.tsx | UI | [ ] |

## Platform-Dependent Items (プラットフォーム依存)

| ID | Category | Item | File | Verification Procedure | Status |
|----|----------|------|------|------------------------|--------|
| P-001 | Background | Service Worker | sw.js | オフライン時にアプリが動作すること | [ ] |
| P-002 | Notification | Push通知 | notification.ts | タスク完了時に通知が表示されること | [ ] |

## Summary
- Domain: X items
- Application: Y items
- Infrastructure: Z items
- Interfaces: W items
- Platform-Dependent: V items (手動確認必須)
```

* **DoD:** Context 内の全設計ファイル（domain.yaml, application.yaml）から漏れなく項目が抽出されていること。

#### Design Files - Inventory 同期ルール

**重要:** 設計ファイル（`contexts/*/domain.yaml` 等）が更新された場合、以下の手順で Inventory を同期すること。

1. `git diff design/` で変更内容を確認
2. 追加された項目を Inventory に追記
3. 削除された項目を Inventory から削除（または ~~取り消し線~~ でマーク）
4. Inventory の `Design Files Hash` と `Last Synced` を更新

```bash
# 設計ファイルのハッシュ取得コマンド例
git log -1 --format=%H -- design/contexts/todo/
```

#### 機能追加時の Inventory 再生成ルール（重要）

**既存Contextに機能を追加した場合**、必ず `/bda-inventory` を再実行し、差分を明確化すること。

| シナリオ | 対応 |
|---------|------|
| 新規Contextの追加 | `/bda-inventory {new_context}` で新規作成 |
| 既存Contextへの機能追加 | `/bda-inventory {context}` で差分を確認・追記 |
| Context間連携の追加 | 両方のContextの Inventory を更新 |
| 設計ファイルの修正 | 該当 Inventory を同期 |

```bash
# 機能追加後の推奨フロー
1. 設計ファイル（domain.yaml, application.yaml）を更新
2. /bda-inventory {context} を実行
3. 差分を確認し、実装計画を立てる
4. 実装を進める
```

#### Completion Gate 2.5
- [ ] `domain.yaml` の全 Aggregate が Inventory に記載されている
- [ ] `domain.yaml` の全 Domain Event が Inventory に記載されている
- [ ] `domain.yaml` の全 Repository Interface が Inventory に記載されている
- [ ] `application.yaml` の全 UseCase が Inventory に記載されている
- [ ] `application.yaml` の全 Query が Inventory に記載されている
- [ ] Platform-Dependent Items が明示的に分離されている
- [ ] 各項目に対応ファイル名が記載されている
- [ ] **Platform-Dependent Items に具体的な確認手順が記載されている**
- [ ] **設計ファイルのハッシュが Inventory に記録されている**

### [Step 3] 詳細設計 (Detailed Design & Skeleton)

DDD/Clean Architecture に基づき、各レイヤーの「枠」を作る。

* **Action:**
  1. **Domain Layer:**
     - Aggregate クラス（Entity + Value Object）のスケルトン
     - Domain Event の定義
     - Repository Interface の定義
  2. **Application Layer:**
     - UseCase クラスのスケルトン（`raise NotImplementedError`）
     - Query クラスのスケルトン
  3. **Infrastructure Layer:**
     - Repository 実装のスケルトン
  4. **Interfaces Layer:**
     - Controller / ViewModel のスケルトン
  5. **Docstrings:** 全メソッドへのドキュメント記述（引数・戻り値・例外）
  6. **DI:** コンストラクタによる依存性注入の構造化

* **DoD:** 実装コードが一行もなくても、インターフェース定義のみでテストコードが書き始められる状態であること。

#### ディレクトリ構成例（Backend: Python/FastAPI）

```
src/
├── domain/
│   └── todo/
│       ├── entity.py           # Todo Aggregate
│       ├── value_objects.py    # TodoId, TodoTitle
│       ├── events.py           # TodoCompleted
│       └── repository.py       # ITodoRepository (ABC)
├── application/
│   └── todo/
│       ├── create_todo.py      # CreateTodoUseCase
│       ├── complete_todo.py    # CompleteTodoUseCase
│       └── queries/
│           └── dashboard_stats.py
├── infrastructure/
│   └── todo/
│       ├── repository_impl.py  # TodoRepositoryImpl
│       └── orm_models.py       # SQLAlchemy models
└── interfaces/
    └── api/
        └── todo_controller.py  # FastAPI Router
```

#### Completion Gate 3
- [ ] Inventory の Domain Layer Items に対応するファイルが作成されている
- [ ] Inventory の Application Layer Items に対応するファイルが作成されている
- [ ] Inventory の Infrastructure Layer Items に対応するファイルが作成されている（スケルトン可）
- [ ] Inventory の Interfaces Layer Items に対応するファイルが作成されている（スケルトン可）
- [ ] 全インターフェースに Docstrings が記載されている
- [ ] DI構造が明確になっている（Repository は Interface を通じて注入）

### [Step 4] テスト先行 (Red Phase)

Step 2 の UseCase/Query をコード化し、正しく「失敗」させる。

* **Action:**
  - Mock Repository を注入し、UseCase に対するテストケースを作成
  - Domain Event の発行をテストで検証
  - Query の入出力をテストで検証

* **DoD:** コンパイルエラーはなく、実行時に `NotImplementedError` または `AssertionError` でテストが失敗すること。

#### テスト例（Python/pytest）

```python
def test_complete_todo_should_emit_event():
    # Given
    mock_repo = MockTodoRepository()
    mock_repo.add(Todo(id=TodoId(1), title="Test", status=PENDING))
    use_case = CompleteTodoUseCase(repo=mock_repo, publisher=mock_publisher)

    # When
    result = use_case.execute(todo_id=TodoId(1), user_id=UserId(1))

    # Then
    assert result.status == "COMPLETED"
    assert mock_publisher.published_events[0].type == "TodoCompleted"
```

#### Completion Gate 4
- [ ] 全 UseCase に対するテストケースが存在する
- [ ] 全 Query に対するテストケースが存在する
- [ ] Domain Event の発行がテストで検証されている
- [ ] テストが「正しい理由で」失敗している（NotImplementedError または AssertionError）

### [Step 5] ドメイン・アプリケーション層実装 (Green Phase)

外部依存をMockにしたまま、Domain Layer と Application Layer を完成させる。

* **Action:**
  - Aggregate の振る舞い（behaviors）を実装
  - UseCase のフロー（flow）を実装
  - Domain Event の発行ロジックを実装

* **DoD:** Step 4 のテストが全て **PASS (Green)** すること。

#### Completion Gate 5
- [ ] Step 4 で作成した全テストが PASS している
- [ ] Inventory の Domain Layer Items が全て `[x]` になっている
- [ ] Inventory の Application Layer Items が全て `[x]` になっている
- [ ] Domain Event が正しく発行されている

### [Step 6] Infrastructure & Interfaces 実装 (Implementation & Wiring)

現実世界（Real Infra / UI）の実装と結合。**Platform-Dependent Items もこのステップで実装する。**

* **Action:**
  1. **Infrastructure Layer:**
     - Repository 実装（DB接続、ORM設定）
     - Event Publisher 実装（メッセージキュー、インメモリ等）
  2. **Interfaces Layer (Backend):**
     - Controller 実装（REST API エンドポイント）
     - DTO / Request / Response の定義
  3. **Interfaces Layer (Frontend):**
     - ViewModel / Custom Hook の実装
     - UI Component の実装
  4. **Platform (重要):** Implementation Inventory の Platform-Dependent Items を全て実装

#### Sub-Step 6.1: Platform Service 実装

**テスト困難な項目を確実に実装するためのサブステップ。**

| 作業 | 確認方法 | 完了基準 |
|------|---------|---------|
| DB Migration | `alembic upgrade head` 成功 | テーブルが作成される |
| API Server | `uvicorn main:app` 起動 | エンドポイントがレスポンスを返す |
| 通知 | 該当イベント発火時に確認 | 通知が表示される |
| 認証 | JWT トークン発行・検証 | 認可が機能する |

* **Sub-Step 6.5 (Linting):** 静的解析 (`ruff`, `mypy`, `eslint`) を実行。
* **DoD:** アプリが起動し、API操作が可能であること。Linterのエラーが0であること。

#### Completion Gate 6
- [ ] Inventory の Infrastructure Layer Items が全て `[x]` になっている
- [ ] Inventory の Interfaces Layer Items が全て `[x]` になっている
- [ ] **Inventory の Platform-Dependent Items が全て `[x]` になっている（重要）**
- [ ] API が起動し、エンドポイントが応答する
- [ ] Frontend が起動し、画面が表示される
- [ ] Linter エラーが 0

### [Step 7] 結合テスト (Integration Testing)

Infrastructure Layer と Interfaces Layer の結合テスト。

* **Action:**
  - Repository 実装の結合テスト（テスト用DBを使用）
  - Controller の結合テスト（TestClient を使用）
  - ViewModel/Hook のテスト

* **DoD:** 実際のDB/APIを使用したテストが全て PASS すること。

#### テスト例（FastAPI TestClient）

```python
def test_create_todo_api():
    # Given
    client = TestClient(app)

    # When
    response = client.post("/todos", json={"title": "Test Todo"})

    # Then
    assert response.status_code == 201
    assert response.json()["title"] == "Test Todo"
```

#### Completion Gate 7
- [ ] Repository 実装のテストが全て PASS
- [ ] Controller のテストが全て PASS
- [ ] ViewModel/Hook のテストが全て PASS
- [ ] Inventory の Test 列が全て埋まっている（Platform-Dependent を除く）

### [Step 8] E2E テスト (End-to-End Testing)

ユーザーシナリオに基づくE2Eテスト。

* **Action:**
  - Frontend から Backend までの通しテスト
  - `interfaces/frontend/*.yaml` の scenarios をテストコード化
  - 認証フローを含むシナリオテスト

* **DoD:** ユーザー操作をシミュレーションし、システム全体が正しく動作することが検証されていること。

#### Completion Gate 8
- [ ] E2E シナリオテストが全て PASS
- [ ] `interfaces/frontend/*.yaml` の scenarios が全て検証されている
- [ ] 認証を含むフローが動作確認されている
- [ ] Platform-Dependent Items の手動確認が完了している

### [Step 9] Final Verification (最終確認)

**Implementation Inventory との最終照合。**

* **Action:**
  1. Inventory ファイルを開き、全項目の Status が `[x]` であることを確認
  2. 未完了項目がある場合は、該当 Step に戻って実装
  3. 確認完了後、Inventory に完了日時を記録

* **DoD:** Implementation Inventory の全項目が `[x]` であること。

#### Final Completion Gate
- [ ] Inventory の Testable Items: 全て `[x]}
- [ ] Inventory の Platform-Dependent Items: 全て `[x]`
- [ ] 自動テスト: 全て PASS
- [ ] 手動確認: 全て完了
- [ ] `blueprint.yaml` と実装の整合性: 確認済み

---

## 4. コーディング詳細規約 (Coding Standards)

### 4.1 オブジェクト指向設計 (OOP Rules)

* **Interface Usage:** ビジネスロジックとインフラ層は必ず `interface` を定義し、実装クラスはそれを `implements` すること。
* **Dependency Injection:** クラス内部で `new Dependency()` を行うことを禁止する。依存オブジェクトは必ずコンストラクタまたは引数経由で受け取ること。
* *Bad:* `const repo = new UserRepo();`
* *Good:* `constructor(private repo: IUserRepo) {}`



### 4.2 コメントとドキュメンテーション

* **Docstrings:** 公開メソッドには必ず IDE で参照可能な形式（JSDoc/KDoc等）でコメントを付与すること。
* Must: `@param`, `@return`, `@throws`
* Must: メソッドの責務と「何をしていないか」の明記



### 4.3 ファイル命名規則 (File Naming)

#### Backend (Python/FastAPI)
* **Domain:** `entity.py`, `value_objects.py`, `events.py`, `repository.py`
* **Application:** `create_{aggregate}.py`, `{action}_{aggregate}.py`
* **Infrastructure:** `{aggregate}_repository_impl.py`, `orm_models.py`
* **Interfaces:** `{aggregate}_controller.py`, `dto.py`

#### Frontend (TypeScript/Next.js)
* **ViewModel:** `use{Feature}.ts`
* **View:** `page.tsx`, `{Component}.tsx`

### 4.4 ディレクトリ構成 (Directory Structure)

プロジェクト全体の標準ディレクトリ構成。**設計ファイル（design/）と実装（src/）を分離**する。

```
/project-root
├── design/                         # BDA設計ファイル群
│   ├── blueprint.yaml              # Entry Point
│   ├── guidelines/
│   │   └── tech_stack.yaml
│   ├── contexts/                   # Bounded Contexts
│   │   ├── identity/
│   │   │   ├── main.yaml
│   │   │   ├── domain.yaml
│   │   │   └── application.yaml
│   │   └── todo/
│   │       ├── main.yaml
│   │       ├── domain.yaml
│   │       └── application.yaml
│   ├── interfaces/
│   │   ├── backend/
│   │   │   └── todo_controller.yaml
│   │   └── frontend/
│   │       ├── dashboard.yaml
│   │       └── routing.yaml
│   └── workflows/
│       └── onboarding.yaml
│
├── _docs/                          # ドキュメント・管理資料
│   ├── inventory/                  # Implementation Inventory
│   │   ├── identity_inventory.md
│   │   └── todo_inventory.md
│   └── requirements/               # 要件定義書
│
├── backend/                        # Backend (Python/FastAPI)
│   └── src/
│       ├── domain/                 # Domain Layer
│       │   ├── identity/
│       │   │   ├── entity.py
│       │   │   ├── events.py
│       │   │   └── repository.py   # Interface (ABC)
│       │   └── todo/
│       │       ├── entity.py
│       │       ├── value_objects.py
│       │       ├── events.py
│       │       └── repository.py
│       ├── application/            # Application Layer
│       │   ├── identity/
│       │   │   ├── register_user.py
│       │   │   └── login.py
│       │   └── todo/
│       │       ├── create_todo.py
│       │       ├── complete_todo.py
│       │       └── queries/
│       ├── infrastructure/         # Infrastructure Layer
│       │   ├── persistence/
│       │   │   ├── orm_models.py
│       │   │   └── todo_repository_impl.py
│       │   └── event_publisher.py
│       └── interfaces/             # Interfaces Layer (API)
│           └── api/
│               ├── todo_controller.py
│               └── auth_controller.py
│
└── frontend/                       # Frontend (Next.js)
    └── src/
        ├── app/                    # App Router
        │   ├── dashboard/
        │   │   └── page.tsx
        │   └── auth/
        ├── hooks/                  # ViewModels
        │   └── useTodoList.ts
        └── components/             # UI Components
```

#### ディレクトリと DDD Layer の対応

| DDD Layer | Backend (Python) | Frontend (Next.js) |
|-----------|------------------|-------------------|
| Domain | `src/domain/{context}/` | - |
| Application | `src/application/{context}/` | - |
| Infrastructure | `src/infrastructure/` | - |
| Interfaces | `src/interfaces/api/` | `src/app/`, `src/hooks/`, `src/components/` |
| Design Files | `design/` | `design/` |
| Inventory | `_docs/inventory/` | `_docs/inventory/` |


### 4.5 メソッド設計と複雑度 (Method Design)

Single Responsibility: 1つのメソッドは「1つの動作」のみを行うこと。名前通りの動作以外（副作用）を含まないこと。

Complexity Limit: メソッドの循環的複雑度（Cyclomatic Complexity）は 10 以下、行数は原則 50行以下 を目安とする。これを超える場合は、処理をプライベートメソッド等へ分割すること。

---

## 5. 品質保証チェックリスト (QA Checklist)

プルリクエスト（または実装完了）時には以下の項目を確認する。

### 5.1 基本チェック
* [ ] **BDA Compliance:** 設計ファイル（`design/contexts/*/domain.yaml` 等）と実装の整合性は取れているか？
* [ ] **Test Coverage:** Step 4, 5, 7, 8 のテストは全て Green か？
* [ ] **Linting:** `ruff check` / `mypy` / `eslint` はエラーなしか？
* [ ] **Deprecation Warning:** 非推奨APIの警告が出ていないか？（下記参照）
* [ ] **Strict DI:** Repository は Interface（ABC）を通じて注入されているか？
* [ ] **Documentation:** 主要なロジックに Docstrings はあるか？

#### Deprecation Warning の対応（重要）

ビルド時に **Deprecation Warning** が出力された場合、**放置せず即座に対応**すること。

| 状況 | 対応 |
|------|------|
| 警告が出た | 新しいAPIに置き換える |
| 新APIが不明 | 公式ドキュメントで代替を確認 |
| 代替がない | コメントで理由を明記し、Issue登録 |

```kotlin
// Bad: 警告を放置
.menuAnchor()  // 'menuAnchor()' is deprecated

// Good: 新APIに更新
.menuAnchor(MenuAnchorType.PrimaryNotEditable, enabled = true)
```

**理由:** Deprecation Warning を放置すると、将来のバージョンアップで突然動作しなくなるリスクがある。

### 5.2 DDD Compliance チェック
* [ ] **Domain Layer:** Aggregate が技術詳細（DB, Framework）に依存していないか？
* [ ] **Domain Event:** 振る舞い（behaviors）に定義された Event が発行されているか？
* [ ] **UseCase:** Application Layer の UseCase が単一責任になっているか？
* [ ] **Repository Interface:** Domain Layer に Interface、Infrastructure Layer に実装があるか？

### 5.3 Implementation Inventory チェック（重要）
* [ ] **Inventory 存在確認:** `_docs/inventory/{context}_inventory.md` が存在するか？
* [ ] **Domain Layer Items:** 全項目が `[x]` か？
* [ ] **Application Layer Items:** 全項目が `[x]` か？
* [ ] **Infrastructure Layer Items:** 全項目が `[x]` か？
* [ ] **Interfaces Layer Items:** 全項目が `[x]` か？
* [ ] **Platform-Dependent Items:** 全項目が `[x]` か？

### 5.4 漏れ防止の最終確認
* [ ] **Domain Events:** `domain.yaml` に定義された全 Event が実装されているか？
* [ ] **UseCases:** `application.yaml` に定義された全 UseCase が実装されているか？
* [ ] **Queries:** `application.yaml` に定義された全 Query が実装されているか？
* [ ] **$ref 整合性:** 全ての `$ref` 参照先ファイルが存在するか？

### 5.5 設計レビューチェックリスト（Step 2 完了時）

設計ファイル作成後、実装に入る前に以下を確認すること。

#### Context間依存
* [ ] 新Contextは既存Contextと連携が必要か？
* [ ] 連携が必要な場合、`blueprint.yaml` に依存関係が記載されているか？
* [ ] 依存方向は適切か？（循環依存がないか）

#### UI詳細仕様
* [ ] `interfaces/` にUI振る舞いの詳細が記載されているか？
* [ ] 入力フィールドのタイプ（TextField, AutoComplete, Dropdown等）が明記されているか？
* [ ] バリデーションルールが明記されているか？
* [ ] エラー時・空状態の表示が定義されているか？

#### 非機能要件
* [ ] パフォーマンス要件（大量データ時の挙動）が検討されているか？
* [ ] オフライン時の挙動が検討されているか？
* [ ] セキュリティ（入力値検証、認証）が検討されているか？

#### Inventory同期
* [ ] 設計変更後、`/bda-inventory` が再実行されているか？
* [ ] Inventoryに新規項目が追加されているか？

---

## 6. ワークフロー概要図

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   BDA 3.3.0 Development Workflow                        │
│                     (Context-Driven Development)                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [Step 1] 要件定義 ──→ Bounded Context の特定                           │
│      ↓                                                                  │
│  [Step 2] ドメインモデル確定                                            │
│      │   ├─ contexts/{name}/domain.yaml     (Aggregate, Event, Repo)   │
│      │   ├─ contexts/{name}/application.yaml (UseCase, Query)          │
│      │   └─ interfaces/*.yaml                (Controller, ViewModel)   │
│      ↓                                                                  │
│  [Step 2.5] Implementation Inventory 作成                               │
│      │   └─ _docs/inventory/{context}_inventory.md                      │
│      │       ┌─────────────────────────────────────┐                    │
│      │       │  DDD Layer 別分類:                   │                    │
│      │       │  ├─ Domain Layer Items              │                    │
│      │       │  ├─ Application Layer Items         │                    │
│      │       │  ├─ Infrastructure Layer Items      │                    │
│      │       │  ├─ Interfaces Layer Items          │                    │
│      │       │  └─ Platform-Dependent Items ★      │                    │
│      │       └─────────────────────────────────────┘                    │
│      ↓                                                                  │
│  [Step 3] Skeleton 作成 ──→ DDD Layer 別にファイル作成                  │
│      ↓                                                                  │
│  [Step 4] Red Phase ──→ UseCase/Query のテスト作成（失敗）              │
│      ↓                                                                  │
│  [Step 5] Green Phase ──→ Domain + Application Layer 実装              │
│      ↓                                                                  │
│  [Step 6] Infrastructure & Interfaces 実装                              │
│      │   ├─ 6.1 Repository 実装 (DB接続)                                │
│      │   ├─ 6.2 Controller 実装 (REST API)                              │
│      │   ├─ 6.3 ViewModel/View 実装 (Frontend)                          │
│      │   └─ 6.4 Platform Service ★                                     │
│      ↓                                                                  │
│  [Step 7] Integration Testing ──→ DB/API 結合テスト                     │
│      ↓                                                                  │
│  [Step 8] E2E Testing ──→ Frontend ↔ Backend 通しテスト                 │
│      ↓                                                                  │
│  [Step 9] Final Verification ──→ Inventory 全項目 [x] 確認              │
│      ↓                                                                  │
│  ✓ DONE                                                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

★ Platform-Dependent: 自動テスト困難な項目。Inventory で明示的に追跡し、
                      手動確認で検証する。
```

---

## 7. Implementation Inventory テンプレート

新しい Context の実装時は、以下のテンプレートを `_docs/inventory/{context}_inventory.md` にコピーして使用する。

```markdown
# Implementation Inventory: {Context名}Context

## Meta

| Key | Value |
|-----|-------|
| **Context** | {Context名} |
| **作成日** | YYYY-MM-DD |
| **最終更新** | YYYY-MM-DD |
| **担当者** | {名前} |
| **BDA Version** | 3.3.0 |
| **Design Files Hash** | {git hash} |
| **Last Synced** | YYYY-MM-DD HH:MM |

## Design Files Reference

| File | Description |
|------|-------------|
| `design/contexts/{context}/main.yaml` | Context 定義 |
| `design/contexts/{context}/domain.yaml` | Aggregate, Event, Repository |
| `design/contexts/{context}/application.yaml` | UseCase, Query |

## Domain Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| D-001 | Aggregate | {Aggregate名} | domain/{context}/entity.py | Unit | [ ] |
| D-002 | Value Object | {VO名} | domain/{context}/value_objects.py | Unit | [ ] |
| D-003 | Domain Event | {Event名} | domain/{context}/events.py | Unit | [ ] |
| D-004 | Repository Interface | I{Aggregate}Repository | domain/{context}/repository.py | - | [ ] |
| D-005 | Domain Service | {Service名} | domain/{context}/services.py | Unit | [ ] |

## Application Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| A-001 | UseCase | Create{Aggregate}UseCase | application/{context}/create_{aggregate}.py | Unit | [ ] |
| A-002 | UseCase | {Action}{Aggregate}UseCase | application/{context}/{action}_{aggregate}.py | Unit | [ ] |
| A-003 | Query | Get{Resource}Query | application/{context}/queries/{resource}.py | Unit | [ ] |
| A-004 | Event Handler | Handle{Event}Handler | application/{context}/handlers/{event}.py | Unit | [ ] |

## Infrastructure Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| I-001 | Repository Impl | {Aggregate}RepositoryImpl | infrastructure/{context}/repository_impl.py | Integration | [ ] |
| I-002 | ORM Model | {Aggregate}ORM | infrastructure/persistence/orm_models.py | Integration | [ ] |
| I-003 | Event Publisher | EventPublisher | infrastructure/event_publisher.py | Integration | [ ] |

## Interfaces Layer Items

| ID | Category | Item | File | Test | Status |
|----|----------|------|------|------|--------|
| IF-001 | Controller | {Aggregate}Controller | interfaces/api/{aggregate}_controller.py | Integration | [ ] |
| IF-002 | DTO | {Aggregate}DTO | interfaces/api/dto.py | - | [ ] |
| IF-003 | ViewModel | use{Feature} | frontend/hooks/use{Feature}.ts | Unit | [ ] |
| IF-004 | View | {Feature}Page | frontend/app/{feature}/page.tsx | UI | [ ] |

## Platform-Dependent Items (プラットフォーム依存)

| ID | Category | Item | File | Verification Procedure | Status | 確認日 | 確認者 |
|----|----------|------|------|------------------------|--------|--------|--------|
| P-001 | Database | DB Migration | alembic/versions/*.py | `alembic upgrade head` が成功すること | [ ] | | |
| P-002 | Auth | JWT認証 | infrastructure/auth/ | 認証トークンが正しく発行・検証されること | [ ] | | |
| P-003 | Notification | Push通知 | infrastructure/notification/ | イベント発火時に通知が表示されること | [ ] | | |

## UseCase / Query Coverage

`application.yaml` との対応:

| Name (from YAML) | Type | Implementation File | Test | Status |
|------------------|------|---------------------|------|--------|
| Create{Aggregate} | UseCase | create_{aggregate}.py | Unit | [ ] |
| {Action}{Aggregate} | UseCase | {action}_{aggregate}.py | Unit | [ ] |
| Get{Resource} | Query | queries/{resource}.py | Unit | [ ] |

## Summary

| Layer | Total | Completed | Remaining |
|-------|-------|-----------|-----------|
| Domain | 0 | 0 | 0 |
| Application | 0 | 0 | 0 |
| Infrastructure | 0 | 0 | 0 |
| Interfaces | 0 | 0 | 0 |
| Platform-Dependent | 0 | 0 | 0 |
| **Total** | **0** | **0** | **0** |

## Completion Log

| Step | Completed | Date | Notes |
|------|-----------|------|-------|
| Step 2.5 Inventory作成 | [ ] | | |
| Step 3 Skeleton作成 | [ ] | | |
| Step 5 Domain/Application完了 | [ ] | | |
| Step 6 Infrastructure/Interfaces完了 | [ ] | | |
| Step 9 Final確認 | [ ] | | |

## Change History (設計ファイル同期履歴)

| Date | Design Files Hash | Changes | Updated By |
|------|-------------------|---------|------------|
| YYYY-MM-DD | abc1234 | 初回作成 | {名前} |
```

---

## 8. 参考資料

- **BDA仕様書:** `BDA_specification.md`
- **サンプル設計:** `sample/design/`
- **GitHub:** https://github.com/miris-mimitako/Blueprint-Driven-Architecture_BDA
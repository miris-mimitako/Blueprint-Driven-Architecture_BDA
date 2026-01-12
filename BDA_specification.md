---
Version: 3.0.0 (AI-Native)
---
# Blueprint-Driven Architecture (BDA) Specification

## Core Philosophy

BDAは、アプリケーション開発における「実装の繰り返し」と「仕様の形骸化」を排除するために設計されたアーキテクチャスタイルです。
この仕様はフリースタイルベースラインを採用しています。
最低限の定義はここにありますが、ユーザーが任意に追加、削除することでAIによるコーディングを最高精度に高めることを求めています。

| BDA Term Definition | Android (MVVM) | Web (React / Component) |
| --- | --- | --- |
| **Domain** | Data Class | TypeScript Interface / Zod Schema |
| **Feature** | Screen / ViewModel | Page Component / Custom Hook |
| **State (Output)** | `StateFlow<UiState>` | Hook return values / Context |
| **Input (Event)** | ViewModel methods | Event Handlers / Callbacks |
| **Routing** | Jetpack Navigation | React Router / Next.js Routing |
| **Scenario** | `@Preview` Provider | Storybook (`.stories.tsx`) |

### Blueprint is the App (青写真こそがアプリである)

- blueprint.yaml は単なる設定ファイルではなく、アプリケーションの**正体（Identity）**である
- ソースコード（Kotlin, TSなど）は、青写真から出力される**「成果物（Artifact）」**に過ぎない。

### Schema First, Logic Second (構造が先、ロジックは後)

- 「データ構造(Schema)」と「入出力(I/O)」が決まれば、アーキテクチャは確定する。
- ロジックの実装は、確定した枠組みの中を埋める作業（Fill-in-the-blanks）に限定される。

### Simulation Driven (シミュレーション駆動)

- 実装前に「シナリオ」を定義することで、実行可能な仕様書（Storybook/Preview）とテストケースを同時に生成する。

## Architecture Overview (全体像)

### Top Level / Root Structure

トップレベルレイヤを定義します。

```yaml
blueprint: "3.0.0 (AI-Native)" # BDA version

meta:
    # アプリのメタデータ（ID, パッケージ名, バージョン等）

guidelines:
    # AIへの技術スタックと実装ルールの指示

domain:
    # アプリケーション全体で共有される「名詞（データ型）」の定義 
    # 複数のページにわたって利用される変数やクラスなどの定義

api:
    # Backend/Frontend間のAPI契約（エンドポイント定義）

features:
    # 各画面・機能ごとの「MVVMスペック, Components, hooks （I/O, State, Scenarios）」の定義

routing:
    # 画面間の遷移ルールと全体構造（サイトマップ）の定義
```

#### Detail Specs

##### A. meta Section

アプリケーションの基本情報を定義します。

```yaml
meta:
  appName: "NextFastTodo"
  version: "0.1.0"
```

##### B. guidelines Section

AIへの技術スタックと実装ルールを指示するセクションです。Backend/Frontend両方の技術選定を明示的に定義します。

役割: AIが実装を行う際の技術的なガイドライン。

```yaml
guidelines:
  backend:
    framework: "FastAPI (Python)"
    db: "SQLite (with SQLAlchemy)"
    auth: "OAuth2 with JWT (python-jose)"
    validation: "Pydantic v2"
    style: "PEP8, Type Hints必須"
  
  frontend:
    framework: "Next.js 14+ (App Router)"
    language: "TypeScript"
    styling: "Tailwind CSS"
    state: "React Hooks (Context API or Zustand)"
    apiClient: "Axios (generates types from backend schemas)"
    style: "Functional Components, Prettier"
```

##### C. domain Section

アプリ内で流通するデータ型（Value Object / Entity）を定義します。

役割: 言語に依存しない型定義。

生成物: Kotlin data class, TypeScript interface, Python Pydantic Model.

```yaml
domain:
  schemas:
    - name: User
      properties:
        - { name: id, type: Integer }
        - { name: email, type: String }
        - { name: is_active, type: Boolean }

    - name: UserCreate
      properties:
        - { name: email, type: String }
        - { name: password, type: String }

    - name: Token
      properties:
        - { name: access_token, type: String }
        - { name: token_type, type: String }

    - name: TodoItem
      properties:
        - { name: id, type: Integer }
        - { name: title, type: String }
        - { name: description, type: String? }  # ?はOptional
        - { name: completed, type: Boolean }
        - { name: user_id, type: Integer }
```

##### D. api Section

Backend実装とFrontendクライアントの契約（API仕様）を定義します。

役割: エンドポイントの入出力仕様を明確化。

構成要素:
- endpoints: APIのグループ定義
- base: ベースパス
- operations: 各HTTPメソッドと入出力の定義

```yaml
api:
  endpoints:
    - name: Auth
      base: "/auth"
      operations:
        - { method: POST, path: "/token", input: UserCreate, output: Token, desc: "Login to get JWT" }
        - { method: POST, path: "/register", input: UserCreate, output: User, desc: "Create new user" }
    
    - name: Todos
      base: "/todos"
      operations:
        - { method: GET, path: "/", output: "List<TodoItem>", desc: "Get my todos" }
        - { method: POST, path: "/", input: TodoItem, output: TodoItem, desc: "Create todo" }
        - { method: PUT, path: "/{id}", input: TodoItem, output: TodoItem, desc: "Update todo" }
        - { method: DELETE, path: "/{id}", output: Boolean, desc: "Delete todo" }
```

##### E. features Section

BDAの核となる部分。MVVMの「境界（Boundary）」を定義します。

役割: 機能ごとの仕様（ブラックボックス）定義。

構成要素:

- id: 機能の一意識別子
- name: 画面/コンポーネント名
- path: URLパス（Web）
- State: 画面が持つべき情報（Output）。domain で定義した型を使用可能。
- Inputs: 画面が受け付ける操作（Input）。引数定義を含む。
- Scenarios: Given（初期状態）と When（操作）に対する Then（期待値）のセット。

```yaml
features:
  - id: LoginFeature
    name: LoginPage
    path: "/login"
    
    # [State]
    # React: const { email, isLoading, error } = useLogin();
    # Android: val uiState by viewModel.uiState.collectAsState()
    state:
      - { name: email, type: String, default: "" }
      - { name: password, type: String, default: "" }
      - { name: isLoading, type: Boolean, default: false }
      - { name: error, type: String?, default: null }

    # [Inputs]
    # React: const { onEmailChange } = useLogin(); <input onChange={onEmailChange} />
    # Android: onClick = { viewModel.onEmailChange(value) }
    inputs:
      - { name: onEmailChange, args: [{name: val, type: String}] }
      - { name: onPasswordChange, args: [{name: val, type: String}] }
      - { name: onSubmit, args: [] }
      - { name: onRegisterClick, args: [] }

    # [Scenarios] -> Storybook / Preview / Test Cases
    scenarios:
      # 表示状態テスト（givenのみ）
      - name: "Initial View"
        given:
          state: { email: "", password: "", isLoading: false, error: null }

      - name: "Loading State"
        given:
          state: { email: "test@example.com", isLoading: true }

      - name: "Login Error"
        given:
          state: { error: "Invalid credentials" }

      # 操作テスト（given + when + then）
      - name: "Submit Empty Form"
        given:
          state: { email: "", password: "" }
        when:
          - { input: onSubmit }
        then:
          state: { error: "Email is required" }

      - name: "Submit Valid Form"
        given:
          state: { email: "test@example.com", password: "password123" }
        when:
          - { input: onSubmit }
        then:
          state: { isLoading: true }
```

##### F. routing Section

画面の階層構造と遷移ルールを定義します。

役割: アプリの骨格（Navigation Graph）定義。

構成要素:

- entry: アプリの初期表示パス
- structure: 画面の階層構造
- guard: 認証ガードなどのアクセス制御

```yaml
routing:
  entry: /login
  
  # [Structure] 画面構造とURLパスのマッピング
  structure:
    - name: RootLayout
      children:
        - { name: Login, feature: LoginFeature }
        - { name: Dashboard, feature: TodoListFeature, guard: AuthGuard }
```

##### G. routing Section (Advanced)

より複雑なアプリケーションでは、詳細な遷移定義が可能です。

```yaml
routing:
  entry: /splash
  
  # [Structure] 画面構造とURLパスのマッピング
  structure:
    - name: RootLayout
      type: Stack
      children:
        - name: Splash
          path: "/splash"
          ref: SplashFeature
          
        - name: AuthFlow
          path: "/auth"
          type: Stack
          children:
            - { name: Login, path: "login", ref: LoginFeature }
            - { name: Register, path: "register", ref: RegisterFeature }
            
        - name: Dashboard
          path: "/dashboard"
          type: Layout  # WebではLayoutコンポーネント、MobileではBottomNavなど
          children:
            - { name: Home, path: "", ref: HomeFeature }

  # [Transitions] 遷移ロジック
  transitions:
    - from: Splash
      on: Loaded
      to: /auth/login
      type: Replace # Historyを書き換え

    - from: Login
      on: LoginSuccess
      to: /dashboard
      args: { userId: "${event.userId}" }
```

## BDA structure

```md
/design
  ├── blueprint.yaml        # [Root] エントリーポイント
  ├── domain/               # [Schemas] データ型定義
  │    ├── user.yaml
  │    └── product.yaml
  ├── features/             # [Features] 機能スペック
  │    ├── auth/
  │    │    ├── login.yaml
  │    │    └── register.yaml
  │    └── home/
  │         └── home.yaml
  ├── routing/              # [Routing] 画面遷移定義
  │    └── main_nav.yaml
  └── openapi.yaml          # [OpenAPI] API定義(Optional)
```

## サンプルプロジェクト

`/sample` ディレクトリに実際のblueprint.yamlの例があります。

- `todo_app/blueprint.yaml` - Next.js + FastAPI を使用したTodoアプリケーションの例

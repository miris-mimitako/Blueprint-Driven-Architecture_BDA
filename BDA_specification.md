---
Version: 0.0.1 (Cross-Platform / Web & Mobile)
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

```blueprint.yaml

blueprint: "1.0.0" #BDA version

meta:
    # アプリのメタデータ（ID, パッケージ名, バージョン等）

domain:
    # アプリケーション全体で共有される「名詞（データ型）」の定義 
    # 複数のページにわたって利用される変数やクラスなどの定義

features:
  # 各画面・機能ごとの「MVVMスペック, Components, hooks （I/O, State, Scenarios）」の定義

routing:
  # 画面間の遷移ルールと全体構造（サイトマップ）の定義

```

#### Detail Specs

##### A. domain Section

アプリ内で流通するデータ型（Value Object / Entity）を定義します。

役割: 言語に依存しない型定義。

生成物: Kotlin data class, TypeScript interface.

```yaml
domain:
  schemas:
    - name: User
      properties:
        - { name: id, type: String }
        - { name: role, type: Enum, values: [Admin, Member] }
```

##### B. features Section

BDAの核となる部分。MVVMの「境界（Boundary）」を定義します。

役割: 機能ごとの仕様（ブラックボックス）定義。

構成要素:

State: 画面が持つべき情報（Output）。domain で定義した型を使用可能。

Inputs: 画面が受け付ける操作（Input）。引数定義を含む。

Scenarios: Given（初期状態）と When（操作）に対する Then（期待値）のセット。


```yaml
features:
  - id: LoginFeature
    name: Login
    
    # [State]
    # React: const { email, isLoading, error } = useLogin();
    # Android: val uiState by viewModel.uiState.collectAsState()
    state:
      - { name: email, type: String, default: "" }
      - { name: isLoading, type: Boolean, default: false }
      - { name: error, type: String?, default: null }

    # [Inputs]
    # React: const { onLogin } = useLogin(); <button onClick={onLogin} />
    # Android: onClick = { viewModel.onLogin() }
    inputs:
      - name: onEmailChange
        args: [{ name: value, type: String }]
      - name: onLoginClick
        args: []

    # [Scenarios] -> Storybook / Preview
    scenarios:
      - name: "Initial State"
        given: 
          state: { email: "", isLoading: false }
      
      - name: "Loading State"
        given: 
          state: { email: "test@example.com", isLoading: true }
```

##### C. routing Section
画面の階層構造と遷移ルールを定義します。

役割: アプリの骨格（Navigation Graph）定義。

構成要素:

Structure: Stack, Tabs, Drawer などのUI構造。

Transitions: 「どの画面(From)」から「どのイベント(Trigger)」で「どこへ(To)」行くか。

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
  └── openapi.yaml # [OpenAPI] API定義(Optional)
  ```
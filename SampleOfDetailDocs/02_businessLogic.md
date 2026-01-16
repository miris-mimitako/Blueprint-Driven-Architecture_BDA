## Step 2: ビジネスロジックの確定 (BDA Blueprint Definition)

このYAMLファイルが、これから作る機能の全ての「正解」となります。

### `blueprint.yaml` (Partial / Focus Timer Feature)

```yaml
# [Domain Section]
# アプリ全体で扱う「集中セッション」のデータ型定義
domain:
  schemas:
    - name: FocusRecord
      properties:
        - { name: id, type: String }
        - { name: durationMinutes, type: Integer }
        - { name: completedAt, type: String } # ISO8601

    - name: FocusRecordCreate
      properties:
        - { name: durationMinutes, type: Integer }

# [API Section]
# 完了時にBackendへ実績を保存するための契約
api:
  endpoints:
    - name: Focus
      base: "/focus"
      operations:
        - { method: POST, path: "/record", input: FocusRecordCreate, output: FocusRecord, desc: "Record a completed focus session" }

# [Features Section] - CORE LOGIC
# 画面（ViewModel/State）の振る舞い定義
features:
  - id: FocusTimerFeature
    name: FocusTimerPage
    path: "/focus"
    
    # [State] 画面が持つべき状態 (Output)
    # これがUIに描画される唯一の情報源となる
    state:
      - { name: remainingSeconds, type: Integer, default: 1500 } # Default 25min * 60
      - { name: status, type: String, default: "IDLE" }         # values: IDLE, RUNNING, PAUSED, COMPLETED
      - { name: sessionCount, type: Integer, default: 0 }        # 今日の達成回数
      - { name: isSaving, type: Boolean, default: false }        # API保存中フラグ

    # [Inputs] 画面が受け付ける操作 (Input/Event)
    # ユーザー操作だけでなく、タイマーによる定期実行(onTick)もイベントとして定義する
    inputs:
      - { name: onStartClick, args: [] }
      - { name: onPauseClick, args: [] }
      - { name: onResumeClick, args: [] }
      - { name: onResetClick, args: [] }
      - { name: onTick, args: [] } # 1秒ごとに呼ばれるシステムイベント

    # [Scenarios] 振る舞いの仕様書 (Simulation / Test Cases)
    # これがStep 4(Test)とStep 5(Mock)の入力となる
    scenarios:
      # 1. 初期表示
      - name: "Initial State (Ready)"
        given:
          state: { remainingSeconds: 1500, status: "IDLE" }

      # 2. 開始アクション
      - name: "Start Timer"
        given:
          state: { status: "IDLE" }
        when:
          - { input: onStartClick }
        then:
          state: { status: "RUNNING" }

      # 3. カウントダウン処理（ビジネスロジックの核）
      - name: "Tick Down"
        given:
          state: { status: "RUNNING", remainingSeconds: 1500 }
        when:
          - { input: onTick }
        then:
          state: { remainingSeconds: 1499 }

      # 4. 一時停止
      - name: "Pause Timer"
        given:
          state: { status: "RUNNING" }
        when:
          - { input: onPauseClick }
        then:
          state: { status: "PAUSED" }

      # 5. 完了判定とAPIコール
      - name: "Complete Timer"
        given:
          state: { status: "RUNNING", remainingSeconds: 1 }
        when:
          - { input: onTick }
        then:
          state: { remainingSeconds: 0, status: "COMPLETED", isSaving: true, sessionCount: 1 } 
          # Note: ここでAPIコールが発生するが、Stateとしては「保存中」になることが期待値

```

---

### Step 2 の成果物解説

BDAにおけるビジネスロジック確定とは、以下の3つが固定されることを意味します。

1. **データ構造 (Domain/State):**
* UIの状態が `remainingSeconds` (Int) と `status` (String) だけで完全に表現できることを定義しました。


2. **インターフェース (Inputs/API):**
* ユーザーができることは `Start`, `Pause`, `Reset` のみ。
* 時間の経過は `onTick` というイベントとして抽象化しました（これによりテストが容易になります）。


3. **因果関係 (Scenarios):**
* 「`onTick` が来たら `remainingSeconds` を減らす」というロジックを自然言語ではなく、Given/When/Thenのデータセットとして定義しました。



---

### Step 2 の Definition of Done (完了の定義)

> **DoD (Step 2):**
> * `blueprint.yaml` に必要なセクション (`domain`, `api`, `features`) が記述されていること。
> * `state`（出力）と `inputs`（入力）の型が定義されていること。
> * 主要なユースケースを網羅する `scenarios` が最低3つ以上（初期状態、正常系アクション、境界値など）定義されていること。
> * このYAMLがあれば、プログラミング言語を使わずに「アプリの紙芝居」が頭の中で完全に再生できる状態であること。
> 
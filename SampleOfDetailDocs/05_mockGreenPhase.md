## **Step 5: Mockを作って合格となるロジックを確立すること (Green Phase)**

このステップでは、Step 4で失敗させたテストを合格（Green）させるために、**ビジネスロジックの実装**を行います。

ここで言う「Mockを作って」とは、**「外部依存（RepositoryやAPI）はMockのままにし、純粋なロジック（Service）だけを実装・検証する」** ことを指します。これにより、ネットワークやDBの都合に左右されず、Step 2で定めた仕様（振る舞い）がコード上で正しく再現されているかを確定させます。

---

### 1. Implementation (ロジックの実装)

スケルトンだった `FocusTimerService.ts` に中身を記述します。
Step 2 の `scenarios` で定義した「条件（Given）」と「結果（Then）」を満たすように実装します。

**`src/features/focus/logic/FocusTimerService.ts`**

```typescript
import { IFocusTimerService } from '../domain/IFocusTimerService';
import { IFocusRepository } from '../domain/IFocusRepository';
import { FocusTimerState } from '../types';

export class FocusTimerService implements IFocusTimerService {
  private state: FocusTimerState;

  constructor(
    private readonly repository: IFocusRepository,
    initialState?: FocusTimerState
  ) {
    this.state = initialState ?? {
      remainingSeconds: 1500,
      status: 'IDLE',
      sessionCount: 0,
      isSaving: false
    };
  }

  getState(): FocusTimerState {
    // 参照渡しを防ぐためコピーを返す
    return { ...this.state };
  }

  start(): void {
    // 仕様: IDLE または PAUSED のときのみ開始可能
    if (this.state.status === 'IDLE' || this.state.status === 'PAUSED') {
      this.state.status = 'RUNNING';
    }
  }

  pause(): void {
    // 仕様: RUNNING のときのみ一時停止可能
    if (this.state.status === 'RUNNING') {
      this.state.status = 'PAUSED';
    }
  }

  reset(): void {
    // 常に初期状態へ戻す
    this.state = {
      remainingSeconds: 1500,
      status: 'IDLE',
      sessionCount: this.state.sessionCount, // 今日の実績数は維持
      isSaving: false
    };
  }

  tick(): void {
    // 仕様: RUNNING 以外のときは何もしない
    if (this.state.status !== 'RUNNING') return;

    if (this.state.remainingSeconds > 0) {
      this.state.remainingSeconds--;
    }

    // 0になった瞬間の処理 (Complete)
    if (this.state.remainingSeconds === 0) {
      this.handleCompletion();
    }
  }

  /**
   * 完了処理（内部メソッド）
   * APIコールを行い、結果に応じて状態を更新する
   */
  private handleCompletion(): void {
    this.state.status = 'COMPLETED';
    this.state.isSaving = true;
    this.state.sessionCount++;

    // 非同期で保存（リポジトリMockが呼ばれる）
    // Note: 実際のアプリではエラーハンドリングやawaitが必要だが、
    // ここではロジックの流れ（Repositoryを呼ぶこと）を確立する。
    this.repository.saveRecord({ durationMinutes: 25 })
      .then(() => {
        this.state.isSaving = false;
        console.log("Session saved successfully.");
      })
      .catch((err) => {
        this.state.isSaving = false;
        console.error("Failed to save session:", err);
        // 必要ならエラー用のStateを更新する処理を追加
      });
  }
}

```

---

### 2. Test Verification (合格の確認)

再度テストランナーを実行します。Step 4で作成したテストコードは変更しません。

**実行結果シミュレーション:**

```text
PASS  src/features/focus/logic/FocusTimerService.test.ts
  FocusTimerService (BDA Scenarios)
    ✓ Scenario: Initial State (Ready) (2ms)
    ✓ Scenario: Start Timer (1ms)
    ✓ Scenario: Tick Down (1ms)
    ✓ Scenario: Complete Timer (3ms)

Test Suites: 1 passed, 1 total
Tests:       4 passed, 4 total

```

**結果:** すべて **GREEN** になりました。
これにより、**「外部APIがまだ存在しなくても、仕様通りの振る舞いをするビジネスロジック」** が確立されました。

---

### Step 5 の Definition of Done (完了の定義)

> **DoD (Step 5):**
> * `FocusTimerService` のメソッドがすべて実装され、`Not implemented yet` エラーが解消されていること。
> * Step 4 で作成したテストケースが **全て合格（Pass）** していること。
> * Mock（Repository）に対するメソッド呼び出しが適切に行われていることが検証されていること。
> * コード内に複雑な分岐がある場合、それに対応するテストケースも追加でパスしていること。
> 
> 

## **Step 4: テストで不合格にすること (Red Phase)**

このステップでは、Step 2で定義した `blueprint.yaml` の **Scenarios** を、Step 3で作成した **Interface/Skeleton** を使って実行可能なテストコードに変換します。

スケルトンの中身は `throw new Error("Not implemented yet")` なので、当然テストは落ちます（**RED**）。しかし、ここで重要なのは **「テストコード自体はコンパイルエラーにならず、正しく記述できる（＝インターフェース設計が正しい）」** ことを証明することです。

---

### 1. Test Setup (Mock & Subject Creation)

テストフレームワーク（Jest/Vitest等）を使用する想定です。
DIパターンの利点を活かし、`IFocusRepository` のMockを作成して注入します。

**`src/features/focus/logic/FocusTimerService.test.ts`**

```typescript
import { FocusTimerService } from './FocusTimerService';
import { IFocusRepository } from '../domain/IFocusRepository';
import { FocusTimerState } from '../types';

// Mock作成 (Jest Style)
const mockSaveRecord = jest.fn();
const mockRepository: IFocusRepository = {
  saveRecord: mockSaveRecord,
};

// テスト対象 (SUT: System Under Test)
let service: FocusTimerService;

beforeEach(() => {
  jest.clearAllMocks();
  // DI: リポジトリを注入してインスタンス化
  service = new FocusTimerService(mockRepository);
});

describe('FocusTimerService (BDA Scenarios)', () => {
  
  // Scenario 1: Initial State
  test('Scenario: Initial State (Ready)', () => {
    // Given: (Default constructor)
    
    // When: getting state
    const state = service.getState();

    // Then: match BDA spec
    expect(state).toEqual({
      remainingSeconds: 1500,
      status: 'IDLE',
      sessionCount: 0,
      isSaving: false
    });
  });

  // Scenario 2: Start Timer
  test('Scenario: Start Timer', () => {
    // Given: state is IDLE (default)

    // When: input onStartClick
    // ★ここで "Not implemented yet" エラーが発生してテストが落ちる想定
    service.start(); 

    // Then: status becomes RUNNING
    expect(service.getState().status).toBe('RUNNING');
  });

  // Scenario 3: Tick Down
  test('Scenario: Tick Down', () => {
    // Given: state is RUNNING
    // 手動で初期状態をセット（テスト用コンストラクタを活用）
    service = new FocusTimerService(mockRepository, {
        remainingSeconds: 1500,
        status: 'RUNNING',
        sessionCount: 0,
        isSaving: false
    });

    // When: input onTick
    service.tick();

    // Then: remainingSeconds decreses by 1
    expect(service.getState().remainingSeconds).toBe(1499);
  });

  // Scenario 5: Complete Timer (Mock Verification)
  test('Scenario: Complete Timer', () => {
    // Given: 1 second remaining
    service = new FocusTimerService(mockRepository, {
        remainingSeconds: 1,
        status: 'RUNNING',
        sessionCount: 0,
        isSaving: false
    });

    // When: input onTick
    service.tick();

    // Then: 
    // 1. Status is COMPLETED (or saving)
    // 2. Repository was called (DI verification)
    const state = service.getState();
    expect(state.remainingSeconds).toBe(0);
    // expect(state.status).toBe('COMPLETED'); // 実装次第でSaving/Completed遷移
    
    // Mock検証: 外部依存が正しく呼ばれたか
    expect(mockSaveRecord).toHaveBeenCalledWith({ durationMinutes: 25 });
  });
});

```

---

### 2. Expected Failure Output (不合格の確認)

テストランナーを実行し、以下の結果になることを確認します。

```text
FAIL  src/features/focus/logic/FocusTimerService.test.ts
  FocusTimerService (BDA Scenarios)
    ✓ Scenario: Initial State (Ready)  <-- コンストラクタは実装済みなので通るかも
    ✕ Scenario: Start Timer            <-- Failed: Error: Not implemented yet
    ✕ Scenario: Tick Down              <-- Failed: Error: Not implemented yet
    ✕ Scenario: Complete Timer         <-- Failed: Error: Not implemented yet

Test Suites: 1 failed, 1 total
Tests:       3 failed, 1 passed, 4 total

```

---

### Step 4 の Definition of Done (完了の定義)

> **DoD (Step 4):**
> * BDAで定義された `scenarios` がすべてテストコード（`test` / `it` ブロック）として記述されていること。
> * コンパイルエラー（型エラー、importエラー）が出ていないこと。
> * テストを実行し、期待通りに **「実装未完了のエラー（Not implemented yet）」または「Assertion Error（期待値不一致）」で失敗** すること。
> * **注意:** テストコード自体のバグで落ちていないこと（テストのGiven/When/Thenが論理的に正しいこと）。
> 
> 

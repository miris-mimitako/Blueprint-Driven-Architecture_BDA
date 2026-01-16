## **Step 8: 結合テストを行い合格すること (Integration Testing)**

最後のステップです。
ここでは、Step 7で個別に検証した「View（コンポーネント）」、「Adapter（Hook）」、「Logic（Service）」、「Infra（Repository）」をすべて結合し、**「ユーザーが画面を操作した結果、裏側で正しくAPIが呼ばれるか」**という一連の流れ（End-to-Endに近い振る舞い）を検証します。

Step 6の実装でクラス間の依存（`new`）が固定されているため、ここでは一番外側にある **`global.fetch` (ネットワーク層)** をMock化することで、全ての層が正しく連携しているかテストします。

---

### 1. Integration Test Scenario

BDAのStep 2で定義したシナリオの中で、最も複雑な**「タイマー完了と保存（Complete Timer）」**のフローを結合テストとして実装します。

**`src/app/focus/page.integration.test.tsx`**

```tsx
import { render, screen, fireEvent, act } from '@testing-library/react';
import FocusTimerPage from './page';

// 1. Setup Network Mock
// 実際の通信はさせず、fetchをジャックして成功レスポンスを返す
global.fetch = jest.fn(() =>
  Promise.resolve({
    ok: true,
    json: () => Promise.resolve({ id: "123", durationMinutes: 25, completedAt: "2024-01-01" }),
  })
) as jest.Mock;

describe('FocusTimer Feature Integration', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    jest.useFakeTimers(); // タイマー(setInterval)を制御下に置く
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  test('User can start timer and complete a session', async () => {
    // --- Phase 1: Initial Render ---
    render(<FocusTimerPage />);
    expect(screen.getByText('25:00')).toBeInTheDocument();
    expect(screen.getByText('Status: IDLE')).toBeInTheDocument();

    // --- Phase 2: User Interaction (Start) ---
    const startButton = screen.getByText('Start');
    fireEvent.click(startButton);

    // UIが即座に反応しているか (View -> Hook -> Service)
    expect(screen.getByText('Status: RUNNING')).toBeInTheDocument();

    // --- Phase 3: Time Travel (Simulation) ---
    // 25分経過させる (Tickループの検証)
    act(() => {
      jest.advanceTimersByTime(25 * 60 * 1000);
    });

    // --- Phase 4: Verification (Outcome) ---
    // 画面表示が00:00になっているか
    expect(screen.getByText('00:00')).toBeInTheDocument();
    
    // API保存処理が走ったか (Service -> Repository -> Network)
    expect(global.fetch).toHaveBeenCalledTimes(1);
    expect(global.fetch).toHaveBeenCalledWith(
      expect.stringContaining('/focus/record'),
      expect.objectContaining({
        method: 'POST',
        body: JSON.stringify({ durationMinutes: 25 })
      })
    );
  });
});

```

このテストが通るということは、**「ボタンを押してから、ロジックが走り、時間が経過し、最終的にAPIリクエストが飛ぶ」** という配線がどこも断線していないことを証明します。

---

### Step 8 の Definition of Done (完了の定義)

> **DoD (Step 8):**
> * 主要なユースケース（今回の場合は完了フロー）を通す結合テストが記述されていること。
> * `jest.useFakeTimers` などを用いて、時間経過や非同期処理を含むフローが制御・検証されていること。
> * モック化されたのはネットワーク層（`fetch`）のみであり、それ以外のアプリケーションコードは本物が動いていること。
> * テストが **PASS (Green)** すること。
> 
> 

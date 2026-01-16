## **Step 6: 実装すること (Implementation Phase)**

このステップでは、確立されたビジネスロジック（Brain）を、現実世界（API/UI）と接続します。
Mockではなく**「本物のRepository」**を実装し、ReactコンポーネントからServiceを利用できるように**「配線（Wiring）」**を行います。

OOP/DIの原則を守るため、Reactコンポーネントの中に直接 `new FocusTimerService(...)` を書くことは避け、**Adapterパターン**（Custom Hook）を用いて接続します。

---

### 1. Real Repository Implementation (インフラ層の実装)

Step 3で定義したインターフェース `IFocusRepository` に従って、実際のAPI通信を行うクラスを実装します。

**`src/features/focus/infra/FocusRepository.ts`**

```typescript
import { IFocusRepository } from '../domain/IFocusRepository';
import { FocusRecord, FocusRecordCreate } from '../domain/models';

/**
 * 実際のAPIエンドポイントと通信するリポジトリ実装。
 */
export class FocusRepository implements IFocusRepository {
  private readonly baseUrl: string;

  constructor(baseUrl: string = '/api') {
    this.baseUrl = baseUrl;
  }

  /** @inheritdoc */
  async saveRecord(record: FocusRecordCreate): Promise<FocusRecord> {
    const response = await fetch(`${this.baseUrl}/focus/record`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(record),
    });

    if (!response.ok) {
      throw new Error(`API Error: ${response.statusText}`);
    }

    return response.json();
  }
}

```

---

### 2. ViewModel / Adapter Implementation (接続層の実装)

ここが重要です。**「Plain Old TypeScript Object (Service)」** の変化を **「ReactのState更新」** に変換するためのアダプター（Custom Hook）を作ります。これがMVVMでいうViewModelの役割を果たします。

**`src/features/focus/useFocusTimer.ts`**

```typescript
import { useState, useEffect, useMemo, useRef } from 'react';
import { FocusTimerService } from './logic/FocusTimerService';
import { FocusRepository } from './infra/FocusRepository';
import { UseFocusTimerReturn } from './types';

/**
 * Service(Logic)とView(React)を接続するカスタムフック。
 * DIコンテナの役割も兼ね、インスタンスの生成とライフサイクルを管理します。
 */
export const useFocusTimer = (): UseFocusTimerReturn => {
  // 1. Dependency Injection (Composition Root)
  // Reactのレンダリング間でインスタンスを保持するためにuseMemo/useRefを使用
  const service = useMemo(() => {
    const repository = new FocusRepository(); // 本物のRepositoryを注入
    return new FocusTimerService(repository);
  }, []);

  // 2. State Syncing
  // Serviceの内部状態をReactのStateに同期させる
  // (実際のアプリではServiceがObserverパターン等を実装しているとより良いが、ここでは簡易的にTickごとに同期)
  const [state, setState] = useState(service.getState());

  // 3. Tick Loop & State Update
  useEffect(() => {
    const intervalId = setInterval(() => {
      // LogicのTickを実行
      service.tick();
      
      // 変更後の状態を取得してReactを再描画
      // (最適化のため、変更検知ロジックを入れるのが一般的)
      setState(service.getState());
    }, 1000);

    return () => clearInterval(intervalId);
  }, [service]);

  // 4. Action Binding
  // UIイベントをServiceのメソッドに委譲
  return {
    ...state,
    onStartClick: () => {
      service.start();
      setState(service.getState()); // 即時反映
    },
    onPauseClick: () => {
      service.pause();
      setState(service.getState());
    },
    onResumeClick: () => {
      service.start(); // Resume is same as start in this logic
      setState(service.getState());
    },
    onResetClick: () => {
      service.reset();
      setState(service.getState());
    },
    onTick: () => { /* useEffectで自動化されているため外部呼出し不要 */ }
  };
};

```

---

### 3. UI Implementation (View層の実装)

最後に、Step 3で設計したコンポーネントツリーを実装します。ロジックは全てHookに追い出してあるため、ここは純粋な「表示」と「イベント発火」のみになります。

**`src/app/focus/page.tsx`**

```tsx
'use client'; // Next.js App Router Client Component

import React from 'react';
import { useFocusTimer } from '@/features/focus/useFocusTimer';

export default function FocusTimerPage() {
  // Logicとの接続
  const { 
    remainingSeconds, 
    status, 
    onStartClick, 
    onPauseClick, 
    onResetClick 
  } = useFocusTimer();

  // Helper: 秒を MM:SS に変換
  const formatTime = (sec: number) => {
    const m = Math.floor(sec / 60).toString().padStart(2, '0');
    const s = (sec % 60).toString().padStart(2, '0');
    return `${m}:${s}`;
  };

  return (
    <div className="flex flex-col items-center justify-center h-screen bg-gray-100">
      <div className="p-8 bg-white rounded-xl shadow-lg text-center">
        <h1 className="text-2xl font-bold mb-4">Focus Timer</h1>
        
        {/* Status Display */}
        <div className="text-sm text-gray-500 mb-2">Status: {status}</div>
        
        {/* Timer Display */}
        <div className="text-6xl font-mono font-bold mb-8 text-blue-600">
          {formatTime(remainingSeconds)}
        </div>

        {/* Control Buttons */}
        <div className="space-x-4">
          {status === 'IDLE' && (
            <button 
              onClick={onStartClick}
              className="px-6 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
            >
              Start
            </button>
          )}

          {status === 'RUNNING' && (
            <button 
              onClick={onPauseClick}
              className="px-6 py-2 bg-yellow-500 text-white rounded hover:bg-yellow-600"
            >
              Pause
            </button>
          )}

          {status === 'PAUSED' && (
            <button 
              onClick={onStartClick} // Resume
              className="px-6 py-2 bg-green-500 text-white rounded hover:bg-green-600"
            >
              Resume
            </button>
          )}

          <button 
            onClick={onResetClick}
            className="px-6 py-2 border border-gray-300 rounded hover:bg-gray-50"
          >
            Reset
          </button>
        </div>
      </div>
    </div>
  );
}

```

---

### Step 6 の Definition of Done (完了の定義)

> **DoD (Step 6):**
> * インフラ層（`FocusRepository`）が実装され、APIコールが可能であること（疎通はStep 8で行う）。
> * Adapter層（`useFocusTimer`）により、OOPのServiceクラスがReactのライフサイクルと正しく統合されていること。
> * UI層（`page.tsx`）が実装され、画面が表示されること。
> * ブラウザでボタンを押すと、画面上のタイマーが減り始めること（手動確認でOK）。
> 
> 

## Step 6.5: Linterによる静的解析 (Static Analysis)

人間がコードレビューやテストをする前に、機械（Linter）がコードの品質を保証します。
Step 1 の `guidelines` セクションで定義したルール（Next.js / TypeScript / Prettier）への適合を確認します。

### 1. 実行コマンド (Execution)

プロジェクトルートで以下のコマンドを実行し、構文エラーやスタイル違反がないか確認します。

```bash
# Linting (論理的・構文的ミスの検出)
npm run lint

# Formatting Check (コードスタイルの確認)
npx prettier --check .

```

### 2. 修正プロセス (Fix Process)

もしエラーが出た場合は、以下のように対応します。

**ケースA: 自動修正可能なスタイル違反**

```bash
npm run lint:fix
# または
npx prettier --write .

```

**ケースB: 論理エラー（未使用変数、型不一致、Hookの依存配列漏れなど）**

* **エラー例:** `React Hook useEffect has a missing dependency: 'service'.`
* **対応:** コードを手動で修正します。この段階で修正することで、Step 7以降の不可解なバグを防ぎます。

---

### Step 6.5 の Definition of Done (完了の定義)

> **DoD (Step 6.5):**
> * `npm run lint` が **エラー0 (Exit Code 0)** で終了すること。
> * `TypeScript` のコンパイルエラー（型エラー）が一つもないこと。
> * コードフォーマット（Prettier）が適用済みであること。
> * 未使用の `import` や変数（Dead Code）が削除されていること。
> 
> 
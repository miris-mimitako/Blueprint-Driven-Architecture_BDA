## Step 3: 詳細設計 (Detailed Design - OOP/DI Style)

BDAの `blueprint.yaml` を入力とし、実際のコード構造（スケルトン）を作成します。ここではロジックを実装せず、**「型」と「説明（ドキュメント）」と「依存関係の注入経路」**のみを構築します。

### 1. Repository Interface Definition (外部依存の抽象化)

APIやDBへのアクセスは必ずインターフェース化し、Mockに差し替え可能にします。

**`src/features/focus/domain/IFocusRepository.ts`**

```typescript
import { FocusRecord, FocusRecordCreate } from './models';

/**
 * 集中セッションの記録を行うリポジトリインターフェース。
 * 外部APIまたはローカルDBへの永続化を担当します。
 */
export interface IFocusRepository {
  /**
   * 完了した集中セッションを記録します。
   * * @param record - 作成するセッションの実績データ
   * @returns 保存されたセッションデータ（ID付与済み）
   * @throws {Error} 通信エラーまたはバリデーションエラー時
   */
  saveRecord(record: FocusRecordCreate): Promise<FocusRecord>;
}

```

### 2. Service Interface Definition (ビジネスロジックの抽象化)

ViewModelやUIが利用する「機能の振る舞い」を定義します。

**`src/features/focus/domain/IFocusTimerService.ts`**

```typescript
import { FocusTimerState } from './types';

/**
 * FocusTimerのビジネスロジックを管理するサービス。
 * 状態管理、タイマー制御、完了時の保存処理をカプセル化します。
 */
export interface IFocusTimerService {
  /** 現在の状態を取得します（Reactiveな参照ではない点に注意） */
  getState(): FocusTimerState;

  /**
   * タイマーを開始します。
   * ステータスが IDLE または PAUSED の場合のみ有効です。
   */
  start(): void;

  /**
   * タイマーを一時停止します。
   * ステータスが RUNNING の場合のみ有効です。
   */
  pause(): void;

  /**
   * タイマーをリセットし、初期状態(IDLE)に戻します。
   */
  reset(): void;

  /**
   * システムチック（1秒経過）を処理します。
   * 残り時間を減算し、0になった場合は完了処理を実行します。
   */
  tick(): void;
}

```

### 3. Skeleton Implementation with DI (実装のスケルトン)

インターフェースを実装したクラスを作成します。ロジックは空ですが、依存性注入（DI）の構造とDocstringによる補完準備を完了させます。

**`src/features/focus/logic/FocusTimerService.ts`**

```typescript
import { IFocusTimerService } from '../domain/IFocusTimerService';
import { IFocusRepository } from '../domain/IFocusRepository';
import { FocusTimerState } from '../types';

/**
 * FocusTimerのビジネスロジック実装クラス。
 */
export class FocusTimerService implements IFocusTimerService {
  private state: FocusTimerState;

  /**
   * @param repository - 実績保存用のリポジトリ（DIにより注入）
   * @param initialState - 初期状態（テスト時に注入可能）
   */
  constructor(
    private readonly repository: IFocusRepository,
    initialState?: FocusTimerState
  ) {
    // デフォルト値の設定
    this.state = initialState ?? {
      remainingSeconds: 1500,
      status: 'IDLE',
      sessionCount: 0,
      isSaving: false
    };
  }

  /** @inheritdoc */
  getState(): FocusTimerState {
    return { ...this.state };
  }

  /** @inheritdoc */
  start(): void {
    // TODO: Implement Step 6
    throw new Error("Not implemented yet");
  }

  /** @inheritdoc */
  pause(): void {
    // TODO: Implement Step 6
    throw new Error("Not implemented yet");
  }

  /** @inheritdoc */
  reset(): void {
    // TODO: Implement Step 6
    throw new Error("Not implemented yet");
  }

  /** @inheritdoc */
  tick(): void {
    // TODO: Implement Step 6
    // Description: 残り時間を減らし、0でrepository.saveRecordを呼ぶロジックが必要
    throw new Error("Not implemented yet");
  }
}

```

---

### Step 3 の Definition of Done (修正版)

この工程が完了したと言える条件は以下の通りです。

> **DoD (Step 3 - OOP/DI Edition):**
> 1. **Interface Defined:** 外部依存（Repository）とロジック（Service）のインターフェースファイルが存在すること。
> 2. **Docstrings:** 全てのインターフェースメソッドに、引数・戻り値・例外・振る舞いの説明（Docstring）が記述されていること。IDEでホバーした際に説明が表示されること。
> 3. **Skeleton Created:** 実装クラス（Skeleton）が作成され、インターフェースを `implements` していること。
> 4. **DI Ready:** 実装クラスが、依存オブジェクトをコンストラクタ経由で受け取る（DI）構造になっていること。内部で `new Dependency()` をしていないこと。
> 5. **Compilable:** 中身は `throw Error` だが、文法エラーがなくコンパイル（トランスパイル）が通る状態であること。
> 
> 

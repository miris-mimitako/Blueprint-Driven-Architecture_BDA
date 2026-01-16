# コーディング実装規約（BDA v3.0.0準拠）構成案

## 1. 概要 (Overview)

本規約は、**「手戻りの排除」**と**「AIとの協働効率の最大化」**を目的とした開発標準である。
アーキテクチャスタイルとして **Blueprint-Driven Architecture (BDA) v3.0.0** を採用し、以下の3つの原則を徹底する。

1. **Blueprint is the App:** `blueprint.yaml`（青写真）こそがアプリケーションの正体であり、ソースコードはその出力結果に過ぎない。
2. **Schema First, Logic Second:** データ構造と入出力を先に確定させ、ロジック実装はその隙間を埋める作業とする。
3. **Simulation Driven:** 実装前にシナリオ（テストケース）を定義し、検証可能な状態から開発をスタートする。

開発フローは厳格な **8ステップ（+Linting）** で構成され、各工程には明確な「完了の定義 (Definition of Done)」が設けられる。これにより、ロジックの不整合や設計の曖昧さを初期段階で完全に排除する。

---

## 2. 目次 (Table of Contents)

### 第1章 基本原則とアーキテクチャ (Philosophy & Architecture)

* **1.1 BDA v3.0.0 概要**
* フリースタイルベースラインの定義
* ディレクトリ構造と役割（Domain, Features, UI, Infra）


* **1.2 アーキテクチャレイヤー**
* **Domain:** 言語非依存のデータ型定義
* **Logic (Service):** 純粋なビジネスロジック（POJO/POTO）
* **Adapter (Hook/ViewModel):** ロジックとUIの接続層
* **View (Component):** 表示とイベント発火のみを担当
* **Infra (Repository):** 外部システムとの通信



### 第2章 開発ワークフロー (The 8-Step Workflow)

本規約の中核となるプロセス定義。

* **2.1 [Step 1] 要件定義 (Requirement Definition)**
* 自然言語による機能スコープの確定
* Input/Outputの洗い出し


* **2.2 [Step 2] ビジネスロジック確定 (Blueprint Definition)**
* `blueprint.yaml` の作成（Domain, API, Features, Scenarios）
* **成果物:** 完全な仕様が記述されたYAML


* **2.3 [Step 3] 詳細設計 (Detailed Design & Skeleton)**
* OOP/DI原則に基づいたインターフェース設計
* Docstrings（コード補完用コメント）の記述
* スケルトンコードの作成（実装は空の状態）


* **2.4 [Step 4] テスト先行 (Red Phase)**
* YAMLシナリオに基づくテストコードの実装
* 「未実装エラー(NotImplemented)」による不合格の確認


* **2.5 [Step 5] ロジック確立 (Green Phase)**
* Mockを活用した純粋ロジックの実装
* ビジネスロジック単体でのテスト合格


* **2.6 [Step 6] 実装と接続 (Implementation & Wiring)**
* Infra層（Real Repository）の実装
* Adapter層によるDIとState管理
* View層の実装
* **[Step 6.5] 静的解析 (Linting/Formatting)**


* **2.7 [Step 7] 単体テスト (Unit Testing)**
* Adapter (Hook) のテスト
* View (Component) のレンダリングテスト


* **2.8 [Step 8] 結合テスト (Integration Testing)**
* ネットワーク層のみをMock化したE2Eフロー検証



### 第3章 コーディング詳細規約 (Coding Standards)

* **3.1 オブジェクト指向設計 (OOP Rules)**
* Interface/Abstractの強制
* Dependency Injection (DI) の徹底（内部での`new`禁止）


* **3.2 命名規則 (Naming Conventions)**
* File/Directory Naming
* Interface (`I` prefix) / Implementation naming


* **3.3 コメントとドキュメンテーション**
* Docstringの必須項目
* AIへのコンテキスト提供用コメント



### 第4章 品質保証 (Quality Assurance)

* **4.1 テスト戦略**
* テストピラミッドの考え方（Logic重視）
* Mockの使用基準


* **4.2 定義された完了基準 (Definition of Done)**
* 各ステップにおけるDoDチェックリスト


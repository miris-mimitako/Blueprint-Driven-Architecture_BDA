# Blueprint-Driven Architecture (BDA) Project

## Overview

このリポジトリは **BDA 3.3.0 (Modular DDD)** の仕様書とサンプル実装を含みます。
BDAは、YAMLベースの設計ファイルからDDD準拠のアプリケーションを開発するためのアーキテクチャフレームワークです。

## Project Structure

```
/
├── BDA_specification.md      # BDA仕様書 (v3.3.0)
├── codingStepSpecification.md # 実装規約 (8-Step Workflow)
├── CLAUDE.md                 # このファイル
├── .claude/
│   └── commands/             # BDA Skills (Slash Commands)
│       ├── bda-validate.md   # 設計ファイル検証
│       └── bda-inventory.md  # Inventory自動生成
└── sample/
    └── design/               # サンプル設計ファイル
        ├── blueprint.yaml    # Entry Point
        ├── guidelines/
        ├── contexts/         # Bounded Contexts
        └── interfaces/       # Controllers, ViewModels
```

## Core Concepts

### Reference Transparency
設計ファイルは `$ref` + `note` パターンで分割し、AIが部分的に読み込んでも全体像を把握できる状態を維持する。

```yaml
contexts:
  - $ref: "./contexts/todo/main.yaml"
    note: "【タスク管理】TodoのCRUD、ステータス変更を行うコア機能"
```

### DDD Layers

| Layer | 責務 | 設計ファイル |
|-------|------|-------------|
| Domain | Aggregate, Event, Repository Interface | `contexts/{name}/domain.yaml` |
| Application | UseCase, Query, Event Handler | `contexts/{name}/application.yaml` |
| Infrastructure | Repository Impl, DB, External APIs | (実装のみ) |
| Interfaces | Controller, ViewModel | `interfaces/` |

## Development Workflow

1. **Step 1**: 要件定義 → Bounded Context特定
2. **Step 2**: ドメインモデル確定 → domain.yaml, application.yaml作成
3. **Step 2.5**: Implementation Inventory生成 → `/bda-inventory`
4. **Step 3-5**: Skeleton → Red → Green
5. **Step 6**: Infrastructure & Interfaces実装
6. **Step 7-9**: Testing & Verification → `/bda-check`

## Available Commands

### Phase 1: 基盤

| Command | Description |
|---------|-------------|
| `/bda-validate` | 設計ファイルの整合性チェック（$ref, 必須項目, Context間依存） |
| `/bda-inventory {context}` | 設計ファイルからImplementation Inventoryを自動生成 |

### Phase 2: 生産性向上

| Command | Description |
|---------|-------------|
| `/bda-init context {name}` | 新規Bounded Contextの雛形を生成（main.yaml, domain.yaml, application.yaml） |
| `/bda-init feature {name}` | 新規Frontend Featureの雛形を生成 |
| `/bda-skeleton {context}` | InventoryからDDD準拠のコードスケルトンを生成 |

### Phase 3: 品質保証

| Command | Description |
|---------|-------------|
| `/bda-test {context}` | 設計ファイルからテストコードを自動生成（Unit/Integration） |
| `/bda-check {context}` | Inventory vs 実装コードの進捗チェック・差分レポート |
| `/bda-check --all` | 全Contextの進捗を一括チェック |
| `/bda-compliance` | DDDアーキテクチャ準拠チェック（依存方向、レイヤー分離） |
| `/bda-compliance --fix-suggestions` | 違反箇所の修正提案を表示 |

### Usage Examples

```bash
# === Phase 1: 設計 ===
# 設計を検証
/bda-validate

# Inventoryを生成
/bda-inventory todo

# === Phase 2: 実装 ===
# 新規Contextを作成
/bda-init context payment

# コードスケルトンを生成
/bda-skeleton payment

# === Phase 3: 品質保証 ===
# テスト自動生成
/bda-test payment
/bda-test payment --layer application

# 進捗チェック
/bda-check payment
/bda-check --all

# アーキテクチャ準拠チェック
/bda-compliance
/bda-compliance --fix-suggestions
```

## Coding Standards

- **Domain First**: ビジネスドメインを先に確定、技術詳細は後
- **Interface Segregation**: Repository は Domain Layer に Interface、Infrastructure に実装
- **Event-Driven**: Context間連携は Domain Event を介して疎結合に

## Key Files to Read

新機能を実装する際は、以下の順で読むこと:

1. `design/blueprint.yaml` - 全体構造の把握
2. `design/contexts/{target}/main.yaml` - 対象Contextの概要
3. `design/contexts/{target}/domain.yaml` - Aggregate, Event定義
4. `design/contexts/{target}/application.yaml` - UseCase, Query定義
5. `codingStepSpecification.md` - 実装手順の確認

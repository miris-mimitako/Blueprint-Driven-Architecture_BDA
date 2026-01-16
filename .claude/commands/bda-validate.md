# /bda-validate - BDA設計ファイル検証

設計ファイル（design/）の整合性をチェックし、問題を報告します。

## 実行手順

### 1. Entry Point の確認

`design/blueprint.yaml` を読み込み、以下を確認:

- [ ] `blueprint:` バージョンが "3.3.0" 以上
- [ ] `meta:` セクションが存在（appName, version, architecture）
- [ ] `contexts:` セクションが存在

### 2. $ref 整合性チェック

blueprint.yaml 内の全ての `$ref` について:

```
FOR EACH $ref in blueprint.yaml:
  1. 参照先ファイルが存在するか確認
  2. `note:` が併記されているか確認
  3. 参照先ファイル内の $ref も再帰的にチェック
```

**エラー例:**
- `[ERROR] $ref "./contexts/identity/main.yaml" - ファイルが存在しません`
- `[WARN] $ref "./contexts/todo/main.yaml" - note が未記載です`

### 3. Context 構造チェック

各 Context (`contexts/{name}/`) について:

**必須ファイル:**
- [ ] `main.yaml` - Context定義
- [ ] `domain.yaml` - ドメインモデル
- [ ] `application.yaml` - ユースケース

**domain.yaml 必須項目:**
- [ ] `aggregates:` - 少なくとも1つのAggregate
  - [ ] 各Aggregateに `name`, `properties`, `behaviors`
  - [ ] `root: true` を持つAggregateが存在
- [ ] `repositories:` - Aggregate毎のRepository Interface

**application.yaml 必須項目:**
- [ ] `useCases:` - 少なくとも1つのUseCase
  - [ ] 各UseCaseに `name`, `input`, `output`

### 4. Domain Event 整合性

```
FOR EACH aggregate.behavior WHERE emits IS DEFINED:
  1. emits で指定された Event が domain.yaml の events: に存在するか
  2. Event の properties が定義されているか
```

**エラー例:**
- `[ERROR] Todo.complete emits "TodoCompleted" but event is not defined in events:`

### 5. Context 間依存チェック

```
FOR EACH application.yaml:
  IF trigger: "Event: {OtherContext}.{EventName}" EXISTS:
    1. 参照先 Context が contexts: に存在するか
    2. 参照先 Event が定義されているか
```

### 6. レポート出力

```markdown
# BDA Validation Report

## Summary
- Contexts checked: X
- Errors: Y
- Warnings: Z

## Errors
| File | Line | Message |
|------|------|---------|
| ... | ... | ... |

## Warnings
| File | Line | Message |
|------|------|---------|
| ... | ... | ... |

## Passed Checks
- [x] All $ref files exist
- [x] All notes present
- ...
```

---

## 使用例

```
/bda-validate
```

引数なしで実行すると、`design/` ディレクトリ全体を検証します。

```
/bda-validate contexts/todo
```

特定のContextのみを検証します。

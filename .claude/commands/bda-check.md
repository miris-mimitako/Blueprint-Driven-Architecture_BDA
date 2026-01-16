# /bda-check - 実装進捗チェック

Implementation Inventoryと実装コードを照合し、進捗状況と差分を報告します。

## 引数

```
/bda-check {context_name}
/bda-check --all
/bda-check {context_name} --update-inventory
```

## 実行手順

### 1. Inventoryファイルの読み込み

```
_docs/inventory/{context}_inventory.md
```

から全アイテムを抽出します。

### 2. 実装ファイルの存在チェック

各Inventory ItemのFileカラムに対応するファイルが存在するか確認:

```
FOR EACH item in Inventory:
  file_path = resolve_path(item.File)  # e.g., domain/todo/entity.py

  IF file_exists(file_path):
    IF contains_implementation(file_path):  # NotImplementedError がない
      status = "✅ Implemented"
    ELSE:
      status = "🔶 Skeleton"
  ELSE:
    status = "❌ Missing"
```

### 3. テストファイルの存在チェック

```
FOR EACH item WHERE item.Test != "-":
  test_path = resolve_test_path(item.File)
  # e.g., domain/todo/entity.py → tests/domain/todo/test_entity.py

  IF file_exists(test_path):
    IF tests_pass(test_path):
      test_status = "✅ Pass"
    ELSE:
      test_status = "🔴 Fail"
  ELSE:
    test_status = "❌ No Test"
```

### 4. 進捗レポート生成

```markdown
# BDA Implementation Check Report

**Context:** {ContextName}
**Generated:** YYYY-MM-DD HH:MM
**Inventory:** _docs/inventory/{context}_inventory.md

---

## Summary

| Metric | Count | Percentage |
|--------|-------|------------|
| Total Items | 29 | 100% |
| ✅ Implemented | 5 | 17% |
| 🔶 Skeleton | 10 | 34% |
| ❌ Missing | 14 | 48% |

| Test Status | Count |
|-------------|-------|
| ✅ Pass | 3 |
| 🔴 Fail | 2 |
| ❌ No Test | 24 |

---

## Progress by Layer

### Domain Layer (8 items)

| ID | Item | File | Status | Test |
|----|------|------|--------|------|
| D-001 | User Aggregate | domain/identity/entity.py | ✅ Implemented | ✅ Pass |
| D-002 | UserId | domain/identity/value_objects.py | ✅ Implemented | ✅ Pass |
| D-003 | Email | domain/identity/value_objects.py | 🔶 Skeleton | ❌ No Test |
| D-004 | UserRegistered | domain/identity/events.py | ❌ Missing | ❌ No Test |
| ... | ... | ... | ... | ... |

**Progress: 2/8 (25%)**

### Application Layer (6 items)

| ID | Item | File | Status | Test |
|----|------|------|--------|------|
| A-001 | RegisterUserUseCase | application/identity/register_user.py | 🔶 Skeleton | 🔴 Fail |
| ... | ... | ... | ... | ... |

**Progress: 0/6 (0%)**

### Infrastructure Layer (5 items)
...

### Interfaces Layer (6 items)
...

### Platform-Dependent (4 items)

| ID | Item | Verification | Status | 確認日 |
|----|------|--------------|--------|--------|
| P-001 | DB Migration | alembic upgrade head | ❌ Not Verified | - |
| ... | ... | ... | ... | ... |

---

## Action Items

### 🔴 High Priority (Missing Core Components)

1. **D-004 UserRegistered Event** - Domain Eventが未実装
   - File: `domain/identity/events.py`
   - Required by: RegisterUserUseCase

2. **A-001 RegisterUserUseCase** - テスト失敗中
   - File: `application/identity/register_user.py`
   - Error: `NotImplementedError at line 45`

### 🟡 Medium Priority (Skeleton Only)

1. **D-003 Email Value Object** - バリデーション未実装
2. ...

### 🟢 Completed

- D-001 User Aggregate
- D-002 UserId

---

## Next Steps

1. Domain Layerの残り6アイテムを実装 (Step 5)
2. 全Domain Layerアイテムのテストを作成 (Step 4)
3. Application Layerに進む
```

---

## 実装状態の判定ロジック

### Python ファイルの場合

```python
def check_implementation_status(file_path: str) -> str:
    content = read_file(file_path)

    # NotImplementedError があればSkeleton
    if "NotImplementedError" in content or "raise NotImplementedError" in content:
        return "Skeleton"

    # pass のみの関数があればSkeleton
    if re.search(r"def \w+\([^)]*\):\s*pass", content):
        return "Skeleton"

    # TODO: が多数あればSkeleton
    if content.count("# TODO:") > 3:
        return "Skeleton"

    return "Implemented"
```

### TypeScript ファイルの場合

```typescript
function checkImplementationStatus(filePath: string): string {
    const content = readFile(filePath);

    // throw new Error("Not implemented") があればSkeleton
    if (content.includes("Not implemented") || content.includes("TODO")) {
        return "Skeleton";
    }

    return "Implemented";
}
```

---

## オプション

| Option | Description |
|--------|-------------|
| `--all` | 全Contextをチェック |
| `--update-inventory` | Inventoryファイルの Status を自動更新 |
| `--ci` | CI用出力（exit code: 0=all done, 1=incomplete） |
| `--json` | JSON形式で出力 |

## Inventory自動更新

`--update-inventory` オプション使用時:

```markdown
# Before
| D-001 | User | domain/identity/entity.py | Unit | [ ] |

# After
| D-001 | User | domain/identity/entity.py | Unit | [x] |
```

ステータス記号:
- `[ ]` - 未着手
- `[S]` - Skeleton作成済み
- `[x]` - 実装完了
- `[T]` - テスト作成済み

---

## CI/CD 統合

```yaml
# .github/workflows/bda-check.yml
- name: BDA Implementation Check
  run: |
    claude /bda-check --all --ci
```

失敗条件:
- Missing items > 0 (リリース前)
- Test failures > 0

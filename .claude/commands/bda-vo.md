# /bda-vo - Value Object 生成

DDD準拠のValue Objectコードを自動生成します。
3原則（不変性・完全性・カプセル化）を満たす実装を出力します。

## 引数

```
/bda-vo {ValueObjectName}
/bda-vo {ValueObjectName} --lang {typescript|python}
/bda-vo {ValueObjectName} --type {string|number|object}
```

## 前提条件

- `design/guidelines/coding-standards.yaml` が存在すること
- `design/guidelines/tech_stack.yaml` で技術スタックが定義されていること

---

## 実行フロー

1. **入力を解析**
   - `$ARGUMENTS` から Value Object 名を取得
   - オプションから言語・ベース型を取得（デフォルト: TypeScript, string）

2. **ユーザーに確認**
   以下を質問:
   - ベース型（string / number / object）
   - 検証ルール（正規表現、範囲、必須フィールドなど）
   - エラーメッセージ（日本語）

3. **coding-standards.yaml を参照**
   - `design/guidelines/coding-standards.yaml` を読み込む
   - `value_objects.typescript_template` または `python_template` を取得

4. **コードを生成**
   - テンプレートのプレースホルダを置換
   - 検証ロジックを組み込む

5. **出力先を提案**
   - Backend: `backend/src/domain/{context}/value_objects.ts` または `.py`
   - 共通: `shared/domain/value_objects/`

---

## 生成テンプレート

### TypeScript (Zod + Branded Type)

```typescript
import { z } from 'zod';

// ================================
// Branded Type 定義
// ================================
declare const {Name}Brand: unique symbol;
type {Name} = {BaseType} & { readonly [{Name}Brand]: never };

// ================================
// Zod Schema（検証ルール）
// ================================
const {camelName}Schema = z
  .{baseValidator}()
  {validationChain}
  .transform((val): {Name} => val as {Name});

// ================================
// Factory（Parse, don't validate）
// ================================
export const {Name} = {
  /**
   * 検証して {Name} を生成（失敗時は例外）
   * @throws ZodError 検証失敗時
   */
  parse: (value: unknown): {Name} => {
    return {camelName}Schema.parse(value);
  },

  /**
   * 検証して Result型で返す（例外を投げない）
   */
  safeParse: (value: unknown):
    | { success: true; data: {Name} }
    | { success: false; error: z.ZodError } => {
    return {camelName}Schema.safeParse(value);
  },

  /**
   * 型ガード
   */
  is: (value: unknown): value is {Name} => {
    return {camelName}Schema.safeParse(value).success;
  },
} as const;

// 型エクスポート
export type { {Name} };
```

### Python (Pydantic v2)

```python
from typing import Self
from pydantic import GetCoreSchemaHandler
from pydantic_core import CoreSchema, core_schema
import re


class {Name}(str):
    """
    {description}

    不変性: strを継承し、変更不可
    完全性: __new__で検証、不正値は生成不可
    カプセル化: 専用型として区別

    Validation:
        {validation_description}
    """
    _pattern = re.compile(r"{regex_pattern}")

    @classmethod
    def _validate(cls, value: str) -> str:
        if not cls._pattern.match(value):
            raise ValueError("{error_message}")
        return value

    def __new__(cls, value: str) -> Self:
        validated = cls._validate(value)
        return super().__new__(cls, validated)

    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type: type, handler: GetCoreSchemaHandler
    ) -> CoreSchema:
        return core_schema.no_info_after_validator_function(
            cls._validate,
            core_schema.str_schema(),
            serialization=core_schema.to_string_ser_schema(),
        )
```

---

## プレースホルダ一覧

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `{Name}` | PascalCase名 | `EmployeeId` |
| `{camelName}` | camelCase名 | `employeeId` |
| `{BaseType}` | ベース型 | `string`, `number` |
| `{baseValidator}` | Zodバリデータ | `string`, `number` |
| `{validationChain}` | 検証チェーン | `.regex(/^EMP-\d{8}$/, 'msg')` |
| `{description}` | 説明文 | `社員ID（EMP- + 8桁数字）` |
| `{regex_pattern}` | 正規表現 | `^EMP-\d{8}$` |
| `{error_message}` | エラーメッセージ | `社員IDは...` |

---

## 使用例

### 例1: 社員ID

```
/bda-vo EmployeeId
```

質問への回答:
- ベース型: `string`
- 検証ルール: `^EMP-\d{8}$`
- エラーメッセージ: `社員IDは "EMP-" + 8桁の数字である必要があります`

### 例2: 金額

```
/bda-vo Money --type object
```

質問への回答:
- フィールド: `amount: number (>= 0)`, `currency: 'JPY' | 'USD'`
- エラーメッセージ: `金額は0以上である必要があります`

### 例3: メールアドレス

```
/bda-vo Email
```

質問への回答:
- ベース型: `string`
- 検証ルール: Zodビルトインの `email()` を使用
- 正規化: 小文字変換

---

## 3原則チェックリスト

生成されたコードは以下を満たすことを確認:

- [ ] **不変性**: `readonly` / `frozen=True` が適用されている
- [ ] **完全性**: Factory経由でのみ生成可能、直接newできない
- [ ] **カプセル化**: Branded Type / 専用クラスで型が区別されている

---

## オプション

| Option | Description |
|--------|-------------|
| `--lang {ts\|py}` | 出力言語を指定（デフォルト: tech_stack.yamlに従う） |
| `--type {string\|number\|object}` | ベース型を指定 |
| `--output {path}` | 出力先パスを指定 |
| `--dry-run` | コードを出力するが、ファイルには書き込まない |

---

## 生成後の作業

1. 生成されたファイルをレビュー
2. 必要に応じてテストを追加
3. `/bda-check` で進捗を確認

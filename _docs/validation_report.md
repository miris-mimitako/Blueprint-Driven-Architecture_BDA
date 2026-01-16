# BDA Validation Report

**Generated:** 2026-01-16
**Target:** sample/design/

---

## Summary

| Metric | Count |
|--------|-------|
| Contexts checked | 1 (todo) |
| **Errors** | **4** |
| Warnings | 0 |

---

## Errors

| File | $ref | Message |
|------|------|---------|
| blueprint.yaml:19 | `./contexts/identity/main.yaml` | ファイルが存在しません |
| blueprint.yaml:27 | `./workflows/onboarding.yaml` | ファイルが存在しません |
| blueprint.yaml:34 | `./interfaces/backend/todo_controller.yaml` | ファイルが存在しません |
| blueprint.yaml:43 | `./interfaces/frontend/routing.yaml` | ファイルが存在しません |

---

## Warnings

なし

---

## Passed Checks

### Blueprint Structure
- [x] `blueprint:` バージョンが 3.3.0
- [x] `meta:` セクションが存在（appName, version, architecture）
- [x] `contexts:` セクションが存在
- [x] 全ての `$ref` に `note:` が併記されている

### TodoContext (contexts/todo/)
- [x] `main.yaml` が存在
- [x] `domain.yaml` が存在
- [x] `application.yaml` が存在

### TodoContext - domain.yaml
- [x] `aggregates:` が定義されている
- [x] Aggregate に `name`, `properties`, `behaviors` が存在
- [x] `root: true` を持つ Aggregate が存在
- [x] `events:` が定義されている
- [x] `repositories:` が定義されている

### TodoContext - application.yaml
- [x] `useCases:` が定義されている
- [x] 各 UseCase に `name`, `input`, `output` が存在
- [x] `queries:` が定義されている

### Domain Event Integrity
- [x] `Todo.complete` emits `TodoCompleted` → events: に定義あり

### Frontend Interface
- [x] `interfaces/frontend/dashboard.yaml` が存在
- [x] state, inputs, scenarios が定義されている

---

## Existing Files

```
sample/design/
├── blueprint.yaml              ✅
├── guidelines/
│   └── tech_stack.yaml         ✅
├── contexts/
│   ├── identity/
│   │   └── main.yaml           ❌ 不足
│   └── todo/
│       ├── main.yaml           ✅
│       ├── domain.yaml         ✅
│       └── application.yaml    ✅
├── workflows/
│   └── onboarding.yaml         ❌ 不足
└── interfaces/
    ├── backend/
    │   └── todo_controller.yaml ❌ 不足
    └── frontend/
        ├── dashboard.yaml      ✅
        └── routing.yaml        ❌ 不足
```

---

## Recommendations

1. **IdentityContext を作成する** - 認証機能が必要な場合
   ```
   contexts/identity/
   ├── main.yaml
   ├── domain.yaml      # User Aggregate
   └── application.yaml # RegisterUser, Login UseCases
   ```

2. **todo_controller.yaml を作成する** - REST API定義
   ```yaml
   # interfaces/backend/todo_controller.yaml
   name: TodoController
   path: "/todos"
   handlers:
     - { method: POST, path: "/", useCase: CreateTodo }
     - { method: PUT, path: "/{id}/complete", useCase: CompleteTodo }
     - { method: GET, path: "/stats", query: GetDashboardStats }
   ```

3. **不要な$refを削除する** - 未実装のものは一時的にコメントアウト

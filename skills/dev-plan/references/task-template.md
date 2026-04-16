# タスクファイル テンプレート

## テンプレート

各タスクファイルは以下のフォーマットで `docs/dev/plans/<plan-name>/tasks/NNN-task-name.md` に保存する。

```markdown
---
id: "NNN"
title: "タスク名"
status: pending
priority: 1
dependencies: []
estimated_complexity: medium
---

# Task: タスク名

## Goal

[このタスクが達成すべきこと（1-2文で簡潔に）]

## Interfaces

[このタスクで実装/使用するインターフェース・型定義をコードブロックで記述]

> 信号機: 各インターフェース要素に 🔵🟡🔴 を付与

## Test Strategy

- [ ] テストケース1: [具体的な振る舞いの記述]
- [ ] テストケース2: [具体的な振る舞いの記述]
- [ ] エッジケース: [境界値・異常系の記述]

## Implementation Notes

- 参照すべき既存コード: [ファイルパス]
- 実装のヒント: [アプローチの要点]
- 注意事項: [既存コードとの整合性、副作用等]

## Files

- 新規: [作成するファイルパス]
- 変更: [変更するファイルパス]
- テスト: [テストファイルパス]
```

## フロントマター仕様

### id (必須)
- 3桁の連番文字列: "001", "002", ...
- Plan内で一意

### title (必須)
- タスクの簡潔な名称
- 動詞で始める（「認証サービスを実装」「バリデーションを追加」等）

### status (必須)
- `pending`: 未着手
- `in_progress`: 実装中
- `done`: 完了（テスト通過済み）

### priority (必須)
- `1`: 最高（基盤・インターフェース定義等）
- `2`: 高（コア機能）
- `3`: 中（標準機能）
- `4`: 低（補助機能）
- `5`: 最低（オプション）

### dependencies (必須)
- このタスクが依存する他のタスクIDの配列
- 例: `["001", "002"]`
- 空配列 `[]` は依存なし

### estimated_complexity (必須)
- `low`: テスト+実装をバッチ生成可能。シンプルなCRUD、ユーティリティ関数等
- `medium`: 標準的なRed-Green-Refactorサイクル。通常のビジネスロジック等
- `high`: 段階的にテスト追加・実装。複雑なアルゴリズム、複数の外部依存等

## 各セクションの記述ガイド

### Goal
- 1-2文で完結する
- 「何を」「なぜ」が分かるように
- テスト可能な形で記述する

### Interfaces
- 言語に応じたインターフェース/型定義を記述
- TypeScript: `interface`, `type`
- Rust: `trait`, `struct`
- Go: `interface`, `struct`
- Python: `Protocol`, `dataclass`, `TypedDict`
- 各要素に信号機マークを付与する

信号機の付与例:

```typescript
interface AuthService {
  login(email: string, password: string): Promise<AuthResult>;  // 🔵 既存パターン準拠
  logout(token: string): Promise<void>;                          // 🔵 既存パターン準拠
  refreshToken(token: string): Promise<AuthResult>;              // 🟡 ベストプラクティスから推定
  revokeAllSessions(userId: string): Promise<void>;              // 🔴 セキュリティポリシー要確認
}
```

### Test Strategy
- チェックリスト形式で列挙
- 「正しく動作する」のような曖昧表現を避ける
- 具体的な入力→期待出力のペアで記述する
- 正常系、異常系、エッジケースを含める

良い例:
```
- [ ] 有効なメール+パスワードでログインするとAuthResultが返る
- [ ] 無効なパスワードでログインするとAuthErrorが投げられる
- [ ] 空文字のメールでログインするとValidationErrorが投げられる
- [ ] トークン期限切れ時にrefreshTokenで新しいトークンが取得できる
```

悪い例:
```
- [ ] ログイン機能が正しく動作する
- [ ] エラーハンドリングが適切
```

### Implementation Notes
- 既存コードへのファイルパスを具体的に指定
- 既存パターンの踏襲ポイントを明記
- 使用するライブラリ・ユーティリティがあれば記載

### Files
- 相対パスで記載
- 「新規」「変更」「テスト」に分類
- テストファイルは対象実装ファイルと1:1対応を推奨

## タスク分解の指針

### 分割の粒度
- 1タスク = dev-impl 1回で完了する単位
- テストファイル1つ + 実装ファイル1-2つ程度
- 目安: テストケース3-8個 / タスク

### 依存関係の設計
- インターフェース定義タスクを最初に配置する（priority: 1）
- 型定義 → 実装 → 統合 の順で依存関係を設計
- 循環依存を避ける

### 命名規則
- ファイル名: `NNN-kebab-case-description.md`
- 例: `001-auth-interfaces.md`, `002-auth-service.md`, `003-auth-middleware.md`

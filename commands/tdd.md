---
name: tdd
description: TDDセッションを開始。t-wada方式のRed-Green-Refactorサイクル、80%カバレッジ目標、Subagent連携
---

# /tdd コマンド

このコマンドは **tdd-guide** エージェントを呼び出し、TDDセッションを開始します。

## 使用方法

```
/tdd [機能名や説明]
```

## 動作の仕組み

tdd-guide エージェントは以下を行います：

1. **tdd-session** スキルを読み込む
2. 引数に基づいてTDDセッションを開始

## TDDサイクル

```
RED → GREEN → REFACTOR → REPEAT

RED:      失敗するテストを1つ書く
GREEN:    最小限のコードで通す
REFACTOR: テストをパスさせたままコード改善
REPEAT:   次のテストへ
```

## セッションの流れ

### Phase 1: 準備
- 関連Issueの確認/作成
- Explore Subagentで既存コードを調査
- Plan Subagentでテスト計画

### Phase 2: TDDサイクル
- 各テストに対してRed→Green→Refactor
- 1テストずつ進める（t-wada方式）

### Phase 3: 完了
- カバレッジ80%以上を確認
- PR作成を提案

## t-wada方式 原則

1. 失敗するテストなしにコードを書かない
2. テストは1つずつ
3. 最小限のコードで通す（YAGNI）
4. 通ったらすぐリファクタ検討
5. テストコードもリファクタ対象

## 使用例

```
/tdd ユーザー認証機能
/tdd マーケット検索APIの最適化
/tdd バグ修正: ログイン時のセッション問題
```

## 関連スキル

- `~/.claude/skills/tdd-session/SKILL.md` - 詳細なTDDガイド
- `~/.claude/agents/tdd-guide.md` - TDDエージェント

## 関連コマンド

- `/plan` - 実装計画（TDD前に使用可）
- `/test-coverage` - カバレッジ確認
- `/code-review` - 実装後のレビュー

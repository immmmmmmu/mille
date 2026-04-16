# mille — Issue駆動マルチモード開発パイプライン

classmethod/tsumiki をベースにフォークした独立フレームワーク。
4つの開発モード（greenfield / feature / fix / refine）を統合エントリポイント `/mille:kickoff` で提供。

## アーキテクチャ

```
/mille:kickoff → モード自動検出 → 適切なパイプラインを実行
  ├── greenfield: Phase 0-8（新規開発、Issue駆動、並列Wave TDD）
  ├── feature:    dev-context → dev-plan → 実装 → QA → フィードバック
  ├── fix:        investigate → dev-impl → レビュー → マージ
  └── refine:     refine-plan → refine-execute → レビュー
```

## ベンダリング方針

- 本家 tsumiki のコマンド・スキルを取り込み、`mille:` プレフィックスで提供
- 本家の更新は VENDORING.md で追跡し、有用な変更のみチェリーピック
- 独自拡張（kickoff, issue-workflow, feedback-triage, ci-autofix 等）は本プロジェクト固有

## コマンド体系

### 統合エントリポイント
- `/mille:kickoff <名前> [PRDパス] [--mode greenfield|feature|fix|refine] [options]`

### 独自コマンド
- `/mille:ship` — Phase 6 QA パイプライン単体実行
- `/mille:tdd` — TDD セッション開始
- `/mille:feedback` — Phase 8 フィードバック手動トリガー

### ベンダリングコマンド（tsumiki 由来）
- `/mille:kairo-*` — Kairo パイプライン（requirements, design, tasks, loop, implement）
- `/mille:tdd-*` — TDD フェーズ制御（red, green, refactor, verify-complete）
- `/mille:dev-*` — Dev Skills（context, plan, impl, run, verify, debug, screen-spec, webtest）
- `/mille:refine-*` — 小規模変更（plan, execute）
- `/mille:rev-*` — リバースエンジニアリング（tasks, design, requirements, specs）
- `/mille:dcs:*` — Deep Code Synthesis 分析
- `/mille:auto-debug`, `/mille:build-fix`, `/mille:env-fix` 等 — デバッグ系

## 信頼性レベル

ドキュメント内の信頼性アノテーション:
- 🔵 確定（ソースから確認済み）
- 🟡 推測（合理的な推論）
- 🔴 不明（要確認）

## Issue 駆動ルール

- greenfield/feature モードは Epic Issue + TASK Issue ツリーで進捗管理
- TASK Issue は Wave（並列実行グループ）でラベル付け
- Phase 8 フィードバックは自動分類（BUG/SPEC-GAP/ENHANCE/PERF）→ 適切なモードで再突入

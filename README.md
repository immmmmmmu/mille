# mille

Issue 駆動の 4 モード開発パイプライン for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

[classmethod/tsumiki](https://github.com/classmethod/tsumiki) をベースにフォークした独立フレームワーク。  
単一コマンド `/mille:kickoff` でプロジェクト状態を自動検出し、最適なパイプラインを実行する。

## 特徴

- **4 モード統合** — greenfield / feature / fix / refine を単一エントリポイントで切り替え
- **Issue 駆動** — Epic Issue + TASK Issue + Milestone で GitHub 上の進捗を自動追跡
- **Wave 並列実行** — 依存関係グラフからトポロジカルソートし、Agent worktree で並列 TDD 実装
- **自動 QA パイプライン** — コードレビュー、E2E テスト生成、CI 失敗の自動修正（最大 3 回リトライ）
- **フィードバックループ** — BUG / SPEC-GAP / ENHANCE / PERF を自動分類し、適切なモードに再突入

## インストール

Claude Code の Plugin としてインストール:

```bash
claude plugin add immmmmmmu/mille
```

## クイックスタート

```
# 新規プロジェクト（greenfield）
/mille:kickoff ユーザー認証システム docs/prd/auth.md

# 既存コードに機能追加（feature）
/mille:kickoff 検索フィルタ追加 --mode feature

# バグ修正（fix）
/mille:kickoff ログイン失敗修正 --mode fix --issue 42

# 小規模変更（refine）
/mille:kickoff 環境変数リネーム --mode refine
```

## 4 つの開発モード

| モード | 用途 | パイプライン |
|--------|------|-------------|
| **greenfield** | 新規開発 | Phase 0-8 フル（要件定義 → 設計 → タスク分割 → 並列 TDD → QA → フィードバック） |
| **feature** | 機能追加 | コンテキストスキャン → 計画 → 実装 → QA |
| **fix** | バグ修正 | 根本原因調査 → 回帰テスト先行修正 → 軽量レビュー |
| **refine** | 小変更 | 対話的計画 → 一括実行 → 最小レビュー |

### greenfield フロー

```
/mille:kickoff → モード自動検出 → Phase 0-8
  Phase 0    技術スタック        自動
  Phase 1    要件定義           確認ゲート
  Phase 2    技術設計           確認ゲート
  Phase 3    タスク分割 + Issue 起票  確認ゲート
  Phase 4    環境構築           確認ゲート（初回のみ）
  Phase 5    並列 TDD 実装      自動（Wave 並列）
  Phase 6    QA パイプライン     自動（マージ判定のみ確認）
  Phase 7    ポストマージ        完全自動
  Phase 8    フィードバック      自動分類 → 再突入
```

設計（Phase 1-3）は人間が判断、実装以降（Phase 5-8）は自動化。

## コマンド一覧

### エントリポイント

| コマンド | 説明 |
|---------|------|
| `/mille:kickoff` | 4 モード統合エントリポイント |
| `/mille:ship` | Phase 6 QA パイプライン単体実行 |
| `/mille:tdd` | TDD セッション開始 |

### Kairo パイプライン（greenfield 向け）

`kairo-requirements` / `kairo-design` / `kairo-tasks` / `kairo-loop` / `kairo-tasknote`

### Dev Skills（feature / fix 向け）

`dev-context` / `dev-plan` / `dev-impl` / `dev-run` / `dev-verify` / `dev-debug` / `dev-screen-spec` / `dev-webtest-plan` / `dev-webtest` / `dev-init` / `dev-navigate`

### TDD

`tdd-red` / `tdd-green` / `tdd-refactor` / `tdd-verify-complete` / `tdd-requirements` / `tdd-testcases`

### リバースエンジニアリング

`rev-requirements` / `rev-design` / `rev-tasks` / `rev-specs`

### DCS（Deep Code Synthesis 分析）

`dcs:bug-analysis` / `dcs:edgecase-analysis` / `dcs:feature-rubber-duck` / `dcs:impact-analysis` / `dcs:incremental-dev` / `dcs:performance-analysis` / `dcs:sequence-diagram-analysis` / `dcs:state-transition-analysis` / `dcs:code-question` / `dcs:test-performance-analysis`

### デバッグ・修正

`auto-debug` / `build-fix` / `env-fix` / `flaky-fix` / `timeout-fix`

全コマンドは `/mille:help` で詳細を確認できる。

## tsumiki との違い

| 観点 | tsumiki v1.3.0 | mille v1.0 |
|------|---------------|------------|
| エントリポイント | コマンドごとに個別実行 | `/mille:kickoff` で統合 |
| 並列タスク実行 | なし（全タスク直列） | Wave 並列（Agent worktree） |
| GitHub Issues 連携 | なし | Epic / TASK / Milestone / Feedback Issue ツリー |
| フィードバックループ | なし | 自動分類 → 適切なモードで再突入 |
| CI 自動修正 | なし | エラー分類 → 修正 → 最大 3 回リトライ |
| QA パイプライン | テスト・lint・ファイルサイズ | レビュー + E2E + CI 修正 + マージ判定 |
| ポストマージ自動化 | なし | ドキュメント・リリースノート・メトリクス |
| リスク制御 | なし | WTF-likelihood + Scope Drift Detection |

詳細は [docs/comparison-with-tsumiki.md](docs/comparison-with-tsumiki.md) を参照。

## ドキュメント

- [リファレンスガイド](docs/reference.md) — 全コマンド・スキル・オプションの詳細
- [設計書](docs/design.md) — Phase 0-8 の内部設計
- [tsumiki 比較](docs/comparison-with-tsumiki.md) — 機能マトリクス
- [ベンダリング記録](VENDORING.md) — 本家からの取り込み履歴

## ライセンス

MIT License. 詳細は [LICENSE](LICENSE) を参照。

tsumiki の原作: [classmethod/tsumiki](https://github.com/classmethod/tsumiki)（MIT License）

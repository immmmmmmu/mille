# mille リファレンスガイド

## 概要

mille は Issue 駆動の 4 モード開発パイプライン。classmethod/tsumiki をベースにフォークした独立フレームワーク。

統合エントリポイント `/mille:kickoff` がプロジェクト状態を自動検出し、最適なパイプラインを実行する。

## 4つの開発モード

| モード | 用途 | パイプライン | 主な利用場面 |
|--------|------|-------------|-------------|
| **greenfield** | 新規開発 | Phase 0-8 フル | 新プロジェクト、新機能システム |
| **feature** | 機能追加 | F0-F6 | 既存コードへの機能追加 |
| **fix** | バグ修正 | X0-X6 | バグ調査・修正 |
| **refine** | 小変更 | R0-R5 | 設定変更、ドキュメント修正 |

---

## クイックスタート

```bash
# 新規プロジェクト（greenfield）
/mille:kickoff ユーザー認証システム docs/prd/auth.md

# 既存コードに機能追加（feature）
/mille:kickoff 検索フィルタ追加 --mode feature

# バグ修正（fix）
/mille:kickoff ログイン失敗修正 --mode fix --issue 42

# 小規模変更（refine）
/mille:kickoff 環境変数リネーム --mode refine
```

---

## greenfield モード（Phase 0-8）

新規開発向けフルパイプライン。設計は人間が判断、実装以降は自動化。

```
Phase -1   モード選択        → 自動検出 + 確認
Phase 0    技術スタック       → 自動（CLAUDE.md があれば）
Phase 0.5  リサーチ          → 自動（不足情報を並列調査）
Phase 1    要件定義          → ✋ 確認ゲート（EARS 記法、信頼性レベル付き）
Phase 1.5  Constitution      → 自動（--with-constitution 時のみ）
Phase 2    技術設計          → ✋ 確認ゲート（Shadow Path 検証、Error & Rescue Registry）
Phase 3    タスク分割        → ✋ 確認ゲート + GitHub Issue 自動起票
Phase 4    環境構築          → ✋ 確認ゲート（初回のみ）
Phase 5    並列 TDD 実装     → 自動（Wave 単位で Agent worktree 並列）
Phase 6    品質保証          → 自動（マージ判定のみ ✋、--auto-merge で省略可）
Phase 7    ポストマージ       → 完全自動（ドキュメント、リリースノート、メトリクス）
Phase 8    フィードバック     → 自動分類 → 適切なモードで再突入
```

### greenfield オプション

| オプション | 説明 |
|-----------|------|
| `--skip-impl` | Phase 5-7 をスキップ（設計までで停止） |
| `--skip-bootstrap` | Phase 4 をスキップ |
| `--skip-research` | Phase 0.5 をスキップ |
| `--with-constitution` | Phase 1.5 でプロジェクト原則を生成 |
| `--light` | 各フェーズで軽量規模を自動選択 |
| `--auto-merge` | Phase 6 のマージ確認ゲートを省略 |
| `--parallel N` | Phase 5 の並列数（デフォルト: 3） |
| `--sequential` | Phase 5 を直列実行（kairo-loop 互換） |

### Issue ツリー構造

```
📋 Epic Issue: "{要件名}"
│  labels: epic, mille
│
├── 🔖 Milestone: "仕様策定"
│   ├── #{N} 要件定義        labels: spec, phase-1
│   ├── #{N} 技術設計        labels: design, phase-2
│   └── #{N} タスク分割      labels: planning, phase-3
│
├── 🔖 Milestone: "実装"
│   ├── #{N} TASK-0001       labels: task, wave-1
│   ├── #{N} TASK-0002       labels: task, wave-1
│   └── #{N} TASK-0003       labels: task, wave-2, blocked
│
├── 🔖 Milestone: "品質保証"
│   └── #{N} マージ判定      labels: merge, phase-6
│
└── 🔖 Milestone: "フィードバック"
    └── #{N} [BUG] ...       labels: bug, feedback
```

### 成果物ディレクトリ

```
docs/
├── spec/{要件名}/
│   ├── requirements.md      # EARS 記法の要件定義書
│   ├── interview-record.md  # ヒアリング記録
│   ├── note.md              # コンテキストノート
│   ├── prep.md              # 事前準備タスク（v1.3.0）
│   ├── user-stories.md      # ユーザーストーリー
│   └── acceptance-criteria.md
├── design/{要件名}/
│   ├── architecture.md      # アーキテクチャ設計
│   ├── dataflow.md          # データフロー図
│   ├── interfaces.ts        # TypeScript 型定義
│   ├── database-schema.sql  # DB スキーマ
│   └── api-endpoints.md     # API 仕様
└── tasks/{要件名}/
    ├── overview.md           # タスク概要・依存関係
    ├── TASK-0001.md
    └── TASK-0002.md
```

---

## feature モード（Phase F0-F6）

既存コードベースへの機能追加。dev-* で計画し、実績ある Phase 5-8 インフラで実装。

```
Phase F0   dev-context       → プロジェクト構造スキャン
Phase F1   dev-plan          → 機能計画・タスク分割
Phase F1.5 dev-screen-spec   → 画面仕様（Web の場合、オプション）
Phase F2   Issue ブリッジ     → dev-plan タスク → GitHub Issue 起票
Phase F3   実装              → > 5件: Phase 5 並列 TDD / ≤ 5件: dev-run
Phase F4   QA パイプライン    → Phase 6 再利用
Phase F5   ポストマージ       → Phase 7 再利用
Phase F6   フィードバック     → Phase 8 再利用
```

### feature 成果物

```
docs/dev/
├── context.md                    # プロジェクトコンテキスト
└── plans/{名前}/
    ├── plan.md                   # 計画概要
    ├── tasks/
    │   ├── 001-{タスク名}.md
    │   └── 002-{タスク名}.md
    ├── issue-map.md              # GitHub Issue マッピング
    └── reports/                  # dev-verify 検証結果
```

---

## fix モード（Phase X0-X6）

バグ修正に特化した軽量パイプライン。

```
Phase X0   dev-context       → コンテキスト取得（なければ生成）
Phase X1   調査              → /investigate で根本原因分析（3-strike ルール）
Phase X2   修正計画          → 影響ファイル数で分岐
Phase X3   実装              → dev-impl（TDD: 回帰テスト先行）
Phase X4   検証              → dev-verify（テスト全量、回帰なし確認）
Phase X5   QA（軽量版）      → コードレビュー + CI チェック
Phase X6   マージ            → PR マージ + Issue クローズ
```

---

## refine モード（Phase R0-R5）

設定変更・ドキュメント修正・小規模コード変更。

```
Phase R0   dev-context       → コンテキスト取得（なければ生成）
Phase R1   refine-plan       → 対話的に変更箇所・影響・計画を特定
Phase R2   確認ゲート         → 計画を表示、承認を取得
Phase R3   refine-execute    → テスト修正・実装・ビルド・テストを一括実行
Phase R4   QA（最小版）      → Light レビュー + CI のみ
Phase R5   マージ            → PR マージ or 直接コミット
```

---

## feature / fix / refine 共通オプション

| オプション | 説明 |
|-----------|------|
| `--issue <番号>` | 既存 GitHub Issue にリンク |
| `--plan-only` | 計画段階で停止 |

---

## フィードバックループ（Phase 8）

プロダクト稼働中のフィードバックを自動分類し、適切なモードで再突入:

| 分類 | 定義 | 再突入モード |
|------|------|------------|
| **BUG** | 既存仕様の実装が動作しない | fix モード |
| **SPEC-GAP** | 仕様に記載がないケースが発生 | feature モード |
| **ENHANCE** | 新しい機能要求 | feature モード（大規模は greenfield） |
| **PERF** | パフォーマンス劣化 | fix モード |

深刻度: CRITICAL（即時自動対応）/ HIGH / MEDIUM / LOW

---

## コマンド一覧

### エントリポイント

| コマンド | 説明 |
|---------|------|
| `/mille:kickoff` | 4 モード統合エントリポイント（モード自動検出） |

### Kairo パイプライン（greenfield Phase 1-5）

| コマンド | 説明 |
|---------|------|
| `/mille:kairo-requirements` | EARS 記法の要件定義書を作成 |
| `/mille:kairo-design` | 要件定義に基づく技術設計文書を生成 |
| `/mille:kairo-tasks` | 設計に基づく 1 日粒度のタスク分割 |
| `/mille:kairo-loop` | 指定 TASK 範囲の kairo 実装を自動実行 |
| `/mille:kairo-tasknote` | Kairo 開発コンテキスト情報をノートにまとめる |

### Dev Skills（feature / fix モード）

| コマンド | 説明 |
|---------|------|
| `/mille:dev-context` | プロジェクト構造を 4 方向スキャンし context.md を生成 |
| `/mille:dev-plan` | 要件分析 → インターフェース設計 → テスト可能なタスク分割 |
| `/mille:dev-impl` | TDD ベースの単一タスク実装 |
| `/mille:dev-run` | タスク範囲の自動実装ループ（impl → verify → debug） |
| `/mille:dev-verify` | ビルド・テスト・lint・ファイルサイズの一括検証 |
| `/mille:dev-debug` | 7 カテゴリ別エラー自動診断（compile/test/runtime/config/lint/deps/webtest） |
| `/mille:dev-screen-spec` | ソースコードから画面仕様書を自動生成 |
| `/mille:dev-webtest-plan` | Playwright E2E テスト計画を生成 |
| `/mille:dev-webtest` | Playwright E2E テストを実行 |
| `/mille:dev-init` | 対話的な技術スタック選定 + プロジェクトスキャフォールド |
| `/mille:dev-navigate` | やりたいことをヒアリングし最適なスキルをナビゲーション |

### Refine（refine モード）

| コマンド | 説明 |
|---------|------|
| `/mille:refine-plan` | 既存コードの小規模修正計画を対話的に作成 |
| `/mille:refine-execute` | refine-plan の実行（テスト・修正・確認を一括） |

### TDD（テスト駆動開発）

| コマンド | 説明 |
|---------|------|
| `/mille:tdd` | TDD セッション開始（t-wada 方式 Red-Green-Refactor） |
| `/mille:tdd-red` | Red フェーズ — 失敗するテストケースを作成 |
| `/mille:tdd-green` | Green フェーズ — テストを通す最小限の実装 |
| `/mille:tdd-refactor` | Refactor フェーズ — コード品質改善 |
| `/mille:tdd-verify-complete` | 全テスト成功 + カバレッジを確認 |
| `/mille:tdd-requirements` | TDD 用の要件整理 |
| `/mille:tdd-testcases` | テストケースの洗い出し |
| `/mille:tdd-tasknote` | TDD コンテキスト情報をノートにまとめる |
| `/mille:tdd-todo` | タスクファイルから TODO リストを作成 |

### リバースエンジニアリング（rev-*）

| コマンド | 説明 |
|---------|------|
| `/mille:rev-requirements` | 既存コードから EARS 形式の要件定義書を逆生成 |
| `/mille:rev-design` | 既存コードから技術設計文書を逆生成 |
| `/mille:rev-tasks` | 既存コードからタスク一覧を逆生成 |
| `/mille:rev-specs` | 既存コードからテストケース・仕様書を逆生成 |

### DCS（Deep Code Synthesis 分析）

| コマンド | 説明 |
|---------|------|
| `/mille:dcs:bug-analysis` | バグの根本原因を詳細分析 |
| `/mille:dcs:edgecase-analysis` | 9 カテゴリのエッジケース・エラー状態を洗い出し |
| `/mille:dcs:feature-rubber-duck` | アイデアを整理して実現可能な PRD を作成 |
| `/mille:dcs:impact-analysis` | 変更の影響範囲を全レイヤーで分析 |
| `/mille:dcs:incremental-dev` | 増分開発の計画を立案（複数アプローチ比較） |
| `/mille:dcs:performance-analysis` | 性能問題の原因を調査しボトルネックを特定 |
| `/mille:dcs:sequence-diagram-analysis` | Mermaid 形式のシーケンス図を作成 |
| `/mille:dcs:state-transition-analysis` | データの状態遷移フローを包括分析 |
| `/mille:dcs:code-question` | ソースコードに関する質問に回答（最大 3 回反復） |
| `/mille:dcs:test-performance-analysis` | テスト実行速度を分析しリファクタリングを提案 |

### QA・運用

| コマンド | 説明 |
|---------|------|
| `/mille:ship` | Phase 6 QA パイプライン単体実行（test → review → CI 修正 → マージ判定） |
| `/mille:auto-debug` | テストエラーの自動デバッグ（原因調査 → 修正 → 再実行） |
| `/mille:build-fix` | ビルドエラーの自動修正 |
| `/mille:env-fix` | 環境依存問題の自動修正 |
| `/mille:flaky-fix` | 不安定テストの原因分析と安定化 |
| `/mille:timeout-fix` | テストタイムアウトの分析と高速化 |

### セットアップ・ユーティリティ

| コマンド | 説明 |
|---------|------|
| `/mille:init-tech-stack` | 技術スタックの初期選定 |
| `/mille:direct-setup` | DIRECT タスクの設定作業を実行 |
| `/mille:direct-verify` | DIRECT タスクの動作確認テスト |
| `/mille:orchestrate` | 複雑な依頼を自動分析し適切なエージェントチームで実行 |
| `/mille:help` | コマンド一覧・詳細ヘルプ・困りごと検索 |

---

## スキル一覧

| スキル | 説明 |
|--------|------|
| `mille:dev-context` | プロジェクト構造の自動分析 → context.md 生成 |
| `mille:dev-plan` | 要件分析 → タスク分割（full-spec / lightweight） |
| `mille:dev-impl` | TDD ベースの単一タスク実装（normal / quick モード） |
| `mille:dev-run` | タスク範囲の自動実装ループ（最大 30 タスク） |
| `mille:dev-verify` | テスト・ビルド・lint・ファイルサイズの一括検証 |
| `mille:dev-debug` | 7 カテゴリ別エラー自動診断 |
| `mille:dev-screen-spec` | ソースコード / 受け入れ基準から画面仕様を自動生成 |
| `mille:dev-webtest-plan` | Playwright E2E テスト計画の生成 |
| `mille:dev-webtest` | Playwright E2E テストの実行 |
| `mille:dev-init` | 対話的プロジェクト初期化 + スキャフォールド |
| `mille:dev-navigate` | 最適なスキルのナビゲーション |
| `mille:kairo-implement` | TDD コマンドを使ったタスク実装（`--hil` で人間介入） |
| `mille:issue-workflow` | GitHub Issue ツリー管理（Epic/TASK/フィードバック Issue） |
| `mille:feedback-triage` | フィードバック自動分類 → 適切なモードで再突入 |
| `mille:ci-autofix` | CI 失敗 → 自動修正ループ（最大 3 回、失敗時は Issue 報告） |
| `mille:tdd-session` | t-wada 方式 TDD セッション（80% カバレッジ目標） |

---

## 信頼性レベル

mille のドキュメント全体で使用される信頼性アノテーション:

| レベル | 意味 | 扱い |
|--------|------|------|
| 🔵 確定 | ソースから確認済み | そのまま採用 |
| 🟡 推測 | 合理的な推論 | 確認を推奨 |
| 🔴 不明 | 要確認 | 実装前に解決が必要 |

---

## 途中中止・再開

| 中止ポイント | 再開コマンド |
|------------|------------|
| greenfield Phase 0 完了後 | `/mille:kairo-requirements {要件名}` |
| greenfield Phase 1 完了後 | `/mille:kairo-design {要件名}` |
| greenfield Phase 2 完了後 | `/mille:kairo-tasks {要件名} --task` |
| greenfield Phase 3 完了後 | `/project-bootstrap` |
| greenfield Phase 4 完了後 | `/mille:kairo-loop {要件名} TASK-0001 TASK-{N}` |
| greenfield Phase 5 完了後 | `/mille:ship` |
| greenfield Phase 7 完了後 | `/mille:feedback {内容}` |
| feature Phase F1 完了後 | `/mille:dev-run {名前} 001 {最終}` |
| fix Phase X1 完了後 | `/mille:dev-impl "{修正内容}"` |

---

## ベンダリング情報

| 項目 | 内容 |
|------|------|
| ベース | classmethod/tsumiki v1.2.0 + v1.3.0 |
| ライセンス | MIT |
| 初回取り込み | 2026-04-05 |
| 独自拡張 | kickoff（4 モード）, ship, issue-workflow, feedback-triage, ci-autofix, tdd-session |
| 更新方針 | 本家 diff を確認し有用な変更をチェリーピック（VENDORING.md に記録） |

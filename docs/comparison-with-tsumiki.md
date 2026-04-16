# mille vs tsumiki v1.3.0 — 機能比較

## 概要

mille は classmethod/tsumiki v1.3.0 をフォークした独立フレームワーク。
本家の全コマンド・スキルをベンダリングした上で、Issue 駆動・並列実行・4 モード統合・フィードバックループなどの独自機能を追加している。

---

## 比較サマリー

| 観点 | tsumiki v1.3.0 | mille v1.0 |
|------|---------------|------------|
| エントリポイント | コマンドごとに個別実行 | `/mille:kickoff` で統合（モード自動検出） |
| 開発モード | Kairo / Dev Skills / DCS の 3 系統 | greenfield / feature / fix / refine の 4 モード |
| 並列タスク実行 | なし（全タスク直列） | Wave 並列（Agent worktree、デフォルト 3 並列） |
| GitHub Issues 連携 | なし（ファイル + Claude Code Tasks のみ） | Epic/TASK/Milestone/Feedback Issue ツリー |
| フィードバックループ | なし | Phase 8: 自動分類 → 適切なモードで再突入 |
| CI 自動修正 | なし | ci-autofix: エラー分類 → 修正 → 最大 3 回リトライ |
| QA パイプライン | dev-verify（テスト・lint・ファイルサイズ） | Phase 6: レビュー + E2E + CI 修正 + マージ判定 |
| ポストマージ自動化 | なし | Phase 7: ドキュメント・リリースノート・メトリクス |
| リスク制御 | なし | WTF-likelihood + Scope Drift Detection |
| 確認ゲート設計 | --hil フラグ + AskUserQuestion | Phase ごとに明示配置（自動/手動の使い分け） |

---

## 1. ワークフロー設計

### tsumiki v1.3.0

3 つの独立したワークフローを提供。ユーザーが状況に応じてコマンドを選ぶ。

```
Kairo ワークフロー（新規開発向け）:
  init-tech-stack → kairo-requirements → kairo-design → kairo-tasks → kairo-implement/loop

Dev Skills ワークフロー（既存コード向け）:
  dev-init/dev-context → dev-plan → dev-impl/dev-run → dev-verify → dev-debug

DCS ワークフロー（分析専用）:
  dcs:* → .dcs/ にレポート出力
```

**問題点**: どのワークフローを使うべきか、ユーザーが判断する必要がある。`dev-navigate` で補助はあるが、ワークフロー間の連携は手動。

### mille v1.0

Phase -1 でプロジェクト状態を自動検出し、4 モードのうち最適なものを推奨。

```
/mille:kickoff → Phase -1（モード自動検出）
  ├── greenfield → Phase 0-8（Kairo ベース + 並列 + QA + フィードバック）
  ├── feature   → F0-F6（Dev Skills + Phase 5-8 インフラ再利用）
  ├── fix       → X0-X6（investigate + dev-impl + Phase 6 軽量）
  └── refine    → R0-R5（refine-plan/execute + Phase 6 最小）
```

**利点**: 単一コマンドで開始でき、モード間の共通インフラ（Phase 6-8）が再利用される。

---

## 2. 並列タスク実行

### tsumiki v1.3.0

**並列実行の仕組みがない。** 全タスクは依存関係順に直列実行される。

- `kairo-loop`: TASK-0001 → TASK-0002 → ... を順番に実行
- `dev-run`: タスク範囲を順番に impl → verify → debug
- 唯一の並列: `dev-webtest --parallel N`（Playwright テストのみ）

### mille v1.0

**Wave 並列実行**を Phase 5 で提供。

```
Wave 算出: TASK ファイルの blocked_by 依存関係グラフからトポロジカルソート
  TASK-0001 (依存なし)     → Wave 1
  TASK-0002 (依存なし)     → Wave 1
  TASK-0003 (← 0001)      → Wave 2
  TASK-0004 (← 0001,0002) → Wave 2

実行:
  Wave 1: Agent worktree 1 → TASK-0001 ∥ Agent worktree 2 → TASK-0002
  Wave 2: Agent worktree 1 → TASK-0003 ∥ Agent worktree 2 → TASK-0004
```

- デフォルト 3 並列（`--parallel N` で変更可能）
- `--sequential` で直列実行も可能（kairo-loop 互換）
- feature モードでもタスク > 5 件なら Wave 並列を使用

---

## 3. GitHub Issues 連携

### tsumiki v1.3.0

**GitHub Issues 連携は一切ない。** 進捗管理は:
- ファイルの `status` フィールド（pending/in_progress/done）
- Claude Code の TaskList/TaskUpdate API

### mille v1.0

**Issue ツリーで全フェーズを追跡。**

```
📋 Epic Issue
├── 🔖 Milestone: "仕様策定"
│   ├── #{N} 要件定義        ✅ 自動クローズ
│   └── #{N} 技術設計        ✅ 自動クローズ
├── 🔖 Milestone: "実装"
│   ├── #{N} TASK-0001       labels: wave-1
│   └── #{N} TASK-0002       labels: wave-2, blocked
├── 🔖 Milestone: "品質保証"
│   └── #{N} マージ判定
└── 🔖 Milestone: "フィードバック"
    ├── #{N} [BUG] ...       labels: bug, critical
    └── #{N} [SPEC-GAP] ...  labels: spec-gap
```

- Phase 3 完了時に Epic + TASK Issue を一括起票
- PR に `Closes #{Issue番号}` を付与
- Phase 7 で Epic Issue の完了判定 + クローズ
- Phase 8 でフィードバック Issue を自動起票

---

## 4. 品質保証パイプライン

### tsumiki v1.3.0

`dev-verify` による検証:
1. タスクステータス確認
2. テスト実行
3. カバレッジ確認（context.md の閾値）
4. ビルド確認
5. lint 実行
6. 500 行ファイルサイズルール
7. 信頼性シグナル集約（🔵/🟡/🔴）

**コードレビュー、E2E テスト生成、CI 修正、マージ判定はない。**

### mille v1.0

**Phase 6: 多層 QA パイプライン**

```
6-1. 自動レビュー（並列）
     - code-reviewer: 差分サイズ連動深度（Light/Standard/Deep）
     - security-reviewer: Confidence Gate 8/10
     - 結果分類: AUTO-FIX / CRITICAL / その他

6-2. E2E テスト
     - requirements.md の Critical Path を自動検出
     - e2e-runner で Playwright テスト生成・実行

6-3. CI 修正ループ（最大 3 回）
     - エラー分類: codegen → lint → type → test
     - 分類に応じた自動修正 + push + CI 再実行
     - 3 回失敗 → needs-human ラベル + Issue コメント

6-4. マージ判定
     - 全 PR を ✅/⚠️/❌ で分類
     - --auto-merge で条件充足 PR を自動マージ
```

---

## 5. フィードバックループ

### tsumiki v1.3.0

**フィードバックの仕組みがない。** 実装完了で終了。

- `dev-run` の verify → fix サイクル（最大 2 回）はあるが、プロダクション運用後のフィードバック対応ではない
- バグ報告 → 修正のフローは手動で `refine-plan` → `refine-execute` を使う

### mille v1.0

**Phase 8: 自動フィードバックループ**

```
入力ソース → 自動分類 → 適切なモードで再突入

分類ロジック:
  Step 1: requirements.md の FR/NFR と照合
  Step 2: BUG vs PERF（エラー発生 vs レスポンス劣化）
  Step 3: SPEC-GAP vs ENHANCE（同一ドメイン vs 新規ドメイン）
  Step 4: 深刻度（CRITICAL / HIGH / MEDIUM / LOW）

再突入:
  BUG      → fix モード（CRITICAL は確認ゲートなし）
  SPEC-GAP → feature モード（1 日バッチ後にまとめて再突入）
  ENHANCE  → feature モード（大規模は新規 greenfield）
  PERF     → fix モード（プロファイリング → 最適化）
```

---

## 6. リスク制御

### tsumiki v1.3.0

**リスク制御の仕組みがない。**

- 信頼性レベル（🔵/🟡/🔴）でドキュメントの確実性は示すが、実装中のリスク監視はない
- `auto-debug` は 3 回リトライ後にユーザーに委ねる

### mille v1.0

**2 つのリスク制御メカニズム**

**WTF-likelihood（Wave 単位の暴走防止）:**
```
リスクスコア計算:
  revert 発生: +15%
  共通モジュール変更: +5%/件
  修正数 N: N × 0.5%

停止条件:
  20% 超 → 一時停止してユーザーに相談
  30 修正/セッション → ハードキャップ（強制停止）
```

**Scope Drift Detection（要件外変更の検出）:**
```
全 Wave 完了後:
  1. 各 TASK ブランチの git diff --name-only を集約
  2. タスク一覧と照合
  3. タスクに関連しないファイル変更を [SCOPE DRIFT] としてフラグ
  ※ 情報提供のみ（ブロックしない）
```

---

## 7. 確認ゲート設計

### tsumiki v1.3.0

- `--hil` フラグで `kairo-implement` に人間介入ポイントを追加
- `dev-run` / `dev-verify` で AskUserQuestion による任意のエスカレーション
- **パイプライン全体の確認ゲート設計はない**

### mille v1.0

**Phase ごとに明示的な確認ゲート配置:**

```
Phase 0   技術スタック     → 自動
Phase 0.5 リサーチ         → 自動
Phase 1   要件定義         → ✋ 確認ゲート（仕様判断）
Phase 1.5 Constitution     → 自動
Phase 2   技術設計         → ✋ 確認ゲート（設計判断）
Phase 3   タスク分割       → ✋ 確認ゲート + Issue 自動起票
Phase 4   環境構築         → ✋ 確認ゲート（初回のみ）
Phase 5   並列 TDD 実装    → 自動（Wave 単位で並列）
Phase 6   品質保証         → 自動（マージ判定のみ ✋、--auto-merge で省略可）
Phase 7   ポストマージ     → 完全自動
Phase 8   フィードバック   → 分類による
```

**設計思想**: 設計（Phase 1-3）は人間が判断、実装以降（Phase 5-8）は自動化。

---

## 8. ポストマージ自動化

### tsumiki v1.3.0

**ポストマージ自動化はない。** 実装 + 検証で終了。

### mille v1.0

**Phase 7: 5 つのポストマージ自動化ステップ**

| ステップ | 内容 |
|---------|------|
| 7-1 | ドキュメント自動更新（CLAUDE.md, README.md, CHANGELOG.md） |
| 7-2 | Epic Issue 更新 + クローズ判定 |
| 7-3 | リリースノート下書き（`gh release create --draft`） |
| 7-4 | メトリクス記録（コミット数、テストカバレッジ、Wave 並列効率） |
| 7-5 | canary モニタリング起動（デプロイ先がある場合） |

---

## 9. 設計品質レビュー

### tsumiki v1.3.0

- `kairo-design` が設計文書を生成するが、**自動レビューは行わない**
- 信頼性レベル（🔵/🟡/🔴）のアノテーションのみ

### mille v1.0

**Phase 2 の多角的レビュー:**

| 検証項目 | 内容 |
|---------|------|
| FR トレーサビリティ | 各 FR に対応する設計要素があるか |
| NFR 反映度 | 非機能要件が設計に反映されているか |
| Shadow Path 検証 | Happy/Nil/Empty/Error の 4 パスがカバーされているか |
| Error & Rescue Registry | エラーパターンをテーブル化、サイレント失敗を検出 |
| Dual-Voice レビュー | architect + /codex による独立レビュー（利用可能時） |

**Phase 3 の累積整合性チェック:**

| 検証項目 | 内容 |
|---------|------|
| 設計→タスク網羅性 | 全コンポーネントにタスクがあるか |
| 要件→設計→タスク トレーサビリティ | 途中で消失した要件がないか |
| 🔴 依存タスク識別 | 不明項目に依存するタスクのリスク |
| 粒度検証 | 1 日単位として適切か（過大/過小の検出） |

---

## 10. コマンド・スキル数の比較

### tsumiki v1.3.0

| カテゴリ | コマンド数 | スキル数 |
|---------|----------|---------|
| Kairo パイプライン | 6 | 1 |
| Dev Skills | 11 | 11 |
| TDD | 8 | 0 |
| Rev（リバースエンジニアリング） | 4 | 0 |
| DCS（分析） | 10 | 0 |
| デバッグ・修正 | 5 | 0 |
| Refine | 2 | 0 |
| ユーティリティ | 4 | 0 |
| **合計** | **50** | **12** |

### mille v1.0

| カテゴリ | コマンド数 | スキル数 |
|---------|----------|---------|
| 統合エントリ（kickoff） | 1 | 0 |
| Kairo パイプライン | 5 | 1 |
| Dev Skills | 11 | 11 |
| TDD | 9 | 1 |
| Rev（リバースエンジニアリング） | 4 | 0 |
| DCS（分析） | 10 | 0 |
| デバッグ・修正 | 5 | 0 |
| Refine | 2 | 0 |
| QA・運用（ship, feedback） | 2 | 0 |
| Issue 管理 | 0 | 1 |
| CI 自動修正 | 0 | 1 |
| フィードバック分類 | 0 | 1 |
| ユーティリティ | 5 | 0 |
| **合計** | **54** | **16** |

mille の追加: +4 コマンド（kickoff, ship, tdd, feedback）、+4 スキル（issue-workflow, feedback-triage, ci-autofix, tdd-session）

---

## 11. 機能マトリクス（一覧比較）

| 機能 | tsumiki v1.3.0 | mille v1.0 |
|------|:-:|:-:|
| **ワークフロー** | | |
| 新規開発パイプライン（Kairo） | ✅ | ✅ greenfield モード |
| 既存コード向けパイプライン（Dev Skills） | ✅ | ✅ feature モード |
| バグ修正専用パイプライン | ❌ | ✅ fix モード |
| 小変更パイプライン（Refine） | ✅ | ✅ refine モード |
| 統合エントリポイント | ❌ | ✅ /mille:kickoff |
| モード自動検出 | ❌ | ✅ Phase -1 |
| **実装** | | |
| TDD（Red-Green-Refactor） | ✅ | ✅ |
| DIRECT モード（非 TDD） | ✅ | ✅ |
| 並列タスク実行 | ❌ | ✅ Wave 並列 |
| Agent worktree 活用 | ❌ | ✅ |
| **品質保証** | | |
| テスト実行 | ✅ dev-verify | ✅ Phase 6 |
| カバレッジ確認 | ✅ dev-verify | ✅ Phase 6 |
| lint 実行 | ✅ dev-verify | ✅ Phase 6 |
| ファイルサイズルール | ✅ 500 行 | ❌ 独自ルールなし |
| 自動コードレビュー | ❌ | ✅ code-reviewer + security-reviewer |
| レビュー深度の自動調整 | ❌ | ✅ Light/Standard/Deep |
| E2E テスト自動生成 | ❌ | ✅ Critical Path 検出 + Playwright |
| CI 失敗自動修正 | ❌ | ✅ ci-autofix（最大 3 回） |
| マージ判定 | ❌ | ✅ Phase 6-4 |
| **設計品質** | | |
| 信頼性レベル（🔵/🟡/🔴） | ✅ | ✅ |
| FR トレーサビリティ検証 | ❌ | ✅ Phase 1-3 レビュー |
| Shadow Path 検証 | ❌ | ✅ Happy/Nil/Empty/Error |
| Error & Rescue Registry | ❌ | ✅ |
| Dual-Voice レビュー | ❌ | ✅ architect + /codex |
| 累積整合性チェック | ❌ | ✅ Phase 3 |
| **進捗管理** | | |
| ファイルベース進捗 | ✅ status フィールド | ✅ |
| Claude Code Tasks 連携 | ✅ kairo-implement, dev-run | ✅ |
| GitHub Issues 連携 | ❌ | ✅ Epic/TASK/Milestone |
| Wave ラベル | ❌ | ✅ wave-1, wave-2, ... |
| **フィードバック** | | |
| --hil（人間介入フラグ） | ✅ | ✅ |
| AskUserQuestion エスカレーション | ✅ | ✅ |
| 確認ゲートの明示配置 | ❌ | ✅ Phase ごと |
| フィードバック自動分類 | ❌ | ✅ BUG/SPEC-GAP/ENHANCE/PERF |
| フィードバック再突入 | ❌ | ✅ Phase 8 → 適切なモード |
| バッチ処理（SPEC-GAP/PERF） | ❌ | ✅ 1 日バッファ |
| **リスク制御** | | |
| WTF-likelihood | ❌ | ✅ 20% 停止、30 修正キャップ |
| Scope Drift Detection | ❌ | ✅ |
| **ポストマージ** | | |
| ドキュメント自動更新 | ❌ | ✅ Phase 7-1 |
| リリースノート自動生成 | ❌ | ✅ Phase 7-3 |
| メトリクス記録 | ❌ | ✅ Phase 7-4 |
| canary モニタリング | ❌ | ✅ Phase 7-5 |
| **分析** | | |
| DCS（Deep Code Synthesis） | ✅ 10 コマンド | ✅ 10 コマンド |
| リバースエンジニアリング | ✅ 4 コマンド | ✅ 4 コマンド |
| **Web テスト** | | |
| 画面仕様自動生成 | ✅ dev-screen-spec | ✅ |
| E2E テスト計画 | ✅ dev-webtest-plan | ✅ |
| Playwright テスト実行 | ✅ dev-webtest | ✅ |
| **カスタマイズ** | | |
| docs/rule/ カスタムルール | ✅ | ✅（ベンダリング済み） |
| モデル選択フラグ | ✅ --model, --think-model | ✅（ベンダリング済み） |

---

## 12. mille にない tsumiki の機能

| 機能 | 説明 | mille での対応 |
|------|------|---------------|
| 500 行ファイルサイズルール | dev-verify が 500 行超ファイルをフラグ | Phase 6 にはない（dev-verify 自体は使用可能） |
| dev-verify の 8 チェック | テスト・カバレッジ・ビルド・lint・サイズ・信頼性シグナルの統合検証 | Phase 6 で分散実行（統合レポートは dev-verify を直接呼べば利用可能） |
| Docker サンドボックス | claude_docker/ によるサンドボックス実行 | 未取り込み（プロダクション非推奨のため） |
| docs/rule/ 自動読み込み | コマンド実行前にプロジェクトルールを自動適用 | ベンダリング済みコマンドに含まれる |
| dev-run の verify → fix 自動サイクル（2 回） | 実装後の自動修復 | dev-run はベンダリング済み（そのまま利用可能） |

---

## 13. 選定ガイド

### tsumiki v1.3.0 が向いているケース

- 小規模プロジェクトで Issue 管理が不要
- 直列実行で十分（タスク数が少ない）
- 本家のアップデートを即座に取り込みたい
- フレームワークのカスタマイズを最小限にしたい

### mille v1.0 が向いているケース

- Issue 駆動で進捗を GitHub 上で追跡したい
- 並列実行で実装速度を上げたい
- 実装後の QA（レビュー・E2E・CI 修正）を自動化したい
- プロダクション運用後のフィードバックループが必要
- 新規開発・機能追加・バグ修正・小変更を統一エントリで管理したい
- 設計品質を多角的にレビューしたい（Shadow Path、トレーサビリティ）

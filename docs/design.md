# mille:kickoff v2 設計書

## 目的

mille:kickoff を Issue 駆動 + 自動化強化で進化させる。
設計判断は人間、実装〜マージ〜運用フィードバックは自動化。

## 全体フロー

```
┌─────────────────────────────────────────────────────────────┐
│  設計フェーズ（人間が判断）                                    │
│  Phase 0-4: 技術スタック → 要件 → 設計 → タスク → 環境構築   │
│  確認ゲート: Phase 1, 2, 3, 4                                │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│  実装フェーズ（自動化）                                       │
│  Phase 5: 並列TDD実装（Wave単位）                            │
│  Phase 6: 自動レビュー・E2E・マージ                           │
│  Phase 7: ポストマージ（ドキュメント・メトリクス・監視）        │
│  確認ゲート: Phase 6-4 マージ判定のみ（--auto-merge で省略可） │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│  フィードバックフェーズ（検知は自動、判断は分類による）         │
│  Phase 8: 運用フィードバック収集・分類・自動起票               │
│  → 分類結果に応じて Phase 1 / 3 / 5 に再突入                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Issue 駆動アーキテクチャ

### Issue ツリー構造

```
📋 Epic Issue: "{要件名}"
│  labels: epic, mille
│  body: 要件サマリー、信頼性分布、全Phase進捗チェックリスト
│
├── 🔖 Milestone: "仕様策定"
│   ├── #{N} 要件定義        labels: spec, phase-1
│   ├── #{N} 技術設計        labels: design, phase-2
│   └── #{N} タスク分割      labels: planning, phase-3
│
├── 🔖 Milestone: "実装"
│   ├── #{N} TASK-0001: ...  labels: task, wave-1
│   ├── #{N} TASK-0002: ...  labels: task, wave-1
│   ├── #{N} TASK-0003: ...  labels: task, wave-2, blocked
│   └── ...
│
├── 🔖 Milestone: "品質保証"
│   ├── #{N} 自動レビュー    labels: review, phase-6
│   ├── #{N} E2Eテスト       labels: e2e, phase-6
│   └── #{N} マージ判定      labels: merge, phase-6
│
└── 🔖 Milestone: "フィードバック"  ← Phase 8 で自動作成
    ├── #{N} [BUG] ログイン失敗時のエラーハンドリング不足
    ├── #{N} [SPEC-GAP] メール認証フローが未定義
    └── #{N} [ENHANCE] OAuth2 対応追加
```

### Issue テンプレート

#### Epic Issue

```markdown
## {要件名}

### 概要
{requirements.md の概要セクション}

### 信頼性分布
- 🔵 確定: {N}件
- 🟡 推測: {N}件
- 🔴 不明: {N}件

### Phase 進捗
- [ ] Phase 1: 要件定義
- [ ] Phase 2: 技術設計
- [ ] Phase 3: タスク分割
- [ ] Phase 4: 環境構築
- [ ] Phase 5: TDD実装
- [ ] Phase 6: 品質保証
- [ ] Phase 7: ポストマージ

### 成果物リンク
| Phase | 成果物 |
|-------|--------|
| 1 | docs/spec/{要件名}/requirements.md |
| 2 | docs/design/{要件名}/architecture.md |
| 3 | docs/tasks/{要件名}/overview.md |
```

#### TASK Issue

```markdown
## TASK-{NNNN}: {タスク名}

### タスクファイル
docs/tasks/{要件名}/TASK-{NNNN}.md

### 対応する設計要素
- FR: {FR-ID} ({FR名})
- コンポーネント: {コンポーネント名}
- API: {エンドポイント}（該当する場合）

### 受け入れ基準
{TASK-NNNN.md から抽出}

### 依存関係
- blocked by: #{前提Issue番号}
- blocks: #{後続Issue番号}

### 信頼性レベル
🔵{N} 🟡{N} 🔴{N}

### Wave
Wave {N}（並列実行グループ）
```

#### フィードバック Issue（Phase 8 で自動起票）

```markdown
## [{分類}] {タイトル}

### 検出元
{canary / E2E / ユーザー報告 / ログ分析}

### 分類
- タイプ: {BUG / SPEC-GAP / ENHANCE / PERF}
- 深刻度: {CRITICAL / HIGH / MEDIUM / LOW}
- 影響範囲: {FR-ID / コンポーネント名}

### 再現手順（BUG の場合）
{自動検出されたエラーログ / スタックトレース}

### 関連する仕様
- 元要件: #{Epic Issue番号}
- 関連FR: {FR-ID}
- 関連TASK: #{TASK Issue番号}

### 推奨対応
- 再突入先: {Phase 1 / Phase 3 / Phase 5}
- 理由: {仕様不足 / 実装バグ / 新規要件}
- 見積もり effort: CC: ~{N}min
```

---

## Phase 変更詳細

### Phase 3 変更: タスク分割 + Issue 自動起票

Phase 3 完了時に以下を自動実行:

```
1. Epic Issue 作成（未作成の場合）
2. Milestone 作成: "仕様策定", "実装", "品質保証"
3. 仕様 Issue 作成: 要件定義、技術設計、タスク分割（完了済みとしてクローズ）
4. TASK Issue 一括起票:
   - TASK-NNNN.md ごとに1 Issue
   - blocked_by → Issue 間リンク（gh issue edit --add-label blocked）
   - Wave 番号をラベルに付与（wave-1, wave-2, ...）
5. Epic Issue の body を更新（進捗チェックリスト）
```

#### Wave 算出ロジック

```
入力: TASK-NNNN.md 群の依存関係グラフ
出力: Wave 割り当て

1. 依存関係のないタスクを Wave 1 に割り当て
2. Wave 1 のみに依存するタスクを Wave 2 に割り当て
3. 再帰的に繰り返し
4. 循環依存がある場合はエラー報告

例:
  TASK-0001 (依存なし)     → Wave 1
  TASK-0002 (依存なし)     → Wave 1
  TASK-0003 (← 0001)      → Wave 2
  TASK-0004 (← 0001,0002) → Wave 2
  TASK-0005 (← 0003)      → Wave 3
```

### Phase 5 変更: 並列TDD実装

```
Phase 5 実行フロー:

1. Wave 算出（Phase 3 で計算済み）
2. Wave ごとに並列実行:

   Wave 1:
   ├── Agent worktree 1 → TASK-0001
   │   ├── git checkout -b task/0001
   │   ├── /mille:kairo-implement TASK-0001
   │   ├── git push -u origin task/0001
   │   └── gh pr create --body "Closes #{TASK-0001 Issue番号}"
   │
   ├── Agent worktree 2 → TASK-0002
   │   └── （同上）
   │
   └── Agent worktree 3 → TASK-0005
       └── （同上）

   → 全 Worker 完了を待機
   → WTF-likelihood チェック（Wave 単位で集計）

   Wave 2:
   ├── Wave 1 の PR が全て CI グリーンか確認
   │   → 失敗あり: CI修正ループ発動（後述）
   │   → 全グリーン: Wave 2 開始
   └── （Wave 1 と同じパターン）

3. 全 Wave 完了後:
   - Scope Drift Detection
   - 受け入れ基準充足度チェック
```

#### 並列数の制御

```
デフォルト: 3 並列（Agent worktree の負荷を考慮）
--parallel N: 並列数を明示指定
--sequential: 直列実行（従来の kairo-loop 互換）
```

### Phase 6（新設）: 自動品質保証パイプライン

```
6-1. 自動レビュー
     各 TASK PR に対して並列実行:
     - code-reviewer: 差分サイズ連動深度（Light/Standard/Deep）
     - security-reviewer: Confidence Gate 8/10
     結果の分類:
     - AUTO-FIX → 自動コミット（フォーマット、import順序等）
     - CRITICAL → PR にブロッキングコメント + Issue にフラグ
     - その他 → PR コメントに記録（ブロックしない）

6-2. E2Eテスト
     requirements.md の Critical Path を自動検出:
     - 認証フロー
     - 決済フロー
     - データ永続化フロー
     e2e-runner で Playwright テストを生成・実行
     失敗 → CI修正ループに委譲

6-3. CI修正ループ（PR 単位で自動実行）
     トリガー: CI 失敗
     ループ（最大3回）:
       1. エラー分類: lint → type → test → codegen の優先順
       2. auto-debug で原因特定・修正
       3. fix commit + push
       4. CI 再実行待ち
     3回失敗 → Issue コメント「自動修正に失敗。手動対応が必要」
              + label: needs-human

6-4. マージ判定（確認ゲート）
     全 PR の状態を集約:
     - ✅ 全CI グリーン + CRITICAL ゼロ + E2E PASS
     - ⚠️ MEDIUM 指摘あり（ブロックしない）
     - ❌ CRITICAL あり or CI 赤（ブロック）

     AskUserQuestion:
       "マージ可能な PR:
        ✅ #20 TASK-0001, ✅ #21 TASK-0002
        ⚠️ #22 TASK-0003 (MEDIUM指摘2件)
        ❌ #23 TASK-0004 (CI赤、自動修正失敗)

        1. ✅⚠️ を一括マージ（推奨）
        2. ✅ のみマージ
        3. 個別に確認
        4. 全て保留"

     --auto-merge 指定時:
       ✅ の PR を確認なしで自動マージ
       ⚠️ は PR コメントに記録してマージ
       ❌ はブロック（Issue に報告）
```

### Phase 7（新設）: ポストマージ自動化

```
全て確認ゲートなし（完全自動）:

7-1. ドキュメント自動更新
     doc-updater エージェント:
     - CLAUDE.md: 新規コンポーネント・APIの追記
     - README.md: 機能一覧の更新
     - CHANGELOG.md: マージされた PR から自動生成
     → コミット + プッシュ

7-2. Epic Issue 更新 + クローズ
     - 全子 Issue のステータスを確認
     - 全完了 → Epic Issue に完了サマリーを記載してクローズ
     - 未完了あり → 残タスクを Epic body に明示

7-3. リリースノート下書き
     全 TASK Issue の変更を集約:
     - 新機能（feat）
     - バグ修正（fix）
     - 破壊的変更（breaking）
     gh release create --draft で下書き作成

7-4. メトリクス記録（/retro 連携）
     - コミット数、変更行数、テストカバレッジ
     - 信頼性レベル解決率（🔴→🔵 変換数）
     - Wave 並列効率（実時間 / 直列見積もり時間）
     - CI修正ループ発動回数
     → Obsidian Daily Note に記録

7-5. canary モニタリング起動
     デプロイ先がある場合:
     - /canary でヘルスチェック開始
     - 15分間隔でページロード・コンソールエラー・レスポンス時間を監視
     - 異常検出 → Phase 8 に自動遷移
```

---

## Phase 8（新設）: フィードバックループ

Phase 7 完了後、プロダクトは稼働状態に入る。
ここからの変更要求・バグ・改善を **自動検知→分類→適切な Phase に再突入** する仕組み。

### 8-1. フィードバック収集（自動）

```
入力ソース:
┌─────────────────────┐
│ canary モニタリング   │→ エラーログ、レスポンス劣化
│ CI/CD               │→ テスト失敗、ビルドエラー
│ GitHub Issue（手動）  │→ ユーザー/チームからの報告
│ Sentry/ログ監視      │→ ランタイムエラー（MCP連携時）
└─────────────────────┘
```

### 8-2. 自動分類（トリアージ）

検出されたフィードバックを4タイプに自動分類:

```
┌─────────────┬───────────────────────┬────────────────────────┐
│ 分類         │ 定義                  │ 再突入先                │
├─────────────┼───────────────────────┼────────────────────────┤
│ BUG         │ 既存仕様の実装が       │ Phase 5 直接           │
│             │ 動作しない             │ （新TASK Issue起票→    │
│             │                       │  修正実装→PR）         │
├─────────────┼───────────────────────┼────────────────────────┤
│ SPEC-GAP    │ 仕様に記載がない       │ Phase 1 に戻る         │
│             │ ケースが発生した       │ （要件追加ヒアリング→  │
│             │                       │  設計→タスク→実装）    │
├─────────────┼───────────────────────┼────────────────────────┤
│ ENHANCE     │ 新しい機能要求         │ Phase 1 に戻る         │
│             │                       │ （新Epic or 既存Epic   │
│             │                       │  に子Issue追加）       │
├─────────────┼───────────────────────┼────────────────────────┤
│ PERF        │ パフォーマンス劣化     │ Phase 3 に戻る         │
│             │ （仕様は満たしている） │ （調査TASK起票→       │
│             │                       │  最適化実装→PR）       │
└─────────────┴───────────────────────┴────────────────────────┘
```

### 8-3. 自動分類ロジック

```
入力: エラーログ / Issue 本文 / canary レポート

Step 1: 既存仕様との照合
  - requirements.md の FR/NFR と照合
  - 該当する FR がある → BUG or PERF
  - 該当する FR がない → SPEC-GAP or ENHANCE

Step 2: BUG vs PERF の判別
  - エラー/例外が発生 → BUG
  - 機能は動くがレスポンス劣化 → PERF

Step 3: SPEC-GAP vs ENHANCE の判別
  - 既存機能の延長線上（同一ドメイン内） → SPEC-GAP
  - 新しいドメイン/機能領域 → ENHANCE

Step 4: 深刻度の判定
  - CRITICAL: サービス停止、データ損失、セキュリティ
  - HIGH: 主要機能が使えない
  - MEDIUM: 回避手段がある不具合
  - LOW: 見た目の問題、改善提案
```

### 8-4. 再突入フロー

#### BUG → Phase 5 直接再突入

```
1. フィードバック Issue を自動起票（BUGテンプレート）
2. 既存 Epic Issue の Milestone "フィードバック" に紐付け
3. /investigate で根本原因分析（3-strike ルール適用）
4. 修正 TASK Issue 起票（TASK-FXXX 採番）
5. Phase 5 に再突入（単一タスク、直列実行）
6. Phase 6 の CI修正ループ + レビューを経て PR
7. Closes #{フィードバック Issue}

自動化度: CRITICAL は即座に自動実行、HIGH以下は確認ゲート1回
```

#### SPEC-GAP → Phase 1 再突入

```
1. フィードバック Issue を自動起票（SPEC-GAPテンプレート）
2. 既存 requirements.md の該当セクションを特定
3. Phase 1 に再突入:
   - kairo-requirements を差分モードで実行
   - 既存要件を保持しつつ、不足分を追加
   - 信頼性レベルを再評価（🔴→🟡→🔵）
4. Phase 2: 差分設計（既存設計に追加のみ）
5. Phase 3: 追加 TASK 起票（既存 Wave の後に追加 Wave）
6. Phase 5-7: 通常フロー

自動化度: Phase 1 の確認ゲートで人間が判断（仕様の妥当性）
```

#### ENHANCE → Phase 1 再突入（新規 or 既存 Epic 拡張）

```
判定: 既存 Epic の範囲内か？
  - 範囲内 → 既存 Epic に子 Issue 追加
  - 範囲外 → 新規 Epic Issue 作成

範囲内の場合:
  1. 既存 Epic の Milestone "フィードバック" に Issue 追加
  2. Phase 1 に差分モードで再突入
  3. 以降は SPEC-GAP と同じフロー

範囲外の場合:
  1. 新規 Epic Issue 作成
  2. /mille:kickoff を新規要件として実行
  3. 元 Epic Issue に「関連: #{新Epic}」をリンク

自動化度: 範囲判定 + Phase 1 確認ゲートで人間が判断
```

#### PERF → Phase 3 再突入

```
1. フィードバック Issue を自動起票（PERFテンプレート）
2. パフォーマンス調査 TASK を起票:
   - プロファイリング実行
   - ボトルネック特定
   - 最適化方針の策定
3. Phase 3 に再突入（調査TASK + 最適化TASK）
4. Phase 5-7: 通常フロー

自動化度: 調査結果の確認ゲート1回（最適化方針の承認）
```

### 8-5. フィードバックの優先順位制御

```
処理順序:
  1. CRITICAL BUG → 即時対応（他の作業を中断）
  2. HIGH BUG → 次の作業サイクルで対応
  3. SPEC-GAP → バッチで収集し、まとめて Phase 1 再突入
  4. PERF → バッチで収集
  5. ENHANCE → バックログ管理（定期的に優先度見直し）
  6. LOW → バックログ

バッチ処理:
  SPEC-GAP と PERF は即座に再突入せず、一定期間（デフォルト: 1日）
  バッファリングしてからまとめて処理する。
  理由: 個別に Phase 1 再突入すると、毎回ヒアリングが発生して非効率。
  関連する SPEC-GAP をまとめて1回のヒアリングで解決する。
```

### 8-6. フィードバック→再突入の追跡

```
Epic Issue の body に自動追記:

### フィードバック履歴
| # | 分類 | 深刻度 | 再突入先 | 状態 |
|---|------|--------|---------|------|
| #40 | BUG | CRITICAL | Phase 5 | ✅ 修正済み |
| #41 | SPEC-GAP | MEDIUM | Phase 1 | 🔄 要件追加中 |
| #42 | ENHANCE | LOW | バックログ | 📋 保留 |
```

---

## 確認ゲート配置の最終形

```
Phase 0   技術スタック     → 自動（CLAUDE.md があれば）
Phase 0.5 リサーチ         → 自動
Phase 1   要件定義         → ✋ 確認ゲート（仕様判断）
Phase 1.5 Constitution     → 自動
Phase 2   技術設計         → ✋ 確認ゲート（設計判断）
Phase 3   タスク分割       → ✋ 確認ゲート + Issue自動起票
Phase 4   環境構築         → ✋ 確認ゲート（初回のみ）
Phase 5   並列TDD実装      → 自動（Wave 単位で並列）
Phase 6   品質保証         → 自動（6-4 マージ判定のみ ✋、--auto-merge で省略可）
Phase 7   ポストマージ     → 完全自動
Phase 8   フィードバック   → 分類による:
                             - CRITICAL BUG: 自動対応
                             - その他 BUG: ✋ 1回
                             - SPEC-GAP/ENHANCE: ✋ Phase 1 で判断
                             - PERF: ✋ 調査結果確認 1回
```

---

## 実装対象ファイル

| 優先度 | ファイル | 変更内容 |
|--------|---------|---------|
| 1 | `.claude/commands/mille:kickoff.md` | Phase 3 に Issue 起票追加、Phase 5 並列化、Phase 6-8 新設 |
| 2 | `.claude/skills/issue-workflow/SKILL.md` | mille 統合版に書き直し |
| 3 | `.claude/skills/ci-autofix/SKILL.md` | 新規: CI失敗→自動修正ループ |
| 4 | `.claude/skills/feedback-triage/SKILL.md` | 新規: Phase 8 フィードバック分類・再突入 |
| 5 | `.claude/commands/ship.md` | Phase 6 との統合（/ship が Phase 6 を呼ぶ形に） |

## コマンド体系

```
/mille:kickoff {要件名}              → Phase 0-7 一気通貫
/mille:kickoff {要件名} --skip-impl  → Phase 0-4 まで（設計のみ）
/mille:kickoff {要件名} --auto-merge → Phase 6-4 の確認ゲート省略
/mille:kickoff {要件名} --sequential → Phase 5 を直列実行（従来互換）
/mille:kickoff {要件名} --parallel N → Phase 5 の並列数指定

/ship                                  → Phase 6 単体実行（既存PR対象）
/feedback {Issue番号 or エラー内容}     → Phase 8 単体実行
```

---
name: issue-workflow
description: mille v2 統合 Issue 駆動ワークフロー - Epic Issue / TASK Issue / フィードバック Issue / Milestone を gh CLI で管理し、mille:kickoff の全フェーズを GitHub Issue ツリーで追跡する。
---

# Issue 駆動ワークフロー（mille v2 統合）

## 概要

mille:kickoff の Phase 3 完了時に Epic Issue を起点とした Issue ツリーを構築し、
Phase 5-8 の実装・品質保証・フィードバックを GitHub Issue で一元管理する。

## Issue ツリー構造

```
Epic Issue: "{要件名}"
│  labels: epic, mille
│  body: 要件サマリー、信頼性分布、Phase 進捗チェックリスト
│
├── Milestone: "仕様策定"
│   ├── #{N} 要件定義        labels: spec, phase-1
│   ├── #{N} 技術設計        labels: design, phase-2
│   └── #{N} タスク分割      labels: planning, phase-3
│
├── Milestone: "実装"
│   ├── #{N} TASK-0001: ...  labels: task, wave-1
│   ├── #{N} TASK-0002: ...  labels: task, wave-1
│   ├── #{N} TASK-0003: ...  labels: task, wave-2, blocked
│   └── ...
│
├── Milestone: "品質保証"
│   ├── #{N} 自動レビュー    labels: review, phase-6
│   ├── #{N} E2Eテスト       labels: e2e, phase-6
│   └── #{N} マージ判定      labels: merge, phase-6
│
└── Milestone: "フィードバック"  ← Phase 8 で自動作成
    ├── #{N} [BUG] ...
    ├── #{N} [SPEC-GAP] ...
    └── #{N} [ENHANCE] ...
```

## コマンド体系

```
/issue-workflow create {要件名}         → Epic + TASK Issue 一括起票
/issue-workflow status {Epic Issue番号}  → 進捗サマリー表示
/issue-workflow feedback {内容}          → フィードバック Issue 起票
/issue-workflow close {Epic Issue番号}   → 完了処理
```

---

## 1. `/issue-workflow create {要件名}`

Phase 3 完了後に実行する。TASK-NNNN.md 群と requirements.md を入力として、
Epic Issue + Milestone + 全 TASK Issue を一括起票する。

### Step 1: 入力ファイルの収集

```bash
# mille の成果物ディレクトリを特定
SPEC_DIR="docs/spec/{要件名}"    # requirements.md
TASK_DIR="docs/tasks/{要件名}"   # TASK-NNNN.md 群

# 必須ファイルの存在確認
ls "$SPEC_DIR/requirements.md"
ls "$TASK_DIR"/TASK-*.md
```

### Step 2: Wave 算出

TASK-NNNN.md の `blocked_by` フィールドから依存関係グラフを構築し、Wave を算出する。

```
アルゴリズム:
1. 全 TASK ファイルを読み取り、blocked_by を抽出
2. 依存関係のないタスクを Wave 1 に割り当て
3. Wave N のみに依存するタスクを Wave N+1 に割り当て
4. 再帰的に繰り返す
5. 循環依存を検出した場合はエラー報告して停止

例:
  TASK-0001 (依存なし)     → Wave 1
  TASK-0002 (依存なし)     → Wave 1
  TASK-0003 (← 0001)      → Wave 2
  TASK-0004 (← 0001,0002) → Wave 2
  TASK-0005 (← 0003)      → Wave 3
```

循環依存の検出時:
```
[ERROR] 循環依存を検出:
  TASK-0003 → TASK-0005 → TASK-0003
対処: TASK ファイルの blocked_by を修正してから再実行してください。
```

### Step 3: Milestone 作成

4つの Milestone を作成する。

```bash
OWNER_REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')

for MS in "仕様策定" "実装" "品質保証" "フィードバック"; do
  gh api "repos/$OWNER_REPO/milestones" \
    -f title="$MS" \
    -f state="open"
done
```

### Step 4: Epic Issue 作成

requirements.md から概要と信頼性分布を抽出し、Epic Issue を作成する。

```bash
gh issue create \
  --title "{要件名}" \
  --label "epic,mille" \
  --body "$(cat <<'EPIC_BODY'
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

### フィードバック履歴
| # | 分類 | 深刻度 | 再突入先 | 状態 |
|---|------|--------|---------|------|
EPIC_BODY
)"
```

`EPIC_NUMBER` を取得して以降のステップで参照する。

### Step 5: 仕様 Issue 作成（完了済みとしてクローズ）

Phase 1-3 は既に完了しているため、記録用に Issue を作成して即クローズする。

```bash
for ITEM in "要件定義:spec:phase-1" "技術設計:design:phase-2" "タスク分割:planning:phase-3"; do
  TITLE=$(echo "$ITEM" | cut -d: -f1)
  LABELS=$(echo "$ITEM" | cut -d: -f2,3 | tr : ,)

  ISSUE_URL=$(gh issue create \
    --title "$TITLE" \
    --label "$LABELS" \
    --milestone "仕様策定" \
    --body "Epic: #$EPIC_NUMBER")

  gh issue close "$ISSUE_URL" --reason completed
done
```

### Step 6: TASK Issue 一括起票

各 TASK-NNNN.md を読み取り、子 Issue として起票する。

```bash
for TASK_FILE in "$TASK_DIR"/TASK-*.md; do
  TASK_ID=$(basename "$TASK_FILE" .md)  # TASK-0001
  WAVE_LABEL="wave-$(get_wave "$TASK_ID")"
  BLOCKED_BY=$(extract_blocked_by "$TASK_FILE")

  gh issue create \
    --title "$TASK_ID: {タスク名}" \
    --label "task,$WAVE_LABEL" \
    --milestone "実装" \
    --body "$(cat <<TASK_BODY
## $TASK_ID: {タスク名}

### タスクファイル
$TASK_FILE

### 対応する設計要素
- FR: {FR-ID} ({FR名})
- コンポーネント: {コンポーネント名}
- API: {エンドポイント}（該当する場合）

### 受け入れ基準
{TASK-NNNN.md から抽出}

### 依存関係
$BLOCKED_BY

### 信頼性レベル
{TASK-NNNN.md から抽出}

### Wave
$WAVE_LABEL（並列実行グループ）

Epic: #$EPIC_NUMBER
TASK_BODY
)"
done
```

blocked_by がある TASK には `blocked` ラベルも追加する:
```bash
gh issue edit $ISSUE_NUMBER --add-label "blocked"
```

Issue 作成後、依存関係を Issue 間リンクとして記録する:
```bash
# blocked_by に対応する Issue 番号を特定し、body に依存リンクを追加
# - blocked by: #XX
# - blocks: #YY
```

### Step 7: Epic Issue 更新

全 TASK Issue の番号を Epic Issue の body に追記する。Phase 1-3 のチェックを完了済みに更新する。

```bash
# Epic body を更新: Phase 1-3 にチェックを入れる
gh issue edit $EPIC_NUMBER --body "$(updated_epic_body)"
```

---

## 2. `/issue-workflow status {Epic Issue番号}`

Epic Issue 配下の全 Issue の状態を集約して表示する。

### 出力フォーマット

```
Epic: #42 ユーザー認証基盤

Phase 進捗: 5/7 完了
  [x] Phase 1-4
  [x] Phase 5: TDD実装
  [ ] Phase 6: 品質保証
  [ ] Phase 7: ポストマージ

Milestone: 実装
  Wave 1: 3/3 完了
    ✅ #50 TASK-0001: DB スキーマ定義
    ✅ #51 TASK-0002: 認証ミドルウェア
    ✅ #52 TASK-0003: ユーザーモデル
  Wave 2: 1/2 完了
    ✅ #53 TASK-0004: ログイン API
    🔄 #54 TASK-0005: セッション管理 (PR #60 open)
  Wave 3: 0/1 未着手
    ⏳ #55 TASK-0006: OAuth2 連携 (blocked by #54)

Milestone: 品質保証
  ⏳ 未着手

フィードバック: 1件
  🐛 #70 [BUG] CRITICAL: トークンリフレッシュ競合 → Phase 5 再突入済み
```

### 実装

```bash
EPIC=$1

# Epic Issue の body を取得
gh issue view "$EPIC" --json body,title

# 子 Issue を Milestone ごとに取得
for MS in "仕様策定" "実装" "品質保証" "フィードバック"; do
  gh issue list --milestone "$MS" --json number,title,state,labels
done

# PR リンクを確認
gh pr list --json number,title,headRefName,state
```

---

## 3. `/issue-workflow feedback {内容}`

Phase 8 のフィードバック Issue を起票する。4テンプレートから適切なものを選択する。

### 分類ロジック

```
入力: エラーログ / Issue 本文 / canary レポート

Step 1: 既存仕様との照合
  - requirements.md の FR/NFR と照合
  - 該当する FR あり → BUG or PERF
  - 該当する FR なし → SPEC-GAP or ENHANCE

Step 2: BUG vs PERF
  - エラー/例外が発生 → BUG
  - 機能は動くがレスポンス劣化 → PERF

Step 3: SPEC-GAP vs ENHANCE
  - 既存機能の延長線上（同一ドメイン内） → SPEC-GAP
  - 新しいドメイン/機能領域 → ENHANCE

Step 4: 深刻度の判定
  - CRITICAL: サービス停止、データ損失、セキュリティ
  - HIGH: 主要機能が使えない
  - MEDIUM: 回避手段がある不具合
  - LOW: 見た目の問題、改善提案
```

### BUG テンプレート

```bash
gh issue create \
  --title "[BUG] {タイトル}" \
  --label "bug,feedback,{深刻度}" \
  --milestone "フィードバック" \
  --body "$(cat <<'EOF'
## [BUG] {タイトル}

### 検出元
{canary / E2E / ユーザー報告 / ログ分析}

### 分類
- タイプ: BUG
- 深刻度: {CRITICAL / HIGH / MEDIUM / LOW}
- 影響範囲: {FR-ID / コンポーネント名}

### 再現手順
{自動検出されたエラーログ / スタックトレース}

### 関連する仕様
- 元要件: #{Epic Issue番号}
- 関連FR: {FR-ID}
- 関連TASK: #{TASK Issue番号}

### 推奨対応
- 再突入先: Phase 5
- 理由: 実装バグ
- 見積もり effort: CC: ~{N}min
EOF
)"
```

### SPEC-GAP テンプレート

```bash
gh issue create \
  --title "[SPEC-GAP] {タイトル}" \
  --label "spec-gap,feedback,{深刻度}" \
  --milestone "フィードバック" \
  --body "$(cat <<'EOF'
## [SPEC-GAP] {タイトル}

### 検出元
{canary / E2E / ユーザー報告 / ログ分析}

### 分類
- タイプ: SPEC-GAP
- 深刻度: {CRITICAL / HIGH / MEDIUM / LOW}
- 影響範囲: {FR-ID / コンポーネント名}

### 不足している仕様
{仕様に記載がないケースの説明}

### 関連する仕様
- 元要件: #{Epic Issue番号}
- 関連FR: {FR-ID}（最も近い既存要件）

### 推奨対応
- 再突入先: Phase 1
- 理由: 仕様不足
- 見積もり effort: CC: ~{N}min
EOF
)"
```

### ENHANCE テンプレート

```bash
gh issue create \
  --title "[ENHANCE] {タイトル}" \
  --label "enhancement,feedback,{深刻度}" \
  --milestone "フィードバック" \
  --body "$(cat <<'EOF'
## [ENHANCE] {タイトル}

### 検出元
{canary / E2E / ユーザー報告 / ログ分析}

### 分類
- タイプ: ENHANCE
- 深刻度: {CRITICAL / HIGH / MEDIUM / LOW}
- 影響範囲: {FR-ID / コンポーネント名}

### 要求内容
{新機能の説明}

### 関連する仕様
- 元要件: #{Epic Issue番号}

### 推奨対応
- 再突入先: Phase 1
- 理由: 新規要件
- 見積もり effort: CC: ~{N}min
EOF
)"
```

### PERF テンプレート

```bash
gh issue create \
  --title "[PERF] {タイトル}" \
  --label "performance,feedback,{深刻度}" \
  --milestone "フィードバック" \
  --body "$(cat <<'EOF'
## [PERF] {タイトル}

### 検出元
{canary / E2E / ユーザー報告 / ログ分析}

### 分類
- タイプ: PERF
- 深刻度: {CRITICAL / HIGH / MEDIUM / LOW}
- 影響範囲: {FR-ID / コンポーネント名}

### パフォーマンス劣化の詳細
{レスポンス時間、スループット等の具体的な数値}

### 関連する仕様
- 元要件: #{Epic Issue番号}
- 関連FR: {FR-ID}
- 関連TASK: #{TASK Issue番号}

### 推奨対応
- 再突入先: Phase 3
- 理由: パフォーマンス劣化（仕様は満たしている）
- 見積もり effort: CC: ~{N}min
EOF
)"
```

### フィードバック起票後の自動処理

1. Epic Issue の「フィードバック履歴」テーブルに行を追加する
2. CRITICAL BUG は即座に Phase 5 再突入を提案する
3. SPEC-GAP / ENHANCE はバッチ収集を推奨する（まとめて Phase 1 再突入）

```bash
# Epic Issue の body にフィードバック履歴を追記
gh issue edit $EPIC_NUMBER --body "$(append_feedback_row)"
```

---

## 4. `/issue-workflow close {Epic Issue番号}`

Epic 配下の全 Issue が完了していることを確認し、完了処理を行う。

### Step 1: 完了確認

```bash
EPIC=$1

# 未完了 Issue の確認
OPEN_ISSUES=$(gh issue list --milestone "実装" --state open --json number,title)
OPEN_QA=$(gh issue list --milestone "品質保証" --state open --json number,title)

# 未完了があればブロック
if [ -n "$OPEN_ISSUES" ] || [ -n "$OPEN_QA" ]; then
  echo "[ERROR] 未完了の Issue があります:"
  echo "$OPEN_ISSUES"
  echo "$OPEN_QA"
  echo "全 Issue を完了してから再実行してください。"
  exit 1
fi
```

### Step 2: Milestone クローズ

```bash
for MS_NUMBER in $(gh api "repos/$OWNER_REPO/milestones" --jq '.[].number'); do
  gh api "repos/$OWNER_REPO/milestones/$MS_NUMBER" \
    -X PATCH -f state="closed"
done
```

### Step 3: Epic Issue の Phase 進捗を全チェック

```bash
# Phase 1-7 全てにチェックを入れた body で更新
gh issue edit $EPIC_NUMBER --body "$(final_epic_body)"
```

### Step 4: 完了サマリーをコメントして Epic をクローズ

```bash
gh issue comment $EPIC_NUMBER --body "$(cat <<'EOF'
## 完了サマリー

### 実装結果
- TASK Issue: {N}件完了
- PR: {N}件マージ済み
- フィードバック Issue: {N}件対応済み

### メトリクス
- 総コミット数: {N}
- 変更行数: +{N} / -{N}
- テストカバレッジ: {N}%
- Wave 数: {N}
- CI修正ループ発動: {N}回

### 信頼性レベル解決
- 🔴→🔵: {N}件
- 🔴→🟡: {N}件
- 残存🔴: {N}件
EOF
)"

gh issue close $EPIC_NUMBER --reason completed
```

---

## Wave 算出の詳細

### 入力

TASK-NNNN.md の `blocked_by` フィールド（TASK ID のリスト）。

### アルゴリズム

```
function compute_waves(tasks):
  assigned = {}
  remaining = set(tasks)

  wave = 1
  while remaining is not empty:
    current_wave = []
    for task in remaining:
      deps = task.blocked_by
      if deps is empty OR all deps are in assigned:
        current_wave.append(task)

    if current_wave is empty:
      # remaining にタスクがあるのに Wave に追加できない = 循環依存
      report_cycle(remaining)
      abort

    for task in current_wave:
      assigned[task.id] = wave
    remaining -= set(current_wave)
    wave += 1

  return assigned
```

### 循環依存の検出

全タスクを処理しきる前に Wave に追加できるタスクがなくなった場合、
残存タスク群に循環依存がある。残存タスクの依存関係を辿って循環パスを報告する。

---

## Epic Issue 自動更新

以下のタイミングで Epic Issue の body を更新する:

| タイミング | 更新内容 |
|-----------|---------|
| Phase 3 完了 | Phase 1-3 チェック、TASK Issue リンク追加 |
| Phase 4 完了 | Phase 4 チェック |
| Wave N 完了 | 対応する TASK Issue のステータス反映 |
| Phase 5 完了 | Phase 5 チェック |
| Phase 6 完了 | Phase 6 チェック、品質保証結果サマリー |
| Phase 7 完了 | Phase 7 チェック、メトリクス追記 |
| フィードバック起票 | フィードバック履歴テーブルに行追加 |
| フィードバック解決 | フィードバック履歴の状態を更新 |

更新は `gh issue edit` で body 全体を置換する:
```bash
gh issue edit $EPIC_NUMBER --body "$(generate_epic_body)"
```

---

## PR との連携

TASK Issue を PR で自動クローズするため、PR の body に `Closes #XX` を含める。

```bash
gh pr create \
  --title "feat: {TASK-NNNN の内容}" \
  --body "$(cat <<EOF
Closes #{TASK Issue番号}
Epic: #{EPIC Issue番号}

## 変更内容
{変更の要約}

## テスト
- テストカバレッジ: {N}%
EOF
)"
```

Wave 内の全 TASK PR がマージされたら、次の Wave の `blocked` ラベルを除去する:
```bash
# Wave N+1 の blocked ラベルを除去
gh issue edit $NEXT_WAVE_ISSUE --remove-label "blocked"
```

---

## ラベル設計

| ラベル | 用途 |
|--------|------|
| `epic` | Epic Issue |
| `mille` | mille 管理下の Issue |
| `task` | TASK Issue |
| `wave-1`, `wave-2`, ... | Wave 番号（並列実行グループ） |
| `blocked` | 依存する Issue が未完了 |
| `spec` | 仕様関連 |
| `design` | 設計関連 |
| `planning` | 計画関連 |
| `phase-1` 〜 `phase-7` | Phase 番号 |
| `review` | レビュー |
| `e2e` | E2Eテスト |
| `merge` | マージ判定 |
| `feedback` | フィードバック Issue |
| `bug` | BUG 分類 |
| `spec-gap` | SPEC-GAP 分類 |
| `enhancement` | ENHANCE 分類 |
| `performance` | PERF 分類 |
| `critical`, `high`, `medium`, `low` | 深刻度 |
| `needs-human` | 自動修正失敗、手動対応が必要 |

---

## 関連エージェント

- `planner` - Epic / TASK Issue 作成時の計画支援、Wave 算出の検証

---
name: feedback-triage
description: |
  Phase 8 フィードバック自動分類・再突入スキル。運用中のフィードバック（バグ、仕様漏れ、機能追加、パフォーマンス問題）を
  4タイプに自動分類し、適切な Phase に再突入させる。mille:kickoff v2 の Phase 8 で使用。
  使用場面: (1) /feedback でフィードバックを分類・再突入 (2) /canary からの異常検出を受け取る
  (3) /feedback --batch でバッファリング項目をまとめて処理 (4) /feedback --status でフィードバック履歴表示
trigger: /feedback
---

# Feedback Triage — Phase 8 フィードバック自動分類・再突入

運用中のフィードバックを自動検知→分類→適切な Phase に再突入させる。
mille:kickoff v2 の Phase 8 として機能する。

## コマンド体系

```
/feedback {エラー内容 or Issue番号}  → 自動分類 + 再突入提案
/feedback --batch                    → バッファリング中の項目をまとめて処理
/feedback --status {Epic Issue番号}  → フィードバック履歴表示
```

## フィードバック収集（入力ソース）

```
┌─────────────────────┐
│ canary モニタリング   │→ エラーログ、レスポンス劣化（自動入力）
│ CI/CD               │→ テスト失敗、ビルドエラー
│ GitHub Issue（手動）  │→ ユーザー/チームからの報告
│ Sentry/ログ監視      │→ ランタイムエラー（MCP連携時）
└─────────────────────┘
```

/canary からの異常検出は自動入力として受け付ける。canary レポートに含まれるエラーログ・レスポンス劣化情報をそのまま分類パイプラインに投入する。

## 4タイプ分類

| 分類 | 定義 | 再突入先 |
|------|------|---------|
| BUG | 既存仕様の実装が動作しない | Phase 5 直接 |
| SPEC-GAP | 仕様に記載がないケースが発生 | Phase 1 に戻る |
| ENHANCE | 新しい機能要求 | Phase 1 に戻る |
| PERF | パフォーマンス劣化（仕様は満たしている） | Phase 3 に戻る |

## 自動分類ロジック

```
入力: エラーログ / Issue 本文 / canary レポート

Step 1: 既存仕様との照合
  - requirements.md の FR/NFR を読み込む
  - 該当する FR がある → BUG or PERF（Step 2 へ）
  - 該当する FR がない → SPEC-GAP or ENHANCE（Step 3 へ）

Step 2: BUG vs PERF の判別
  - エラー/例外が発生している → BUG
  - 機能は動作するがレスポンス劣化・タイムアウト → PERF

Step 3: SPEC-GAP vs ENHANCE の判別
  - 既存機能の延長線上（同一ドメイン内の未カバーケース） → SPEC-GAP
  - 新しいドメイン/機能領域 → ENHANCE

Step 4: 深刻度の判定
  - CRITICAL: サービス停止、データ損失、セキュリティ脆弱性
  - HIGH: 主要機能が使えない
  - MEDIUM: 回避手段がある不具合
  - LOW: 見た目の問題、改善提案
```

### 分類判定の実行手順

1. プロジェクトの `docs/spec/*/requirements.md` を読み込む
2. フィードバック内容のキーワード・スタックトレース・エラーコードを抽出する
3. FR/NFR の受け入れ基準と照合し、Step 1-4 を順に実行する
4. 分類結果と根拠をまとめ、ユーザーに提示する（自動分類に自信がない場合は候補を2つ提示）

## 優先順位

処理順序（上が最優先）:

1. **CRITICAL BUG** → 即時対応（他の作業を中断）
2. **HIGH BUG** → 次の作業サイクルで対応
3. **SPEC-GAP** → バッチで収集し、まとめて Phase 1 再突入
4. **PERF** → バッチで収集
5. **ENHANCE** → バックログ管理（定期的に優先度見直し）
6. **LOW** → バックログ

## 再突入フロー

### BUG → Phase 5 直接再突入

```
1. フィードバック Issue を自動起票（BUG テンプレート）
2. 既存 Epic Issue の Milestone "フィードバック" に紐付け
3. /investigate で根本原因分析（3-strike ルール適用）
4. 修正 TASK Issue 起票（TASK-FXXX 採番）
5. Phase 5 に再突入（単一タスク、直列実行）
6. Phase 6 の CI修正ループ + レビューを経て PR
7. Closes #{フィードバック Issue}

自動化度:
  - CRITICAL → 即座に自動実行（確認ゲートなし）
  - HIGH 以下 → 確認ゲート1回（対応方針の承認）
```

### SPEC-GAP → Phase 1 再突入

```
1. フィードバック Issue を自動起票（SPEC-GAP テンプレート）
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

### ENHANCE → Phase 1 再突入（新規 or 既存 Epic 拡張）

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

### PERF → Phase 3 再突入

```
1. フィードバック Issue を自動起票（PERF テンプレート）
2. パフォーマンス調査 TASK を起票:
   - プロファイリング実行
   - ボトルネック特定
   - 最適化方針の策定
3. Phase 3 に再突入（調査TASK + 最適化TASK）
4. Phase 5-7: 通常フロー

自動化度: 調査結果の確認ゲート1回（最適化方針の承認）
```

## バッチ処理

SPEC-GAP と PERF は即座に再突入せず、バッファリングしてまとめて処理する。

- **デフォルトバッファリング期間**: 1日
- **理由**: 個別に Phase 1 再突入すると毎回ヒアリングが発生して非効率。関連する SPEC-GAP をまとめて1回のヒアリングで解決する。

### `/feedback --batch` 実行時の動作

```
1. バッファリング中のフィードバック Issue を一覧表示
2. 関連性でグループ化（同一 FR/コンポーネントをまとめる）
3. グループごとに再突入先と対応方針を提示
4. ユーザー承認後、一括で再突入フローを開始
```

### バッファリング管理

```bash
# バッファリング中の Issue は以下のラベルで管理
labels: feedback, buffered, {分類タイプ小文字}

# バッチ処理対象の Issue を取得
gh issue list --label "feedback,buffered" --json number,title,labels
```

## Issue 起票

issue-workflow スキルの feedback テンプレートを使用する。

### フィードバック Issue テンプレート

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
- 理由: {仕様不足 / 実装バグ / 新規要件 / パフォーマンス劣化}
- 見積もり effort: CC: ~{N}min
```

### Issue 起票コマンド

```bash
gh issue create \
  --title "[{分類}] {タイトル}" \
  --body "{テンプレートから生成}" \
  --label "feedback,{分類タイプ小文字},{深刻度小文字}" \
  --milestone "フィードバック"
```

## フィードバック履歴の追跡

### `/feedback --status {Epic Issue番号}` 実行時の動作

Epic Issue の body に以下のテーブルを自動追記・更新する:

```markdown
### フィードバック履歴
| # | 分類 | 深刻度 | 再突入先 | 状態 |
|---|------|--------|---------|------|
| #40 | BUG | CRITICAL | Phase 5 | ✅ 修正済み |
| #41 | SPEC-GAP | MEDIUM | Phase 1 | 🔄 要件追加中 |
| #42 | ENHANCE | LOW | バックログ | 📋 保留 |
| #43 | PERF | HIGH | Phase 3 | 🔄 調査中 |
```

### 集計サマリー

```
フィードバック集計（Epic #{番号}）:
- 合計: {N}件
- BUG: {N}件（修正済み: {N}件、対応中: {N}件）
- SPEC-GAP: {N}件（解決済み: {N}件、バッファリング中: {N}件）
- ENHANCE: {N}件（採用: {N}件、バックログ: {N}件）
- PERF: {N}件（改善済み: {N}件、調査中: {N}件）
```

## 実行ワークフロー

### `/feedback {エラー内容 or Issue番号}` の完全フロー

```
1. 入力の判定
   - Issue番号（#NNN or NNN） → gh issue view で内容取得
   - エラー内容（テキスト）    → そのまま分類パイプラインへ
   - canary レポート          → エラーログ/レスポンスデータを抽出

2. プロジェクトの requirements.md を読み込む
   - docs/spec/*/requirements.md を検索
   - FR/NFR の一覧を取得

3. 自動分類（Step 1-4）を実行

4. 分類結果をユーザーに提示
   ┌─────────────────────────────────────────────┐
   │ 分類結果:                                     │
   │ タイプ: BUG                                   │
   │ 深刻度: HIGH                                  │
   │ 影響範囲: FR-AUTH-001（ログイン認証）           │
   │ 根拠: requirements.md の FR-AUTH-001 に       │
   │       「メール+パスワードで認証できる」と記載   │
   │       あるが、特定条件でエラーが発生            │
   │ 推奨: Phase 5 に再突入                        │
   │       → /investigate で根本原因分析            │
   │       → 修正TASK起票 → TDD実装                │
   └─────────────────────────────────────────────┘

5. ユーザー承認後:
   a. フィードバック Issue 起票
   b. Epic Issue のフィードバック履歴を更新
   c. 再突入フローを開始
      - BUG: /investigate → TASK起票 → Phase 5
      - SPEC-GAP: バッファリング（--batch で処理）
      - ENHANCE: 範囲判定 → Phase 1 or 新規 Epic
      - PERF: バッファリング（--batch で処理）
   d. CRITICAL BUG の場合は確認ゲートなしで即時実行
```

## 関連エージェント

- `debugger` — BUG 調査（/investigate 経由で根本原因分析）
- `research-analyst` — SPEC-GAP 分析（要件の不足範囲を調査）
- `planner` — ENHANCE 計画（新機能の実装計画策定）

## 関連スキル・コマンド

- `/investigate` — BUG の根本原因分析（3-strike ルール）
- `/canary` — ポストデプロイ監視（異常検出を本スキルに自動入力）
- `/mille:kickoff` — ENHANCE の範囲外判定時に新規 Epic として実行
- `issue-workflow` — Issue 起票テンプレートとラベル管理

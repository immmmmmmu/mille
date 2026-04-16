---
description: "Pre-merge パイプライン（mille:kickoff v2 Phase 6 統合）。test → review → coverage → CI修正ループ → マージ判定を自動化。単体実行も /mille:kickoff からの呼び出しも可能。"
---

# /ship — Pre-Merge Pipeline (Phase 6 統合)

gstack の `/ship` パターン + mille:kickoff v2 の Phase 6（自動品質保証パイプライン）を統合。
単体で使う場合は従来通り PR 作成まで。mille:kickoff から呼ばれた場合は Phase 6 のフル機能が有効。

**mille:kickoff 連携時の追加機能**:
- CI修正ループ（`/ci-autofix` 連携、最大3回自動リトライ）
- E2Eテスト自動生成（requirements.md の Critical Path から）
- マージ判定の一括表示（複数 TASK PR を集約）

$ARGUMENTS: オプション（`--draft`, `--skip-review`, `--skip-test`）

## 前提条件チェック

1. ベースブランチ上にいないことを確認（main/master 上なら中止）
2. `git diff --stat` で変更サマリーを表示
3. 未コミットの変更があれば含めるか確認

## Step 1: テスト実行

```bash
# テストフレームワークの自動検出
# package.json の scripts.test、vitest.config、jest.config 等を探索
```

### テストブートストラップ（テスト未設定時）

テストフレームワークが見つからない場合:
1. プロジェクトのランタイムを検出（Node.js, Bun, Deno 等）
2. 適切なテストフレームワークを提案（vitest 推奨）
3. 3-5件の初期テストを自動生成
4. CI 設定ファイルを生成

### テスト結果の分類

- **新規失敗（ブロッキング）**: 今回の変更で発生 → 修正必須
- **既存失敗（非ブロッキング）**: 変更前から存在 → 情報提供のみ
- **全テスト PASS**: 次ステップへ

## Step 2: コードレビュー

`code-reviewer` エージェントを起動（Phase 1 で追加済みの差分サイズ連動深度を使用）:
- Light（<50行）/ Standard（50-199行）/ Deep（200行以上）

### 結果の分類

- **AUTO-FIX**: 機械的な修正（フォーマット、import 順序）→ 自動適用
- **ASK**: 判断が必要な指摘 → AskUserQuestion でバッチ提示
- **CRITICAL**: ブロッキング → 修正必須

## Step 3: テストカバレッジ監査

変更されたコードパスに対するテストカバレッジを確認:
- カバーされていないパスを特定
- 30パス未満のギャップ → テスト自動生成
- 30パス以上のギャップ → ユーザーに報告

## Step 4: Bisectable Commits

変更を論理的な単位でコミット分割:
1. インフラ/設定変更
2. 型定義/インターフェース
3. 実装コード
4. テストコード
5. ドキュメント

各コミットは単独でビルドが通る状態を維持。

## Step 5: 再検証ゲート

Step 2-3 でコードが変更された場合（AUTO-FIX、テスト追加）:
- テストを再実行
- 失敗した場合は修正してから続行

## Step 6: Push & PR作成

```bash
# リモートにプッシュ（upstream tracking 設定）
git push -u origin $(git branch --show-current)

# PR作成
gh pr create --title "{title}" --body "{body}"
```

### PR ボディ自動生成

```markdown
## Summary
{git log からの変更サマリー}

## Changes
{ファイル別の変更概要}

## Test Results
- テスト: {PASS/FAIL} ({N}/{M} 件)
- カバレッジ: {N}%
- 新規テスト: {N}件追加

## Review Results
- code-reviewer: {APPROVE/WARN/BLOCK}
- 指摘: CRITICAL {N}, HIGH {N}, MEDIUM {N}

## Test plan
- [ ] 自動テスト PASS 確認済み
- [ ] コードレビュー完了
{追加のテスト項目}
```

## 必須停止条件（自動化を中断）

- ベースブランチ上にいる
- 今回の変更による新規テスト失敗
- CRITICAL レビュー指摘が未解決
- merge conflict が解決不能
- `--draft` 指定なしでの MINOR/MAJOR バージョンバンプ

## オプション

- `--draft`: ドラフト PR として作成
- `--skip-review`: レビューをスキップ（緊急 hotfix 用）
- `--skip-test`: テストをスキップ（ドキュメントのみの変更用）
- `--base=branch`: ベースブランチを明示指定

## mille:kickoff Phase 6 連携

mille:kickoff から呼ばれた場合（`--from-mille` フラグ）、追加ステップを実行:

### Step 7: CI修正ループ

CI が失敗した場合、`/ci-autofix` スキルを自動呼び出し:
- エラー分類: lint → type → test → codegen の優先順
- 最大3回リトライ
- 3回失敗 → Issue コメント + `needs-human` ラベル

### Step 8: E2Eテスト

`requirements.md` の Critical Path（認証、決済、データ永続化）を自動検出し、
`e2e-runner` エージェントで Playwright テストを生成・実行。

### Step 9: マージ判定（複数PR集約）

全 TASK PR の状態を一括表示:
- ✅ 全CI グリーン + CRITICAL ゼロ + E2E PASS
- ⚠️ MEDIUM 指摘あり（ブロックしない）
- ❌ CRITICAL あり or CI 赤

`--auto-merge` 指定時は ✅⚠️ を確認なしでマージ。

## 関連コマンド

- `/code-review` — レビューのみ実行
- `/verify` — ビルド + 型 + lint + テスト検証
- `/canary` — デプロイ後の監視（ship の後に使用）
- `/ci-autofix` — CI修正ループ単体実行
- `/feedback` — Phase 8 フィードバックトリガー

## 関連スキル

- `ci-autofix` — CI失敗→自動修正ループ
- `issue-workflow` — Issue 管理（TASK PR と Issue の紐付け）

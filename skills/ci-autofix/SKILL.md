---
name: ci-autofix
description: CI失敗→自動修正ループ。エラー分類（lint/型/テスト/codegen）→修正→再実行を最大3回リトライ。3回失敗でIssueコメント+needs-humanラベル。mille:kickoff v2 Phase 6-3 で使用。
triggers:
  - "ci-autofix"
  - "CI修正"
  - "CI失敗"
  - "CI fix"
  - "ci autofix"
  - "自動修正ループ"
---

# CI Autofix スキル

CI 失敗を検知し、エラー分類→修正→再実行を最大3回リトライする自動修正ループ。
mille:kickoff v2 の Phase 6-3 で使用するほか、単体で `/ci-autofix` コマンドとしても呼び出し可能。

## コマンド体系

```
/ci-autofix                    → 直近の CI 失敗を自動修正
/ci-autofix {PR番号}           → 特定 PR の CI 失敗を修正
/ci-autofix --dry-run          → 修正内容を表示（適用しない）
```

## 関連エージェント

| エージェント | 役割 |
|---|---|
| `build-error-resolver` | ビルドエラーの解析・修正提案 |
| `debugger` | テスト失敗時の根本原因調査 |

## 関連スキル

| スキル | 連携内容 |
|---|---|
| `mille:auto-debug` | テスト失敗時に委譲 |
| `mille:build-fix` | ビルドエラー修正に委譲 |
| `mille:env-fix` | 環境依存問題の修正に委譲 |

---

## ワークフロー概要

```
CI失敗検知
  │
  ▼
┌─────────────────────────────────────────┐
│  ループ（最大3回）                        │
│                                         │
│  1. gh run view でエラーログ取得          │
│  2. エラー分類（lint → 型 → テスト → codegen）│
│  3. 分類に応じた修正実行                  │
│  4. fix commit + push                    │
│  5. CI 再実行待ち（gh run watch）         │
│     ├── 成功 → ループ終了、完了報告       │
│     └── 失敗 → 次のリトライへ            │
│         （別のエラー分類で再試行）         │
└─────────────────────────────────────────┘
  │
  ▼ 3回失敗
Issue コメント「自動修正に失敗。手動対応が必要」
+ label: needs-human
```

---

## Phase 1: CI 失敗検知とエラーログ取得

### 1.1 失敗した CI run の特定

```bash
# 引数なし: カレントブランチの直近の失敗 run
gh run list --branch "$(git branch --show-current)" --status failure --limit 1 --json databaseId,conclusion,name

# PR 番号指定: その PR の HEAD コミットの失敗 run
gh pr view {PR番号} --json headRefName --jq '.headRefName' \
  | xargs -I{} gh run list --branch {} --status failure --limit 1 --json databaseId,conclusion,name
```

### 1.2 エラーログの取得

```bash
# run のログをダウンロード
gh run view {RUN_ID} --log-failed 2>&1 | head -200

# ログが長い場合は ERROR/FAIL 行のみ抽出（context-hygiene パターン）
gh run view {RUN_ID} --log-failed 2>&1 | grep -iE "(error|fail|FAIL|Error)" | head -50
```

重要: ログ全体を読まない。ERROR/FAIL 行のみ抽出してコンテキストウィンドウを節約する。

---

## Phase 2: エラー分類

以下の優先順でエラーを分類する。複数種類のエラーが混在する場合、上位の分類から順に修正する。

### 分類優先順

| 優先度 | 分類 | 検出パターン | 修正方法 |
|--------|------|-------------|---------|
| 1 | **codegen** | `prisma generate`, `graphql-codegen`, `proto`, `openapi` | 生成コマンドを実行して commit |
| 2 | **lint** | `eslint`, `ruff`, `prettier`, `biome`, `stylelint` | auto-fix コマンドを実行 |
| 3 | **型エラー** | `tsc`, `mypy`, `pyright`, `TS2xxx`, `type-check` | エラー箇所を解析して修正 |
| 4 | **テスト失敗** | `vitest`, `jest`, `pytest`, `FAIL`, `AssertionError` | auto-debug に委譲 |

### 分類ロジック

```
入力: CI エラーログ（ERROR/FAIL 行のみ）

Step 1: codegen 判定
  - "prisma generate" / "graphql-codegen" / "protoc" / "openapi-generator" を含む
  - → 分類: codegen

Step 2: lint 判定
  - "eslint" / "ruff" / "prettier" / "biome" / "stylelint" / "formatting" を含む
  - → 分類: lint

Step 3: 型エラー判定
  - "TS2" / "TS7" / "TS18" / "error TS" / "mypy" / "pyright" / "type-check" を含む
  - → 分類: 型エラー

Step 4: テスト失敗判定
  - "FAIL" / "AssertionError" / "vitest" / "jest" / "pytest" / "test" を含む
  - → 分類: テスト失敗

Step 5: 分類不能
  - 上記いずれにも該当しない
  - → build-error-resolver エージェントに丸ごと委譲
```

---

## Phase 3: 分類別の修正実行

### 3.1 codegen 修正

コード生成が古い（スキーマ変更後に再生成していない）場合。

```bash
# Prisma
npx prisma generate

# GraphQL Codegen
npx graphql-codegen

# Protocol Buffers
buf generate

# OpenAPI
npx openapi-generator-cli generate -i openapi.yaml -g typescript-axios -o src/api/generated
```

修正後:
```bash
git add -A  # 生成ファイルは全てステージ
git commit -m "fix: regenerate codegen artifacts"
git push
```

### 3.2 lint 修正

auto-fix が効くエラーを自動修正する。

```bash
# プロジェクトの lint ツールを検出して実行
# package.json の scripts から lint:fix / format を探す

# ESLint
npx eslint --fix .

# Prettier
npx prettier --write .

# Ruff (Python)
ruff check --fix .
ruff format .

# Biome
npx biome check --write .
```

修正後:
```bash
git add -A
git commit -m "fix: auto-fix lint errors"
git push
```

### 3.3 型エラー修正

auto-fix が効かないため、エラー箇所を解析して手動修正する。

```
1. tsc --noEmit 2>&1 | head -30 でエラー一覧を取得
2. 各エラーのファイル:行番号を特定
3. 該当箇所を Read で確認
4. 型定義の追加/修正を Edit で適用
5. tsc --noEmit で修正確認
```

修正が複雑な場合は `mille:build-fix` スキルに委譲する。

修正後:
```bash
git add {修正したファイル}
git commit -m "fix: resolve type errors"
git push
```

### 3.4 テスト失敗修正

`mille:auto-debug` スキルに委譲する。auto-debug は以下を実行する:

```
1. 失敗テストの特定
2. エラーメッセージの解析
3. 原因の推定（実装バグ / テスト不備 / 環境問題）
4. 環境問題 → mille:env-fix に再委譲
5. 実装バグ → 修正パッチの適用
6. テスト不備 → テストの修正（ただし安易にテストを書き換えない）
```

修正後:
```bash
git add {修正したファイル}
git commit -m "fix: resolve test failures"
git push
```

### 3.5 分類不能エラー

build-error-resolver エージェントに全エラーログを渡して解析・修正を委譲する。

```
Task(build-error-resolver):
  "以下の CI エラーログを解析し、修正方法を特定して実装してください:
   {エラーログ}
   
   修正後、git commit + push してください。"
```

---

## Phase 4: CI 再実行と待機

### 4.1 push 後の CI 再実行待ち

```bash
# push したコミットの CI run を待機（最大10分）
gh run watch --exit-status 2>&1 | tail -5
```

`gh run watch` はブロッキングコマンドのため、tmux で実行するか、タイムアウト付きで呼ぶ。

### 4.2 結果判定

```
CI 成功 → 完了報告を出力してループ終了
CI 失敗 → リトライカウントをインクリメント
  - リトライ < 3 → Phase 1 に戻る（新しいエラーログで再分類）
  - リトライ = 3 → Phase 5（エスカレーション）へ
```

リトライ時の注意:
- 前回と同じエラーが出た場合、同じ修正を繰り返さない
- 前回の修正で新しいエラーが発生した場合、新しいエラーを優先して修正する
- 2回目以降は修正内容をより慎重に検討する（1回目の auto-fix が不十分だった可能性）

---

## Phase 5: エスカレーション（3回失敗時）

3回リトライしても CI が通らない場合、人間への引き継ぎを行う。

### 5.1 Issue コメント

```bash
# PR に紐づく場合
gh pr comment {PR番号} --body "$(cat <<'EOF'
## CI 自動修正失敗

3回のリトライで CI を修正できませんでした。手動対応が必要です。

### 試行した修正
| # | 分類 | 修正内容 | 結果 |
|---|------|---------|------|
| 1 | {分類1} | {修正内容1} | 失敗: {残存エラー概要} |
| 2 | {分類2} | {修正内容2} | 失敗: {残存エラー概要} |
| 3 | {分類3} | {修正内容3} | 失敗: {残存エラー概要} |

### 最終エラーログ（抜粋）
```
{最後の CI run のエラーログ上位20行}
```

### 推奨アクション
{エラーの傾向から推測される根本原因と、人間が確認すべきポイント}
EOF
)"
```

### 5.2 ラベル付与

```bash
# needs-human ラベルが存在しない場合は作成
gh label create needs-human --description "自動修正に失敗。手動対応が必要" --color "D93F0B" 2>/dev/null || true

# PR にラベル付与
gh pr edit {PR番号} --add-label needs-human

# Issue にラベル付与（TASK Issue がある場合）
gh issue edit {ISSUE番号} --add-label needs-human
```

---

## --dry-run モード

`--dry-run` 指定時は修正を適用せず、以下を出力する:

```markdown
## CI Autofix Dry Run

### CI Run: {RUN_ID}
### ブランチ: {ブランチ名}

### エラー分類結果
| 優先度 | 分類 | エラー数 | 修正方針 |
|--------|------|---------|---------|
| 1 | lint | 5件 | `npx eslint --fix .` で自動修正可能 |
| 2 | 型エラー | 2件 | `src/api/user.ts:45`, `src/lib/auth.ts:12` の型定義修正 |

### 修正対象ファイル（推定）
- `src/api/user.ts` — Property 'email' does not exist on type 'User'
- `src/lib/auth.ts` — Type 'string | undefined' is not assignable to type 'string'

### 修正を適用するには
`/ci-autofix` を --dry-run なしで実行してください。
```

---

## GitHub Actions テンプレート

プロジェクトが CI 自動修正を GitHub Actions ワークフローとして組み込む場合、以下のテンプレートをコピーして使用する。

### `.github/workflows/ci-autofix.yml`

```yaml
name: CI Autofix

on:
  workflow_run:
    workflows: ["CI"]  # メインCI ワークフロー名に合わせる
    types:
      - completed

permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  autofix:
    if: >
      github.event.workflow_run.conclusion == 'failure' &&
      github.event.workflow_run.head_branch != 'main'
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_branch }}
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Attempt autofix (lint)
        id: lint-fix
        continue-on-error: true
        run: |
          npx eslint --fix . 2>/dev/null || true
          npx prettier --write . 2>/dev/null || true
          if git diff --quiet; then
            echo "fixed=false" >> "$GITHUB_OUTPUT"
          else
            echo "fixed=true" >> "$GITHUB_OUTPUT"
          fi

      - name: Attempt autofix (codegen)
        id: codegen-fix
        continue-on-error: true
        run: |
          npx prisma generate 2>/dev/null || true
          if git diff --quiet; then
            echo "fixed=false" >> "$GITHUB_OUTPUT"
          else
            echo "fixed=true" >> "$GITHUB_OUTPUT"
          fi

      - name: Commit and push fixes
        if: steps.lint-fix.outputs.fixed == 'true' || steps.codegen-fix.outputs.fixed == 'true'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add -A
          git commit -m "fix: auto-fix CI errors (lint/codegen)"
          git push

      - name: Label PR on persistent failure
        if: failure()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PR_NUMBER=$(gh pr list \
            --head "${{ github.event.workflow_run.head_branch }}" \
            --json number --jq '.[0].number')
          if [ -n "$PR_NUMBER" ]; then
            gh label create needs-human \
              --description "自動修正に失敗。手動対応が必要" \
              --color "D93F0B" 2>/dev/null || true
            gh pr edit "$PR_NUMBER" --add-label needs-human
            gh pr comment "$PR_NUMBER" --body \
              "CI 自動修正を試みましたが、修正できないエラーが残っています。手動対応が必要です。"
          fi
```

### テンプレートの使い方

1. `.github/workflows/ci-autofix.yml` にコピー
2. `workflows: ["CI"]` をプロジェクトのメイン CI ワークフロー名に変更
3. codegen ステップをプロジェクトの生成ツールに合わせて調整
4. 必要に応じて Python (ruff) や Go (gofmt) のステップを追加

---

## mille:kickoff v2 Phase 6-3 との統合

Phase 6 から呼び出される場合の追加動作:

```
Phase 6-3 からの呼び出し:
  入力: PR 番号のリスト（Wave 内の全 PR）
  
  各 PR に対して:
    1. CI ステータスを確認（gh pr checks {PR番号}）
    2. 失敗している PR のみ ci-autofix ループを実行
    3. 結果を集約して Phase 6-4（マージ判定）に返す
  
  返却値:
    - fixed: CI が通った PR 番号リスト
    - failed: 自動修正できなかった PR 番号リスト + エラー概要
```

---

## 安全策

### 修正しない範囲

以下のケースでは自動修正を試みず、即座にエスカレーションする:

- **セキュリティ関連のテスト失敗** — 認証・認可テストの失敗は意図的なブロックの可能性がある
- **データベースマイグレーションエラー** — スキーマ変更は人間の判断が必要
- **環境変数・シークレットの欠落** — CI 環境の secrets 設定は人間が行う
- **依存関係のバージョンコンフリクト** — lockfile のコンフリクトは意図を確認してから解決する

### コミット制約

- 各リトライのコミットは1つまで（修正を小さく保つ）
- force push は行わない
- main/master ブランチへの直接コミットは行わない
- 修正コミットには必ず `fix:` プレフィックスを付ける

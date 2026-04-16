---
name: kickoff
description: mille v1.0 — 4モード対応 Issue 駆動開発パイプライン。greenfield（新規）/ feature（機能追加）/ fix（バグ修正）/ refine（小変更）を自動検出し、適切なパイプラインを実行。
---

# Mille Kickoff v1.0（4モード Issue 駆動パイプライン）

開発の種類を自動検出し、最適なパイプラインで実行する統合エントリポイント。

**4つのモード:**
- **greenfield**: 新規開発（Phase 0-8 フルパイプライン）
- **feature**: 既存コードへの機能追加（dev-context → dev-plan → 実装 → QA）
- **fix**: バグ修正（調査 → 修正 → レビュー）
- **refine**: 小規模変更（refine-plan → refine-execute → レビュー）

**設計思想**: 設計は人間が判断、実装以降は自動化。
**全コンテキストは GitHub Issue ツリーに集約**（Single Source of Truth）。

## 使用方法

```
/mille:kickoff <名前> [PRDファイルパス] [options]
```

## オプション

- `<名前>`: 要件・機能・修正の名前（必須）
- `[PRDファイルパス]`: PRDファイルがある場合そのパス

### モード指定
- `--mode greenfield`: 新規開発（デフォルト: 自動検出）
- `--mode feature`: 既存コードへの機能追加
- `--mode fix`: バグ修正・調査
- `--mode refine`: 小規模変更

### greenfield モード専用
- `--skip-impl`: Phase 5-7（実装〜ポストマージ）をスキップ
- `--skip-bootstrap`: Phase 4（環境構築）をスキップ
- `--skip-research`: Phase 0.5（リサーチ）をスキップ
- `--with-constitution`: Phase 1.5 で Constitution（プロジェクト原則）を生成
- `--light`: 各フェーズで「軽量」規模を自動選択
- `--auto-merge`: Phase 6-4 のマージ確認ゲートを省略（条件充足PRを自動マージ）
- `--parallel N`: Phase 5 の並列数を指定（デフォルト: 3）
- `--sequential`: Phase 5 を直列実行（従来の kairo-loop 互換）

### feature / fix / refine モード共通
- `--issue <番号>`: 既存 GitHub Issue にリンク
- `--plan-only`: 計画段階で停止

## 引数バリデーション

実行開始時に検証:
1. `$ARGUMENTS` に要件名が含まれていること
2. PRDファイルパスが指定された場合、Read ツールで読み取り可能であること
3. 要件名がない場合はエラー表示して終了:
   ```
   エラー: 要件名を指定してください
   使用法: /mille:kickoff <要件名> [PRDファイルパス] [--skip-impl] [--light]
   例: /mille:kickoff ユーザー認証システム docs/prd/auth.md
   ```

---

## Phase -1: モード選択（全モード共通）

Phase 0 の前に実行。`--mode` が明示指定されていない場合、自動検出 + ユーザー確認を行う。

### -1-1. 自動検出ロジック

```
1. docs/spec/{名前}/ が存在 → 既存の仕様あり
2. src/ 等のソースディレクトリが存在 → 既存コードベース
3. docs/dev/context.md が存在 → dev-context 実行済み
4. --issue 指定あり → fix or feature を推奨
5. 上記いずれもなし → greenfield を推奨
```

### -1-2. モード選択ゲート

`--mode` 未指定時、AskUserQuestion で確認:

```
プロジェクト状態を検出しました:
- ソースコード: {あり（{ファイル数}件）/ なし}
- 既存仕様: {あり（docs/spec/{名前}/）/ なし}
- docs/dev/context.md: {あり / なし}

推奨モード: {greenfield / feature / fix / refine}
理由: {根拠}

1. greenfield — 新規開発（Phase 0-8 フルパイプライン）
2. feature — 機能追加（dev-context → dev-plan → 実装 → QA）
3. fix — バグ修正（調査 → 修正 → レビュー）
4. refine — 小変更（refine-plan → refine-execute → レビュー）
```

### -1-3. モードルーティング

| 選択モード | 遷移先 |
|-----------|--------|
| greenfield | Phase 0 へ（以降は従来の Phase 0-8 パイプライン） |
| feature | Phase F0 へ（feature モードセクション） |
| fix | Phase X0 へ（fix モードセクション） |
| refine | Phase R0 へ（refine モードセクション） |

---

## コンテキスト管理戦略（greenfield モード）

### 原則

- **Phase 0**: メインコンテキストで技術スタック確認
- **Phase 1-3**: 各フェーズを mille コマンドとして Task ツールで実行
- **Phase 4**: Project Bootstrap で開発環境構築（実装前に必要）
- **Phase 5**: kairo-loop で自動実装（最もコンテキストを消費）
- 各フェーズ完了時に **HANDOFF 要約** を生成し、メインには要約のみ保持

### HANDOFF フォーマット

```markdown
## HANDOFF: Phase N → Phase N+1

- 成果物: {ファイルパス}（{概要}）
- 信頼性分布: 🔵{N} 🟡{N} 🔴{N}
- 次フェーズへの注意点: {1-2 項目}
```

---

## Phase 0: 技術スタック確認（メイン）

### 0-1. 前提条件の確認

1. `CLAUDE.md` を確認 — 技術スタックが記載されているか
2. `docs/tech-stack.md` を確認 — 存在するか

### 0-2. 技術スタック未定義の場合

tech-stack が見つからない場合、mille の init-tech-stack を実行:

```
Task ツールで general-purpose サブエージェントを起動:
  /mille:init-tech-stack
```

### 0-3. PRDファイルの事前確認

PRDファイルが指定されている場合:
1. Read ツールで読み込み可能か確認
2. 内容の概要をユーザーに表示

### HANDOFF

```
Phase 0 完了: 技術スタック確認
- tech-stack: {CLAUDE.md / docs/tech-stack.md / 新規作成}
- PRD: {あり（パス） / なし}
- 要件名: {要件名}
```

---

## Phase 0.5: リサーチ（サブエージェント並列、オプション）

`--skip-research` 指定時、または PRD の情報が十分な場合はスキップ。

PRDファイルまたは要件概要を分析し、不足情報を research-analyst で自動調査する。
research-analyst による不足情報の自動調査。

### 0.5-1. リサーチ要否の判定

PRDファイルがある場合、以下を評価:

| カテゴリ | 充足度 | 不足時のリサーチ例 |
|---------|-------|-----------------|
| ビジネスゴール | ◎/○/△/✕ | 類似サービスの事業モデル調査 |
| ユーザーストーリー | ◎/○/△/✕ | ペルソナ・ユースケース深掘り |
| 機能要件 | ◎/○/△/✕ | 類似製品の機能一覧調査 |
| 非機能要件 | ◎/○/△/✕ | 業界標準のNFR値調査 |
| 技術制約 | ◎/○/△/✕ | 技術選択肢の比較調査 |

**全カテゴリが ◎/○ の場合**: リサーチをスキップし Phase 1 へ進む。
**△/✕ があ場合**: 該当カテゴリのリサーチを実行。

### 0.5-2. 並列リサーチ実行

Task ツールで `research-analyst` を **並列で** 起動（最大3並列）。
各エージェントに不足カテゴリのリサーチトピックを割り当てる。

### 0.5-3. リサーチ結果の保存

リサーチ結果を `docs/spec/{要件名}/research-notes.md` に保存。
Phase 1（kairo-requirements）実行時に、このファイルを追加コンテキストとして参照させる。

### HANDOFF

```
Phase 0.5 完了: リサーチ
- リサーチ実施: {あり（{N}トピック） / スキップ}
- 保存先: docs/spec/{要件名}/research-notes.md
- 充足度改善: {カテゴリ: △→○} 等
```

---

## Phase 1: 要件定義（サブエージェント）

Task ツールで `general-purpose` サブエージェントを起動し、mille の kairo-requirements を実行。

### サブエージェントへの指示

```
/mille:kairo-requirements {要件名} {PRDファイルパス}
```

- `--light` 指定時: 作業規模の質問で「軽量開発」を自動選択
- PRDファイルがある場合: パスを引数に含める
- AskUserQuestion による段階的ヒアリングが実行される（ユーザー対話あり）

### 生成される成果物

```
docs/spec/{要件名}/
├── requirements.md          # EARS記法の要件定義書
├── interview-record.md      # ヒアリング記録
├── note.md                  # コンテキストノート
├── user-stories.md          # ユーザーストーリー（フル時）
└── acceptance-criteria.md   # 受け入れ基準（フル時）
```

### 自動レビュー（サブエージェント）

確認ゲートの前に、成果物の品質チェックを自動実行する。
Task ツールで `qa-expert` サブエージェントを起動し、以下を検証:

1. **PRD/リサーチ網羅性**: PRD またはリサーチ結果の各項目が requirements.md に反映されているか。未反映項目をリストアップ
2. **🔴項目の妥当性**: 🔴（不明）項目に対してユーザーへの質問が漏れていないか
3. **EARS記法品質**: 曖昧な表現（「適切に」「速やかに」等）、テスト不可能な要件がないか
4. **受け入れ基準の検証可能性**: 各要件に対して検証可能な受け入れ基準が定義されているか

レビュー結果を `docs/spec/{要件名}/review-phase1.md` に保存。

### 確認ゲート（メインで実行）

AskUserQuestion:
```
Phase 1 完了: 要件定義書を生成しました。

## 成果物
- docs/spec/{要件名}/requirements.md
- 信頼性分布: 🔵{N} 🟡{N} 🔴{N}

## 自動レビュー結果
- PRD網羅性: {N}/{M} 項目反映済み {未反映があれば: （未反映: {項目名}）}
- 🔴未解決項目: {N}件 {あれば: （{項目名}）}
- EARS品質: {問題なし / {N}件の曖昧表現検出}
- 受け入れ基準: {全件検証可能 / {N}件が検証困難}

## 推奨アクション
{レビューで検出された問題があれば具体的な修正提案を記載}

1. 続行する（技術設計へ）
2. 指摘事項を修正してから続行
3. 中止する
```

---

## Phase 1.5: Constitution 生成（オプション）

`--with-constitution` 指定時のみ実行。プロジェクト原則を明文化し、以降のフェーズで参照する。
プロジェクト原則を明文化し、以降のフェーズで参照する。

### 1.5-1. Constitution の生成

Task ツールで `general-purpose` サブエージェントを起動。

Phase 1 の成果物（requirements.md, note.md）から以下を抽出し Constitution を生成:

| requirements.md のセクション | Constitution のセクション |
|---------------------------|------------------------|
| 概要・背景 | Core Principles |
| NFR | Quality Standards, Performance |
| ユーザーストーリー | User Experience Principles |
| 受け入れ基準 | Acceptance Criteria Guidelines |
| コンテキストノート（note.md） | Domain Glossary |

### 1.5-2. 既存 Constitution とのマージ

`.specify/memory/constitution.md` が既に存在する場合:
- 既存セクションは保持
- 新規情報を対応セクションに追加
- 矛盾する項目は追加しない

保存先: `.specify/memory/constitution.md`

### HANDOFF

```
Phase 1.5 完了: Constitution
- 状態: {生成/マージ/スキップ}
- 保存先: .specify/memory/constitution.md
- Core Principles: {N}項目、Glossary: {N}用語
```

---

## Phase 2: 技術設計（サブエージェント）

Task ツールで `general-purpose` サブエージェントを起動し、mille の kairo-design を実行。

### サブエージェントへの指示

```
/mille:kairo-design {要件名}
```

- Phase 1 の成果物（requirements.md 等）を自動参照
- `--light` 指定時: 作業規模の質問で「軽量設計」を自動選択
- 信頼性レベル（🔵🟡🔴）付きで設計文書を生成

### 生成される成果物

```
docs/design/{要件名}/
├── architecture.md      # アーキテクチャ設計
├── dataflow.md          # データフロー図
├── design-interview.md  # 設計ヒアリング記録
├── interfaces.ts        # TypeScript型定義
├── database-schema.sql  # DBスキーマ
└── api-endpoints.md     # API仕様
```

### 自動レビュー + トレーサビリティ検証（サブエージェント）

確認ゲートの前に、設計品質と要件→設計のトレーサビリティを自動検証する。
Task ツールで `architect` サブエージェントを起動し、以下を検証:

1. **FR トレーサビリティ**: requirements.md の各 FR（機能要件）に対応する設計要素（コンポーネント、API、DB テーブル）があるか。未対応 FR をリストアップ
2. **NFR 反映度**: requirements.md の NFR（非機能要件）が設計に反映されているか（例: パフォーマンス要件→キャッシュ設計、セキュリティ要件→認証設計）
3. **設計判断の根拠**: architecture.md で技術選択の理由が明記されているか
4. **インターフェース整合性**: interfaces.ts と api-endpoints.md の型定義・エンドポイント定義が矛盾していないか
5. **Shadow Path 検証**: dataflow.md の各データフローに対し、以下の4パスがカバーされているか検証。未カバーのパスを `[GAP]` としてフラグ:
   - **Happy path**: データが正常に流れるケース
   - **Nil path**: 入力が null/undefined のケース
   - **Empty path**: 入力が空（空文字、空配列、0件）のケース
   - **Error path**: 上流が失敗するケース
6. **Error & Rescue Registry**: 設計から特定できるエラーパターンをテーブル化:

   | メソッド/エンドポイント | 例外/エラー | レスキューアクション | ユーザー影響 |
   |----------------------|-----------|-------------------|-----------|

   テスト未定義かつサイレント失敗するエラーパターンは `[CRITICAL GAP]` としてフラグ。

レビュー結果を `docs/design/{要件名}/review-phase2.md` に保存。

### Dual-Voice レビュー（オプション）

architect レビュー後、`/codex` スキルが利用可能な場合はセカンドオピニオンを取得:

1. architect のレビュー結果（review-phase2.md）を要約
2. `/codex` に設計文書を渡し、独立した視点でレビュー依頼
3. 両者の見解を比較:
   - **一致**: そのまま採用（review-phase2.md に「Dual-Voice 合意」と記録）
   - **不一致**: `[TASTE DECISION]` として確認ゲートでユーザーに提示
4. `/codex` が利用不可の場合はスキップ（architect 単体レビューで続行）

### 確認ゲート（メインで実行）

AskUserQuestion:
```
Phase 2 完了: 技術設計文書を生成しました。

## 成果物
- docs/design/{要件名}/
- 信頼性分布: 🔵{N} 🟡{N} 🔴{N}

## 自動レビュー結果
- FR カバレッジ: {N}/{M} FR に設計要素あり {未対応があれば: （未対応: {FR-ID}）}
- NFR 反映: {N}/{M} {未反映があれば: （未反映: {NFR名}）}
- 設計判断根拠: {全件記載 / {N}件が根拠なし}
- インターフェース整合性: {問題なし / {N}件の不整合}
- Shadow Path: {N}/{M} データフローで4パスカバー済み {GAP があれば: （[GAP]: {フロー名} の {パス種別}）}
- Error & Rescue: {N}件のエラーパターン特定 {CRITICAL GAP があれば: （[CRITICAL GAP]: {N}件）}
{Dual-Voice 実施時:}
- Dual-Voice: {一致{N}件 / 不一致{N}件}
{不一致があれば:}
  - [TASTE DECISION] {不一致の概要}

## 推奨アクション
{レビューで検出された問題があれば具体的な修正提案を記載}

1. 続行する（タスク分割へ）
2. 指摘事項を修正してから続行
3. 中止する
```

---

## Phase 3: タスク分割（サブエージェント）

Task ツールで `general-purpose` サブエージェントを起動し、mille の kairo-tasks を実行。

### サブエージェントへの指示

```
/mille:kairo-tasks {要件名} --task
```

- `--task` オプションで Claude Code Tasks に自動登録
- Phase 1-2 の成果物を自動参照
- 1日単位の粒度でタスク分割
- TASK-0001〜 の個別ファイルを生成

### 生成される成果物

```
docs/tasks/{要件名}/
├── overview.md      # タスク概要・依存関係
├── TASK-0001.md     # 個別タスク
├── TASK-0002.md
└── ...
```

### 累積整合性チェック（サブエージェント）

**実装前の最終関門**。Phase 1〜3 の全成果物を横断的に検証し、要件→設計→タスクの一貫性を担保する。
Task ツールで `qa-expert` サブエージェントを起動し、以下を検証:

1. **設計→タスク網羅性**: design/ の各コンポーネント・API・DB テーブルに対応するタスクがあるか。未対応設計要素をリストアップ
2. **タスク依存関係の妥当性**: タスク間の依存関係が設計の実装順序と整合しているか（例: DB スキーマ作成タスクが API 実装タスクより先か）
3. **要件→設計→タスクのトレーサビリティ**: requirements.md の各 FR が設計を経由してタスクに到達しているか。途中で消失した要件をリストアップ
4. **信頼性レベルの伝播**: 🔴（不明）が残っている要件に依存するタスクの識別。実装時にブロックされるリスクの洗い出し
5. **見積もり妥当性**: タスクの粒度が1日単位として適切か（過大・過小なタスクの検出）

レビュー結果を `docs/tasks/{要件名}/review-integrity.md` に保存。

### 確認ゲート（メインで実行）

AskUserQuestion:
```
Phase 3 完了: タスク分割が完了しました。

## 成果物
- docs/tasks/{要件名}/
- タスク数: {N}件（TASK-0001 〜 TASK-{NNNN}）
- Claude Code Tasks に登録済み

## 累積整合性チェック結果
- 設計→タスク網羅性: {N}/{M} コンポーネント対応済み {未対応があれば: （未対応: {要素名}）}
- 依存関係: {問題なし / {N}件の順序不整合}
- 要件トレーサビリティ: {N}/{M} FR がタスクに到達 {消失があれば: （消失: {FR-ID}）}
- 🔴依存タスク: {N}件（実装時にブロックリスク）
- 粒度: {適切 / {N}件が要調整（{過大/過小}）}

## Wave 構成（並列実行グループ）
- Wave 1: {N}件（{TASK-ID一覧}）— 依存なし
- Wave 2: {N}件（{TASK-ID一覧}）— Wave 1 に依存
- ...

## GitHub Issue 起票
{リポジトリあり:}
- Epic Issue: #{番号} "{要件名}"
- TASK Issue: {N}件起票済み（wave-1: {N}件, wave-2: {N}件, ...）
- Milestone: 仕様策定(完了), 実装, 品質保証, フィードバック
{リポジトリなし:}
- GitHub リポジトリ未検出: Issue 起票スキップ（ローカルファイルで続行）

## 推奨アクション
{レビューで検出された問題が��れば具体的な修正提案を記載}

1. 続行する（開発環境構築へ）
2. 指摘事項を修正してから続行
3. Issue のみ確認して修正後に続行
4. タスク分割まで完了（環境構築・実装は後で手動実行）
5. 中止する
```

`--skip-impl` かつ `--skip-bootstrap` 指定時、または選択肢4の場合: 完了サマリーへ進む。

### 3-4. Issue 自動起票（GitHub Issue ツリー構築）

累積整合性チェック通過後、GitHub リポジトリが存在する場合に自動実行。
`/issue-workflow create {要件名}` スキルを内部呼び出し。

#### Epic Issue 作成

```bash
gh issue create \
  --title "{要件名}" \
  --label "epic,mille" \
  --body "$(cat <<'EOF'
## {要件名}

### 概要
{requirements.md の概要セクション}

### 信頼性分布
- 🔵 確定: {N}件 / 🟡 推測: {N}件 / 🔴 不明: {N}件

### Phase 進捗
- [x] Phase 1: 要件定義
- [x] Phase 2: 技術設計
- [x] Phase 3: タスク分割
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
EOF
)"
```

#### Wave 算出

TASK ファイル群の依存関係グラフから並列実行可能なグループを自動算出:

```
1. 依存関係のないタスクを Wave 1 に割り当て
2. Wave 1 のみに依存するタスクを Wave 2 に割り当て
3. 再帰的に繰り返し
4. 循環依存がある場合はエラー報告
```

#### TASK Issue 一括起票

```bash
# 各 TASK-NNNN.md に対して:
gh issue create \
  --title "TASK-{NNNN}: {タスク名}" \
  --label "task,wave-{N}" \
  --milestone "実装" \
  --body "$(cat <<'EOF'
## TASK-{NNNN}: {タスク名}

### タスクファイル
docs/tasks/{要件名}/TASK-{NNNN}.md

### 対応する設計要素
- FR: {FR-ID} / コンポーネント: {名前} / API: {エンドポイント}

### 受け入れ基準
{TASK-NNNN.md から抽出}

### 依存関係
- blocked by: #{前提Issue番号}
- blocks: #{後続Issue番号}

### 信頼性レベル
🔵{N} 🟡{N} 🔴{N}

### Wave
Wave {N}（並列実行グループ）
EOF
)"
```

#### Milestone 作成

```bash
# 4つの Milestone を作成
gh api repos/{owner}/{repo}/milestones -f title="仕様策定" -f description="Phase 1-3"
gh api repos/{owner}/{repo}/milestones -f title="実装" -f description="Phase 5 TDD"
gh api repos/{owner}/{repo}/milestones -f title="品質保証" -f description="Phase 6"
gh api repos/{owner}/{repo}/milestones -f title="フィードバック" -f description="Phase 8"
```

**GitHub リポジトリがない場合**: Issue 起票をスキップし、ローカルファイルのみで続行。

---

## Phase 4: Project Bootstrap（サブエージェント）

**実装前に開発環境を構築する。** CLAUDE.md、コマンド、フック、ルール、ドメインスキルを先に整えることで、Phase 5 の実装時にプロジェクト固有の規約・ツール設定が適用される。

`--skip-bootstrap` 指定時はスキップし、Phase 5 へ進む。

### 4-1. /init による基本セットアップ

AskUserQuestion:
```
Phase 4: 開発環境構築
実装前にプロジェクトの開発環境を構築します。

Step 1: CLAUDE.md がまだない場合、公式の /init で基本セットアップを行いますか？
（/init は対話的に CLAUDE.md の骨格・基本フック・スキル設定をガイドします）

1. /init を実行してから続行する（推奨）
2. /init をスキップして一括生成する
3. 環境構築自体をスキップする（後で /project-bootstrap を手動実行）
```

**「/init を実行する」の場合**:
- ユーザーに `! /init` を実行するよう案内（対話的コマンドのため）
- `/init` 完了後、4-2 へ進む

**「スキップ」の場合**: Phase 5 へ進む（`--skip-impl` 指定時は完了サマリーへ）。

### 4-2. mille 成果物からの情報マッピング

Task ツールで `general-purpose` サブエージェントを起動。

mille の成果物を `/project-bootstrap` の入力として再利用:

| bootstrap が通常聞く項目 | mille 成果物からの回答 |
|-------------------------|----------------------|
| プロジェクトのドメイン・目的 | docs/spec/{要件名}/requirements.md の概要・背景 |
| 技術スタック | CLAUDE.md / docs/tech-stack.md（Phase 0 で確認済み） |
| アーキテクチャ | docs/design/{要件名}/architecture.md |
| データモデル | docs/design/{要件名}/database-schema.sql |
| API仕様 | docs/design/{要件名}/api-endpoints.md |
| ドメイン用語 | docs/spec/{要件名}/note.md のコンテキスト情報 |
| 非機能要件 | docs/spec/{要件名}/requirements.md の NFR セクション |

**mille で未収集の項目**（以下のみ AskUserQuestion で補完）:
- 開発フロー（GitHub Flow / Git Flow / Trunk-based）
- デプロイ先（Vercel / AWS / GCP 等）
- ブランチ戦略・レビュープロセス

### 4-3. 成果物の生成

`/project-bootstrap` の Phase 3 と同じパターンで生成。
**`/init` で生成済みの CLAUDE.md がある場合は不足セクションの追加のみ**:

- **CLAUDE.md**（補完 or 新規作成）
  - mille の信頼性レベル（🔵🟡🔴）分布をサマリーとして含める
  - docs/spec/, docs/design/, docs/tasks/ の成果物配置を記載
- **`.claude/commands/`** - dev, build, test, lint, deploy, review, feature-workflow（検出ツールのみ）
- **`.claude/agents/`** - domain-expert, project-reviewer
- **`.claude/settings.json`** - PostToolUse フック（lint, type-check）
- **`.claude/rules/`** - project-conventions, architecture, domain-glossary, workflow

### 4-4. ドメインスキル自動抽出

mille 成果物から `/project-bootstrap` Phase 3f のドメインスキル自動抽出ロジックを実行。

抽出対象:
- `docs/spec/{要件名}/requirements.md` の FR → 状態遷移、認証、決済、バリデーションスキル
- `docs/design/{要件名}/architecture.md` → API パターン、外部連携スキル
- `docs/design/{要件名}/api-endpoints.md` → API リソースパターンスキル
- `docs/design/{要件名}/database-schema.sql` → データ変換スキル
- `docs/spec/{要件名}/requirements.md` の NFR → セキュリティ、キャッシュスキル

ユーザー確認ゲートを経て `.claude/skills/` にドメインスキルを生成（最大10件）。

### 4-5. mille 固有の追加設定

通常の `/project-bootstrap` に加えて以下を生成:

- **`.claude/commands/kairo.md`**（プロジェクト用ショートカット）:
  ```
  /mille:kairo-loop {要件名} {次のTASK-ID} {最終TASK-ID}
  ```
- **`.claude/rules/mille-workflow.md`**:
  - mille の成果物ディレクトリ構成（docs/spec/, docs/design/, docs/tasks/）
  - 信頼性レベルの意味と扱い方
  - kairo-loop 実行時の Auto Mode 推奨設定

**既存ファイルは上書きしない**（マージまたはスキップ）。

### HANDOFF

```
Phase 4 完了: Project Bootstrap
- /init: {実行済み/スキップ}
- CLAUDE.md: {補完/新規作成}
- コマンド: {N}件、エージェント: {N}件、フック: {N}件、ルール: {N}件
- ドメインスキル: {N}件（{スキル名一覧}）
- mille 固有: kairo ショートカット、mille-workflow ルール
```

---

## Phase 5: 並列TDD実装（自動）

`--skip-impl` 指定時はスキップし、完了サマリーへ進む。

### 5-1. 実装モードの選択

AskUserQuestion:
```
TDD実装ループを開始します。
Phase 4 で構築した開発環境（CLAUDE.md、フック、ルール）が適用されます。

Wave 構成: {Wave数}グループ（{タスク総数}件）
- Wave 1: {N}件（並列実行可能）
- Wave 2: {N}件（Wave 1 完了後）
- ...

1. 並列実行（推奨）— Wave 内を {並列数} 並列で実行
2. 直列実行 — 従来の kairo-loop 互換
3. 範囲指定 — 特定の TASK-ID 範囲を指定
4. スキップ（実装は後で手動実行）
```

### 5-2. 並列実装（デフォルト）

Wave ごとに Agent worktree で並列実行:

```
Wave {N} 実行:
  ├── Agent worktree 1 → TASK-{NNNN}
  │   ├── git checkout -b task/{NNNN}
  │   ├── /mille:kairo-implement TASK-{NNNN}
  │   ├── git push -u origin task/{NNNN}
  │   └── gh pr create --body "Closes #{TASK Issue番号}"
  │
  ├── Agent worktree 2 → TASK-{NNNN}
  │   └── （同上）
  │
  └── Agent worktree 3 → TASK-{NNNN}
      └── （同上）

  → 全 Worker 完了を待機
  → WTF-likelihood チェック（Wave 単位）
  → 次の Wave へ
```

**並列数**: デフォルト 3。`--parallel N` で変更可能。
**直列実行**: `--sequential` 指定時は従来の `kairo-loop` を使用。

### 5-2.5. WTF-likelihood 自己制御（セーフティネット）

修正の累積リスクを Wave 単位で監視し暴走を防止する:

**リスクスコア計算**（Wave 完了ごとに評価）:
- revert 発生: +15%
- コンポーネントファイル（共通モジュール）変更: +5%/件
- 修正数 N: 基礎リスク = N × 0.5%

**停止条件**:
- リスク 20% 超: 一時停止し AskUserQuestion でユーザーに相談
- ハードキャップ: 30修正/セッション（超過時は強制停止）

### 5-3. コンテキスト管理

- Auto Mode を有効化するようユーザーに案内: `Shift+Tab で Auto Mode ON`
- context-hygiene のテスト出力最小化ルールを適用
- 必要に応じて `/compact` を提案

### 5-3.5. Scope Drift Detection（実装完了時）

全 Wave 完了後、要件外の変更が紛れ込んでいないかを検出:

1. 各 TASK ブランチの `git diff --name-only` を集約
2. `docs/tasks/{要件名}/overview.md` のタスク一覧と照合
3. タスクに関連しないファイル変更を `[SCOPE DRIFT]` としてフラグ

**情報提供のみ**（ブロックはしない）。

### 5-4. 受け入れ基準充足度チェック（サブエージェント）

全 Wave 完了後、実装が要件を満たしているかを最終検証する。
Task ツールで `qa-expert` サブエージェントを起動し検証:

1. **受け入れ基準の充足**: requirements.md の受け入れ基準に対応するテストが PASS しているか
2. **FR 実装完了度**: 各 FR に対応するコードが存在するか
3. **🟡🔴 項目の解決状況**: 実装を通じて解決されたか
4. **テストカバレッジ**: 80% 以上か
5. **テスト品質ティア**:
   - ★★★ エッジケースまでテスト済み
   - ★★ ハッピーパスのみ
   - ★ スモークテストのみ
   - ☆ テストなし
   Critical path で ★★ 以下を `[COVERAGE GAP]` としてフラグ

レビュー結果を `docs/implements/{要件名}/review-acceptance.md` に保存。

### 5-5. Epic Issue 更新

```bash
# Epic Issue の Phase 5 チェックボックスを更新
gh issue edit #{Epic番号} --body "..."  # Phase 5 を [x] に
```

Phase 5 完了後、**確認ゲートなし**で Phase 6 に自動進行。

---

## Phase 6: 自動品質保証パイプライン

Phase 5 の各 TASK PR に対して自動実行。確認ゲートはマージ判定（6-4）のみ。

### 6-1. 自動レビュー（並列）

各 TASK PR に対して並列実行:
- `code-reviewer` エージェント: 差分サイズ連動深度（Light <50行 / Standard 50-199行 / Deep 200行以上）
- `security-reviewer` エージェント: Confidence Gate 8/10

結果の分類:
- **AUTO-FIX**: 機械的な修正（フォーマット、import順序等）→ 自動コミット
- **CRITICAL**: ブロッキング → PR にコメント + Issue にフラグ
- **その他**: PR コメントに記録（ブロックしない）

### 6-2. E2Eテスト

requirements.md の Critical Path を自動検出:
- 認証フロー、決済フロー、データ永続化フロー

`e2e-runner` エージェントで Playwright テストを生成・実行。
失敗 → CI修正ループ（6-3）に委譲。

### 6-3. CI修正ループ（PR 単位で自動実行）

`/ci-autofix` スキルを内部呼び出し。

トリガー: CI 失敗
ループ（最大3回）:
1. エラー分類: lint → type → test → codegen の優先順
2. `auto-debug` / `build-fix` / `env-fix` で原因特定・修正
3. fix commit + push
4. CI 再実行待ち

3回失敗 → Issue コメント「自動修正に失敗。手動対応が必要」+ label: `needs-human`

### 6-4. マージ判定（確認ゲート）

全 PR の状態を集約:

AskUserQuestion:
```
Phase 6 完了: 品質保証パイプラインが完了しました。

## PR ステータス
{各PRの状態を一覧表示}
- ✅ #{番号} TASK-{NNNN}: 全CI グリーン、CRITICAL ゼロ
- ⚠️ #{番号} TASK-{NNNN}: MEDIUM 指摘 {N}件（ブロックなし）
- ❌ #{番号} TASK-{NNNN}: CI赤 / CRITICAL あり

## レビューサマリー
- AUTO-FIX 適用: {N}件
- CRITICAL: {N}件（{あれば: 対象PR一覧}）
- E2E: {PASS / FAIL（{失敗テスト名}）}
- CI修正ループ: {N}回発動（{成功/失敗}）

1. ✅⚠️ を一括マージ（推奨）
2. ✅ のみマージ
3. 個別に確認
4. 全て保留
```

`--auto-merge` 指定時: 確認ゲートをスキップ。✅ の PR を自動マージ、⚠️ も PR コメント記録してマージ、❌ はブロック。

---

## Phase 7: ポストマージ（完全自動）

確認ゲートなし。全て自動実行。

### 7-1. ドキュメント自動更新

`doc-updater` エージェント:
- CLAUDE.md: 新規コンポーネント・APIの追記
- README.md: 機能一覧の更新
- CHANGELOG.md: マージされた PR から自動生成
→ コミット + プッシュ

### 7-2. Epic Issue 更新 + クローズ

- 全子 Issue のステータスを確認
- 全完了 → Epic Issue に完了サマリーを記載してクローズ
- 未完了あり → 残タスクを Epic body に明示

### 7-3. リリースノート下書き

```bash
gh release create --draft --title "v{バージョン}: {要件名}" --notes "..."
```

全 TASK Issue の変更を集約: 新機能（feat）、バグ修正（fix）、破壊的変更（breaking）

### 7-4. メトリクス記録（/retro 連携）

- コミット数、変更行数、テストカバレッジ
- 信頼性レベル解決率（🔴→🔵 変換数）
- Wave 並列効率（実時間 / 直列見積もり時間）
- CI修正ループ発動回数
→ Obsidian Daily Note に記録（`/retro` スキル連携）

### 7-5. canary モニタリング起動

デプロイ先がある場合:
- `/canary` でヘルスチェック開始
- 異常検出 → Phase 8 に自動遷移

---

## Phase 8: フィードバックループ

Phase 7 完了後、プロダクト稼働中のフィードバックを自動検知→分類→適切な Phase に再突入。
`/feedback` コマンドで手動トリガーも可能。

### 8-1. フィードバック収集（自動）

入力ソース:
- canary モニタリング → エラーログ、レスポンス劣化
- CI/CD → テスト失敗、ビルドエラー
- GitHub Issue（手動）→ ユーザー/チームからの報告

### 8-2. 自動分類（トリアージ）

`/feedback-triage` スキルを内部呼び出し。4タイプに自動分類:

| 分類 | 定義 | 再突入先 |
|------|------|---------|
| BUG | 既存仕様の実装が動作しない | Phase 5 直接 |
| SPEC-GAP | 仕様に記載がないケースが発生 | Phase 1 に戻る |
| ENHANCE | 新しい機能要求 | Phase 1 に戻る |
| PERF | パフォーマンス劣化（仕様は満たしている） | Phase 3 に戻る |

深刻度: CRITICAL / HIGH / MEDIUM / LOW

### 8-3. 再突入フロー

#### BUG → Phase 5 直接再突入

1. フィードバック Issue 自動起票（BUG テンプレート）
2. `/investigate` で根本原因分析（3-strike ルール適用）
3. 修正 TASK Issue 起票（TASK-FXXX 採番）
4. Phase 5 に再突入（単一タスク、直列実行）
5. Phase 6 の CI修正ループ + レビューを経て PR
6. `Closes #{フィードバック Issue}`

**CRITICAL は即座に自動実行。HIGH 以下は確認ゲート1回。**

#### SPEC-GAP → Phase 1 再突入

1. フィードバック Issue 自動起票
2. `kairo-requirements` を差分モードで実行（既存要件を保持、不足分を追加）
3. Phase 2: 差分設計 → Phase 3: 追加 TASK 起票 → Phase 5-7: 通常フロー

**Phase 1 の確認ゲートで人間が仕様の妥当性を判断。**

#### ENHANCE → Phase 1 再突入（新規 or 既存 Epic 拡張）

- 既存 Epic の範囲内 → 既存 Epic に子 Issue 追加 → Phase 1 差分モード
- 範囲外 → 新規 Epic Issue 作成 → `/mille:kickoff` を新規実行

#### PERF → Phase 3 再突入

1. パフォーマンス調査 TASK を起票（プロファイリング→ボトルネック特定）
2. 最適化 TASK を起票 → Phase 5-7: 通常フロー

**調査結果の確認ゲート1回（最適化方針の承認）。**

### 8-4. 優先順位制御

```
処理順序:
1. CRITICAL BUG → 即時自動対応（他の作業を中断）
2. HIGH BUG → 次の作業サイクルで対応
3. SPEC-GAP → バッチ収集（デフォルト1日）してまとめて Phase 1 再突入
4. PERF → バッチ収集
5. ENHANCE → バックログ管理
6. LOW → バックログ
```

### 8-5. フィードバック追跡

Epic Issue の body に自動追記:

```markdown
### フィードバック履歴
| # | 分類 | 深刻度 | 再突入先 | 状態 |
|---|------|--------|---------|------|
| #{N} | BUG | CRITICAL | Phase 5 | ✅ 修正済み |
| #{N} | SPEC-GAP | MEDIUM | Phase 1 | 🔄 要件追加中 |
| #{N} | ENHANCE | LOW | バックログ | 📋 保留 |
```

---

## 完了サマリー

全フェーズ完了後に表示:

```
═══════════════════════════════════════
 MILLE KICKOFF v2 COMPLETE: {要件名}
═══════════════════════════════════════

## 生成された成果物

| Phase | ディレクトリ | 内容 |
|-------|-------------|------|
| 0 | CLAUDE.md / docs/tech-stack.md | 技術スタック |
| 1 | docs/spec/{要件名}/ | 要件定義書（EARS記法、信頼性レベル付き） |
| 2 | docs/design/{要件名}/ | 技術設計文書 |
| 3 | docs/tasks/{要件名}/ | タスク一覧（{N}件、{Wave数}Wave） |
| 3 | GitHub Issues | Epic #{番号} + TASK Issue {N}件 |
{Phase 4 実行時:}
| 4 | CLAUDE.md | プロジェクト概要・規約 |
| 4 | .claude/ | コマンド、エージェント、フック、ルール、スキル |
{Phase 5 実行時:}
| 5 | src/, tests/ | TDD実装コード（{Wave数}Wave 並列実行） |
| 5 | PR | TASK PR {N}件 |
{Phase 6 実行時:}
| 6 | PR comments | レビュー結果、E2Eテスト結果 |
{Phase 7 実行時:}
| 7 | CHANGELOG.md | リリースノート |
| 7 | GitHub Release | ドラフトリリース |

## 信頼性レベル分布（全体）

- 🔵 青信号（確定）: {N}件
- 🟡 黄信号（推測）: {N}件
- 🔴 赤信号（不明）: {N}件

## メトリクス

- 実装タスク: {N}件（{完了}件完了、{未完了}件未完了）
- テストカバレッジ: {N}%
- 並列効率: {N}x（直列比）
- CI修正ループ: {N}回発動

## 次のステップ

{Phase 5-7 スキップ時:}
1. /mille:kairo-loop {要件名} TASK-0001 TASK-{最終} で実装開始
2. Auto Mode (Shift+Tab) を有効化して自律実行

{全Phase 実行時:}
1. フィードバック待ち（/feedback でバグ報告・機能追加を受け付け）
2. /canary でヘルスチェック監視中（デプロイ済みの場合）
3. 🟡🔴 の未解決項目があれば /feedback で SPEC-GAP として再突入
```

---

## 確認ゲート配置まとめ

```
Phase 0   技術スタック     → 自動
Phase 0.5 リサーチ         → 自動
Phase 1   要件定義         → ✋ 確認ゲート（仕様判断）
Phase 1.5 Constitution     → 自動
Phase 2   技術設計         → ✋ 確認ゲート（設計判断）
Phase 3   タスク分割       → ✋ 確認ゲート + Issue自動起票
Phase 4   環境構築         → ✋ 確認ゲート（初回のみ）
Phase 5   並列TDD実装      → 自動（Wave 単位で並列）
Phase 6   品質保証         → 自動（6-4 マージ判定のみ ✋、--auto-merge で省略可）
Phase 7   ポストマージ     → 完全自動
Phase 8   フィードバック   → 分類による（BUG CRITICAL: 自動 / その他: ✋）
```

---

## 途中中止・再開

中止時、生成済み成果物はそのまま保持。個別コマンドで再開:

| 中止ポイント | 再開コマンド |
|------------|------------|
| Phase 0 完了後 | `/mille:kairo-requirements {要件名}` |
| Phase 1 完了後 | `/mille:kairo-design {要件名}` |
| Phase 2 完了後 | `/mille:kairo-tasks {要件名} --task` |
| Phase 3 完了後 | `/project-bootstrap`（mille 成果物を自動参照） |
| Phase 4 完了後 | `/mille:kairo-loop {要件名} TASK-0001 TASK-{N}` |
| Phase 5 完了後 | `/ship`（Phase 6 単体実行） |
| Phase 6 完了後 | Phase 7 は自動進行（手動不要） |
| Phase 7 完了後 | `/feedback {内容}`（Phase 8 手動トリガー） |

---

---

# ════════════════════════════════════════════
# feature モード（既存コードへの機能追加）
# ════════════════════════════════════════════

## Phase F0: プロジェクトコンテキスト取得

`docs/dev/context.md` が存在し鮮度が十分（7日以内）なら再利用。それ以外は再生成。

Task ツールで `general-purpose` サブエージェントを起動:

```
/mille:dev-context
```

出力: `docs/dev/context.md`（プロジェクト構造、技術スタック、アーキテクチャの自動スキャン結果）

---

## Phase F1: 機能計画

Task ツールで `general-purpose` サブエージェントを起動:

```
/mille:dev-plan {名前} "{要件の説明}"
```

- `--light` 指定時 or タスク数 ≤ 5 の見込み: lightweight モード
- それ以外: full-spec モード（EARS ベース要件定義込み）
- PRD ファイルがある場合: 内容をプロンプトに含める

### 生成される成果物

```
docs/dev/plans/{名前}/
├── plan.md              # 計画概要
├── tasks/
│   ├── 001-{タスク名}.md
│   ├── 002-{タスク名}.md
│   └── ...
└── reports/             # （dev-verify で後から生成）
```

### Phase F1.5: 画面仕様（Web プロジェクトの場合、オプション）

Web フロントエンドを含むプロジェクトの場合:

```
/mille:dev-screen-spec from-plan
```

### 確認ゲート

AskUserQuestion:
```
Phase F1 完了: 機能計画を作成しました。

## 成果物
- docs/dev/plans/{名前}/
- タスク数: {N}件

## タスク一覧
{タスクの要約リスト}

1. 続行する（Issue 起票 → 実装へ）
2. 計画を修正してから続行
3. ここで停止（--plan-only 相当）
```

---

## Phase F2: タスク→Issue ブリッジ

dev-plan のタスクファイルから GitHub Issue を自動起票。
`/mille:issue-workflow` スキルを内部呼び出し。

1. dev-plan タスクファイル（`docs/dev/plans/{名前}/tasks/*.md`）を読み取り
2. Epic Issue 作成（未作成の場合）
3. 各タスクから TASK Issue を起票（Phase 3 と同じフォーマット）
4. Wave 算出（依存関係グラフから並列実行グループを計算）
5. マッピングファイルを保存: `docs/dev/plans/{名前}/issue-map.md`

---

## Phase F3: 実装

タスク数に応じて実装方式を選択:

- **タスク > 5件**: Phase 5 並列 TDD（Wave + Agent worktree）を実行
  - 各 Agent worktree で `/mille:kairo-implement` を使用
  - PR に `Closes #{TASK Issue番号}` を付与
- **タスク ≤ 5件**: `/mille:dev-run` で順次実装
  - dev-run の impl → verify → debug ループを使用
  - 完了後に PR 作成、`Closes #{TASK Issue番号}` を付与

### 5-2.5. WTF-likelihood 自己制御

並列実行時のみ適用（greenfield Phase 5 と同じルール）。

---

## Phase F4-F6: QA → ポストマージ → フィードバック

greenfield の Phase 6-8 を再利用:
- **Phase F4**: Phase 6 QA パイプライン（自動レビュー、E2E、CI修正、マージ判定）
- **Phase F5**: Phase 7 ポストマージ（ドキュメント更新、Issue クローズ、メトリクス）
- **Phase F6**: Phase 8 フィードバックループ

---

# ════════════════════════════════════════════
# fix モード（バグ修正・調査）
# ════════════════════════════════════════════

## Phase X0: プロジェクトコンテキスト取得

`docs/dev/context.md` がなければ生成（Phase F0 と同じ）。

```
/mille:dev-context
```

---

## Phase X1: 調査

### --issue 指定時

```bash
gh issue view {番号} --json body,title,labels
```

Issue 本文から再現手順・エラーメッセージを取得。

### 根本原因分析

`/investigate` スキルを使用（3-strike ルール適用）:
1. 仮説を立てて検証
2. 3回失敗したらユーザーにエスカレーション
3. 出力: 原因特定、影響ファイル一覧

---

## Phase X2: 修正計画

影響ファイル数に応じて分岐:

- **影響ファイル < 3**: dev-impl クイックモード（即座に修正開始）
- **影響ファイル ≥ 3**: TASK Issue 起票 + dev-impl 通常モード

### 確認ゲート

AskUserQuestion:
```
根本原因を特定しました。

## 原因
{根本原因の説明}

## 影響ファイル
{ファイルリスト}

## 修正方針
{修正方針の説明}

1. 修正を開始する（TDD: 回帰テスト先行）
2. 方針を変更する
3. 中止する
```

---

## Phase X3: 実装

Task ツールで `general-purpose` サブエージェントを起動:

```
/mille:dev-impl "{修正内容の説明}"
```

TDD: 回帰テストを先に書き、テストが失敗することを確認してから修正。

---

## Phase X4: 検証

```
/mille:dev-verify
```

テスト全量実行、回帰がないことを確認。

---

## Phase X5: QA パイプライン（軽量版）

Phase 6 の軽量版:
- コードレビュー（単一 PR、Light 深度）
- CI チェック
- E2E は Critical Path 影響時のみ
- CI 失敗 → `/mille:ci-autofix` で自動修正（最大3回）

---

## Phase X6: マージ + Issue クローズ

PR マージ後:
- `--issue` 指定時: `Closes #{Issue番号}` で自動クローズ
- フィードバック Issue として起票されていた場合: Epic Issue のフィードバック履歴を更新

---

# ════════════════════════════════════════════
# refine モード（小規模変更）
# ════════════════════════════════════════════

## Phase R0: プロジェクトコンテキスト取得

`docs/dev/context.md` がなければ生成。

---

## Phase R1: 変更計画

Task ツールで `general-purpose` サブエージェントを起動:

```
/mille:refine-plan
```

対話的に変更箇所・影響範囲・計画を特定。

---

## Phase R2: 確認ゲート

AskUserQuestion:
```
変更計画を作成しました。

## 変更内容
{計画の要約}

## 影響範囲
{影響を受けるファイル・コンポーネント}

1. 実行する
2. 計画を修正する
3. 中止する
```

---

## Phase R3: 実行

```
/mille:refine-execute
```

テスト修正・実装・ビルド・テストを一括実行。

---

## Phase R4: QA パイプライン（最小版）

- Light レビュー（50行未満の差分を想定）
- CI チェック
- E2E なし

---

## Phase R5: マージ or 直接コミット

PR が作成された場合: マージ
直接コミットの場合: プッシュ

---

# ════════════════════════════════════════════
# 共通セクション
# ════════════════════════════════════════════

## Phase 8 フィードバックループの再突入ルーティング

Phase 8 のフィードバック分類結果に応じて、適切なモードで再突入:

| 分類 | 再突入モード | 具体的な動き |
|------|------------|-------------|
| BUG | fix モード | `/mille:kickoff {修正名} --mode fix --issue {番号}` |
| SPEC-GAP | feature モード | `/mille:kickoff {機能名} --mode feature --issue {番号}` |
| ENHANCE | feature モード（大規模は greenfield） | `/mille:kickoff {機能名} --mode feature` |
| PERF | fix モード | `/mille:kickoff {最適化名} --mode fix --issue {番号}` |

---

## 関連コマンド

### 統合エントリポイント
- `/mille:kickoff` - 本コマンド（4モード統合）

### greenfield 個別フェーズ
- `/mille:kairo-requirements` - Phase 1 の個別実行
- `/mille:kairo-design` - Phase 2 の個別実行
- `/mille:kairo-tasks` - Phase 3 の個別実行
- `/mille:kairo-loop` - Phase 5 の直列実行

### feature / fix 関連
- `/mille:dev-context` - プロジェクトコンテキスト生成
- `/mille:dev-plan` - 機能計画
- `/mille:dev-impl` - 単一タスク実装
- `/mille:dev-run` - バッチ実装ループ
- `/mille:dev-verify` - 検証
- `/mille:dev-debug` - エラー診断

### refine 関連
- `/mille:refine-plan` - 変更計画
- `/mille:refine-execute` - 変更実行

### QA・運用
- `/mille:ship` - Phase 6 QA パイプライン単体実行
- `/mille:feedback` - Phase 8 手動トリガー
- `/mille:auto-debug` - テストエラーの自動デバッグ

## 関連スキル

- `mille:issue-workflow` - Issue 駆動ワークフロー（Epic/TASK/フィードバック Issue 管理）
- `mille:ci-autofix` - CI 失敗→自動修正ループ
- `mille:feedback-triage` - フィードバック自動分類・再突入
- `mille:tdd-session` - TDD メソドロジー

## 関連エージェント

- `planner` - Phase 0: 技術スタック確認、Phase 8: ENHANCE 計画
- `research-analyst` - Phase 0.5: 不足情報の並列リサーチ
- `code-reviewer` - Phase 6-1: 自動レビュー
- `security-reviewer` - Phase 6-1: セキュリティレビュー
- `e2e-runner` - Phase 6-2: E2Eテスト
- `doc-updater` - Phase 7-1: ドキュメント自動更新
- `debugger` - fix モード: 根本原因調査

## 引数

$ARGUMENTS:
- 第1引数: 名前（必須）
- 第2引数: PRDファイルパス（オプション）
- `--mode greenfield|feature|fix|refine`: モード指定（省略時は自動検出）
- `--issue <番号>`: 既存 Issue にリンク（fix/feature モード）
- `--plan-only`: 計画段階で停止（feature/fix/refine モード）
- `--skip-impl`: Phase 5-7 をスキップ（greenfield）
- `--skip-bootstrap`: Phase 4 をスキップ（greenfield）
- `--skip-research`: Phase 0.5 をスキップ（greenfield）
- `--with-constitution`: Phase 1.5 で Constitution 生成（greenfield）
- `--light`: 軽量モード（greenfield）
- `--auto-merge`: Phase 6-4 のマージ確認ゲートを省略（greenfield）
- `--parallel N`: Phase 5 の並列数（デフォルト: 3、greenfield/feature）
- `--sequential`: Phase 5 を直列実行（greenfield）

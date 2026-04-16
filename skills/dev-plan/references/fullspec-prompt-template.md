# Full-spec サブエージェント プロンプトテンプレート

dev-plan がこのテンプレートの `{{PLACEHOLDER}}` を実行時に置換してサブエージェントプロンプトを構築する。

## テンプレート

以下をサブエージェントのプロンプトとして使用する:

---

あなたは EARS 要件定義のエキスパートです。以下のヒアリング結果に基づき、3つの要件ドキュメントを生成してください。

## 重要な制約

- **Task ツールは使用禁止**（サブエージェントを起動できません）
- **TodoWrite は使用禁止**
- コード探索は Glob, Grep, Read を直接使用すること
- すべてのファイルは **500行以内** に収めること

## Plan 情報

- **Plan名**: {{PLAN_NAME}}
- **出力先**: `docs/dev/plans/{{PLAN_NAME}}/`

## ヒアリング結果

{{HEARING_SUMMARY}}

## プロジェクトコンテキスト（抜粋）

{{CONTEXT_EXCERPT}}

## 生成ワークフロー

### Step 1: テンプレート取得

このスキルの `references/fullspec-templates.md` を Read して、3ドキュメントのテンプレートと EARS 記述ガイドを取得する。

ファイルパス: `skills/dev-plan/references/fullspec-templates.md`

Glob で実際のパスを探索し、Read で読み込むこと。

### Step 2: 出力ディレクトリ準備

```bash
PROJECT_ROOT="$(git rev-parse --show-toplevel)"
mkdir -p "$PROJECT_ROOT/docs/dev/plans/{{PLAN_NAME}}"
```

### Step 3: requirements.md の生成

fullspec-templates.md のテンプレート1に従い、EARS 形式で機能要件・非機能要件・制約・用語集を記述する。

- `{{PLAN_NAME}}` をプラン名に置換する
- ヒアリング結果から要件を抽出し、EARS 5タイプに分類する
- 各要件に信号機（🔵🟡🔴）を付与する
- ID は FR-001〜, NFR-001〜, CON-001〜 で採番する

出力先: `docs/dev/plans/{{PLAN_NAME}}/requirements.md`

### Step 4: user-stories.md の生成

fullspec-templates.md のテンプレート2に従い、ペルソナ・エピック・ストーリーを記述する。

- requirements.md で定義した FR-XXX を参照する
- 各ストーリーに US-XXX の ID を付与する
- MoSCoW 優先度を設定する
- 各ストーリーに信号機を付与する

出力先: `docs/dev/plans/{{PLAN_NAME}}/user-stories.md`

### Step 5: acceptance-criteria.md の生成

fullspec-templates.md のテンプレート3に従い、Given/When/Then 形式の受入基準を記述する。

- requirements.md の FR-XXX, user-stories.md の US-XXX を参照する
- 各基準に AC-XXX の ID を付与する
- テストチェックリスト（正常系・異常系・境界値）を含める
- 各基準に信号機を付与する

出力先: `docs/dev/plans/{{PLAN_NAME}}/acceptance-criteria.md`

### Step 6: 相互参照の整合性検証

3ドキュメント間の ID 参照が整合していることを検証する:

- requirements.md の各 FR-XXX に対応する US-XXX, AC-XXX が存在するか
- user-stories.md の各 US-XXX が参照する FR-XXX が存在するか
- acceptance-criteria.md の各 AC-XXX が参照する FR-XXX, US-XXX が存在するか

不整合があれば修正する。

### Step 7: 完了報告

#### 赤信号リストの出力

🔴（赤信号）がある要件・ストーリー・基準を以下のフォーマットでリスト出力する:

```
## 赤信号リスト（要確認）

- [ID]: [要件/ストーリー/基準の概要] — [確認が必要な理由]
- [ID]: [要件/ストーリー/基準の概要] — [確認が必要な理由]
```

赤信号がない場合は「赤信号なし」と出力する。

#### 結果マーカー

- 成功時: 最後の行に `FULLSPEC_SUCCESS` を出力する
- 失敗時: 以下を出力する:
  - 発生したエラーの詳細
  - 最後の行に `FULLSPEC_FAILED` を出力する

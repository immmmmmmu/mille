# ベンダリング記録

## ソース

- リポジトリ: [classmethod/tsumiki](https://github.com/classmethod/tsumiki)
- ライセンス: MIT

## 初回取り込み: 2026-04-05

### v1.2.0 ベース（ローカル運用実績あり、そのまま移動）

| カテゴリ | ファイル | 元バージョン | カスタマイズ |
|---------|---------|------------|-----------|
| kairo コマンド | kairo-requirements.md | v1.2.0→v1.3.0 | PREP.md 生成対応を取り込み |
| kairo コマンド | kairo-design.md | v1.2.0 | なし |
| kairo コマンド | kairo-tasks.md | v1.2.0 | なし |
| kairo コマンド | kairo-loop.md | v1.2.0 | なし |
| kairo コマンド | kairo-tasknote.md | v1.2.0 | なし |
| tdd コマンド | tdd-red/green/refactor/verify-complete/requirements/testcases/tasknote/todo | v1.2.0 | なし |
| rev コマンド | rev-tasks/design/requirements/specs | v1.2.0 | なし |
| dcs コマンド | bug-analysis/edgecase-analysis/feature-rubber-duck/impact-analysis/incremental-dev/performance-analysis/sequence-diagram-analysis/state-transition-analysis | v1.2.0 | なし |
| デバッグ | auto-debug/build-fix/env-fix/flaky-fix/timeout-fix | v1.2.0 | なし |
| その他 | orchestrate/help/init-tech-stack/direct-setup/direct-verify | v1.2.0 | なし |
| スキル | kairo-implement/ | v1.2.0 | なし |

### v1.3.0 新規取り込み

| カテゴリ | ファイル | カスタマイズ |
|---------|---------|-----------|
| Dev Skills コマンド | dev-context/plan/impl/run/verify/debug/screen-spec/webtest-plan/webtest/init/navigate | なし（参照を mille: に置換のみ） |
| Dev Skills スキル | skills/dev-context/plan/impl/run/verify/debug/screen-spec/webtest-plan/webtest/init/navigate | なし（参照を mille: に置換のみ） |
| refine | refine-plan/refine-execute | なし |
| DCS 追加 | dcs/code-question, dcs/test-performance-analysis | なし |
| kairo 追加 | kairo-task-verify | なし |

## チェリーピック履歴

（本家の新バージョンから取り込んだ変更をここに記録）

### チェリーピック手順

```bash
# 1. 本家の最新変更を確認
gh api repos/classmethod/tsumiki/releases/latest | jq '.tag_name, .body'

# 2. 前回取り込みからの diff を確認
# 本家リポジトリをクローンして diff
git clone --depth=1 https://github.com/classmethod/tsumiki.git /tmp/tsumiki-latest
diff -rq /tmp/tsumiki-latest/commands/ mille/commands/ | grep -v "Only in mille"

# 3. 有用な変更を手動マージ
# 4. この VENDORING.md にチェリーピック記録を追加
```

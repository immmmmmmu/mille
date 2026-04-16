---
name: tdd-session
description: TDDセッション（統合版）- t-wada方式のテスト駆動開発。Subagent活用、Issue連携、80%カバレッジ目標。新機能実装・バグ修正・リファクタリング時に使用
triggers:
  - "tdd"
  - "tdd-session"
  - "テスト駆動"
  - "TDD"
---

# TDD Session（統合版）

t-wada方式の厳格なTDDと、everything-claude-codeの詳細パターンを統合。

## セッションの流れ

### Phase 1: 準備（Subagents使用）

1. **Issue確認/作成**: 対応するGitHub Issueがあるか確認
2. **Explore Subagent** で既存コードとテストパターンを調査
3. **Plan Subagent** でテストケースを設計
4. **TodoWrite** でタスクリストを作成

```bash
# Issue確認
gh issue list --label "enhancement"

# Issue作成（必要な場合）
gh issue create --title "[機能名]" \
  --body "## 概要\n\n## 受け入れ基準\n- [ ] " \
  --label "enhancement"
```

### Phase 2: TDDサイクル（Red → Green → Refactor）

各テストケースに対して厳格に実行：

```
┌─────────────────────────────────────────┐
│  RED: 失敗するテストを1つ書く           │
│  - テストを実行して失敗を確認           │
│  - 失敗メッセージを読む                 │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  GREEN: 最小限のコードで通す            │
│  - 汚くても動けばOK                     │
│  - ハードコードでもOK                   │
│  - テストが通ることを確認               │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  REFACTOR: コードをきれいにする         │
│  - テストを緑に保つ                     │
│  - 重複除去、命名改善                   │
│  - テストコードもリファクタ対象         │
└─────────────────┬───────────────────────┘
                  ↓
              次のテストへ
```

### Phase 3: 完了

1. 全テストが通ることを確認
2. カバレッジ80%以上を確認
3. TodoWriteでタスク完了をマーク
4. PR作成を提案（`Closes #issue-number`）

## t-wada方式 厳守原則

1. **失敗するテストなしにプロダクションコードを書かない**
2. **テストは1つずつ書く** - 複数のテストを一度に書かない
3. **最小限のコードで通す** - 将来の拡張を先取りしない（YAGNI）
4. **テストが通ったらすぐリファクタリングを検討する**
5. **テストコードもリファクタリング対象**

## カバレッジ要件

| 対象 | 最低カバレッジ |
|-----|--------------|
| 全体 | 80% |
| 財務計算・認証 | 100% |
| コアビジネスロジック | 100% |

```bash
# カバレッジ確認
npm run test:coverage
# または
uv run pytest --cov --cov-report=html
```

## テストパターン

### AAA パターン（基本形）

```typescript
test('should return high score for liquid market', () => {
  // Arrange（準備）
  const market = {
    totalVolume: 100000,
    bidAskSpread: 0.01,
    activeTraders: 500
  }

  // Act（実行）
  const score = calculateLiquidityScore(market)

  // Assert（検証）
  expect(score).toBeGreaterThan(80)
})
```

### ユニットテスト

```typescript
describe('calculateSimilarity', () => {
  it('returns 1.0 for identical embeddings', () => {
    const embedding = [0.1, 0.2, 0.3]
    expect(calculateSimilarity(embedding, embedding)).toBe(1.0)
  })

  it('handles null gracefully', () => {
    expect(() => calculateSimilarity(null, [])).toThrow()
  })
})
```

### API統合テスト

```typescript
describe('GET /api/markets/search', () => {
  it('returns 200 with valid results', async () => {
    const request = new NextRequest('http://localhost/api/markets/search?q=test')
    const response = await GET(request, {})
    const data = await response.json()

    expect(response.status).toBe(200)
    expect(data.success).toBe(true)
  })

  it('returns 400 for missing query', async () => {
    const request = new NextRequest('http://localhost/api/markets/search')
    const response = await GET(request, {})
    expect(response.status).toBe(400)
  })
})
```

### E2Eテスト（Playwright）

```typescript
test('user can search and view market', async ({ page }) => {
  await page.goto('/')
  await page.fill('input[placeholder="Search"]', 'election')
  await page.waitForTimeout(600) // デバウンス

  const results = page.locator('[data-testid="market-card"]')
  await expect(results).toHaveCount(5, { timeout: 5000 })

  await results.first().click()
  await expect(page).toHaveURL(/\/markets\//)
})
```

## モックパターン

### Supabase

```typescript
jest.mock('@/lib/supabase', () => ({
  supabase: {
    from: jest.fn(() => ({
      select: jest.fn(() => ({
        eq: jest.fn(() => Promise.resolve({
          data: [{ id: 1, name: 'テスト' }],
          error: null
        }))
      }))
    }))
  }
}))
```

### Redis

```typescript
jest.mock('@/lib/redis', () => ({
  searchMarketsByVector: jest.fn(() => Promise.resolve([
    { slug: 'test-1', similarity_score: 0.95 }
  ]))
}))
```

### OpenAI

```typescript
jest.mock('@/lib/openai', () => ({
  generateEmbedding: jest.fn(() => Promise.resolve(
    new Array(1536).fill(0.1)
  ))
}))
```

## 必須エッジケース

1. **Null/Undefined**: 入力がnullの場合
2. **空**: 配列/文字列が空の場合
3. **無効な型**: 間違った型が渡された場合
4. **境界値**: 最小/最大値
5. **エラー**: ネットワーク障害、DB接続エラー
6. **特殊文字**: Unicode、絵文字、SQL文字

## テスト実行コマンド

| フレームワーク | コマンド |
|---------------|---------|
| pytest | `uv run pytest tests/ -v` |
| Jest | `npm test` |
| Vitest | `npm test` |
| Flutter | `flutter test` |
| Go | `go test ./...` |
| Rust | `cargo test` |
| Playwright | `npx playwright test` |

## Subagent使用ガイド

| 場面 | Subagent | 用途 |
|------|----------|------|
| 調査 | Explore | 既存コード・テストパターンの把握 |
| 設計 | Plan | テストケース分解、実装方針決定 |
| 実行 | Bash | テストコマンドの実行 |

## テスト品質チェックリスト

- [ ] すべてのパブリック関数にユニットテストがある
- [ ] すべてのAPIエンドポイントに統合テストがある
- [ ] 重要なユーザーフローにE2Eテストがある
- [ ] エッジケースがカバーされている
- [ ] エラーパスがテストされている
- [ ] 外部依存関係にモックを使用している
- [ ] テストが独立している（共有状態がない）
- [ ] カバレッジが80%以上

## 実行手順

引数として機能名が与えられたら：

1. 関連Issueの確認（なければ作成提案）
2. Explore Subagentで関連コードを調査
3. Plan Subagentでテスト計画を立てる
4. TodoWriteでタスクを作成
5. 各タスクに対してRed→Green→Refactorを実行
6. カバレッジを確認（80%以上）
7. 完了後、PRの作成を提案

## 自律モード（tdd-auto）

`/tdd-auto <機能名>` で全自動TDDを実行。手動介入なしでRed→Green→Refactorを繰り返す。

### 自律ループ

```
1. 機能をテスト可能な単位に分解
2. TaskCreateで各テストをタスク化（依存関係=blockedBy設定）

while (未完了のテストがある) {
  1. 次のテストを選択
  2. RED: 失敗するテストを書く → 実行して失敗確認
  3. GREEN: 最小限の実装 → 実行してパス確認
  4. REFACTOR: コード改善（テスト緑のまま）
  5. TaskUpdate: 完了マーク
}
```

### 自己修正ルール

テストが3回連続で失敗した場合:
1. エラーメッセージを詳細分析
2. 関連コードを再読み込み
3. 別のアプローチを試行
4. それでも失敗 → ユーザーに相談

### 出力フォーマット

```markdown
## TDD Auto Summary

### Feature: <機能名>

### Tests Created
| # | Test | Status | Coverage |
|---|------|--------|----------|
| 1 | creates user with valid data | ✅ | 100% |

### Implementation Files
- `src/services/user.ts` (new)
- `tests/user.test.ts` (new)

### Coverage
- Statements: 92% / Branches: 85% / Functions: 100% / Lines: 91%

### Iterations
- Total: 8 RED-GREEN-REFACTOR cycles
- Self-corrections: 2
```

## Phase 1 詳細: Explore Subagent 調査チェックリスト

Explore Subagent（thoroughness: medium以上）で以下を調査:

1. **既存テストの構造**: test/ディレクトリの構成とパターン
2. **命名規則**: 既存テストファイルの命名パターン
3. **共通ヘルパー**: テストで使われているユーティリティ
4. **モックの使い方**: mocktail等の使用パターン
5. **関連コード**: 対象に関連する既存実装
6. **セットアップ/ティアダウン**: 初期化パターン

## Phase 1 詳細: Plan Subagent テストケース設計

Plan Subagentでテストケースを優先順位付け:

1. **ハッピーパス**: 正常系の最もシンプルなケース
2. **バリエーション**: 異なる入力パターン
3. **境界値**: 最小値、最大値、空、null
4. **エラーケース**: 異常系、例外

実装順序の決め方:
- 依存関係が少ないものから
- 最もシンプルなケースから
- 基盤となる機能を先に

## 詳細リファレンス

- [t-wada方式 TDD原則](references/tdd-principles.md)

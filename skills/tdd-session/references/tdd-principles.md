# t-wada方式 TDD原則

## Red-Green-Refactorサイクル

```
1. Red:    失敗するテストを1つ書く（コンパイルエラーも「失敗」）
2. Green:  テストを通す最小限のコードを書く（汚くてもよい）
3. Refactor: テストが通ったまま、コードをきれいにする
```

## 厳守すべき原則

1. **失敗するテストなしにプロダクションコードを書かない**
2. **テストは1つずつ書く** - 複数のテストを一度に書かない
3. **最小限のコードで通す** - 将来の拡張を先取りしない（YAGNI）
4. **テストが通ったらすぐリファクタリングを検討する**
5. **テストコードもリファクタリング対象**

## 各フェーズの詳細

### Red（失敗するテストを書く）

- テストファイルを作成または開く
- 1つのテストケースのみを書く
- テストを実行して失敗を確認
- 失敗メッセージを確認してから次へ

チェックリスト:
- [ ] テストは1つだけか？
- [ ] テスト名は意図を明確に表しているか？
- [ ] AAAパターンになっているか？
- [ ] テストが失敗することを確認したか？

### Green（最小限のコードで通す）

- テストを通すための最小限のコードを書く
- 汚くても良い、動けば良い
- ハードコードでもよい - まず通すことが最優先
- テストを実行して成功を確認

やってはいけないこと:
- テストが求めていない機能を追加する
- 「ついでに」他の部分も直す
- 将来必要になりそうなコードを書く

### Refactor（リファクタリング）

前提: すべてのテストが通っている状態

プロダクションコード:
- 重複の除去（DRY）
- 命名の改善
- メソッドの抽出
- 責務の分離
- マジックナンバーの定数化

テストコード:
- 重複したセットアップの共通化
- テスト名の改善
- 不要なアサーションの削除
- テストヘルパーの抽出

## テストの書き方（AAA パターン）

### Python (pytest)

```python
def test_正常系_フォーマット済みタイムスタンプを返す():
    # Arrange（準備）
    log = JournalLog(
        id="test-id",
        created_at=datetime(2024, 1, 15, 10, 30),
        raw_text="テスト",
        duration_seconds=60,
    )

    # Act（実行）
    result = log.formatted_timestamp

    # Assert（検証）
    assert result == "2024-01-15 10:30"
```

### JavaScript/TypeScript (Jest)

```typescript
test('should return formatted timestamp when createdAt is provided', () => {
  // Arrange（準備）
  const log = new JournalLog({
    id: 'test-id',
    createdAt: new Date(2024, 0, 15, 10, 30),
    rawText: 'テスト',
    durationSeconds: 60,
  });

  // Act（実行）
  const result = log.formattedTimestamp;

  // Assert（検証）
  expect(result).toBe('2024-01-15 10:30');
});
```

### Flutter/Dart

```dart
test('should return formatted timestamp when createdAt is provided', () {
  // Arrange（準備）
  final log = JournalLog(
    id: 'test-id',
    createdAt: DateTime(2024, 1, 15, 10, 30),
    rawText: 'テスト',
    durationSeconds: 60,
  );

  // Act（実行）
  final result = log.formattedTimestamp;

  // Assert（検証）
  expect(result, '2024-01-15 10:30');
});
```

## テストの命名規則

### 英語形式
```
形式: should_[期待する結果]_when_[条件]

例:
- 'should return empty list when no logs exist'
- 'should throw ArgumentError when duration is negative'
- 'should save log when valid data provided'
```

### 日本語形式（pytest推奨）
```
形式: test_[条件]_[期待する結果]

例:
- test_ログがない場合_空リストを返す
- test_負の時間が指定された場合_ValueErrorを発生
- test_有効なデータの場合_ログを保存する
```

## やってはいけないこと

- テストを書く前にプロダクションコードを書く
- 一度に複数のテストを書く
- テストが失敗している状態でリファクタリングする
- 「あとでテストを書く」と言う

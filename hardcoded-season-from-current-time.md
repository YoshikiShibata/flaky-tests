# 現在時刻から導出される値のハードコードによる Flaky Test

## 概要

E2E テストで、サーバーが現在時刻から動的に導出する値（季節、期間区分、料金プランなど）をテストの期待値にハードコードしていたため、テスト作成時と異なる時期に実行すると失敗する現象が発生した。

## エラー内容

```
mismatch (-want +got):
- "derived_field": "VALUE_A",
+ "derived_field": "VALUE_B",
```

テスト作成時の時期に対応する値がハードコードされていたが、異なる時期に実行するとサーバーが別の値を返すため不一致となった。

## 根本原因

### サーバー側のロジック

サーバーはレコードの作成時刻など、時刻フィールドの月や日から動的に値を導出する：

```go
// 例: 時刻から区分を導出するロジック
func deriveCategory(t time.Time) Category {
    switch t.Local().Month() {
    case time.March, time.April:
        return CategoryA
    case time.May, time.June:
        return CategoryB
    // ...
    }
}
```

### テスト側の問題

テストデータの時刻フィールドにはデフォルトで `time.Now()` が使われるが、期待値はテスト作成時の導出結果にハードコードされていた。テスト作成時には正しかったが、別の時期に実行して初めて失敗した。

### 問題の構造

1. テストデータの入力値が `time.Now()` に依存する
2. サーバーがその入力値から動的に別の値を導出する
3. テストの期待値がその導出結果をハードコードしている
4. テスト作成時と異なる時期に実行すると不一致が発生する

## 解決策

### 採用した修正

テストデータ作成時の時刻を変数に保持し、その時刻からサーバーと同じロジックで期待値を動的に計算する：

```go
// 時刻を変数に保持し、テストデータ作成と期待値計算の両方で使用する
createTime := time.Now()
record, err := db.CreateRecord(ctx, ...,
    db.WithCreateTime(createTime))

// サーバーと同じロジックで期待値を動的に計算する
want := &pb.Response{
    DerivedField: expectedCategory(createTime),
}

func expectedCategory(t time.Time) pb.Category {
    switch t.Local().Month() {
    case time.March, time.April:
        return pb.CATEGORY_A
    // ... サーバー側の導出ロジックと同じマッピング
    }
}
```

### ポイント

- `time.Now()` を複数回呼ばず、一度だけ取得した値を共有する（境界での不一致を防ぐ）
- 期待値の計算ロジックはサーバー側の導出ロジックと一致させる

### 代替案

1. **入力時刻を固定する**: 時刻を特定の時期に固定し、対応する期待値をハードコードする（シンプルだが、導出ロジックの検証が特定の入力に限定される）
2. **導出フィールドを比較から除外する**: 比較オプションで除外する（検証が弱くなる）

## 教訓

1. **サーバーが現在時刻から導出する値は、期待値をハードコードしてはいけない**: テスト作成時には正しくても、別の時期に実行すると失敗する
2. **テストデータの入力値と期待値の導出元を一致させる**: 同じ時刻変数を入力と期待値計算の両方で使う
3. **`time.Now()` は一度だけ呼ぶ**: 複数回呼ぶと境界での不一致リスクがある
4. **「いつ実行しても同じ結果」になるか確認する**: 月・季節・年をまたぐケースを想定する

## 分類

- **カテゴリ**: 時間依存ハードコード問題（Time-Dependent Hardcoded Value）
- **発生頻度**: 導出ロジックの区分境界ごと（月次、季節ごとなど）
- **重要度**: 中（特定の時期にのみ失敗するため発見が遅れやすい）

## 関連パターン

- [month_boundary_master_data_switch.md](./month_boundary_master_data_switch.md) - 月末境界による月別設定値の切り替わり
- [flaky-test-time-boundary-patterns.md](./flaky-test-time-boundary-patterns.md) - 時間境界に関する他のパターン

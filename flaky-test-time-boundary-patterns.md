# 時刻境界で失敗するフレーキーテストの原因と対策

## はじめに

E2Eテストがたまに失敗する「フレーキーテスト」に悩まされたことはありませんか？
原因の一つに、テストコードとサーバーコードで時刻計算のタイミングがずれる問題があります。

本記事では、実際に遭遇した2つのケースを通じて、
時刻境界でのフレーキーテスト問題とその対策パターンを紹介します。

## 問題の本質

テストで期待値を計算する際に `time.Now()` を使うと、
サーバーが処理した時刻とテストが計算した時刻がずれる可能性があります。

特に以下のケースで問題が発生しやすいです：
- 日境界（00:00:00付近）でテストが実行される
- サーバーが時刻を「日」単位で丸めて計算する
- タイムゾーン（UTC/JST）の境界を跨ぐ

## ケース1: レスポンスに基準時刻が含まれている場合

### シナリオ

リソース作成APIを呼び出すと、サーバーが `CreatedAt` と `ExpiredAt` を計算して返します。
`ExpiredAt` は「`CreatedAt` の日の23:59:59 + 有効期間」として計算されます。

### 問題のあったコード

```go
func TestCreateResource(t *testing.T) {
    // APIを呼び出し
    response := client.CreateResource(ctx, request)

    // 期待値を計算（問題：time.Now() を別途呼んでいる）
    now := time.Now().In(jst)
    wantExpiredAt := time.Date(
        now.Year(), now.Month(), now.Day(),
        23, 59, 59, 0, jst,
    ).Add(time.Duration(validHours) * time.Hour)

    // 検証
    if !response.ExpiredAt.Equal(wantExpiredAt) {
        t.Errorf("ExpiredAt mismatch: got %v, want %v",
            response.ExpiredAt, wantExpiredAt)
    }
}
```

**問題点**: API呼び出し時刻とテストの `time.Now()` が日境界を跨ぐと、
サーバーとテストで異なる日を基準に計算してしまいます。

```
例: 23:59:59 にAPIが処理され、00:00:01 にテストが time.Now() を呼んだ場合

サーバー: 1月15日 23:59:59 + 24時間 = 1月16日 23:59:59
テスト:   1月16日 23:59:59 + 24時間 = 1月17日 23:59:59  ← 1日ずれる！
```

### 修正後のコード

```go
func TestCreateResource(t *testing.T) {
    // APIを呼び出し
    response := client.CreateResource(ctx, request)

    // サーバーが返した CreatedAt を基準に期待値を計算
    createdAt := response.CreatedAt.In(jst)
    wantExpiredAt := time.Date(
        createdAt.Year(), createdAt.Month(), createdAt.Day(),
        23, 59, 59, 0, jst,
    ).Add(time.Duration(validHours) * time.Hour)

    // 検証
    if !response.ExpiredAt.Equal(wantExpiredAt) {
        t.Errorf("ExpiredAt mismatch: got %v, want %v",
            response.ExpiredAt, wantExpiredAt)
    }
}
```

**ポイント**: サーバーが返した `CreatedAt` を基準に期待値を計算することで、
テスト実行時刻に依存しない検証が可能になります。

## ケース2: レスポンスに基準時刻が含まれていない場合

### シナリオ

リソース発行APIを呼び出すと、有効期限付きのリソースが作成されます。
`ExpiredAt` は「発行日の23:59:59 + 30日」として計算されますが、
レスポンスには発行時刻が含まれていません。

### 問題のあったコード

```go
func TestIssueResource(t *testing.T) {
    // APIを呼び出し
    client.IssueResource(ctx, request)

    // リソースを取得
    resource := client.GetResource(ctx)

    // 期待値を計算（問題：API呼び出しと別のタイミングで time.Now() を呼んでいる）
    now := time.Now().In(jst)
    wantExpiredAt := time.Date(
        now.Year(), now.Month(), now.Day(),
        23, 59, 59, 0, jst,
    ).Add(30 * 24 * time.Hour)

    // 検証
    if !resource.ExpiredAt.Equal(wantExpiredAt) {
        t.Errorf("ExpiredAt mismatch: got %v, want %v",
            resource.ExpiredAt, wantExpiredAt)
    }
}
```

### 修正後のコード

```go
func TestIssueResource(t *testing.T) {
    // API呼び出しの前後で時刻を記録
    before := time.Now().In(jst)
    client.IssueResource(ctx, request)
    after := time.Now().In(jst)

    // 日境界を跨いだ場合はスキップ
    if before.Day() != after.Day() {
        t.Skip("API呼び出し中に日が変わったためスキップ")
    }

    // リソースを取得
    resource := client.GetResource(ctx)

    // API呼び出し直前の時刻を基準に期待値を計算
    wantExpiredAt := time.Date(
        before.Year(), before.Month(), before.Day(),
        23, 59, 59, 0, jst,
    ).Add(30 * 24 * time.Hour)

    // 検証
    if !resource.ExpiredAt.Equal(wantExpiredAt) {
        t.Errorf("ExpiredAt mismatch: got %v, want %v",
            resource.ExpiredAt, wantExpiredAt)
    }
}
```

**ポイント**:
- API呼び出しの前後で時刻を記録
- 日が変わった場合は `t.Skip()` でスキップ
- API呼び出し直前の時刻を基準に期待値を計算

## 対策パターンのまとめ

### パターン1: サーバーが返した値を基準にする

サーバーのレスポンスに基準となる時刻（`CreatedAt`, `StartedAt` など）が
含まれている場合は、その値を使って期待値を計算します。

```go
// Good: サーバーが返した値を基準に計算
createdAt := response.CreatedAt
wantExpiredAt := calculateExpiredAt(createdAt)

// Bad: 独立して time.Now() を呼ぶ
wantExpiredAt := calculateExpiredAt(time.Now())
```

### パターン2: API呼び出し前後で時刻を記録し、日境界をスキップ

サーバーのレスポンスに基準時刻が含まれていない場合は、
API呼び出しの前後で時刻を記録し、日が変わった場合はスキップします。

```go
before := time.Now().In(jst)
response := callAPI()
after := time.Now().In(jst)

if before.Day() != after.Day() {
    t.Skip("日境界を跨いだためスキップ")
}

wantExpiredAt := calculateExpiredAt(before)
```

## まとめ

時刻境界でのフレーキーテストを防ぐためのポイント：

1. **`time.Now()` の呼び出しタイミングを意識する** - サーバー処理とテスト計算で同じ時刻を使う
2. **サーバーのレスポンスを活用する** - 可能な限りサーバーが返した値を基準にする
3. **日境界のスキップを検討する** - 稀なケースはスキップで対処
4. **サーバー側の計算ロジックを理解する** - 期待値の計算はサーバーと同じロジックで

これらの対策により、深夜のCI実行でも安定したテストを実現できます。

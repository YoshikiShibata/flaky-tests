# セットアップ・クリーンアップの順序が引き起こす孤立参照による Flaky Test

## 概要

テストヘルパーが「子レコードを先に INSERT し、親レコードを後で INSERT する」順序になっていると、`t.Cleanup` の LIFO により setup 中・cleanup 中の双方で「子レコードは存在するが、参照先の親レコードが存在しない」状態が一瞬発生する。

並列実行される別テストが、この一瞬の不整合状態でテーブルをグローバルに走査し、子レコードの参照を辿るような処理を実行すると、参照先未存在エラーで失敗する。

## 発生条件

以下の3条件が揃うと発生する:

1. **参照関係を持つ複数レコードのセットアップ** — `Child.ParentID` のような参照を持つレコード群を別々の `SetUp*` ヘルパーで作成する
2. **挿入順序が参照方向と逆** — Child → Parent の順で挿入している（あるいは ParentID を Child に設定した上で Parent を後から作る）
3. **並列実行される別テストがグローバル走査と参照解決を行う** — 例: ジョブテストがテーブル全行を走査し、各行の参照先を別テーブルから取得する

## なぜ問題が起きるか

### Setup 中の不整合状態

```go
// 旧コード: Child を先に挿入
bsp := db.SetUpChild(t, func(c *Child) {
    c.ParentID = uuid.NewString() // まだ存在しない Parent.ID
})

db.SetUpParent(t, func(p *Parent) {
    p.ID = bsp.ParentID
})
```

`SetUpChild` 完了から `SetUpParent` 完了までの間、Child は存在するが対応 Parent が存在しない。

### Cleanup 中の不整合状態（LIFO）

`t.Cleanup` は LIFO で実行されるため、cleanup 順序は setup の逆になる:

```
setup:   Child → Parent
cleanup: Parent → Child  ← Parent 削除後、Child が孤立
```

`Parent.Delete` 実行後 `Child.Delete` 実行前の間、Child は存在するが Parent は存在しない。

### 並列テストの干渉

ジョブテストなどが「テーブルを全行走査して各行の参照先を取得する」処理を実行する場合、上記の不整合期間中に処理が走ると参照先未存在エラーで失敗する。

```
TestA setup:  Child INSERT → (この間に TestB がジョブ実行) → Parent INSERT
                                  ↓
                            TestB が Child を見つけ Parent を探すが見つからない → fail
```

## 解決策

### 挿入順序を参照方向に合わせる

Parent を先に作成し、その ID を使って Child を作成する:

```go
// 修正後: Parent を先に挿入
parent := db.SetUpParent(t, func(p *Parent) {
    p.SomeField = "..."
})

db.SetUpChild(t, func(c *Child) {
    c.ParentID = parent.ID
})
```

これにより:

- **Setup 中**: Parent 挿入完了 → Child 挿入完了。Child が存在するときは必ず Parent も存在する
- **Cleanup 中（LIFO）**: Child 削除 → Parent 削除。Parent が存在する間は Child が孤立しない

setup・cleanup のどの瞬間においても、Child が存在する間は対応 Parent が存在することが保証される。

## 一般化したパターン

| 項目 | 内容 |
|------|------|
| 不整合の種類 | 参照先未存在（外部キー制約相当の整合性違反） |
| 発生タイミング | Setup の中間・Cleanup の中間 |
| 発見の難しさ | 単独実行・小規模並列では再現しにくい。並列実行する別テストがグローバル走査するときだけ発生する |
| 検出方法 | 「purchase is not found」のような参照先未存在エラーが、検証対象のレコード ID と無関係な ID で発生していないか確認する |
| 解決方法 | 参照される側（親）を先に挿入する。`t.Cleanup` の LIFO 性質を利用して cleanup 中も整合性を保つ |

## チェックリスト

参照関係を持つレコードを複数の `SetUp*` ヘルパーで作成するテストでは、以下を確認する:

1. **挿入順序は参照方向と一致しているか？** — `Child.ParentID = parent.ID` のような参照を作るなら、Parent を先に挿入する
2. **`t.Cleanup` の LIFO 順序を考慮しているか？** — 挿入順序を逆にしたものが cleanup 順序になる。cleanup 中も参照整合性を保てるか
3. **同じパッケージ内に並列実行される別テストはあるか？** — 特にグローバル走査するジョブテスト等
4. **そのジョブテストはテーブルを全行走査し、参照先を取得するか？** — する場合、不整合期間に干渉する

## 教訓

1. **`t.Cleanup` は LIFO** — setup 順序と cleanup 順序は逆になる。両方の中間状態を考慮する
2. **「全レコード setup 完了後に検証」だけを考えると見落とす** — setup の途中・cleanup の途中も他テストから観測される
3. **グローバル走査するジョブとの組み合わせが特に危険** — テストが自分の作ったレコードしか触らなくても、他テストの中間状態を踏み抜く可能性がある
4. **並列テストパッケージ内のセットアップヘルパーは「常に参照整合性を保つ順序」で書く** — これは実コードのトランザクション順序設計と同じ原則

## 関連パターン

- [全データ更新ジョブによるテスト間干渉](./global-data-modification-interference.md) — グローバル走査ジョブが他テストデータを上書きするパターン。本パターンは「上書き」ではなく「参照解決失敗」だが、グローバル走査と並列テストの組み合わせという点は共通
- [逐次ループの並行化におけるループキャリー依存性の見落とし](./loop-carried-dependency-in-concurrent-test-setup.md) — 並列化により暗黙の順序依存が壊れるパターン

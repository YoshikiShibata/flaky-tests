# deadline 到達時のステータスコードは client timer と server 応答のレースで揺れる

## 概要

RPC が context deadline で打ち切られることを検証するテストで、期待するステータスコードを
`DeadlineExceeded` **一つに固定**してアサートすると Flaky になる。

deadline 到達の瞬間には、**両端で同時にタイマーが切れる**からである。

1. **client 側**: 自分の ctx deadline が切れる → gRPC ライブラリが `DeadlineExceeded` を返す
2. **server 側**: 伝播された deadline で handler の ctx も切れる → 処理中の DB／外部呼び出しが
   `context deadline exceeded` を返す → アプリのエラー分類・ステータス変換処理が
   「一時的エラー」と判定し、`Unavailable` 等の**アプリ定義のコード**で応答する

client が最終的に観測するのは「自分の timer が先に発火する」か「server の応答が先に届く」かの
**レースの勝者**であり、どちらも起こり得る。低負荷では 1 が勝ち続けるので何十回流しても再現せず、
高負荷（並列実行・長時間ループ）で timer 発火や待機 goroutine の再スケジュールが数 ms 遅れたときだけ 2 が勝つ。

重要なのは、**このレースはプロダクトのバグではない**という点。server が deadline 起因のエラーを
retriable として分類して返すのは正しい設計（呼び出し側にリトライ／再配信を促す）であり、
client が `DeadlineExceeded` を返すのも正しい。**「観測されるコードは一意に定まらない」という事実を
テストが知らなかっただけ**である。この点が、同じく「非決定的なステータス」でも
[先着エラーのマッピング取りこぼし](./errgroup-first-error-status-mapping-gap.md)（あちらは実バグ）
とは決定的に異なる。

## 発生条件

1. テストが RPC 呼び出しに**短い context deadline** を張り、deadline で打ち切られることを検証している
2. その deadline は**メタデータ経由で server 側にも伝播**している（gRPC ではデフォルト挙動）
3. server 側の処理が deadline 到達を**自前で観測してエラーを返す**（＝ ctx を尊重した DB クライアント・
   外部呼び出しを使っている）
4. server のエラー分類・ステータス変換処理が ctx deadline 由来のエラーを **retriable 等に分類し独自コードで応答**する
5. テストが `status.Code(err) != codes.DeadlineExceeded` のような**単一コード一致**を要求している

## なぜ「原理的には client が勝つはず」なのに負けるのか

gRPC の deadline 伝播は**絶対時刻ではなく相対時間**で行われる。

- client はヘッダ書き込み時刻 `t_write` に「残り時間」を計算して `grpc-timeout` ヘッダに載せる
  → `timeout = D_client − t_write`
- server はヘッダ**受信時刻** `t_recv` にそれを足して絶対時刻にする
  → `D_server = t_recv + (D_client − t_write) ≥ D_client`（`t_recv ≥ t_write` のため）

つまり **server 側 deadline は必ず client 側 deadline 以降**（差は転送遅延ぶん。loopback なら µs〜ms）。
さらに server は ctx が切れてからエラーを組み立て・ログ出力し・trailers を送り返す往復ぶんのハンデも負う。
だから通常は client が勝ち、テストは安定して通る。

負けるのは次のいずれかが起きたとき:

- **client 側 timer の発火／待機 goroutine の再スケジュールの遅れ**。高負荷（多数の並列テスト、長時間稼働、GC）では
  ランタイムのタイマー処理と待機 goroutine の再スケジュールが数 ms 単位で遅れる。この遅れが
  上記のハンデ（µs〜ms）を上回ると、server の応答が先に届く。
- **同時 ready 時のランダム選択**。受信側は
  `select { case <-ctxDone: …; case m := <-recv: … }` の形で待つ。両方の case が ready に
  なっていれば Go の `select` は**一様ランダム**に選ぶ。マージンが極小なのでこの分岐に入る確率も無視できない。

要するに勝敗マージンは **ミリ秒未満**。「レースの勝率が高いだけ」であって、勝ちは保証されていない。

## 典型的な症状

「サーバー側のリトライが固定回数上限を持たず、リクエストの context deadline まで続くこと」を
検証するテスト（フォールト注入でリトライ可能エラーを返し続けさせ、deadline で打ち切らせる型）が
連続実行の数十周目で初めて落ちる。

```go
reqCtx, cancel := context.WithTimeout(ctx, 15*time.Second)
defer cancel()

if _, err := client.SomeRPC(reqCtx, req); status.Code(err) != codes.DeadlineExceeded {
    t.Fatalf("got code %s, want DeadlineExceeded: err=%v", status.Code(err), err)
}
```

```
    xxx_test.go:NN: got code Unavailable, want DeadlineExceeded: err=rpc error: code = Unavailable desc = Unavailable
--- FAIL: TestXxx (15.11s)
```

同時刻の server 側ログ:

```
Unavailable: retriable error from ...: failed to ...: context deadline exceeded
```

### ログから何が確定できるか（重要）

エラーの中身が gRPC ステータスではなく**素の Go の `context deadline exceeded`**（`ctx.Err()` 由来）
であることに注目する。これは **ctx が Done になった瞬間にしか発生しない**。したがって:

- リクエストは deadline いっぱいまで生き延びていた
- ＝ **テストが検証したかった「固定回数上限が無く deadline まで続く」という性質は成立していた**
- 壊れていたのは「その事実をどう観測するか」だけ

テスト全体の所要時間（`--- FAIL … (15.11s)`）が deadline と整合していることも、
「途中で別要因が早期に落とした」のではないことを裏付ける。

**エラーメッセージが gRPC status 由来か素の `ctx.Err()` 由来かを見分けると「いつ壊れたか」が一発で分かる。**
前者なら下流の呼び出しが独自に timeout した可能性があるが、後者は自分の ctx が切れたことしか意味しない。

## 解決策: コードは両方許容し、検証したい性質は別の軸で測る

```go
const reqTimeout = 15 * time.Second
reqCtx, cancel := context.WithTimeout(ctx, reqTimeout)
defer cancel()

start := time.Now()
_, err := client.SomeRPC(reqCtx, req)
elapsed := time.Since(start)

// deadline 到達時、client 側 timer の DeadlineExceeded と、server 側が同じ deadline で
// 検知した ctx エラーを retriable 分類して返す Unavailable が競合する。
// 呼び出し側にとってはどちらも同じ「一時的失敗（リトライ／再配信で吸収される）」なので両方許容する。
if code := status.Code(err); code != codes.DeadlineExceeded && code != codes.Unavailable {
    t.Fatalf("got code %s, want DeadlineExceeded or Unavailable: err=%v", code, err)
}
// 本テストの主眼「リトライが deadline まで続くこと」は経過時間で担保する。
// 固定回数上限があれば、また別要因の一過性エラーで早期に落ちれば、ここより早く返る。
if minElapsed := reqTimeout - time.Second; elapsed < minElapsed {
    t.Fatalf("returned after %s, want >= %s (retry must continue until the deadline): err=%v",
        elapsed, minElapsed, err)
}
```

ポイントは **「コードの許容を広げた」だけで終わらせない**こと。単に
`DeadlineExceeded || Unavailable` を許すと、**deadline 到達前に別要因で返る `Unavailable`**
（接続の一過性スタック等。cf. [単一コネクション転送スタック](./single-connection-stall-hangs-fixed-deadline-write.md)）も
通してしまい、テストの検出力が落ちる。**緩めた軸のぶんを、別の軸（経過時間）で埋め合わせる。**

client 側からは `desc` がマスクされて `code = Unavailable desc = Unavailable` としか見えず、
メッセージで原因を切り分けられないことも多い。その場合、経過時間が唯一の実用的な判別軸になる。

### どうしてもコードを一意に固定したい場合

- **client 側の打ち切りを確定させたい**なら、ctx に deadline を張るのをやめて
  `context.WithCancel` + `time.AfterFunc(d, cancel)` にする。deadline が server に伝播しないので
  server は自発的にエラーを返さず、観測されるコードは確定的に `Canceled` になる。
  （ただし server 側は RST_STREAM を受け取るまで走り続ける点に注意）
- **server 側の分類結果を検証したい**なら、client の deadline を十分長くし、
  server 側に短いタイムアウトをフォールト注入して「必ず server が先に返す」構図にする。

いずれも「両端のタイマーを同時刻に置かない」ことでレースそのものを消している。
**同時刻に置いたまま勝者を仮定するのが誤り。**

## 一般化したパターン

> **境界で同時に発火する 2 つの主体があるとき、どちらが観測されるかを仮定してはいけない。**

- deadline / timeout は client と server の**両方**が持つ。しかも同じ時刻に設定されている
  （gRPC は deadline を伝播するので、明示的に別値にしない限り必ずそうなる）。
- 「client が先に切れるはず」という論証（`D_server ≥ D_client`）は**平常時の勝率の話**でしかなく、
  スケジューリング遅延で簡単に逆転する。マージンが µs〜ms なら実質コイントスと同じ扱いをするべき。
- テストが assert すべきは**観測された症状（コード）**ではなく、**検証したい性質**そのもの。
  今回なら「固定上限で打ち切られていないこと」＝「deadline まで時間を使い切ったこと」。
  症状は複数の等価な形で現れうるが、性質は一つ。

同型の落とし穴:

- クライアント／サーバー双方のリトライが同時に効く場合の「試行回数」アサート
- キャンセル伝播と処理完了の競合（`Canceled` を期待したら成功レスポンスが返る）
- タイムアウト付きロック取得で「取得失敗」を期待したら、解放が一瞬先に起きて成功する

## レビュー・実装チェックリスト

- [ ] deadline / timeout での失敗を検証するテストで、ステータスコードを**単一値**に固定していないか
- [ ] その deadline は server 側にも伝播しているか（gRPC ならデフォルトで伝播する）
- [ ] server 側は ctx deadline を自分で観測して**独自のコード**に分類して返す実装になっていないか
      （retriable 分類 → `Unavailable` 変換などは典型）
- [ ] コードの許容を広げたなら、**失われた検出力を別の軸で補っている**か（経過時間・副作用の有無・状態遷移）
- [ ] 「単独実行で N 回通った」を安全証明として扱っていないか
      （cf. [長時間ランニングテストの重要性](./importance-of-long-running-tests.md)）
- [ ] エラーメッセージが gRPC status 由来か素の `ctx.Err()` 由来かを確認して「いつ壊れたか」を特定したか

## 関連概念

- **deadline propagation**: gRPC は `grpc-timeout` ヘッダで残り時間（相対値）を伝える。
  server 側の絶対 deadline は受信時刻起点なので、必ず client 側以降になる。
- **エラー分類（retriable / permanent / canceled）**: ctx deadline 由来のエラーを retriable に含めるのは、
  再配信・リトライで吸収させたいワーカーとしては正しい設計。テスト側がその設計を知らないと
  「なぜ `Unavailable` が返るのか」を実装バグと誤診しやすい。
- **`select` の一様ランダム選択**: 複数 case が同時 ready のとき Go は乱択する。極小マージンのレースでは
  この性質自体が非決定性の増幅器になる。

## 関連ドキュメント

- [並行取得の「先着エラー」非決定性とエラー→ステータスマッピングの取りこぼし](./errgroup-first-error-status-mapping-gap.md)
  — 同じ「ステータスコードが揺れる」でも、あちらは**プロダクトの実バグ**、本件は**テストの前提の誤り**。
  server の応答が設計通りか否かが最初の切り分け点になる。
- [エミュレータへの書き込みが単一コネクションの転送スタックで稀にタイムアウトする](./single-connection-stall-hangs-fixed-deadline-write.md)
  — 同じ `Unavailable` でも deadline 到達前に返る別要因。経過時間チェックを残す理由。
- [長時間ランニングテストの重要性](./importance-of-long-running-tests.md)
  — 数十回連続 PASS しても次の周で落ちる実例。

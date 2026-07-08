# プロセス共有 singleflight がリーダーの ctx キャンセルを無関係なフォロワーへ伝搬させる

## 概要

`golang.org/x/sync/singleflight` で重複呼び出しをまとめている処理が、**呼び出し元スコープの（キャンセルされうる）`context.Context`** を使って実処理を実行していると、**最初に `Do` へ入った呼び出し元（リーダー: leader）の ctx がキャンセルされた瞬間、同じキーを待っている無関係な呼び出し元（フォロワー）全員がそのキャンセルを相続**する。

`singleflight.Group.Do(key, fn)` は「リーダーの `fn`」だけを実行し、その結果（値・エラー）を実行中に合流した全フォロワーへ**共有**する。`fn` の中でリーダーの ctx を使って DB/RPC を呼ぶと、その ctx のキャンセルはリーダー固有の事情（別リクエストの errgroup 兄弟エラー、クライアント切断、deadline 等）であっても、**自分の ctx が健全なフォロワーまで巻き込む**。

結果として、あるリクエストのキャンセルが、それとは無関係な別リクエスト（別テスト・別ユーザー・別 tid）を `context canceled` で失敗させる。これは**テストの問題ではなくプロダクトの並行性バグ**であり、E2E の並列実行がそれを露出させる。

これは「プロセス共有の状態が、リクエスト/テスト単位の分離を貫通する」という [プロセス共有キャッシュによる test-scoped fake override の無効化](./process-shared-cache-vs-tid-scoped-fake.md) と同根の構造問題である（あちらはキャッシュが tid スコープを貫通、こちらは singleflight が ctx スコープを貫通）。

## 症状

- 特定テストを単独実行すると常に成功する。全 E2E を並列実行したときだけ、**無関係なテスト**がランダムに失敗する。
- 失敗するテストの操作（例: ある共有マスタを読む正常系のセットアップ RPC）は、キャンセルとは無縁の正常系である。それでも `Internal`（`context canceled`）で落ちる。
- 失敗したリクエストのクライアント側には deadline も明示 cancel も無い（例: E2E クライアントの deadline は数分単位で、失敗は 0.2 秒未満）。にもかかわらずサーバ側の DB クエリが `context canceled` になっている。
- ログ全体を通して、その `context canceled` が**ちょうど 1 回しか出ない**ことがある（後述の errgroup マスキングにより、キャンセルの発生源＝リーダー側では握り潰され、フォロワー側にだけ表面化するため）。
- キャッシュ TTL 失効直後や cold start など、**cache miss が起きて singleflight に呼び出しが集中する一瞬**でのみ顕在化する。

## 根本原因

### 構造的要因

1. **重複排除がプロセス共有**: `singleflight.Group` がパッケージ変数などプロセス全体で 1 インスタンス。キーもリクエスト/tid/ユーザーを含まない共有キー。
2. **`Do` の `fn` が呼び出し元スコープの ctx を使う**: `fn` の中で `repo.Get(ctx)` のように、リクエストごとの（＝キャンセルされうる）ctx をそのまま渡している。
3. **リーダーの ctx がキャンセルされうる**: 特に呼び出し元が `errgroup.WithContext` で派生した ctx を使っている場合、**兄弟の並行処理が（何であれ）エラーを返すと共有 ctx がキャンセル**される（`errgroup` は最初の非 nil エラーで派生 ctx を cancel する）。クライアント切断・deadline でも同様。

### 発生メカニズム

1. cache miss が起き、複数リクエストがほぼ同時に `singleflightGroup.Do(key, fn)` に入る。最初の 1 本が**リーダー**、残りは**フォロワー**として待機する。
2. リーダーは自分の ctx で `fn`（例: `GetAll(ctx)`）を実行する。
3. リーダーの ctx が**実行中にキャンセル**される。典型例:
   - リーダーが `errgroup` 配下のリクエストで、**兄弟の並行処理がエラーを返し** errgroup が派生 ctx を cancel した。
   - リーダーのクライアントが切断／deadline 超過した。
4. `fn` の中の `GetAll(ctx)` が `context canceled` を返す。
5. `singleflight` はその error を**リーダーと全フォロワーへ返す**。フォロワーは自分の ctx が健全でも `context canceled` を受け取り、失敗する。

### なぜ発見が遅れるか（errgroup による発生源側マスキング）

リーダーのキャンセルが **errgroup 兄弟エラー**由来のとき、キャンセルの発生源側（リーダーのリクエスト）では **singleflight のキャンセルエラーは表に出ない**。`errgroup.Wait()` は**最初に起きたエラーだけ**を返すため、先にエラーを返した兄弟（＝キャンセルの原因）のエラーが勝ち、後から `context canceled` になった singleflight 側のエラーは**握り潰される**。

その結果、`context canceled` は**フォロワー側にだけ**表面化する。ログを見ても発生源のリクエストには全く別のエラー（兄弟のエラー）しか出ておらず、両者の因果が見えにくい。「無関係なリクエストが 1 回だけ謎の `context canceled` で落ちる」という観測しづらい形になる。

（`errgroup.Wait()` が先着エラーしか返さない非決定性そのものについては [並行取得の「先着エラー」非決定性とエラー→ステータスマッピングの取りこぼし](./errgroup-first-error-status-mapping-gap.md) を参照。）

### なぜ稀か

`singleflight` に入るのは cache miss のときだけ。したがって発火には「**cache が cold な一瞬**に、キャンセルされる運命のリクエストが**たまたまリーダー**になり、かつ別の無関係リクエストが**同時にフォロワーとして合流**する」というタイミングの一致が要る。cache が warm な通常時は `Do` にすら入らないため、長時間・多数回・高並列で回したときにだけ観測される（[長時間ランニングテストの重要性](./importance-of-long-running-tests.md) と同じ構図）。

## 具体例

### 状況

ある共有マスタ（全件が小さいテーブル）を全件取得する処理が、プロセス共有 singleflight でまとめられていた。

```go
var singleflightGroup = singleflight.Group{}   // ← プロセス共有

func (f *Fetcher) findSetting(ctx context.Context, kind Kind) (*Setting, error) {
    if s, err := f.cache.Get(); err == nil {   // cache hit なら singleflight に入らない
        return s[kind], nil
    }
    // cache miss 時のみ singleflight で DB 取得をまとめる
    v, err, _ := singleflightGroup.Do("GetAll", func() (any, error) {
        settings, err := f.repo.GetAll(ctx)    // ★ リーダーの ctx をそのまま使う
        if err != nil {
            return nil, err
        }
        f.cache.Put(settings, ttl)
        return settings, nil
    })
    ...
}
```

### 何が起きたか

- ある読み取り系 RPC `P` は、分岐条件が成立すると、**マスタ取得**と**外部サービス呼び出し**の 2 つのゴルーチンを**同一の `errgroup` ctx (`egCtx`)** で並行実行していた。
- マスタ取得ゴルーチンが `findSetting` 経由で `singleflightGroup.Do("GetAll")` の**リーダー**になり、`GetAll(egCtx)` を実行。同時に別リクエスト `C`（同じマスタを読む・素のリクエスト ctx を使用）が**フォロワー**として合流した。
- 外部サービス呼び出しゴルーチンがエラーを返す（E2E では fake がキャンセルを注入。本番ならクライアント切断や依存先エラー）。→ `errgroup` が `egCtx` を cancel。
- `GetAll(egCtx)` が `context canceled` になり、`singleflight` がその error をフォロワー `C` にも返す。
- `C` は自分の ctx が健全なのに `Internal`（`context canceled`）で失敗し、それをセットアップに使っていた別テストが落ちた。
- リーダー（`P`）側は errgroup 先着エラー（外部サービスのエラー）だけを返し、マスタ取得のキャンセルは握り潰された。だから `context canceled` は**フォロワーにだけ 1 回**現れた。

補足: `P` が「マスタ取得を行う」分岐に入ったのは、明示設定の無い対象に対して**既定設定がその分岐条件を成立させる**仕様のため。「このテストは特別な設定をしていないから関係ない」という思い込みは誤りだった（既定値の意味を確認すること）。

## 解決策

### 原則: 共有される実処理は「特定呼び出し元のキャンセル」から切り離す

`singleflight`（や類似の「1 本にまとめて結果を共有する」機構）で実行する処理は、**特定の呼び出し元のキャンセルに巻き込まれてはならない**。リーダーの ctx をそのまま渡さず、キャンセル伝播を断つ。

#### 修正（推奨）: `context.WithoutCancel` + 独自 timeout

```go
v, err, _ := singleflightGroup.Do("GetAll", func() (any, error) {
    // 共有される取得は特定呼び出し元のキャンセルから切り離し、独自 timeout で束ねる
    fetchCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), getAllTimeout)
    defer cancel()
    settings, err := f.repo.GetAll(fetchCtx)
    ...
})
```

- `context.WithoutCancel(ctx)`（Go 1.21+）はキャンセル/deadline の伝播だけを断ち、**ctx の値（tid・trace 等）は保持**する。
- deadline も外れるので、暴走防止に**必ず独自 timeout を付ける**。値は対象処理の性質に合わせる（小テーブルの単一リードなら数秒。過大にするとフォロワーの最大待ち時間も伸びる点に注意）。
- この timeout は**フォロワーの最大待ち時間**でもある。健全時レイテンシを十分上回り、かつハングを打ち切れる範囲に取る。

#### 代替: deadline は引き継ぎ「キャンセルだけ」断つ

上流予算を尊重したい場合は、deadline を引き継ぎつつキャンセル伝播だけ断つ:

```go
fetchCtx := context.WithoutCancel(ctx)
if dl, ok := ctx.Deadline(); ok {
    fetchCtx, cancel = context.WithDeadline(fetchCtx, dl)
} else {
    fetchCtx, cancel = context.WithTimeout(fetchCtx, getAllTimeout)
}
defer cancel()
```

ただし singleflight ではリーダーの deadline がフォロワーにも効く軽微な非対称が残る。

### 何を直さないか

- `singleflight` による重複排除そのものは正しい。消すのはキャンセル伝播だけ。
- 実処理が**本物のエラー**（Aborted、実 timeout 等）を返した場合に全フォロワーへ返るのは singleflight の本質であり正しい挙動（次回呼び出しで再取得される）。修正が消すのは「spurious なキャンセル伝播」だけ。

## 予防（PR レビュー時の観点）

`singleflight`・`errgroup`・メモ化・シングルトンなど「**1 プロセス内で処理をまとめる／前回の結果を共有する**」種類のコードでは、以下を設計レビューで明示的に確認する。

| 観点 | 確認内容 |
|---|---|
| 共有機構が使う ctx | `Do`/`Once` の中で**呼び出し元スコープの ctx** をそのまま使っていないか。リーダーのキャンセルが共有結果に波及しないか |
| ctx の由来 | その ctx は `errgroup.WithContext` 由来か、クライアント接続に紐づくか（＝キャンセルされうるか） |
| キャンセルの分離 | 共有実行を `context.WithoutCancel`＋timeout（または deadline 引き継ぎ）で切り離しているか |
| errgroup 兄弟の影響 | 同一 `errgroup` 配下に「エラーを返しうる処理」と「共有機構に入る処理」が混在していないか。前者のエラーが後者を巻き込まないか |
| エラーの観測性 | errgroup 先着エラーで共有機構側のキャンセルが握り潰され、**発生源のログに出ない**構造になっていないか |

「pass しているから問題ない」は通用しない。この種の破綻は cache が cold・高並列・タイミング一致のときにだけ表面化するため、導入 PR 時点では緑になりがち。

## 検出のヒント

| 観点 | 確認内容 |
|---|---|
| 単独実行 | 成功する → 並列実行依存の flaky |
| 失敗リクエストのクライアント | deadline も明示 cancel も無いのにサーバ側が `context canceled` |
| 失敗の分布 | `context canceled` が**フォロワー側に 1 回だけ**現れ、発生源が見えない |
| 実装 | `singleflight.Group.Do(...)` / `sync.Once` などの `fn` 内で呼び出し元 ctx を使用 |
| ctx の由来 | 近傍に `errgroup.WithContext` があり、兄弟処理がエラー/キャンセルしうる |
| 発火条件 | cache miss（TTL 失効直後・cold start）に呼び出しが集中する経路 |

## 教訓

1. **`singleflight` の `fn` に呼び出し元スコープの ctx を渡すな**: リーダーの ctx はリーダー固有の事情でキャンセルされうる。共有される実処理は `context.WithoutCancel`＋timeout でキャンセルから切り離す。
2. **プロセス共有の重複排除は「リクエスト分離」を貫通する**: キャッシュが tid 分離を貫通する（cf. [process-shared-cache-vs-tid-scoped-fake](./process-shared-cache-vs-tid-scoped-fake.md)）のと同様、singleflight は ctx（キャンセル）分離を貫通する。「自分の ctx は健全だから安全」は共有機構の下では成立しない。
3. **`errgroup` 兄弟のエラーは共有 ctx をキャンセルする**: `errgroup.WithContext` の派生 ctx を共有機構へ持ち込むと、無関係な兄弟の失敗がキャンセルとして波及する。伝搬の本質は「何のエラーだったか」ではなく「共有 ctx がキャンセルされたか」。
4. **errgroup 先着エラーが発生源をマスクする**: 発生源側では原因の兄弟エラーが勝ち、キャンセル自体は握り潰される。被害はフォロワー側にだけ出るので、両者の因果を能動的に結びつけて調査する（cf. [errgroup-first-error-status-mapping-gap](./errgroup-first-error-status-mapping-gap.md)）。
5. **`context canceled` を安易に「環境要因」で片付けない**: クライアントに deadline/cancel が無いのにサーバが `context canceled` を返すなら、共有機構経由でのキャンセル伝搬を疑う。「稀だから environment flake」と結論する前に、キャンセルの発生源（リーダー）を特定する。
6. **既定値の意味を確認する**: 「このテストは特別な設定をしていないから無関係」は、既定設定が実は分岐条件を満たす場合に誤りになる（本件では、明示設定の無い対象に対する既定のマスタ設定が「マスタ取得を行う」分岐を成立させていた）。分岐条件は既定値まで遡って確認する。

## 関連

- [プロセス共有キャッシュによる test-scoped fake override の無効化](./process-shared-cache-vs-tid-scoped-fake.md) — 同根（プロセス共有状態がリクエスト分離を貫通）
- [並行取得の「先着エラー」非決定性とエラー→ステータスマッピングの取りこぼし](./errgroup-first-error-status-mapping-gap.md) — errgroup 先着エラーの非決定性・発生源マスキング
- [長時間ランニングテストの重要性](./importance-of-long-running-tests.md) — cache cold × 高並列 × タイミング一致でしか出ない稀なバグの露出

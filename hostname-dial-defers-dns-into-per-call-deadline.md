# ホスト名で張った gRPC 接続の初回 DNS 解決が per-call 締切を食い潰す

## 概要

ローカルの fake / emulator を `localhost:<port>` のような**ホスト名**で dial していると、gRPC の
チャネルは **dns resolver** 経由になる。名前解決は「チャネル生成時」ではなく**最初の RPC のとき**に
実行され、しかもその **per-call 締切の中で**行われる。つまり `context.WithTimeout(ctx, 2*time.Second)`
の 2 秒は「サーバの処理時間」ではなく「**名前解決＋接続確立＋処理**」の合計に対する予算になっている。

この名前解決が一過性に数秒スタックすると、リクエストは**ワイヤに乗る前に**締切を使い切り、
`DeadlineExceeded` として落ちる。接続先は正しく、サーバは健全で、ハンドラには一度も到達していない。
そのため調査は「サーバが遅い」「接続が張れない」「fake が listen していない」の方向へ引っ張られ、
真因（**まだアドレスが 1 つも無い**）から遠ざかる。

決定的な性質は 3 つ。

1. **`grpc.NewClient` は dns スキームが既定**で、`grpc.Dial` / `grpc.DialContext` は **passthrough が既定**。
   同じ `"localhost:50051"` という文字列を渡しても、どちらの API を使ったかで名前解決するかしないかが
   変わる。見た目では区別できない
2. **`localhost` でもネットワーク DNS へクエリが飛ぶ**。gRPC の dns resolver は A/AAAA だけでなく
   service config 用の **TXT レコード `_grpc_config.<host>`** も引く（既定で有効）。TXT は `/etc/hosts`
   に無いので必ず外へ出る
3. **fail-fast（`WaitForReady=false`）は初回解決中には効かない**。チャネルが `TRANSIENT_FAILURE` に
   落ちて初めて即時 `Unavailable` を返す仕様なので、解決待ちの間は黙って待つ。結果として返るのは
   `Unavailable` ではなく `DeadlineExceeded` になり、「接続の問題」ではなく「サーバが応答しない」に見える

対策は締切を延ばすことではない。**IP リテラルで dial して resolver を通らないようにする**。

## 症状

- 単独実行や短時間では再現しない。**長時間の連続ループでごく稀に**落ちる（観測例: 297 周に 1 回）
- 失敗はすべて同一のエラー。`DeadlineExceeded` が上位で retriable に分類され、
  `Unavailable: ... : rpc error: code = DeadlineExceeded desc = context deadline exceeded` の形で出る
- **fake が注入したエラーではなく本物の client 締切**。ここが最初の手掛かりになる
- **そのチャネルを使うテストだけが全滅し、使わないテストは全て通る**。時刻ではなくコードパスで
  綺麗に二分される（観測例: 到達 12 本が全滅 / 非到達 9 本が全 PASS）
- 各テストの所要時間が「per-call 締切 × リトライ回数 + backoff」にぴったり張り付く
- 同一プロセスの**別チャネル**（同じ host:port 宛でも）は同時刻に正常応答していることがある
- 負荷・周回数との相関がない（10 周目でも 297 周目でも起きる。累積劣化ではない）

## 誤診しやすい仮説と、その潰し方

この型は「症状の見た目」が真因から遠いので、順番に潰す必要がある。実際の調査で立てて否定した仮説を
そのまま挙げる。

| 仮説 | 潰し方 |
|---|---|
| 壁時計の後方ステップ（NTP 補正） | `context.WithTimeout` も `go test` の計測も monotonic clock 由来なので**原理的に無関係**。`log show --predicate 'process=="timed"'` で `settimeofday` が 0 件であることも確認 |
| ホスト全体のフリーズ / CPU スターベーション | 同一プロセスの**別チャネルが同時刻に正常往復**していれば否定。`pmset -g log` で sleep/wake が無いことも確認 |
| fake サーバのハンドラがブロックしている | ハンドラのコードを読んで I/O・ログ出力・長いロック保持が無いことを確認。決定的なのは次項の「注入エラーがタイムアウトする」 |
| fake が listen していない / 接続先が違う | **過去に PASS した実行**が、その fake が返す**登録済みデータ**を検証して通っていれば静的な配線ミスは否定できる。デフォルト応答で通ってしまうテストではダメで、fake 固有の値をアサートしているテストを根拠にする |
| 接続が refused されている | refused なら `TRANSIENT_FAILURE` → fail-fast で即 `Unavailable`。締切を丸ごと使い切って `DeadlineExceeded` になっているなら refused ではない。ただし**SDK / gax が `Unavailable` をリトライしていないこと**を確認しないとこの推論は成立しない（リトライがあれば refused でも締切まで粘って化ける） |
| accept が詰まっている（fd 枯渇など） | `ulimit -n` / `kern.maxfilesperproc` を確認。桁が十分なら否定的。なお**accept エラーのログは既定で出ない**（後述）ので「ログに無い」は根拠にならない |
| IPv6（`::1`）への connect が黒穴になっている | grpc-go の pick_first は Happy Eyeballs 実装で **250ms 後に次アドレスを並行試行**する（`connectionDelayInterval`）。`::1` が黒穴でも数秒のハングは作れない |

### 「注入エラーがタイムアウトする」が最強の切り分け

fake に「即座にエラーを返す」応答を仕込んだテストが**その注入エラーではなく本物の
`DeadlineExceeded`** を受け取っていたら、**リクエストはハンドラに到達していない**ことがほぼ直接
証明される。到達していれば即座に注入エラーが返るはずだからだ。

これで「サーバ側の処理」「fake の実装」「ロック」といった候補が一掃され、残るのは
**リクエストが送信される前**の領域だけになる。

### 「ログに無い」が根拠にならない罠

grpc-go の `grpclog` は `GRPC_GO_LOG_SEVERITY_LEVEL` が未設定だと **ERROR のみ stderr に出し、
INFO と WARNING を捨てる**（`grpclog/loggerv2.go`）。

```go
switch logLevel {
case "", "ERROR", "error": // If env is unset, set level to ERROR.
    errorW = os.Stderr
case "WARNING", "warning":
    warningW = os.Stderr
case "INFO", "info":
    infoW = os.Stderr
}
```

subchannel の状態遷移・アドレス試行・accept のバックオフはすべて INFO / WARNING なので、
**既定では失敗した数秒間の中身が一切残らない**。この事実を知らないと「ログに何も出ていないから
ネットワーク層は正常」と誤って結論してしまう。

## 決定的な切り分け: プロセス内の gRPC ログ

OS の DNS ログ（macOS なら `mDNSResponder`）を掘るより、**プロセス内の grpc ログ**のほうが速くて
確実だった。過去の調査では OS 側ログで「`localhost` の A/AAAA 解決は毎回 0 秒」を見て
**「DNS は無罪」と誤結論**していた。実際に遅かったのは同じ `lookup()` の中の TXT 検索側で、
OS ログの見る場所が違っていた。

```
GRPC_GO_LOG_SEVERITY_LEVEL=info
GRPC_GO_LOG_VERBOSITY_LEVEL=2
```

これを**被疑プロセスの環境変数**として渡す（`os.Setenv` では間に合わない。`grpclog` は package の
`init()` で env を読んでロガーを確定するため、`TestMain` の中で設定しても遅い。子プロセスなら
exec 時の env に入れる、同一プロセスなら起動コマンドに前置する）。

読むべきは **`Channel exiting idle mode` と `Resolver state updated` の間隔**。ここが名前解決の所要時間。

```
# 失敗した実行（stall している）
06:09:37  [Channel #13] Channel exiting idle mode        ← 初回 RPC で名前解決を開始
06:09:41  （テストが 2s × 2 tries の締切を使い切って FAIL）
06:09:42  [Channel #13] Resolver state updated            ← 約 5 秒後にやっと解決完了
06:09:42  [Channel #13] ::1 refused → 127.0.0.1 → READY    ← 解決後の接続は同一秒内

# 同じプロセスの別チャネル（正常）
06:09:37  [Channel #12] Channel exiting idle mode
06:09:37  [Channel #12] Resolver state updated            ← 同一秒で完了
06:09:37  [Channel #12] ::1 refused → 127.0.0.1 → READY
```

**チャネルごとに独立した resolver が動く**ので、「同じ host:port 宛の 2 本のうち片方だけが数秒
スタックする」ことが起こりうる。この非対称は「ホスト全体の問題」を否定する証拠にもなる。

修正後は順序が**逆転**する。これが効いていることの確認になる。

```
# IP リテラルにした後
07:45:26  [Channel #9] Resolver state updated             ← Build() 内で同期確定（解決なし）
07:45:26  [Channel #9] picks a new address "127.0.0.1:50869"
07:45:26  [Channel #9] Channel exiting idle mode          ← アドレスが先に確定している
07:45:26  [Channel #9] Channel Connectivity change to READY
```

## 根本原因

### `NewClient` と `DialContext` で既定スキームが違う

これが最大の落とし穴。**同じターゲット文字列でも API が違えば挙動が違う**。

| dial API | 既定スキーム | 名前解決 |
|---|---|---|
| `grpc.NewClient(target, ...)` | **dns** | **する** |
| `grpc.Dial` / `grpc.DialContext(ctx, target, ...)`（deprecated） | **passthrough** | **しない** |

`Dial` → `NewClient` への移行はまさに現在進行中の話なので、**「今まで平気だった箇所が、dial API を
乗り換えた瞬間に DNS 依存になる」**という形で新しく踏みうる。移行の差分レビューでは
「ターゲット文字列がホスト名かどうか」を必ず見る。

同じ理由で、**SDK 経由の接続がどちらを使っているかを実コードで確認する**必要がある。観測例では:

- 明示的に `grpc.NewClient(host)` を呼び、`option.WithGRPCConn(cc)` で SDK に渡している箇所 → **dns（該当）**
- `option.WithEndpoint(host)` だけ渡して SDK 内部に dial させている箇所 → SDK の実装が
  `var dialContext = grpc.DialContext` だったため **passthrough（非該当）**
- emulator 用のクライアントライブラリ → 内部で `passthrough:///` を明示していて **非該当**

**「ホスト名を渡している箇所」を grep しただけでは影響範囲を絞れない。** dial API まで辿ること。

### 名前解決は初回 RPC 時・per-call 締切の中で行われる

`grpc.NewClient` は lazy で、生成時には接続も解決もしない。最初の RPC でチャネルが idle を抜け、
そこで resolver が動く。

```go
func (d *dnsResolver) lookup() (*resolver.State, error) {
    ctx, cancel := context.WithTimeout(d.ctx, ResolvingTimeout) // 既定 30s
    defer cancel()
    srv, srvErr := d.lookupSRV(ctx)      // EnableSRVLookups=false なら no-op
    addrs, hostErr := d.lookupHost(ctx)  // LookupHost("localhost")
    ...
    if d.enableServiceConfig {
        state.ServiceConfig = d.lookupTXT(ctx) // LookupTXT("_grpc_config.localhost")
    }
    return &state, nil
}
```

要点:

- `lookupHost` と `lookupTXT` が**逐次実行**され、**両方終わるまでアドレスを 1 つも返さない**
- `enableServiceConfig` は `envconfig.EnableTXTServiceConfig`（環境変数
  `GRPC_ENABLE_TXT_SERVICE_CONFIG`、**既定 true**）と `!DisableServiceConfig` の AND。つまり
  **既定で TXT 検索が走る**
- `_grpc_config.localhost` は `/etc/hosts` に無いので**必ずネットワーク DNS へ出る**。
  「`localhost` なんてローカルで解決するでしょ」という直感が外れる
- resolver 自身の締切は 30 秒。per-call 締切（2〜3 秒）よりはるかに長いので、**resolver は諦めない。
  先に諦めるのは常に呼び出し側**

### fail-fast は解決中には効かない

`WaitForReady=false`（既定）は「READY でなければ即失敗」ではなく、
「**接続が壊れていると判明したら復旧を待たずに諦める**」という意味。状態別の挙動は次のとおり。

| チャネルの状態 | `WaitForReady=false`（既定 / fail-fast） | `WaitForReady=true` |
|---|---|---|
| READY | 即送信 | 即送信 |
| IDLE | 解決・接続を開始し、**待つ** | 待つ |
| CONNECTING（初回解決中を含む） | **待つ** | 待つ |
| TRANSIENT_FAILURE | **即 `Unavailable`**（原因を添えて） | 待ち続ける |

したがって初回解決中に締切が来ると、返るのは `Unavailable`（＝接続の問題）ではなく
**`DeadlineExceeded`**（＝相手が応答しない、に見える）。**エラーコードが調査を誤った方向へ誘導する**
のがこの型のいやな所。

### 締切の予算がストールより短い経路だけが落ちる

同じストールに晒されていても、落ちるかどうかは**予算**で決まる。

| 経路（例） | per-call 締切 | リトライ | 合計予算 | 5 秒ストール時 |
|---|---|---|---|---|
| A | 2s | 2 回（+200ms backoff） | 4.2s | **落ちる** |
| B | 3s | なし | 3s | **落ちる** |
| C | 10s | 2 回 | 10s 超 | 耐える |
| D | per-call 締切なし | — | 上位の契約次第 | 耐える（遅くなるだけ） |

**アプリ層のリトライは救いにならない。** リトライしても同じチャネルの同じ解決を待つだけなので、
「2s × 2 回」は「2s の試行を 2 回無駄にする」ことにしかならない。むしろ合計予算が
`ResolvingTimeout` に届かない限り、**何回リトライしても全滅する**。

### 発見されやすさが経路ごとに大きく違う

これが「同じ原因なのに片方だけ問題として認識される」理由。

- **予算が短い × 並行テストが多数チャネルの初回 RPC を同時に叩く** → 1 回のストールが
  大量 FAIL として現れ、目立つ。原因調査が始まる
- **予算が長い、または使うテストが 1 本** → 数十周に 1 回、単発のフレークとして現れる。
  「一過性」で片付けられ、リトライヘルパーで**症状だけ吸収**されて原因が記録されない

観測例では、後者のケースが別のドキュメントに「初回接続（TCP + HTTP/2 handshake）のスタック」と
**誤った原因で記録**されていた。前者を追い込んで初めて、両者が同一原因だと分かった。

**単発フレークをリトライで黙らせるとき、原因の仮説を書き残しておくと後で照合できる。**
「初回接続のコスト」のように**測っていない説明**を確定形で書くと、同じ原因の別事象と結びつかなくなる。

### おまけ: ホスト名だと必ず失敗する接続試行が 1 本入る

listener を IPv4 専用（`tcp4`）で bind しているのに `localhost` で接続すると、resolver は
`[::1]` と `127.0.0.1` の**両方**を返し、pick_first は順に試す。`[::1]` 側には誰も listen して
いないので、**毎回必ず `connection refused` が 1 本発生する**。

```
[Channel #12 SubChannel #32] Subchannel picks a new address "[::1]:49841" to connect
addrConn.createTransport failed to connect to {Addr: "[::1]:49841"}. Err: connection refused
[Channel #12 SubChannel #32] Subchannel Connectivity change to TRANSIENT_FAILURE
[Channel #12 SubChannel #33] Subchannel picks a new address "127.0.0.1:49841" to connect
```

正常時は即 refused なのでコストはほぼゼロだが、無駄であることは間違いない。IP リテラル化で
これも同時に消える。

## 解決策

### IP リテラルで dial する

grpc-go の dns builder は、host が IP リテラルなら **`Build()` の中で同期的にアドレスを確定して
`deadResolver` を返す**。

```go
func (b *dnsBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
    host, port, err := parseTarget(target.Endpoint(), defaultPort)
    ...
    // IP address.
    if ipAddr, err := formatIP(host); err == nil {
        addr := []resolver.Address{{Addr: ipAddr + ":" + port}}
        cc.UpdateState(resolver.State{Addresses: addr, Endpoints: []resolver.Endpoint{{Addresses: addr}}})
        return deadResolver{}, nil
    }
    // DNS address (non-IP).
    ...
}
```

これで `lookupHost` / `lookupTXT` が**両方消える**。副次的に `[::1]` 試行も消える。

```go
// 悪い例: ホスト名。dns resolver 経由になり、初回 RPC が名前解決を per-call 締切の中で待つ
fakeHost := "localhost:" + grpcServer.Port()

// 良い例: IP リテラル。resolver を完全に迂回する
fakeHost := "127.0.0.1:" + grpcServer.Port()
```

**listener 側も揃える**とより明快になる（`net.Listen("tcp", "127.0.0.1:0")`）。ループバックだけで
待ち受ければ、外部インタフェースに口を開けない副次的な利点もある。

**なぜ「変更箇所にコメントを書く」ことが重要か**: `localhost` と `127.0.0.1` は等価に見えるので、
後の整合性リファクタで簡単に戻される。理由（dns スキームになる・初回解決が締切の中に入る）を
その場に残す。

### 代替案とその評価

| 案 | 効果 | 評価 |
|---|---|---|
| **IP リテラル** | 解決 2 本が消える + `::1` 試行も消える | **第一候補。** 呼び出し側の 1 行 |
| `passthrough:///host:port` を明示 | 解決が消える | ターゲット文字列を SDK の別オプションにも渡している場合に副作用が出うる |
| `grpc.WithDisableServiceConfig()` | TXT 検索だけ消える | `lookupHost` は残る。本番の dial にも影響するなら要注意 |
| **per-call 締切を延ばす** | ストールが新締切以内なら通る | **非推奨。** 後述 |
| 起動時に conn を warm up（READY まで待つ） | 締切の外で解決・接続を済ませられる | 有効だが本番コードに手が入る。IP 化で足りるなら不要 |

### 締切を延ばすのが筋悪な理由

「fake は in-memory で即答するのだから 2 秒は厳しすぎる」という説明は**誤り**である。ループバックの
プロセス間通信で正常応答は sub-ms オーダーで、2 秒は正常時の 1000 倍以上のマージンがある。
つまり**締切がきつかったのではなく、完全に止まっていた**。

したがって締切を延ばす対処は「厳しすぎる締切を直す」ではなく
「**ストールが新しい締切以内に自然回復することに賭ける**」でしかない。回復時間を測っていないので
効く保証がなく、しかも本番相当の締切値を緩めると**次に本物の遅延が起きたときのシグナルを失う**。

一般化すると: **「原理的に高速なはずの相手 + 固定締切ちょうどで失敗」は、遅延ではなくハングの signature。**
ハングの対処は締切の padding ではなく、ハングする経路自体を消すか、別の経路に乗り換えることになる。
（姉妹ケース: [エミュレータへの書き込みが単一コネクションの転送スタックで稀にタイムアウトする](./single-connection-stall-hangs-fixed-deadline-write.md)
では「新コネクションへの乗り換え」が答えだった。本ケースは「解決経路そのものを消す」が答え）

## タイミング解析の落とし穴: 揃っているのは開始か終了か

「並行して走っていた全リクエストが**同じ瞬間に一斉に**失敗した」という観察は、
**確かめずに書くと逆を言ってしまう**。実際に「終了時刻 − duration」を計算したら、揃っていたのは
**開始（RESUME）時刻**で、終了は 300ms ばらけていた。

| subtest | 終了 | duration | 開始 = 終了 − duration |
|---|---|---|---|
| A | 702.349 | 4.37 | 697.979 |
| B | 702.520 | 4.53 | 697.990 |
| C | 702.653 | 4.66 | 697.993 |

- 開始が 24ms 以内に集中しているのは `t.Parallel()` が一斉に解放するため（当然の結果で情報量はない）
- 終了が 304ms ばらけているのは、seed の重さなどで**その先の RPC を呼び始めた時刻がずれた**分
- **どのリクエストも自分の開始から独立に締切を丸ごと使い切っている**

この計算をすると解釈が変わる。「一瞬の詰まりが解けて全部が同時に失敗した」のではなく、
「**約 4.5 秒間、継続的に使えない状態が続き、窓の途中で始まった試行も例外なく全滅した**」が正しい。
前者なら瞬間的なスケジューリング hiccup を疑うが、後者は**持続的な状態**（＝アドレスが無いまま）を
示す。診断の方向がまるで違う。

なお Go の `testing` は `t.Parallel()` で待たされた時間を duration から**除外**するので、
「終了 − duration」は `=== CONT` された時刻に相当する。`-v` の `=== CONT` の順序と照合して裏を取れる。

### 失敗したテストの集合が何で決まるかを見る

「早く RESUME された順に落ちた」のような時間的説明を立てる前に、**PASS したテストが FAIL したテストの
間に挟まっていないか**を確認する。挟まっていれば時間ではない。

観測例では PASS 群が FAIL 群の CONT 順の間に入っていて、判別条件は純粋に
「**そのチャネルを使うコードパスを通るか**」だった。100% / 0% で二分される集合が見えたら、
それは環境の揺れではなく**特定の依存に紐づいた決定的な故障**である。

## 偽陽性 PASS に注意

`DeadlineExceeded` は上位で `Unavailable` に丸められることが多い。すると
**「`Unavailable` が返ることを確認するテスト」が、注入したエラーではなく本物のストールで PASS する**。

観測例では、fake に `Unavailable` を注入するテストが、本物の `DeadlineExceeded` を受け取りながら
**4.44 秒かけて PASS** していた。所要時間だけが異常のサインで、結果は緑。

- retriable なエラークラスをまとめて 1 コードに畳んでいるなら、**コードだけのアサートは
  「意図した理由で失敗したこと」を検証していない**
- 可能なら**エラーメッセージや副作用まで**見る。少なくとも、注入したエラーのテストが
  **不自然に長い**ことに気づけるようにしておく（所要時間を見る習慣）
- 逆に言えば、**「異常に遅い PASS」は同じ原因の別の顔**。FAIL だけ見ていると取りこぼす

## 一般化・レビューチェックリスト

**dial の書き方**

- [ ] ローカルの fake / emulator を dial するとき、ターゲットは**IP リテラル**か（`127.0.0.1:<port>`）
- [ ] listener 側もループバックに bind して揃えているか
- [ ] ホスト名を使う必要が本当にあるか（証明書の SAN 検証など理由があるなら、その理由をコメントに書く）
- [ ] `localhost` → `127.0.0.1` の変更に**理由コメント**が付いているか（等価に見えるので戻されやすい）

**影響範囲の特定**

- [ ] 「ホスト名を渡している箇所」ではなく「**どの dial API に届くか**」で影響を判定したか
      （`NewClient` = dns / `DialContext` = passthrough）
- [ ] SDK 経由の接続について、SDK 内部が使う dial 関数を実コードで確認したか
- [ ] `Dial` → `NewClient` 移行の差分で、ターゲットがホスト名の箇所が dns 依存に変わっていないか

**締切の設計**

- [ ] per-call 締切が「**名前解決＋接続確立＋処理**」の合計予算であることを踏まえた値か
- [ ] 初回 RPC（＝チャネルの lazy な立ち上げ）が、最も締切の厳しい経路に当たっていないか
- [ ] アプリ層リトライの合計予算が、下位層のタイムアウト（`ResolvingTimeout` = 30s 等）と
      整合しているか。届かないならリトライは全滅するだけ

**診断の手順**

- [ ] `GRPC_GO_LOG_SEVERITY_LEVEL=info` / `GRPC_GO_LOG_VERBOSITY_LEVEL=2` を**プロセスの env**に入れたか
      （`os.Setenv` は `init()` に間に合わない）
- [ ] `Channel exiting idle mode` → `Resolver state updated` の間隔を見たか
- [ ] 「ログに出ていない」を根拠にする前に、そのログが**既定で出力されるレベルか**を確認したか
- [ ] 「注入エラーがタイムアウトしたか」でハンドラ到達可否を切り分けたか
- [ ] 同一プロセスの別チャネルが同時刻に正常だったかを確認したか（ホスト全体説の否定）
- [ ] 「終了 − duration」で RESUME 時刻を出し、**揃っているのは開始か終了か**を確かめたか
- [ ] 失敗集合が**コードパスで二分されていないか**を確認したか
- [ ] OS のネットワークログを掘る前に、**プロセス内の観測**を入れたか（見る場所を間違えやすい）

**記録の作法**

- [ ] 単発フレークをリトライで吸収するとき、**測っていない原因説明を確定形で書いていないか**
      （「初回接続のコスト」等）。仮説は仮説として書く
- [ ] 「異常に遅いが PASS」も同じ原因の症例として拾えているか

## 教訓

1. **`grpc.NewClient` と `grpc.DialContext` は既定スキームが違う。** 同じターゲット文字列で名前解決の
   有無が変わる。移行期のリファクタで静かに DNS 依存が生まれる
2. **per-call 締切は名前解決を含む。** 「サーバの処理時間」の予算だと思っていると、解決の遅延が
   `DeadlineExceeded` になって「サーバが遅い」と誤読する
3. **`localhost` でもネットワーク DNS へ出る。** service config 用の TXT レコード検索は既定で有効で、
   `/etc/hosts` では解決できない
4. **fail-fast は初回解決中には効かない。** 返るエラーコードが「接続の問題」を示さないので、
   コードだけを見ると誤誘導される
5. **固定締切ちょうどで失敗 + 原理的に高速な相手 = 遅延ではなくハング。** 対処は締切の padding ではなく、
   ハングする経路を消すか乗り換えること
6. **「一斉に同時刻で失敗」は計算して確かめる。** 揃っているのが開始なのか終了なのかで、
   「瞬間的な hiccup」か「持続的な状態」かという診断の向きが反転する
7. **失敗集合がコードパスで 100%/0% に二分されるなら、それは環境の揺れではない。** 特定の依存に
   紐づいた決定的な故障を、低頻度ゆえにフレークと呼んでいるだけ
8. **プロセス内の観測 > OS ログの考古学。** OS 側の DNS ログで「A/AAAA は速い」を見て「DNS は無罪」と
   誤結論した。同じ関数の中の別のクエリが遅かった。当事者のプロセスに喋らせるほうが速く確実
9. **測っていない原因説明を確定形で記録すると、同一原因の別事象と結びつかなくなる。** 別サービスで
   「初回接続のスタック」として記録・回避されていた事象が、実はこれと同一原因だった

## 関連ドキュメント

- [エミュレータへの書き込みが単一コネクションの転送スタックで稀にタイムアウトする](./single-connection-stall-hangs-fixed-deadline-write.md)
  — 「固定締切ちょうどで失敗するのはハング」の姉妹ケース。あちらは「新コネクションへ乗り換える」が答え
- [deadline 到達時のステータスコードは client timer と server 応答のレースで揺れる](./deadline-arrival-status-race-client-vs-server.md)
  — 締切まわりのエラーコードが揺れる別の型
- [壁時計の後方ステップ（NTP補正）による順序依存テストのフレーキー失敗](./wallclock-backward-step-by-ntp.md)
  — 本ケースで最初に疑って否定した仮説。`timed` ログの追跡手順
- [テストランナーの結果キャッシュがヒットして「テストの副作用」が実行されない](./test-cache-hit-skips-server-startup-side-effect.md)
  — 「テストコードを読んでも原因が無い」型の別例
- [長時間ランニングテストの重要性](./importance-of-long-running-tests.md)
  — 297 周に 1 回の事象は、長時間ループでしか見つからない

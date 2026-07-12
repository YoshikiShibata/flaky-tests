# キャッシュ／singleflight のキーが「結果が依存する入力」を取りこぼす

## 概要

プロセス共有のインメモリキャッシュや singleflight（重複呼び出しの集約）で、**キャッシュ／inflight のキーが、返す結果が実際に依存する入力の一部しか含んでいない**と、その取りこぼした入力の値が異なる呼び出し元同士で**結果が混線**する。

典型は、結果が「カテゴリ（例: 種別 enum）」だけでなく「時刻 `now`」にも依存するのに、キーには種別しか使っていないケース。あるいは locale・tenant・権限スコープなど、**出力を左右するが暗黙になっている次元**をキーから落としているケース。

とりわけ singleflight の **waiter（フォロワー）パスは、leader が計算した結果を「自分の入力に合っているか」を再確認せずそのまま受け取る**ため、leader と waiter で取りこぼした入力の値が食い違うと、waiter は自分の入力では本来ありえない結果を掴む。

これは「プロセス共有の状態が、リクエスト／テスト単位の分離を貫通する」という
[プロセス共有キャッシュによる test-scoped fake override の無効化](./process-shared-cache-vs-tid-scoped-fake.md) ／
[プロセス共有 singleflight がリーダーの ctx キャンセルを無関係なフォロワーへ伝搬させる](./singleflight-shared-ctx-cancellation-propagation.md)
と同根の構造問題である。あちらは「ctx スコープ／tid スコープの貫通」、こちらは「**入力の一次元がキーから抜けている**」という貫通。

## 症状

- 対象テストを単独実行すると常に成功する。全 E2E を並列実行したときだけ、間欠的に失敗する（数十回に 1 回など）。
- 失敗テストは「自分で用意したデータが、自分のリクエストで見えない（取りこぼされる／古い値になる）」という形で落ちる。セットアップは正しく、直前に書いたはずのデータが応答に無い。
- 失敗するのは、**共有キャッシュの取りこぼし次元が、他の並行テストと大きく違う値**を使っているテスト（例: debug-time で `now` を遠い未来／過去に pin しているテスト）。
- cache miss が起きて集約に呼び出しが集中する一瞬（TTL 失効直後・cold start・バケット切り替わり）でのみ顕在化する。
- **本番では観測されない**ことが多い（後述）。

## 根本原因

### 構造的要因

1. **キャッシュ／inflight がプロセス共有**。キーはリクエスト／テスト／ユーザーを含まない共有キー。
2. **キーが結果の依存項の一部しか含まない**。「同じ種別なら同じ結果」という暗黙の前提が、実際には「同じ種別 **かつ** 同じ時刻帯（／locale／tenant …）なら同じ結果」だった。
3. **singleflight の waiter が leader の結果を無検証で受け取る**。waiter 自身の入力（`now` 等）と leader の入力が一致するかを照合しない。
4. （キャッシュ側）**TTL 判定が取りこぼした次元を跨ぐ**。「leader が入力 A で埋めた値」を、入力 B の呼び出し元が TTL 内という理由だけで読んでしまう。

### 発生メカニズム（時刻 `now` を例に）

結果が `[開始, 終了)` のような **`now` 依存の絞り込み**で決まるのに、キャッシュ／inflight のキーが種別のみだとする。

1. cache miss が起き、複数の呼び出しがほぼ同時に集約へ入る。1 本が **leader**、残りは **waiter**。
2. leader は自分の `now = R` でクエリし、「時刻 R で有効な集合」を得る。
3. waiter は自分の `now = D`（`R` と大きく異なる）で待ち合流していたが、**leader の「R 時点の集合」をそのまま受け取る**。
4. waiter 側で結果を `now = D` で再フィルタする段があると、R 時点の集合は D では全滅し、**本来 D で有効なはずの要素が nil／空**になる。
5. あるいは再フィルタが無ければ、waiter は **他人（R）向けの結果を自分（D）の結果として表示**してしまう。

キャッシュ経由でも同型: leader が `now = R` で埋めたエントリを、`now = D` の呼び出しが「TTL 未失効」だけを見て読み、R 向けの結果を掴む。

### 誤結果になる条件（「遠さ」ではなく「跨ぎ」）

誤結果になるのは **`|leaderの入力 − waiterの入力| ≥ 結果が変わる粒度`** のとき。時刻の例なら、有効期間ウィンドウ幅を跨ぐ差があれば十分で、「遠い未来」である必要はない（過去方向でも、ウィンドウが狭ければ小さな差でも）発火する。ウィンドウを広げるのは安全域を広げるだけで**根本解決にならない**。

### なぜ本番では起きにくいか（テスト固有性の切り分け）

多くの場合この不具合は **E2E 特有**で本番実害が無い。理由:

- 本番では取りこぼした次元（`now` 等）の値が、並行呼び出し間で**ごく僅かしか違わない**（実時計なら数ミリ秒〜数秒）。その差の中で結果が変わる確率は極小。
- さらに、そのキャッシュが元々 **TTL 相当の staleness を製品仕様として許容**している場合、waiter が受け取る「数秒前の leader の結果」は、その許容範囲に**必ず収まる**。
- E2E は debug-time でこの次元を**数十年ずらせる**（かつ他テストは実時計を使う）ため、本番では起こり得ない「大きく食い違う入力の同時集約」を人工的に作ってしまう。

→ 「本番バグか、E2E 特有か」を必ず切り分ける。切り分けの軸は「**取りこぼした次元の値が、本番の並行呼び出し間でどれだけ乖離しうるか**」。ただし E2E 特有だとしても、キーが結果の依存項を落としているのは**設計上の欠陥**であり、修正はプロダクトコード側で行うのが筋（テスト側の小細工では再発の芽が残る）。

### なぜ稀か / なぜ発見が遅れるか

集約に入るのは cache miss の一瞬だけ。発火には「cold な瞬間に、乖離した入力の呼び出しが**たまたま同時に leader/waiter として合流**する」タイミング一致が要る。warm 時は集約に入らないため、**長時間・多数回・高並列**で初めて観測される（[長時間ランニングテストの重要性](./importance-of-long-running-tests.md) と同じ構図）。

## 具体例

### 状況

種別 `kind` ごとに「時刻 `now` で有効な要素集合」を返す処理が、プロセス共有キャッシュ＋ inflight 集約でまとめられていた。キーは `kind` のみ（`lookup` は説明用の汎用メソッド名）。

```go
// キーが kind だけ。now を含まない。
cache    map[Kind]*entry              // entry{ items, expiresAt }
inflight map[Kind]*inflightEntry

func (r *Repo) lookup(ctx context.Context, kind Kind, now time.Time) ([]*Item, error) {
    r.mu.Lock()
    if e, ok := r.cache[kind]; ok && now.Before(e.expiresAt) { // ★ now が leader と違っても TTL 内なら返す
        defer r.mu.Unlock()
        return e.items, nil
    }
    if f, ok := r.inflight[kind]; ok {                          // ★ waiter は leader の結果を無検証で受け取る
        r.mu.Unlock()
        <-f.done
        return f.items, f.err
    }
    f := &inflightEntry{done: make(chan struct{})}
    r.inflight[kind] = f
    r.mu.Unlock()
    // leader: SELECT ... WHERE Start <= @now AND End > @now  ← 結果は now 依存
    items, err := r.query(ctx, kind, now)
    // ... cache[kind] = {items, expiresAt: now.Add(TTL)}; close(f.done)
    return items, err
}
```

呼び出し側は返った集合を **さらに `now` で再フィルタ**していた（`now ∈ [Start, End)`）。

### 何が起きたか

- 多数の並行テストが実時刻（例: 2026 年）でこの処理を叩き、`lookup` へ同時到達。
- 1 つのテストだけ debug-time で `now = 2070` を pin し、2070 に有効な要素を作って検証。
- 実時刻テストが inflight leader になった瞬間、2070 テストが waiter として合流 → **「2026 で有効な集合（2070 の要素を含まない）」を受け取る**。
- 呼び出し側の再フィルタが `now = 2070` で全滅 → 期待した要素が `nil` → 間欠 FAIL。

### 対策（キーを結果の依存項に一致させる）

`now` を TTL 幅の**バケット**に切り捨て、キャッシュ／inflight のキーに含める。

```go
func bucket(now time.Time) int64 { return now.Truncate(TTL).UnixNano() } // location 非依存に UnixNano

type inflightKey struct { kind Kind; bucket int64 }

// cache は kind ごと 1 エントリのまま（bucket 不一致なら miss 扱いで上書き）→ リークしない
if e, ok := r.cache[kind]; ok && e.bucket == bucket(now) { ... }   // 同一バケットのみヒット
// inflight は (kind, bucket) 単位 → 乖離した now は別 leader で独立 fetch。完了時 delete でリークなし
```

効果と不変性:

- **同一バケットの呼び出しのみ**集約／キャッシュを共有し、乖離した `now` は必ず独立に fetch する。
- 本番挙動は不変: 実時計の連続呼び出しは同一バケットに入るので、1 クエリ集約と TTL キャッシュの効果は維持。staleness も従来同等（≤ TTL）。
- 副作用はバケット境界での再 fetch が稀に増える程度で無視できる。
- cache は種別ごと 1 エントリ（バケット不一致で上書き）に保つと**メモリリークしない**。inflight は完了時 delete でリークしない。

（別解: waiter が leader の入力を記録・照合し、乖離が大きければ集約に乗らず自前 fetch する。より複雑で、通常はバケットキー化で十分。）

## 一般化・チェックリスト

プロセス共有のキャッシュ／singleflight／memoize を見たら:

- [ ] **結果が依存する入力を全部列挙**したか？ そのうち**キーに含めていない**ものはないか（`now`／時間帯・locale・tenant・通貨・権限スコープ・API バージョン等）。
- [ ] singleflight の **waiter が受け取る leader の結果**は、waiter 自身の入力でも妥当か？ waiter は入力を照合しているか、それとも無検証で受け取っているか。
- [ ] キャッシュの **TTL 判定が、キーから落とした次元を跨いで**ヒットしていないか。
- [ ] 「同じ X なら同じ結果」という**暗黙の前提**は本当か。実は「同じ X **かつ** 同じ Y」ではないか。
- [ ] キーに次元を足したとき、**エントリが際限なく増えてリーク**しないか（長寿命キャッシュは 1 スロット上書き、短命 inflight は完了時削除）。
- [ ] 時刻を map キーにするなら `time.Time` そのものではなく **`UnixNano()` 等の絶対値**を使う（`time.Time` は monotonic/location を含み比較・キーが不安定）。
- [ ] E2E がこの次元を**大きくずらす**（debug-time で遠い時刻を pin する等）設計なら、共有キャッシュ越しに他テストと混線しうると疑う。

### 教訓

- **キャッシュ／集約のキーは「結果が依存する入力の全体」でなければならない**。一部でも欠けると、欠けた次元で呼び出し元が異なるときに結果が混線する。singleflight の waiter は「他人の入力で計算された結果」を掴みうる。
- 間欠 E2E 失敗は、**プロダクトの共有状態設計の欠陥を検出している**ことがある。ただし「本番でも壊れるか」は、取りこぼした次元が本番の並行呼び出し間でどれだけ乖離しうるかで切り分ける。E2E 特有でも修正はプロダクトコード側（キー設計）で行う。

## 関連

- [プロセス共有キャッシュによる test-scoped fake override の無効化](./process-shared-cache-vs-tid-scoped-fake.md) — 共有キャッシュが tid スコープの分離を貫通する同根事例
- [プロセス共有 singleflight がリーダーの ctx キャンセルを無関係なフォロワーへ伝搬させる](./singleflight-shared-ctx-cancellation-propagation.md) — singleflight の leader/waiter 共有に起因する別種の貫通（ctx キャンセル）
- [共有フィクスチャプールの再利用による残留状態の干渉](./shared-fixture-pool-residual-state.md) — 共有状態がテスト分離を壊す別型
- [長時間ランニングテストの重要性](./importance-of-long-running-tests.md) — cache miss の一瞬でしか出ない不具合は長時間・高並列でのみ観測される

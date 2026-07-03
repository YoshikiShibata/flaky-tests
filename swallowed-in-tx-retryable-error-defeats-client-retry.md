# トランザクション内の retryable エラー（Aborted）握りつぶしがクライアント自動リトライを無効化する（Emulator では隠れ 本番相当エンジンで顕在化）

## 概要

読み書きトランザクションのコールバック内で、DB 操作（read / write）が返した
**retryable なエラー（Spanner の `Aborted` 等）を、ログして `return nil` で握りつぶす**と、
クライアントライブラリの**自動リトライが発火せず**、abort 済みのトランザクションのまま
commit に進んでしまう。

このとき commit がどう振る舞うかは **DB エンジンによって異なる**:

- **緩いエンジン（テスト用エミュレータ）**: この commit をサイレントに受理する
  （成功したように見えるが実際には何も起きない／自己回復する）ため、**バグが隠れる**。
- **厳密なエンジン（本番相当）**: この commit を**非 retryable なエラーで拒否**する
  （例: multiplexed session で「precommit token が無い」＝ `InvalidArgument`）ため、
  **致命的に失敗する**。

結果として「差分と無関係な best-effort 後処理が、本番相当エンジンでだけ稀に落ちる」
フレークになる。**重要な点として、この件では本番相当エンジンの方が正しく（fail loud）、
エミュレータの方が不正（silent）**であり、「エミュレータは単に緩いだけ」という通常の構図の
反転になっている。

## 前提知識：クライアント管理のトランザクションリトライ

Spanner 等の read-write トランザクションは、**コールバックが retryable エラー（`Aborted`）を
「返す」ことで、クライアントがトランザクション全体を自動的にリトライ**する設計になっている。

```
コールバックが Aborted を return  → クライアントが自動リトライ（正しい経路）
コールバックが nil を return       → commit に進む
```

したがって、コールバック内で `Aborted` を握りつぶして `nil` を返すと、**リトライ機構を
バイパスして abort 済みトランザクションを commit しようとする**。これがバグの本体。

## 発生条件

1. `RunInTransaction` 等のコールバック内で DB 操作（read / write）を実行している
2. その操作のエラーを `if err != nil { log; return nil }` で握りつぶしている
   （「best-effort だから失敗しても続行」のつもり）
3. 握りつぶす対象に **retryable な `Aborted`** が含まれ得る
4. commit が前提条件を厳密に要求するエンジン（例: multiplexed session で precommit token を
   要求する本番相当エンジン）で実行される

条件 2 が本体。「best-effort で無視」の意図で全エラーを握りつぶすと、無視してよいエラー
（NotFound 等）だけでなく、**リトライすべき retryable エラーまで握りつぶしてしまう**。

## なぜ発見が遅れるか

### 緩いエンジン（エミュレータ）が握りつぶしを隠す

read が `Aborted` を返す → コールバックが握りつぶして nil → commit。このとき:

- 緩いエンジンは、abort 済み／前提を満たさない commit を**サイレントに受理**することがある
  （エラーを返さない、または retryable エラーを返してクライアントが自己回復する）。
- さらに、エミュレータには **multiplexed session 系の commit をサイレントに受理／握りつぶす
  既知のバグ**（実装が本番の厳密さを再現できていない）が存在するバージョンがある。この場合、
  握りつぶしバグは**二重に隠蔽**される。

いずれにせよ**エラーが表面化しない**ため、テストは通り続け、握りつぶしバグに気付けない。

> **参考：本事例での具体バージョン**
> - 使用していたエミュレータ: `cloud-spanner-emulator` **`1.5.47`**（この multiplexed session の
>   commit サイレント受理バグを含む）。
> - 修正されたバージョン: **`1.5.50`**（`cloud-spanner-emulator` issue #282
>   "Write operations (Apply/Update) silently fail under concurrent load" として修正）。
> - つまり `1.5.47 < 1.5.50` のため、当時のエミュレータは未修正で、握りつぶしバグを
>   サイレントに隠していた。エミュレータを `1.5.50` 以降に更新すると、この種のバグを
>   エミュレータ側でも fail loud で捕まえやすくなる。

### 厳密なエンジン（本番相当）が暴く

同じシーケンスで、厳密なエンジンは commit を**非 retryable なエラーで拒否**する
（例: 「A Commit request ... must specify a precommit token」= `InvalidArgument`）。
`InvalidArgument` は retryable ではないためクライアントはリトライせず、致命的に失敗する。

### 原因が埋もれやすい

落ちるのは「差分と無関係な best-effort 後処理」であり、しかも本番相当エンジンに
載せ替えたときだけ非決定的に出る。**「エミュレータで出ないから環境要因」で片付けると
本質を見失う**。

## 具体例（一般化）

### 握りつぶしていたコード

```go
// トランザクション後の best-effort な更新（外部から tx 外で呼ばれ、自前で tx を張る）
func someBestEffortUpdate(ctx context.Context, id string) error {
    return tx.RunInTransaction(ctx, func(ctx context.Context) error {
        entity, err := repo.Find(ctx, id) // ★ tx 内の read
        if err != nil {
            log.Error(err, "...")          // (Aborted を含む) エラーをログし
            return nil                     // ★ 握りつぶして nil → commit に進む
        }
        if entity.NeedsUpdate() {
            entity.Update(...)
            if err := repo.Put(ctx, entity); err != nil {
                return err
            }
        }
        return nil
    })
}
```

### 観測される失敗（一般化）

本番相当エンジンで、read の `Aborted` の直後に commit が非 retryable エラー
（例: precommit token 欠如の `InvalidArgument`）で失敗する。同一トランザクション内の
連続した操作であることは、ログの request_id / 連番の requestID で確認できる。

## 対策

**コールバック内では DB 操作のエラーを握りつぶさず return する。** best-effort にしたい場合は、
握りつぶしを **トランザクションの外側**、または **期待される特定の sentinel に限定**する。

```go
func someBestEffortUpdate(ctx context.Context, id string) error {
    return tx.RunInTransaction(ctx, func(ctx context.Context) error {
        entity, err := repo.Find(ctx, id)
        if err != nil {
            // 「存在しなければスキップ」のような期待される sentinel だけを best-effort 扱いにする
            if errors.Is(err, ErrNotFound) {
                log.Info("skipping: not found ...")
                return nil
            }
            // それ以外 (retryable な Aborted 等) は握りつぶさず返す
            return liberrorsWrap(err, "...")
        }
        ...
        return nil
    })
}
```

ポイント:

- **リトライは「コールバックが `Aborted` を返す」ことで発火**する。これはクライアント側の
  ロジックであり、commit の挙動（エンジン依存）に依存しない。したがってこの直し方は
  **エンジン非依存で正しく動く**（緩いエンジンで commit がどう振る舞おうと関係ない）。
- 「特定 sentinel だけスキップ、それ以外は伝播」にすると、best-effort の意図
  （例: 対象が無ければ何もしない）を保ちつつ、retryable エラーは正しくリトライされる。
- 全エラーを best-effort で無視したい場合でも、**握りつぶしは tx の外**で行う
  （`RunInTransaction` の戻り値をログして無視）。コールバック内で `return nil` しない。

## チェックリスト

以下が揃うと、このパターンのバグが潜在している可能性があります。

- [ ] `RunInTransaction` 等のコールバック内で `if err != nil { ...; return nil }` している
- [ ] 握りつぶす対象が DB 操作の戻りエラー（retryable な `Aborted` を含み得る）である
- [ ] best-effort のスキップが特定 sentinel（NotFound 等）に限定されておらず、全エラーを握りつぶしている
- [ ] commit が前提を厳密に要求するエンジン（multiplexed session 等）と、緩いエミュレータで
      挙動が食い違う
- [ ] テスト用エミュレータのバージョンが、既知の「commit をサイレントに受理する」系バグの
      修正を含んでいない（緩さが握りつぶしバグを隠していないか）

## 教訓

1. **トランザクションのコールバック内で、DB 操作のエラーを握りつぶして `nil` を返さない。**
   retryable エラーはコールバックから返し、クライアントの自動リトライに委ねる。
2. **best-effort の無視は「tx の外」または「特定 sentinel 限定」で行う。** 全エラーの
   握りつぶしは、無視してよいエラーと共にリトライすべきエラーまで潰す。
3. **「エミュレータで出ないが本番相当エンジンで出る」= エミュレータが正しくエラーを
   出せていない可能性を疑う。** 本番相当エンジンの厳密さが実バグを暴くことがあり、
   その場合エミュレータ側（バージョン／既知バグ）こそが不正なこともある
   （＝実エンジンで E2E を回す価値そのもの）。
4. リトライは「コールバックがエラーを返す」ことで発火させ、**commit の挙動に依存させない**。
   これがエンジン非依存に正しく動く鍵。

## 関連

- [テスト用DBエミュレータと本番DBエンジンの挙動差で表面化する Flaky Test](./emulator-vs-production-db-divergence.md)
  — 行順序・分離レベルなど、エミュレータの緩いデータ意味論が非決定性を隠すケース。
- [エミュレータの直列化に隠されていた並列サブテストのデータ競合](./data-race-masked-by-emulator-serialization.md)
  — 並行性バグ（データ競合・トランザクション内 `time.Now()` の時刻衝突）が直列化で隠れるケース。

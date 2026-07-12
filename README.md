# flaky-tests

E2E テスト・並列実行環境で発生した flaky test の事例と対策パターン集。

## 索引

### テスト分離・並列実行

- [全データ更新ジョブによるテスト間干渉](./global-data-modification-interference.md) — 全行 `InsertOrUpdate` するジョブテストが、他テストのセットアップデータを上書きする
- [グローバル集計値の before/after 比較が並行書き込みで揺れる](./global-aggregate-before-after-assertion.md) — 「操作が状態を変えないこと」を全件横断 API の件数 before/after 比較で検証していたため、無関係な並行テストが 2 スナップショットの間に集計対象行を commit すると ±1 のズレで、実行のたびに成功したり失敗したりする（flaky）。誰も他人のデータを壊していないのにアサーションのスコープが広すぎて壊れる型。対象 API が正当にグローバルなのでロックではなく「観測を自テストのエンティティに絞る＋件数でなく ID 集合の同一性で比較」で決定化
- [テストデータのハードコード値とドメイン定数の不一致](./hardcoded-domain-values-in-test-data.md) — 不正値が DB に残り、別テストの全件取得 API で変換エラーになる
- [プロセス共有キャッシュによる test-scoped fake override の無効化](./process-shared-cache-vs-tid-scoped-fake.md) — サービス側のキャッシュが tid スコープ fake を貫通してしまう
- [プロセス共有 singleflight がリーダーの ctx キャンセルを無関係なフォロワーへ伝搬させる](./singleflight-shared-ctx-cancellation-propagation.md) — `Do` の中で呼び出し元スコープの ctx を使うと、あるリクエスト（errgroup 兄弟エラー／クライアント切断）のキャンセルが、同じキーを待つ無関係なリクエストを `context canceled` で巻き込む。発生源は errgroup 先着エラーに握り潰され、被害はフォロワー側にだけ表面化する
- [キャッシュ／singleflight のキーが「結果が依存する入力」を取りこぼす](./coalescing-cache-key-omits-result-dependency.md) — 共有キャッシュ／inflight のキーが種別 enum しか含まず、結果が依存する `now`（時刻）を落としていると、debug-time で `now` を大きくずらしたテストが waiter として合流したとき leader（実時刻）の結果を無検証で受け取り、自分の入力では本来ありえない結果（期待要素の nil／空）を掴む。「キーは結果が依存する入力の全体でなければならない」型。`now` を TTL 幅のバケットに切り捨ててキーへ含めることで、本番挙動を変えずに決定化。本番実害は無いが設計欠陥なのでプロダクト側で修正
- [逐次ループの並行化におけるループキャリー依存性の見落とし](./loop-carried-dependency-in-concurrent-test-setup.md) — 並行ループ化で暗黙の実行順序依存が壊れる
- [セットアップ・クリーンアップの順序が引き起こす孤立参照](./setup-cleanup-order-orphan-reference.md) — Child→Parent の順で setup すると、LIFO cleanup により「Child は存在するが参照先 Parent が存在しない」状態が発生する
- [並行取得の「先着エラー」非決定性とエラー→ステータスマッピングの取りこぼし](./errgroup-first-error-status-mapping-gap.md) — errgroup が返す先着エラーが非決定的で、マッピング層が一部センチネルを取りこぼすと観測ステータスが揺れる
- [共有フィクスチャプールの再利用による残留状態の干渉](./shared-fixture-pool-residual-state.md) — 再利用される共有プールが過去テストの副作用を蓄積したエンティティを配り、「初期状態」前提のテストが偽陰性で flaky になる
- [テストヘルパーの「直接DB→API/usecase経由」化が生む隠れた共有依存とロック抜け](./helper-api-routing-hidden-shared-dependency.md) — ヘルパーを本物のリクエスト処理経由に変えると、その処理が読む共有マスタへの依存とロック義務が暗黙に増え、削除窓と競合してフレークになる
- [状態共有する逐次サブテストの途中失敗によるカスケード失敗](./stateful-sequential-subtests-cascade-on-midstep-failure.md) — `t.Run` で区切った状態累積の逐次シナリオで、途中 1 ステップのフレーク失敗が後続の off-by-one 連鎖失敗を量産し原因が埋もれる。`t.Fatalf` は親ループを止めないため、失敗時 `break` で打ち切る

### 時刻・境界

- [時刻境界で失敗するフレーキーテストの原因と対策](./flaky-test-time-boundary-patterns.md) — クライアント／サーバーの時刻計算ずれ
- [現在時刻から導出される値のハードコードによる Flaky Test](./hardcoded-season-from-current-time.md) — 季節・期間区分などをハードコードしている
- [月末境界による Flaky Test：月別設定値の切り替わり](./month_boundary_master_data_switch.md) — 月をまたぐとマスターデータが切り替わる
- [業務時間境界をまたぐテストデータによるフレーキーテスト](./time-frame-boundary-crossing.md) — 相対時刻がドメイン固有の日付境界をまたぐ
- [テストデータの有効期限が次の暦境界とちょうど一致するパターン](./data-lifetime-aligned-to-day-boundary.md) — 暦境界アンカーの StartTime と整数倍 duration の組み合わせで、テスト実行中の境界跨ぎでデータが消失する
- [壁時計の後方ステップ（NTP補正）による順序依存テストのフレーキー失敗](./wallclock-backward-step-by-ntp.md) — NTP補正で `time.Now()` が後方ジャンプし、作成時刻DESCソートが作成順前提と食い違う。macOS `timed` ログでの追跡方法付き

### ロック・並行制御

- [E2E テストの並列実行を高速化する MultiLock の設計と実装](./e2e_test_lock_optimization_multilock.md) — Hold and Wait 解消のためのロック設計
- [ロック競合による性能劣化](./lock_contention_performance_degradation.md) — 一部ロック保持中に他ロック待機でカスケードブロック
- [排他制御の削除・縮小リファクタの非対称リスク](./lock-reduction-asymmetry.md) — 「不要に見える」保護削除の見落とし
- [ユニークキー INSERT でも「最新を選ぶグローバルセレクタ」を奪うと writer になる](./unique-insert-hijacks-global-latest-selector.md) — 自分専用の行しか触らなくても、`ORDER BY version DESC LIMIT 1` 等の選択結果を差し替えるなら Exclusive が必要。Shared writer は 2 回読みの並行 reader と干渉する

### クエリ・データ意味論

- [DB クエリの LIMIT と後フィルタリングによるフレーキーテスト](./db_query_limit_with_post_filtering.md) — 小さい LIMIT がフィルタ後 0 件を引き起こす
- [keyset ページネーションの tie-breaker 欠如による行の取りこぼし](./keyset-pagination-missing-tiebreaker.md) — 非一意なソートキー（`CreateTime`）に strict `<` だけで cursor を組むと、同値行がページ境界で脱落する。間欠的な E2E 失敗が、テストのフレークではなくサービス側のページネーション実装バグを検出していた事例

### DB 実装依存（エミュレータ vs 本番エンジン）

- [テスト用DBエミュレータと本番DBエンジンの挙動差で表面化する flaky](./emulator-vs-production-db-divergence.md) — エミュレータの緩い「行順序」「分離レベル」が潜在的な非決定性を覆い隠し、本番相当 DB でのみ flaky に落ちる。(1) ORDER BY 無しクエリ × 固定順アサーション、(2) 並行入金に依存した残高がタイミングで枯渇、の2事例
- [エミュレータの直列化に隠されていた並列サブテストのデータ競合](./data-race-masked-by-emulator-serialization.md) — `t.Parallel()` サブテストが外側スコープの共有変数（`err`）に両方から書き込むデータ競合。エミュレータの RW トランザクション直列化により 2 処理が同時並行にならず、周囲の同期（atomic カウンタ・gRPC）と相まって happens-before で順序付いていたため `-race` でも長期間検出されず、本番相当エンジン（Omni）の並行実行で初めて顕在化。1 件の競合で多数のテストが連鎖 FAIL する
- [トランザクション内の retryable エラー（Aborted）握りつぶしがクライアント自動リトライを無効化する](./swallowed-in-tx-retryable-error-defeats-client-retry.md) — tx コールバック内で read の `Aborted` をログして `return nil` で握りつぶすと、クライアントの自動リトライが発火せず abort 済み tx のまま commit に進む。緩いエミュレータはこの commit をサイレントに受理してバグを隠し、厳密な本番相当エンジンは非 retryable エラー（precommit token 欠如）で拒否して致命的に失敗する。本番相当エンジンが正しく／エミュレータが不正という構図の反転が学びの核
- [トランザクション再試行で使い回すモデルへの ID 書き戻しが INSERT を UPDATE に転落させる](./tx-retry-model-mutation-flips-insert-to-update.md) — クライアント管理リトライはコールバックを丸ごと再実行するが、捕捉した可変オブジェクトはリセットされない。「保存時に採番 ID を引数へ書き戻す」副作用が Aborted 再試行をまたいで残り、2 回目は ID 有→ UPDATE に転落して `row not found`。低負荷では通り高負荷で間欠失敗する。冪等化（試行冒頭で状態復元）で解消

### テスト基盤・コンテナ実行環境

- [エミュレータへの書き込みが単一コネクションの転送スタックで稀にタイムアウトする](./single-connection-stall-hangs-fixed-deadline-write.md) — ホスト→コンテナの `localhost` 接続が通るユーザーランドのポートフォワード中継が、高い接続生成レートのピークで**特定の 1 本のフローだけ**を宙吊りにし、書き込みの `Close` が固定タイムアウトで `context deadline exceeded` に落ちる。エミュレータ自体は健全（同時刻に毎秒数百件を処理）でサーバ側ログが「1 本だけ孤立して滞留」を裏付ける。先に fsync を**仮説**として memory backend 化し高頻度は見かけ上収まったが、その仮説は未検証のまま低頻度で再発。「インメモリなのに固定タイムアウト張り付き」の矛盾が、証拠で裏を取れる真因（転送ハング）に気づく鍵に——**妥当そうな修正で症状が消えても真因を証明したことにはならない**という教訓付き。対策は単発タイムアウト延長ではなく**短タイムアウト × 新コネクションでのリトライ**。多重化 gRPC 系が同症状を起こしにくい理由も整理

### テスト戦略全般

- [長時間ランニングテストの重要性](./importance-of-long-running-tests.md) — テストは「バグの不在」を証明できない

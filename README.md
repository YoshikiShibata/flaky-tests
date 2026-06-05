# flaky-tests

E2E テスト・並列実行環境で発生した flaky test の事例と対策パターン集。

## 索引

### テスト分離・並列実行

- [全データ更新ジョブによるテスト間干渉](./global-data-modification-interference.md) — 全行 `InsertOrUpdate` するジョブテストが、他テストのセットアップデータを上書きする
- [テストデータのハードコード値とドメイン定数の不一致](./hardcoded-domain-values-in-test-data.md) — 不正値が DB に残り、別テストの全件取得 API で変換エラーになる
- [プロセス共有キャッシュによる test-scoped fake override の無効化](./process-shared-cache-vs-tid-scoped-fake.md) — サービス側のキャッシュが tid スコープ fake を貫通してしまう
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

### クエリ・データ意味論

- [DB クエリの LIMIT と後フィルタリングによるフレーキーテスト](./db_query_limit_with_post_filtering.md) — 小さい LIMIT がフィルタ後 0 件を引き起こす

### テスト戦略全般

- [長時間ランニングテストの重要性](./importance-of-long-running-tests.md) — テストは「バグの不在」を証明できない

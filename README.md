# flaky-tests

E2E テスト・並列実行環境で発生した flaky test の事例と対策パターン集。

## 索引

### テスト分離・並列実行

- [全データ更新ジョブによるテスト間干渉](./global-data-modification-interference.md) — 全行 `InsertOrUpdate` するジョブテストが、他テストのセットアップデータを上書きする
- [テストデータのハードコード値とドメイン定数の不一致](./hardcoded-domain-values-in-test-data.md) — 不正値が DB に残り、別テストの全件取得 API で変換エラーになる
- [プロセス共有キャッシュによる test-scoped fake override の無効化](./process-shared-cache-vs-tid-scoped-fake.md) — サービス側のキャッシュが tid スコープ fake を貫通してしまう
- [逐次ループの並行化におけるループキャリー依存性の見落とし](./loop-carried-dependency-in-concurrent-test-setup.md) — 並行ループ化で暗黙の実行順序依存が壊れる
- [セットアップ・クリーンアップの順序が引き起こす孤立参照](./setup-cleanup-order-orphan-reference.md) — Child→Parent の順で setup すると、LIFO cleanup により「Child は存在するが参照先 Parent が存在しない」状態が発生する

### 時刻・境界

- [時刻境界で失敗するフレーキーテストの原因と対策](./flaky-test-time-boundary-patterns.md) — クライアント／サーバーの時刻計算ずれ
- [現在時刻から導出される値のハードコードによる Flaky Test](./hardcoded-season-from-current-time.md) — 季節・期間区分などをハードコードしている
- [月末境界による Flaky Test：月別設定値の切り替わり](./month_boundary_master_data_switch.md) — 月をまたぐとマスターデータが切り替わる
- [業務時間境界をまたぐテストデータによるフレーキーテスト](./time-frame-boundary-crossing.md) — 相対時刻がドメイン固有の日付境界をまたぐ

### ロック・並行制御

- [E2E テストの並列実行を高速化する MultiLock の設計と実装](./e2e_test_lock_optimization_multilock.md) — Hold and Wait 解消のためのロック設計
- [ロック競合による性能劣化](./lock_contention_performance_degradation.md) — 一部ロック保持中に他ロック待機でカスケードブロック
- [排他制御の削除・縮小リファクタの非対称リスク](./lock-reduction-asymmetry.md) — 「不要に見える」保護削除の見落とし

### クエリ・データ意味論

- [DB クエリの LIMIT と後フィルタリングによるフレーキーテスト](./db_query_limit_with_post_filtering.md) — 小さい LIMIT がフィルタ後 0 件を引き起こす

### テスト戦略全般

- [長時間ランニングテストの重要性](./importance-of-long-running-tests.md) — テストは「バグの不在」を証明できない

# E2Eテストの並列実行を高速化するMultiLockの設計と実装

## はじめに

並列実行されるE2Eテストでは、共有リソース（データベースのマスタテーブルなど）へのアクセスを制御するためにロック機構が必要です。しかし、従来のロック取得方式には「Hold and Wait」パターンによる性能劣化という問題がありました。

本記事では、この問題を解決するために設計・実装した「MultiLock」について解説します。

## 問題：Hold and Waitによる連鎖的な待機

### 従来の方式

従来の方式では、デッドロックを防ぐためにリソースの順序付けを行い、テストが複数のロックを必要とする場合は決められた順序で1つずつ取得していました。

```go
func TestExample(t *testing.T) {
    t.Parallel()
    lock.ResourceA(t)  // ResourceAのロックを取得（順序: A → B）
    lock.ResourceB(t)  // ResourceBのロックを取得（ResourceAを保持したまま待機）
    // テスト実行
}
```

### 発生した問題

この方式では、以下のような連鎖的な待機が発生します。

```
時刻 0:00 - テスト開始
┌─────────────────────────────────────────────────────────────────┐
│ TestA: ResourceA取得 → ResourceB待ち（ResourceA保持中）          │
│ TestB: ResourceB取得 → ResourceC待ち（ResourceB保持中）          │
│ TestC: ResourceCを使用中                                        │
│ TestD: ResourceAだけ必要だが、TestAが保持中なので待機            │
│ TestE: ResourceBだけ必要だが、TestBが保持中なので待機            │
└─────────────────────────────────────────────────────────────────┘
```

**結果：**
- TestDはResourceAだけ必要なのに、TestAがResourceBを待っている間ずっと待機
- TestEも同様にTestBに不要にブロックされる
- 本来並列実行できるテストが直列化され、実行時間が大幅に増加
- 実際の環境では**59分以上の遅延**が観測された

### これはデッドロックではない

この問題は「デッドロック」ではありません。循環待機（Circular Wait）は発生しておらず、いずれ全てのテストは完了します。しかし、不要な待機の連鎖により、テスト全体の実行時間が大幅に増加してしまいます。

この問題は、デッドロックの4条件（Coffman条件）のうち「Hold and Wait（保持と待機）」に起因する性能劣化です。

## 解決策：MultiLockによるアトミックなロック取得

### 設計方針

MultiLockは以下の方針で設計しました。

1. **All or Nothing**: 必要なロックを全て同時に取得するか、1つも取得しない
2. **待機中はロックを保持しない**: 他のテストをブロックしない
3. **既存コードとの互換性**: 段階的な移行が可能

### 実装

```go
// MultiLockの使用例
func TestExample(t *testing.T) {
    t.Parallel()
    lock.NewMultiLock(t,
        lock.Exclusive(LockResourceA),
        lock.Exclusive(LockResourceB),
    ).AcquireAll(t)
    // テスト実行
    // t.Cleanup()で自動的に解放される
}
```

**内部動作：**

```go
func (ml *MultiLock) AcquireAll(t *testing.T) {
    m.mu.Lock()
    defer m.mu.Unlock()

    // 全てのロックが取得可能になるまで待機
    // この間、何もロックを保持しない
    for !m.canAcquireAll(ml.requests) {
        m.cond.Wait()
    }

    // 全てのロックを一度に取得
    m.doAcquireAll(ml.requests)

    // テスト終了時に自動解放
    t.Cleanup(func() { ml.ReleaseAll(t) })
}
```

### 改善後の動作

```
時刻 0:00 - テスト開始
┌─────────────────────────────────────────────────────────────────┐
│ TestA: ResourceA+B待ち（何も保持していない）                     │
│ TestB: ResourceB+C待ち（何も保持していない）                     │
│ TestC: ResourceCを使用中                                        │
│ TestD: ResourceAだけ必要 → すぐに取得して実行開始！              │
│ TestE: ResourceBだけ必要 → すぐに取得して実行開始！              │
└─────────────────────────────────────────────────────────────────┘

時刻 0:05 - TestC, TestD, TestE完了
┌─────────────────────────────────────────────────────────────────┐
│ TestA: ResourceA+Bが空いた → 取得して実行開始                   │
│ TestB: ResourceB+C待ち                                          │
└─────────────────────────────────────────────────────────────────┘

時刻 0:10 - TestA完了
┌─────────────────────────────────────────────────────────────────┐
│ TestB: ResourceB+Cが空いた → 取得して実行開始                   │
└─────────────────────────────────────────────────────────────────┘

時刻 0:15 - 全テスト完了
```

## Shared Lock（共有ロック）のサポート

MultiLockは排他ロック（Exclusive）だけでなく、共有ロック（Shared）もサポートしています。

```go
// 読み取り専用のテスト（複数が同時実行可能）
lock.NewMultiLock(t, lock.Shared(LockMissionMaster)).AcquireAll(t)

// 書き込みを行うテスト（排他的に実行）
lock.NewMultiLock(t, lock.Exclusive(LockMissionMaster)).AcquireAll(t)
```

**ロック互換性：**

| 保持中 \ 要求 | Shared | Exclusive |
|--------------|--------|-----------|
| なし          | ✅ 取得可 | ✅ 取得可 |
| Shared       | ✅ 取得可 | ❌ 待機   |
| Exclusive    | ❌ 待機  | ❌ 待機   |

## 設計上のトレードオフ：Writer Starvation

### 選択した方針

MultiLockでは、Writer Priority（書き込み優先）を採用していません。これは意図的な設計判断です。

**理由：**
複数ロックをアトミックに取得する場合、全てのロックに対してWriter Priorityを適用すると、まだ取得していないロックに対してもReaderをブロックしてしまいます。これでは「Hold and Wait」と同様の問題が発生します。

### Writer Starvationの影響は限定的

E2Eテストの文脈では、Writer Starvationの影響は限定的です。

1. **テスト数は有限**: 無限にReaderが来続けることはない
2. **各テストは短時間で完了**: 待ち時間には上限がある
3. **並列度は制限されている**: 同時実行数には上限がある

**比較：**

| 問題 | 発生条件 | 影響 |
|------|---------|------|
| Hold and Wait（修正前） | 複数ロック取得時 | 連鎖的遅延、59分以上 |
| Writer Starvation | 多数のShared vs 少数のExclusive | 一部テストが少し待つ程度 |

## 性能特性

### ロック数と並列実行効率

MultiLockでは、必要なロック数が少ないテストほど並列実行されやすくなります。

```
ロック1個のテスト: 条件が緩い → 並列実行されやすい
ロック5個のテスト: 条件が厳しい → 他のテストの隙間で実行
```

これは自然で望ましい特性です。ほとんどのテストは1〜2個のロックで済むため、大多数のテストがスムーズに並列実行されます。

### 期待される効果

| 観点 | Before | After |
|------|--------|-------|
| ロック保持時間 | 待機中も保持 | 実行中のみ |
| 他テストへの影響 | 不要にブロック | 最小限 |
| 並列実行効率 | 低い | 高い |
| 最悪ケースの遅延 | 59分以上 | 数分程度 |

## 実装のポイント

### 1. ロック名の定数化

ハードコードされた文字列ではなく、定数を使用することでタイプミスを防止します。

```go
// locks.go
const (
    LockMissionMaster    = "mission_master"
    LockCoinAwardSetting = "coin_award_setting"
    // ...
)

// テストファイル
lock.NewMultiLock(t, lock.Exclusive(LockMissionMaster)).AcquireAll(t)
```

### 2. 自動解放

`t.Cleanup()`を使用して、テスト終了時に自動的にロックを解放します。手動での解放忘れを防止できます。

### 3. 冪等なRelease

`ReleaseAll()`は複数回呼び出しても安全です。`t.Cleanup()`との組み合わせで問題が発生しません。

```go
func (ml *MultiLock) ReleaseAll(t *testing.T) {
    if !ml.acquired {
        return // 既に解放済み、何もしない
    }
    // 解放処理
}
```

## まとめ

MultiLockの導入により、以下の改善を実現しました。

1. **Hold and Wait問題の解消**: 待機中にロックを保持しないことで、連鎖的な遅延を防止
2. **並列実行効率の向上**: ロック数が少ないテストが優先的に実行される
3. **シンプルなAPI**: `NewMultiLock().AcquireAll()`の1行で複数ロックを安全に取得
4. **自動リソース管理**: `t.Cleanup()`による自動解放

E2Eテストの実行時間短縮は、開発者の生産性向上に直結します。ロック機構の設計を見直すことで、大幅な改善が可能であることを示せたと考えています。

## 参考

- Coffman条件（デッドロックの4条件）
- Go の `sync.Cond` を使った条件変数の実装
- Reader-Writer Lock のセマンティクス

---

James Cutajar著、柴田 芳樹 訳『Go言語で学ぶ並行プログラミング』（インプレス、2024年）「第11章 デッドロックの回避」（p.260）では、Coffman条件について詳しく解説されています。

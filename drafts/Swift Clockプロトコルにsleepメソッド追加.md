# Swift Clockプロトコルにsleepメソッド追加

- [Swift Clockプロトコルにsleepメソッド追加](#swift-clockプロトコルにsleepメソッド追加)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
    - [提案内容](#提案内容)
  - [ソースの互換性、ABIの安定性への影響、APIのレジリエンスへの影響](#ソースの互換性abiの安定性への影響apiのレジリエンスへの影響)
  - [検討した代替案](#検討した代替案)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [実装](#実装)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

Swift 5.7で導入された`Clock`プロトコルは、将来のある瞬間まで中断する方法を提供するが、一定期間スリープする方法を提供し ていない。これは、ある瞬間までスリープする方法と一定期間スリープする方法の両方を提供する`Task`のstaticな`sleep`メソッドとは異なる。

このAPIの不均衡は、すべての`Clock`に`sleep(for:)`メソッドを追加するのに十分な理由かもしれませんが、本当の問題は`Clock`の存在型を扱うときに発生する。`Instant`のassociated typeは完全に消去され、`Duration`だけがprimary associated typeとして保存されるため、`Instant`を扱うAPIはすべて、存在型にはアクセスできない。つまり、存在型の`Clock`の`sleep(until:)`を実行することはできず、実際に何かをすることはできない。


## 内容

### 動機

存在型は、機能に依存性を注入する便利な方法を提供し、ある種の依存性を本番で使用し、別の種類の依存性をテストで使用することができるようにする。この最も典型的なバージョンはAPIクライアントである。本番ではAPIクライアントに実際のネットワークリクエストをさせたいが、テストではモックデータを返すだけにしておきたい。

`Clock`の現在の設計では、`ContinuousClock`を使用するために、ある機能に`Clock`の存在型を注入することはできないが、テストでは他の制御可能な`Clock`を使用することができる。

例えば、5秒待ったらウェルカムメッセージを表示する機能のロジックに`ObservableObject`を用意したとする。

```swift
class FeatureModel: ObservableObject {
  @Published var message: String?
  func onAppear() async {
    do {
      try await Task.sleep(until: .now.advanced(by: .seconds(5)))
      self.message = "Welcome!"
    } catch {}
  }
}
```

もし、このテストを書いた場合、テストスイートはアサーションを行う前に実際5秒経過するのを待つしかないだろう。

```swift
let model = FeatureModel()

XCTAssertEqual(model.message, nil)
await model.onAppear() // 5秒待つ
XCTAssertEqual(model.message, "Welcome!")
```

これは、テストを書かない人にも影響する。もし、あなたの機能をXcodeのプレビューに入れたら、ウェルカムメッセージが表示されるまでに、5秒経過するのを待たなければならないdならないでしょうだろう。つまり、そのメッセージのスタイリングを素早く反復修正することができない。

このような問題を解決するには、グローバルで制御不能な`Task.sleep`に手を出さず、代わりに機能を`Clock`に注入する。そして、それは一般的に存在型を使って行われるが、残念ながらそれはうまくいかない。

```swift
class FeatureModel: ObservableObject {
  @Published var message: String?
  let clock: any Clock<Duration>

  func onAppear() async {
    do {
      try await self.clock.sleep(until: self.clock.now.advanced(by: .seconds(5))) // 🛑
      self.message = "Welcome!"
    } catch {}
  }
}
```

なぜなら、`Instant`は完全に消去されており、`.now`にアクセスしてそれを進める方法がないからである。

同様の理由で、`Clock`の存在型で`Task.sleep(until:clock:)`を呼び出すことはできない。

```swift
try await Task.sleep(until: self.clock.now.advanced(by: .seconds(5)), clock: self.clock) // 🛑
```

代わりに必要なのは、`Clock`の`sleep(for:)`メソッドで、ある瞬間までスリープするのではなく、ある期間だけスリープすることができるようにする。

```swift
class FeatureModel: ObservableObject {
  @Published var message: String?
  let clock: any Clock<Duration>

  func onAppear() async {
    do {
      try await self.clock.sleep(for: .seconds(5)) // ✅
      self.message = "Welcome!"
    } catch {}
  }
}
```

`Clock`の`sleep(for:)`メソッドがないと、機能の中で`Clock`の存在型を使うことができない、ジェネリックを導入せざるを得ない。

### 提案内容

`Clock`プロトコルに1つの拡張メソッドを追加する。

```swift
extension Task where Success == Never, Failure == Never {
  /// 連続したクロックで、与えられた時間だけ現在のタスクを一時停止する。
  ///
  /// 時間終了前にタスクがキャンセルされた場合、`CancellationError`をスローする
  ///
  /// この関数は、現在処理を実行しているスレッドをブロックしない。
  ///
  ///       try await Task.sleep(for: .seconds(3))
  ///
  /// - Parameter duration: 待機時間
  public static func sleep<C: Clock>(
    for duration: C.Duration,
    tolerance: C.Duration? = nil,
    clock: C = ContinuousClock()
  ) async throws {
    try await sleep(until: clock.now.advanced(by: duration), tolerance: tolerance, clock: clock)
  }
}
```

これにより、ある瞬間までスリープするのではなく、`Clock`を使ってある期間だけスリープすることができるようになる。

さらに、`clock.sleep(for:)`と`Task.sleep(for:)`のAPIを類似させるために、`Task.sleep(for:)`にも`clock`と`tolerance`の引数を追加する。

```swift
extension Task where Success == Never, Failure == Never {
  /// 連続したクロックで、与えられた時間だけ現在のタスクを一時停止する。
  ///
  /// 時間終了前にタスクがキャンセルされた場合、`CancellationError`をスローする
  ///
  /// この関数は、現在処理を実行しているスレッドをブロックしない。
  ///
  ///       try await Task.sleep(for: .seconds(3))
  ///
  /// - Parameter duration: 待機時間
  public static func sleep<C: Clock>(
    for duration: C.Duration,
    tolerance: C.Duration? = nil,
    clock: C = ContinuousClock()
  ) async throws {
    try await sleep(until: clock.now.advanced(by: duration), tolerance: tolerance, clock: clock)
  }
}
```

また、`Task.sleep(until:)`の`clock`引数にはデフォルト値を追加する。

```swift
extension Task where Success == Never, Failure == Never {
  public static func sleep<C: Clock>(
    until deadline: C.Instant,
    tolerance: C.Instant.Duration? = nil,
    clock: C = ContinuousClock()
  ) async throws {
    try await clock.sleep(until: deadline, tolerance: tolerance)
  }
}
```

## ソースの互換性、ABIの安定性への影響、APIのレジリエンスへの影響

これは追加であるため、互換性、安定性、レジリエンスの問題はないはず。

## 検討した代替案

このメソッドを標準ライブラリに追加せず、人々が自分で定義することも可能なので、追加しないことも可能。


## 参考リンク

### Forums

- [](https://forums.swift.org/t/pitch-clock-sleep-for/60376)
- [SE-0374: Add `sleep(for:)` to Clock](https://forums.swift.org/t/se-0374-add-sleep-for-to-clock/60787)
- [[Accepted] SE-0374: Add `sleep(for:)` to Clock](https://forums.swift.org/t/accepted-se-0374-add-sleep-for-to-clock/62148)

### 実装

- [apple/swift#61222](https://github.com/apple/swift/pull/61222)

### プロポーザルドキュメント

- [Add sleep(for:) to Clock](https://github.com/apple/swift-evolution/blob/main/proposals/0374-clock-sleep-for.md)
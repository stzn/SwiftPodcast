# Swift 新しい時間の概念(Clock, Instant and Duration)

- [Swift 新しい時間の概念(Clock, Instant and Duration)](#swift-新しい時間の概念clock-instant-and-duration)
  - [概要](#概要)
  - [内容](#内容)
    - [時間の3つの概念](#時間の3つの概念)
    - [現状の問題点](#現状の問題点)
      - [Foundation](#foundation)
      - [Dispatch](#dispatch)
    - [用語説明](#用語説明)
    - [統一的な時間の概念を定義するために必要なこと](#統一的な時間の概念を定義するために必要なこと)
    - [今回の提案の範囲](#今回の提案の範囲)
    - [提案の詳細](#提案の詳細)
      - [Clockプロトコル](#clockプロトコル)
      - [Instantプロトコル](#instantプロトコル)
      - [DurationProtocolプロトコル](#durationprotocolプロトコル)
      - [Duration構造体](#duration構造体)
    - [Task](#task)
    - [Clockプロトコルの具体的な実装](#clockプロトコルの具体的な実装)
      - [ContinuousClock構造体](#continuousclock構造体)
      - [SuspendingClock構造体](#suspendingclock構造体)
    - [Standard Library以外のClock実装](#standard-library以外のclock実装)
      - [UTCClock構造体](#utcclock構造体)
    - [Swift ConcurrencyでClockの利用](#swift-concurrencyでclockの利用)
  - [参考リンク](#参考リンク)

## 概要

現在、Swiftで時間を扱う方法はたくさんあるが、統一的で汎用的な方法が存在しない。今回新たに時間の概念を定義し、それを使って時間を扱うAPIを導入したい。

## 内容

### 時間の3つの概念

1. 「今」の概念を提供するアイテムと一定時間後に起動する方法(= 時刻 clockと呼ばれる)
2. ある瞬間を表す概念(= 瞬間 instantと呼ばれる)
3. 経過時間を測定する概念(= 期間 durationと呼ばれる)

### 現状の問題点

この3つを表すAPIや型がたくさんある。同時並行処理を扱う場合、これらのすべての概念への統一されたアクセサがあれば、sleep、他の時間的概念に必要なプリミティブを構築するのに役立つ。

APIの例

#### Foundation

- `Date`: UTC時間の2001年1月1日を起点としてある瞬間を構築する
- `TimeInterval`: 2つの瞬間の期間を秒数で表す

#### Dispatch

- `DispatchTime`: システムが起動してからの時間
- `DispatchWallTime`: プログラムの開始から終了までを計測した時間
- `DispatchTimeInterval`: 秒、ミリ秒、マイクロ秒、ナノ秒の値

### 用語説明

エポック: 時刻の基準点
realtime, wall-clock time: プログラムの開始から終了までを計測した時間。NTPなどで変更されることがある。
uptime time: どのくらいマシンが起動しているかの経過秒数
monotonic clock(モノトニック時間): 何らかのエポックからの単調増加な時刻。決して後戻りすることがない。システム・コール関数などで使う。

### 統一的な時間の概念を定義するために必要なこと

時間を扱う標準的な方法を定義するには、時計の計測方法を特定の概念に限定することが重要な場合に、その境界を維持するようにする必要がある。

例: realtimeのdeadlineを受け取るAPIにmonotonicを渡せてはならない。

一方でこういった特定の情報を知らなくても時間のAPIを扱えるようにもする必要がある。バランスが重要。

同様に、すべての実装は、複数のオペレーティングシステムのバックエンド（Linux、Darwin、Windowsなど）をサポートするのに十分な堅牢性とパフォーマンスを備えている必要があるが、一般的なユースケースに対応するのに十分に簡単に扱える必要もある。

パフォーマンスという意味では、測定の実行に大きなオーバーヘッドを発生させることなく、関数の実行パフォーマンスの測定などのタスクを実行するのに十分なパフォーマンスを備えている必要がある。

### 今回の提案の範囲

プロセスでの作業のスケジューリングに使用される時間に焦点を当てる。この目的で最も有用なclockは、プロセスを実行しているマシンが開始されてからの時間を計算する単純なlocal clockになる。

### 提案の詳細

#### Clockプロトコル

2つの基本的な要件がある

- 与えられた瞬間の後にウェークアップ※する方法(※スリープ状態をレディー状態にする)
- 今の概念を生成する方法

```swift
public protocol Clock: Sendable {
  associatedtype Instant: InstantProtocol
  
  var now: Instant { get }
  
  func sleep(until deadline: Instant, tolerance: Instant.Duration?) async throws 

  var minimumResolution: Instant.Duration { get }
  
  func measure(_ work: () async throws -> Void) reasync rethrows -> Duration
}
```

`associatedtype`に、ある瞬間を表す`Instant`プロトコル(後述)に準拠した型を持つので、`Clock`と`Instant`は紐づいている。そのため、ある`Clock`のインスタンスと他の`Clock`の`Instant`を比較することはできず、`Clock`間での境界が維持されるようになっている。ただし、既存のAPIと新しいAPI間のやり取りをスムーズにするためや使いやすさという点で、2つの`Instant`間の期間はどんな`Clock`でも比較することができるようになっている(`Duration`という構造体を返す)。ただし使い方を間違えないように注意する必要がある。

時間の精度を調整することもできる。ある`Clock`はナノ秒単位の精度を持つし、ある`Clock`はマイクロ秒単位での精度を持つ。デフォルトは1ナノ秒。経過時間が指定している精度以下の場合は0になる。

与えられた瞬間の後にウェークアップするには`sleep`メソッドを使う。ただし、あまり頻繁にやりすぎるとシステムのリソースを使いすぎる可能性もあるので、システムが起動するまでの遅延が許容できる範囲を示す`tolerance`を指定できる。

#### Instantプロトコル

「今」の概念と、与えられた期間分の時間を進める方法のみ定義する必要がある。比較可能(`Comparable`)かつ`Key`として保存できる(`Hashable`)必要もある。

このプロトコルを使って期間を定義することで、適切な値を保持できるストレージの型を使用すること加えて、`Instant`に紐づく`Clock`も型安全になるメカニズムを提供する。

`Instant`が有用な主な理由は、組み合わせて使えること。例えば、ある関数で`Instant`型のdeadlineを引数に受け取って、そのdeadlineをさらに他の関数に渡した場合でも、お互いに関数の実行がそのdeadlineに影響しないようにできている。つまり時間の経過は一定である。  
例えば、URLリクエストのタイムアウトは、コネクションタイムアウトやデータ送信時のタイムアウト、レスポンスを受け取るまでのタイムアウトなどがあるが、これらはそれぞれのステップで計測される一方で、指定した最終期限(deadline)は影響を受けずに進む。

乗算できてしまうのは不適切のため、`Stridable`には準拠せず独自のオペレータを定義している。

```swift
public protocol InstantProtocol: Comparable, Hashable, Sendable {
  associatedtype Duration: DurationProtocol
  func advanced(by duration: Duration) -> Self
  func duration(to other: Self) -> Duration
}

extension InstantProtocol {
  public static func + (_ lhs: Self, _ rhs: Duration) -> Self
  public static func - (_ lhs: Self, _ rhs: Duration) -> Self
  
  public static func += (_ lhs: inout Self, _ rhs: Duration)
  public static func -= (_ lhs: inout Self, _ rhs: Duration)
  
  public static func - (_ lhs: Self, _ rhs: Self) -> Duration
}
```

#### DurationProtocolプロトコル

「期間」を表す概念を表したもの。比較可能(`Comparable`)かつ加算できる(`AdditiveArithmetic`)。

```swift
public protocol DurationProtocol: Comparable, AdditiveArithmetic, Sendable {
  static func / (_ lhs: Self, _ rhs: Int) -> Self
  static func /= (_ lhs: inout Self, _ rhs: Int)
  static func * (_ lhs: Self, _ rhs: Int) -> Self
  static func *= (_ lhs: inout Self, _ rhs: Int)
  
  static func / (_ lhs: Self, _ rhs: Self) -> Double
}
```

#### Duration構造体

「期間」を表す`DurationProtocol`の具体的な実装。全ての`Clock`で使われている。参照時点からの過去と未来をナノ秒と秒で表現する。人間の測定値（または機械で測定された精度）から構築できるが、カレンダーの概念(日、月、年など)は持たない。

直列化可能(`Codable`)、比較可能(`Comparable`)、`Key`として保存できる(`Hashable`)、加算できる(`AdditiveArithmetic`)ようになっている。

ストレージは、除算時な適切な丸め精度（露出よりも高い精度を保存する可能性が高い）と、潜在的に合理的な`Instant`に十分な範囲を内部に保証しなければならない。これは、秒とナノ秒を損失のないスケールで+/-数千年の全範囲にまたがって保存することができることを指す。

```swift
public struct Duration: Sendable {
  public var seconds: Int64 { get }
  public var nanoseconds: Int64 { get }
}

extension Duration {
  public static func seconds<T: BinaryInteger>(_ seconds: T) -> Duration
  public static func seconds(_ seconds: Double) -> Duration
  public static func milliseconds<T: BinaryInteger>(_ milliseconds: T) -> Duration
  public static func milliseconds(_ milliseconds: Double) -> Duration
  public static func microseconds<T: BinaryInteger>(_ microseconds: T) -> Duration
  public static func microseconds(_ microseconds: Double) -> Duration
  public static func nanoseconds<T: BinaryInteger>(_ value: T) -> Duration
}

extension Duration: Codable { }
extension Duration: Hashable { }
extension Duration: Equatable { }
extension Duration: Comparable { }
extension Duration: AdditiveArithmetic { }

extension Duration {
  public static func / (_ lhs: Duration, _ rhs: Double) -> Duration
  public static func /= (_ lhs: inout Duration, _ rhs: Double)
  public static func / (_ lhs: Duration, _ rhs: Int) -> Duration
  public static func /= (_ lhs: inout Duration, _ rhs: Int)
  public static func / (_ lhs: Duration, _ rhs: Duration) -> Double
  public static func * (_ lhs: Duration, _ rhs: Double) -> Duration
  public static func *= (_ lhs: inout Duration, _ rhs: Double)
  public static func * (_ lhs: Duration, _ rhs: Int) -> Duration
  public static func *= (_ lhs: inout Duration, _ rhs: Int)
}

extension Duration: DurationProtocol { }
```

### Task

既存の`Task.sleep`メソッドはsleepの特定の動作はないが、内部では、Darwinではcontinuous clock、Linuxではsuspending clockを使用している。

これらのAPIはdeprecatedになって、下記のような新しいAPIを提供する。

```swift
extension Task {
  @available(*, deprecated, renamed: "Task.sleep(for:)")
  public static func sleep(_ duration: UInt64) async
  
  @available(*, deprecated, renamed: "Task.sleep(for:)")
  public static func sleep(nanoseconds duration: UInt64) async throws
  
  public static func sleep(for: Duration) async throws
  
  public static func sleep<C: Clock>(until deadline: C.Instant, tolerance: C.Instant.Duration? = nil, clock: C) async throws
}
```

※ 新しいAPIの`Task.sleep(for:)`は`ContinuousClock`を使っている。

### Clockプロトコルの具体的な実装

広く一般的に使われるものはStandard Libraryで実装している。

#### ContinuousClock構造体

`Instant`がローカルのみで使われ、マシンがスリープ中も正確に時間を進めたい場合に有効な`Clock`。

Darwinの場合はmonotonic clockに由来する時間を参照する。Linuxの場合はuptime clockに由来する時間

```swift

public struct ContinuousClock {
  public init()
  
  public static var now: Instant { get }
}

extension ContinuousClock: Clock {
  public struct Instant {
    public static var now: ContinuousClock.Instant { get }
  }

  public var now: Instant { get }
  public var minimumResolution: Duration { get }
  public func sleep(until deadline: Instant, tolerance: Duration? = nil) async throws
}

extension ContinuousClock.Instant: InstantProtocol {
  public func advanced(by duration: Duration) -> ContinuousClock.Instant
  public func duration(to other: ContinuousClock.Instant) -> Duration
}

extension Clock where Self == ContinuousClock {
  public static var continuous: ContinuousClock { get }
}
```

#### SuspendingClock構造体

ローカルプロセススコープまたはクロスマシンスコープのインスタントが適切でない場合に有効な`Clock`。マシンの起動時からの時間(スリープ中は止まる)を表す。マシンの範囲内でプロセス間の通信を行うことができる。

Darwinの場合はmonotonic clockに由来する時間を参照する。Linuxの場合はuptime clockに由来する時間を参照する。

```swift
public struct SuspendingClock {
  public init()
  
  public static var now: Instant { get }
}

extension SuspendingClock: Clock {
  public struct Instant {
    public static var now: SuspendingClock.Instant { get }
  }

  public var minimumResolution: Duration { get }
  public func sleep(until deadline: Instant, tolerance: Duration? = nil) async throws
}

extension SuspendingClock.Instant: InstantProtocol {
  public func advanced(by duration: Duration) -> SuspendingClock.Instant
  public func duration(to other: SuspendingClock.Instant) -> Duration
}

extension Clock where Self == SuspendingClock {
  public static var suspending: SuspendingClock { get }
}
```

### Standard Library以外のClock実装

#### UTCClock構造体

UTCベースのカレンダーの概念を伴ったFoundation内の`Clock`

- 起動時間が現在のUTC時間で調節される
- `Instant`の型は`Date`
- `Date`から`ContinuousClock`や`SuspendingClock`への変換方法を提供する(逆も提供されている)
- `Date`の実装は、システムクロックで定義されている2001年1月1日以降の秒数で処理されるため、ネットワーク時間（または手動）の更新は、システムクロックのスキューに応じて、その時点を順方向または逆方向にシフトする可能性がある
- 閏秒を指定するメソッドもある

```swift

public struct UTCClock {
  public init()
  
  public static var now: Date { get }
}

extension UTCClock: Clock {
  public var minimumResolution: Duration { get }
  public func sleep(until deadline: Date, tolerance: Duration? = nil) async throws
}

extension Date {
  public func leapSeconds(to other: Date) -> Duration
  public init(_ instant: ContinuousClock.Instant)
  public init(_ instant: SuspendingClock.Instant)
}

extension ContinuousClock.Instant {
  public init?(_ instant: Date)
}

extension SuspendingClock.Instant {
  public init?(_ instant: Date)
}

extension Date: InstantProtocol {
  public func advanced(by duration: Duration) -> Date
  public func duration(to other: Date) -> Duration
}

extension Clock where Self == UTCClock {
  public static var utc: UTCClock { get }
}
```

※ 雑談
`Date`をStandard Libraryに移動する話が当初あったが、特定のカレンダーに制限されるのであまりよろしくないという結論からFoundationにとどまることに。
カレンダーが関わらない時間の概念に`Date`という名前は不適切ではないかという話がスレッドに上がっていたが、UTCClockと組み合わせることで適切ではないかと現在は考えられている。

### Swift ConcurrencyでClockの利用

Swift ConcurrencyのTask.sleepメソッドなどに`Clock`を引数に取ることができようになり、スケジュールのコントロールがより簡単に。

例えば、`Manual Clock`を使ってテスト時のスケジューリングが簡単になる。

```swift

public final class ManualClock: Clock, @unchecked Sendable {
  public struct Instant: InstantProtocol {
    var offset: Duration = .zero
    
    public func advanced(by duration: Duration) -> ManualClock.Instant {
      Instant(offset: offset + duration)
    }
    
    public func duration(to other: ManualClock.Instant) -> Duration {
      other.offset - offset
    }
    
    public static func < (_ lhs: ManualClock.Instant, _ rhs: ManualClock.Instant) -> Bool {
      lhs.offset < rhs.offset
    }
  }
  
  struct WakeUp {
    var when: Instant
    var continuation: UnsafeContinuation<Void, Never>
  }
  
  public private(set) var now = Instant()
  
  // 効率的にデータ構造を最適化して、実行順序も保証された
  // ウェイクアップしたいスリープポイントを保持する汎用的なストレージ
  var wakeUps = [WakeUp]()
  
  // 現在時刻の調整やウェークアップは異なるスレッドやTaskから行われる可能性があるため
  // ロックで制御する必要がある(critical sectionを作成する)
  let lock = os_unfair_lock_t.allocate(capacity: 1)
  
  deinit {
    lock.deallocate()
  }
  
  public func sleep(until deadline: Instant, tolerance: Duration? = nil) async throws {
    // 保留中のウェイクアップをリストにエンキューする
    return await withUnsafeContinuation {
      if deadline <= now {
        $0.resume()
      } else {
        os_unfair_lock_lock(lock)
        wakeUps.append(WakeUp(when: deadline, continuation: $0))
        os_unfair_lock_unlock(lock)
      }
    }
  }
  
  public func advance(by amount: Duration) {
    // 現在時刻を進めて、実行が必要な保留中のウェイクアップを集める
    os_unfair_lock_lock(lock)
    now += amount
    var toService = [WakeUp]()
    for index in (0..<(wakeUps.count)).reversed() {
      let wakeUp = wakeUps[index]
      if wakeUp.when <= now {
        toService.insert(wakeUp, at: 0)
        wakeUps.remove(at: index)
      }
    }
    os_unfair_lock_unlock(lock)
    
    // ロック外で実行する
    toService.sort { lhs, rhs -> Bool in
      lhs.when < rhs.when
    }
    for item in toService {
      item.continuation.resume()
    }
  }
}
```

## 参考リンク

- https://github.com/apple/swift-evolution/blob/e5cdfebdbaa576e4bc0b509cef9e4a277ece841f/proposals/0329-clock-instant-date-duration.md
- https://github.com/apple/swift/pull/40609
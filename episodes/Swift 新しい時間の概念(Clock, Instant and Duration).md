# Swift 新しい時間の概念(Clock, Instant and Duration)

収録日: 2022/03/20

- [Swift 新しい時間の概念(Clock, Instant and Duration)](#swift-新しい時間の概念clock-instant-and-duration)
  - [概要](#概要)
  - [用語紹介](#用語紹介)
  - [概要](#概要-1)
    - [時間の3つの概念](#時間の3つの概念)
  - [内容](#内容)
    - [時間を扱う方法を定義するために考慮が必要なこと](#時間を扱う方法を定義するために考慮が必要なこと)
      - [APIは適切な型のみを扱えるようにする](#apiは適切な型のみを扱えるようにする)
      - [一般的なユースケースでも簡単に正しく扱えるようにする](#一般的なユースケースでも簡単に正しく扱えるようにする)
      - [パフォーマンス](#パフォーマンス)
      - [時間自体は常に、特定の分析フレームを基準に測定される](#時間自体は常に特定の分析フレームを基準に測定される)
      - [時間、時刻、期間の主な目的](#時間時刻期間の主な目的)
    - [現状の問題点](#現状の問題点)
      - [APIの例](#apiの例)
        - [Foundation](#foundation)
        - [Dispatch](#dispatch)
    - [今回の提案の範囲](#今回の提案の範囲)
      - [Go](#go)
      - [Rust](#rust)
      - [Kotlin](#kotlin)
    - [提案の詳細](#提案の詳細)
      - [時計](#時計)
      - [時刻](#時刻)
      - [DurationProtocol](#durationprotocol)
      - [Duration構造体](#duration構造体)
    - [Clockプロトコルの具体的な実装](#clockプロトコルの具体的な実装)
      - [ContinuousClock構造体](#continuousclock構造体)
      - [SuspendingClock構造体](#suspendingclock構造体)
    - [標準ライブラリ以外のClock実装](#標準ライブラリ以外のclock実装)
      - [UTCClock構造体](#utcclock構造体)
    - [Swift ConcurrencyでClockの利用](#swift-concurrencyでclockの利用)
      - [Task](#task)
      - [Custom Clockの例: Manual Clock](#custom-clockの例-manual-clock)
  - [採用されなかった検討事項](#採用されなかった検討事項)
    - [単一の時刻表現](#単一の時刻表現)
    - [Date/UTCClockをより低レイヤへ](#dateutcclockをより低レイヤへ)
    - [DurationProtocolの一般化された算術とプロトコルの定義](#durationprotocolの一般化された算術とプロトコルの定義)
    - [時計とTask Sleepの許容値のオプション](#時計とtask-sleepの許容値のオプション)
    - [名前の代替案](#名前の代替案)
    - [付録](#付録)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [関連PR](#関連pr)
    - [その他](#その他)

## 概要

現在、Swiftで時間を扱う方法はたくさんあるが、統一的で汎用的な方法が存在しない。今回新たに時間の概念を定義し、それを使って時間を扱うAPIを導入したい。

## 用語紹介

- エポック: あることの起点となる時刻。基準点
- NTP: Network Time Protocolの略。TCP/IPネットワークを通じて正しい時刻情報をサーバに問い合わせる際に使用する通信プロトコル。コンピュータはNTPサーバに問い合わせて内部時計を正しく設定している
- real time(リアルタイム時間), wall clock time(壁時計時間): あるプログラムの開始から終了までを計測した時間。NTPサーバによる時間補正の影響を受ける
- monotonic time(モノトニック時間または単調時間): 決して後戻りすることがない単調に増加する時間
- uptime: オペレーティングシステム（OS）が起動してから現在までに経過した時間。稼働時間

ドキュメントの末尾にも用語の一覧が記載されている[付録](#付録)

## 概要

### 時間の3つの概念

1. 「今」の概念を提供するアイテムとある時点で目覚める方法(= 時計 clockと呼ばれる)
2. ある時点を表す概念(= 時刻 instantと呼ばれる)
3. 経過時間を測定する概念(= 期間 durationと呼ばれる)

時間の長さは、ネットワークコネクションのタイムアウトといった高度な概念から、あるタスクがスリープする時間量まで、多岐に渡ってAPIの中で異なる型として使用されている。現在、時間の測定を行うAPIは、`NSTimeInterval`(別名`TimeInterval`)、`DispatchTimeInterval`、さらには`timespec`などの型を取る。

## 内容

### 時間を扱う方法を定義するために考慮が必要なこと

時間を扱う標準的な方法を定義するには、様々なことを考慮する必要がある。

#### APIは適切な型のみを扱えるようにする

時間の計測方法を特定の時間の概念の中に限定することが重要な場合に、その能力を保持できることを保証する必要がある。例えば、リアルタイム時間の期限を受け取るAPIにモノトニック時間を渡せてはならない。

#### 一般的なユースケースでも簡単に正しく扱えるようにする

一方で、こういった特定の時間の型を知らなくても高レベルのAPIを扱えるようにして、エルゴノミクスを考慮する必要もある。Swiftを学び始めた開発者に、各OSが利用可能な無数の時計の概念の違いの理解を強制するのは喜ばしいことではない。同様に、すべての実装は、複数のオペレーティングシステムのバックエンド(Linux、Darwin、Windowsなど)をサポートするのに十分な堅牢性とパフォーマンスを備えていなければならないが、一般的なユースケースで正しく扱える程度に十分に簡単であるべき。実際には、期間は時刻と時計への段階的開示(※)であるべき。

※ 理解を容易にし、間違いを起こしにくくする手段

#### パフォーマンス

パフォーマンスという意味では、大きなオーバーヘッドを発生させることなく、どんな期間の型(あるいは時計の型)であっても、関数の実行パフォーマンスの測定などのタスクを実行するのに十分なパフォーマンスを備えていなければならない。つまり、期間を表現しているタイプは小さく、おそらくある種の(またはグループの)PoD型(Plain Old Data)※に裏打ちされているべき。

※ C言語のデータと互換を持つデータ構造

#### 時間自体は常に、特定の分析フレームを基準に測定される

時間自体は常に、特定の分析フレームを基準に測定される。例えば、uptimeは、マシンが起動された時間に対する相対的な観点で測定されるが、他の時計は特定のエポックに関連しているかもしれない。特定の基準点に対して表現された時刻は、潜在的に精度が損失されて変換される可能性があるが、他の時刻の場合は変換できないこともある。したがって、これらの変換は、一般的なプロトコル要件として一律に表現することはできない。(何かしらの変換方法が必要になってくる)

#### 時間、時刻、期間の主な目的

時計の主な目的は、後で実行する作業をスケジュールする方法を提供すること。時刻は、そのスケジューリングの一時的な基準点を提供すること。期間は、2つの時点の間の経過時間を表す高精度の積分時間になるように特別に設計されていること。

### 現状の問題点

この3つを表すAPIや型がたくさんある。

#### APIの例

##### Foundation

- `Date`: UTC時間の2001年1月1日を起点としてある瞬間を構築する
- `TimeInterval`: 2つの時刻の期間を秒数で表す

##### Dispatch

- `DispatchTime`: システムが起動してからの時間
- `DispatchWallTime`: プログラムの開始から終了までを計測した時間
- `DispatchTimeInterval`: 秒、ミリ秒、マイクロ秒、ナノ秒の値


これらは明白に唯一の定義ではないが、同時並行処理を扱う場合、これらのすべての概念への統一されたアクセサがあれば、sleep、他の時間的概念に必要なプリミティブを構築するのに役立つ。

### 今回の提案の範囲

プロセスでの作業のスケジューリングに使用される時間に焦点を当てる。この目的で最も有用な時計は、プロセスを実行しているマシンが開始されてからの時間を計算する単純なローカル時計である。

時間は、グレゴリオ暦の「2021年4月1日」などのカレンダを使用して人間の言葉で表すこともできる。標準ライブラリと`Foundation`の　それぞれの責務に応じて、カレンダの定義と、カレンダ内の日付間の移動に関連する計算は、`Foundation`の`Calendar`、`DateComponents`、`TimeZone`、および`Date`型に任せる。

今回の実装を検討するために、3つの言語(Go, Rust, Kotlin)を分析した。

#### Go

https://pkg.go.dev/time  
https://golang.org/src/time/time.go

Goは、時間を壁掛時計の基準点(`uint64`)、extで追加のナノ秒フィールド(`int64`)、および場所(ポインタ)の構造体として保持する。また、期間を`int64`(ナノ秒)のエイリアスとして保持する。

Goは、特定の時間を指定するために基準点を制御することはできない。単調または壁掛時計のいずれかになる。基本実装は、単調時計と壁掛時計の値の両方を一緒にカプセル化しようとする。一般的なユースケースの場合、これはほとんどまたはまったく影響を与えない可能性があるが、使用時の段階的開示を認識するために必要な特異性が欠けている。

#### Rust

https://doc.rust-lang.org/stable/std/time/struct.Duration.html

Rustは、期間を`u64`秒および`u32`ナノ秒として保持する。時間の測定には、ほとんどのプラットフォームで単調時計を使用しているように見える`Instant`を使用する。

#### Kotlin

https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.time/-duration/

Kotlinは、`Duration`を`Long`に加えて、ミリ秒またはナノ秒で構成される単位識別子として保持する。測定関数は期間を返さないが、代わりにミリ秒単位の`Long`値などからの変換関数に依存し、現在の測定関数はシステムの稼働時間を使用して基準点を決定する。

### 提案の詳細

#### 時計

2つの基本的な要件がある

- 与えられた時刻の後に起動する(スリープ状態をレディー状態にする)方法
- 「今」の概念を生成する方法

```swift
public protocol Clock: Sendable {
    associatedtype Instant: InstantProtocol

    var now: Instant { get }

    func sleep(until deadline: Instant, tolerance: Instant.Duration?) async throws 

    var minimumResolution: Instant.Duration { get }
}

extension Clock {
    func measure(_ work: () async throws -> Void) reasync rethrows -> Instant.Duration
}
```

※ [Amend SE-0329 to add Clock.Duration](https://github.com/apple/swift-evolution/pull/1618)で`associatedtype`に`Duration`が追加になりました。これはprimary associated typeとして使用したいということが主な動機だそうです(これに関しては別エピソードで紹介)。

```swift
public protocol Clock: Sendable {
    associatedtype Duration: DurationProtocol
    associatedtype Instant: InstantProtocol where Instant.Duration == Duration
    ...
}
```



`associatedtype`に、ある時刻を表す`InstantProtocol`プロトコル(後述)に準拠した型を持つので、時計と時刻は紐づいている。例えば、特定の時計の時刻と他のすべての時計の時刻を比較することはできない。ただし、既存のAPIと新しいAPI間のやり取りをスムーズにするためや、使いやすさという点を考慮して、2つの時刻間の期間は比較することができるようになっている。ただし、時計を跨るので間違って使わないように注意する必要がある。プロトコル階層を時計と時刻だけにすることで、すべての場合で利用可能な期間を簡潔な形式で簡単に表現できる。これは特に既存の型の代わりに期間の概念を定義した型を利用する可能性のあるAPIの場合に有用である。

時計の精度を調整することもできる。ある時計はナノ秒単位の精度を持つし、ある時計はマイクロ秒単位での精度を持つ。デフォルトは1ナノ秒(`.nanosecond(1)`)。経過時間が指定している精度以下の場合は、正確な情報を提供できない。

時計はある作業量を測定するためにも使われる。つまり、時計には、メトリックのワークロードを測定するだけでなく、パフォーマンスベンチマークのワークロードも測定できるようにするための拡張機能が必要になる。これはデフォルト実装を提供して簡単に使用できるようにしている。

```swift
let elapsed = someClock.measure {
  someWorkToBenchmark()
}
```

`now`を提供する以外の時計の主な用途は、与えられた時刻の後に起動すること。これにより、指定された時刻の後に実行される作業をスケジュールすることができる。スケジュールされた作業の起動は、電力に影響を与える可能性があり、 特にCPUを頻繁に立ち上げると、過度の電力消費が発生する可能性がある。そこで、期限への許容範囲を示すことで、カーネルからの基礎となるスケジューリングメカニズムに、微調整された期限を使って起動できるようにする。これにより、スケジュールされている他の作業と一緒に作業をグループ化して、より電力効率の高い実行を行うことができる。許容値を指定しないと、適切な値を選択するために、許容値がその時計の実装の詳細に依存していると時計の実装者に推論される。許容範囲は、システムがスリープを遅らせることができる期限後の最大期間。

```swift
func delayedHello() async throws {
    try await someClock.sleep(until: .now.advanced(by: .seconds(3))
    print("hello delayed world")
}
```

上記の例では、時計は呼び出された瞬間から3秒までスリープされ、その後出力される。スリープ機能が一時停止されている間にタスクがキャンセルされた場合、`sleep`はエラースローする。この例では、許容値はデフォルトで時計によって`nil`に設定され、期限に適用できる許容値の「ディーラの選択」(システム依存)として残されている。

#### 時刻

前述したように、比較できる(`Comparable`)かつ`Key`として保存できる(`Hashable`)必要だが、今という概念と与えられた期間分の時間を進める方法のみを定義すれば良い。

`associatedtype`に、ある期間を表す`DurationProtocol`プロトコル(後述)に準拠した型を持つので、その型に適切なストレージの型を使用でき、さらに時刻に紐づく時計も型安全になるメカニズムを提供する。

時刻が有用な主な理由は、組み合わせて使えること。例えば、時刻の期限を引数に受け取る関数から、さらに他の期限を受け取る関数を呼び出す場合、この元の期限を変更せずに次の関数に渡すことができる。つまり、その期限が経過した瞬間に、既存の呼び出しや関数間の実行時間に干渉することはない。この一般的な例の1つは、URLリクエストに関連するタイムアウト。タイムアウトは、どうやって実行期限が発生するかを完全にはカプセル化しない。接続が確立され、データが送信され、応答が受信されるまでの期限がある。これらすべてにまたがるタイムアウトには、各ステップでの測定を組み合わせる必要があるが、期限は全体を通して静的。

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

`InstantProtocol`には、`advance(by:)`メソッドと`duration(to:)`メソッドに加えて、期間を加算および減算する演算子がある。ただし、`AdditiveArithmetic`には準拠していない。このプロトコルでは、2つのインスタント値を加算し、ゼロ値を定義する必要がある(これは`Clock`から取得されるため、静的にすべての`InstantProtocol`型を知ることができない)。さらに、`InstantProtocol`は`Strideable`ではない。これは、`SignedNumeric`にする必要があり、`Duration`に、2つの期間には不適切な別の`Duration`を掛けることが必要になる。

`Strideable`が`SignedNumeric`を必要としなくなった場合、または`SignedNumeric`が`self`の乗算が必要なくなった場合に、`InstantProtocol`またはこれに準拠する型で必要とするかを検討をするべき。

#### DurationProtocol

「期間」を表す概念を表したもの。比較可能(`Comparable`)かつ加算できる(`AdditiveArithmetic`)。ただし、`InstantProtocol`と同様の理由で`Stridable`ではない。

[AdditiveArithmetic](https://github.com/apple/swift/blob/eb93dea828a9e02b27ecff397eba3f37d8ac401d/stdlib/public/core/Integers.swift#L62)

```swift
public protocol DurationProtocol: Comparable, AdditiveArithmetic, Sendable {
    static func / (_ lhs: Self, _ rhs: Int) -> Self
    static func /= (_ lhs: inout Self, _ rhs: Int)
    static func * (_ lhs: Self, _ rhs: Int) -> Self
    static func *= (_ lhs: inout Self, _ rhs: Int)

    static func / (_ lhs: Self, _ rhs: Self) -> Double
}

extension DurationProtocol {
    public static func /= (_ lhs: inout Self, _ rhs: Int) {
        lhs = lhs / rhs
    }

    public static func *= (_ lhs: inout Self, _ rhs: Int) {
        lhs = lhs * rhs
    }
}
```

期間の効率的な計算を保証するには、`DurationProtocol`に準拠する型が実装する必要のある加算演算以外にも、いくつかの追加の方法、2進整数による除算と乗算、および`Double`の値を作成する除算が必要。これにより、タイマーのスケジューリングやバックオフアルゴリズム(※)などの概念を実現するための最小限の関数セットが提供される。

※ Ethernetで、コリジョンが生じたときに、次の送信までの待ち時間を決めるアルゴリズム

このプロトコル定義は、`VectorSpace`の概念に非常に近い。

`Comparable`と`AdditiveArithmetic`がより洗練されたプロトコルになった場合、改善の可能性があるとして検討されるべき。

※ Durationという名前について

特定の時計には、時間的概念以外の期間を表す可能性がある。例えば、GPUに関連付けられた時計は期間をフレーム数として表すことができるが、手動の時計はそれらをステップとして見なせる。ただし、ほとんどの時計は、期間の型を継続的に測定した秒/ナノ秒などの整数で表される期間(Duration)として表される。時計を実装することはほとんどなく、`Swift.Duration`の名前を拡張して使用することは期待通りで、時計との通常のやり取りには影響しないと考えている。

#### Duration構造体

「期間」を表す`DurationProtocol`の具体的な実装。すべての時計で使われている。基準点からの過去と未来をアト秒(※)と秒で表現する。人間の測定値(または機械で測定された精度)から構築できるが、カレンダの概念(日、月、年など)は持たない。

※ アト秒は1×10⁻¹⁸秒。比較のために、アト秒を1秒にすると、1秒は約317.1億年になる。

直列化可能(`Codable`)、比較可能(`Comparable`)、`Key`として保存できる(`Hashable`)ことに加え、加算および減算も可能である(そしてゼロ値にも意味がある)。

2つの`TimeInterval`変数の乗算に関する前述の問題のため、これらは明示的に`Numeric`ではない。しかし、バックオフを計算するためのアドホックな除算と乗算を行うユーティリティがある。

`Duration`は、より細かい精度を考慮に入れる必要がある。こうすると内部のストレージは除算時の適切な丸め精度(外に現れるよりも高い精度)と、潜在的に合理的な期間の全範囲をカバーするのに十分な範囲を保証することができる。

秒とアト秒を保持することで、+/-数千年の全範囲を損失のないスケールでカバーできることを指す。すべてのシステムがその全範囲を必要とするわけではないが、オペレーティングシステムで表現される全範囲にわたってアト秒の精度を適切に表すには、これらの値を表すためにSwiftが完全な128ビットストレージで動作する必要がある。

そのため、期間を2つのコンポーネントに分割するため、既存のタイプへの変換を公開する必要がある。期間のこれらのコンポーネントは、秒部分およびアト秒部分としての `timespec`などの既存のAPIとの相互運用性のために公開される(完全な精度が失われないようにするため)。

Swiftが128ビットのストレージをサポートできる符号付き整数型を導入した場合、`Duration`は、コンポーネントのアクセサとイニシャライザを、格納されているアト秒への直接アクセスと初期化に置き換えることを検討する必要がある。


```swift
public struct Duration: Sendable {
    public var components: (seconds: Int64, attoseconds: Int64) { get }
    public init(secondsComponent: Int64, attosecondsComponent: Int64)
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

### Clockプロトコルの具体的な実装

広く一般的に使われるものは標準ライブラリで実装している。

#### ContinuousClock構造体

時刻がローカルのみで使われ、マシンがスリープ中も正確に時間を進めたい場合に有効な時計。

Darwinの場合はmonotonic時間に由来する時間を参照する。Linuxの場合はuptimeに由来する時間を参照する。

推論される基本型のプロパティとしてその時計インスタンスにアクセスするための拡張機能も提供する。

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

ローカルプロセスまたはクロスマシンの範囲での時刻が適切でない場合、つまり稼働時間は、マシンがスリープしている間は増分しないが、マシンの起動時間を参照するという目的のために使われる時計。マシンの範囲内でプロセス間通信を行うことができる。

他と同様に推論される基本型のプロパティとして時計インスタンスにアクセスするための拡張機能も提供する。

マシンがスリープしている間は増加しないという概念を最もよく表しているため、Darwinの場合はmonotonic時間に由来する時間を参照する。Linuxの場合はuptimeに由来する時間を参照する。

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

### 標準ライブラリ以外のClock実装

#### UTCClock構造体

UTCベースのカレンダの概念を伴った`Foundation`内の時計

起動時間が現在のUTC時間で調節される。

時刻の型は`Date`。`Date`から`ContinuousClock`や`SuspendingClock`の時刻へ相互変換する方法を提供する。

`UTCClock`は`Date`で定義された期限後に起動することができる。`Date`の実装は、システム時計で定義されている2001年1月1日以降の秒数で処理されるため、ネットワーク時間(または手動)の更新は、システム時計の歪みに応じて、その時点を順方向または逆方向にシフトする可能性がある。

保持している値は、タイムゾーン、夏時間、またはカレンダに依存しないが、現在のNTPサーバの更新によってうるう秒が適用される可能性がある。以前は公開されていなかったこの特定のエッジケースに照らして、`Date`は、特定のデータと別の日付の間で経過したうるう秒の期間を指定する新しい方法を提供する。これは、歴史的な意味でこれらのうるう秒を説明する方法を提供する。タイムゾーンデータベースと同様に、うるう秒はソフトウェアの更新とともに更新される(追加のうるう秒が計画されている場合)。

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

※ 標準ライブラリかFoundationか

このプロポーザルの以前のイテレーションの中で、`WallClock`の概念(UTCに基づく時計)を標準ライブラリに提供したが、フィードバックの後、この型は、カレンダ計算のコンテキストなしでは有用ではない可能性があると感じ、`Foundation`はこのような計算を行う場所であるため、UTCに基づく時計はそのレイヤーに属している方がより適切にと感じる。

この時計は、現在のUTC時間に基づいて発火時間を調整する。これは、カレンダを介した計算によって作成された特定の時刻によって少しの作業がスケジュールされている場合、システム時刻がその期限に達すると、この時計がスリープから復帰できることを意味する。

また`Date`もWallClockと共に標準ライブラリに移動する話が当初あったが、特定のカレンダに依存するのは有用ではないという結論から`Foundation`にとどまることに。

※ `Date`を使うことに関して

`Date`は、`Calendar`、`TimeZone`を使用して解釈される特定の時点のストレージとしてや、ユーザに表示するためのフォーマット機能とともに使用するのが最適。macOSおよびiOSのSDKの既存の`Date` APIを調べるとは、これを使用するプロパティと関数の大部分にすでに当てはまっていた。`Date`という名前が適切かどうかの議論は、主に「非カレンダ」コンテキストでの使用に焦点が当てられており、`Date`と`UTCClock`の組み合わせが、これらの型間の関係を強化し、いつ使用すべきかを明確にするのに役立つことを期待している。

このアプローチは、これらのAPIとの互換性を維持しながら、まれに必要になる場合にスケジューリングに`Date`を使用する機能を提供する。

### Swift ConcurrencyでClockの利用

#### Task

既存の`Task.sleep`メソッドは特定のスリープ動作はないが、内部では、Darwinの場合continuous clock、Linuxではsuspending clockを使用している。

これまでのAPIはdeprecatedになり、下記のような新しいAPIを提供する。

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

※ 新しいAPIの`Task.sleep(for:)`は`Clock`の`sleep(until:,tolerance:)`を呼び出している。

[TaskSleep](https://github.com/apple/swift/blob/e675b310f89bb9fdfe6e139bdece3dcd28ad636f/stdlib/public/Concurrency/TaskSleep.swift)

#### Custom Clockの例: Manual Clock

上記のTask.sleepメソッドに`Clock`を引数に取ることができようになり、スケジュールのコントロールがより簡単になる。

例えば、下記のようなCustom Clockを使ってテスト時のスケジューリングができるようになる。

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

## 採用されなかった検討事項

### 単一の時刻表現

Goと同様に、monotonic、uptime、およびwall clock時間を表す単一な型が検討された。ただし、このアプローチでは互換性に問題が生じる。時刻は、ある点では大きいかもしれませんが、他の点では小さいか等しいかもしれない。`InstantProtocol`の必要条件として`Comparable`を適切に順守するために、時刻を1つの統合された型に結合することは理想的ではないと感じている。

###　プロトコル階層の反転

もう1つの調査は、時刻と時計のスキームを逆にして使用することだが、これは、特定の時計または時刻を使用する関数の一般的なシグネチャを作成するのがはるかに困難である。

### Date/UTCClockをより低レイヤへ

当初、ストレージを`Double`から`Duration`に変更することに加えて、`Date`を標準ライブラリに移動させるという考えが含まれてた。この動きにはいくつかの面で強い反対があり、最終的にはそれには説得力があった。主な反対意見は、`Date`という名前に対するもの。標準ライブラリまたはConcurrencyライブラリ内に追加のコンテキストを加えるAPIがないことを考えると、これは、`Date`がカレンダの概念と簡単に混同される可能性があることを意味した(`Date`は間違いなくそういったものではない)。さらに、`Date`にはうるう秒の概念が欠けていることも問題視された(これは、`Date`の追加機能として有用性であると思われるため`Foundation`の変更として受け入れられ、提案されている)。

また、元の改訂版には、`WallClock`の概念があった。多くの議論の結果、この型は実際にはUTCに基づく時計を表すため、wall clockという名前は誤解を招くと感じる(`Date`にうるう秒の履歴が含まれる場合)。しかしさらに、UTC時計を介したスケジューリングの有用性は一般的なタスクではなく、スケジューリング用の時計の大部分は、実際には、マシンがスリープしている間に時間が経過する時計または機械が眠っている間は時間が経過しません。その責務は、UTC時計の適切な場所が、カレンダ計算APIと並んで、その特殊なタスクのためのより高いレベルのフレームワークにあると私たちが感じていることを意味する。それが`Foundation`。

### DurationProtocolの一般化された算術とプロトコルの定義

`DurationProtocol`がより一般的な算術の形式を持つことも検討された。これは、値の切り捨てを誤って実装する可能性という潜在的な落とし穴がある。整数型として渡されるほとんどの値は`Int`のため、このインターフェースは`Int`を介した乗算と除算を使用するだけでより適切に機能する考えられる。その文脈で、代わりに`Double`を使用することも検討されたが、これは例えば1単位を超えて明確に分割できない「ステップ」や「フレーム」などの期間を定義する型ではうまく機能しない。その動作とそれがどのように丸めまたはアサートされるかなどを定義するのは、まだ`DurationProtocol`に準拠した型領域下にある。

算術と同様に、また、`InstantProtocol`の関連型を単に`Comparable ＆ AdditiveArithmetic ＆ Sendable`のプロトコル合成にすることも検討された。これには、バックオフ(Zenoのアルゴリズム)やデバウンス、タイマー結合などの[高速パス](https://en.wikipedia.org/wiki/Fast_path)の機能がない。それらのいくつかは、加算のループの観点から書き直すことができるが、場合によっては、欠落した間隔でホットループが発生したり、それらを実装できなかったりする可能性がある(バックオフの分割など)。


### 時計とTask Sleepの許容値のオプション

XcodeなどのIDEからのヒントとして`.none`のオートコンプリートが存在し、それらの命名法は、`sleep`関数の`tolerance`パラメーターについて誤解を招く可能性があることが提起された。これはおそらくオートコンプリートとして公開するのに理想的な名前ではないことに同意するが、パラメータを渡さない、または`nil`を渡す代わりに`.none`を使用するコードは、スタイル的に問題があり、Swiftの以前のバージョンからやり残しであると判断された。この分野のソリューションは、`Clock`と`Task`だけでなく、オプショナルのパラメータを持つ他のメソッドにも適用できるはずであると結論付けられた。さらに、提案されている`ContinuousClock`、`SuspendingClock`、`UTCClock`は、パラメータ値がないことが最も重要であるため、提案されているAPIの問題というよりも、Xcodeのオートコンプリートのバグである可能性がある。一種の直接型受け渡し機能。つまり、この問題についてはより一般的な解決策に取り組む必要があり、オプショナルの期間型を現状残すべき。

### 名前の代替案

このプロポーザルの間に考慮された多くの名前がある(これらはいくつかのハイライトです):

プロトコル`Clock`は、以下の名前が検討された:

- `ClockProtocol` - プロトコルにサフィックスは不要であり、命名ガイドラインに違反していると見なされた

プロトコル`InstantProtocol`は、以下の名前が検討された:

- `ReferencePoint` - これはあいまいすぎて、時間の概念を捉えていない
- `Deadline`/`DeadlineProtocol` - すべての時刻の型が実際に期限であるとは限らないため、命名法が混乱
- 関連型の`InstantProtocol.Duration`は、他のいくつかの名前が検討された。`TimeSpan`と`Interval`。これらの名前には対称性がない。`Clock`には`InstantProtocol`である`Instant`があり、`InstantProtocol`には`DurationProtocol`である`Duration`がある

プロトコル`DurationProtocol`は、以下の名前が検討された:

- 「フレーム」や「ステップ」などの概念でトランザクションを実行する他の時計型のAPIの柔軟性を確保するために、検討されていないが、最終的には拒否された

`ContinuousClock`は、以下の名前が検討された:

- `MonotonicClock` - 残念ながら、DarwinとLinuxは単調の定義が異なる
- `UniformClock` - これは、マシンがスリープしていない間、両方の増分が均一であるため、この時計と`SuspendingClock`の動作の違いを明確にするものではない

`SuspendingClock`は、以下の名前が検討された:

- UptimeClock - `MonotonicClock`がLinuxとDarwinの動作に関してあいまいさを持っているのと同じ
- `AbsoluteClock` - マッハ主義にすぐに没頭していない場合は非常にあいまい
- `ExecutionClock` - この名前は、(Darwin上)`CLOCK_UPTIME_RAW`よりも`CLOCK_PROCESS_CPUTIME_ID`の概念を暗示しいる
- `DiscontinuousClock` - 不連続関数の数学的概念にルーツがあるが、マシンがスリープ状態のときに進まないのは時計であることがすぐにはわからない

`Duration`型は、以下の名前が検討された:

- `Interval` - これは非常にあいまいであり、時間以外の多くの他の概念を参照する可能性がある
- `nanosecondsPortion`と`secondsPortion`は、`nanoseconds`と`seconds`という名前で検討されたが、これらの名前は丸めのあいまいさをもたらす。portionという用語を用いると、単なる丸め/切り捨てられた値ではなく、それらの小数部の構成が推測される

`Date`型は、以下の名前が検討された:

- `Timestamp` - まともな代替手段だが、カレンダーに関連付けられていることに関しては、まだ少しあいまい。また、文字列のような意味合いがある(ログでの使用方法を含む)
- `Timepoint`/`TimePoint` - あいまいさは少ないが、最終的にはすでに存在する数千のAPI(iOSおよびmacOS SDKに含まれているものを数えるだけで、存在する可能性のある他の使用サイトは言うまでもない)をやめるのに十分な説得力のない合理的な代替手段
- `WallClock.Instant`/`UTCClock.Instant` - これは、`Date`が今日表すのと同じアイデアを綴る非常に冗長

`Task.sleep(for:tolerance:clock:)`APIは、以下の名前が検討された:

- `Task.sleep(_:tolerance:clock:)` - これはまだ文法的に正しく「for」という不要な単語を省略できるが、この追加単語を使用してもまだ十分に読みやすく、deprecatedになったAPIからの移行のためのより良いfix-itも提供される。以降において「for」を維持する価値があると考えた

### 付録

時間は相対的であり、時間的タイプはそれ以上に相対的である。このドキュメントでは、読者が明確に知っておくべき時間的タイプの分類に関していくつかの議論がある。

- カレンダー(Calendar): 時間を測定するための人間のロケールベースのシステム
- 時計(Clock): 時間を測定し、その時間がどのように流れるかを理解するメカニズム
- 連続時間(Continuous Time): 常に増分し、システムがスリープしている間も増分を停止しない時間。これは、ストップウォッチスタイルの時間として役立つ。開始時が基準点でありマシンごとにほぼ完全に異なる
- 日付(Date): 日付値は、特定のカレンダーシステムやタイムゾーンに関係なく、単一の時点をカプセル化する。日付値は、絶対参照日を基準にした時間間隔を表す
- 期限(Deadline): 一般的な用語では、これは時刻として定義される制限。つまり、目的を達成する必要がある狭い時間のフィールド
- 期間(Duration): 2つの期限または参照ポイントの間で経過した時間の長さ
- 時刻(Instant): まさにその瞬間
- 単調時間(Monotonic Time): DarwinとBSDは、これを連続時間(Continuous Time)と定義している。ただし、Linuxはこれを常に増分する時間として定義しているが、システムがスリープしている間は増分を停止する
ネットワーク更新時間(Network Update Time): NTPを介して送信される壁時計時間(wall clock time)の値。NTPはネットワークに接続されているマシンの壁掛け時計(Wall Clock)を同期するために使用される
- 時間的(Temporal): 時間の概念に関連した
- タイムゾーン(Time Zone): 太陽時(※)の頂点を12:00前後に保つことを目的とした、準地理空間の描写で時間を正規化する、任意の政治的に定義されたシステム  
※ 太陽の運動を地表上から観測し、天球上で最も高い位置に達する、もしくは正中の時刻を正午とするという考え方に基づく時刻系
- 許容範囲(Tolerance): 特定の時点の前後の期間は正確であると見なされる期間
- 稼働時間(Uptime): DarwinとBSDは、これを、スリープ時に一時停止する絶対時間として定義している。ただし、Linuxはこれをスリープ中に一時停止するのではなく、ブートに関連する時間として定義している
- 壁時計時間(Wall Clock Time): 時計から読むような時間。様々な理由で前方または後方に調整される場合がある。この文脈で言うと、タイムゾーンやロケールに固有ではなく、絶対参照日から測定された時間。ネットワークの更新により、相対論的ドリフト、プロセッサの不正確さ、またはハードウェアの電力特性に応じて、時計の歪みが逆方向または順方向に調整される場合がある

## 参考リンク

### Forums

- [SE-0329: Clock, Instant, Date, and Duration](https://forums.swift.org/t/se-0329-clock-instant-date-and-duration/53309)
- [[Pitch (back from revision)] Clock, Instant, and Duration](https://forums.swift.org/t/pitch-back-from-revision-clock-instant-and-duration/54035)
- [SE-0329 (Second Review): Clock, Instant, and Duration](https://forums.swift.org/t/se-0329-second-review-clock-instant-and-duration/54509)
https://forums.swift.org/t/se-0329-third-review-clock-instant-and-duration/54727
### プロポーザルドキュメント

- [Clock, Instant, and Duration](https://github.com/apple/swift-evolution/blob/main/proposals/0329-clock-instant-duration.md)
- [[Accepted] SE-0329: Clock, Instant, and Duration](https://forums.swift.org/t/accepted-se-0329-clock-instant-and-duration/55324)

### 関連PR

- [Clock/Instant/Duration](https://github.com/apple/swift/pull/40609)

### その他

- [Foundation Date](https://developer.apple.com/documentation/foundation/date)
- [Dispatch DispatchTime](https://developer.apple.com/documentation/dispatch/dispatchtime/)
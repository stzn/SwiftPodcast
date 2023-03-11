# Swift Custom Actor Executors

- [Swift Custom Actor Executors](#swift-custom-actor-executors)
  - [概要](#概要)
  - [動機](#動機)
  - [内容](#内容)
  - [詳細](#詳細)
    - [低レベルの設計](#低レベルの設計)
    - [Executor](#executor)
    - [Serial Executors](#serial-executors)
    - [Jobs](#jobs)
    - [カスタムのSerialExecutorを持つActor](#カスタムのserialexecutorを持つactor)
    - [Executorのアサーション](#executorのアサーション)
  - [Actor executorsの推論](#actor-executorsの推論)
  - [Executorの等価チェックの詳細](#executorの等価チェックの詳細)
    - [同じSerialExecutorに委譲する一意のExecutor](#同じserialexecutorに委譲する一意のexecutor)
    - [同じ実行コンテキストを提供する異なるExecutor](#同じ実行コンテキストを提供する異なるexecutor)
    - [デフォルトのSwiftランタイムExecutor](#デフォルトのswiftランタイムexecutor)
  - [ソース互換性](#ソース互換性)
  - [ABI stabilityへの影響](#abi-stabilityへの影響)
  - [APIレジリエンスへの影響](#apiレジリエンスへの影響)
  - [代替案](#代替案)
  - [将来の検討](#将来の検討)
    - [MainActor Executorのオーバーライド](#mainactor-executorのオーバーライド)
    - [Executorスイッチング](#executorスイッチング)
    - [タスクのExecutorの指定](#タスクのexecutorの指定)
    - [DelegateActorプロパティ](#delegateactorプロパティ)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

Swift Concurrencyが成熟し続けるにつれて、非同期タスクが実際に実行される場所を正確に制御することを利用者に提供することがますます重要になってきている。

この提案は、ActorのExecutorをカスタマイズするための基本的なメカニズムを導入している。Actorは、Actorモデルによって保証された排他制御とActorの分離を維持しながら、実行中の任意のタスクを「どこで」実行するかに影響を与えることができる。

> 注意: この提案はActorのExecutorをカスタマイズするためのAPIセットのみを定義しており、その他の種類のExecutor制御は対象外である

## 動機

Swift Concurrencyのデザインは、コードが実際に実行される方法の詳細について、意図的に曖昧にしている。ほとんどのコードは、特定のオペレーティングシステムのスレッドで実行されるといったような、実行環境の特定のプロパティに依存しない。その代わりに、(Actor分離と呼ばれる)他のコードが同時に特定の変数にアクセスしないような、高レベルのセマンティックプロパティのみを必要する。タスクがスレッドにスケジュールされる方法についての柔軟性を維持することは、Swift がデフォルトで特定のパフォーマンスの落とし穴を避けることができるようにしている。

それにもかかわらず、コードが実行される方法をより細かく制御することは、時には便利である。

- コードは、ある特定の方法でコードを実行することを期待する既存のシステムと協力する必要がある可能性がある  
  
  例えば、あるプラットフォームではUIコードをメインスレッドで実行する必要があったり、シングルスレッドのイベントループ型ランタイムは、ランタイム自身が所有・管理する同じスレッドからすべての呼び出しが行われると仮定するなど、システムはある種のタスクを特殊な方法でスケジュールするよう求めているかもしれない。  
  
  別の例として、プロジェクトには、共有のキューで何らかの状態を保護する既存のコードが大量に存在する場合がある。原理的には、これはActorパターンであり、Actorを使ってコードを書き換えることができる。しかし、それを行うことは不可能であるか、少なくともすぐに行うことは非現実的であるかもしれない。ActorのExecutorとして既存のキューを使用することで、より徐々にActorを導入していくことができる。

- コードは、特定のシステムスレッドで実行されることに依存する場合がある  
  
  例えば、一部のライブラリはスレッドローカル変数に状態を保持し、間違ったスレッドでコードを実行すると、ライブラリの仮定が崩れることになる。  
  
  別の例として、すべての実行環境が均質であるとは限りません。一部のスレッドは、特別な機能を持つプロセッサに固定されている場合があります。

- プログラマがコードを実行する場所をより明確にすることで、コードの性能が向上する可能性がある  
  
  例えば、あるActorが別のActorに対して頻繁にリクエストを行い、Actorが同時に実行されることによる利益がほとんどない場合、同じExecutorを使用するように構成すると、Actor間のスイッチングによる実行時コストを削減することができる  
  
  別の例では、非同期関数が中断を挟むことなく同じActorに多くの呼び出しを行う場合、そのActorのExecutorで明示的に関数を実行すると、最終的に Swift は多くのスイッチングによるオーバーヘッドを回避できるかもしれない(あるいは、それらの呼び出しを「atomicに」実行するために必要かもしれない)。

これは Swift ConcurrencyランタイムのCustom Executorsとカスタマイズポイントを議論する最初の提案で、最も基本的なカスタマイズポイントのみを導入しているが、Actorの実行セマンティクスをより厳密に制御したいユーザーにすでに大きな価値を提供することを確信している。

## 内容

Actorが自分の好きなExecutorを持つnonisolatedなunownedExecutorプロパティを宣言することで、実行を要求されるExecutorを宣言できるようにすることを提案する。

## 詳細

### 低レベルの設計

ExecutorのAPI設計は、高性能な実装をサポートすることを目的としており、Custom Executorは主にエキスパートによって実装されることを想定している。そのため、以下の設計では、考えられる他のほとんどの目標よりも、抽象化コストを確実に排除することに重点を置いている。特に、プロトコルによって指定される atomic な操作は、一般に、実装が正しく使用することを要求される不透明で unsafe な型によって表現される。これらの操作は、Swift Concurrencyの高レベルの言語操作と同様に、より便利な API を実装するために使用される。

### Executor

最初に、次に説明するすべての特定の種類の Executorの親プロトコルとして機能する、`Executor`プロトコルを紹介する。これは最も単純な種類の　Executorで、投入されたタスクについていかなる順序の保証も提供しない。Executorは、送られてきたJobを並行して実行するか、順次実行するかを決定することができる。

このプロトコルは Swift Concurrencyの導入以来ずっと Swift に存在していたが、この提案では、言語内で新しく導入された move-only 機能を利用するために、その API を改訂します。既存の `UnownedJob`API は、move-only `Job`を受け入れるものの代わりに、非推奨となります。`UnownedJob`型は利用可能なままです(そして同様に安全ではない)。なぜなら、今日でもいくつかの使用パターンは、move-only型の最初の改訂版ではサポートされていないため。

Concurrencyのランタイムは、Executorの`enqueue(_:)`メソッドを使用して、与えられたExecutorにあるタスクをスケジュールする。

```swift
/// Jobを実行することができるサービス
@available(SwiftStdlib 5.1, *)
public protocol Executor: AnyObject, Sendable {
  // この要件は、オーバーライドしないものとして、ここで繰り返される。
  // そのため、冗長な witness-table エントリを取得する。
  // これにより、基本的なタスクスケジューリング操作のために、根本のプロトコルまで掘り下げて探すことを避けることができる。
  @available(SwiftStdlib 5.9, *)
  func enqueue(_ job: __owned Job)

  @available(SwiftStdlib 5.1, *)
  @available(*, deprecated, message: "Implement the enqueue(Job) method instead")
  func enqueue(_ job: UnownedJob)
}
```

既存のAPIからのマイグレーションを支援するために、コンパイラは `Hashable.hashValue`から `Hashable.hash(into:)`へのマイグレーションの方法と同様の支援を提供する予定である。`enqueue(UnownedJob)`を実装した既存の Executor実装はまだ動作するが、非推奨の警告が表示される。

```swift
final class MyOldExecutor: SerialExecutor {
  // WARNING: 'Executor.enqueue(UnownedJob)' is deprecated as a protocol requirement;
  //          conform type 'MyOldExecutor' to 'Executor' by implementing 'enqueue(Job)' instead
  func enqueue(_ job: UnownedJob) {
    // ...
  }
}
```

Executorは、そのJob を実行するときに、特定の順序規則に従うことが要求される。

- `SerialExecutor.runJobSynchronously(_:)`の呼び出しは、`enqueue(_:)`の呼び出しの後に起こらなければならない
- もし、Executorが SerialなExecutorであれば、すべてのJobの実行は完全に順序付けられなければならないり。`enqueue(_:`)で同じ Executorに投入された2つの異なるJobAとBに対して、AのすべてのイベントがBのすべてのイベントの前に起こるか、BのすべてのイベントがAのすべてのイベントよりも先に起こらなければならない(happen-before)
  - 例えば、一方のJobの優先度が他方より高い場合など。しかし、A と B はそれぞれ独立して、他方が実行される前に完了しなければならないことに注意する必要がある

### Serial Executors

また、Actor がタスク(Job)の連続実行を保証するために使用する`SerialExecutor`プロトコルを定義している。

```swift

/// Job を1つずつ実行するサービスであり，具体的には．
/// Job 実行間の相互排他を保証する．
///
/// Serial Executorは、Actor(またはDistributed Actor)に提供することができる。
/// その Actor で実行されるすべてのタスクが、この Executorに enqueue されることを保証するため。
///
/// Serial Executorは一般に、Job の特定の実行順序を保証しない。
/// タスクの優先順位や他のメカニズムを用いて、自由に順序を変更することができる。
@available(SwiftStdlib 5.1, *)
public protocol SerialExecutor: Executor {
  /// この Executorの値を最適化された形に変換して borrow する。
  /// Executorへの参照にする。
  @available(SwiftStdlib 5.1, *)
  func asUnownedSerialExecutor() -> UnownedSerialExecutor
}

@available(SwiftStdlib 5.9, *)
extension SerialExecutor {
  // 多くの実装はこのデフォルト実装で十分である.
  func asUnownedSerialExecutor() -> UnownedSerialExecutor {
    UnownedSerialExecutor(ordinary: self)
  }
}
```

`SerialExecutor`は、参照カウントのオーバーヘッドを発生させずに Executorを渡すために Swift ランタイムによって使用される `UnownedSerialExecutor`でそれ自体をラップすること以外、新しい API を導入していない。

```swift
/// `SerialExecutor`への未所有の参照値。
/// これは、コアのスケジューリング処理で内部的に使用される最適化された型である。
/// 抽象的にActor を扱う場合でも、不必要な参照カウントを避けるために、unowned 参照となっている。
/// 一般に、これを可能にするためにコア操作に課される特別な制約がある。
/// 例えば、Actor を生存させるためには Actor に関連する Executorも生存させておかなければならない。
/// これらが異なるオブジェクトである場合、Executorは Actor から強く参照されなければならない。
@available(SwiftStdlib 5.1, *)
@frozen
public struct UnownedSerialExecutor: Sendable {
  public init<E: SerialExecutor>(ordinary executor: __shared E)
}
```

`SerialExecutor`は、カスタムの Executorを使用するときに発生するスレッドスイッチの量を減らすことができる 「スイッチング」 をサポートするように拡張される可能性がある。この拡張の議論については、将来の方向性を参照。

### Jobs

`Job`は、Executorが実行すべきタスクのかたまりを表現したものです。例えば、`Task`は事実上、実行するために Executorに enqueue された一連の Job から構成されています。Swift Concurrencyで作成される最も一般的なタイプのJobが「部分タスク」であるにもかかわらず、この API を「部分タスク」だけに制限したり、タスクとあまり密接に結びつけたくないので、「Job」という名前が選ばれました。

Swift Concurrencyが何らかのタスクを実行する必要があるときはいつでも、Job が実行されるべき特定のExecutor上で `UnownedJobs`をenqueueします。`UnownedJob`型は、Swiftの低レベルの Job 表すopaqueなラッパーです。それは意味のある検査やコピーはできず、決して2回以上実行されてはいけません。

```swift
@available(SwiftStdlib 5.9, *)
@_moveOnly
public struct Job: Sendable {
  /// このJobの優先順位を返す。
  ///
  /// Jobがタスクの場合、Jobの優先度はTaskPriorityと等しくなる。
  public var priority: Priority { get }
}

@available(SwiftStdlib 5.9, *)
extension Job {
 // TODO: JobPriorityは、技術的にはTaskPriorityと同じ。
  // しかし、"Task..." という名前のAPIを、タスクでないJobに対して公開するのは、間違っているように感じる。
  // タスクだけではない可能性がある。
  //
  // TODO: 別の方法として、ここで`Priority = TaskPriority`という型付けをすることができる。
  public struct Priority {
    public typealias RawValue = UInt8
    public var rawValue: RawValue

    /// この`UnownedJob/Priority`は`TaskPriority`に変換される。
    public var asTaskPriority: TaskPriority? { ... }

    public var description: String { ... }
  }
}
```

この言語機能の最初の初期の反復におけるmove-only型にはまだ多くの制限があるため、私たちは `UnownedJob`型も提供している。これはJobの安全でない「未所有」バージョンである。`UnownedJob`を必要とする理由の1つは、Jobが一般的なコンテキストで使用される場合。なぜなら、現在利用可能なmove-only型の初期バージョンでは、そのような型は一般的なコンテキストで表示できないためである。例えば、[Job]を使った素朴なキュー実装はコンパイラに拒否されるが、`UnownedJob`(つまり[UnownedJob])を使った表現なら可能。

```swift

@available(SwiftStdlib 5.1, *)
@frozen
public struct UnownedJob: Sendable, CustomStringConvertible {
  /// move-onlyのJobを消費して、安全でない、unownedなJobを作成する。
  ///
  /// これは現在、コレクションにJobを格納することを意図しているときに必要かもしれない。
  /// あるいは、move-only型の初期の実装の制約でジェネリックを使用しない場合に必要かもしれまない。
  @available(SwiftStdlib 5.9, *)
  @usableFromInline
  internal init(_ job: __owned Job) { ... }

  @available(SwiftStdlib 5.9, *)
  public var priority: Priority { ... }

  public var description: String { ... }
}
```

Jobのdescriptionには、JobまたはタスクIDが含まれる。これは、タスクのdump、Instrumentsや他のデバッグツール(`swift-inspect`など)のタスクリストと関連付けるために使用される。タスクIDは、タスクに割り当てられた一意の番号で、スケジューリング問題をデバッグする際に役立つ。これは、現在、タスクを検査する際にInstrumentsなどのツールで公開されているものと同じIDで、デバッグログとプロファイルツールの観察結果と関連付けることができる。

最終的には、Executorは実際にJobを実行したいと思うだろう。それは、それがenqueueされたときにすぐに行うかもしれないし、別のスレッドかもしれない。これはExecutorに完全に決定が任されている。Jobの実行は `SerialExecutor`プロトコルで提供されている `runJobSynchronously`メソッドを呼び出すことで行われる。

`Job`の実行は`Job`をconsumeするため、誤って同じ`Job`を2回実行することはできない。それが許されるなら、それはundefineな挙動になるだろう。

```swift
@available(SwiftStdlib 5.9, *)
extension SerialExecutor {
  /// Jobを同期的に実行する。
  ///
  /// これはJobを消費する。
  @_alwaysEmitIntoClient
  @inlinable
  public func runJobSynchronously(_ job: __owned Job) {
    _swiftJobRun(UnownedJob(job), self)
  }

  /// Jobを同期的に実行する。
  ///
  /// Jobは「一度だけ」実行する。実行後にJobにアクセスすると不定の挙動になる。
  /// - Parameter job:
  @_alwaysEmitIntoClient
  @inlinable
  public func runUnownedJobSynchronously(_ job: UnownedJob) {
    _swiftJobRun(job, self)
  }
}

@available(SwiftStdlib 5.9, *)
extension UnownedSerialExecutor {
  @_alwaysEmitIntoClient
  @inlinable
  public func runJobSynchronously(_ job: __owned Job)

  @_alwaysEmitIntoClient
  @inlinable
  public func runUnownedJobSynchronously(_ job: UnownedJob)
}
```

### カスタムのSerialExecutorを持つActor

すべてのActorは暗黙に Actor(またはDistributedActor)プロトコルに準拠しており、これらのプロトコルには `unownedExecutor`というプロパティの形で実行するExecutorのカスタマイズポイントが含まれている。

ActorのExecutorは、`Executor`プロトコルを詳細にした`SerialExecutor`プロトコルに準拠しなればならず、Actorの排他制御の保証を実装するために、十分な保証を提供する。将来的には、`SerialExecutor`は 「スイッチング」をサポートするように拡張されるかもしれない。これは、Executorが現在実行中のスレッドを互いに「貸し出す」ことができる互換性を持つActor間の呼び出しで、スレッドスイッチングを回避するための技術である。この提案では、このスイッチングのセマンティクスはカバーしない。

Actorは、タスクを実行するためにどの`SerialExecutor`を使用すべきかを選択することは、`Actor`および`DistributedActor`プロトコルの`unownedExecutor`要件によって表現される。

```swift
public protocol Actor: AnyActor {
  /// このActorのExecutorを、最適化されたunownedなリファレンスとして取得する。
  ///
  /// このプロパティは常に、与えられたActorのインスタンスに対して同じExecutorとして評価されなければならず。
  /// Actorを保持することでExecutorを生かし続けなければならない。
  ///
  /// このプロパティは、タスクをこのActorにスケジュールする必要がある場合に暗黙的にアクセスされる。
  /// このアクセスは、厳密に必須ではない場合に、他のタスクと統合されたり、削除されたり、並べ替えられたりする可能性がある
  /// したがって、このプロパティ内で目に見える副作用を生じることは強く推奨しない。
  nonisolated var unownedExecutor: UnownedSerialExecutor { get }
}

public protocol DistributedActor: AnyActor {
  /// ...
  nonisolated var unownedExecutor: UnownedSerialExecutor { get }
}

```

> 注意: `AnyActor`はランタイムに存在しない「マーカープロトコル」であり、プロトコル要件を持つことができないため、このプロトコル要件を`AnyActor`で直接表現することはできません。

コンパイラは、明示的な実装が提供されない限り、すべての`(distributed) actor`宣言のこの要件のデフォルト実装を合成する。これは、プラットフォームに適切なメカニズム(例えば、`Dispatch`)を使用するデフォルトの`SerialExecutor`を使用する。この合成されたデフォルトの実装を使用するActorは、「デフォルトのActor」(`SerialExecutor`のデフォルト実装を使用するActor)と呼ばれる。

開発者は、Actorのこのプロトコル要件を実装することで、Actorが使用するExecutorを宣言単位でカスタマイズすることができる。例えば、`MainActor`の`sharedUnownedExecutor`staticプロパティのおかげで、同じ`SerialExecutor`(つまり「メインスレッド」)の使用が保証されている他のActorを宣言することが可能である。


```swift

(distributed) actor MainActorsBestFriend {
  nonisolated var unownedExecutor: UnownedSerialExecutor {
    MainActor.sharedUnownedExecutor
  }
  func greet() {
    print("Main-friendly...")
    try? await Task.sleep(for: .seconds(3))
  }
}

@MainActor
func mainGreet() {
  print("Main hello!")
}

func test() {
  Task { await mainGreet() }
  Task { await MainActorsBestFriend().greet() }
  // 合法な実行:
  // 1)
  //   - Main-friendly... hello!
  //   - Main hello!
  // 2)
  //   - Main hello!
  //   - Main-friendly... hello!
}
```

上記のスニペットは、`MainActor`と`MainActorsBestFriend`は異なるActorであり、したがって一般に同時実行が許可されているが、これらは同じMainActor(メインスレッド)の`SerialExecutor`を共有しているので、決して同時実行されないことを説明している。

また、このようにライブラリ固有のデフォルトのExecutorがすでに定義されているプロトコルを、ライブラリが提供することも可能である。

```swift

protocol WithSpecifiedExecutor: Actor {
  nonisolated var executor: LibrarySpecificExecutor { get }
}

protocol LibrarySpecificExecutor: SerialExecutor {}

extension LibrarySpecificActor {
  /// Actorの実行を調整するSerialExecutorとして、WithSpecifiedExecutorExecutorを設定する
  nonisolated var unownedExecutor: UnownedSerialExecutor {
    executor.asUnownedSerialExecutor()
  }
}

/// シンプルな「呼び出しスレッド上で実行する」JobのExecutor。
/// 一般的にExecutorは、代わりに別のスレッドでJobをenqueueし、処理する必要がある。
/// 必要でないときに効率的にhopを回避する方法は、「Executorのスイッチング」機能の一部として提供される予定。
final class InlineExecutor: SpecifiedExecutor, CustomStringConvertible {
  public func enqueue(_ job: __owned Job) {
    runJobSynchronously(job)
  }
}
```

これは、そのようなActorを実装するライブラリのユーザが、そのActorに対してライブラリ固有のExecutorを提供することを保証する。

```swift
actor MyActor: WithSpecifiedExecutor {

  nonisolated let executor: SpecifiedExecutor

  init(executor: SpecifiedExecutor) {
    self.executor = executor
  }
}
```

ライブラリは、このようなExecutorのデフォルトの実装も提供することができる。

### Executorのアサーション

[Pitch](https://forums.swift.org/t/pitch-unsafe-assume-on-mainactor/63074/)で紹介した`unsafeAssumeOnMainActor`APIと同様に、Custom Executorの導入により、他のConcurrencyランタイムから確信を持ってSwift Concurrencyに移行できるという同じ理論的根拠が、Custom Executorを持つActorにも適用できる。

まだ Swift Concurrencyを使用していないイベントループの重いコードでよくあるパターンは、同期コードの一部が想定したイベントループで実行されることを保証/検証することである。Executorをカスタマイズ可能にする目的の一つは、そのようなイベントループを `SerialExecutor`に適合させることで Swift Concurrencyを採用できるようにすることなので、Actorと Swift Concurrencyを完全に採用する方向に進んでいる間にライブラリが確信を得るために、コードが本当に適切なExecutorで実行されているかどうかをチェックできるようにすることは有用なことである。

例えば、Swift NIO は、同期チェックのオーバーヘッドを避けるために、いくつかの同期メソッドで意図的に同期チェックを避けているが、DEBUGモードでは、与えられたコードが期待されるイベントループで実行されていることをassertしている。

```swift
private var _channel: Channel
internal var channel: Channel {
  self.eventLoop.assertInEventLoop()
  assert(self._channel != nil || self.destroyed)
  return self._channel ?? DeadChannel(pipeline: self)
}
```

Dispatchベースのシステムにも同様の機能があり、`dispatchPrecondition`というAPIがある。

```swift
func checkIfMainQueue() {
  dispatchPrecondition(condition: .onQueue(DispatchQueue.main))
}
```

一般的に、Swift Concurrencyでは、特定のActorにメソッドを置くか、Global Actorアノテーションを使用することによって、正しい実行コンテキストにあることを静的に保証できるため、そのようなpreconditionは必要ない。

```swift
@MainActor
func definitelyOnMainActor() {}

actor Worker {}
extension Worker {
  func definitelyOnWorker() {}
}
```

時には、特に既存のコードベースを Swift Concurrencyに移行するとき、それが期待されるExecutor上で実行されているかどうかを同期コードでassertする機能がSwift Concurrencyへの移行中に開発者にもっと確信をもたらすことができることを私たちは認識している。これらの移行をサポートするために、我々は以下の方法を提案する。

```swift

/// 現在のタスクが期待されるExecutor上で実行されているかどうかをチェックする。
///
/// 複数のActorが同じSerialExecutorを共有する場合、このassertionは特定のActorインスタンスではなく、Executorをチェックすることに注意。
///
/// 一般的に、Swiftのプログラムは、Global ActorやCustom Executorを使用するなどして、特定のExecutorが使用されていることが静的にわかるように構築する必要がある。
/// しかし、いくつかのAPIでは、特にこのようなassertionを頻繁に使用する他のランタイムからSwift Concurrencyに移行する場合、このための追加の実行時チェックを提供することが有用な場合がある。
@available(SwiftStdlib 5.9, *)
public func preconditionTaskIsOnExecutor(
  _ executor: some Executor,
  _ message: @autoclosure () -> String = "",
  file: String = #fileID, line: UInt = #line)

// preconditionTaskIsOnExecutor(_:_:file:line)と同じだが、DEBUGモードのみ
@available(SwiftStdlib 5.9, *)
public func assertTaskIsOnExecutor(
_ executor: some Executor,
_ message: @autoclosure () -> String = "",
file: String = #fileID, line: UInt = #line)
```

このAPIは2つのActorがExecutorを共有するときは常にtrueを返すことに注意する必要がある。SerialExecutorを共有することは同じ分離された領域で実行することを意味するが、これは動的にしか分からないため、そのようなActor間の呼び出しには待ち時間が必要である。

```swift
actor A {
  nonisolated var unownedExecutor: UnownedSerialExecutor { MainActor.sharedUnownedExecutor}

  func test() {}
}

actor B {
  nonisolated var unownedExecutor: UnownedSerialExecutor { MainActor.sharedUnownedExecutor}

  func test(a: A) {
    await a.test() // awaitが必要。なぜなら、同じExecutorにあることを静的に知ることができない。
  }
}
```

将来的には、Actor間の関係が静的に表現されている場合(ActorBがAの特定のインスタンスと同じ`SerialExecutor`上にあることを宣言する)ような特定のActorインスタンス間では`await`が必要ないような静的チェックを可能にすることが考えられる。しかし、そのような機能はこの最初の提案の範囲ではなく、今は動的な側面のみを提案している。

現時点では、Dispatchと同様、これらのAPIは「assert」/「precondition」バージョンしか提供していない。また、現在のところ、特定のExecutorにいるかどうかを動的に確認する方法は公開されていない。

## Actor executorsの推論

> 注：このAPIは当初、Custom Executorとは別に提案されましたが、この機能に取り組むうちに、Custom ExecutorやExecutorでのアサーションととても密接に関係しているかがわかった。最初のピッチのスレッドは[Pitch: MainActorの安全でないアサート](https://forums.swift.org/t/pitch-unsafe-assume-on-mainactor/63074/)。
この提案の改訂版では、`assumeOnMainActorExecutor(_:)`メソッドを導入し、同期コードがMainActorのExecutorのコンテキスト内で呼び出されたことを安全に推論できるようにした。非同期コードでこの要件を満たすには、`@MainActor`を使用して関数をアノテートし、静的にこの要件を保証するのが正しい方法だからである。

同期コードは、このassumeメソッドを使用することで、MainActorのExecutor上で実行されていると推論することができる:

```swift
/// MainActorの実行中にこの関数が呼び出されたかどうか、実行時にテストを行う。
/// その後、操作が呼び出され、その結果が返される。
/// 
/// - 注意：MainActorとの競合状態を防ぐため、別の実行コンテキストから呼び出された場合、
/// このメソッドはクラッシュする。
@available(*, noasync)
func assumeOnMainActorExecutor<T>(
    _ operation: @MainActor () throws -> T,
    file: StaticString = #fileID, line: UInt = #line
) rethrows -> T
```

assertやprecondition APIと同様に、このチェックはActorのExecutorに対して行われるため、複数のActorが同じExecutor上で実行されている場合、そのActorから起動された同期コードでもこのチェックは成功しする。つまり、以下のようなコードも正しい:

```swift

func check(values: MainActorValues) /* synchronous! */ {
  // values.get("any") // error: main actor isolated, cannot perform async call here
  assumeOnMainActorExecutor {
    values.get("any") // 正しい&安全
  }
}

actor Friend {
  var unownedExecutor: UnownedSerialExecutor { 
    MainActor.sharedUnownedExecutor
  }
  
  func callCheck(values: MainActorValues) {
    check(values) // 正しい
  }
}

actor Unknown {
  func callCheck(values: MainActorValues) {
    check(values) // MainActorのExecutor上で実行していないためクラッシュ
  }
}

@MainActor
final class MainActorValues {
  func get(_: String) -> String { ... } 
}
```

> 注: `@SomeActor () -> T`関数型のGlobalActorの分離を抽象化することはできないので、現在、私たちはこのAPIのバージョンを任意のGlobalActorに対して提供していない。しかし、マクロを使用して今日そのようなAPIを実装することは可能であり、十分に重要だと見なされればフォローアップの提案で説明できるはず。このようなAPIは`#assumeOnGlobalActorExecutor(GlobalActor.self)`と記述する必要がある。

MainActorに特化したAPIに加えて、同じ形のAPIが独自のActorに提供され、提供されているActorと同じSerialExecutorで実行されることが保証されることで同時アクセス違反が起こらない場合、分離したMainActorの参照を得ることができる。

```swift
@available(*, noasync)
func assumeOnActorExecutor<Act: Actor T>(
    _ operation: (isolated Act) throws -> T,
    file: StaticString = #fileID, line: UInt = #line
) rethrows -> T

@available(*, noasync)
func assumeOnLocalDistributedActorExecutor<Act: DistributedActor T>(
    _ operation: (isolated Act) throws -> T,
    file: StaticString = #fileID, line: UInt = #line
) rethrows -> T
```

これらのassumeメソッドは、チェックが特定のインスタンスではなくActorのExecutorについて実行されるという意味で、先ほど説明した`assumeOnMainActorExecutor`と同じセマンティクスを持つ。言い換えれば、多くの独自のActorが同じSerialExecutorを共有する場合、現在実行中のExecutorが同じであるとわかっている限り、このチェックはそれらのそれぞれについてパスするだろう。

## Executorの等価チェックの詳細

前の2つのセクションでは、様々なassert、precondition、assume APIを説明したが、これらはすべて「同じSerialExecutor」という概念に依存しています。デフォルトでは、すべてのActorはそれ自身のSerialExecutorインスタンスを取得し、そのようなインスタンスはそれぞれ一意である。したがって、Executorを共有することなく、各ActorのSerialExecutorはそれ自体に対してユニークであり、したがって、precondition APIは、チェックがExecutorの識別子に対して実行されるにもかかわらず、実際は「我々はこの特定のActorにいるか」をチェックします。

### 同じSerialExecutorに委譲する一意のExecutor

この提案で議論したいのは、「同じExecutor」をチェックする2つのケースである。まず、いくつかのActorがSerialExecutorを共有したいとしても、開発者は様々なpreconditionチェックのためにこの「同じSerialExecutor上の異なるActorは同じ実行コンテキストにある」というセマンティクスを受け取りたくない場合がある。

この解決策は、Executorの実装方法にあり、具体的には、他の既存のExecutorの周りにラッパーExecutorを提供することが常に可能である。こうすることで、たとえ同じSerialExecutor上でスケジューリングすることになったとしても、一意なExecutorIDを割り当てることができるようになる。例として、次のようなものがある。

```swift
final class SpecificThreadExecutor: SerialExecutor { ... }

final class UniqueSpecificThreadExecutor: SerialExecutor {
  let delegate: SpecificThreadExecutor
  init(delegate: SpecificThreadExecutor) {
    self.delegate = delegate
  }
  
  func enqueue(_ job: consuming Job) {
    delegate.enqueue(job)
  }
  
  func asUnownedSerialExecutor() -> UnownedSerialExecutor {
    UnownedSerialExecutor(ordinary: self)
  }
}

actor Worker {
  let unownedExecutor: UnownedSerialExecutor
  
  init(executor: SpecificThreadExecutor) {
    let uniqueExecutor = UniqueSpecificThreadExecutor(delegate: executor)
    self.unownedExecutor = uniqueExecutor.asUnownedSerialExecutor()
  }
  
  func test(other: Worker) {
    assert(self !== other)
    assertOnActorExecutor(other) // クラッシュを期待
    // 結果的に同じExecutorに処理を委譲するが`other` は異なる一意のExecutorである
  }
}
```

### 同じ実行コンテキストを提供する異なるExecutor

また、SerialExecutorのIDの互換性チェックをするオプショナルの拡張を導入し、Executorがチェックに参加することを可能にする。これは、先ほど説明したことと逆の状況を扱うためである。つまり異なるExecutorが実際には同じ排他的シリアル実行コンテキスト上にあり、これらのassertion APIで引っかからないためにSwiftランタイムに知らせたいときである。

異なる一意のExecutorインスタンスを持つが同じ排他的シリアル実行コンテキスト上で動作する一例は、異なるキューを「ターゲット」することができるDispatchQueueがある。言い換えれば、DispatchQueue`Q1`と`Q2`が同じキュー`Qx`（あるいは「Main」DispatchQueue）をターゲットにすることが可能である。

この機能を簡単に利用するために、`UnownedSerialExecutor`を公開する場合、Executorは`init(complexEquality:)`イニシャライザを使用しなければならない。

```swift
extension MyQueueExecutor { 
  public func asUnownedSerialExecutor() -> UnownedSerialExecutor {
    UnownedSerialExecutor(complexEquality: self)
  } 
}
```

一意のイニシャライザは、「もしExecutorのポインタが同じなら、それは同じExecutorの排他的実行コンテキスト上にある」というExecutorの等価チェックは現在の高速パスのセマンティクスを維持するが、等価チェックが失敗した場合に「深いチェック」をするコードパスを追加する。

「これは同じ（または互換性のある）シリアル実行コンテキストですか」というチェックを実行するとき、Swiftランタイムは、最初にExecutorオブジェクトへの生のポインタを比較する。それらが等しくなく、問題のExecutorが`complexEquality`を持つ場合、いくつかの追加の型チェックの後、次の`isSameExclusiveExecutionContext(other:)`メソッドが呼び出される。

```swift
protocol SerialExecutor {

  // ...以前に議論したプロトコル要件 ...

  /// このExecutorが複雑な等価セマンティクスを持っていて、ランタイムが2つのExecutorを比較する必要がある場合、
  /// まず通常のポインタベースの等価チェックを試み、失敗したら両方のExecutorの型を比較し、同じであれば、最後にこの方法を呼び出す。
  /// このメソッドは細心の注意を払って実装する必要がある。
  /// 誤って `true` を返すと、別の実行コンテキスト（例えばスレッド）から、別のActorによって分離されていることを意図していたコードを実行できてしまうからである。
  /// このチェックは、Executorの切り替えを行う際には使用されない。
  /// `preconditionTaskOnActorExecutor`、`preconditionTaskOnActorExecutor`、`assumeOnActorExecutor`や同様のAPIで同じ「排他的シリアル実行コンテキスト」についてassertする場合に使用する。
  /// - Parameter other:
  /// returns: `self` と `other` のExecutorが相互に排他的であり、一方を想定したコードを他方で実行することがConcurrencyの観点から安全である場合、true を返す。
  func isSameExclusiveExecutionContext(other: Self) -> Bool
}

extension SerialExecutor {
  func isSameExclusiveExecutionContext(other: Self) -> Bool {
    self === other
  }
}
```

このAPIは、例えば将来のDispatchQueueのようなExecutorが「深い」チェックを実行し、例えば両方のExecutorが実際に同じスレッドやキューをターゲットにしていればtrueを返し、したがって適切に分離された相互排他的なシリアル実行コンテキストを保証することができる。

APIは、ユーザコードへのこのやや重い呼び出しを使用して完全に無関係なExecutorを比較することを避けるために、両方のExecutorが同じ型でなければならないことを明示的に強制する。上述したAPIを使用する目的で、Executorを比較するための具体的なロジックは以下の通りである

Executorの型(ExecutorRef、特にImplementation/witnessテーブルフィールドに格納するビット)を検査し、両方が同じであれば、次のようになる。

- **「通常」**（または「一意」として知られる）、これは間違いなく 「ルート」Executorと考えることができる
- 作成
  - 現在の`UnownedSerialExecutor.init(ordinary:)`を使う
- 比較
  - 2つのExecutorのポインタを直接比較する
    - 結果を返す
- **complexEquality**は、「内側」Executorとみなされるかもしれない、例えばより深く等価チェックをするために正確な識別子が必要なもの
  - 作成
    - ランタイムが認識できる特定のビットを設定し、必要に応じてのcomplexEqualityを行うコードパスに到達する`UnownedSerialExecutor(complexEquality:)`がある
    - 比較
      - 両方のExecutorが`complexEquality`である場合、2つのExecutorの識別子を直接比較する。
        - trueの場合、return(これはファストパスで、通常のExecutorと同じである)する
      - Executorの証人テーブルを比較し、互換性があるかどうかを確認する
        - falseの場合、returnする
      - Executorに実装された`executor1.isSameExclusiveExecutionContext(executor2)`を呼び出す
        - 結果を返す

これらのチェックは、タスクの切り替えを完全に最適化するには不十分である可能性が高く、将来的にはタスクの切り替えを最適化する他のメカニズムが提供される予定である(「将来の検討」参照)。

### デフォルトのSwiftランタイムExecutor

SwiftのConcurrencyでは、MainActorのExecutorやデフォルトのglobal concurrent Executorなど、すでに多くのデフォルトExecutorを提供しており、すべての(デフォルトの)Actorは、それ自身のActorごとにインスタンス化した`SerialExecutor`インスタンスがある。

`MainActor`のExecutorは、`MainActor`上の`sharedUnownedExecutor`staticプロパティを介して利用可能。

```swift
@available(SwiftStdlib 5.1, *)
@globalActor public final actor MainActor: GlobalActor {
  public nonisolated var unownedExecutor: UnownedSerialExecutor
  public static var sharedUnownedExecutor: UnownedSerialExecutor
}
```

つまり、`MainActor`が実行しているのと同じExecutorに他のActorを乗せることは、次のパターンを使って可能。

```swift
actor Friend {
  nonisolated var unownedExecutor: UnownedSerialExecutor {
    MainActor.sharedUnownedExecutor
  }
}
```

デフォルトのglobal concurrent Executorは、現在、置き換えができない。

## ソース互換性

これらのAPIの多くは、Swift Concurrencyの最初の導入以来、既存のpublicな型である(そして、back-deploymentライブラリに含まれている)。この提案のすべての型と一部の機能は、この提案のすべての型と一部の機能は、既存の実行APIとの間でソースと動作の互換性を保つことができる方法で設計されている。

以前からある("unowned")APIを非推奨としながら、ソース互換性のある方法でmove-only JobベースのenqueueAPIを導入するために、特別な余地がある。

## ABI stabilityへの影響

Swift COncurrencyランタイムは、最初の導入以来、すでにExecutor、Job、タスクを使用しており、そのため、この提案は、既存のすべてのランタイムエントリーポイントと型とのABI互換性を維持している。

`SerialExecutor`の設計は現在、non reentrantなActorをサポートしておらず、常に同期的にディスパッチがするExecutorもサポートしていない(例えば、従来のmutexを取得するだけなど)。

この提案における新しいAPIと既存のAPIの関係をさらに説明するために、議論された型に`@available`を残すことを選択した。注目すべきは、Jobの実行のような操作については、この提案で導入される以前は公式なAPIが存在しないことである。

## APIレジリエンスへの影響

APIによっては、特定のExecutor上で実行されることに依存するものもあるが、この提案は、実装の詳細であるのとは対照的に、インターフェースでそれを形式化する努力をしないので、APIレジリエンスに影響を与えることはない。

将来、Actorのように宣言駆動の自動的なExecutorスイッチングに拡張されれば、APIレジリエンスに影響を与えるだろう。

## 代替案

ActorがCustom Executorにオプトインするために提案された方法は、タイプミスや同様のエラーによってActorが誤ってデフォルトExecutorを使用したままになるという意味でもろい。これは、ActorがデフォルトExecutorの使用を明示的にオプトインすることを要求することで完全に緩和することができるが、それは一般的なケースに対して受け入れがたい負担となるだろう。そうでなくても、宣言に特別な意味を持たせるような修飾子を用意し、コンパイラがその意味を認識しない場合に文句を言うことは可能である。しかし、このような名前依存のデザインを使用する既存の機能は、dynamic member lookup([SE-0195](https://github.com/rjmccall/swift-evolution/blob/custom-executors/proposals/0195-dynamic-member-lookup.md))のように多数存在しする。「特別な意義」修飾子は、より総合的に設計・検討されるべき。

## 将来の検討

### MainActor Executorのオーバーライド

`MainActor`の特別なセマンティクスと非同期の`main`関数との相互作用のために、その`SerialExecutor`のカスタマイズは、他のExecutorのカスタマイズよりも少しトリッキーである。プログラムの`main`関数がメインスレッドで実行されることと、`MainActor`のコードもメインスレッドで実行されることの両方を保証しなければならない。これはまた、`main`関数が実際に終了コードを返すという面白い複雑さをもたらす。

`MainActor`と同様に、非同期`main`関数で使われる`SerialExecutor`をオーバーライドすることが可能なはず。正確なセマンティクスはまだ設計されていないが、私たちは、非同期処理が行われる前にメインのExecutorを置き換えることができるAPIを想定しており、この方法で`MainActor`から期待される同期実行を保証し続けることができる。

```swift
// DRAFT; Names of protocols or exact shape of such replacement API are non-final.

protocol MainActorSerialExecutor: [...]SerialExecutor { ... }
func setMainActorExecutor(_ executor: some MainActorSerialExecutor) { ... }

@main struct Boot { 
  func main() async { 
    // メインの「本来の」スレッドの上
    
    // 以下の呼び出しを行う必要がある。
    // - 中断ポイントに遭遇する前
    setMainActorExecutor(...) 
    
    // まだメインの「本来の」スレッドの上
    await hello() // RunLoopSerialExecutorのメインスレッドに「本来の」メインスレッドの制御を与える
    // メインスレッドのまま、選択されたMainActorExecutorで実行される。
  }
}

@MainActor
func hello() {
  // MainActor(メインスレッド)であることが保証される。
  // 選択されたMainActorのExecutor上で実行される
  print("Hello")
}
```

### Executorスイッチング

Executorスイッチングは、Actor/Executor間でhopしようとするとき、対象のExecutorが呼び出しスレッドを「引き継ぐ」ことができる場合に、不要なスレッドのhopを回避する機能である。これにより、Swift はスレッドhopを減らし、呼び出しのスケジューリングを最適化することができる。例えば、Actorが同じExecutorIDでスケジュールされ、それらがスイッチングできる場合、完全にスレッドhopを避けることができ、実行は複数のExecutorを通して "タスクに従う"ことが可能になる。

スイッチングの初期のスケッチは、以下のメソッドをExecutorプロトコルに追加することに焦点をあてていた。

```swift
// DRAFT; このスニペットで言及されている名前と特定のAPIは未定である

protocol SerialExecutor: Executor {
  // .....既存のAPI ...... 
  
  /// このExecutorが現在のスレッドを放棄し、そのスレッドで実行を開始することは可能か？
  /// そして、別のActorの実行を開始できるようにすることは可能か？
  var canGiveUpThread: Bool { get }

  /// canGiveUpThread() が以前にtrueを返したとすると、現在のスレッドを放棄する。
  func giveUpThread()

  /// 現在のActor上でタスクの実行を開始しようとする。
  /// 成功すればtrueを返す。
  func tryClaimThread() -> Bool
}
```

これらのAPIの形状が、この機能の潜在的なユースケースをすべてサポートするのに十分であると確信したときに、Custom Executorが効率的にスイッチングに参加できるように、これらの、または同様のAPIの追加を検討する予定である。

### タスクのExecutorの指定

タスクのExecutorを指定することは、意外と厄介な問題が多いので、当面はそのような機能は導入しないことにする。具体的には、`Task(startingOn: someExecutor) { ... }`にExecutorを渡すと、指定されたExecutorでタスクが開始される。しかし、このタスクのbodyがすべて`someExecutor`上で実行されることを期待されているのか(つまり、`await`の後に毎回ホップバックしなければならない)のか、あるいは、単にその上で開始すれば十分で、可能ならそれ以上のJobスケジューリングを回避し続ける(つまり、積極的にスイッチングしてもよい)のか、という細かいセマンティクスがある。

### DelegateActorプロパティ

以前のCustom Executorのピッチには、Actorが宣言できる`delegateActor`というコンセプトがあった。Actorは別のActorインスタンスと同じExecutor上で実行することができる。同時に、これはコンパイル時にコンパイラに十分な情報を提供し、両方のActorは同じ分離領域内にあると仮定することができ、それらのActor間の待ち時間をスキップすることができる(！)。Custom Executorでは動的に保持される性質が、この方法ではコンパイラと型システムによって静的に強化されることになる。

## 参考リンク

### Forums

- [[Pitch] Custom Actor Executors](https://forums.swift.org/t/pitch-custom-actor-executors/63135)
- [SE-0392: Custom Actor Executors](https://forums.swift.org/t/se-0392-custom-actor-executors/63599)

### プロポーザルドキュメント

- [Custom Actor Executors](https://github.com/ktoso/swift-evolution/blob/wip-custom-executors/proposals/nnnn-custom-actor-executors.md)

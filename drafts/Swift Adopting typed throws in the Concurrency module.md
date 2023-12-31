# Adopting typed throws in the Concurrency module

- [Adopting typed throws in the Concurrency module](#adopting-typed-throws-in-the-concurrency-module)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
    - [提案](#提案)
    - [`Task`の生成と完了(completion)](#taskの生成と完了completion)
      - [Continuations](#continuations)
        - [Taskのキャンセル](#taskのキャンセル)
      - [タスクグループ](#タスクグループ)
      - [タスクローカル](#タスクローカル)
      - [MainActor](#mainactor)
      - [`AsyncSequence` \& `AsyncIteratorProtocol`](#asyncsequence--asynciteratorprotocol)
      - [`AsyncThrowingStream`](#asyncthrowingstream)
      - [エラーをスローする変換シーケンス](#エラーをスローする変換シーケンス)
    - [ソース互換性](#ソース互換性)
    - [ABI安定への影響](#abi安定への影響)
    - [検討された代替案](#検討された代替案)
      - [`Clock.sleep`に型付きスローを採用する](#clocksleepに型付きスローを採用する)
      - [異なるエラー型をスローする変換シーケンス](#異なるエラー型をスローする変換シーケンス)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

 [SE-0413](https://github.com/apple/swift-evolution/blob/main/proposals/0413-typed-throws.md)はSwiftに型付きエラーを導入し、開発者がスローするエラーの型を明示的に記述できるようにした。しかし、標準ライブラリの`Concurrency`モジュールでの型付きスローの採用は延期された。

Swift-evolutionのスレッド:
- [Typed throw functions - Evolution / Discussion - Swift Forums](https://forums.swift.org/t/typed-throw-functions/38860
- [Status check: typed throws](https://forums.swift.org/t/status-check-typed-throws/66637)

## 内容

### 動機

`Concurrency`ライブラリには、型付きスローを採用することで、ユーザコードを通じてスローする型の維持に役立つ場所が数多くある。さらに、以前は2つの別々の実装が必要だった現在のメソッドや型のいくつかを統一できる。

### 提案

### `Task`の生成と完了(completion)

[`Task`](https://developer.apple.com/documentation/swift/task)APIは、`Result`と同様に`Failure`型を持つが、`Failure == any Error`と`Failure== Never`のオーバーロードのパターンを使用して、スローする/しないバージョンを処理する。例えば、`Task`のイニシャライザは以下のオーバーロードされたペアとして定義されている:

```swift
init(priority: TaskPriority?, operation: () async -> Success) where Failure == Never
init(priority: TaskPriority?, operation: () async throws -> Success) where Failure == any Error
```

これら2つのイニシャライザは、型付きスローを使用した1つのイニシャライザで置き換えられる。さらに、現在イニシャライザに`@discardableResult`アノテーションが付いていることが問題になっている。このアノテーションは、タスクの結果が格納されない場合に警告を表示しないが、これが本当に意味を持つのは、`Task<Void, Never>`または`Task<Never, Never>`の場合だけある。そこで、以下の新しいイニシャライザを追加することを提案する。

```swift
public init(
  priority: TaskPriority? = nil,
  operation: @Sendable @escaping () async throws(Failure) -> Success
)

@discardableResult
public init(
  priority: TaskPriority? = nil,
  operation: @Sendable @escaping () async throws(Failure) -> Success
) where Success == Void, Failure == Never

@discardableResult
public init(
  priority: TaskPriority? = nil,
  operation: @Sendable @escaping () async throws(Failure) -> Success
) where Success == Never, Failure == Never
```

その結果、表現力が増し(型付けされたエラーの情報が維持される)、かつシンプルになった。同じ変換をdetached タスクを生成する`detached`関数にも適用でき、2つのオーバーロードは以下のように置き換えられる:

```swift
public static func detached(
  priority: TaskPriority? = nil,
  operation: @Sendable @escaping () async throws(Failure) -> Success
)

@discardableResult
public static func detached(
  priority: TaskPriority? = nil,
  operation: @Sendable @escaping () async throws(Failure) -> Success
) where Success == Void, Failure == Never

@discardableResult
public static func detached(
  priority: TaskPriority? = nil,
  operation: @Sendable @escaping () async throws(Failure) -> Success
) where Success == Never, Failure == Never
```

`Task`の`value`プロパティも同様にオーバーロードされる：

```swift
extension Task where Failure == Never {}
  var value: Success { get async }
}
extension Task where Failure == any Error {
  var value: Success { get async throws }
}
```

これらは、以下のように置き換えられる:

```swift
extension Task {
  var value: Success { get async throws(Failure) }
}
```

#### Continuations

現在、各Continuationの作成メソッドには、エラーをスローする/しないに対応するために 2 つのバリエーションがある。しかし、メソッド名に `Throwing` を付けないほうが自然な表記であるため、以下の2つの新しいメソッドを追加することを提案する:

```swift
public func withCheckedContinuation<T, Failure: Error>(
    function: String = #function,
    _ body: (CheckedContinuation<T, Failure>) -> Void
) async throws(Failure) -> T

public func withUnsafeContinuation<T, Failure: Error>(
  _ fn: (UnsafeContinuation<T, Failure>) -> Void
) async throws(Failure) -> T
```

##### Taskのキャンセル

`Task`APIのいくつかは、`CancellationError`のみをスローようにドキュメントに書かれており、型付きスローを採用できる。例えば、`checkCancellation`の場合:

```swift
public static func checkCancellation() throws(CancellationError)
```

さらに、`withTaskCancellationHandler`メソッドにも型付きスローを採用できる:

```swift
public func withTaskCancellationHandler<T, Failure: Error>(
  operation: () async throws(Failure) -> T,
  onCancel handler: @Sendable () -> Void
) async throws(Failure) -> T
```

#### タスクグループ

タスクグループのAPIも型付きスローをサポートする必要がある。ここで重要なのは、`Failure`には2つの層があることだ。第一に、子タスク自身によって生成されるFailure。第二に、`withThrowingTaskGroup`API自身が返すFailureである。そこで、以下のAPIを追加することを提案する:

```swift
public func withTaskGroup<ChildTaskResult, ChildTaskFailure: Error, GroupResult, GroupFailure: Error>(
  of childTaskResultType: ChildTaskResult.Type,
  childTaskFailureType: ChildTaskFailure.Type = Never.self,
  returning returnType: GroupResult.Type = GroupResult.self,
  throwing failureType: GroupFailure.Type = Never.self,
  body: (inout ThrowingTaskGroup<ChildTaskResult, ChildTaskFailure>) async throws(GroupFailure) -> GroupResult
) async throws(GroupFailure) -> GroupResult
```

さらに、`ThrowingTaskGroup`APIに型付きスローを追加する:

```swift
public struct ThrowingTaskGroup<ChildTaskResult: Sendable, Failure: Error> {
  public mutating func next() async throws(Failure) -> ChildTaskResult?
  public mutating func waitForAll() async throws(Failure)
}

struct ThrowingTaskGroup<ChildTaskResult: Sendable, Failure: Error> {
  public struct Iterator: AsyncIteratorProtocol {
    public mutating func next() async throws(Failure) -> Element? 
  }
}
```

#### タスクローカル

`withValue`メソッドにも型付きスローを採用する:

```swift
public final class TaskLocal<Value: Sendable> {
  public func withValue<R, Failure: Error>(
    _ valueDuringOperation: Value,
    operation: () async throws(Failure) -> R,
    file: String = #fileID,
    line: UInt = #line
  ) async throws(Failure) -> R 
}
```

#### MainActor

`MainActor`は静的なrethrowする`run`メソッドを提供しているが、このメソッドにも型付きスローを採用する:

```swift
public static func run<T: Sendable, Failure: Error>(
  resultType: T.Type = T.self,
  body: @MainActor @Sendable () throws(Failure) -> T
) async throws(Failure) -> T
```

#### `AsyncSequence` & `AsyncIteratorProtocol`

`AsyncSequence`イテレータは、非同期イテレータの`next()`操作の`throws`で説明されているように、イテレーション中にエラーをスローできる:

```swift
public protocol AsyncIteratorProtocol {
  associatedtype Element
  mutating func next() async throws -> Element?
}
```

このプロトコルに新しい`Failure` associatedtypeを追加することを提案する:

```swift
public protocol AsyncIteratorProtocol {
  associatedtype Element
  associatedtype Failure: Error = any Error
  mutating func next() async throws -> Element?
}
```

次に、型付きスローを採用した新しい`next2(TODO: 名前要検討)`メソッドをプロトコルに追加することを提案する。さらに、`next`と`next2`の両方にデフォルトの実装を追加することを提案する。後者は、APIやABIを破壊することなく型付きスローを採用することを可能にする。

```swift
public protocol AsyncIteratorProtocol {
  associatedtype Element
  associatedtype Failure: Error = any Error
  mutating func next() async throws -> Element?
  mutating func next2() async throws(Failure) -> Element?
}

extension AsyncIteratorProtocol {
    func next() async rethrows -> Element? {
      do {
        return try next2()
      } catch {
        throw error
      }
    }

    func next2() async throws(Failure) -> Element? {
      do {
        return try next()
      } catch {
        throw error as! Failure
      }
    }
}
```

新しい`Failure`associatedtypeにより、非同期シーケンスは、エラーをスローするかどうか(そしてどのようなエラーか)についての情報を失うことなく構成できる。

さらに新しい`Failure`型が導入されたことで、これらのプロトコルに[primary associated type](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md)を採用できる:

```swift
public protocol AsyncIteratorProtocol<Element, Failure> {
  associatedtype Element
  associatedtype Failure: Error = any Error
  mutating func next() async throws(Failure) -> Element?
  mutating func next2() async throws(Failure) -> Element?
}

public protocol AsyncSequence<Element, Failure> {
  associatedtype AsyncIterator: AsyncIteratorProtocol
  associatedtype Element where AsyncIterator.Element == Element
  associatedtype Failure where AsyncIterator.Failure == Failure
  func makeAsyncIterator() -> AsyncIterator
}
```

これにより、Opaque型(`some AsyncSequence<String, any Error>`)と存在型(`any AsyncSequence<Image, NetworkError>`)の両方で`AsyncSequence`が使用できるようになる。

#### `AsyncThrowingStream`

エラースローしない`AsyncStream`はジェネリックな`Failure`パラメータを持たないため、型付きスローを採用できない。現在の`AsyncThrowingStream`のイニシャライザは`Failure`の型を`Error`に制約している。

```swift
public init(
  _ elementType: Element.Type = Element.self,
  bufferingPolicy limit: Continuation.BufferingPolicy = .unbounded,
  _ build: (Continuation) -> Void
) where Failure == Error

public init(
  unfolding produce: @escaping () async throws -> Element?
) where Failure == Error 
```

この`Failure`の制約をなくすために2つのイニシャライザを追加することを提案する:

```swift
public init(
  of elementType: Element.Type = Element.self,
  throwing failureType: Failure.Type = Failure.self,
  bufferingPolicy limit: Continuation.BufferingPolicy = .unbounded,
  _ build: (Continuation) -> Void
)

public init(
  unfolding produce: @escaping () async throws(Failure) -> Element?
)
```

さらに、`Failure`の制約をなくした`makeStream`メソッドを追加することも提案する:

```swift
@backDeployed(before: SwiftStdlib 5.9)
public static func makeStream(
    of elementType: Element.Type = Element.self,
    throwing failureType: Failure.Type = Failure.self,
    bufferingPolicy limit: Continuation.BufferingPolicy = .unbounded
) -> (stream: AsyncThrowingStream<Element, Failure>, continuation: AsyncThrowingStream<Element, Failure>.Continuation) {
  var continuation: AsyncThrowingStream<Element, Failure>.Continuation!
  let stream = AsyncThrowingStream<Element, Failure>(bufferingPolicy: limit) { continuation = $0 }
  return (stream: stream, continuation: continuation!)
}
```

#### エラーをスローする変換シーケンス

`Concurrency`モジュールには、`AsyncSequence`のアルゴリズムがいくつか含まれている。これらのアルゴリズムは具象型に支えられており、エラーをスローする/しない両方の種類がある。スローするタイプのものは一般的な`Failure`パラメータを持たないため、型付きスローを採用できない。しかし、ジェネリックな`Failure`パラメータを持つ新しい基底型を追加し、Opaque return typeを使用することを提案する。しかし、重要なことは、これらの新しいメソッドは新しい型を導入するため、バックデプロイできないことである。これらが、私たちが追加する新しい型付きスローAPIである:

```swift
extension AsyncSequence {
  public func map<Transformed>(
    _ transform: @Sendable @escaping (Element) async throws(Failure) -> Transformed
  ) -> some AsyncSequence<ElementOfResult, Failure>

  public func map<Transformed>(
    _ transform: @Sendable @escaping (Element) async throws(Failure) -> Transformed
  ) -> some (AsyncSequence<ElementOfResult, Failure> & Sendable) where Element: Sendable, Transformed: Sendable

  public func compactMap<ElementOfResult>(
    _ transform: @Sendable @escaping (Element) async throws(Failure) -> ElementOfResult?
  ) -> some AsyncSequence<ElementOfResult, Failure>

  public func compactMap<ElementOfResult>(
    _ transform: @Sendable @escaping (Element) async throws(Failure) -> ElementOfResult?
  ) -> some (AsyncSequence<ElementOfResult, Failure> & Sendable) where Element: Sendable, ElementOfResult: Sendable
  
  public func drop(
    while predicate: @Sendable @escaping (Element) async throws(Failure) -> Bool
  ) -> some AsyncSequence<Element, Failure>
  
  public func drop(
    while predicate: @Sendable @escaping (Element) async throws(Failure) -> Bool
  ) -> some (AsyncSequence<Element, Failure> & Sendable) where Element: Sendable

  public func filter(
    _ isIncluded: @Sendable @escaping (Element) async throws(Failure) -> Bool
  ) -> some AsyncSequence<Element, Failure>

  public func filter(
    _ isIncluded: @Sendable @escaping (Element) async throws(Failure) -> Bool
  ) -> some (AsyncSequence<Element, Failure> & Sendable) where Element: Sendable

  public func flatMap<SegmentOfResult: AsyncSequence>(
    _ transform: @Sendable @escaping (Element) async throws(Failure) -> SegmentOfResult
  ) -> some AsyncSequence<ElementOfResult, Failure> where SegmentOfResult.Failure = Failure

  public func flatMap<SegmentOfResult: AsyncSequence>(
    _ transform: @Sendable @escaping (Element) async throws(Failure) -> SegmentOfResult
  ) -> some (AsyncSequence<ElementOfResult, Failure> & Sendable) where SegmentOfResult.Failure = Failure, Element: Sendable, SegmentOfResult: Sendable

  public func prefix(
    while predicate: @Sendable @escaping (Element) async throws(Failure) -> Bool
  ) -> some AsyncSequence<Element, Failure> // これは現在理由もなくrethrowsになっている: https://github.com/apple/swift/issues/66922

  public func prefix(
    while predicate: @Sendable @escaping (Element) async throws(Failure) -> Bool
  ) -> some (AsyncSequence<Element, Failure> & Sendable) where Element: Sendable
}
```

### ソース互換性

影響なし

### ABI安定への影響

提案されている変更は純粋に追加的なものである。`for await`を使用して新しくコンパイルされたコードは、`AsyncIteratorProtocol`の`next`メソッドの代わりに、新しい`next2`メソッドを使用する。

### 検討された代替案

#### `Clock.sleep`に型付きスローを採用する

ほとんどの`Clock`の実装は、`sleep`メソッドから`CancellationError`のみをスローする。しかし、現在これを強制するものはなく、別のエラーをスローする実装があるかもしれない。`CancellationError`だけを投げるようにプロトコルを制限することは、破壊的な変更になるだろう。

#### 異なるエラー型をスローする変換シーケンス

新しく提案された`AsyncSequence`の変換メソッドは、クロージャがスローする型を`AsyncSequence`の基底の`Failure`型に制限する。クロージャは異なるエラー型をスローできるが、その場合、エラーのためのUnionのような型をスローする必要がある。

## 参考リンク

### Forums

- [[Pitch] Typed throws in the Concurrency module](https://forums.swift.org/t/pitch-typed-throws-in-the-concurrency-module/68210)

### プロポーザルドキュメント

- [Typed throws in the Concurrency module](https://github.com/apple/swift-evolution/pull/2197)
# Backpressure support for AsyncStream

- [Backpressure support for AsyncStream](#backpressure-support-for-asyncstream)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
      - [バックプレッシャー](#バックプレッシャー)
      - [マルチ／シングル コンシューマのサポート](#マルチシングル-コンシューマのサポート)
      - [ダウンストリーム コンシューマの終了](#ダウンストリーム-コンシューマの終了)
      - [アップストリーム プロデューサの終了](#アップストリーム-プロデューサの終了)
    - [提案](#提案)
      - [バックプレッシャーをサポートしたAsyncStreamの作成](#バックプレッシャーをサポートしたasyncstreamの作成)
      - [ダウンストリーム コンシューマの終了](#ダウンストリーム-コンシューマの終了-1)
      - [アップストリーム プロデューサの終了](#アップストリーム-プロデューサの終了-1)
    - [詳細](#詳細)
    - [他のルート非同期シーケンスとの比較](#他のルート非同期シーケンスとの比較)
      - [swift-async-algorithm: AsyncChannel](#swift-async-algorithm-asyncchannel)
      - [swift-nio: NIOAsyncSequenceProducer](#swift-nio-nioasyncsequenceproducer)
  - [ソース互換性](#ソース互換性)
  - [ABIの互換性](#abiの互換性)
  - [将来の検討事項](#将来の検討事項)
    - [順応型バックプレッシャー戦略](#順応型バックプレッシャー戦略)
    - [要素サイズ依存戦略](#要素サイズ依存戦略)
    - [`Async[Throwing]Stream.Continuation`の非推奨化](#asyncthrowingstreamcontinuationの非推奨化)
    - [WriterとAsyncWriterプロトコルの導入](#writerとasyncwriterプロトコルの導入)
  - [代替案](#代替案)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

<!-- 最後にTable of Contentsを入れる -->

## 概要

[SE-0314](https://github.com/apple/swift-evolution/blob/main/proposals/0314-async-stream.md)は、ルートの非同期シーケンスとして動作する新しい`Async[Throwing]Stream`型を導入した。これら2つの型は、デリゲートのような同期コールバックから非同期シーケンスへの橋渡しを可能にする。この提案では、バックプレッシャーシステムを非同期シーケンスにブリッジすることを目的として、非同期ストリームを構築する新しい方法を追加する。さらに、本提案は、消費タスクがキャンセルされた場合と、生産側が終了を指示した場合の両方のキャンセル動作を明確にすることを目的としている。

## 内容

### 動機

`AsyncSequence`プロトコルと`Async[Throwing]Stream`型を過去数年間にわたり広範囲に使用してきた結果、`AsyncSequence`実装がサポートする必要があるいくつかの重要な動作の詳細があることを学んだ。これらの動作とは:

1. バックプレッシャー
2. マルチ／シングル コンシューマのサポート
3. ダウンストリーム コンシューマの終了
4. アップストリーム プロデューサの終了

一般的に、`AsyncSequence`の実装は2種類に分けられる: `Async[Throwing]Stream`のような値の元となるルート非同期シーケンスと、`AsyncMapSequence`のような変換非同期シーケンス。ほとんどの変換非同期シーケンスは、動作を実装すべきベース非同期シーケンスにあらゆるdemandを転送するので、上記の動作を暗黙的に満たす。一方、ルート非同期シーケンスは、上記の動作がすべて正しく実装されていることを確認する必要がある。`Async[Throwing]Stream`の現在の動作を見て、これらの動作を実現しているか、またどのように実現しているかを見てみよう。

#### バックプレッシャー

ルート非同期シーケンスは、バックプレッシャーをそのシーケンスを生成するシステムに中継する必要がある。`Async[Throwing]Stream`は、設定可能なバッファを提供し、`yield()`メソッドから現在のバッファの深さを含む`Async[Throwing]Stream.Continuation.YieldResult`を返すことで、バックプレッシャーをサポートすることを目指している。しかし、`yield()`で現在のバッファの深さを提供するだけでは、バックプレッシャーシステムを非同期シーケンスにブリッジするには不十分である。現在のAPIで実装可能な唯一のバックプレッシャー戦略は、ある期間生産を停止し、その後投機的に再び生産を行うタイムドバックオフである。これは非常に非効率的なパターンであり、高いレイテンシーと非効率的なリソースの使用をもたらす。

#### マルチ／シングル コンシューマのサポート

`AsyncSequence`プロトコル自体は、実装が複数のコンシューマをサポートしているかどうかについては想定していない。これにより、ユニキャストおよびマルチキャスト非同期シーケンスの作成が可能になる。ユニキャスト非同期シーケンスとマルチキャスト非同期シーケンスの違いは、複数のイテレータを作成できるかどうかである。`AsyncStream`は複数のイテレータの作成をサポートしており、複数のコンシューマを正しく処理する。一方、`AsyncThrowingStream`も複数のイテレータをサポートしていますが、複数のイテレータがsuspendした場合に`fatalError`が発生する。元の提案にはこうある:

> 任意のシーケンスと同様に、AsyncStreamを複数回反復したり、複数のイテレータを作成して別々に反復したりすると、予期しない一連の値が生成される可能性がある。

この文はどのような動作にも対応できる余地を残しているが、私たちは、ルート非同期シーケンスの動作を明確に区別することが有益であることを学んだ。

#### ダウンストリーム コンシューマの終了

ダウンストリームのコンシューマの終了により、プロデューサはコンシューマにこれ以上値が生成されないことを通知できる。`Async[Throwing]Stream`は、`Async[Throwing]Stream.Continuation`の`finish()`または`finish(throwing:)`メソッドを呼び出すことで、これをサポートしている。しかし、`Async[Throwing]Stream`は、`finish`メソッドのいずれかが呼び出される前に`Continuation`が終了する可能性がある場合を処理しない。これは現在、終了しない非同期ストリームにつながる。動作を変更することは可能だが、意味的に壊れたコードになる可能性がある。

#### アップストリーム プロデューサの終了

アップストリームのプロデューサの終了は、ダウンストリームのコンシューマの終了の逆で、コンシューマが終了するとプロデューサに通知される。現在、`Async[Throwing]Stream` は `Continuation` の `onTermination` プロパティを公開している。`onTermination` クロージャは、コンシューマが終了すると呼び出される。コンシューマは、4 つのケースに分けて終了することができる:

1. 非同期シーケンスが終了し、イテレータが作成されなくなった
2. イテレータが`deinit`されたかつ、非同期シーケンスがユニキャストである
3. 消費タスクがキャンセルされた
4. 非同期シーケンスが`nil`かエラーを返した

しかし、`Async[Throwing]Stream`は複数のコンシューマをサポートしているため(マルチ/シングル・コンシューマのサポートのセクションで説明)、1つのコンシューマのタスクがキャンセルされると、すべてのコンシューマが終了する。これは一般的にマルチキャスト非同期シーケンスでは期待されない。

### 提案

上記の動機は、ルート非同期シーケンスから期待される振る舞いを整理し、それらを`Async[Throwing]Stream`の振る舞いと比較したものである。これらは、`Async[Throwing]Stream`が期待から乖離している動作である。

- バックプレッシャー: プロデューサへの「再開(resumption)」シグナルを公開していない
- マルチ/シングル・コンシューマ:
  - throwingとnon-throwing型で実装が異なる
  - 提案ではユニキャスト非同期シーケンスと位置づけられているが、複数のコンシューマをサポートしている
- コンシューマの終了: `deinit`する際の`Continuation`を処理しない
- プロデューサの終了: 最初のコンシューマの終了時に起こってしまう

このセクションでは、上記のすべての動作を実装する`Async[Throwing]Stream`の新しいAPIを提案する

#### バックプレッシャーをサポートしたAsyncStreamの作成

新しい `makeStream(of: backpressureStrategy:)`メソッドを使用して、`Async[Throwing]Stream`インスタンスを作成できるようにする。このメソッドは、ストリームとソースを返する。ソースは、非同期ストリームに新しい値を書き込むために使用できる。新しいAPIは特に、マルチプロデューサ/シングルコンシューマパターンを提供する。

```swift
let (stream, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)
```

新しく提案されたAPIは、バックプレッシャーシステムをブリッジする3つの異なる方法を提供する。基本となるのは、マルチステップの同期インターフェイスである。以下はその使用例である:

```swift
do {
    let writeResult = try source.write(contentsOf: sequence)
    
    switch writeResult {
    case .produceMore:
       // さらに生成する
    case .enqueueCallback(let callbackToken):
        source.enqueueCallback(token: callbackToken, onProduceMore: { result in
            switch result {
            case .success:
               // さらに生成する
            case .failure(let error):
                // 基礎となるプロデューサを終了させる
            }
        })
    }
} catch {
    // 非同期ストリームがすでに終了している場合、`write(contentsOf:)`をスローする
}
```

上記のAPIは、同期プロデューサと非同期シーケンスの橋渡しをするときに、最もコントロールしやすく、最も高いパフォーマンスを提供する。まず、`WriteResult`を返す`write(contentsOf:)`を使って値を書き込む必要がある。この結果は、`enqueueCallback(callbackToken: onProduceMore:)`メソッドを呼び出すことで、より多くの値を生成する必要があること、またはコールバックをキューに入れる必要があることを示す。このコールバックは、バックプレッシャー戦略がより多くの値を生成する必要があると判断した時点で呼び出される。このAPIは、最大のパフォーマンスで最大の柔軟性を提供することを目的としている。コールバックは、プロデューサをsuspendする必要がある場合にのみ割り当てる必要がある。

さらに、上記のAPIは、非同期ストリームに値を書き込むための、より高レベルで使いやすいAPIのための構築ブロックである。以下は2つの上位APIの例である。

```swift
// 新しい値を書き込み、コールバックを提供する。
try source.write(contentsOf: sequence, onProduceMore: { result in
    switch result {
    case .success:
        // さらに生成する
    case .failure(let error):
        // 基礎となるプロデューサを終了させる
    }
})

// このメソッドは、より多くの値が生成されるまで一時停止する。
try await source.write(contentsOf: sequence)
```

上記のAPIを使えば、コールバックベース、ブロッキング、非同期のどのシステムであっても、非同期ストリームに効果的にブリッジすることができるはずだ。

#### ダウンストリーム コンシューマの終了

> 終了動作に関する次の2つの例を読むときには、新しく提案されたAPIが厳密なユニキャスト非同期シーケンスを提供していることに留意してほしい。

`finish()`を呼び出すと、下流のコンシューマが終了する。以下はその例:

```swift

// `finish()`を呼び出して終了させる
let (stream, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)

_ = try await source.write(1)
source.finish()

for await element in stream {
    print(element)
}
print("Finished")

// Prints
// 1
// Finished
```

コンシューマを終了するもう1つの方法は、ソースを終了することである。これは`finish()`を呼び出すのと同じ効果があり、どのコンシューマも無期限にスタックしないようにする。

```swift
// ソースのdeinitで終了させる
let (stream, _) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)

for await element in stream {
    print(element)
}
print("Finished")

// Prints
// Finished
```

ソースが終了した後にさらに要素を書き込もうとすると、書き込みメソッドからエラーがスローされる。

#### アップストリーム プロデューサの終了

プロデューサは `onTerminate` コールバックを通して終了の通知を受ける。プロデューサの終了は以下のシナリオで起こる:

```swift
// タスクのキャンセルで終了させる
let (stream, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)

let task = Task {
    for await element in stream {

    }
}
task.cancel()
```

```swift
// シーケンスのdeinitで終了させる
let (_, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)
```

```swift
// イテレータのdeinitで終了させる
let (stream, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)
_ = stream.makeAsyncIterator()
```

```swift
// finishを呼んで終了させる
let (stream, source) = AsyncStream.makeStream(
    of: Int.self,
    backpressureStrategy: .watermark(low: 2, high: 4)
)

_ = try source.write(1)
source.finish()

for await element in stream {}

// 全ての要素が消費された後にonTerminateが呼ばれる
```

下流のコンシューマの終了と同様に、プロデューサの終了後にさらに要素を書き込もうとすると、`write`メソッドからエラーがスローされる。

### 詳細

AsyncStreamとAsyncThrowingStreamの新しいAPIは以下の通りである:

```swift
/// Error that is thrown from the various `write` methods of the
/// ``AsyncStream.Source`` and ``AsyncThrowingStream.Source``.
/// 
/// This error is thrown when the asynchronous stream is already finished when
/// trying to write new elements.
public struct AsyncStreamAlreadyFinishedError: Error {}

extension AsyncStream {
    /// A mechanism to interface between producer code and an asynchronous stream.
    ///
    /// Use this source to provide elements to the stream by calling one of the `write` methods, then terminate the stream normally
    /// by calling the `finish()` method.
    public struct Source: Sendable {
        /// A strategy that handles the backpressure of the asynchronous stream.
        public struct BackpressureStrategy: Sendable {
            /// When the high watermark is reached producers will be suspended. All producers will be resumed again once
            /// the low watermark is reached.
            public static func watermark(low: Int, high: Int) -> BackpressureStrategy {}
        }

        /// A type that indicates the result of writing elements to the source.
        @frozen
        public enum WriteResult: Sendable {
            /// A token that is returned when the asynchronous stream's backpressure strategy indicated that production should
            /// be suspended. Use this token to enqueue a callback by  calling the ``enqueueCallback(_:)`` method.
            public struct CallbackToken: Sendable {}

            /// Indicates that more elements should be produced and written to the source.
            case produceMore

            /// Indicates that a callback should be enqueued.
            ///
            /// The associated token should be passed to the ``enqueueCallback(_:)`` method.
            case enqueueCallback(CallbackToken)
        }

        /// A callback to invoke when the stream finished.
        ///
        /// The stream finishes and calls this closure in the following cases:
        /// - No iterator was created and the sequence was deinited
        /// - An iterator was created and deinited
        /// - After ``finish(throwing:)`` was called and all elements have been consumed
        /// - The consuming task got cancelled
        public var onTermination: (@Sendable () -> Void)?

        /// Writes new elements to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// - Parameter sequence: The elements to write to the asynchronous stream.
        /// - Returns: The result that indicates if more elements should be produced at this time.
        public func write<S>(contentsOf sequence: S) throws -> WriteResult where Element == S.Element, S: Sequence {}

        /// Write the element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// - Parameter element: The element to write to the asynchronous stream.
        /// - Returns: The result that indicates if more elements should be produced at this time.
        public func write(_ element: Element) throws -> WriteResult {}

        /// Enqueues a callback that will be invoked once more elements should be produced.
        ///
        /// Call this method after ``write(contentsOf:)`` or ``write(_:)`` returned ``WriteResult/enqueueCallback(_:)``.
        ///
        /// - Important: Enqueueing the same token multiple times is not allowed.
        ///
        /// - Parameters:
        ///   - token: The callback token.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced.
        public func enqueueCallback(token: WriteResult.CallbackToken, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) {}

        /// Cancel an enqueued callback.
        ///
        /// Call this method to cancel a callback enqueued by the ``enqueueCallback(callbackToken:onProduceMore:)`` method.
        ///
        /// - Note: This methods supports being called before ``enqueueCallback(callbackToken:onProduceMore:)`` is called and
        /// will mark the passed `token` as cancelled.
        ///
        /// - Parameter token: The callback token.
        public func cancelCallback(token: WriteResult.CallbackToken) {}

        /// Write new elements to the asynchronous stream and provide a callback which will be invoked once more elements should be produced.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then `onProduceMore` will be invoked with
        /// a `Result.failure`.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced. This callback might be
        ///   invoked during the call to ``write(contentsOf:onProduceMore:)``.
        public func write<S>(contentsOf sequence: S, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) where Element == S.Element, S: Sequence {}

        /// Writes the element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then `onProduceMore` will be invoked with
        /// a `Result.failure`.
        ///
        /// - Parameters:
        ///   - sequence: The element to write to the asynchronous stream.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced. This callback might be
        ///   invoked during the call to ``write(_:onProduceMore:)``.
        public func write(_ element: Element, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) {}

        /// Write new elements to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// This method returns once more elements should be produced.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        public func write<S>(contentsOf sequence: S) async throws where Element == S.Element, S: Sequence {}

        /// Write new element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// This method returns once more elements should be produced.
        ///
        /// - Parameters:
        ///   - sequence: The element to write to the asynchronous stream.
        public func write(_ element: Element) async throws {}

        /// Write the elements of the asynchronous sequence to the asynchronous stream.
        ///
        /// This method returns once the provided asynchronous sequence or the  the asynchronous stream finished.
        ///
        /// - Important: This method does not finish the source if consuming the upstream sequence terminated.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        public func write<S>(contentsOf sequence: S) async throws where Element == S.Element, S: AsyncSequence {}

        /// Indicates that the production terminated.
        ///
        /// After all buffered elements are consumed the next iteration point will return `nil`.
        ///
        /// Calling this function more than once has no effect. After calling finish, the stream enters a terminal state and doesn't accept
        /// new elements.
        public func finish() {}
    }

    /// Initializes a new ``AsyncStream`` and an ``AsyncStream/Source``.
    ///
    /// - Parameters:
    ///   - elementType: The element type of the stream.
    ///   - backpressureStrategy: The backpressure strategy that the stream should use.
    /// - Returns: A tuple containing the stream and its source. The source should be passed to the
    ///   producer while the stream should be passed to the consumer.
    public static func makeStream(
        of elementType: Element.Type = Element.self,
        backpressureStrategy: Source.BackpressureStrategy
    ) -> (`Self`, Source) {}
}

extension AsyncThrowingStream {
    /// A mechanism to interface between producer code and an asynchronous stream.
    ///
    /// Use this source to provide elements to the stream by calling one of the `write` methods, then terminate the stream normally
    /// by calling the `finish()` method. You can also use the source's `finish(throwing:)` method to terminate the stream by
    /// throwing an error
    public struct Source: Sendable {
        /// A strategy that handles the backpressure of the asynchronous stream.
        public struct BackpressureStrategy: Sendable {
            /// When the high watermark is reached producers will be suspended. All producers will be resumed again once
            /// the low watermark is reached.
            public static func watermark(low: Int, high: Int) -> BackpressureStrategy {}
        }

        /// A type that indicates the result of writing elements to the source.
        @frozen
        public enum WriteResult: Sendable {
            /// A token that is returned when the asynchronous stream's backpressure strategy indicated that production should
            /// be suspended. Use this token to enqueue a callback by  calling the ``enqueueCallback(_:)`` method.
            public struct CallbackToken: Sendable {}

            /// Indicates that more elements should be produced and written to the source.
            case produceMore

            /// Indicates that a callback should be enqueued.
            ///
            /// The associated token should be passed to the ``enqueueCallback(_:)`` method.
            case enqueueCallback(CallbackToken)
        }

        /// A callback to invoke when the stream finished.
        ///
        /// The stream finishes and calls this closure in the following cases:
        /// - No iterator was created and the sequence was deinited
        /// - An iterator was created and deinited
        /// - After ``finish(throwing:)`` was called and all elements have been consumed
        /// - The consuming task got cancelled
        public var onTermination: (@Sendable () -> Void)? {}

        /// Writes new elements to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// - Parameter sequence: The elements to write to the asynchronous stream.
        /// - Returns: The result that indicates if more elements should be produced at this time.
        public func write<S>(contentsOf sequence: S) throws -> WriteResult where Element == S.Element, S: Sequence {}

        /// Write the element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// - Parameter element: The element to write to the asynchronous stream.
        /// - Returns: The result that indicates if more elements should be produced at this time.
        public func write(_ element: Element) throws -> WriteResult {}

        /// Enqueues a callback that will be invoked once more elements should be produced.
        ///
        /// Call this method after ``write(contentsOf:)`` or ``write(_:)`` returned ``WriteResult/enqueueCallback(_:)``.
        ///
        /// - Important: Enqueueing the same token multiple times is not allowed.
        ///
        /// - Parameters:
        ///   - token: The callback token.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced.
        public func enqueueCallback(token: WriteResult.CallbackToken, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) {}

        /// Cancel an enqueued callback.
        ///
        /// Call this method to cancel a callback enqueued by the ``enqueueCallback(callbackToken:onProduceMore:)`` method.
        ///
        /// - Note: This methods supports being called before ``enqueueCallback(callbackToken:onProduceMore:)`` is called and
        /// will mark the passed `token` as cancelled.
        ///
        /// - Parameter token: The callback token.
        public func cancelCallback(token: WriteResult.CallbackToken) {}

        /// Write new elements to the asynchronous stream and provide a callback which will be invoked once more elements should be produced.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then `onProduceMore` will be invoked with
        /// a `Result.failure`.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced. This callback might be
        ///   invoked during the call to ``write(contentsOf:onProduceMore:)``.
        public func write<S>(contentsOf sequence: S, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) where Element == S.Element, S: Sequence {}

        /// Writes the element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then `onProduceMore` will be invoked with
        /// a `Result.failure`.
        ///
        /// - Parameters:
        ///   - sequence: The element to write to the asynchronous stream.
        ///   - onProduceMore: The callback which gets invoked once more elements should be produced. This callback might be
        ///   invoked during the call to ``write(_:onProduceMore:)``.
        public func write(_ element: Element, onProduceMore: @escaping @Sendable (Result<Void, Error>) -> Void) {}

        /// Write new elements to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// first element of the provided sequence. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// This method returns once more elements should be produced.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        public func write<S>(contentsOf sequence: S) async throws where Element == S.Element, S: Sequence {}

        /// Write new element to the asynchronous stream.
        ///
        /// If there is a task consuming the stream and awaiting the next element then the task will get resumed with the
        /// provided element. If the asynchronous stream already terminated then this method will throw an error
        /// indicating the failure.
        ///
        /// This method returns once more elements should be produced.
        ///
        /// - Parameters:
        ///   - sequence: The element to write to the asynchronous stream.
        public func write(_ element: Element) async throws {}

        /// Write the elements of the asynchronous sequence to the asynchronous stream.
        ///
        /// This method returns once the provided asynchronous sequence or the  the asynchronous stream finished.
        ///
        /// - Important: This method does not finish the source if consuming the upstream sequence terminated.
        ///
        /// - Parameters:
        ///   - sequence: The elements to write to the asynchronous stream.
        public func write<S>(contentsOf sequence: S) async throws where Element == S.Element, S: AsyncSequence {}

        /// Indicates that the production terminated.
        ///
        /// After all buffered elements are consumed the next iteration point will return `nil` or throw an error.
        ///
        /// Calling this function more than once has no effect. After calling finish, the stream enters a terminal state and doesn't accept
        /// new elements.
        ///
        /// - Parameters:
        ///   - error: The error to throw, or `nil`, to finish normally.
        public func finish(throwing error: Failure?) {}
    }

    /// Initializes a new ``AsyncThrowingStream`` and an ``AsyncThrowingStream/Source``.
    ///
    /// - Parameters:
    ///   - elementType: The element type of the stream.
    ///   - failureType: The failure type of the stream.
    ///   - backpressureStrategy: The backpressure strategy that the stream should use.
    /// - Returns: A tuple containing the stream and its source. The source should be passed to the
    ///   producer while the stream should be passed to the consumer.
    public static func makeStream(
        of elementType: Element.Type = Element.self,
        throwing failureType: Failure.Type = Failure.self,
        backpressureStrategy: Source.BackpressureStrategy
    ) -> (`Self`, Source) where Failure == Error {}
}
```

### 他のルート非同期シーケンスとの比較

#### swift-async-algorithm: AsyncChannel

`AsyncChannel`はマルチコンシューマ/マルチプロデューサのルート非同期シーケンスで、2 つのタスク間の通信に使用できる。非同期のプロダクションAPIのみを提供し、内部バッファを持たない。つまり、どのプロデューサもその値が消費されるまで中断される。`AsyncChannel`は複数のコンシューマを扱うことができ、FIFO順で再開する。

#### swift-nio: NIOAsyncSequenceProducer

NIOチームは、NIOチャネルのインバウンドストリームをConcurrencyにブリッジするために使用できる高性能シーケンスを提供することを目標に、独自のルート非同期シーケンスを作成した。`NIOAsyncSequenceProducer`は非常に汎用的で完全にインライン化可能な型であり、非常に使いやすい。この提案は、この型からの学習に大きく触発されているが、標準ライブラリに適合する、より柔軟で使いやすいAPIを作成しようとしている。

## ソース互換性

影響なし

## ABIの互換性

影響なし

## 将来の検討事項

### 順応型バックプレッシャー戦略

high/low watermark戦略はネットワーキングコードでは一般的だが、将来的には順応型戦略のような他の戦略も提供できるだろう。順応型戦略は、消費と生成の速度に基づいてバックプレッシャーを調整する。提案された新しいAPIを使えば、さらなる戦略を簡単に追加することができる。

### 要素サイズ依存戦略

ストリームの要素がコレクション型である場合、各要素は実際のメモリサイズが異なる可能性があるため、提案されたhigh/low watermarkバックプレッシャー戦略は予期しない結果につながる可能性がある。将来的には、コレクションのサイズの検査をサポートする新しいバックプレッシャー戦略を提供できるだろう。

### `Async[Throwing]Stream.Continuation`の非推奨化

将来的には、`WriteResult`を破棄するだけで、新しい提案のAPIは非バックプレッシャープロデューサをブリッジすることも可能であるため、現在のContinuationベースのAPIを非推奨にすることもできる。新しいAPIがカバーしない唯一のユースケースは、2つのイテレータが同時にストリームを消費しない限り、ストリームに対して複数のイテレータを作成できる、現在の`AsyncStream`の*anycast*動作である。これは、`swift-async-algorithms`パッケージの`broadcast`などの追加アルゴリズムで解決できる。

開発者に新しいAPIを採用する時間を与えるために、現在のAPIの非推奨化は将来のバージョンに延期すべきである。特に、これらの新しいAPIは現在のConcurrencyランタイムのようにバックデプロイされないため。

### WriterとAsyncWriterプロトコルの導入

新しく導入された`Source`型は、さまざまな書き込みメソッドを提供する。ファイルの抽象化やネットワーキングAPIなど、他の場所でも似たような型が使われているのを見たことがある。将来的には、新しい`Writer`と`AsyncWriter`プロトコルを導入して、`Writer`の上に汎用的なアルゴリズムを記述できるようにするかもしれない。`Source`型は、これらの新しいプロトコルに準拠することができる。

## 代替案

WIP

## 参考リンク

### Forums

- [[Pitch] New APIs for Async[Throwing]Stream with backpressure support](https://forums.swift.org/t/pitch-new-apis-for-async-throwing-stream-with-backpressure-support/65449)
- [SE-0406: Backpressure support for AsyncStream](https://forums.swift.org/t/se-0406-backpressure-support-for-asyncstream/66771)

### プロポーザルドキュメント

- [Backpressure support for AsyncStream](https://github.com/apple/swift-evolution/blob/main/proposals/0406-async-stream-backpressure.md)
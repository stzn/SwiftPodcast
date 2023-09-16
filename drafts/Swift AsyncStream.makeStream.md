# Swift AsyncStream.makeStream

- [Swift AsyncStream.makeStream](#swift-asyncstreammakestream)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
    - [提案内容](#提案内容)
    - [詳細](#詳細)
    - [ソース互換性](#ソース互換性)
    - [ABI互換性](#abi互換性)
    - [APIレジリエンスへの影響](#apiレジリエンスへの影響)
    - [検討された代替案](#検討された代替案)
      - [タプルの代わりに具象型を返す](#タプルの代わりに具象型を返す)
      - [`AsyncStream<Element>.init()`にContinuationを渡す](#asyncstreamelementinitにcontinuationを渡す)
      - [何もしない代替案](#何もしない代替案)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

`AsyncStream`と`AsyncThrowingStream`インスタンスを生成するためのヘルパーメソッドを導入することで、Streamのcontinuationにアクセスしやすくする。

## 内容

### 動機

[SE-0314]()で、標準ライブラリが提供する`AsyncSequence`のルートとして動作する`AsyncStream`と`AsyncThrowingStream`を導入した。

`Async[Throwing]Stream`をしばらく使っていると、一般的な使い方は、`Continuation`と`Async[Throwing]Stream`を別の場所に渡すことであることがわかった。この場合、イニシャライザに渡されるクロージャから`Async[Throwing]Stream.Continuation`をエスケープする必要がある。`Continuation`をエスケープすることは、暗黙的にアンラップされたオプショナルを回避する必要があるため、少し不便である。さらに、クロージャは`Continuation`のライフタイムがクロージャにスコープされていることを意味するが、そういう風に使われていない。下記が現在のAsyncStream APIの使用例である。

```swift
var cont: AsyncStream<Int>.Continuation!
let stream = AsyncStream<Int> { cont = $0 }
// Sendableのワーニングを避けるためにletに`Continuation`を割り当てる
let continuation = cont

await withTaskGroup(of: Void.self) { group in
  group.addTask {
    for i in 0...9 {
      continuation.yield(i)
    }
    continuation.finish()
  }

  group.addTask {
    for await i in stream {
      print(i)
    }
  }
}
```

### 提案内容

このギャップを埋めるために、`AsyncStream`と`AsyncThrowingStream`に、Streamと`Continuation`の両方を返す新しい静的メソッド`makeStream`を追加することを提案する。新しく提案する便利なメソッドの使用例は以下のようになる:

```swift

let (stream, continuation) = AsyncStream.makeStream(of: Int.self)

await withTaskGroup(of: Void.self) { group in
  group.addTask {
    for i in 0...9 {
      continuation.yield(i)
    }
    continuation.finish()
  }

  group.addTask {
    for await i in stream {
      print(i)
    }
  }
}
```

### 詳細

`AsyncStream`と`AsyncThrowingStream`にそれぞれ以下のコードを追加することを提案する。これらのメソッドは、以前のSwiftバージョンへのバックデプロイできるようにマークされている。

```swift
@available(SwiftStdlib 5.1, *)
extension AsyncStream {
  /// 新しい`AsyncStream`と`AsyncStream.Continuation`を初期化する。
  ///
  /// - Parameters:
  ///   - elementType: Streamの要素の型。
  ///   - limit: Streamが使用するべきバッファリングを制限するポリシー。
  /// - Returns: StreamとContinuationを含むタプル。
  ///            Streamは消費者(Consumer)に渡され、Continuationは生産者(Producer)に渡されるべき。
  @backDeployed(before: SwiftStdlib 5.9)
  public static func makeStream(
      of elementType: Element.Type = Element.self,
      bufferingPolicy limit: Continuation.BufferingPolicy = .unbounded
  ) -> (stream: AsyncStream<Element>, continuation: AsyncStream<Element>.Continuation) {
    var continuation: AsyncStream<Element>.Continuation!
    let stream = AsyncStream<Element>(bufferingPolicy: limit) { continuation = $0 }
    return (stream: stream, continuation: continuation!)
  }
}

@available(SwiftStdlib 5.1, *)
extension AsyncThrowingStream {
  /// 新しい`AsyncThrowingStream`と`AsyncThrowingStream.Continuation`を初期化する。
  ///
  /// - Parameters:
  ///   - elementType: Streamの要素の型。
  ///   - failureType: Streamの失敗型。
  ///   - limit: Streamが使用するべきバッファリングを制限するポリシー。
  /// - Returns: StreamとContinuationを含むタプル。
  ///            Streamは消費者(Consumer)に渡され、Continuationは生産者(Producer)に渡されるべき。
  @backDeployed(before: SwiftStdlib 5.9)
  public static func makeStream(
      of elementType: Element.Type = Element.self,
      throwing failureType: Failure.Type = Failure.self,
      bufferingPolicy limit: Continuation.BufferingPolicy = .unbounded
  ) -> (stream: AsyncThrowingStream<Element, Failure>, continuation: AsyncThrowingStream<Element, Failure>.Continuation) where Failure == Error {
    var continuation: AsyncThrowingStream<Element, Failure>.Continuation!
    let stream = AsyncThrowingStream<Element, Failure>(bufferingPolicy: limit) { continuation = $0 }
    return (stream: stream, continuation: continuation!)
  }
}
```

### ソース互換性

追加なので影響なし。

### ABI互換性

影響なし。

### APIレジリエンスへの影響

なし。

### 検討された代替案

#### タプルの代わりに具象型を返す

当初はファクトリの結果型としてタプルを使うつもりだったが、具象型に関するよりよいドキュメントを提供できると思ったので、レビューの前に撤回した。しかし、レビューの間、フィードバックの大半はタプルをベースにしたアプローチに傾いた。2つのアプローチを比較した結果、私はレビューのフィードバックに同意した。タプルベースのアプローチには2つの大きな利点がある:

1. ContinuationとStreamストリームはそれぞれ生産者と消費者によって保持されるべきなので、返された型付けを再構築するようにユーザをうながすことができる。
2. これにより、メソッドをバックデプロイすることができる。

#### `AsyncStream<Element>.init()`にContinuationを渡す

ピッチの中で、ユーザが `AsyncStream<Element>.init()`にContinuationを渡せるようにすることが提案された:

1. 継続は複数のストリームに渡すことができる
2. ストリームに渡されない継続は役に立たない。

結局のところ、`AsyncStream.Continuation`は`AsyncStream`の1つのインスタンスと深く結合しているため、この結合を伝え、ユーザが誤用するのを防ぐAPIを作成する必要がある。

#### 何もしない代替案

`Async[Throwing]Stream`を現状のままにしておくこともできる。しかし、これは標準ライブラリの一部であるため、StreamとそのContinuationを作成するためのよりよいメソッドを提供すべきである。


## 参考リンク

### Forums

- [pitch](https://forums.swift.org/t/pitch-convenience-async-throwing-stream-makestream-methods/61030)
- [review](https://forums.swift.org/t/se-0388-convenience-async-throwing-stream-makestream-methods/63139)
- [acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0388-convenience-async-throwing-stream-makestream-methods/63568)

### プロポーザルドキュメント

- [Convenience Async[Throwing]Stream.makeStream methods](https://github.com/apple/swift-evolution/blob/main/proposals/0388-async-stream-factory.md)
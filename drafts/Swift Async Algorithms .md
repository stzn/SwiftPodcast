# Swift Async Algorithms

- [Swift Async Algorithms](#swift-async-algorithms)
  - [概要](#概要)
  - [内容](#内容)
  - [非同期シーケンスの結合](#非同期シーケンスの結合)
    - [Chain](#chain)
    - [Combine Latest](#combine-latest)
    - [Merge](#merge)
    - [Zip](#zip)
    - [Joined](#joined)
  - [非同期シーケンスの生成](#非同期シーケンスの生成)
    - [AsyncLazySequence](#asynclazysequence)
    - [Channel](#channel)
  - [非同期イテレーションのパフォーマンスの最適化](#非同期イテレーションのパフォーマンスの最適化)
    - [AsyncBufferedByteIterator](#asyncbufferedbyteiterator)
  - [他の役に立つ非同期シーケンス](#他の役に立つ非同期シーケンス)
    - [AdjacentPairs](#adjacentpairs)
    - [Chunked](#chunked)
      - [グループ化](#グループ化)
      - [投影](#投影)
        - [カウントまたはシグナル](#カウントまたはシグナル)
        - [カウントのみ](#カウントのみ)
        - [シグナルのみ](#シグナルのみ)
        - [カウントまたはシグナル](#カウントまたはシグナル-1)
    - [Compacted](#compacted)
    - [RemoveDuplicates](#removeduplicates)
    - [Intersperse](#intersperse)
  - [時間内に処理する非同期シーケンス](#時間内に処理する非同期シーケンス)
    - [Debounce](#debounce)
    - [Throttle](#throttle)
    - [Timer](#timer)
  - [非同期シーケンスからすべての値を取得](#非同期シーケンスからすべての値を取得)
    - [Collection Initializers](#collection-initializers)
      - [RangeReplaceableCollection](#rangereplaceablecollection)
      - [Dictionary](#dictionary)
      - [SetAlgebra](#setalgebra)
  - [Taskの管理](#taskの管理)
    - [Task.select](#taskselect)
  - [効果(Effects)](#効果effects)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)

## 概要

Swiftの安全でシンプルでパフォーマンスの高い非同期プログラミングへの取り組みの一環として、Swift5.5で導入された`AsyncSequence`を基にした時間の経過と共に値を処理するアルゴリズムを提供するオープンソースのパッケージが登場した。

## 内容

## 非同期シーケンスの結合

### Chain

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Chain.md

2つ以上の非同期シーケンスを順番につなげる。結果の非同期シーケンスの要素は、最初の非同期シーケンスの要素から順番に、次に2番目、またはエラーが発生するまで順番に構成される。

Elementの型が同じ任意のAsyncSequenceで使える。

```swift
let preamble = [
    "// Some header to add as a preamble",
    "//",
    ""
].async
let lines = chain(preamble, URL(fileURLWithPath: "/tmp/Sample.swift").lines)

for try await line in lines {
    print(line)
}
```

上記の例は、2つの`AsyncSequence`を連鎖する方法を示している。この場合、ファイルの行の内容の`lines`の前に`preamble`を追加している。

`chain`関数は、引数として2つ以上のシーケンスを取る。

結果の`AsyncChainSequence`は非同期シーケンスであり、引数もそれに準拠している場合は`Sendable`に条件付きで準拠(conditional conformance)している。

連鎖する非同期シーケンスのいずれかがイテレーションの終わりに達すると、次の非同期シーケンスに進む。最後の非同期シーケンスがイテレーションの終わりに達すると、`AsyncChainSequence`は終了する。

任意の時点で、構成するベースの非同期シーケンスの1つがイテレーション中にエラーがスローされた場合、結果の`AsyncChainSequence`のイテレーションからそのエラーがスローされ終了する。`AsyncChainSequence`のスロー動作は、構成するいずれかのベースからスローされたときにスローされ、構成するすべてのベースからスローされない場合は、エラーはスローされない。

### Combine Latest

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/CombineLatest.md

2つ以上の非同期シーケンスから生成された最新の値をタプルの非同期シーケンスに合成する。

```swift
let appleFeed = URL("http://www.example.com/ticker?symbol=AAPL").lines
let nasdaqFeed = URL("http://www.example.com/ticker?symbol=^IXIC").lines

for try await (apple, nasdaq) in combineLatest(appleFeed, nasdaqFeed) {
    print("AAPL: \(apple) NASDAQ: \(nasdaq)")
}
```

いくつかのサンプルを入力した結果

| Timestamp   | appleFeed | nasdaqFeed | combined output               |                 
| ----------- | --------- | ---------- | ----------------------------- |
| 11:40 AM    | 173.91    |            |                               |
| 12:25 AM    |           | 14236.78   | AAPL: 173.91 NASDAQ: 14236.78 |
| 12:40 AM    |           | 14218.34   | AAPL: 173.91 NASDAQ: 14218.34 |
|  1:15 PM    | 173.00    |            | AAPL: 173.00 NASDAQ: 14218.34 |


`combineLatest`関数は、引数として2つ以上の非同期シーケンスを取り、非同期シーケンスである`AsyncCombineLatestSequence`を生成する。

`AsyncCombineLatestSequence`を構成するベースの非同期シーケンスは、最新の値を生成するために同時並行にイテレートする必要があるため、これらのシーケンスは子タスクに送信できる必要がある。つまり、ベースの前提条件は、ベースの非同期シーケンス、それらのイテレータ、およびそれらが生成する要素がすべて`Sendable`であるあることを意味する。

最初の要素が生成される前にいずれかのベースが終了した場合、`AsyncCombineLatestSequence`のイテレーションは決して満たされない。したがって、ベースのイテレータが最初のイテレーションで`nil`を返す場合、`AsyncCombineLatestSequence`イテレータはすぐに`nil`を返し、終了したことを示す。この特定のケースでは、他のベースの未処理のイテレーションはキャンセルされる。最初の要素が生成された後の場合、少なくとも1つのベースで最新の値を満たすことができるため、この動作は異なる。これは、ベースから返された要素で構成される最初のタプルの構築を超えて、すべてのベースのイテレーションが終了に達したときにのみ、`AsyncCombineLatestSequence`のイテレーションは終了に到達することを意味する。

`AsyncCombineLatestSequence`のスロー動作は、いずれかのベースからスローされた場合に、合成した非同期シーケンスからそのエラーがスローされる。いずれかの時点(最初のイテレーション内またはそれ以降)で、任意のベースからエラーがスローされた場合、他のイテレーションはキャンセルされ、スローされたエラーはすぐに呼び出し側のイテレーションにスローされる。

### Merge

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Merge.md

同じ`Element`型を共有する2つ以上の非同期シーケンスを1つの単一の非同期シーケンスに結合する。

```swift
let appleFeed = URL(string: "http://www.example.com/ticker?symbol=AAPL")!.lines.map { "AAPL: " + $0 }
let nasdaqFeed = URL(string:"http://www.example.com/ticker?symbol=^IXIC")!.lines.map { "^IXIC: " + $0 }

for try await ticker in merge(appleFeed, nasdaqFeed) {
    print(ticker)
}
```

いくつかのサンプルを入力した結果

| Timestamp   | appleFeed | nasdaqFeed | merged output   |                 
| ----------- | --------- | ---------- | --------------- |
| 11:40 AM    | 173.91    |            | AAPL: 173.91    |
| 12:25 AM    |           | 14236.78   | ^IXIC: 14236.78 |
| 12:40 AM    |           | 14218.34   | ^IXIC: 14218.34 |
|  1:15 PM    | 173.00    |            | AAPL: 173.00    |


`merge`関数は、引数として2つ以上の非同期シーケンスを取り、非同期シーケンスである`AsyncMergeSequence`を生成する。

`AsyncMergeSequence`を構成するベースは、最新の値を生成するために同時並行にイテレーションする必要があるため、これらのシーケンスは子タスクに送信できる必要がある。これは、ベースの前提条件は、ベースの非同期シーケンス、それらのイテレータ、およびそれらが生成する要素が`Sendable`でなければならないことを意味する。

`AsyncMergeSequence`をイテレーションする場合、すべてのベースの非同期シーケンスが終了するとシーケンスが終了する。これは、これ以上要素が生成される可能性がないことを意味する。

`AsyncMergeSequence`のエラーのスロー動作は、ベースのいずれかからスローされた場合、合成した非同期シーケンスからそのエラーをスローされる。いずれかの時点でいずれかのベースからエラーがスローされた場合、他のイテレーションはキャンセルされ、スローされたエラーはすぐに呼び出し側のイテレーションにスローされる。

### Zip

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Zip.md

2つ以上の非同期シーケンスから生成された最新の値をタプルの非同期シーケンスに結合する。

```swift
let appleFeed = URL(string: "http://www.example.com/ticker?symbol=AAPL")!.lines
let nasdaqFeed = URL(string: "http://www.example.com/ticker?symbol=^IXIC")!.lines

for try await (apple, nasdaq) in zip(appleFeed, nasdaqFeed) {
    print("APPL: \(apple) NASDAQ: \(nasdaq)")
}
```

いくつかのサンプルを入力した結果

| Timestamp   | appleFeed | nasdaqFeed | combined output               |                 
| ----------- | --------- | ---------- | ----------------------------- |
| 11:40 AM    | 173.91    |            |                               |
| 12:25 AM    |           | 14236.78   | AAPL: 173.91 NASDAQ: 14236.78 |
| 12:40 AM    |           | 14218.34   |                               |
|  1:15 PM    | 173.00    |            | AAPL: 173.00 NASDAQ: 14218.34 |


`zip`関数は、引数として2つ以上の非同期シーケンスを取り、結果として非同期シーケンスである`AsyncZipSequence`を生成する。

`AsyncZipSequence`の各イテレーションは、すべてのベースのイテレータが値を生成するのを待つ。このイテレーションは、単一のタプル結果を生成するために同時並行に実行される。ベースのイテレーションのいずれかが`nil`を返すことによって終了した場合、`AsyncZipSequence`のイテレーションは即座に満たされないと見なされ、`nil`を返し、他のベースのすべてのイテレーションはキャンセルされる。ベースのいずれかのイテレーションがエラーをスローした場合、同時並行に実行されている他のイテレーションはキャンセルされ、生成されたエラーが再スローされ、イテレーションが終了する。

`AsyncZipSequence`では、イテレーションが同時並行に行われる必要がある。これは、ベースのシーケンス、それらの要素、およびイテレータがすべて`Sendable`でなければならないことを意味する。これにより、`AsyncZipSequence`は実質`Sendable`になる。

`AsyncZipSequence`のスローの元は、そのベースによって決まる。つまり、いずれかのベースがエラーをスローする可能性がある場合、`AsyncZipSequence`のイテレーションがスローする可能性がある。ベースがスローできない場合、`AsyncZipSequence`はスローしない。

### Joined

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Joined.md

`Element`型を順番に共有する非同期シーケンスの非同期シーケンスを連結する。結果の非同期シーケンスの要素は、最初の非同期シーケンスの要素から順番に、次に2番目(以下同様)に、またはエラーが発生するまで含まれる。 `chain`に似ているが、連結する非同期シーケンスの数が事前にわからない点が異なる。

オプションで、他の各シーケンスの間にセパレータの非同期シーケンスの要素を挿入できる。

```swift
let sequenceOfURLs: AsyncSequence<URL> = ...
let sequenceOfLines = sequenceOfURLs.map { $0.lines }
let joinedWithSeparator = sequenceOfLines.joined(separator: ["===================="].async)

for try await lineOrSeparator in joinedWithSeparator {
    print(lineOrSeparator)
}
```

この例は、`URL`の`AsyncSequence`を、各ファイルの間に区切り行を入れて、これらの各ファイルの行の`AsyncSequence`に順番に変換する方法を示している。

結果の`AsyncJoinedSequence`または`AsyncJoinedBySeparatorSequence`型は非同期シーケンスで、引数が準拠している場合は`Sendable`に条件付きで準拠(conditional conformance)している。

結合されている非同期シーケンスのいずれかがイテレーションの終わりに達すると、結合されたシーケンスのイテレーションは、(存在する場合)セパレータの非同期シーケンスに進む。 セパレータの非同期シーケンスが終了すると、またはセパレータが指定されていない場合は、次の非同期シーケンスに進む。最後の非同期シーケンスがイテレーションの終わりに達すると、`AsyncJoinedSequence`または`AsyncJoinedBySeparatorSequence`はそのイテレーションを終了する。構成する非同期シーケンスの1つがイテレーション中にエラーをスローした場合、`AsyncJoinedSequence`または`AsyncJoinedBySeparatorSequence`のイテレーションはそのエラーをスローし、イテレーションを終了する。

## 非同期シーケンスの生成

### AsyncLazySequence

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Lazy.md

非同期シーケンスを非同期シーケンスに変換する。この操作は、すべての`Sequence`型で使用できる。

```swift
let numbers = [1, 2, 3, 4].async
let characters = "abcde".async
```

この変換は、`AsyncSequence`で特に利用可能な操作をテストするのに役立つが、他の`AsyncSequence`型と組み合わせて、よく知られたデータソースを提供するのにも役立つ。

### Channel

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Channel.md

`AsyncStream`は、SwiftのConcurrencyを使用しないコンテキストから、使用するコンテキストにバッファリングされた要素を送信するメカニズムを導入した。その設計は、潜在的なユースケースの一部にしか対応しておらず、 2つの同時実行ドメインにわたって取り出されるback pressureが欠けていた。

back pressureをサポートし、あるタスクから別のタスクへの複数の値の通信を可能にするシステムを実現するために、新しい型であるチャネル(Channel)を導入した。チャネルは、イテレーションの消費を待機する非同期の送信機能を備えた参照型の非同期シーケンス。チャネルによって送信された各値は、イテレーションでその値の消費がされるまで待機する。その待機行動により、消費側から加えられたback pressureのアフォーダンスが生産側に伝達されるようになる。これは、生産が消費を超えず、消費が生産を超えないことを意味する。終了イベントをチャネルに送信すると、すべての生産者と消費者の保留中のすべての操作が即座に再開される。

チャネルは、特に、あるタスクが値を生成し、別のタスクがその値を消費する場合、タスク間の通信タイプとして使用することを目的としている。一方で、pause/resumeを経由して`send`および`fail`によって適用されるback pressureにより、値の生成がイテレーションからの値の消費を超えないことが保証されている。これらの各メソッドは、イベントをキューに入れた後にpauseし、イテレータで`next`が次に呼び出されたときにresumeされる。一方、`finish`を呼び出すと、すべての生産者と消費者の保留中のすべての操作がすぐに再開される。したがって、suspendされたすべての`send`操作は即座に`resume`され、`suspend`されたすべての`next`操作は、イテレーションの終了を示す`nil`を生成することによって行われる。`send`へのそれ以上の呼び出しはすぐに`resume`される。

```swift
let channel = AsyncChannel<String>()
Task {
    while let resultOfLongCalculation = doLongCalculations() {
        await channel.send(resultOfLongCalculation)
    }
    channel.finish()
}

for await calculationResult in channel {
    print(calculationResult)
}
```

上記の例では、タスクを使用して重い計算を実行している。それぞれが`send`メソッドを介して他のタスクに送信される。`send`の呼び出しは、チャネルの次のイテレーションが呼び出されたときに返される。

`AsyncChannel`と`AsyncThrowingChannel`は、[Subject](https://developer.apple.com/documentation/combine/subject/)から大きく影響を受けているが、SwiftのConcurrencyを使用してback pressureを適用するという重要な違いがある。

## 非同期イテレーションのパフォーマンスの最適化

### AsyncBufferedByteIterator

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/BufferedBytes.md

非同期の`read`関数から派生したバイトシーケンスをイテレーションするのに役立つ非常に効率的なイテレータを提供する。

この型は、ファイル記述子または同様の読み取りソースに裏打ちされた`UInt8`の要素を使用して`AsyncSequence`型を作成するためのインフラストラクチャを提供する。


```swift
struct AsyncBytes: AsyncSequence {
    public typealias Element = UInt8
    var handle: ReadableThing

    internal init(_ readable: ReadableThing) {
        handle = readable
    }

    public func makeAsyncIterator() -> AsyncBufferedByteIterator {
        return AsyncBufferedByteIterator(capacity: 16384) { buffer in
            // This runs once every 16384 invocations of next()
            return try await handle.read(into: buffer)
        }
    }
}
```

`next`を呼び出すたびに、イテレータはバッファがいっぱいになっているかどうかを確認する、バッファがある程度のバイトでいっぱいになると、そのバッファからバイトを直接返すための高速パスが使用される。バッファがいっぱいになっていない場合は、`read`関数が呼び出されて、次にいっぱいになったバッファが取得さえる。その時点で、そのバッファから1バイトが取り出される。

`read`関数が0を返した場合、それ以上バイトを読み取らないことを示し、イテレータは終了したと判断され、`read`関数への追加の呼び出しは行われない。

`read`関数がスローされた場合、エラーはイテレーションによってスローされます。その後のイテレータの呼び出しは、`read`関数を呼び出さずに`nil`を返す。

イテレーション中にタスクがキャンセルされた場合、イテレーションは、`read`関数が呼び出されたパスでのみキャンセルをチェックし、`CancellationError`をスローする。

## 他の役に立つ非同期シーケンス

### AdjacentPairs

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/AdjacentPairs.md

`adjacentPairs`APIは、隣接する値を集める目的で使用される。この操作は、隣接する`adjacentPairs`メソッドを呼び出すことにより、すべての`AsyncSequence`で使用できる。

```swift
extension AsyncSequence {
    public func adjacentPairs() -> AsyncAdjacentPairsSequence<Self>
}
```

`adjacentPairs`アルゴリズムは、元の`Element`型のペアを含む(サイズ2の)タプルの要素を生成する。

このアルゴリズムのインターフェースは、すべての`AsyncSequence`型で使用できる。返された`AsyncAdjacentPairsSequence`は、条件付きで`Sendable`に準拠している(conditional conformance)。

そのイテレータは、`next`関数で返された前の要素を追跡し、毎回更新する。

```swift
for await (first, second) in (1...5).async.adjacentPairs() {
    print("First: \(first), Second: \(second)")
}

// First: 1, Second: 2
// First: 2, Second: 3
// First: 3, Second: 4
// First: 4, Second: 5
```

タプルの`AsyncSequence`を処理する`Dictionary.init(_:uniquingKeysWith:)`APIとうまく組み合う。

```swift
Dictionary(uniqueKeysWithValues: url.lines.adjacentPairs())
```

### Chunked

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Chunked.md

非同期シーケンスからの値をグループすることは、それらの値を効率的に書き込みが含まれるタスクや特定の構造化されたデータの入力処理に役立つことがよくある。

#### グループ化

グループチャンクは、2つの連続する要素をクロージャに渡して、それらが同じグループにあるかどうかを判定することによって決定される。`AsyncChunkedByGroupSequence`イテレータがベースのシーケンスから最初の要素を受け取ると、すぐにグループに追加される。2番目のアイテムを受け取ると、前のアイテムと現在のアイテムが同じグループに属しているかどうかを判定する。それらが同じグループにない場合、イテレータは最初のアイテムのグループを発行し、2番目のアイテムを含む新しいグループが作成される。同じグループにあると宣言されたアイテムは、新しいグループが宣言されるか、イテレータがベースのシーケンスの終わりを見つけるまで蓄積される。ベースの配列が終了すると、最後のグループが出力される。ベースのシーケンスがエラーをスローした場合、`AsyncChunkedByGroupSequence`はそのエラーをすぐに再スローし、現在のグループを破棄する。

非同期シーケンスが次の値を出力する例を考えてみる: `10、20、30、10、40、40、10、20`。次のようなチャンク化する操作があるとする。

```swift
let chunks = numbers.chunked { $0 <= $1 }
for await numberChunk in chunks {
    print(numberChunk)
}
```
この結果は下記になる。

```
[10, 20, 30]
[10, 40, 40]
[10, 20]
```

`Array`はチャンクのデフォルトの型ですが、`RangeReplaceableCollection`型を使用するオーバーロードのおかげで、同じサンプルを`ContiguousArray`のインスタンス、または代わりに他の`RangeReplaceableCollection`にチャンク化できる。

```swift
let chunks = numbers.chunked(into: ContiguousArray.self) { $0 <= $1 }
for await numberChunk in chunks {
    print(numberChunk)
}
```

この変形は、`[Element].self`をパラメーターとして渡すメイン実装のファンネルメソッド。

#### 投影

一部のシナリオでは、チャンクは、異なる要素を比較することによってではなく、要素自体によって決定される。これは、要素に、それが属するチャンクを判別できるある種の識別子がある場合に当てはまる。2つの連続する要素の投影が異なる場合、現在のチャンクが出力され、新しい要素に対して新しいチャンクが作成される。

`AsyncChunkedOnProjectionSequence`のイテレータがベースのシーケンスから`nil`を受信すると、最後のチャンクを発行する。ベースのシーケンスがエラーをスローすると、イテレータは現在のチャンクを破棄し、そのエラーを再スローする。

`chunked(by:)`メソッドと同様に、このアルゴリズムには、各チャンクの型として`RangeReplaceableCollection`を使用することもできる。

次の例は、名前のシーケンスを最初の文字でチャンク化する方法を示している。

```swift
let names = URL(fileURLWithPath: "/tmp/names.txt").lines
let groupedNames = names.chunked(on: \.first!)
for try await (firstLetter, names) in groupedNames {
    print(firstLetter)
    for name in names {
        print("  ", name)
    }
}
```

この種の投影によるチャンク化の特別な特性は、非同期シーケンスの要素が順序付けられていることがわかっている場合、チャンク非同期シーケンスの出力は、`Dictionary`の`AsyncSequence`イニシャライザを使用して`Dictionary`を初期化するのに適していること。これは、並べ替えの特性に一致するように投影を簡単に設計できるため、出力がチャンクを「値」として持つ一意の「キー」のペアの配列のパターンに一致することを保証できる。

上記の例では、名前が順序付けられていることがわかっている場合は、各「最初の文字」投影の一意性を利用して、次のように`Dictionary`を初期化できる。

```swift
let names = URL(fileURLWithPath: "/tmp/names.txt").lines
let nameDirectory = try await Dictionary(uniqueKeysWithValues: names.chunked(on: \.first!))
```

##### カウントまたはシグナル

チャンクは、要素自体ではなく、外部要因によって決定される場合がある。チャンクを特定のサイズに制限したり、「シグナル」と呼ばれる別の非同期シーケンスでチャンクを区切ったりすることができる。この特定のチャンク化ファミリは、値に関係なく、要素が個々の要素よりもチャンクとしてより効率的に処理されるシナリオで役立つ。

このファミリは、メソッドの2つのサブファミリに分けらる。シグナルとオプショナルのカウントを使用するもの(`AsyncChunksOfCountOrSignalSequence`を返す)と、カウントのみを処理するもの(`AsyncChunksOfCountSequence`を返す)。両方のサブファミリは、要素の型として収集される。指定されていない場合は`Array`です。これらのサブファミリは、再スローする。ベースの`AsyncSequence`がスローする可能性がある場合は、チャンクシーケンスもスローする可能性がある。同様に、ベースの`AsyncSequence`がスローしない場合、チャンクシーケンスもスローできない。


##### カウントのみ

チャンクサイズの制限が`ofCount`引数で指定されている場合、シーケンスは、最大で指定された数の要素で`Collected`型のチャンクを生成する。チャンクが指定されたサイズに達すると、非同期シーケンスはすぐにそれを発行する。

たとえば、`UInt8`バイトの非同期シーケンスは、次のように最大1024バイトの`Data`インスタンスにチャンク化できる。

```swift
let packets = bytes.chunks(ofCount: 1024, into: Data.self)
for try await packet in packets {
    write(packet)
}
```

##### シグナルのみ

シグナル非同期シーケンスが指定されている場合、チャンク非同期シーケンスは、シグナルが送信されるたびにチャンクを発行する。シグナル要素の値は無視される。チャンク非同期シーケンスが前回の出力以降に要素を蓄積していない場合、シグナルに応答して値は出力されない。

時間はチャンクの望ましい描写をシグナリングするよくある方法であるため、`AsyncTimerSequence`を使用する事前に特殊化された一連のオーバーロードがある。これらは、`AsyncTimerSequence`の静的メンバイニシャライザを使用して簡単な初期化を可能にする。

例として、ログメッセージの非同期シーケンスは、次のように4秒のセグメントでログの配列にチャンク化できる。

```swift
let fourSecondsOfLogs = logs.chunked(by: .repeating(every: .seconds(4)))
for await chunk in fourSecondsOfLogs {
    send(chunk)
}
```

##### カウントまたはシグナル

カウントとシグナルの両方が指定されている場合、チャンクが指定されたサイズに達するか、シグナル非同期シーケンスが発行するたびに、チャンク非同期シーケンスがチャンクを発行する。シグナルによってチャンクが出力されると、累積要素数はゼロにリセットされる。`AsyncTimerSequence`をシグナルとして使用する場合、タイマーは`AsyncChunksOfCountOrSignalSequence`のイテレータで`next`が初めて呼び出された瞬間から開始され、その瞬間から一定の間隔で出力される。タイマーの出力のスケジューリングは、カウントに基づいて出力されるチャンクの影響を受けないことに注意。

上記の例のように、このコードは最大1024バイトの`Data`インスタンスを出力するが、チャンクも毎秒出力される。

```swift
let packets = bytes.chunks(ofCount: 1024 or: .repeating(every: .seconds(1)), into: Data.self)
for try await packet in packets {
    write(packet)
}
```

どのチャンク化ファミリのどの設定でも、ベースの非同期シーケンスが終了すると、次の2つのいずれかが発生する:
1. 部分的なチャンクが発行される
2. チャンクが発行されない(例: その前のチャンクの出力によってイテレータが要素を受信しなかった場合)。

エラーがスローされた場合を除いて、ベースの非同期シーケンスの要素が破棄されることはない。

### Compacted

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Compacted.md

オプショナルの値を含む`Sequence`型で`.compactMap { $0 }`が必要になるのが一般的であるのと同様に、`AsyncSequence`型にも同じユースケースがある。この共通タスクは、型がオプショナルの値を判定するためにクロージャを使用する必要があることを意味する。これは、実行パフォーマンスと入力のAPI効率の両方でより効率的に実行できる。

```swift
extension AsyncSequence {
    public func compacted<Unwrapped>() -> AsyncCompactedSequence<Self, Unwrapped>
        where Element == Unwrapped?
}
```

これは、動作の観点から`.compactMap { $0 }`を記述することと同じだが、クロージャを実行または保存する必要がないため、推論が容易で効率的。

Effectの観点から見た`AsyncCompactedSequence`型は、`AsyncCompactMapSequence`と同じように機能する。ベースの非同期シーケンスがスローされると、`AsyncCompactedSequence`のイテレーションがスローされる可能性がある。同様に、ベースがスローされない場合、`AsyncCompactedSequence`のイテレーションはスローされない。この型は、ベース、ベースの要素、およびベースのイテレータが`Sendable`の場合、条件付きで`Sendable`になる(conditional conformance)。

### RemoveDuplicates

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/RemoveDuplicates.md

時間の経過とともに値を処理する場合、同じ値が連続して発生する可能性がある。すべての実際の値を明確に必要としない場合は、前回と異なる値のみを考慮に入れた方がと便利。特に、これは、`Equatable`である場合、または条件を指定することによって、重複する値を削除できる。

`removeDuplicates()`および`removeDuplicate(by:)` APIは、発生する重複値を削除する。これらのアルゴリズムは前の値に対して判定し、ベースの`AsyncSequence`の最新のイテレーションが、`next`を再度呼び出す直前のイテレーションと同じであるかどうかを判定する。結果の`AsyncRemoveDuplicatesSequence`は、重複する値が隣り合って発生しないことを保証する。これは、一意の新しい値を発行するだけではないので、それとは混同しないように注意が必要。ここで、各値は対象となるコレクションに対して判定される。

`removeDuplicates`ファミリには、3つのバリエーションがある。1つは、`Element`型が`Equatable`であること。このバリエーションは、`.removeDuplicates { $0 == $1 }`の省略形。次のバリエーションは、カスタムの条件を適用できるようにするクロージャバージョン。このアルゴリズムでは、`Element`自体が`Equatable`ではない場合でも、要素の一部を比較できる。最後に、比較メソッドがスローする可能性がある場合に比較を可能にするバリエーション。

`Element`型が`Equatable`または`non-throwing`な条件を指定するタイプである場合、これらはタイプ`AsyncRemoveDuplicatesSequence`型を利用する。スローする条件を指定するタイプは、`AsyncThrowingRemoveDuplicatesSequence`を使用する。これらのタイプは両方とも、ベース、ベースの要素、およびベースのイテレータが`Sendable`である場合に`Sendable`に条件付きで準拠(conditional conformance)している。

`AsyncRemoveDuplicatesSequence`は、ベースの非同期シーケンスがスローした場合は再スローし、ベースの非同期シーケンスがスローしない場合はスローしない。

`AsyncThrowingRemoveDuplicatesSequence`は、ベースの非同期シーケンスがスローした場合に再スローし、指定した条件がスローする可能性があるためにベースの非同期シーケンスがスローしない場合でもスローする可能性がある。

### Intersperse

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Intersperse.md

非同期シーケンスの各要素の間に指定された値を配置する。

```swift
let numbers = [1, 2, 3].async.interspersed(with: 0)
for await number in numbers {
    print(number)
}
// prints 1 0 2 0 3

let empty = [].async.interspersed(with: 0)
// await Array(empty) == []
```

`interspersed(with:)`はセパレータの値を受け取り、非同期シーケンスのすべての要素の間に挿入する。

新しく生成された`AsyncInterspersedSequence`型は、各要素の間にセパレータが挿入されたときの非同期シーケンスを表す。

ベースの非同期シーケンスがイテレーションでスローする場合、`AsyncInterspersedSequence`はイテレーションでスローする。ベースがスローしない場合、`AsyncInterspersedSequence`のイテレーションもスローしない。

`AsyncInterspersedSequence`は、ベースの非同期シーケンスが`Sendable`であり、要素も`Sendable`である場合、`Sendable`に条件付きで準拠(conditional conformance)している。

## 時間内に処理する非同期シーケンス

### Debounce

イベントが期待される消費よりも多く発生する可能性がある場合、その状況を処理する方法は複数ある。1つのアプローチは、特定の非アクティブ期間、つまり「静止時間」が経過した後にのみ値を出力すること。このアルゴリズムは、一般にdebounceと呼ばれる。

debounceアルゴリズムは、イベント間で特定の期間が経過した後に要素を生成する。時計に適用される所定の許容範囲内で処理する。この静止期間中にベースの`AsyncSequence`によって値が生成された場合、値が生成されない期間が経過するまで、または終了イベントが発生しない限り、debounceは次のイテレータを再開しない。

このアルゴリズムのインターフェースは、ベース型、イテレータ、および要素が`Sendable`であるすべての`AsyncSequence`型で使用できる。これは、このアルゴリズムが本質的にイベントのタイミングを管理するタスクを作成するため。時計が`ContinuousClock`である場合、簡略化された実装が提供されている。これにより、`Duration`値を使用して簡単に構築できる。

これらはすべて、時間の経過とともに非同期シーケンスを変換する方法の簡潔な説明に要約される。

```swift
fastEvents.debounce(for: .seconds(1))
```

この場合、イベントの潜在的に高速な非同期シーケンスを、値を発行する前に1秒間イベントが発生せずに待機するシーケンスに変換する。

debounceのアルゴリズムを実装する型は、適用されるベースと同じ`Element`型を出力する。また、ベース型がスローするときにもスローする(同様に、ベース型がスローしない場合はスローしない)。

`AsyncDebounceSequence`を構成する内部に保持された型は、`Sendable`でなければならないため、 `AsyncDebounceSequence`は無条件で常に`Sendable`。

### Throttle

イベントが期待される消費よりも速く発生する可能性がある場合、その状況を処理する方法は複数ある。1つのアプローチは、特定の期間が経過した後に値を発行すること。これらの出力された値は、待機期間中に発生した値から減らすことができる。このアルゴリズムは、一般にthrottleと呼ばれる。

throttleアルゴリズムは、要素間で少なくとも特定の間隔が経過するような要素を生成する。特定の時計に対して測定することによって処理する。値がベースの`AsyncSequence`によって生成された場合、throttleは、期間が経過するまで、または終了イベントが発生しない限り、次のイテレータを再開しない。

このアルゴリズムのインターフェースは、すべての`AsyncSequence`型タイプで使用できる。debounceなどの他のアルゴリズムとは異なり、throttleアルゴリズムでは、間隔が測定されるだけなので、追加のタスクを作成したり、何らかの許容誤差を要求したりする必要はない。時計が`ContinuousClock`である場合、簡略化された実装が提供されている。これにより、`Duration`値を使用して簡単に構築できる。イベントの生成の抑制された領域の左端または右端を表す「最も遅い」または「最も早い」値を出力するような、値を減らすための省略形も提供されている。

これらはすべて、時間の経過とともに非同期シーケンスを変換する方法の簡潔な説明に要約される。

```swift
fastEvents.throttle(for: .seconds(1))
```

この場合、イベントの潜在的に高速な非同期シーケンスを、値を発行する前に1秒間イベントが発生せずに待機するシーケンスに変換する。

この場合、throttleは、イベントの潜在的に高速な非同期シーケンスを、値を発行する前に1秒経過するのを待つシーケンスに変換する。

throttleのアルゴリズムを実装する型は、適用されるベースと同じ`Element`型を出力する。また、ベース型がスローするときにもスローする(同様に、ベース型がスローしない場合はスローしない)。

ベース型が`Sendable`で構成されている場合、`AsyncThrottleSequence`とその`Iterator`は条件的に`Sendable`である。

イベントが測定される時間は、(存在する場合)前回の出力からの時間。最後の出力からthrottleが測定された時点までの期間が経過した場合、その期間は経過としてカウントされる。最初の要素は、開始から最初の要素までの間隔を構築できないため、throttleされていないと見なされる。

### Timer

一定の間隔で要素を生成すると、他のアルゴリズムと組み合わせた場合に役立つ。これらは、特定の時間にコードを呼び出すことから、イベントの区切りとしてそれらの定期的な間隔を使用することまで多岐にわたる。こういったAPIが存在する他のケースもあるが、それらは現在Swift Concurrencyとうまくやり取りできない。`Timer`や`DispatchTimer` などの既存のAPIは、拡張できない内部時計にバインドされている。

新しい`Clock`、`Instant`、`Duration`型を利用する`AsyncTimerSequence`により、`Clock`に準拠する型のカスタム実装とタイマーでやり取りが可能になる。

この非同期シーケンスは、一定間隔が経過した後、`Clock`の`Instant`型の要素を生成する。 その瞬間が、スリープから再開した`now`となる。`next`を呼び出すたびに、`AsyncTimerSequence.Iterator`は次の期限を計算して再開し、それと許容範囲をその時計に渡す。いずれかの時点で、そのイテレーションを実行しているタスクがキャンセルされた場合、イテレーションは`next`の呼び出し時に`nil`を返す。


非同期シーケンスを含むすべての型とそのイテレータは`Sendable`であるため、これらの型も`Sendable`。


## 非同期シーケンスからすべての値を取得

### Collection Initializers

https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Collections.md

`Array`、`Dictionary`および`Set`は、要素のコレクションを保存するための最も一般的に使用されるデータ構造の一部。非同期からコレクションへの移行方法を持つことは、役に立つショートハンドであるだけではなく、非同期を消費する方法に対してダイレクトに意図を表現するための強力な方法でもある。

このタイプの機能は、例とテストに役立つ可能性がある。また、処理する前に完全に値の取得が完了したコレクションを期待する既存のAPIとのインターフェースにも役立つ。

イニシャライザの3つのカテゴリが追加され、これらの3つの主要な種類のイニシャライザを提供する:`Array`、`Dictionary`および`Set`。ただし、これらのイニシャライザは、すべての同様のコレクションに適用できるように、ジェネリックな方法で記述可能。

`RangeReplaceableCollection`は、`AsyncSequence`からコレクションを構築する新しいイニシャライザが追加。これにより、非同期シーケンスから配列を作成できるが、`Data`や`ContiguousArray`などの型を作成することもできる。非同期シーケンスの性質により、このイニシャライザは非同期であり、ベースの非同期シーケンスからエラーをrethrowすることを宣言する必要がある。

```swift
extension RangeReplaceableCollection {
    public init<Source: AsyncSequence>(
        _ source: Source
    ) async rethrows 
        where Source.Element == Element
}
```

`Dictionary`は、既存の`Sequence`ベースのイニシャライザに類似するため、新しく非同期でrethrowするイニシャライザのファミリが追加される。イニシャライザは、`Dictionary`の非同期イニシャライザに加えて、非同期の可能性のある`uniquingKeys`やその他のタスクとのやり取りをしやすくするために非同期になる。

```swift
extension Dictionary {
    public init<S: AsyncSequence>(
        uniqueKeysWithValues keysAndValues: S
    ) async rethrows 
        where S.Element == (Key, Value)

    public init<S: AsyncSequence>(
        _ keysAndValues: S, 
        uniquingKeysWith combine: (Value, Value) async throws -> Value
    ) async rethrows
        where S.Element == (Key, Value)

    public init<S: AsyncSequence>(
        grouping values: S, 
        by keyForValue: (S.Element) async throws -> Key
    ) async rethrows
        where Value == [S.Element]
}
```

`SetAlgebra`は、非同期シーケンスを考慮して、新しく`AsyncSequence`から`SetAlgebra`型を構築するイニシャライザが追加。これにより、非同期シーケンスからセットを作成し、`OptionSet`や`IndexSet`などの型を作成できる。

```swift
extension SetAlgebra {
    public init<Source: AsyncSequence>(
        _ source: Source
    ) async rethrows
        where Source.Element == Element
}
```


各イニシャライザは、初期化に使用されている`AsyncSequence`が有限であることとわかっている場合での利用を目的としている。一般的な用途には以下が含まれる:

- `AsyncBytes`スタイルのシーケンスまたは`lines`アクセサを介したファイルからの読み取り
- `TaskGroup`によって生成された要素の収集
- 無限`AsyncSequence`のプレフィックスへのアクセス

各イニシャライザは、`for-await-in`/`for-try-await-in`の構文を使用して、イニシャライザのシーケンスを直接イテレーションする。さらに、各イニシャライザは、キャンセルを適切に配慮するために、受け取った`AsyncSequence`に依存している。キャンセルが潜在的に起きる可能性がある場合、開発者は、使用されている`AsyncSequence`の動作に応じて、すぐにチェックするか、部分的なシーケンスに基づいて初期化の準備をするべき。

#### RangeReplaceableCollection

```swift
let contents = try await Data(URL(fileURLWithPath: "/tmp/example.bin").resourceBytes)
```

#### Dictionary

```swift
let table = await Dictionary(uniqueKeysWithValues: zip(keys, values))
```

#### SetAlgebra

```swift
let allItems = await Set(items.prefix(10))
```

## Taskの管理

### Task.select

多くのアルゴリズムの基本的な部分は、アクティブなタスクの特定のリストから最初に解決されたタスクを選択できること。これにより、`debounce`や`merge`などのアルゴリズムが可能になる。

アクティブなタスクのリストから完了した最初のタスクを選択することは、`select(2)`と同様のアルゴリズム。これは、子タスクの代わりに、この関数がすでに実行中のタスクも処理でき、選択された際に完了時に他の子タスクをキャンセルせず、リスト内のすべてのタスクの完了を待つ必要がないことを除いて、`TaskGroup`と同様の動作をする。

同じ成功と失敗の型を共有する任意の数の`Task`オブジェクトが与えられた際、`Task.select`は、各タスクの結果をsuspendして待機し、最初のタスクが結果を生成したときに`resume`する。`Task.select`の自体のタスクがsuspend中にそのタスクがキャンセルされると、選択されているタスクもキャンセルされる。これは`TaskGroup`のファミリに似ているが、動作と構造にいくつかの違いがある:

`withTaskGroup`APIは、効率的な子タスクを作成する。`Task.select`APIは、既存のタスクを受け取る。

`withTaskGroup`APIは、すべての子タスクの完了を待ってから戻り値を返す。`Task.select`APIは、最初のタスクの完了を待ってから戻り値を返す。

`withTaskGroup`APIは、戻り値を待機中、すべての未処理の子タスクをキャンセルする。`Task.select`APIは、選択されていないタスクは実行し続けることができる。

`withTaskGroup`は、子タスクない状態で待機できる。`Task.select`APIは、選択するために少なくとも1つのタスクを必要とする。0個の場合はプログラマエラー。

つまり、`withTaskGroup`は作業を並行して実行するのに非常に適しているが、`Task.select`は、値を提供する最初のタスクを見つけることを目的としている。子タスクではないタスクには固有の追加コストがあるため、`Task.select`は、グループとしてより適していると思われる箇所の置き換えに使うべきではなく、高度なアルゴリズムが必要になりそうな場所に提供するべき。

## 効果(Effects)

各アルゴリズムには、特定の動作効果がある。スロー効果の場合、これらは、シーケンスがスローするか、スローしないか、またはエラーを再スローする場合のいずれかになる。一部の非同期シーケンスでの送信可能性の影響は条件付きだが、他のシーケンスでは、`Sendable`の要件を満たすために、構成要素がすべて`Sendable`であることが必要。

<details>
<summary>効果の一覧</summary>

| 種類                                                 | スロー        | `Sendable` |
|-----------------------------------------------------|--------------|------------|
| `AsyncAdjacentPairsSequence`                        | rethrows     | 条件付き |
| `AsyncBufferedByteIterator`                         | throws       | Sendable |
| `AsyncBufferSequence`                               | rethrows     | 条件付き |
| `AsyncBufferSequence.Iterator`                      | rethrows     | 条件付き |
| `AsyncChain2Sequence`                               | rethrows     | 条件付き |
| `AsyncChain2Sequence.Iterator`                      | rethrows     | 条件付き |
| `AsyncChain3Sequence`                               | rethrows     | 条件付き |
| `AsyncChain3Sequence.Iterator`                      | rethrows     | 条件付き |
| `AsyncChannel`                                      | non-throwing | Sendable |
| `AsyncChannel.Iterator`                             | non-throwing | Sendable |
| `AsyncChunkedByGroupSequence`                       | rethrows     | 条件付き |
| `AsyncChunkedByGroupSequence.Iterator`              | rethrows     | 条件付き |
| `AsyncChunkedOnProjectionSequence`                  | rethrows     | 条件付き |
| `AsyncChunkedOnProjectionSequence.Iterator`         | rethrows     | 条件付き |
| `AsyncChunksOfCountOrSignalSequence`                | rethrows     | Sendable |
| `AsyncChunksOfCountOrSignalSequence.Iterator`       | rethrows     | Sendable |
| `AsyncChunksOfCountSequence`                        | rethrows     | 条件付き |
| `AsyncChunksOfCountSequence.Iterator`               | rethrows     | 条件付き |
| `AsyncCombineLatest2Sequence`                       | rethrows     | Sendable |
| `AsyncCombineLatest2Sequence.Iterator`              | rethrows     | Sendable |
| `AsyncCombineLatest3Sequence`                       | rethrows     | Sendable |
| `AsyncCombineLatest3Sequence.Iterator`              | rethrows     | Sendable |
| `AsyncCompactedSequence`                            | rethrows     | 条件付き |
| `AsyncCompactedSequence.Iterator`                   | rethrows     | 条件付き |
| `AsyncDebounceSequence`                             | rethrows     | Sendable |
| `AsyncDebounceSequence.Iterator`                    | rethrows     | Sendable |
| `AsyncExclusiveReductionsSequence`                  | rethrows     | 条件付き |
| `AsyncExclusiveReductionsSequence.Iterator`         | rethrows     | 条件付き |
| `AsyncInclusiveReductionsSequence`                  | rethrows     | 条件付き |
| `AsyncInclusiveReductionsSequence.Iterator`         | rethrows     | 条件付き |
| `AsyncInterspersedSequence`                         | rethrows     | 条件付き |
| `AsyncInterspersedSequence.Iterator`                | rethrows     | 条件付き |
| `AsyncJoinedSequence`                               | rethrows     | 条件付き |
| `AsyncJoinedSequence.Iterator`                      | rethrows     | 条件付き |
| `AsyncLazySequence`                                 | non-throwing | 条件付き |
| `AsyncLazySequence.Iterator`                        | non-throwing | 条件付き |
| `AsyncLimitBuffer`                                  | non-throwing | Sendable |
| `AsyncMerge2Sequence`                               | rethrows     | Sendable |
| `AsyncMerge2Sequence.Iterator`                      | rethrows     | Sendable |
| `AsyncMerge3Sequence`                               | rethrows     | Sendable |
| `AsyncMerge3Sequence.Iterator`                      | rethrows     | Sendable |
| `AsyncRemoveDuplicatesSequence`                     | rethrows     | 条件付き |
| `AsyncRemoveDuplicatesSequence.Iterator`            | rethrows     | 条件付き |
| `AsyncThrottleSequence`                             | rethrows     | 条件付き |
| `AsyncThrottleSequence.Iterator`                    | rethrows     | 条件付き |
| `AsyncThrowingChannel`                              | throws       | Sendable |
| `AsyncThrowingChannel.Iterator`                     | throws       | Sendable |
| `AsyncThrowingExclusiveReductionsSequence`          | throws       | 条件付き |
| `AsyncThrowingExclusiveReductionsSequence.Iterator` | throws       | 条件付き |
| `AsyncThrowingInclusiveReductionsSequence`          | throws       | 条件付き |
| `AsyncThrowingInclusiveReductionsSequence.Iterator` | throws       | 条件付き |
| `AsyncTimerSequence`                                | non-throwing | Sendable |
| `AsyncTimerSequence.Iterator`                       | non-throwing | Sendable |
| `AsyncZip2Sequence`                                 | rethrows     | Sendable |
| `AsyncZip2Sequence.Iterator`                        | rethrows     | Sendable |
| `AsyncZip3Sequence`                                 | rethrows     | Sendable |
| `AsyncZip3Sequence.Iterator`                        | rethrows     | Sendable |
</details>
</br>

## 参考リンク

- [Meet Swift Async Algorithms](https://developer.apple.com/videos/play/wwdc2022/110355)
- [swift-async-algorithms](https://github.com/apple/swift-async-algorithms)
- [Introducing Swift Async Algorithms](https://www.swift.org/blog/swift-async-algorithms/)

### Forums

- [Introducing Swift Async Algorithms](https://forums.swift.org/t/introducing-swift-async-algorithms/56231)
- [Relation to Combine framework](https://forums.swift.org/t/relation-to-combine-framework/56236)

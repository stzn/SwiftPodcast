# Swift Concurrency WWDC22

収録日: 2022/06/26

- [Swift Concurrency WWDC22](#swift-concurrency-wwdc22)
- [Swift Concurrency利用時のよくあるパフォーマンスの問題](#swift-concurrency利用時のよくあるパフォーマンスの問題)
  - [内容](#内容)
  - [Main actorブロッキング](#main-actorブロッキング)
    - [問題点](#問題点)
    - [例](#例)
    - [解決策](#解決策)
  - [actor競合](#actor競合)
    - [問題点](#問題点-1)
    - [解決方法](#解決方法)
  - [スレッドプールの枯渇](#スレッドプールの枯渇)
    - [問題点](#問題点-2)
    - [解決方法](#解決方法-1)
  - [Continuationの誤用](#continuationの誤用)
    - [問題点](#問題点-3)
    - [解決方法](#解決方法-2)
- [データ競合を防ぐ](#データ競合を防ぐ)
  - [要約](#要約)
    - [Taskの分離](#taskの分離)
    - [タスク間の通信](#タスク間の通信)
    - [`Sendable`プロトコル](#sendableプロトコル)
      - [`Sendable`型は安全に共有できる](#sendable型は安全に共有できる)
      - [型の`Sendable`チェック](#型のsendableチェック)
      - [`Sendable`関数型](#sendable関数型)
    - [actor分離](#actor分離)
      - [actor型](#actor型)
      - [相互排除](#相互排除)
      - [actor分離の維持](#actor分離の維持)
      - [actor自身の参照の分離](#actor自身の参照の分離)
      - [actorに分離されていない(Non-isolated)コード](#actorに分離されていないnon-isolatedコード)
      - [@MainActor](#mainactor)
      - [型に@MainActorを付与する](#型にmainactorを付与する)
    - [原子性（高レベルのデータ競合の防止）](#原子性高レベルのデータ競合の防止)
    - [順序](#順序)
  - [例を用いた解説](#例を用いた解説)
    - [タスクの分離](#タスクの分離)
    - [タスク間の通信](#タスク間の通信-1)
    - [値型 - 値の変更はローカル上にしか影響がない](#値型---値の変更はローカル上にしか影響がない)
    - [参照型 - 値の変更はグローバルに影響を与える](#参照型---値の変更はグローバルに影響を与える)
    - [`Sendable`プロトコル](#sendableプロトコル-1)
    - [タスクの境界を越えた`Sendable`チェック](#タスクの境界を越えたsendableチェック)
    - [タスクAPIの`Sendable`制約](#タスクapiのsendable制約)
    - [タイプの`Sendable`チェック](#タイプのsendableチェック)
    - [タスク作成中の`Sendable`チェック](#タスク作成中のsendableチェック)
    - [`Sendable`関数型](#sendable関数型-1)
  - [actor分離](#actor分離-1)
    - [actor型](#actor型-1)
    - [actorは相互排除](#actorは相互排除)
    - [actorは参照分離](#actorは参照分離)
    - [actorに分離されていないasyncコード](#actorに分離されていないasyncコード)
    - [actorに分離されていないsyncコード](#actorに分離されていないsyncコード)
    - [actorのまとめ](#actorのまとめ)
  - [@MainActor](#mainactor-1)
    - [@MainActorの関数とクロージャ](#mainactorの関数とクロージャ)
    - [@MainActorの型](#mainactorの型)
  - [原子性（高レベルのデータ競合の防止）](#原子性高レベルのデータ競合の防止-1)
    - [非トランザクションコード](#非トランザクションコード)
  - [順序](#順序-1)
    - [actorは厳密にFIFOではない](#actorは厳密にfifoではない)
    - [順序づけのためのツール](#順序づけのためのツール)
      - [タスクは順番にコードを実行する](#タスクは順番にコードを実行する)
      - [`AsyncStream`は、要素を順番に配信する](#asyncstreamは要素を順番に配信する)
  - [Strict Concurrency Checking(Swift6へのマイグレーション)](#strict-concurrency-checkingswift6へのマイグレーション)
    - [Minimal mode](#minimal-mode)
    - [Targeted mode](#targeted-mode)
    - [Complete mode](#complete-mode)
    - [推奨](#推奨)
- [Swift Async Algorithm](#swift-async-algorithm)
- [References](#references)

# Swift Concurrency利用時のよくあるパフォーマンスの問題

## 内容

Swift Concurrencyにより、正しい並行処理コードと並列処理コードが簡単に記述でき流ようになった。ただし、間違って使っているコードを作成する可能性はいまだにの凝っている。また、正しく使用しているものの、目指していたパフォーマンス上のメリットが得られていない可能性もある。

主にこれらの問題は4つに分類できる。

- Main actorブロッキング
- Actor競合
- スレッドプールの枯渇
- Continuationの誤用

これらの原因と解決策を見ていく。

## Main actorブロッキング

Main actorブロッキングは、すごい時間のかかる処理をMain actor上で実行した場合に起こる。

### 問題点

Main actorは、メインスレッドですべての処理を実行する特別なactor。UI処理はメインスレッドで実行する必要があり、Main actorを使用すると、UIコードにSwift Concurrencyが使用できる。

ただし、メインスレッドはUIにとって非常に重要であるため、常に使用可能である必要があり、長時間実行される一つの処理に占有されるべきではない。これが発生すると、アプリがロックされて応答しなくなったように見える。

### 例

```swift
@MainActor // 1 
class CompressionState: ObservableObject {
    @Published var files: [FileStatus] = []
    var logs: [String] = []

    func update(url: URL, progress: Double) {
        if let loc = files.firstIndex(where: {$0.url == url}) {
            files[loc].progress = progress
        }
    }

    func update(url: URL, uncompressedSize: Int) {
        if let loc = files.firstIndex(where: {$0.url == url}) {
            files[loc].uncompressedSize = uncompressedSize
        }
    }

    func update(url: URL, compressedSize: Int) {
        if let loc = files.firstIndex(where: {$0.url == url}) {
            files[loc].compressedSize = compressedSize
        }
    }

    func compressAllFiles() {
        for file in files {
            Task {
                let compressedData = compressFile(url: file.url)
                await save(compressedData, to: file.url)
            }
        }
    }

    func compressFile(url: URL) -> Data { // 2
        log(update: "Starting for \(url)")
        let compressedData = CompressionUtils.compressDataInFile(at: url) { uncompressedSize in
            update(url: url, uncompressedSize: uncompressedSize)
        } progressNotification: { progress in
            update(url: url, progress: progress)
            log(update: "Progress for \(url): \(progress)")
        } finalNotification: { compressedSize in
            update(url: url, compressedSize: compressedSize)
        }
        log(update: "Ending for \(url)")
        return compressedData
    }

    func log(update: String) {
        logs.append(update)
    }
}
```

1. `CompressionState`クラス全体を`@MainActor`で実行するようにアノテーションが付けられている。ここでの`@Published`プロパティは、メインスレッドからのみ更新する必要がある。そうしないと、実行時に問題が発生する可能性がある
2. `compressFile`関数は`CompressionState`クラス内にある。そのため、タスクはメインスレッドで実行される

### 解決策

Main actorで実行されているコードはすぐに終了して処理を完了させるか、計算をMain actorからバックグラウンドに移動させる必要がある。処理を通常のactorまたは分離タスク(detached Task)に配置することでバックグラウンドに移動できる。Main actorでは、小さな処理単位で、UIを更新したりメインスレッドで実行する必要のある他のタスクを実行する。

上記のクラスには、2つの異なる可変状態がある。

- `@Published var files：[FileStatus]`: Main actor
- `var logs: [String]`: データ競合を防ぐ必要がありますが、Main actor上にある必要はない

そこで、`logs`を独自のactorにラップする。

```swift
actor ParallelCompressor { // 1
    var logs: [String] = []
    unowned let status: CompressionState

    init(status: CompressionState) {
        self.status = status
    }

    func compressFile(url: URL) -> Data {
        log(update: "Starting for \(url)")
        let compressedData = CompressionUtils.compressDataInFile(at: url) { uncompressedSize in
            Task { @MainActor in
                status.update(url: url, uncompressedSize: uncompressedSize)
            }
        } progressNotification: { progress in
            Task { @MainActor in
                status.update(url: url, progress: progress)
                await log(update: "Progress for \(url): \(progress)")
            }
        } finalNotification: { compressedSize in
            Task { @MainActor in
                status.update(url: url, compressedSize: compressedSize)
            }
        }
        log(update: "Ending for \(url)")
        return compressedData
    }

    func log(update: String) {
        logs.append(update)
    }
}

@MainActor
class CompressionState: ObservableObject {
    @Published var files: [FileStatus] = []
    var compressor: ParallelCompressor!

    init() {
        self.compressor = ParallelCompressor(status: self)
    }

    func update(url: URL, progress: Double) {
        if let loc = files.firstIndex(where: {$0.url == url}) {
            files[loc].progress = progress
        }
    }

    func update(url: URL, uncompressedSize: Int) {
        if let loc = files.firstIndex(where: {$0.url == url}) {
            files[loc].uncompressedSize = uncompressedSize
        }
    }

    func update(url: URL, compressedSize: Int) {
        if let loc = files.firstIndex(where: {$0.url == url}) {
            files[loc].compressedSize = compressedSize
        }
    }

    func compressAllFiles() {
        for file in files {
            Task {
                let compressedData = await compressor.compressFile(url: file.url) // 2
                await save(compressedData, to: file.url)
            }
        }
    }
}
```

1. `logs`変数を参照するコードを`CompressionState`クラスから削除し、`ParallelCompressor` actorに追加
2. `CompressionState`を更新して、`ParallelCompressor`で`compressFile`を呼び出す


## actor競合

上記の例では、UIのフリーズは起きなくなった。大きな改善だが、期待した速度が得られていない。

### 問題点

actorは、複数のタスク間で共有している状態を安全に操作できるようにする。これは、その共有している状態へのアクセスを一つ一つ順番に行うことによって行われる。 一度に1つのタスクのみがactorを占有でき、そのactorを使用する必要がある他のタスクは待機する。

Swift Concurrencyは、非構造化タスク(unstructured task)、タスクグループ(TaskGroup)、および`async let`を使用した並列計算ができる。理想的には、多くのCPUコアを同時に利用できるのが良い。このようなコードからactorを使用する場合、これらのタスク間で共有している一つのactor上で大量の処理を実行するのは注意が必要。複数のタスクが同じactorを同時に使用しようとすると、actorはそれらのタスクの実行を一つ一つ順番に行う。このため、並列計算のパフォーマンス上の利点が失われる。

<img src="../images/Swift Concurrency WWDC22/actor_run_one_task_at_once.png" alt= "actorは一度に一つのタスクのみ実行する" width="100%">

<img src="../images/Swift Concurrency WWDC22/actor_contention.png" alt= "actor競合" width="100%">

下記のコードは、クロージャが主に圧縮処理を実行していることを示している。 `compressFile`関数は`ParallelCompressorActor`の一部であるため、`compressAllFiles`関数の実行全体はこのactorで行われ、他のすべての圧縮処理をブロックしている。

```swift

// Main actor上で実行される
func compressAllFiles() {
    for file in files {
        Task {
            // ParallelCompressor actor
            let compressedData = await compressor.compressFile(url: file.url) // 1
            await save(compressedData, to: file.url)
        }
    }
}
```

<img src="../images/Swift Concurrency WWDC22/compressFile_run_on_actor.png" alt= "compressFileをactor上で実行" width="100%">


### 解決方法

タスクがactorのデータへの排他的アクセスを本当に必要とする場合にのみactor上で実行されるようにする必要がある。他のすべてはactor上で実行する必要がない。タスクを小さい単位(チャンク)に分割して一部のチャンクはactor上で実行し、他のチャンクは実行しない。

上記の例だと、`compressFile`関数をactorの保護から分離タスクに移動する。そして関連する可変状態を更新するために必要な場合に限り、actor上で実行する。したがって、圧縮関数は、actorで保護された状態にアクセスする必要があるまで、スレッドプール内の任意のスレッドで自由に実行できる。また、actorに制約されないため、スレッドの数によってのみ制限され、すべてを同時並列に実行できる。


<img src="../images/Swift Concurrency WWDC22/compressFile_on_any_thread_in_the_thread_pool.png" alt= "compressFileをスレッドプールの任意のスレッドで実行" width="100%">

```swift
actor ParallelCompressor {
    // ..
    nonisolated func compressFile(url: URL) async -> Data { // 1
        await log(update: "Starting for \(url)") // 2
        let compressedData = CompressionUtils.compressDataInFile(at: url) { uncompressedSize in
            Task { @MainActor in
                status.update(url: url, uncompressedSize: uncompressedSize)
            }
        } progressNotification: { progress in
            Task { @MainActor in
                status.update(url: url, progress: progress)
                await log(update: "Progress for \(url): \(progress)")
            }
        } finalNotificaton: { compressedSize in
            Task { @MainActor in
                status.update(url: url, compressedSize: compressedSize)
            }
        }
        await log(update: "Ending for \(url)") // 2
        return compressedData
    }
}

@MainActor
class CompressionState: ObservableObject {
    // ...
    func compressAllFiles() {
        for file in files {
            Task.detached { // 3
                let compressedData = await self.compressor.compressFile(url: file.url) // 4
                await save(compressedData, to: file.url)
            }
        }
    }
}
```

1. `compressFile`関数を`nonisolated`としてマーク
2. すべてのログ呼び出しを`await`キーワードでマーク
3. タスクの作成を更新して、分離タスク(detached task)を作成
4. 分離タスクの場合、明示的に`self`を`capture`する必要がある

## スレッドプールの枯渇

スレッドプールの枯渇は、パフォーマンスを低下させたり、アプリケーションをデッドロックさせたりする可能性がある。

### 問題点

Swift Concurrencyには、タスクに常に実行し続けることを要求する。タスクが何かを待つとき、それは通常中断(suspend)することによって行う(他の処理をブロックするわけではない)。ただし、タスク内のコードで、ファイルやネットワークIOのブロック、ロックの取得などのブロック呼び出しを中断せずに実行することもできる。しかし、これをしてしまうとタスクを実行し続けるという要件が破られる。これが発生すると、タスクは実行中のスレッドを占有し続けるが、実際にCPUコアを使用せずにアイドル状態にしている。この状態は、スレッドプールのスレッド数が制限されている中で、一部がブロックされているためSwift ConcurrencyのランタイムはすべてのCPUコアを完全に使用することができない。これにより、実行できる並列計算の量とアプリの最大パフォーマンスが低下する。極端な場合、スレッドプール全体がブロックしているタスクによって占有された中で、新しいタスクの作成を必要とする何かを待機(await)した場合、デッドロックが発生する可能性がある。

<img src="../images/Swift Concurrency WWDC22/thread_pool_exhaustion.png" alt= "スレッドプールの枯渇" width="100%">

### 解決方法

タスク内での呼び出しのブロックを避ける。

ファイルとネットワークIOは、非同期APIを使用して実行する必要がある。条件変数またはセマフォを待つことは避ける。 必要に応じて、すごい小さい範囲で短時間保持されるロックを使用することはできるが、競合が多いロックや長期間保持されるロックは避けるべき。

これらが必要な場合は、そのコードをSwift Concurrencyのスレッドプールの外に移動させる。

例えば `DispatchQueue`で実行し、`Continuation`を使用してSwift Concurrencyの世界にブリッジする。  
※システムを円滑に動かし続けるため、可能な限りブロックが必要な操作には非同期APIを使用すること。

## Continuationの誤用

`Continuation`を使用するときは、正しく使用するように注意する必要がある。

`Continuation`は現在のタスクを一時中断し、呼び出されたときにタスクを再開するコールバックを提供するこれは、コールバックベースの非同期APIで使用できる。

<img src="../images/Swift Concurrency WWDC22/continuation.png" alt= "continuation" width="100%">

### 問題点

`Continuation`のコールバックには特別な要件がある。それは、1回だけ呼び出す必要があり、それ以上でもそれ以下でも許されないということ。これはコールバックベースのAPIの一般的な要件だが、正式ではなく暗黙の了解になる傾向があり、言語によって強制されてもなく、よく見落とされる。

Swift Concurrencyでは、これは厳密な要件となる。コールバックが2回呼び出されると、プログラムがクラッシュまたは誤った挙動になる。コールバックが呼び出されない場合、タスクはリークする(再開しない)。

```swift
await withCheckedContinuation { continuation in // 1
    externalCallbackBasedAPI { value in // 2
        continuation.resume(returning: value) // 3
    }
}
```

1. `withCheckedContinuation`を使用して`Continuation`を取得
2. コールバックベースのAPIを呼び出す
3. `Continuation`を再開

コードがより複雑な場合は注意が必要。

```swift
await withCheckedContinuation { continuation in
    externalCallbackBasedAPI { value in
        if value.success { // ❌
            continuation.resume(returning: value)
        }
    }
}
```

成功した場合にのみ`Continuation`を再開するようにコールバックを変更。失敗した場合、`Continuation`は再開されず、タスクは永久に中断される。

```swift
await withCheckedContinuation { continuation in
    externalCallbackBasedAPI { value in
        continuation.resume(returning: value)
        continuation.resume(returning: value)　// ❌
    }
}
```

`Continuation`を2回再開する。これもバグで、アプリが誤まった動作をしたりクラッシュする。

### 解決方法

パフォーマンスが絶対的に重要でない限り、`Continuation`には常に`withCheckedContinuation`APIを使用する。`CheckedContinuation`は、誤用を自動的に検出し、エラーにフラグを立てる。`CheckedContinuation`が2回呼び出されると、`Continuation`はトラップされます。`Continuation`がまったく呼び出されない場合、`Continuation`が破棄されると、`Continuation`がリークしたことを警告するメッセージがコンソールに出力される。

# データ競合を防ぐ

`Sendable`とデータ構造の関係、actorの仕組みについて理解を深める。

## 要約

<details>

### Taskの分離

- Swift Concurrencyの実行モデルは、データ競合を引き起こす可能性のある方法でデータが共有されないようにする
- タスク内の処理は最初から最後まで順番通りに実行される
- タスクは非同期であり、タスクは待機(`await`)操作で何度でも中断できる
- タスクは自己完結型。各タスクは独自のリソースを保持し、独立して動作できる

### タスク間の通信

- 値型(構造体 struct、列挙型 enumなど)はタスク間の分離を維持し続ける。それらは他のタスクには何の影響も与えない
- 参照型は、その特定のオブジェクトへの参照のみを提供する。共有された可変データは、データ競合を起こしやすい。 Swiftコンパイラは、それらが誤って別のコンパイラに渡されないようにする

### `Sendable`プロトコル

#### `Sendable`型は安全に共有できる

- `Sendable`プロトコルは、データ競合を発生させることなく、異なるConcurrencyドメイン間で安全に共有できる型を記述するために使用される
-`Success`と呼ばれる`Task`の結果型は、`Sendable`プロトコルに準拠している必要がある
- Concurrencyドメイン間で渡される可能性があるジェネリックパラメータが存在する場合は、`Sendable`の制約を付ける

#### 型の`Sendable`チェック

- 列挙型と構造体は通常、値型を定義する。すべてのインスタンスデータも`Sendable`である限り、それらも`Sendable`であると思われる
- `Sendable`は、条件付き準拠(Conditional Conformance)を使用して、コレクションやその他のジェネリック型を介して伝播できる
- `public`ではない型に対して、これらの`Sendable`への準拠はすべて、Swiftコンパイラによっても推測できる
- クラスは参照型であるため、`final`クラスに不変のストレージしかない場合など、非常に狭い状況でのみ`Sendable`にすることができる
- 独自の内部同期を行う参照型を実装することは可能(たとえば、一貫してロックを使用するなど)。その後、`@ unchecked Sendable`を使用して、コンパイラのチェックを無効にすることができる
- ただし、`@ unchecked Sendable`は、Swiftが提供するデータ競合の安全性を損なう(安全性チェックが行なわれない)ため、注意が必要
- `Sendable`プロトコルに準拠する型のみで構成される`Sendable`も同様にサポートされている

#### `Sendable`関数型

- タスク作成には、新しい独立したタスクでクロージャを実行することも含まれる。これを行うと、元のタスクから値をキャプチャして新しいタスクに渡すことができるため、データ競合が発生しないように`Sendable`チェックが必要
- クロージャは`Sendable`クロージャであると推測される
- `Sendable`クロージャは`Sendable`関数型の値。`@Sendable`は、関数型が`Sendable`プロトコルに準拠していることを示すために関数型に書き込むことができる。これは、その関数型の値を他の分離ドメインに渡して、キャプチャされた状態でもデータ競合を発生させることなく呼び出すことができることを意味する

### actor分離

#### actor型

- actorは、さまざまなタスクからアクセスできる状態を分離する方法を提供するが、これは協調的な方法でデータ競合を防ぐ
- actorは、独自の状態を保持する自己完結型。その状態にアクセスするには、actor上で実行されている必要がある

#### 相互排除

- actorは一度に1つのタスクのみを実行する。これにより、actor内部の状態への同時アクセスがなくなる。タスクが中断ポイントで解放されると別のタスクを実行できる

#### actor分離の維持

- タスクとactor間の相互作用は、`Sendable`以外の型が2つの間を通過しないようにすることで、両方の分離を維持する必要がある

#### actor自身の参照の分離

- actorは参照型だが、クラスとは異なり、同時アクセスを防ぐためにすべてのプロパティとコードを分離するため、別のConcurrencyドメインからそのactorへの参照を持つことは安全
- すべてのactorは暗黙的に`Sendable`
- actorのインスタンスプロパティは、そのactorに分離されている
- actorまたはactorのextensionのインスタンスメソッドもデフォルトで分離されている
- `reduce`アルゴリズムに渡されるクロージャなどの`Sendable`ではないクロージャは、そのactorのコンテキストにある場合、actor分離される
- タスクのイニシャライザはそのコンテキストからactorのコンテキストを継承するため、作成されたタスクは、タスクを作成したactor上にスケジューリングされる
- 分離タスク(detached task)は、作成されたコンテキストから完全に独立しているため、そのactorのコンテキストを継承しない

#### actorに分離されていない(Non-isolated)コード

- Non-isolatedコードは、どのactor上でも実行されないコード。`nonisolated`キーワードを使用することで、actor内にある関数を明示的にactorに分離されていないようにすることができる。actorに分離された状態の一部を読み取りたい場合は、`await`を使用して、必要な状態のコピーを取得する必要がある
- non-isolatedな`async`コードは常にグローバル協調プール(global cooperative pool)で実行される
- 大半のSwiftのコードは、同期的で、どのactorに対しても分離されておらず、指定されたパラメータにのみ依存して動作するため、呼び出されたConcurrencyドメインで実行される

|  呼び出し側 / 呼び出された側  |  non-isolated　+ 同期  |  non-isolated + 非同期 |
| ---- | ---- | ---- |
|  actor  |  actor上で実行  |  GCP上で実行  |
|  non-isolated + 非同期  |  GCP上で実行  |  GCP上で実行  |

GCP: global cooperative pool  

#### @MainActor

- main actorには、プログラムのUIに関連する多くの状態が含まれている
- main actorはmain threadを象徴し、UIの描画と操作がすべて行われる 
- actorなので、一度に1つのタスクしか実行しない
- **UIが応答しなくできなくなる可能性があるため、main actorに過度の処理や長時間の処理をかけないように注意**
- main actorへの分離は、`@MainActor`で示す
- この属性を関数またはクロージャに適用して、コードをmain actorで実行する必要があることを示すことができる
- Swiftコンパイラは、main actorで分離されたコードがmain threadでのみ実行されることを保証する

#### 型に@MainActorを付与する

- `@MainActor`は型に適用できる。その場合、それらの型のインスタンスはmain actorに分離される
- プロパティは、main actor上でのみでアクセスできる
-　明示的にオプトアウトしない限り、メソッドはmain actorに分離されている
- `@MainActor`のクラスは`Sendable`
- `ViewController`への参照をプログラム内の他のタスクやactorと共有でき、それらは非同期的に`ViewController`にコールバックして結果を渡すことができる
- 他のプログラムロジックは、共有状態とタスクを安全にモデル化する他のactorや、独立した処理を記述するタスクを使って、main actorから分離するべき

### 原子性（高レベルのデータ競合の防止）

- 原子性について高レベルで推論する必要がある
- actorは一度に1つのタスクのみを実行する。これにより、プログラムが確実に進行し、デッドロックの可能性が排除れる
- ただし、actorでの実行を停止すると、actorは他のタスクを実行できる。デッドロックを防ぐが、`await`の前後でactorの不変条件を慎重に検討する必要もある
- そうしないと、データ競合によってデータが破損しないとしても、プログラムが予期しない状態になる高レベルのデータ競合が発生する可能性はある
- たとえば、同じactorの状態にアクセスするためのメソッドに2つの`await`があり、これら2つの`await`間で内部の状態が変更されないと想定している。しかし、タスクが中断され、actorが他の優先度の高い処理を行う可能性がある。同じactor上の別のメソッドが最初の`await`で一時中断していたときに状態を変更した場合、2番目の`await`でその状態を取得したときに状態が変更されてしまっている
- メソッドは中断することなくactor上で実行されるため、actor上の同期コードとして作成するべき。そうすることで、関数全体を通してactorの状態は他の誰も変更できないようにできる
- ポイントはトランザクションとして考えること。インターリーブする可能性のある同期操作を特定し、非同期のactor操作をシンプルに保つべき

### 順序

- actorは、システム全体の応答性を維持するために、最初に最も優先度の高い処理を実行する。これにより、同じactorでより優先度の高い処理が行われる前に、優先度の低い処理が発生するという優先順位の逆転が排除される。**これは、厳密に先入れ先出し(FIFO)なserial `DispatchQueue`との大きな違い**
- タスクは最初から最後まで通常の制御フローで実行されるため、タスクは自然に処理を順序付けする
- `AsyncStream`は、`for-await-in`ループを使用してイベントのストリームを反復処理し、各イベントを順番に処理できる。`AsyncStream`は、任意の数のイベントプロデューサで共有できる。これにより、順序を維持しながらストリームに要素を追加できる

</details>
<br/>

## 例を用いた解説

Concurrencyの海は予測不可能であり、一度に多くのことが起こっているが、私たちが舵を取り、Swiftが海をナビゲートするのを手伝うことで、驚くべきものを生み出すことができる。

<img src="../images/Swift Concurrency WWDC22/concurrency_analogy.png" alt= "Concurrencyのイメージ" width="100%">

### タスクの分離

このコンセプトは、Swift Concurrencyの実行モデルの重要なアイデアの1つであり、データ競合を防ぐ。

私たちのConcurrencyの海では、タスクはボートとして表現される。ボートは私たちの主な労働者です。

彼らは
- 最初から最後まで順番に処理を実行する
- 非同期であり、コード内の`await`操作で何度でも作業を中断できる
- 自己完結型。各タスクは独自のリソースを保持し、海にいる他のすべてのボートとは独立して、単独で操作できる

### タスク間の通信

ボートが完全に独立している場合、データ競合がなくても同時並行に処理ができるが、お互いに通信する方法がないとあまり役に立たない。

たとえば、あるボートにパイナップルがあり、それを別のボートと共有したい場とする。そのために、ボートは海で合流し、パイナップルを一方のボートからもう一方のボートに移す。

<img src="../images/Swift Concurrency WWDC22/communication_between_tasks.png" alt= "タスク間通信" width="100%">

### 値型 - 値の変更はローカル上にしか影響がない

パイナップルをその重量と熟度を持つ構造体を定義する。

```swift
enum Ripeness {
    case hard
    case perfect
    case mushy(daysPast: Int)
}

struct Pineapple {
    var weight: Double
    var ripeness: Ripeness

    mutating func ripen() async { … }
    mutating func slice() -> Int { … }
}
```

ボートが海で出会うとき、私たちは実際にパイナップルのインスタンスのコピーをあるボートから次のボートに渡し、各ボートは独自のコピーを持つ。

<img src="../images/Swift Concurrency WWDC22/copy_pineapple.png" alt= "パイナップルのコピー" width="100%">

また、`slice`や`ripen`メソッドを呼び出すなどしてコピーを変更した場合、他のメソッドには影響しない。

<img src="../images/Swift Concurrency WWDC22/pineapple_no_global_effect.png" alt= "パイナップルの変更はグローバルな影響がない" width="100%">

Swiftは、まさにこの理由から常に値型を優先する。変更は局所的な影響しかない。

この原則は、値型が分離を維持するのに役立つ。

### 参照型 - 値の変更はグローバルに影響を与える

鶏を追加。食べるのだけに適しているパイナップルとは異なり、鶏は独自の個性を持つ美しい生き物。したがって、クラスを使用してそれらをモデル化する。

```swift
final class Chicken {
    let name: String
    var currentHunger: HungerLevel
    
    func feed() { … }
    func play() { … }
    func produce() -> Egg { … }
}
```

ボートが出会うと、鶏を共有する。ただし、鶏のような参照型をコピーすると、その特定のオブジェクトへの参照が得られる。

<img src="../images/Swift Concurrency WWDC22/share_chicken.png" alt= "share chicken" width="100%">

その後ボートが別々の方向に進み、ある問題が発生する。両方のボートが同時に作業を行っているが、両方が同じ鶏オブジェクトを参照しているため、独立していない。

その共有された可変データは、一方のボートが鶏に餌をやろうとしていて、もう一方のボートが鶏と遊んでみたい場合など、データ競合が発生しやすく、この1つの鶏は非常に混乱することになる。

<img src="../images/Swift Concurrency WWDC22/chicken_data_race.png" alt= "chicken　data race" width="100%">

これを防ぐためには、鶏が誤ってあるボートから別のボートに渡されないように、Swiftコンパイラでいくつかのチェックを行う必要がある。

### `Sendable`プロトコル

`Sendable`プロトコルは、データ競合を作成せずに異なるConcurrencyドメインで安全に共有できる型を記述するために使用される。

### タスクの境界を越えた`Sendable`チェック

パイナップル構造体は、値型であるため`Sendable`に準拠しているが、鶏クラスはに内部で同期されない参照型であるため`Sendable`にはできない。

```swift
struct Pineapple: Sendable { … } // 値型であるため、Sendableに準拠する
class Chicken: Sendable { } // 内部で同期されない参照型であるため、Sendableに準拠することはできない
```

プロトコルとして`Sendable`をモデリングすることにより、データがConcurrencyドメイン全体で共有される場所を示すことができる。

私たちはタスクから鶏を返そうとする。これは、鶏が`Sendable`ではないため、安全ではないことを示すエラーが出力される。

```swift
// 鶏が`Sendable`ではないため、エラーが発生する
let petAdoption = Task {
    let chickens = await hatchNewFlock()
    return chickens.randomElement()!
}
let pet = await petAdoption.value
```

### タスクAPIの`Sendable`制約

実際の`Sendable`制約は、`Task`構造体の定義から得られる。`Success`と呼ばれるタスクの結果型は、`Sendable`プロトコルに準拠する必要がある。

```swift
struct Task<Success: Sendable, Failure: Error> {
    var value: Success {
        get async throws { … }
    }
}
```

ここでは、異なるConcurrencyドメインに値が渡されるジェネリックパラメータが存在する。

### タイプの`Sendable`チェック

2隻のボートが海で会ってデータを共有したいとき、私たちはそれらが安全であることを確認するためにすべての商品を一貫してチェックする必要がある。

<img src="../images/Swift Concurrency WWDC22/share_data.png" alt= "share data" width="100%">

Swiftコンパイラは、`Sendable`型のみが交換されることを確認する。

コンパイラは、多くの異なるポイントで`Sendable`の正しさをチェックしている。`Sendable`型は構築時に正しいものでなければならず、共有データをそれらを通して密輸することはできない。

列挙と構造体は一般に値型を定義する。この型は、すべてのインスタンスデータをコピーして、独立した値を生成する。したがって、すべてのインスタンスデータが`Sendable`である限り、それらは`Sendable`になる。

```swift
enum Ripeness: Sendable {
    case hard
    case perfect
    case mushy(daysPast: Int)
}

struct Pineapple: Sendable {
    var weight: Double
    var ripeness: Ripeness
}
```

`Sendable`は、条件付き準拠(Conditional Conformance)を使用して、コレクションやその他のジェネリック型を通じて伝播できる。

```swift
// Sendableな型の配列が含まれているため、`Sendable`
struct Crate: Sendable {
    var pineapples: [Pineapple]
}
```

`Sendable`型の配列は`Sendable`であるため、パイナップルでいっぱいの木枠も`Sendable`。

これらのnon-public型の`Sendable`の準拠はすべて、Swiftコンパイラによって推論できるため、熟度、パイナップル、およびクレートはすべて暗黙的に`Sendable`。

しかし、

```swift
// Sendable構造体のCoopの保存されたプロパティflockがSendableではない[Chicken]を保持している
struct Coop: Sendable {
    var flock: [Chicken]
}
```

この型は`Sendable`としてマークすることはできない。なぜなら、鶏は`Sendable`ではない状態を含んでいるため。鶏は`Sendable`ではないため、鶏の配列は`Sendable`ではない。

クラスは参照型であるため、`final`クラスに不変のストレージのみがある場合など、非常に狭い状況でのみ`Sendable`にすることができる。

```swift
// finalクラスに不変のストレージがある場合はSendableにできる
final class Chicken: Sendable {
    let name: String
    var currentHunger: HungerLevel //currentHungerは可変のためChickenSendableにできない
}
```

`Chicken`クラスを`Sendable`にしようとすると、可変状態が含まれているため、エラーが発生する。

代わりに、たとえば、ロックを一貫して使用することにより、独自の内部同期を実行する参照型を実装することができる。

```swift
// @uncheckedを使用できますが、注意！
class ConcurrentCache<Key: Hashable & Sendable, Value: Sendable>: @unchecked Sendable {
    var lock: NSLock
    var storage: [Key: Value]
}
```

これらの型は概念的に`Sendable`ですが、Swiftがそれについて推論する方法はない。

`@unchecked Sendable`を使用して、コンパイラのチェックを無効にすることができる。

これは注意。なぜなら、`@unchecked Sendable`を介して可変状態を"密輸"しているため、Swiftが提供しているデータ競合の安全保証を損なっている。

### タスク作成中の`Sendable`チェック

タスクの作成には、ボートから手漕ぎボートを送るなど、新しい独立したタスクでクロージャを実行することが含まれる。

これを行うと、元のタスクから値をキャプチャして新しいタスクに渡すことができる。そのため、データ競合を導入しないように`Sendable`チェックが必要。

この境界全体で`Sendable`ではない型を共有しようとする場合、Swiftコンパイラはエラーメッセージを作成する。

```swift
let lily = Chicken(name: "Lily")
Task.detached {
	lily.feed() // `Sendable`ではない型の鶏をcaptureしている
}
```

上記では、クロージャは`Sendable`クロージャであると推論されており、下記のように、`@Sendable`が明示的に書かれている場合もある。

```swift
let lily = Chicken(name: "Lily")
Task.detached { @Sendable in
	lily.feed()
}
```

### `Sendable`関数型

`Sendable`クロージャは、`Sendable`関数型の値。

`@Sendable`は、関数型が`Sendable`プロトコルに準拠していることを示すために、関数型に記述できる。

```swift
struct Task<Success: Sendable, Failure: Error> {
    static func detached(
        priority: TaskPriority? = nil,
        operation: @Sendable @escaping () async throws -> Success
    ) -> Task<Success, Failure>
}
```

これは、その関数型の値を他のConcurrencyドメインに渡し、captureされた状態でデータ競合を導入することなく呼び出せることを意味する。

通常、関数型はプロトコルに準拠することはできないが、`Sendable`はコンパイラがセマンティック要件を検証するため特別である。

`Sendable`プロトコルに準拠した`Sendable`型のタプルも同様のサポートがあり、言語全体で`Sendable`を使用できる。

## actor分離

`Sendable`プロトコルは、タスク間で安全に共有できる型を示し、Swiftコンパイラはタスクの分離を維持するためにあらゆるレベルで`Sendable`の準拠をチェックする。

ただし、どこにも共有の可変データがなければ、タスクが意味のある方法で協調することは困難である。

そのため、データ競合を防ぎ、タスクの間でデータを共有する方法が必要。これがactorがやっていること。

### actor型

actorは、さまざまなタスクからアクセスできる状態を、分離してデータ競合を排除するように調整した方法で提供される。

actorは、私たちのConcurrencyの海の島だと見なせる。

<img src="../images/Swift Concurrency WWDC22/actor_island.png" alt= "actor island" width="100%">


```swift
actor Island {
    var flock: [Chicken]
    var food: [Pineapple]

    func advanceTime()
}
```

ボートのように、各島は自己完結型で、海の他のすべてから分離された独自の状態がある。

その状態にアクセスするには、私たちのコードは島の上で実行される必要がある。

### actorは相互排除

一度にコードを実行するために島を訪れることができるボートは1つだけ。これにより、島の州への同時アクセスがないことが保証される。

<img src="../images/Swift Concurrency WWDC22/actor_boat.png" alt= "actor must run on its island" width="100%">

```swift
func nextRound(islands: [Island]) async {
    for island in islands {
        await island.advanceTime()
    }
}
```

島が再び解放されると、別のボートが訪れることができる。

ボートと島の間の相互作用は、`Sendable`ではない型が2つの間を通過しないことを確認することにより、両方の分離を維持する必要がある。

たとえば、ボートから島の群れに鶏を追加すると、異なるConcurrencyドメインから同じ鶏への2つの参照が作成されるため、Swiftコンパイラはそれを拒否する。

<img src="../images/Swift Concurrency WWDC22/actors_try_to_copy_reference.png" alt= "actors try to copy reference" width="100%">

```swift
// 共有できない
❌ await myIsland.addToFlock(myChicken)
```

同様に、島のチキンをペットとして養うためにボートで奪おうとすると、`Sendable`チェックは、このデータ競合が作成できないことを保証する。

<img src="../images/Swift Concurrency WWDC22/actors_try_to_take_away_reference.png" alt= "actors try to take away reference" width="100%">

```swift
// 共有できない
❌ myChicken = await myIsland.adoptPet()
```

### actorは参照分離

actorは参照型だが、クラスとは異なり、すべてのプロパティとコードを分離して同時アクセスを防ぐため、異なるConcurrencyドメインからactorへの参照を持つことは安全。したがって、すべてのactor型は暗黙的に`Sendable`。

これは島への地図を持っているようなもの。地図を使用して島を訪れることができるが、州にアクセスするにはドッキングする手順を経る必要がある。

<img src="../images/Swift Concurrency WWDC22/actor_isolated_internal_mutable_state.png" alt= "actor isolated internal mutable state" width="100%">

actorの分離は、私たちがいるコンテキストによって決定される。

 - actorのインスタンスプロパティはそのactorに分離される
 - actorのインスタンスメソッドまたはactorのextensionもデフォルトで分離される
 - `reduce`アルゴリズムに渡されたクロージャなど、`Sendable`ではないクロージャは、actorのコンテキストに留まり、actorに分離されたコンテキストにあるときはactorに分離される
 - タスクのイニシャライザはまた、actorのコンテキストを継承するため、作成されたタスクはタスクを開始したactorにスケジュールされる
 - 独立したタスクは、作成されたコンテキストとは完全に独立しているため、actorのコンテキストを継承しない

```swift
actor Island {
    var flock: [Chicken] // ⭕️ 分離
    var food: [Pineapple] // ⭕️ 分離

    func advanceTime() { // ⭕️ 分離
        let totalSlices = food.indices.reduce(0) { (total, nextIndex) in
            total + food[nextIndex].slice()
        }

        Task { // ⭕️ 分離
            flock.map(Chicken.produce)
        }

        Task.detached { // ❌ 分離されていない
            let ripePineapples = await food.filter { $0.ripeness == .perfect }
            print("There are \(ripePineapples.count) ripe pineapples on the island")
        }
    }
}
```

分離されたタスクのクロージャのコードは、孤立した`food`プロパティを参照するために`await`する必要があるため、actorの外側にあると見なされていることがわかる。

これは非分離(non-isolatedな)コードです。非分離コードは、どのactorでも実行されないコード。`nonisolated`キーワードを使用して、actorの外に置くことにより、actor内にない関数を明示的に作成できる。

### actorに分離されていないasyncコード

```swift
extension Island {
    nonisolated func meetTheFlock() async {
        let flockNames = await flock.map { $0.name }
        print("Meet our fabulous flock: \(flockNames)")
    }
}
```

actorに分離されていない**async**コードは、常にグローバルな協調スレッドプールで実行される。

ボートが海に出ているときだけ実行されると考えて欲しい。そのため、訪れている島を出て処理をする必要がある。

それは、私たちが私たちと一緒に`Sendable`ではないデータを取得していないことを確認するためにチェックすることを意味する。コンパイラは、潜在的なデータ競合を検出する。

上記のコードでは、`Sendable`ではない鶏のインスタンスが島を出ようとしており潜在的なデータ競合を検出している。

```swift
    nonisolated func meetTheFlock() async {
        // ❌ ChickenはSendableに準拠していない
        let flockNames = await flock.map { $0.name }
    }
```

### actorに分離されていないsyncコード

`greet`操作は、actorに分離されていないsyncコード。それは、ボートや島、または一般的なConcurrencyについては何も知らない。

```swift
func greet(_ friend: Chicken) { }

extension Island {
    func greetOne() {
        if let friend = flock.randomElement() { 
            greet(friend)
        }
    }
}
```

actorに分離された`greetOne`関数からそれを呼ぶことができる。

このsyncコードは、島から呼び出されると島にとどまるので、群れから鶏を自由に動作させることができる。

代わりに、非分離のasync関数で`greet`と呼ばれると、それは海のボート上で実行される。

```swift
func greet(_ friend: Chicken) { }

func greetAny(flock: [Chicken]) async {
    if let friend = flock.randomElement() { 
        greet(friend)
    }
}
```

ほとんどのSwiftコードは次のようになる: syncコードは、あらゆるactorに非分離で、指定された引数のみで動作するため、呼び出されたConcurrencyドメインにとどまる。

### actorのまとめ

 - 各actorインスタンスは、プログラム内の他のすべてから分離されている
 - actorで一度に実行できるタスクは1つだけ
 - `Sendable`チェックは、actorを出入りするときはいつでも発生する
 - actor自身は`Sendable`

## @MainActor

main actorを海の真ん中にある大きな島と考えて欲しい。main actorは「大きい」。つまり、プログラムのUIに関連する多くの状態が含まれているということ。

<img src="../images/Swift Concurrency WWDC22/main_actor_big_island.png" alt= "main actor big island" width="100%">

これは、UIのすべての図面とインタラクションが発生するmain threadを表現する。UIフレームワークとアプリの両方に、それを実行する必要があるコードがたくさんある。しかし、それはactorなので、一度に1つの仕事しか実行されない。

UIが応答不可能になるため、main actorにあまりにも多くの処理や長期にわたる処理をさせないように注意。

<img src="../images/Swift Concurrency WWDC22/main_actor_run_only_one.png" alt= "main actor run only one" width="100%">

### @MainActorの関数とクロージャ

main actorへの分離は、`@MainActor`で表現される。

```swift
@MainActor func updateView() { … }

Task { @MainActor in
    // …
    view.selectedChicken = lily
}

nonisolated func computeAndUpdate() async {
    computeNewValues()
    await updateView()
}
```

この属性は、関数またはクロージャに適用して、main actorでコードを実行する必要があることを示す。Swiftコンパイラは、main actorのコードがmain threadでのみ実行されることを保証する。

### @MainActorの型

@MainActorは型にも適用できる。その場合、これらの型のインスタンスはmain actorに分離されます。

```swift
@MainActor
class ChickenValley {
    var flock: [Chicken]
    var food: [Pineapple]

    func advanceTime() {
        for chicken in flock {
        chicken.eat(from: &food)
        }
    }
}
```

 - main actorでのみプロパティにアクセスできる
 - メソッドは、明示的にオプトアウトしない限り、main actorに分離される
 - main actorのクラスは`Sendable`

私たちのプログラムの他のタスクやactorと、`View Controller`への参照を共有することができ、結果を`View Controller`に非同期に呼び戻すことができまる。私たちのアプリでは、私たちの`View`と`View Controller`がmain actor上で実行される。他のプログラムロジックは、共有状態とタスクを安全にモデル化するために他のactorを使用して、独立した作業を記述するためにタスクを使用して、main actorから分離する必要がある。

<img src="../images/Swift Concurrency WWDC22/main_actor_and_other_actors.png" alt= "main actor and other actors" width="100%">

## 原子性（高レベルのデータ競合の防止）

Swift Concurrencyモデルの目標は、データ競合を排除すること。それが本当に意味することは、それがデータの破損を伴う低レベルのデータ競合を排除すること。

私たちはまだ高レベルの原子性について推論する必要がある。

actorは一度に1つのタスクのみを実行します。ただし、actorは実行をやめると、他のタスクを実行できます。これにより、プログラムが進み、デッドロックの可能性を排除することが保証される。ただし、`await`文をめぐるactorの不変性を慎重に考慮する必要がある。そうしなければ、データが実際に破損していないとしても、高レベルのデータ競合でプログラムが予期しない状態になる可能性がある。

### 非トランザクションコード

島に追加のパイナップルを堆積させる関数がある。actorの外にあるため、非分離のasyncコードで、海で実行される。

```swift
func deposit(pineapples: [Pineapple], onto island: Island) async {
    var food = await island.food
    food += pineapples
    await island.food = food
}
```

パイナップルとパイナップルを堆積させる島にいくつかのパイナップルと地図が与えられている。

最初の操作で、島からfood配列のコピーを取得する。

<img src="../images/Swift Concurrency WWDC22/visit_and_copy_pineapples.png" alt= "visit and copy pineapples" width="100%">

ボートは、`await`キーワードでシグナルを受け取った島を訪れる必要がある。

`food`のコピーを受け取るとすぐに、ボートは仕事を続けるために海に向かう。

つまり、`pineapples`引数からパイナップルを島から得た2つのパイナップルに追加するということ。

<img src="../images/Swift Concurrency WWDC22/add_pineapples.png" alt= "add_pineapples" width="100%">

そのまま関数の最後の行に進む。私たちのボートは、島の`food`配列にこれらの3つのパイナップルに設定するために、再び島を訪れる必要がある。

<img src="../images/Swift Concurrency WWDC22/visit_island_again.png" alt= "visit island again" width="100%">

すべてがうまくいった。しかし、物事は少し異なっていたかもしれない。

たとえば、私たちの最初のボートが島を訪れる順番を待っている間、海賊船が忍び込んですべてのパイナップルを盗んだとする。

```swift
await island.food.takeAll()
```

<img src="../images/Swift Concurrency WWDC22/pirate_take_pineapples_simultaneously.png" alt= "pirate takes pineapples simultaneously" width="100%">

そして私たちの元の船は、島に3つのパイナップルを堆積させる。3つのパイナップルが突然5つのパイナップルに変わった。

何が起きたのか？

ここでは同じactorの状態へのアクセスを待機するために2つの`await`がある。そして、ここでは、島の`foof`配列はこれらの2つの`await`の間で変わらないと想定している。

しかし、これらは待機している。つまり、私たちの仕事はここで中断され、actorは海賊との戦いのように他のより優先順位の高い仕事をすることができる。

この特定のケースでは、Swiftコンパイラは、別のactorの状態を変更する試みを完全に拒否する。

しかし、実際のところ、actorのsyncコードに`deposit`関数を書き直す必要がある。

```swift
extension Island {
    func deposit(pineapples: [Pineapple]) {
        var food = self.food
        food += pineapples
        self.food = food
    }
}
```

これはsyncコードであり、中断することなくactorで実行される。したがって、島の状態は、機能全体を通して他の誰にも変更されない。

actorを書いているときは、何らかの形でインターリーブできるsyncのトランザクション操作の観点から考えること。

それらのすべては、actorから出るときに妥当な状態にあることを保証する必要がある。

actorのasync操作の場合、それらをシンプルに保ち、主にsyncのトランザクション操作オペレーションから形成し、actorが待機する可能性あることに注意。

これにより、actorを最大限に活用して、低レベルと高レベルのデータ競合の両方を排除できる。

## 順序

Concurrencyプログラムでは、多くのことが一度に起こっているため、それらのことが起こる順序は異なる場合がある。

プログラムは、多くの場合、一貫した順序のイベントの処理に依存している。(例: ユーザー入力、サーバからのメッセージ)

Swift Concurrencyは、オペレーションを順序づけするためのツールを提供するが、actorはそのツールではない。

### actorは厳密にFIFOではない

actorは最初に最高優先順位の作業を実行し、システム全体が対応するのをサポートする。これにより、優先順位が低い作業がより優先順位の高い作業よりも先に行われる、優先順位の逆転がなくなる。

これは、serial `DispatchQueue`との大きな違いであり、これは厳密に最初に入ったものが最初に実行される(FIFO)。

### 順序づけのためのツール

Swift Concurrencyには、処理を順序づけするためのいくつかのツールがある。

#### タスクは順番にコードを実行する

タスクは最初から最後まで実行され、通常の制御フローが使用されるため、自然に処理を順序づけする。

#### `AsyncStream`は、要素を順番に配信する

`AsyncStream`を使用して、実際のイベントストリームをモデル化できる。

1つのタスクは、`for-await-in`ループでイベントのストリームを反復し、各イベントを順番に処理できる。

```swift
for await event in eventStream {
    await process(event)
}
```

`AsyncStream`は、任意の数のイベントプロデューサと共有できる。これにより、順序を維持しながら、ストリームに要素を追加できる。

## Strict Concurrency Checking(Swift6へのマイグレーション)

Swift Concurrencyモデルが、分離の概念を使用してデータ競合を排除するように設計されていることについて多くのことを見てきた。これは、タスクとactorの境界で`Sendable`チェックによって維持される。

ただし、あらゆるところのすべての型に`Sendable`をマークするために、今行っているを止めることはできない。代わりに、インクリメンタルなアプローチが必要。そこで、Swift 5.7で、Swiftコンパイラが`Sendable`を厳密にチェックする方法を指定するためのビルド設定を導入する。

### Minimal mode

コンパイラは、`Sendable`として何かを明示的にマークした場所のみを診断する。これは、Swift 5.5と5.6の動作に似ており、上記では警告やエラーはない。

たとえば、`FarmAnimals`モジュールには`Chicken`クラスと`Coop`があり、他のモジュールには`Chicken`の配列がある。

`Chicken`が`Sendable`でない場合、`Coop`が警告を受ける。

```swift
// Module: FarmAnimals

public final class Chicken { // Sendableではない
    let name: String
    init(name: String) {
        self.name = name
    }
}

import FarmAnimals

struct Coop {
    var flock: [Chicken]
}
```

さて、`Sendable`を追加すると、コンパイラは`Coop`型が`Sendable`ではないと警告する。

```swift
import FarmAnimals

struct Coop: Sendable { // 👈🏻 SendableチェックをオンにするためにSendableの追加が必要
    var flock: [Chicken] // ⚠️Stored property 'flock' of 'Sendable'-conforming struct 'Coop' has non-sendable type '[Chicken]'
}
```

`Sendable`関連の問題は、問題を1つずつ簡単に処理できるようにするために、Swift 5系ではエラーではなく警告として提示される。

### Targeted mode

この設定により、async/await、Task、actorなどの迅速なConcurrency機能を既に導入しているコードを`Sendable`チェックできる。

たとえば、このモードだと、新しく作成されたタスクで`Sendable`な型の値をキャプチャする試みが診断される。

```swift
import FarmAnimals

func visit(coop: Coop) async {
    guard let favorite = coop.flock.randomElement() else {
        return
    }

    Task {
        favorite.play() // ⚠️Capture of 'favorite' with non-sendable type 'Chicken' in a `@Sendable` closure
    }
}
```

おそらく、`Sendable`にまだ更新されていないパッケージ、または私たちの独自のモジュールの場合もあるかもしれない。

それらのために、`@preconcurrency`を使用して、そのモジュールから生じる型の`Sendable`の警告を一時的に無効にすることができる。

```swift
@preconcurrency import FarmAnimals // 👈🏻 @preconcurrencyを追加

func visit(coop: Coop) async {
    guard let favorite = coop.flock.randomElement() else {
        return
    }

    Task {
        favorite.play() // 警告なし
    }
}
```

これにより、このソースファイル内の`Chicken`型の`Sendable`警告が抑制される。

ある時点で`FarmAnimals`モジュールが`Sendable`を導入すると、次の2つの内の1つが発生します。

1. `Chicken`は何かしら`Sendable`になる

```swift
// Module: FarmAnimals

public final class Chicken: Sendable { ... }
```

```swift
@preconcurrency import FarmAnimals // ⚠️'@preconcurrency' attribute on module 'FarmAnimals' is unused
```

2. `Chicken`は容認できないことが判明した場合、警告が出力される

```swift
// Module: FarmAnimals

public final class Chicken: Sendable { ... }
```

```swift
@preconcurrency import FarmAnimals
func visit(coop: Coop) async {
    Task {
        favorite.play() // ⚠️Capture of 'favorite' with non-sendable type 'Chicken' in a `@Sendable` closure
    }
}
```

Targeted modeは、既存のコードとの互換性と潜在的なデータ競合の判定とのバランスをとることを試みる。

### Complete mode

データ競合を完全に排除するために意図されたSwift 6のセマンティクスに近い状態にする。以前の2つのモードがチェックするすべてのものをチェックに加え、モジュール内のすべてのコードに対してもチェックが行われる。

たとえば、以下のコードは`DispatchQueue`で処理を同時並行に実行している。

```swift
func doWork(_ body: @escaping () -> Void) {
    DispatchQueue.global().async {
        body() // ⚠️Capture of 'body' with non-sendable type '() -> Void' in a `@Sendable` closure
    }
}

func visit(friend: Chicken) {
    doWork {
        friend.play()
    }
}
```

`DispatchQueue`のasync操作は、実際に`Sendable`クロージャであることがわかっているため、コンパイラは、`DispatchQueue`で実行されているコードから`body`がキャプチャされたときにデータ競合があることを示す警告を生成する。

`body`引数を`Sendable`にすることでこれを修正できる。この修正で警告は排除され、`doWork`のすべての呼び出しは、`Sendable`クロージャを提供する必要があることがわかる。

```swift
func doWork(_ body: @Sendable @escaping () -> Void) {
    DispatchQueue.global().async {
        body() // 警告なし
    }
}

func visit(friend: Chicken) {
    doWork {
        friend.play() // Capture of 'friend' with non-sendable type 'Chicken' in a `@Sendable` closure
    }
}
```

完全なチェックは、プログラムの潜在的なデータ競合を全て洗い出すのに役立る。Swiftのデータ競合を排除するという目標を達成するには、最終的にはチェックをCompleteにする必要がある。

### 推奨

- 目標に向けて段階的に行う: Swift Concurrencyモデルを採用して、データ競合を防ぐためにアプリを設計し、次に徐々により厳格なConcurrencyチェックを可能にして、コードからのエラーになるクラスを排除する

- importされた型の警告を抑制するために`@preconcurrency`をマークすることをためらう必要はない。これらのモジュールがより厳格なConcurrencyチェックを採用すると、コンパイラは私たちの仮定を再チェックしてくれる

# Swift Async Algorithm

`AsyncSequence`を使いこなしてSwift Concurrencyをもっと活用する

詳細はこちら[Swift Async Algorithms](./Swift%20Async%20Algorithms.md)

# References

- [Visualize and optimize Swift concurrency](https://developer.apple.com/videos/play/wwdc2022/110350/)
- [Eliminate data races using Swift Concurrency](https://developer.apple.com/videos/play/wwdc2022/110351/)
- [Meet Swift Async Algorithms](https://developer.apple.com/videos/play/wwdc2022/110355/)

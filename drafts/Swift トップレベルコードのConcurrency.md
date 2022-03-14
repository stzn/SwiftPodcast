# Swift トップレベルコードのConcurrency

- [Swift トップレベルコードのConcurrency](#swift-トップレベルコードのconcurrency)
  - [概要](#概要)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決策](#解決策)
    - [詳細](#詳細)
      - [トップレベルのコードのasyncコンテキストの推論](#トップレベルのコードのasyncコンテキストの推論)
      - [変数](#変数)
    - [既存のコードへの影響](#既存のコードへの影響)
    - [ABI stabilityへの影響](#abi-stabilityへの影響)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [関連PR](#関連pr)

## 概要

Swift Concurrencyの機能をトップレベルのコードでも使えるようにする。特に、トップレベルの変数がどうデータ競合から守られ、どうやってsyncコンテキストからasyncコンテキストに入るのかに注力する。

## 内容

### 問題点

トップレベルのコード宣言のコンテキストは他のスペースとは異なって機能する。トップレベルにConcurrencyの機能を追加する場合、これまで上がっていなかったいくつかの疑問が生じる。

トップレベルコードの変数はグローバルとローカルの中間のように振る舞う。モジュール内ではグローバル変数としてモジュール全体でアクセス可能だが、ローカル変数のように順番に初期化される。グローバル変数はConcurrencyでは危険である。どこにも隔離されている保証がないため、競合状態の影響を受ける。

トップレベルのコードは、機能を安全に試せたり、あると嬉しい小さいscriptを書くことなどが意図されているが、Concurrencyの場合はそう単純ではない。

この変数のおかしくて危険な振る舞いに加え、コンテキストがsyncかasyncかによって関数のオーバーロードの解決方法に影響がある。単純な切り替えだけをすると、隠れた嫌なセマンティクス上の変更が入り、既存のスクリプトを壊してしまう可能性が潜んでいる。

### 解決策

asyncのコンテキストにいる時のみ、トップレベルのコードでConcurrencyを使えるようにする。syncコンテキストにいる時は、トップレベルのコードに変更は加えない。トップレベルのコードをasyncコンテキストで実行するには、`await`をトップレベルの式の一つで使う。

関数宣言やクロージャ内部で使われた`await`はこれに該当しない。

```swift
func doAsyncStuff() async {
  // ...
}

let countCall = 0

let myClosure = {
  await doAsyncStuff() // これはトップレベルの`await`ではない
  countCall += 1
}

await myClosure() // これはトップレベルの`await`
```

トップレベルのグローバル変数は、データ競合を防ぐために暗黙的に`@MainActor`が付与される。ソース破壊を防ぐために、Swift6までは暗黙的に`@preconcurrency`が付与される。

```swift
var a = 10

func bar() {
    print(a)
}

bar()

await something() // トップレベルのコードはasyncコンテキストになる
```

Swift6以降、完全なactor-isolationチェックが行われる。`bar`内の`a`の利用は`bar`が`@MainActor`で隔離されていないため、エラーになる。Swift5系では、エラーなしにコンパイルできる。

### 詳細

#### トップレベルのコードのasyncコンテキストの推論

トップレベルのコードがasyncコンテキストかどうかの推論は[SE-0296 Async/Await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md#closures)の匿名のクロージャの推論と同じルールになる。

トップレベルのコードは、即座に実行されるトップレベルのコンテキスト内で中断点(suspension point)があればasyncコンテキストと推論される。

```swift
func theAnswer() async -> Int { 42 }

async let a = theAnswer() // 暗黙的にawait。トップレベルはasync

await theAnswer() // 明示的にawait。トップレベルはasync

let numbers = AsyncStream(Int.self) { continuation in
    Task {
        for number in 0 .. < 10 {
            continuation.yield(number)
        }
        continuation.finish()
    }
}

for await number in numbers { // 明示的にawait。トップレベルはasync
    print(number)
}
```

上記の例は、それぞれの中断点が、トップレベルのコードをasyncコンテキストにするきっかけとなっていることを示している。`async let a = theAnswer()`は暗黙的で、`await theAnswer()`と`for await number in numbers`は明示的。

ただし、このトップレベルのコードのasyncの推論は、これらのコンテキストは別々にsyncまたはasyncのコンテキストになることができるため、関数やクロージャの本文にまで波及しない。

```swift
func theAnswer() async -> Int { 42 }

let closure1 = { @MainActor in print(42) }
let closure2 = { () async -> Int in await theAnswer() }
```

上記の例は、トップレベルのコードは中断点がないためasyncコンテキストではない。

クロージャ本文がasyncコンテキストかどうかは`FindInnerAsync` ASTWalkerの内部で推論されている。最小限の労力で`FindInnerAsync`はトップレベルのコードの本文も扱えるように一般化することができ、トップレベルのコードとクロージャ本文のasyncの探索を並列で行う。

#### 変数

トップレベルコードの変数は、ローカル変数のように順番に初期化されるが、グローバルスコープ内にあり、グローバル変数として扱われる。データの競合を防ぐために、変数は暗黙的にMain Actorに分離する必要がある。ただし、全てのトップレベルの変数へのアクセスで`await`する必要があるとしたら、それは残念。幸い、他のエントリポイントと同様に、トップレベルコードはメインスレッドで実行されるため、変数に直接アクセスして変更できるように、トップレベルのコードのスペースを暗黙的にMain Actorから分離することができます。しかし、これはまだソースを壊してしまう。変数がMain Actorである場合、関数はMain Actorに分離されていないため、トップレベルのコードのsyncのグローバル関数はエラーを出力する。潜在的なデータ競合があることを示す診断は正しいが、ソースを壊してしまうことも残念。ソース破壊を軽減するために、変数には暗黙的に`@preconcurrency`属性のアノテーションが付けられる。この属性はSwift5コードにのみ適用され、言語モードがSwift 6に更新されると、これらのデータ競合はエラーになる。

`-warn-concurrency`がコンパイラに渡され、トップレベルのコードで`await`がある場合、警告は、他のasyncコンテキストの場合と同様に、Swift5でもエラーになる。`await`がなく、このフラグが渡された場合、トップレベルがasyncコンテキストでなくても、変数はMain Actorによって暗黙的に保護され、Concurrencyチェックが厳密に実行される。トップレベルはasyncコンテキストではないため、実行ループは暗黙的に作成されず、オーバーロード解決の振る舞いは変更されない。

要約すると、トップレベルの変数宣言は、データ競合の安全性とソース破壊の削減のバランスをとるために、`@MainActor @preconcurrency`で宣言されたかのように振る舞う。

グローバル変数の挙動に戻ると、指摘しなければならない追加の設計の詳細がいくつかある。

トップレベルの変数でGlobal Actorを明示的に指定する機能を削除することを提案したい。トップレベルの変数は、グローバル変数とローカル変数のハイブリッドのように扱われ、厄介な結果が生じる。変数はグローバルスコープで宣言されているため、どこでも使用できると見なされる。これにより、次の例のように、メモリの安全性に関する厄介な問題が発生する:

```swift
print(a)
let a = 10
```

この例は、実行時に"0"を出力する。`a`はグローバル変数であるため、`print`で使用できるが、初期化は順次行われるため、まだ初期化されていない。整数型およびその他のプリミティブは、暗黙的に0で初期化される。ただし、クラスは参照型であり、ゼロに初期化されるため、変数がクラス型の場合、segmentation faultが発生する。

最終的には、この穴をメモリモデルで補修したい。そのため、トップレベルの変数を暗黙の`main`関数のローカル変数にする。明示的なGlobal Actorを禁止して、ソース破壊を減らす。

### 既存のコードへの影響

トップレベルはasyncコンテキストではないため、現在、`await`をトップレベルコードで使うことはできない。このプロポーザルの機能は、トップレベルの`await`の存在によって可能になるため、このプロポーザルのscriptはない。

### ABI stabilityへの影響

シグネチャは変わらず影響なし。

## 参考リンク

### Forums

- [Concurrency in Top-Level Code](https://forums.swift.org/t/concurrency-in-top-level-code/55001)

### プロポーザルドキュメント

- [Concurrency in Top-Level Code](https://github.com/apple/swift-evolution/pull/1542)

### 関連PR

- [Fix top-level global-actor isolation crash](https://github.com/apple/swift/pull/40963)
- [Make top-level code variables @MainActor @preconcurrency](https://github.com/apple/swift/pull/40998)
- [Concurrent top-level detection](https://github.com/apple/swift/pull/41061)
- [SE-0343: De-experimentalify async top-level context](https://github.com/apple/swift/pull/41805)

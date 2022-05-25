# Swift トップレベルコードのConcurrency

収録日: 2022/05/22

- [Swift トップレベルコードのConcurrency](#swift-トップレベルコードのconcurrency)
  - [概要](#概要)
  - [用語](#用語)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決策](#解決策)
    - [詳細](#詳細)
      - [トップレベルコードのasyncコンテキストの推論](#トップレベルコードのasyncコンテキストの推論)
      - [変数](#変数)
    - [既存のコードへの影響](#既存のコードへの影響)
    - [ABI stabilityへの影響](#abi-stabilityへの影響)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [関連PR](#関連pr)

## 概要

Swift Concurrencyの機能をトップレベルコードでも使えるようにする。特に、トップレベルの変数をどうデータ競合から守り、どうやってsyncコンテキストからasyncコンテキストへ切り替えるかに注力する。

## 用語

- 分離: `actor`によってデータ競合から守られている(他から分離されている)状態

## 内容

### 問題点

トップレベルのコード宣言のコンテキストは他のスペースとは異なる。トップレベルにConcurrencyの機能を追加する場合、これまで上がっていなかったいくつかの疑問が生じる。

トップレベルコードの変数はグローバルとローカルの中間のように振る舞う。モジュール内ではグローバル変数としてモジュール全体でアクセス可能だが、ローカル変数のように順番に初期化される。グローバル変数は隔離されている保証がどこにもないため、Concurrencyで利用する場合にデータ競合が発生する危険がある。

トップレベルコードは、機能を安全に試せたり、あると嬉しい小さいscriptを書くことなどが意図されているが、Concurrencyの場合はそう単純ではない。

この変数の危険な振る舞いに加え、コンテキストがsyncかasyncかによって関数のオーバーロードの解決方法にも影響がある。単純な切り替えだけをすると、隠れた良くないセマンティクス上の変更が入り、既存のスクリプトを壊してしまう可能性が潜んでいる。

### 解決策

asyncのコンテキストにいる時のみ、トップレベルコードでConcurrencyを使えるようにする。syncコンテキストにいる時は、トップレベルコードに変更は加えない。トップレベルコードをasyncコンテキストで実行するには、`await`をトップレベルの式の一つで使う。

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

await something() // トップレベルコードはasyncコンテキストになる
```

Swift6以降、完全なactorへの分離チェックが行われる。`bar`内の`a`の利用は`bar`が`@MainActor`で分離されていないため、エラーになる。Swift5系では、エラーなしにコンパイルできる。

### 詳細

#### トップレベルコードのasyncコンテキストの推論

トップレベルコードがasyncコンテキストかどうかの推論は[SE-0296 Async/Await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md#closures)の匿名のクロージャの推論と同じルールになる。

トップレベルコードは、即座に実行されるトップレベルのコンテキスト内で中断点(suspension point)があればasyncコンテキストと推論される。

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

上記の例は、それぞれの中断点が、トップレベルコードをasyncコンテキストにするきっかけとなっていることを示している。`async let a = theAnswer()`は暗黙的で、`await theAnswer()`と`for await number in numbers`は明示的。

ただし、このトップレベルコードのasyncの推論は、コンテキストごとにsyncまたはasyncのコンテキストにすることができ、関数やクロージャの本文にまで波及しない。

```swift
func theAnswer() async -> Int { 42 }

let closure1 = { @MainActor in print(42) }
let closure2 = { () async -> Int in await theAnswer() }
```

上記の例は、トップレベルコードに中断点(`await`)がないためasyncコンテキストではない。

クロージャ本文がasyncコンテキストかどうかは`FindInnerAsync` ASTWalkerの内部で推論されている。最小限の労力で`FindInnerAsync`はトップレベルコードの本文も扱えるように一般化することができ、トップレベルコードとクロージャ本文のasyncの探索を並列で行う。

https://github.com/apple/swift/blob/377ca3578fd38e472f9af00d24aecd76d6f15202/lib/AST/Stmt.cpp#L168

#### 変数

トップレベルコードの変数は、ローカル変数のように順番に初期化されるが、グローバルスコープ内にあり、グローバル変数として扱われる。データ競合を防ぐために、変数は暗黙的にMain Actorに分離する必要がある。ただし、全てのトップレベルの変数へのアクセスで`await`する必要があるとしたら、それはとても扱いづらい。幸い、他のエントリポイントと同様に、トップレベルコードはMain Threadで実行されるため、変数に直接アクセスして変更できるように、トップレベルコードのスペースを暗黙的にMain Actorに分離することができる。しかし、これでもまだ既存のソースを壊してしまう可能性がある。変数がMain Actorである場合、sync関数はMain Actorに分離されていないため、トップレベルコードのsyncグローバル関数はエラーを出力する。潜在的なデータ競合があることを示す診断は正しいが、ソースを壊してしまうことは難点である。そこで、ソース破壊を軽減するために、変数には暗黙的に`@preconcurrency`属性のアノテーションが付けられる。この属性はSwift5のコードにのみ適用され、言語モードがSwift6に更新されると、これらのデータ競合はエラーになる。

`-warn-concurrency`がコンパイラに渡され、トップレベルコードで`await`がある場合、警告は、他のasyncコンテキストの場合と同様に、Swift5でもエラーになる。`await`がなく、このフラグが渡された場合、トップレベルがasyncコンテキストでなくても、変数はMain Actorによって暗黙的に保護され、Concurrencyチェックが厳密に実行される。トップレベルはasyncコンテキストではないため、実行ループは暗黙的に作成されず、オーバーロードを解決する際の振る舞いは変更されない。

要約すると、トップレベルの変数宣言は、データ競合の安全性とソース破壊の削減のバランスをとるために、`@MainActor @preconcurrency`で宣言されたかのように振る舞う。

グローバル変数の挙動に戻ると、指摘しなければならない追加の設計の詳細がいくつかある。

トップレベルの変数でGlobal Actorを明示的に指定する機能を削除することを提案したい。トップレベルの変数は、グローバル変数とローカル変数のハイブリッドのように扱われ、厄介な結果が生じる。変数はグローバルスコープで宣言されているため、どこでも使用できると見なされる。これにより、次の例のように、メモリの安全性に関する厄介な問題が発生する:

```swift
print(a)
let a = 10
```

この例は、実行時に"0"を出力する。`a`はグローバル変数であるため、`print`で使用できるが、初期化は順次行われるため、まだ初期化されていない。整数型およびその他のプリミティブは、暗黙的に0で初期化される。ただし、クラスは参照型であり、ゼロに初期化されるため、変数がクラス型の場合、segmentation faultが発生する。

最終的には、この穴をメモリモデルで補修したい。そのため、まだ開発段階ではあるが、トップレベルの変数を暗黙の`main`関数のローカル変数にする。そして、明示的なGlobal Actorを禁止して、ソース破壊を減らす。

### 既存のコードへの影響

トップレベルはasyncコンテキストではないため、現在、`await`をトップレベルコードで使うことはできない。このプロポーザルの機能は、トップレベルの`await`の存在する前提のため、このプロポーザルに影響を受けるscriptは存在しない。

### ABI stabilityへの影響

シグネチャは変わらず影響なし。

## 参考リンク

### Forums

- [Concurrency in Top-Level Code](https://forums.swift.org/t/concurrency-in-top-level-code/55001)

### プロポーザルドキュメント

- [Concurrency in Top-Level Code](https://github.com/apple/swift-evolution/blob/main/proposals/0343-top-level-concurrency.md)

### 関連PR

- [Fix top-level global-actor isolation crash](https://github.com/apple/swift/pull/40963)
- [Make top-level code variables @MainActor @preconcurrency](https://github.com/apple/swift/pull/40998)
- [Concurrent top-level detection](https://github.com/apple/swift/pull/41061)
- [SE-0343: De-experimentalify async top-level contex](https://github.com/apple/swift/pull/41805)

# Swift Region based Isolation

- [Swift Region based Isolation](#swift-region-based-isolation)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
    - [提案内容](#提案内容)
    - [詳細](#詳細)
  - [ソース互換性](#ソース互換性)
  - [ABI互換性](#abi互換性)
  - [将来の方向性](#将来の方向性)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

Swift Concurrencyはメモリをさまざまなアクターやタスクの*分離領域*に分割する。異なる領域に分離された計算は同時に実行する可能性があるため、`Sendable`ではない値が領域をまたいで渡せることを完全に防ぐために、`Sendable`チェックによって共有された可変状態への同時アクセスを遮断している。実際のところ、これは重大なセマンティック上の制限となっており、データ競合を起こさない自然なプログラミングパターンを禁止している。

このドキュメントでは、non-Sendableな値が分離境界を越えて安全に転送できるかどうかを判断するための新しい制御フローに依存するチェックを導入することによって、これらのルールを緩めることを提案する。これは分離領域という概念を導入することで、2つの値が互いに影響し合う可能性があるかをコンパイラが慎重に推論できるようにする。この分離領域を使用することで、分離境界を越えてnon-Sendableな値を転送しても、その値(およびその値を参照する可能性のある他の値)が転送時点以降に呼び出し元で使用されないため、データ競合が発生しないことを証明できる。

## 内容

### 動機

[SE-0302](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)は、non-Sendableな値は分離領域をまたがることはできないと宣言している。次のコードは、アクターに分離された関数の中で新しく値を構築した値を渡す時に`Sendale`チェックに違反している:

```swift
// non-Sendable
class Client {
  init(name: String, initialBalance: Double) { ... }
}

actor ClientStore {
  var clients: [Client] = []

  static let shared = ClientStore()

  func addClient(_ c: Client) {
    clients.append(c)
  }
}

func openNewAccount(name: String, initialBalance: Double) async {
  let client = Client(name: name, initialBalance: initialBalance)
  await ClientStore.shared.addClient(client) // Error! 'Client'は non-`Sendable`!
}
```

これは慎重すぎる。このプログラムが安全である理由は:

- `String`と`Double`は`Sendable`なので、`client`は、コンストラクタのパラメータからnon-`Sendable`な状態にアクセスしていない
- 構築されたばかりの`client`は、`openNewAccount`以外では使用できない
- `client`は`openNewAccount`内では`addClient`以外では使用されていない

上記の単純な例は、Swift のStrict Concurrencyチェックの過剰な制限を示している。プログラマは、このようなデータ競合がない一般的なパターンに対しても`@unchecked Sendable`のような安全でない非常手段を使用する必要がある。

### 提案内容

私たちは、分離境界を越えてnon-Sendableな値を転送することを可能にし、既に別の分離領域に転送されたnon-Sendableな値を使用する側でエラーを診断する、新しい制御フローセンシティブな診断の導入を提案する。

先ほどの例は、`addClient`の呼び出しによって`client`変数が`ClientStore.shared`アクターに転送された後、`client`変数はそれ以上使用されていないため、有効になる。`addClient`の呼び出し後に`openNewAccount`を変更して`client`のメソッドを呼び出すと、nonisolatedなコンテキストからアクター分離にされたコンテキストに既に転送されたnonSendableな値が同時にアクセスされる可能性があるため、コードは不正となる:

```swift
func openNewAccount(name: String, initialBalance: Double) async {
  let client = Client(name: name, initialBalance: initialBalance)
  await ClientStore.shared.addClient(client)
  client.logToAuditStream() // Error! clientStoreの分離領域にすでに転送されているため、データ競合の可能性がある!
}
```

`addClient`を呼び出した後でも、`client`から参照できないことが静的に証明された場合、その他のnonSendable値は安全に使用できる。

形式的には、2つの値xとyは、あるプログラム点pにおいて、以下の場合に同じ分離領域内にあると定義される:

1. xはpでyに別名をつけることができる
2. xまたはxのプロパティは、pでyのプロパティへのチェーンしてアクセスすることで、yから参照可能である

この定義により、xを使用するコードはyに影響を与えられないため、異なる分離領域でnon-Sendableな値を同時に使用できることが保証される。

```swift
let john = Client(name: "John", initialBalance: 0)
let joanna = Client(name: "Joanna", initialBalance: 0)

await ClientStore.shared.addClient(john)
await ClientStore.shared.addClient(joanna) // (1)
```

上記のコードは2つの新しい`Client`インスタンスを作成している。`john`が`joanna`を参照することも、その逆も不可能なので、これら2つの値は異なる分離領域に属する。異なる分離領域にある値は同時に使用できるので、(1)の`joanna`の使用は`ClientStore.shared`の内部で`john`にアクセスするコードと同時に実行される可能性があるが、データ競合からは安全である。

対照的に、`Client`に`friend`プロパティを追加し、`joanna`を`john.friend`に代入すると:

```swift
let john = Client(name: "John", initialBalance: 0)
let joanna = Client(name: "Joanna", initialBalance: 0)

john.friend = joanna // (1)

await ClientStore.shared.addClient(john)
await ClientStore.shared.addClient(joanna) // (2)
```

`(1)`での代入の後、`joanna`は`john.friend`を通して参照できるため、`(1)`では`john`と`joanna`は同じ分離領域にいなければならない。`(2)`での`joanna`へのアクセスは、`ClientStore.shared`内部で`john.friend`にアクセスするコードと同時に実行される可能性がある。そのため`(2)`で`joanna`を使用することはデータ競合の可能性があると診断される。


### 詳細

## ソース互換性

この提案は、受け入れ可能なSwiftプログラムのセットを厳密に拡張するもので、過去のコンパイラによって受け入れられたすべてのSwiftコードでも受け入れ可能なだろう。

## ABI互換性

影響なし

## 将来の方向性

TBD

## 参考リンク

### Forums

- [pitch1](https://forums.swift.org/t/pitch-safely-sending-non-sendable-values-across-isolation-domains/66566)
- [pitch2](https://forums.swift.org/t/pitch-region-based-isolation/67888)
- [review](https://forums.swift.org/t/se-0414-region-based-isolation/68805)
- [decision note](https://forums.swift.org/t/returned-for-revision-se-0414-region-based-isolation/69123)
- [second review](https://forums.swift.org/t/se-0414-second-review-region-based-isolation/69740)
- [acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0414-region-based-isolation/70051)

### プロポーザルドキュメント

- [Region based Isolation](https://github.com/apple/swift-evolution/blob/main/proposals/0414-region-based-isolation.md)
# Generalize AsyncSequence and AsyncIteratorProtocol

- [Generalize AsyncSequence and AsyncIteratorProtocol](#generalize-asyncsequence-and-asynciteratorprotocol)
  - [概要](#概要)
  - [内容](#内容)
    - [提案内容](#提案内容)
    - [詳細内容](#詳細内容)
      - [型付きスローの採用](#型付きスローの採用)
        - [`for try await` ループからのエラーの型推論](#for-try-await-ループからのエラーの型推論)
      - [primary associated typeの導入](#primary-associated-typeの導入)
      - [isolatedパラメータの導入](#isolatedパラメータの導入)
        - [ジェネリックなisolatedパラメータ](#ジェネリックなisolatedパラメータ)
      - [非同期 `for-in` ループでの `next(isolation:)` の呼び出し](#非同期-for-in-ループでの-nextisolation-の呼び出し)
      - [`next()` および `next(isolation:)` のデフォルト実装](#next-および-nextisolation-のデフォルト実装)
      - [`AsyncIteratorProtocol`準拠型のassociated typeの推論](#asynciteratorprotocol準拠型のassociated-typeの推論)
  - [ABI互換性](#abi互換性)
    - [導入することの影響](#導入することの影響)
  - [将来の検討事項](#将来の検討事項)
    - [next(isolation:)にデフォルトの引数を追加](#nextisolationにデフォルトの引数を追加)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

このプロポーザルでは、2つの方法で`AsyncSequence`を一般化する:

1. 特殊な `@rethrows` 属性は、型付きスローの導入によって適切な`throws`のポリモーフィズムに置き換えられる。
2. `AsyncIteratorProtocol` の`next`メソッドの要件の新しいオーバーロードには、アクターの分離を抽象化するための分離パラメータが含まれる

## 内容

`AsyncSequence` と `AsyncIteratorProtocol` は、`throws`の作用(effect)とアクターの分離に対してポリモーフィック(多相的)であるように設計されている。しかし、現在のAPI設計には、ジェネリックコードの表現力、`Sendable`チェック、および実行時のパフォーマンスに影響を与える重大な制限がある。

`AsyncSequence` の中には、イテレーション中にエラーをスローするものもあれば、しないものもある。呼び出し元が、指定されたシーケンスがエラーをスローする場合にのみ`try`を必須にできるように、`AsyncSequence`と`AsyncIteratorProtocol`は、プロトコルに特注の`@rethrows`属性を使用している。`@rethrows`属性は、throwsの作用に対するジェネリックな制約を付けられない為、[`AsyncSequence`にprimary associated typeを導入することもできない](https://forums.swift.org/t/se-0346-lightweight-same-type-requirements-for-primary-associated-types/55869/70)。`AsyncSequence`上のprimary associated typeを導入できれば、`AsyncSequence`上の変換APIなどの制約付きOpaque typeまたはExistential(実存型)の背後に具体的な実装の詳細を隠すことを可能にする:

```swift

extension AsyncSequence {
  // 'AsyncThrowingMapSequence' is an implementation detail hidden from callers.
  public func map<Transformed>(
    _ transform: @Sendable @escaping (Element) async throws -> Transformed
  ) -> some AsyncSequence<Element, any Error> { ... }
}
```

`AsyncSequence`型は、`Element`の型が`Sendable`と`Sendable`ではない両方で動作するように設計されているが、アクター分離されたコンテキストにおいて、`Sendable`ではない`Element`で`AsyncSequence`を使用することは現在のところ不可能である:

```swift
class NotSendable { ... }

@MainActor
func iterate(over stream: AsyncStream<NotSendable>) {
  for await element in stream { // warning: non-sendable type 'NotSendable?' returned by implicitly asynchronous call to nonisolated function cannot cross actor boundary

  }
}
```

`AsyncIteratorProtocol.next()`はnonisolated asyncなので、常にジェネリックなExecutor上で実行され、アクター分離されたコンテキストから呼び出すとアクターの分離境界を越えてしまう。結果が`Sendable`ではない場合、strict concurrencyチェックの下では呼び出しは無効となる。

さらに根本的なことだが、アクター分離されたコンテキストから `AsyncIteratorProtocol.next()`を呼び出すことは、今日の実務ではほぼ常に無効である。ほとんどの`AsyncIteratorProtocol`の具象型は `Sendable` ではない。`AsyncIteratorProtocol` を使用した同時並行イテレーションはプログラマのエラーであり、イテレータはそれを生成した分離ドメインから使用/変異されることを意図している。しかし、アクター分離されたコンテキストでイテレータが生成され、`next()`が呼び出されると、`Sendable` でないイテレータが分離境界を越えて渡されるため、strict concurrencyチェックに引っかかる。

最後に、ジェネリックなExecutor上で常に実行される`next()`は、アクターとジェネリックExecutor間の不要なホップの原因となります。

### 提案内容

このプロポーザルでは、`AsyncSequence` と `AsyncIteratorProtocol` の `@rethrows` 属性を`Failure` primary associated typeに置き換え、`Failure`型をスローすることで既存の`next()`要件をジェネリック化する新しいプロトコル要件を`AsyncIteratorProtocol`に追加し、アクターの分離を抽象化するために新しい要件に分離パラメータを追加する

```swift
@available(SwiftStdlib 5.1, *)
protocol AsyncIteratorProtocol<Element, Failure> {
  associatedtype Element

  mutating func next() async throws -> Element?

  @available(SwiftStdlib 6.0, *)
  associatedtype Failure: Error = any Error

  @available(SwiftStdlib 6.0, *)
  mutating func next(isolation actor: isolated (any Actor)?) async throws(Failure) -> Element?
}

@available(SwiftStdlib 5.1, *)
public protocol AsyncSequence<Element, Failure> {
  associatedtype AsyncIterator: AsyncIteratorProtocol
  associatedtype Element where AsyncIterator.Element == Element

  @available(SwiftStdlib 6.0, *)
  associatedtype Failure = AsyncIterator.Failure where AsyncIterator.Failure == Failure

  func makeAsyncIterator() -> AsyncIterator
}
```

新しい`next(isolation:)`にはデフォルトの実装があり、既存に準拠している型は現在と同じように動作する。`for-in`ループのコード生成は、コンテキストに適切なavailabilityがある場合に、`next()`の代わりに`next(isolation:)`を呼び出すように切り替わる。

### 詳細内容

#### 型付きスローの採用

`AsyncSequence` と `AsyncIteratorProtocol` の具象型は、`next()` を呼び出すとエラーをスローするかどうかを決定する。これは、`AsyncIteratorProtcol.next(isolation:)`要件でスローされるprimary associated typeの`Failure`型として各プロトコルに記述できる。associated typeでスローされるエラーを記述できるので、準拠した具象型は、型パラメータで要件を満たすことができる。つまり、ライブラリは、同じ非同期のイテレーション機能を持つスローする具象型とスローしない具象型バージョンを別々に公開する必要がない。

##### `for try await` ループからのエラーの型推論

`Failure`associated typeは、Swift 6.0の標準ライブラリの実行時のみアクセス可能である。古い標準ライブラリのバージョンに対して実行されるコードは、`AsyncSequence` と `AsyncIteratorProtocol`へ準拠するwitness tableに `Failure` 要件を含まない。これは、`for try await`ループのエラーの型推論に影響する。

`AsyncIteratorProtocol`からスローされるエラー型が、associated typeのwitnessを介して(コンテキストが適切なavailabilityを持つため)、またはイテレータの型が具象型のため利用可能である場合、非同期シーケンスに対するイテレーションはその`Failure`型をスローする:

```swift
struct MyAsyncIterator: AsyncIteratorProtocol {
  typealias Failure = MyError
  ...
}

func iterate<S: AsyncSequence>(over s: S) where S.AsyncIterator == MyAsyncIterator {
  let closure = {
    for try await element in s {
      print(element)
    }
  }
}
```

上記のコードでは、`closure`の型は `() async throws(MyError) -> Void`である。

`AsyncIteratorProtocol`からスローするエラー型が利用できない場合、非同期シーケンスに対するイテレーション処理は`any Error`をスローする:

```swift
@available(SwiftStdlib 5.1, *)
func iterate(over s: some AsyncSequence) {
  let closure = {
    for try await element in s {
      print(element)
    }
  }
}
```

上記のコードでは、`closure`の型は`() async throws(any Error) -> Void`である。

与えられた非同期シーケンスの`Failure`型が`Never`に制約されている場合、`for-in`ループに`try`は必要ない:

```swift
struct MyAsyncIterator: AsyncIteratorProtocol {
  typealias Failure = Never
  ...
}

func iterate<S: AsyncSequence>(over s: S) where S.AsyncIterator == MyAsyncIterator {
  let closure = {
    for await element in s {
      print(element)
    }
  }
}
```

上記のコードでは、`closure`の型は `() async -> Void`である。

#### primary associated typeの導入

`Element`と`Failure` associated typeは、primary associated typeに昇格した。これにより、制約付きの実存(Existential)型と、Opaqueな`AsyncSequence` および `AsyncIteratorProtocol`型を使用できるようになる。

#### isolatedパラメータの導入

`next(isolation:)`の要件は、[`isolated`パラメータ](https://github.com/hborla/swift-evolution/blob/generalize-async-sequence/proposals/0313-actor-isolation-control.md)を使用してアクターの分離を抽象化する。`next(isolation:)`の呼び出し元が、[SE-0414: Region based isolation](https://github.com/hborla/swift-evolution/blob/generalize-async-sequence/proposals/0414-region-based-isolation.md) の下で分離境界を越えて転送できないイテレータ値を渡す場合、その呼び出しは分離境界を越えない場合のみ有効となる。`next(isolation:)`が呼び出し元の分離を共有するかどうかは、isolatedパラメータの値に依存し、`nonisolated`とグローバルアクターの分離に関するジェネリックなルールは以下に記載する。

##### ジェネリックなisolatedパラメータ

関数は`isolated`パラメータを使ってアクター分離を抽象化することができる。また、`nonisolated`のコンテキストを表現するために、`isolated`パラメータをOptionalにできる。

以下の場合、`isolated`パラメータを持つ関数を呼び出すと、呼び出し元の分離が共有される:

- 現在のコンテキストがnonisolated、パラメータの型がOptional、引数の式が`nil`または`Optional.none`への参照である場合
- 現在のコンテキストに(アクター関数で暗黙的に分離された `self`パラメータを含めた)`isolated`パラメータがあり、引数の式が、そのパラメータへの参照またはそのパラメータの非オプション派生(下記参照)である場合
- 現在のコンテキストがグローバルアクター型`T`に分離されており、引数の式が`T.shared`で、`shared`がグローバルアクターのプロトコル要件またはグローバルアクターへの`T`の適合性でそれを提供する具象型である場合

以下の場合、式は`isolated`パラメータの`param`の非optional派生である：

- `param?`(Optionalチェーン演算子)
- `param!`(強制アンラップ演算子)、または
- `param`の*非Optionalバインディング*への参照の場合。例えば`let ref = param` の `ref` のように、`param` からOptionalを取り除くパターンマッチが成功した場合に初期化される `let` 定数。

上記のすべてのケースで引数の式を分析する場合、式の構文と動作における特定の具体的ではない違いは無視しなければならない:

- 括弧
- 作用マーク演算子`try`、`try?`、`try!`と`await`
- 型強制演算子のas(動的なブリッジング変換を行わない場合)、および`Optional`型への暗黙の型変換。

#### 非同期 `for-in` ループでの `next(isolation:)` の呼び出し

非同期`for-in`ループの糖衣構文を分解すると、コンテキストに適切なavailabilityがある場合、`next()`の代わりに`AsyncIteratorProtocol.next(isolation:)`を呼び出し、`(any Actor)?`型に分離された引数値を渡す。引数値は、呼び出しが分離境界を越えないように、常に呼び出し元の分離を表す:

- 呼び出し元のコンテキストが分離されていない場合、引数値は`nil`
- 呼び出し元のコンテキストがグローバルアクター`T`に分離されている場合、引数値は`T.shared`
- 呼び出し元のコンテキストが`isolated`パラメータ`param`を持つ場合、引数値は`param`
- 呼び出し元のコンテキストが`isolated`パラメータ`param`をキャプチャしている場合、引数値は`param`

#### `next()` および `next(isolation:)` のデフォルト実装

既存の `AsyncIteratorProtocol`に準拠している型は `next()` のみを実装しているため、標準ライブラリは `next(isolation:)` のデフォルト実装を提供する:

```swift
extension AsyncIteratorProtocol {
  /// これは、既存の非同期イテレータとの後方互換性を維持するために必要な
  /// next(isolation:)のデフォルト実装
@available(SwiftStdlib 6.0, *)
  @available(*, deprecated, message: "Provide an implementation of 'next(isolation:)'")
  public mutating func next(isolation actor: isolated (any Actor)?) async throws(Failure) -> Element? {
    nonisolated(unsafe) var unsafeIterator = self
    do {
      let element = try await unsafeIterator.next()
      self = unsafeIterator
      return element
    } catch {
      throw error as! Failure
    }
  }
}
```

`next(isolation:)`のデフォルトの実装は、`self`が分離されている可能性のあるコンテキストからnonisolatedなコンテキストに渡すため、必然的に`Sendable`チェックに違反していることに注目してほしい。これは一般的には安全ではないが、`next()`の呼び出しが現在このように動いているのである。`next(isolation:)`を直接実装することで、安全でない動作はなくなる。

`AsyncIteratorProtocol`への準拠が`next(isolation:)`のみを実装できるように、`next()`にもデフォルトの実装が用意されている:

```swift
extension AsyncIteratorProtocol {
  @available(SwiftStdlib 6.0, *)
  public mutating func next() async throws(Failure) -> Element? {
    // `next()`を呼び出すと、ジェネリックExecutor上で常に `next(isolation:)` が実行される
    try await next(isolation: nil)
  }
}
```

`AsyncIteratorProtocol` の両方の関数の要件は、デフォルト実装がお互いの形式で書かれている。つまり、両方実装するとプログラマのエラーになる。デフォルトの実装は Swift 6.0 標準ライブラリでのみ利用可能であるため、Swift 6.0 標準ライブラリより前に利用可能な型は、`next()` の実装を提供する必要がある。

どちらの要件も実装していない準拠を暗黙的に許してしまう状況を避け、`next()` から `next(isolation:)` への準拠への移行を容易にするために、私たちは、deprecated、obsoleted、またはunavailableなデフォルトのwitness実装を使用するプロトコル準拠をwitnessチェッカーが診断する、新しいavailabilityルールを追加する。deprecatedな実装は警告を出し、obsoletedやunavailableな実装はエラーを出力する。

`next(isolation:)`のデフォルト実装はdeprecatedであるため、直接的な実装を提供しない準拠は警告を生成しする。これは、`next(isolation:)`のデフォルトの実装が`Sendable`チェックに違反するため期待通りであり、ソースの互換性のために必要であるが、準拠する型に新しいメソッドを実装するよう積極的に提案することが重要です。

#### `AsyncIteratorProtocol`準拠型のassociated typeの推論

`AsyncIteratorProtocol` 準拠型が `next(isolation:)` 関数を提供する場合、[SE-0413](https://github.com/apple/swift-evolution/blob/main/proposals/0413-typed-throws.md) で説明されているルールを使用して、`next(isolation:)` がエラーをスローするかどうか(そして何をスローするのか)に基づいて `Failure` 型が推論される。

`AsyncIteratorProtocol` に準拠型が `next(isolation:)`のデフォルト実装を使用する場合、`Failure` のassociated typeは、代わりに`next`関数から推論される。`next` 関数からスローされる型が何であれ(スローされない場合は`Never`)、`Failure` 型として推論される。

##　ソース互換性

`AsyncSequence`と`AsyncIteratorProtocol`に対する新しい要件は追加的なもので、デフォルトの実装と`Failure` associated typeの経験則上の推論により、これらのプロトコルに準拠する既存の型は引き続き動作することが保証される。

`AsyncSequence`と`AsyncIteratorProtocol`で使用されている実験的な「rethrow要件への準拠」機能は、ソースの互換性に関していくつかの課題を提示している。すなわち、これらのrethrowするプロトコルへの準拠を、rethrowされるエラーの原因とみなす`rethrows`関数を宣言できる。例えば、以下の`rethrows`関数は現在有効である:

```swift
extension AsyncSequence {
  func contains(_ value: Element) rethrows -> Bool where Element: Hashable { ... }
}
```

実験的な「rethrow要件への準拠」機能の削除により、エラーをスローできるクロージャ引数がないため、この関数は不正となる。このような関数のソース互換性を維持するために、この提案は、`AsyncSequence`と`AsyncIteratorProtocol`の要件が`rethrows`チェックに関われるようにする特定のルールを導入する。これは、`rethrows`関数は、すべての`T: AsyncSequence`または`T: AsyncIteratorProtocol`の準拠要件ごとに`T.Failure`をスローできるとみなされる。この`contains`の場合、それは`Self.Failure`をスローできるということである。これらの `rethrows` 関数の定義を許可するルールは、Swift 6 より前でのみ可能である。

## ABI互換性

この提案は純粋に標準ライブラリの ABI を拡張するものであり、既存の機能を変更するものではない。既存の`next()`要件を修正するのではなく、新しい`next(isolation:)`要件を追加することが、ABIの互換性を維持するために必要であることに注意してほしい。これは`next()`をアクターへの分離を抽象化するために変更すると、実装内の非同期呼び出しの後にそのアクターにホップバックするため、アクターをパラメータとして渡さなければならなくなるからである。型付きスローのABIはrethrowsのABIとも異なるため、型付きスローの採用だけでも新しい要件が必要になる。

### 導入することの影響

`AsyncSequence` と `AsyncIteratorProtocol` のassociated typeの`Failure` 型は、Swift6.0標準ライブラリの実行時でのみ利用可能である。associated typeを通して `Failure` 型にアクセスする必要があるコードは、例えば、動的キャストやジェネリックシグネチャで制約するために、availabilityを設定する必要がある。このため、`next()` と `next(isolation:)`のデフォルトの実装は、Swift 6.0標準ライブラリと同じavailabilityを持つ。

これは、`AsyncIteratorProtocol`準拠する具象型がSwift6.0標準ライブラリよりも前のバージョンで利用できる場合、(`next()`の実装を提供せずに)`next(isolation:)`のみの実装に切り替えられないということである。

同様に、`AsyncSequence` と `AsyncIteratorProtocol` のPrimary associated typeは、Swift 6.0 のavailabilityに制約されなければならない。

`Async{Throwing}Stream.Iterator`のような標準ライブラリの`AsyncIteratorProtocol`の具象型が、`next(isolation:)`を直接実装すれば、アクターで分離されたコンテキストでそれらの`AsyncSequence`の具象型をiterationするコードは、実行時にジェネリックエクゼキュータへのホップが少なくなるかもしれない。

## 将来の検討事項

### next(isolation:)にデフォルトの引数を追加

`next(isolation:)`の呼び出しのほとんどは、そのコンテキストを囲むコンテキストの分離状態を渡す。プロトコル要件がデフォルト引数を持てないという制限を解除し、[アクター分離の継承](https://github.com/apple/swift-evolution/blob/main/proposals/0420-inheritance-of-actor-isolation.md)で説明したように`#isolated`というデフォルト引数の値を追加することを検討してもよいだろう。

## 参考リンク

### Forums

- [pitch](https://forums.swift.org/t/pitch-generalize-asyncsequence-and-asynciteratorprotocol/69283)
- [review](https://forums.swift.org/t/se-0421-generalize-effect-polymorphism-for-asyncsequence-and-asynciteratorprotocol/69662)
- [acceptance](https://forums.swift.org/t/accepted-se-0421-generalize-effect-polymorphism-for-asyncsequence-and-asynciteratorprotocol/69973)

### プロポーザルドキュメント

- [Generalize effect polymorphism for AsyncSequence and AsyncIteratorProtocol](https://github.com/apple/swift-evolution/blob/main/proposals/0421-generalize-async-sequence.md)
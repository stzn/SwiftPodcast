# Swift buildPartialBlockでResultBuilderのボイラープレート削減

- [Swift buildPartialBlockでResultBuilderのボイラープレート削減](#swift-buildpartialblockでresultbuilderのボイラープレート削減)
  - [概要](#概要)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決策](#解決策)
    - [アーリーアダプタからのフィードバック](#アーリーアダプタからのフィードバック)
  - [詳細](#詳細)
  - [ソース互換性](#ソース互換性)
  - [検討された代替案](#検討された代替案)
    - [buildPartialBlockよりも実行可能なbuildBlockオーバーロードを優先する](#buildpartialblockよりも実行可能なbuildblockオーバーロードを優先する)
    - [引数なしbuildPartialBlock()を使用して初期値を形成](#引数なしbuildpartialblockを使用して初期値を形成)
    - [Variadic genericsへの依存](#variadic-genericsへの依存)
    - [別名](#別名)
      - [`buildBlock`というメソッド名をオーバーロードする](#buildblockというメソッド名をオーバーロードする)
      - [buildPartialBlock(first:)の代わりにbuildBlock(_:)を使う](#buildpartialblockfirstの代わりにbuildblock_を使う)
      - [異なる引数ラベル](#異なる引数ラベル)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

Result builderに、ブロックのコンポーネントをペアで組み合わせることができる新しいカスタマイズポイントを導入する。

```swift
@resultBuilder
enum Builder {
    /// Builds a partial result component from the first component.
    static func buildPartialBlock(first: Component) -> Component

    /// Builds a partial result component by combining an accumulated component
    /// and a new component.  
    /// - Parameter accumulated: A component representing the accumulated result
    ///   thus far. 
    /// - Parameter next: A component representing the next component after the
    ///   accumulated ones in the block.
    static func buildPartialBlock(accumulated: Component, next: Component) -> Component
}
```

`buildPartialBlock(first:)`と`buildPartialBlock(accumulated:next:)`の両方が提供されている場合、[result builderの変換](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md#the-result-builder-transform)は、ブロック内のコンポーネントを`buildPartialBlock`への一連の呼び出しに変換し、後続の1行を結果に一気に結合する:

```swift
// 元
{
    expr1
    expr2
    expr3
}

// 変換後
// 注: buildFinalResultとbuildExpressionは現在と同じで定義された場合のみ呼び出される
{
    let e1 = Builder.buildExpression(expr1)
    let e2 = Builder.buildExpression(expr2)
    let e3 = Builder.buildExpression(expr3)
    let v1 = Builder.buildPartialBlock(first: e1)
    let v2 = Builder.buildPartialBlock(accumulated: v1, next: e2)
    let v3 = Builder.buildPartialBlock(accumulated: v2, next: e3)
    return Builder.buildFinalResult(v3)
}
```

このプロポーザルの主な目的は、複数のbuildBlockのオーバーロードによって引き起こされるコードの膨張を減らし、ライブラリがbuilderベースの汎用DSLを楽に簡単に定義できるようにすること。

## 内容

### 問題点

Result builderを利用したDSLの中で、ブロック内のジェネリック型と値を組み合わせて、コンポーネントのジェネリックパラメータを含む新しい型を生成するのが一般的なパターン。例えば、SwiftUIの`ViewBuilder`と`SceneBuilder`は、`buildBlock`を使用して、強力な型を失うことなくViewとSceneを結合する:

```swift
extension SceneBuilder {
    static func buildBlock<Content>(Content) -> Content
    static func buildBlock<C0, C1>(_ c0: C0, _ c1: C1) -> some Scene where C0: Scene, C1: Scene
    ...
    static func buildBlock<C0, C1, C2, C3, C4, C5, C6, C7, C8, C9>(_ c0: C0, _ c1: C1, _ c2: C2, _ c3: C3, _ c4: C4, _ c5: C5, _ c6: C6, _ c7: C7, _ c8: C8, _ c9: C9) -> some Scene where C0: Scene, C1: Scene, C2: Scene, C3: Scene, C4: Scene, C5: Scene, C6: Scene, C7: Scene, C8: Scene, C9: Scene
}
```

可変個引数のジェネリクス(Variadic Generics)がないため、サポートされている引数の個数に対して`buildBlock`をオーバーロードする必要がある。残念ながら、これによりコードサイズが大きくなり、実装とドキュメントでコードが大幅に肥大化し、ボイラープレートを作成して維持するのが困難になることがよくある。

このアプローチは`ViewBuilder`や`SceneBuilder`などの型で機能するが、一部のbuilderは、オーバーロードで実装するには複雑すぎる型の組み合わせルールを定義する必要がある。そのような例の1つは、[Declarative String Processing](https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/DeclarativeStringProcessing.md)の[RegexComponentBuilder](https://github.com/apple/swift-experimental-string-processing/blob/85c7d906dd871364357156126278d9d427936ca4/Sources/_StringProcessing/RegexDSL/Builder.swift#L13)。

正規表現のbuilder DSLは、開発者が正規表現パターンを簡単に作成できるように設計されている。[Strongly typed captures](https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/Evolution/StronglyTypedCaptures.md#strongly-typed-regex-captures)は、builderのイニシャライザを持つ`Regex`の`Match`ジェネリックパラメータの一部となる。

```swift
struct Regex<Match> {
   init(@RegexComponentBuilder _ builder: () -> Self)
}
```

> おさらい: 正規表現キャプチャの基本
> 正規表現にキャプチャグループが含まれていない場合、その`Match`型は`Substring`であり、入力の一致した部分全体を表す:
> ```swift
> //                           > ________________________________
> //                        .0 > |                           .0 |
> //                  ____________________                _________
> let yesCaptures = #/a(?:(b+)c(d+))+e(f)?/# // => Regex<(Substring, Substring, Substring, Substring?)>
> //                      ---- ----   ---                            ---------  ---------  ----------
> //                    .1 | .2 |   .3 |                              .1 |       .2 |       .3 |
> //                       |    |      |                                 |          |          |
> //                       |    |      |_______________________________  |  ______  |  ________|
> //                       |    |                                        |          |
> //                       |    |______________________________________  |  ______  |
> //                       |                                             |
> //                       |_____________________________________________|
> //                                                                 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
> //                                                                          Capture types
> ```

Result builder構文を使用すると、上記の正規表現は次のようになる:

```swift
let regex = Regex {
    "a"                             // => Regex<Substring>
    OneOrMore {                     // {
        Capture { OneOrMore("b") }  //     => Regex<(Substring, Substring)>
        "c"                         //     => Regex<Substring>
        Capture { OneOrMore("d") }  //     => Regex<(Substring, Substring)>
    }                               // } => Regex<(Substring, Substring, Substring)>
    "e"                             // => Regex<Substring>
    Optionally { Capture("f") }     // => Regex<(Substring, Substring?)>
}                                   // => Regex<(Substring, Substring, Substring, Substring?)>
                                    //                      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                                    //                               Capture types

let result = "abbcddbbcddef".firstMatch(of: regex)
// => MatchResult<(Substring, Substring, Substring, Substring?)>
```

`RegexComponentBuilder`は、すべてのコンポーネントのキャプチャタイプをフラットなタプルとして連結し、`Match`タイプが`(Substring、CaptureType ...)`である新しい`Regex`を形成する次のような`RegexComponentBuilder`を定義できる:

```swift
@resultBuilder
enum RegexComponentBuilder {
    static func buildBlock() -> Regex<Substring>
    static func buildBlock<Match>(_: Regex<Match>) -> Regex<Match>
    // Overloads for non-tuples:
    static func buildBlock<WholeMatch0, WholeMatch1>(_: Regex<WholeMatch0>, _: Regex<WholeMatch1>) -> Regex<Substring>
    static func buildBlock<WholeMatch0, WholeMatch1, WholeMatch2>(_: Regex<WholeMatch0>, _: Regex<WholeMatch1>, _: Regex<WholeMatch2>) -> Regex<Substring>
    ...
    static func buildBlock<WholeMatch0, WholeMatch1, WholeMatch2, ..., WholeMatch10>(_: Regex<WholeMatch0>, _: Regex<WholeMatch1>, _: Regex<WholeMatch2>, ..., _: Regex<WholeMatch10>) -> Regex<Substring>
    // Overloads for tuples:
    static func buildBlock<W0, W1, C0>(_: Regex<(W0, C0)>, _: Regex<W1>) -> Regex<(Substring, C0)>
    static func buildBlock<W0, W1, C0>(_: Regex<W0>, _: Regex<(W1, C0)>) -> Regex<(Substring, C0)>
    static func buildBlock<W0, W1, C0, C1>(_: Regex<(W0, C0, C1)>, _: Regex<W1>) -> Regex<(Substring, C0, C1)>
    static func buildBlock<W0, W1, C0, C1>(_: Regex<(W0, C0)>, _: Regex<(W1, C1)>) -> Regex<(Substring, C0, C1)>
    static func buildBlock<W0, W1, C0, C1>(_: Regex<W0>, _: Regex<(W1, C0, C1)>) -> Regex<(Substring, C0, C1)>
    ...
    static func buildBlock<W0, W1, W2, W3, W4, C0, C1, C2, C3, C4, C5, C6>(
       _: Regex<(W0, C0, C1)>, _: Regex<(W1, C2)>, _: Regex<(W3, C3, C4, C5)>, _: Regex<(W4, C6)>
    ) -> Regex<(Substring, C0, C1, C2, C3, C4, C5, C6)>
    ...
}
```

ここでは、各引数の個数のすべてのタプルの組み合わせに対してオーバーロードする必要があります...これは、buildBlockオーバーロードの`O(arity!)`の組み合わせの爆発。これらのメソッドを単独でコンパイルすると、数時間かかる場合がある。

### 解決策

この提案では、異なる種類のリストの作成に似た新しいブロック作成アプローチを導入する。このアプローチでは、単一のメソッドを呼び出してブロック全体を大規模に構築する代わりに、一度に1つの新しいコンポーネントを取得することで部分的なブロックを再帰的に構築するため、オーバーロードの数を大幅に減らすことができる。

2つのユーザ定義のstaticメソッドを介してresult builderに新しいカスタマイズポイントを導入する:

```swift
@resultBuilder
enum Builder {
    static func buildPartialBlock(first: Component) -> Component
    static func buildPartialBlock(accumulated: Component, next: Component) -> Component
}
```

`buildPartialBlock(first:)`と`buildPartialBlock(accumulated:next:)`の両方が定義されている場合、result builder変換は、ブロック内のコンポーネントを`buildPartialBlock`への一連の呼び出しに変換し、コンポーネントを上から下に結合する。

このアプローチを使用すると、`buildBlock`がオーバーロードされた多くのresult builder型を簡略化できる。例えば、SwiftUIの`SceneBuilder`の`buildBlock`オーバーロードは、次のように簡略化できる。

```swift
extension SceneBuilder {
    static func buildPartialBlock(first: some Scene) -> some Scene 
    static func buildPartialBlock(accumulated: some Scene, next: some Scene) -> some Scene 
}
```

同様に、`RegexComponentBuilder`の`buildBlock`のオーバーロードは、`O(arity!)`から`buildPartialBlock(accumulated:next:)`の`O(arity^2)`オーバーロードまで大幅に減らすことができる。引数の個数がが10の場合、300万を超えるオーバーロードと比較すると、100のオーバーロードはとても楽:


```swift
extension RegexComponentBuilder {
    static func buildPartialBlock<M>(first regex: Regex<M>) -> Regex<M>
    static func buildPartialBlock<W0, W1>(accumulated: Regex<W0>, next: Regex<W1>) -> Regex<Substring>
    static func buildPartialBlock<W0, W1, C0>(accumulated: Regex<(W0, C0)>, next: Regex<W1>) -> Regex<(Substring, C0)>
    static func buildPartialBlock<W0, W1, C0>(accumulated: Regex<W0>, next: Regex<(W1, C0)>) -> Regex<(Substring, C0)>
    static func buildPartialBlock<W0, W1, C0, C1>(accumulated: Regex<W0>, next: Regex<(W1, C0, C1)>) -> Regex<(Substring, C0, C1)>
    static func buildPartialBlock<W0, W1, C0, C1>(accumulated: Regex<(W0, C0, C1)>, next: Regex<W1>) -> Regex<(Substring, C0, C1)>
    static func buildPartialBlock<W0, W1, C0, C1>(accumulated: Regex<(W0, C0)>, next: Regex<(W1, C1)>) -> Regex<(Substring, C0, C1)>
    ...
}
```

### アーリーアダプタからのフィードバック

- [regex builder DSL](https://forums.swift.org/t/pitch-regex-builder-dsl/56007)の中で、`buildPartialBlock`は、必要なオーバーロードの数を数百万`((arity!))`から数百`(O(arity^2))`に減らした
- [pointfreeからの報告](https://forums.swift.org/t/pitch-buildpartialblock-for-result-builders/55561/10)によると、`buildPartialBlock`は、生成されたコードの2万1000行の削除を可能にし、引数の個数サポートを強化し、デバッグモードでのコンパイル時間を20秒から2秒未満に短縮した

## 詳細

型が`@resultBuilder`でマークされている場合、その型には以前は少なくとも1つのstatic `buildBlock`メソッドが必要だった。この提案では、このような型には、少なくとも1つのstatic `buildBlock`メソッド、または`buildPartialBlock(first:)`と`buildPartialBlock(accumulated:next:)`の両方が必要になる。

Result builder変換では、コンパイラはbuilder型のstaticメンバ`buildPartialBlock(first:)`および`buildPartialBlock(accumulated:next:)`を次の条件が満たされている場合に検索する:

- `buildPartialBlock(first:)`と`buildPartialBlock(accumulated:next:)`の両方のメソッドが存在する
- 囲んでいる宣言のavailabilityは、`buildPartialBlock(first:)`と`buildPartialBlock(accumulated:next:)`のavailabilityよりも広範囲

次のブロックはこのように変換される:

```swift
// 元
{
    expr1
    expr2
    expr3
}

// 変換後
// 注: buildFinalResultとbuildExpressionは現在と同じで定義された場合のみ呼び出される
{
    let e1 = Builder.buildExpression(expr1)
    let e2 = Builder.buildExpression(expr2)
    let e3 = Builder.buildExpression(expr3)
    let v1 = Builder.buildPartialBlock(first: e1)
    let v2 = Builder.buildPartialBlock(accumulated: v1, next: e2)
    let v3 = Builder.buildPartialBlock(accumulated: v2, next: e3)
    return Builder.buildFinalResult(v3)
}
```

それ以外の場合、result builder変換は、[SE-0289](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md)で提案されているように、代わりに`buildBlock`を呼び出すようにブロックを変換する。


## ソース互換性

このプロポーザルは、ソースを壊すような変更を導入することを意図していない。ただし、既存のresult builder型に`buildPartialBlock(first:)`と`buildPartialBlock(accumulated:next:)`という名前のstatic メソッドがある場合、result builder変換は代わりにそれらのメソッドへの呼び出しを作成し、`buildPartialBlock`の型シグネチャと実装次第ではエラーにあるかもしれない。しかし、そのようなケースは非常にレアなはず。

## 検討された代替案

### buildPartialBlockよりも実行可能なbuildBlockオーバーロードを優先する

提案されているように、result builder変換は、両方が定義されている場合、`buildBlock`よりも`buildPartialBlock`を常に優先する。`buildBlock`の実行可能なオーバーロードを1回呼び出すだけで、より効率的にカスタマイズできると主張する人もいるかもしれない。 ただし、result builder変換は、現在型推論が完了する前に動作するため、引数の型に基づいて`buildBlock`または`buildPairwiseBlock`を呼び出すことを決定するための型チェックの複雑さが増す。result builder変換によって変換される際、result builderの内部要件(`buildBlock`、`buildOptional`など)は、引数の型に依存しない。

### 引数なしbuildPartialBlock()を使用して初期値を形成

提案されているように、result builder変換は、ブロック内の最初のコンポーネントで単項`buildPartialBlock(first;)`を呼び出してから、残りのコンポーネントで`buildPartialBlock(accumulated:next:)`を呼びす。初期の結果を形成するために引数なし`buildPartialBlock()`メソッドを定義するようにユーザに要求することは可能だが、この動作は、SwiftUIの`SceneBuilder`のように空のブロックをサポートすることを意図していないresult builderには最適ではない可能性がある。さらに、提案されたアプローチでは、ユーザが引数なし`buildPartialBlock()`を定義して、空のブロックの構築をサポートすることができる。

### Variadic genericsへの依存

可変個引数ジェネリック(Variadic generics)は、今回のプロポーザルを解決することができると言える。ただし、`RegexComponentBuilder`に必要な連結動作を実現するには、ネストされた型シーケンスを表現し、要素の削除やスプラッティング(※)などのジェネリックパラメータのパックで`Collection`のような変換を実行できる必要がある。

※ パラメータ値のコレクションをまとめてコマンドに渡す手法

```swift
extension RegexComponentBuilder {
    static func buildBlock<(W, (C...))..., R...>(_ components: Regex<(W, C...)>) -> Regex<(Substring, (R.Match.dropFirst()...).splat())>
}
```

このような機能は型推論を大変複雑にする。

### 別名

#### `buildBlock`というメソッド名をオーバーロードする

提案された機能は`buildBlock`と重複しているため、`buildPartialBlock`の代わりに`buildBlock`をメソッドのベース名として再利用し、引数ラベルを使用して2つを組にしたバージョンであるかどうかを区別することもできるという意見もある(例えば、`buildBlock(partiallyAccumulated:next:)`または`buildBlock(combining:into:)`)。

```swift
extension Builder {
    static func buildBlock(_: Component) -> Component
    static func buildBlock(partiallyAccumulated: Component, next: Component) -> Component
}
```

ただし、「buildBlockビ」という句には、メソッドが実際に部分ブロックをビルドしていることを明確に示すものはなく、引数ラベルにはそのような表示を行うための強調できるものがない。

#### buildPartialBlock(first:)の代わりにbuildBlock(_:)を使う

単項ベースの`buildPartialBlock(first:)`と`buildBlock(_ :)`は機能的に同等であると見なすことができるため、`buildBlock`の再利用できるという意見もある。 ただし、overload buildBlockメソッド名で説明したように、「buildBlock」というフレーズは明確ではない。

さらに重要な理由は、`buildPartialBlock(first:)`は、特に引数ラベルが`first:`の場合、開発者が組み合わせの方向を指定できるカスタマイズポイント用のスペースを残すことができる。将来の方向性として、`buildPartialBlock(first:)`の代わりに`buildPartialBlock(last:)`を定義できる。その場合、result builder変換は最初に最後のコンポーネント`buildPartialBlock(last:)`を呼び出し、次に先に変換された各コンポーネントに対して`buildPartialBlock(accumulated:next:)`が呼び出される。

#### 異なる引数ラベル

提案された引数ラベル`accumulated:`と`next:`は、標準ライブラリのいくつかの先例からインスピレーションを得た:

- 「accumulated」は、`Array.reduce（into:_:)`の引数名として使用されている
- 「next」は、`Array.reduce（_:_:)`の引数名として使用されている

一方、`accumulated:`と`next:`の代わりに、いくつかの代替引数ラベルが検討された。

`accumulated:`の代替案:

- `partialResult:`
- `existing:`
- `upper:`
- `_:`

`next:`の代替案:

- `new:`
- `_:`

`accumulated:`と`next:`の方がより明確であると思っている。

## 参考リンク

### Forums

- [[Pitch] `buildPartialBlock` for result builders](https://forums.swift.org/t/pitch-buildpartialblock-for-result-builders/55561)
- [SE-0348: buildPartialBlock for Result Builders](https://forums.swift.org/t/se-0348-buildpartialblock-for-result-builders/56143)

### プロポーザルドキュメント

-[buildPartialBlock for result builders](https://github.com/apple/swift-evolution/blob/main/proposals/0348-buildpartialblock.md)
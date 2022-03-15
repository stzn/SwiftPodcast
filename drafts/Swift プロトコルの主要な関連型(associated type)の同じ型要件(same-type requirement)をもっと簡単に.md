# Swift プロトコルの主要な関連型(associated type)の同じ型要件(same-type requirement)をもっと簡単に

- [Swift プロトコルの主要な関連型(associated type)の同じ型要件(same-type requirement)をもっと簡単に](#swift-プロトコルの主要な関連型associated-typeの同じ型要件same-type-requirementをもっと簡単に)
  - [概要](#概要)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決策](#解決策)
    - [詳細](#詳細)
      - [糖衣シンタックスの位置で制約されたプロトコル](#糖衣シンタックスの位置で制約されたプロトコル)
      - [opaque result typeの制約付きプロトコル](#opaque-result-typeの制約付きプロトコル)
      - [他に使える位置](#他に使える位置)
      - [非サポート位置](#非サポート位置)
    - [その他の代替案](#その他の代替案)
      - [関連型の名前を必須にする 例: Collection<.Element == String>](#関連型の名前を必須にする-例-collectionelement--string)
      - [最初にopaque result type要件のよりジェネリックなシンタックスを実装](#最初にopaque-result-type要件のよりジェネリックなシンタックスを実装)
    - [通常のassociatedtype宣言に主要な関連型のアノテーションを付ける](#通常のassociatedtype宣言に主要な関連型のアノテーションを付ける)
      - [ジェネリックプロトコル(Generic protocols)](#ジェネリックプロトコルgeneric-protocols)
    - [ソース互換性](#ソース互換性)
    - [ABIの安定性への影響](#abiの安定性への影響)
    - [APIのレジリエンスへの影響](#apiのレジリエンスへの影響)
    - [将来の検討事項](#将来の検討事項)
      - [標準ライブラリへの導入](#標準ライブラリへの導入)
      - [制約付き存在型](#制約付き存在型)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

[Improving the UI of generics](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--directly-expressing-constraints)を実現するためのステップとして、このプロポーザルではジェネリックパラメータに準拠し、同じ型要件(※)が必要な関連型(associated type)を制約するための新しいシンタックスを紹介する。

※ 同じ型要件とは以下のようなものを指す

```swift
Collection where Element == String
```

## 内容

### 問題点

ソースファイルの各行を`AsyncSequence`として取得する関数を考える:

```swift
struct LinesAsyncSequence : AsyncSequence {
    struct AsyncIterator : AsyncIteratorProtocol {
        mutating func next() async -> String? { ... }
    }
  
    func makeAsyncIterator() -> AsyncIterator {
        return AsyncIterator()
    }
}

func readLines(_ file: String) -> LinesAsyncSequence { ... }
```

シンタックスハイライトのライブラリを開発していると想定。`Element`の型が`[Token]`で、各行のシンタックスハイライトされたトークンを表現する`SyntaxTokensAsyncSequence`という別の型で戻り値をラップする別の関数を定義したいと考える。

```swift
func readSyntaxHighlightedLines(_ file: String) 
    -> SyntaxTokensAsyncSequence<LinesAsyncSequence> {
      ...
}
```

この時点で、具体的な結果の型はかなり複雑になるので、`some`を使ってopaque result typeの背後にそれを隠したいと思う。

```swift
func readSyntaxHighlightedLines(_ file: String) -> some AsyncSequence {
    ...
}
```

しかし、この`readSyntaxHighlightedLines()`の定義は元のバージョンと比べるとあまり役に立たない。なぜなら、戻り値の`AsyncSequence`の関連型`Element`が`[Token]`であるという要件を表現できないからである。

他の例として、2つの`String`の配列を結合するグローバル関数`concatenate`を考える:

```swift
func concatenate(_ lhs: Array<String>, _ rhs: Array<String>) -> Array<String> {
    ...
}
```

よりジェネリックにすると、こう書くだろう:

```swift
func concatenate<S : Sequence>(_ lhs: S, _ rhs: S) -> S where S.Element == String {
    ...
}
```

この`where`句はジェネリックで複雑な制約を設定できるが、`Array<String>`で具象型でのシンプルな実装と比べるとかなり形が違っており、これを読んだり書く際にはで理解するのに苦労するだろう。これは同じ型要件を具象型と同じように書ける方法があればうれしい。
### 解決策

同じ型要件をプロトコルの「主要な関連型」としてプロトコルの準拠要件と一緒に宣言できる新しいシンタックスを導入する。これはジェネリックの型パラメータに具象型を適用するのと似ていて、`AsyncSequence<String>`や`AsyncSequence<[Lines]>`のように書くことができ、`Array<String>`や`Array<[Lines]>`と同じ感覚と理解を構築できる。

プロトコルにもジェネリックパラメータのリストに似たシンタックスで一つ以上の関連型を宣言できる:

```swift
protocol AsyncSequence<Element> {
    associatedtype Iterator : AsyncIteratorProtocol
        where Element == Iterator.Element
    ...
}

protocol DictionaryProtocol<Key : Hashable, Value> {
    ...
}
```

主要な関連型を持つプロトコルは、プロトコル準拠要件をこれまで書くことができた任意の位置から、山かっこ(`<>`)で囲まれた型パラメータリストを使用して参照できる。

例えば、opaque result typeを主要な関連型を制約できるようになった:

```swift
func readSyntaxHighlightedLines(_ file: String) -> some AsyncSequence<[Token]> {
    ...
}
```

`concatenate`関数も下記のように書くことができる:

```swift
func concatenate<S : Sequence<String>>(_ lhs: S, _ rhs: S) -> S {
    ...
}
```

主要な関連型は、通常、呼び出し側によって提供される関連型に使用することを目的としている。これらの関連型は、準拠した型のジェネリックパラメータとしてよく見られる。例えば、`Array<Element>`と`Set<Element>`はどちらも`Sequence`に準拠しており、`Element`の関連型はジェネリックパラメータとして見られるため、`Element`は`Sequence`の主要な関連型の候補となるのは自然である。これにより、`Sequence<Int>`と、`Array<Int>`、`Set<Int>`の間に明確な類似性ができる。

### 詳細

プロトコル宣言では、山かっこで区切られたオプションの主要な関連型リストをプロトコル名の後に続けることができる。存在する場合、少なくとも1つの主要な関連型を宣言する必要がある。複数の主要な関連型はコンマで区切らる。各主要な関連型は、オプションで継承句を宣言できる。正式な文法は次のように修正され、オプションの主要な関連型プリストの生成がプロトコル宣言に追加される。

- protocol-declaration → attributes<sub>opt</sub> access-level-modifier<sub>opt</sub> `protocol` protocol-name primary-associated-type-list<sub>opt</sub> type-inheritance-clause<sub>opt</sub> generic-where-clause<sub>opt</sub> protocol-body
- primary-associated-type-list → `<` primary-associated-type | primary-associated-type `,` primary-associated-type-list `>`
- primary-associated-type → type-name typealias-assignment<sub>opt</sub>
- primary-associated-type → type-name `:` type-identifier typealias-assignment<sub>opt</sub>
- primary-associated-type → type-name `:` protocol-composition-type default-witness<sub>opt</sub>
- default-witness → `=` type

いくつかの例:

```swift
protocol SetProtocol<Element: Hashable> {
    ...
}

protocol PersistentSortedMap<Key: Comparable & Codable, Value : Codable> {
    ...
}
```

通常の関連型宣言と同様に、デフォルトの型を提供できる:

```swift
protocol GraphProtocol<Vertex: Equatable = String> {}
```

主要な関連型の追加要件は、プロトコルまたは別の関連型の`where`句を使用して記述できる。継承句のシンタックスは、次のようになる:

```swift
protocol SetProtocol<Element> where Element: Hashable {
    ...
}
```

利用側では、*制約付きプロトコル*は、`P<Arg1、Arg2...>`のような一つ以上の型パラメータを使用して記述できるようになった。主要な関連型の数よりも少ない数でもOK。後続の主要な関連型は制約されない。主要な関連型リストをプロトコルに追加することは、互換性のある変更。プロトコルは、以前のように山かっこなしでの参照もできる。

デフォルトの関連型は準拠に関係し、利用側からデフォルトを提供しないことに注意。例えば、上記の`GraphProtocol`では、制約型`Vertex`は`String`に制約されず、指定しないままになる。

#### 糖衣シンタックスの位置で制約されたプロトコル

制約されたプロトコルのシンタックスが現れる可能性のある位置の完全なリストは次の通り。最初の一連のケースでは、新しいシンタックスは主要な関連型を制約する同じ型要件を持つ既存の`where`句のシンタックスと同等。

- extensionで拡張された型

```swift
extension Collection<String> { ... }

// 同等:
extension Collection where Element == String { ... }
```

- 他のプロトコルの継承句

```swift
protocol TextBuffer : Collection<String> { ... }

// 同等:
protocol TextBuffer : Collection where Element == String { ... }
```

- ジェネリックパラメータの継承句

```swift
func sortLines<S : Collection<String>>(_ lines: S) -> S

// 同等:
func sortLines<S : Collection>(_ lines: S) -> S
    where S.Element == String
```

- 関連型の継承句

```swift
protocol Document {
    associatedtype Lines : Collection<String>
}

// 同等:
protocol Document {
    associatedtype Lines : Collection
        where Lines.Element == String
}
```

- `where`句の準拠要件の右側

```swift
func mergeFiles<S : Sequence>(_ files: S)
    where S.Element : AsyncSequence<String>

// 同等:
func mergeFiles<S : Sequence>(_ files: S)
    where S.Element : AsyncSequence, S.Element.Element == String
```

- opaqueパラメータ宣言

```swift
func sortLines(_ lines: some Collection<String>)

// 同等:
func sortLines <C : Collection<String>>(_ lines: C)

// これも同等:
func sortLines <C : Collection>(_ lines: C)
    where C.Element == String
```

- プロトコル型の引数にネストしたopaqueパラメータ宣言を含めることができる

```swift
func sort(elements: inout some Collection<some Equatable>)

// Equivalent to:
func sort<C : Collection, E : Equatable>(elements: inout C)
    where C.Element == E
```

上記のうちのいずれかの位置から参照されている場合、準拠要件の`T : P<Arg1, Arg2...>`は一つ以上の同じ型要件が続く`T: P`という準拠要件に分解される

```swift
T: P
T.PrimaryType1 == Arg1
T.PrimaryType2 == Arg2
...
```

右側の`Arg1`がopaqueパラメータ宣言の場合、同じ型要件の右側で使用するために、新しいジェネリックパラメータが導入される。詳細は[SE-0341 Opaque Parameter Declarations](https://github.com/apple/swift-evolution/blob/main/proposals/0341-opaque-parameters.md)

#### opaque result typeの制約付きプロトコル

制約のあるプロトコルは`some`で特定されるopaque result typeで出てくるかもしれない。この場合、シンタックスはopaque result typeに`where`句を書くことができなかったために以前は書くことができなかったことができる:

```swift
func transformElements<S : Sequence<E>, E>(_ lines: S) -> some Sequence<E>
```

この例では、引数が外部スコープからのジェネリックパラメータに依存できることを示している。

[SE-0328 Structural Opaque Result Types](https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md)でm、戻り値に複数の`some`を使うことができるようになった。これは制約付きプロトコル型に汎用的に使用可能にし、その制約は別のopaque result type不透明な結果タイプになる可能性がある:

```swift
func transform(_: some Sequence<some Equatable>) -> some Sequence<some Equatable>
```

上記では、opaque result typeの`some Sequence<some Equatable>`は、opaqueパラメータ型とは無関係の`some Sequence<some Equatable>`である。パラメータの型は呼び出し側が提供する。opaque result typeは、（おそらく異なる）均一な要素の一連の要素であり、 要素の型は`some Equatable`に準拠することがわかっているがそれ以外の点では呼び出し側には不透明である。

#### 他に使える位置

制約付きプロトコルが表示される可能性がある3つの場所がある:

- 具体型の継承句

```swift
struct Lines : Collection<String> { ... }
```

この位置では、タイプエイリアスを明示的に宣言するのと同様に、関連型を指定するための糖衣シンタックスである.

- タイプエイリアスの基底型として:

```swift
typealias SequenceOfInt = Sequence<Int>
```

タイプエイリアスは、制約付きプロトコル型自体が使用される任意の位置に使用できる。

- プロトコル構成のメンバとして、プロトコル構成が制約付きプロトコル型が妥当な任意の位置に表示される場合:

```swift
func takeEquatableSequence(_ seqs: some Sequence<Int> & Equatable)
```

#### 非サポート位置

このシンタックスを一般化して存在型(Existential)にも有効にすることは自然である。例: `any Collection<String>`。これは、型変換の挙動を慎重に検討する必要がある大きな機能。メタデータと動的キャストの実行時サポートも必要になる。そのため、別の提案によってカバーされる予定。

### その他の代替案

#### 関連型の名前を必須にする 例: Collection<.Element == String>

関連型名を明示的に山かっこ内に書いて制約するといくつかの利点がある

- プロトコル宣言で特別なシンタックスを必要としない
- 明示的な関連型名は、任意の関連型を制限することを可能にする

このアプローチにはいくつかの欠点もある:

- プロトコルの宣言に関連型に関する視覚的な手がかりがない
- 利用側はうんざりするかもしれない。一つのみの主要な関連型を持つプロトコルの場合、その名前を指定の強制が不必要に繰り返される
- 制約付き関連型が宣言のジェネリックパラメータと同じ名前を持つ場合、シンタックスは混乱を招く可能性がある。例えば、次のように:

```swift
func adjacentPairs<Element>(_: some Sequence<Element>,
                            _: some Sequence<Element>)
-> some Sequence<(Element, Element)>
```

これは、名前を付ける代替手段よりも読みやすい:

```swift
 func adjacentPairs<Element>(_: some Sequence<.Element == Element>,
                            _: some Sequence<.Element == Element>)
-> some Sequence<.Element == (Element, Element)>
```

- このより詳細なシンタックスは、既存のシンタックスに対する明確な改善がない。ここでは、`where`句のほとんどは明示的に書かれている。また、`where`句の代わりにジェネリックシグネチャの前の山かっこ内に括弧内に、ほとんどまたは全てのジェネリック制約を指定することを促し、[SE-0081 Move where clause to end of declaration.](https://github.com/apple/swift-evolution/blob/main/proposals/0081-move-where-expression.md)の主な信条に違反する

- 最後に、このシンタックスは、具象型とジェネリクスの間の対称性を欠いている。`Array<Int>`からの汎用化には、単なる`some Collection<Int>`の代わりに、`some Collection<.Element == Int>`という新しいシンタックスを学習および書く必要がある。

このプロポーザルは、将来上記のシンタックスを追加することを妨げるものは何もありません。先頭のドット(または他の記号)が存在することで、どちらの場合も明確に解析することができる

#### 最初にopaque result type要件のよりジェネリックなシンタックスを実装

前述のように、opaque result typeの場合、主要な関連型の同じ型要件を記述できる`where`句を含めることができないため、このプロポーザルは新しい表現力を導入している。

最初に、opaque result typeの汎用的な要件を可能にする言語機能を導入することも可能。可能性の1つとしては、「名前付きopaque result type」。これは、`where`句で要件を設定することができる。

```swift
func readLines(_ file: String) -> some AsyncSequence<String> { ... }

// 同等:
func readLines(_ file: String) -> <S> S
  where S : AsyncSequence, S.Element == String { ... }
```

ただし、このプロポーザルの目的は、具象型とジェネリクスの間に対称性を導入することでジェネリクスをより親しみやすくし、ジェネリクスを他の言語のプログラマがすでに慣れ親しんでいる汎用化のように感じさせること。

opaque result typeのよりジェネリックなシンタックスは、それ自体メリットと見なすことができる。前のセクションで説明した`some Collection<.Element == Int>`シンタックスと同様に、このプロポーザルでは、将来opaque result typeをさらに汎用化することを妨げるものはない。

### 通常のassociatedtype宣言に主要な関連型のアノテーションを付ける

ある種の修飾子を`associatedtype`宣言に追加すると、これは、ジェネリック型がパラメータを宣言する方法ともまた異なるため、段階的開示の原則に反し、APIの利用者の複雑さが増す。また、このプロポーザルを複数の主要な関連型を一般化するようにした場合、将来的には、利用側で主要な関連型の宣言順序を理解する必要がある。

また、プロトコル定義のメンバには現在当てはまらない方法で、宣言の順序を重大なものにしてしまう。

関連型宣言にアノテーションを付けると、新しいコンパイラバージョンでのみ主要な関連型を定義するプロトコルを条件付きで宣言するのが簡単になる。このプロポーザルで説明されているシンタックスは、プロトコル宣言自体に適用され、結果として、下位互換性のある方法でこの機能を採用したいライブラリは、`#if`ブロックの背後にあるプロトコル定義全体を複製する必要がある:

```swift
#if swift(>=5.7)
protocol SetProtocol<Element : Hashable> {
    var count: Int { get }
    ...
}
#else
protocol SetProtocol {
    associatedtype Element : Hashable

    var count: Int { get }
    ...
}
#endif
```

仮に`primary`キーワードが存在した場合、そこだけ重複が必要になる:

```swift
protocol SetProtocol {
#if swift(>=5.7)
    primary associatedtype Element : Hashable
#else
    associatedtype Element : Hashable
#endif
    var count: Int { get }
    ...
}
```

ただし、この方法で関連型宣言を複製することは、依然としてエラーが発生しやすい形式のコード複製で、コードが読みにくくなる。このユースケースは、言語シンタックスの進化を不必要に妨げるものであってはならないと私たちは感じている。古いコンパイラとの互換性を維持しながら新しい言語機能を採用するライブラリの懸念は、このプロポーザルに固有のものではなく、サードパーティのプリプロセッサツールを使用して対処するのが最適。

#### ジェネリックプロトコル(Generic protocols)

このプロポーザルでは、Haskellのマルチパラメータ型クラスまたはRustのジェネリック特性をモデルにした架空の「ジェネリックプロトコル」機能の代わりに、主要な関連型を制約するために山かっこシンタックスを使用する。 このような「ジェネリックプロトコル」は、単一の`Self`準拠型だけでなく、複数の型にわたってパラメータ化できるという考え方である:

```swift
protocol ConvertibleTo<Other> {
    static func convert(_: Self) -> Other
}

extension String : ConvertibleTo<Int> {
    static func convert(_: String) -> Int
}

extension String : ConvertibleTo<Double> {
    static func convert(_: String) -> Double
}
```

主要な関連型の制約は、ジェネリックプロトコルよりも汎用的に有用な機能で、主要な関連型の制約に山かっこシンタックスを使用すると、`Array<Int>`と`Collection<Int>`の明確な類似性により、ユーザが一般的に期待するものが得られると考えている。

このプロポーザル提案には、将来、異なるシンタックスでジェネリックプロトコルを導入することを妨げるものはない。おそらく、関連型の場合のように、型パラメータ間に機能依存性(functional dependency)がないことを明確にするために、`Self`型を他の型よりも優先しないものになる:

```swift
protocol Convertible(from: Self, to: Other) {
    static func convert(_: Self) -> Other
}

extension Convertible(from: String, to: Int) {
    static func convert(_: String) -> Int
}

extension Convertible(from: String, to: Double) {
    static func convert(_: String) -> Int
}
```

### ソース互換性

このプロポーザルは、既存のソースの互換性に影響を与えない。この機能を採用するプロトコルの場合、主要な関連型を削除または変更することは、クライアントにとって重大な変更になる。

### ABIの安定性への影響

この変更は、既存のコードのABIの安定性には影響しません。この新機能は実行時サポートを必要とせず、既存のSwiftランタイムに逆展開可能。

### APIのレジリエンスへの影響

この変更は、APIのレジリエンスには影響しない。この機能を採用するプロトコルの場合、主要な関連型リストの追加または削除はバイナリ間で互換可能な変更である。主要な関連型リストの変更または削除もバイナリ間で互換可能な変更だが、ソースを壊すため推奨されない。

### 将来の検討事項

#### 標準ライブラリへの導入

標準ライブラリで主要な関連型を実際に採用することは、このプロポーザルの範囲外。`Sequence`や`Collection`などの明らかな候補があるが、追加の議論が必要になることは間違いない。

#### 制約付き存在型

上記のように、このプロポーザルだけでは、`Collection<String>`などの制約付きプロトコルを存在型として扱うことはできない。

## 参考リンク

### Forums

- [[Pitch] Light-weight same-type requirement syntax](https://forums.swift.org/t/pitch-light-weight-same-type-constraint-syntax/52889)
- [[Pitch 2] Light-weight same-type requirement syntax](https://forums.swift.org/t/pitch-2-light-weight-same-type-requirement-syntax/55081)
- [SE-0346: Lightweight same-type requirements for primary associated types](https://forums.swift.org/t/se-0346-lightweight-same-type-requirements-for-primary-associated-types/55869)
- [Improving the UI of generics](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--directly-expressing-constraints)
- [New syntax for declaring primary associated types](https://github.com/apple/swift/pull/41640)


### プロポーザルドキュメント

- [Lightweight same-type requirements for primary associated types](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md#require-associated-type-names-eg-collectionelement--string)
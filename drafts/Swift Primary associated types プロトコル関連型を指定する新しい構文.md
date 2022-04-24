# Swift Primary associated types プロトコル関連型を指定する新しい構文

- [Swift Primary associated types プロトコル関連型を指定する新しい構文](#swift-primary-associated-types-プロトコル関連型を指定する新しい構文)
  - [概要](#概要)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決策](#解決策)
    - [詳細](#詳細)
      - [糖衣構文(シンタックスシュガー)内での条件付きプロトコル](#糖衣構文シンタックスシュガー内での条件付きプロトコル)
      - [Opaque Result Typeの条件付きプロトコル](#opaque-result-typeの条件付きプロトコル)
      - [他に使える位置](#他に使える位置)
      - [サポートしていない位置](#サポートしていない位置)
    - [その他の代替案](#その他の代替案)
      - [主要関連型リストのエントリを宣言として扱う](#主要関連型リストのエントリを宣言として扱う)
      - [関連型の名前を必須にする 例: Collection<.Element == String>](#関連型の名前を必須にする-例-collectionelement--string)
      - [最初にOpaque Result Type要件のよりジェネリックな構文を実装](#最初にopaque-result-type要件のよりジェネリックな構文を実装)
    - [通常のassociatedtype宣言に主要関連型のアノテーションを付ける](#通常のassociatedtype宣言に主要関連型のアノテーションを付ける)
      - [ジェネリックプロトコル(Generic protocols)](#ジェネリックプロトコルgeneric-protocols)
    - [ソース互換性](#ソース互換性)
    - [ABIの安定性への影響](#abiの安定性への影響)
    - [APIのレジリエンスへの影響](#apiのレジリエンスへの影響)
    - [将来の検討事項](#将来の検討事項)
      - [標準ライブラリへの導入](#標準ライブラリへの導入)
      - [条件付き存在型](#条件付き存在型)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

[Improving the UI of generics](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--directly-expressing-constraints)を実現するためのステップとして、このプロポーザルではジェネリックパラメータに準拠し、同型要件(※)を介して関連型(associatedtype)を制約するための新しい構文を導入する。

※ 同型要件とは以下のようなものを指す

```swift
Collection where Element == String
```

## 内容

### 問題点

ソースファイルの各行を`Sequence`として取得する関数を考える:

```swift
struct LinesSequence : Sequence {
    struct Iterator : IteratorProtocol {
        mutating func next() async -> String? { ... }
    }
  
    func makeIterator() -> Iterator {
        return Iterator()
    }
}

func readLines(_ file: String) -> LinesSequence { ... }
```

シンタックスハイライト(構文強調)するライブラリを開発していると想定。`Element`の型が`[Token]`で、各行のシンタックスハイライトされたトークンの配列を示す`SyntaxTokensSequence`で戻り値をラップした別の関数を定義したいとする。

```swift
func readSyntaxHighlightedLines(_ file: String) 
    -> SyntaxTokensSequence<LinesSequence> {
      ...
}
```

この時点で、具体的な結果の型はかなり複雑になるので、`some`を使ってOpaque Result Typeで具体的な型を隠したいと思う。

```swift
func readSyntaxHighlightedLines(_ file: String) -> some Sequence {
    ...
}
```

しかし、この`readSyntaxHighlightedLines()`の定義は元のバージョンと比べるとあまり有用ではない。なぜなら、戻り値の`Sequence`の関連型の`Element`が`[Token]`であるという要件を表現できないからである。

他の例として、2つの`String`の配列を結合するグローバル関数`concatenate`を考える:

```swift
func concatenate(_ lhs: Array<String>, _ rhs: Array<String>) -> Array<String> {
    ...
}
```

よりジェネリックにすると、こう書くと思われる:

```swift
func concatenate<S : Sequence>(_ lhs: S, _ rhs: S) -> S where S.Element == String {
    ...
}
```

この`where`句はジェネリックで複雑な制約を設定できるが、`Array<String>`の場合の具体的な型でのシンプルな実装と比べると、かなり形が異なり、これを読み書きする際には理解するのにおそらく苦労する。こういった場合に同型要件を具体的な型と同じように書ける方法があればうれしい。

### 解決策

一つ以上の同型要件をプロトコルの「主要関連型」(primary associated types)としてプロトコルの準拠要件と一緒に宣言できる新しい構文を導入する。これはジェネリックの型パラメータに具体的な型を適用するのと似ていて、`Sequence<String>`や`Sequence<[Lines]>`のように書くことができ、`Array<String>`や`Array<[Lines]>`と同じ感覚で理解できる。

プロトコルにジェネリックパラメータのリストに似た構文で一つ以上の関連型を宣言できる:

```swift
protocol Sequence<Element> {
    associatedtype Iterator : IteratorProtocol
        where Element == Iterator.Element
    ...
}

protocol DictionaryProtocol<Key : Hashable, Value> {
    associatedtype Key : Hashable
    associatedtype Value
    ...
}
```

主要関連型を持つプロトコルは、プロトコルへの準拠要件をこれまで書くことができた場所ならばどこでも、山かっこ(`<>`)で囲まれた型パラメータリストを使用して同じように書くことができる。

例えば、Opaque Result Typeを主要関連型を制約できるようになった:

```swift
func readSyntaxHighlightedLines(_ file: String) -> some Sequence<[Token]> {
    ...
}
```

`concatenate`関数も下記のように書くことができる:

```swift
func concatenate<S : Sequence<String>>(_ lhs: S, _ rhs: S) -> S {
    ...
}
```

主要関連型は、通常、呼び出し側によって提供される関連型に使用されることを目的としている。これらの関連型は、準拠した型のジェネリックパラメータとしてよく現れる。例えば、`Array<Element>`と`Set<Element>`はどちらも`Sequence`に準拠しており、`Element`関連型はそれに応じた具体的な型としてジェネリックパラメータに現れるため、`Element`が`Sequence`の主要関連型の候補となるのは自然である。これにより、制約のあるプロトコル`Sequence<Int>`と、具体的な型の`Array<Int>`、`Set<Int>`の間に明確な対応関係ができる。

### 詳細

プロトコル宣言では、山かっこで区切られた任意の主要関連型リストをプロトコル名の後に続けることができる。山かっこがある場合、少なくとも一つの主要関連型を宣言する必要がある。複数のある場合はカンマで区切る。リスト内の各主要関連型は、プロトコル本文、またはそのプロトコルを継承したプロトコルの内の一つの本文内で宣言されている既に存在する関連型を指定しなければならない。正式な文法は次のように修正され、任意の主要関連型リストの生成がプロトコル宣言に追加される。

- protocol-declaration → attributes<sub>opt</sub> access-level-modifier<sub>opt</sub> `protocol` protocol-name primary-associated-type-list<sub>opt</sub> type-inheritance-clause<sub>opt</sub> generic-where-clause<sub>opt</sub> protocol-body
- **primary-associated-type-list** → `<` primary-associated-type-entry `>`
- **primary-associated-type-entry** → primary-associated-type | primary-associated-type `,` primary-associated-type-type
- **primary-associated-type** → type-name

いくつかの例:

```swift
protocol SetProtocol<Element: Hashable> {
    associatedtype Element : Hashable
    ...
}

protocol SortedMap {
    associatedtype Key
    associatedtype Value
}

// 主要な関連のKeyとValueは継承したSortedMapの中で宣言されている
protocol PersistentSortedMap<Key, Value> : SortedMap {
}
```

利用側では、*条件付きプロトコル*を`P<Arg1、Arg2...>`のように、一つ以上の型パラメータを使って記述できるようになった。型パラメータのリストを完全に省略することができ、この場合はプロトコルには何の制約もない。主要関連型の数よりも少ないまたは多い数の型パラメータを指定するとエラーになる。制約を付けない状態では、プロトコルは以前のように山かっこなしでも参照できるため、主要関連型リストをプロトコルに追加してもソース互換性のある変更である。なぜならな

#### 糖衣構文(シンタックスシュガー)内での条件付きプロトコル

条件付きプロトコルの構文が存在する可能性のある位置の完全なリストは次の通り。最初の一連のケースでは、新しい構文は主要関連型を制約する同型要件を持つ既存の`where`句の構文と等しい。

- extensionで拡張された型

```swift
extension Collection<String> { ... }

// 等しい:
extension Collection where Element == String { ... }
```

- 他のプロトコルからの継承句

```swift
protocol TextBuffer: Collection<String> { ... }

// 等しい:
protocol TextBuffer: Collection where Element == String { ... }
```

- ジェネリックパラメータの継承句

```swift
func sortLines<S: Collection<String>>(_ lines: S) -> S

// 等しい:
func sortLines<S: Collection>(_ lines: S) -> S
    where S.Element == String
```

- 関連型の継承句

```swift
protocol Document {
    associatedtype Lines: Collection<String>
}

// 等しい:
protocol Document {
    associatedtype Lines: Collection
        where Lines.Element == String
}
```

- `where`句の準拠要件の右側

```swift
func merge<S: Sequence>(_ files: S)
    where S.Element: Sequence<String>

// 等しい:
func merge<S: Sequence>(_ files: S)
    where S.Element: Sequence, S.Element.Element == String
```

- Opaqueパラメータ宣言

```swift
func sortLines(_ lines: some Collection<String>)

// 等しい:
func sortLines <C: Collection<String>>(_ lines: C)

// これも等しい:
func sortLines <C: Collection>(_ lines: C)
    where C.Element == String
```

- プロトコル型の引数に、ネストしたOpaqueパラメータ宣言を含めることができる

```swift
func sort(elements: inout some Collection<some Equatable>)

// 等しい:
func sort<C: Collection, E: Equatable>(elements: inout C)
    where C.Element == E
```

上記のうちのいずれかの位置から参照されている場合、準拠要件の`T: P<Arg1, Arg2...>`は一つ以上の同型要件が続く`T: P`という準拠要件に分解される

```swift
T: P
T.PrimaryType1 == Arg1
T.PrimaryType2 == Arg2
...
```

右側の`Arg1`がOpaqueパラメータ宣言の場合、同型要件の右側で使用できるようにするために、新しいジェネリックパラメータが導入された。詳細は[SE-0341 Opaque Parameter Declarations](https://github.com/apple/swift-evolution/blob/main/proposals/0341-opaque-parameters.md)

#### Opaque Result Typeの条件付きプロトコル

条件付きプロトコルは`some`で特定されるOpaque Result Typeで表示される可能性がある。以前はOpaque Result Typeに`where`句を書くことができなかったためにこういったケースを書くことができなかったが、これができるようになる:

```swift
func transformElements<S: Sequence<E>, E>(_ lines: S) -> some Sequence<E>
```

この例では、Opaque Result Typeのパラメータが外部スコープから指定されたジェネリックパラメータに依存できることも示している。

[SE-0328 Structural Opaque Result Types](https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md)で、戻り値に複数の`some`を使うことができるようになった。これは条件付きプロトコル型を汎用的に使用可能にし、その制約は別のOpaque Result Typeになる可能性がある:

```swift
func transform(_: some Sequence<some Equatable>) -> some Sequence<some Equatable>
```

上記のOpaque Result Typeの`some Sequence<some Equatable>`は、Opaqueパラメータ型とは無関係の`some Sequence<some Equatable>`であることに注目。パラメータの型は呼び出し側が提供する。Opaque Result Typeは、(異なる場合もあるが)均一な要素の一連の要素であり、要素の型は`some Equatable`に準拠することがわかっているが、それ以外の点では呼び出し側には何も見えない。

#### 他に使える位置

他にも条件付きプロトコルが出てくる可能性がある箇所が3つある:

- 具体型の継承句

```swift
struct Lines: Collection<String> { ... }
```

この位置では、typealiasを明示的に宣言するのと同様に、関連型を指定するための糖衣構文である.

- typealiasの基底型として:

```swift
typealias SequenceOfInt = Sequence<Int>
```

typealiasは、条件付きプロトコル型自体が使用される任意の位置に使用できる。

- プロトコル合成の一部として、プロトコル合成が条件付きプロトコルを使うことができる任意の位置に表示される場合:

```swift
func takeEquatableSequence(_ seqs: some Sequence<Int> & Equatable)
```

#### サポートしていない位置

この構文を一般化して存在型(Existential)にも有効にすることは自然である。例: `any Collection<String>`。これは、型変換の挙動を慎重に検討する必要がある大きな機能になる。メタデータと動的キャストの実行時サポートも必要。そのため、別の提案によってカバーされる予定。

### その他の代替案

#### 主要関連型リストのエントリを宣言として扱う

このプロポーザルの以前のバージョンでは、主要関連型リストのエントリは、本文で宣言された関連型に名前を付ける代わりに、新しい関連型を宣言していた。つまり、このように書けた:

```swift
protocol SetProtocol<Key> {
    ...
}
```

現在は`associatedtype`の宣言が必要

```swift
protocol SetProtocol<Key : Hashable> {
    associatedtype Key : Hashable
    ...
}
```

主要関連型リストで関連型の宣言を許可すると、重要な点で意味的に異なる場合に、ここで実際に行われているのはジェネリックの宣言であるという混乱を招くと感じた。もう一つの潜在的な混乱を招く要因は、主要関連型がデフォルト型を宣言していた場合:

```swift
protocol SetProtocol<Key: Hashable = String> {
    ...
}
```

これは、「デフォルト付きジェネリックパラメータ」のように見えるが、そうではない。`SetProtocol`を書くことは、`Key`を制約がないことを意味し、`SetProtocol<Int>`と同じではない。現在提案されている形式だと、ここで行われているのが、利用側のジェネリック制約ではなく、プロトコル準拠にデフォルトが宣言されていることをより明確にする:

```swift
protocol SetProtocol<Key> {
    associatedtype Key : Hashable = String
    ...
}
```

#### 関連型の名前を必須にする 例: Collection<.Element == String>

関連型名を明示的に山かっこ内に書いて制約するといくつかの利点がある

- プロトコル宣言で特別な構文を必要としない
- 明示的な関連型名は、任意の関連型を制限することを可能にする

このアプローチにはいくつかの欠点もある:

- プロトコルの宣言に関連型に関する視覚的な手がかりがない
- 利用側は書くのにうんざりするかもしれない。一つのみの主要関連型を持つプロトコルの場合、その名前を指定の強制が不必要に繰り返される
- 制約付き関連型が宣言のジェネリックパラメータと同じ名前を持つ場合、構文は混乱を招く可能性がある。例えば、次のように:

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

- このより詳細な構文は、既存の構文に対する明確な改善がない。ここでは、`where`句のほとんどは明示的に書かれている。また、`where`句の代わりにジェネリックシグネチャの前の山かっこ内に括弧内に、ほとんどまたは全てのジェネリック制約を指定することを促し、[SE-0081 Move where clause to end of declaration.](https://github.com/apple/swift-evolution/blob/main/proposals/0081-move-where-expression.md)の主な信条に違反する

- 最後に、この構文は、具体的な型とジェネリクスの間の対称性を欠いている。`Array<Int>`からの汎用化には、単なる`some Collection<Int>`の代わりに、`some Collection<.Element == Int>`という新しい構文を学習および書く必要がある。

このプロポーザルは、将来上記の構文を追加することを妨げるものは何もありません。先頭のドット(または他の記号)が存在することで、どちらの場合も明確に解析することができる

#### 最初にOpaque Result Type要件のよりジェネリックな構文を実装

前述のように、Opaque Result Typeの場合、主要関連型の同型要件を記述できる`where`句を含めることができないため、このプロポーザルは新しい表現を導入している。

最初に、Opaque Result Typeの汎用的な要件を可能にする言語機能を導入することも可能。可能性の一つとしては、「名前付きOpaque Result Type」。これは、`where`句で要件を設定することができる。

```swift
func readLines(_ file: String) -> some Sequence<String> { ... }

// 同等:
func readLines(_ file: String) -> <S> S
  where S : Sequence, S.Element == String { ... }
```

ただし、このプロポーザルの目的は、具体的な型とジェネリクスの間に対称性を導入することで、ジェネリクスをよりわかりやすくし、他の言語のプログラマがすでに慣れ親しんでいる汎用化の方法と同じような感覚で使用できるようにすること。

Opaque Result Typeのより汎用的な構文は、それ自体メリットと見なすことができる。前のセクションで説明した`some Collection<.Element == Int>`構文と同様に、このプロポーザルでは、将来Opaque Result Typeをさらに汎用化することを妨げるものではない。

### 通常のassociatedtype宣言に主要関連型のアノテーションを付ける

※ これはおそらく以前の一つしか主要関連型が宣言できなかった頃の内容であると思われる。

ある種の修飾子を`associatedtype`宣言に追加すると、これは、ジェネリック型がパラメータを宣言する方法ともまた異なるため、段階的開示の原則に反し、APIの利用者の複雑さが増す。また、このプロポーザルを複数の主要関連型へと一般化するようにした場合、将来的には、利用側で主要関連型の宣言順序を理解する必要がある。

また、プロトコル定義のメンバでは必要のない、宣言の順序に重大な意味を与えてしまう(順番依存が生まれる)。

関連型宣言にアノテーションを付けると、新しいコンパイラバージョンでのみ主要関連型を定義するプロトコルを条件付きで宣言するのが簡単になる。このプロポーザルで説明されている構文は、プロトコル宣言自体に適用され、結果として、下位互換性のある方法でこの機能を採用したいライブラリは、`#if`ブロックの背後にあるプロトコル定義全体を複製する必要がある:

```swift
#if swift(>=5.7)
protocol SetProtocol<Element> {
    associatedtype Element : Hashable
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

ただし、この方法で関連型宣言を複製することは、依然としてエラーが発生しやすい形式のコードの複製で、コードが読みにくくなる。このユースケースは、言語構文の進化を不必要に妨げるものであってはならないと私たちは感じている。古いコンパイラとの互換性を維持しながら新しい言語機能を採用するライブラリの懸念は、このプロポーザルに固有のものではなく、サードパーティのプリプロセッサツールを使用して対処するのが最適。

現在提案されている構文では、プリプロセッサがプロトコル名の後の山かっこの間のすべてを取り除き、下位互換性のある宣言を生成するには十分。このようなプリプロセッサの最小限の実装は、単純な`sed`呼び出し。`sed -e 's/\(protocol .*\)<.*> {/\1 {/'`。

#### ジェネリックプロトコル(Generic protocols)

このプロポーザルでは、Haskellのマルチパラメータ型クラスまたはRustのジェネリック特性をモデルにした架空の「ジェネリックプロトコル」機能の代わりに、主要関連型を制約するために山かっこ構文を使用する。このような「ジェネリックプロトコル」は、単一の`Self`準拠型だけでなく、複数の型にわたってパラメータ化できるという考え方である:

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

主要関連型の制約は、ジェネリックプロトコルよりも汎用的に有用な機能で、主要関連型の制約に山かっこ構文を使用すると、`Array<Int>`と`Collection<Int>`の明確な類似性により、ユーザが一般的に期待するものが得られると考えている。

このプロポーザルには、将来、異なる構文でジェネリックプロトコルを導入することを妨げるものはない。おそらく、関連型の場合のように、型パラメータ間に機能依存性(functional dependency)がないことを明確にするために、`Self`型を他の型よりも優先しないものになる:

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

このプロポーザルは、既存のソースの互換性に影響を与えない。

主要関連型を追加することは、既存のプロトコルと互換性のある変更である。

以下はソース破壊を起こす変更になる

- 既存のプロトコルから主要関連型を削除する
- 既存のプロトコルの主要関連型の順番や内容を変更する

### ABIの安定性への影響

この変更は、既存のコードのABIの安定性には影響しない。この新機能は実行時サポートを必要とせず、既存のSwiftランタイムにback deploy可能。

主要関連型はABIの一部ではないため、次の全てはバイナリ互換可能な変更になる:

- 既存のプロトコルから主要関連型を追加する
- 既存のプロトコルから主要関連型を削除する
- 既存のプロトコルの主要関連型を変更する

### APIのレジリエンスへの影響

この変更は、APIのレジリエンスには影響しない。この機能を採用するプロトコルの場合、主要関連型リストの追加または削除はバイナリ間で互換可能な変更である。主要関連型リストの変更または削除もバイナリ間で互換可能な変更だが、ソースを壊すため推奨されない。

### 将来の検討事項

#### 標準ライブラリへの導入

標準ライブラリで主要関連型を実際に採用することは、このプロポーザルの範囲外である。`Sequence`や`Collection`などの明らかな候補があるが、追加の議論が必要になることは間違いない。

#### 条件付き存在型

上記で言ったように、このプロポーザルでは、`Collection<String>`などの条件付きプロトコルを存在型として扱うことはできない。

## 参考リンク

### Forums

- [[Pitch] Light-weight same-type requirement syntax](https://forums.swift.org/t/pitch-light-weight-same-type-constraint-syntax/52889)
- [[Pitch 2] Light-weight same-type requirement syntax](https://forums.swift.org/t/pitch-2-light-weight-same-type-requirement-syntax/55081)
- [SE-0346: Lightweight same-type requirements for primary associated types](https://forums.swift.org/t/se-0346-lightweight-same-type-requirements-for-primary-associated-types/55869)
- [SE-0346 (second review): Lightweight same-type requirements for primary associated types](https://forums.swift.org/t/se-0346-second-review-lightweight-same-type-requirements-for-primary-associated-types/56414)
- [Improving the UI of generics](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--directly-expressing-constraints)
- [New syntax for declaring primary associated types](https://github.com/apple/swift/pull/41640)
- [[Pitch] Primary Associated Types in the Standard Library](https://forums.swift.org/t/pitch-primary-associated-types-in-the-standard-library/56426)
- [[Accepted] SE-0346: Lightweight same-type requirements for primary associated types](https://forums.swift.org/t/accepted-se-0346-lightweight-same-type-requirements-for-primary-associated-types/56747)


### プロポーザルドキュメント

- [Lightweight same-type requirements for primary associated types](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md#require-associated-type-names-eg-collectionelement--string)
- [Primary Associated Types in the Standard Library](https://github.com/lorentey/swift-evolution/blob/stdlib-pats/proposals/nnnn-primary-associated-types-in-stdlib.md)
- [Amend SE-0329 to add Clock.Duration](https://github.com/apple/swift-evolution/pull/1618)

# Swift 制約付き存在型(any版primary associated types)

収録日: 2022/05/28

- [Swift 制約付き存在型(any版primary associated types)](#swift-制約付き存在型any版primary-associated-types)
  - [概要](#概要)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決策](#解決策)
  - [詳細](#詳細)
    - [制約されたプロトコル型の等価性](#制約されたプロトコル型の等価性)
    - [変性](#変性)
    - [制約された存在型の共変消去](#制約された存在型の共変消去)
  - [ABIへの影響](#abiへの影響)
  - [検討された代替案](#検討された代替案)
  - [将来の検討事項](#将来の検討事項)
    - [ジェネリック制約](#ジェネリック制約)
    - [Opaque制約](#opaque制約)
    - [もっとジェネリックな存在型](#もっとジェネリックな存在型)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

存在型はSwiftで抽象化を表すための型システムの機能を補っている。ジェネリクスと同様に、関数が複数候補のうちの一つの型を受け取ったり、返すことを可能にしている。ジェネリックパラメータとは異なり、存在型は関数の引数で渡された際に事前にその型を知っている必要はない。さらに、関数から返される場合は、具体的な型はインターフェイスに隠れるため、消去されている可能性がある。[SE-0309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)では関連型を持つプロトコルを存在型として使う際の残っていた制限がなくなり、[SE-0346](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md)でプロトコルの関連型へ制約を設けるより軽量なシンタックスが導入された。これらの上に構築する前提で、存在型にも関連型の軽量なシンタックスを再利用できるようにする。

```swift
any Collection<String>
```

根本的には[SE-0346](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md)で導入された`some`に対するシンタックスを`any`でも使えるようにする。

## 内容

### 問題点

[SE-0309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)で関連型を持つプロトコルを存在型として使うことができるようになったものの、その関連型に制約を追加することができず、ジェネリクスと存在型の間にギャップがある。イベントプロデューサとコンシューマの型消去されたスタックの実装を考えてみる:

```swift
protocol Producer {
    associatedtype Event
  
    func poll() -> Self.Event?
}

protocol Consumer {
    associatedtype Event
  
    func respond(to event: Self.Event)
}
```

この仮想のイベントシステムでは、恣意の複数の`Producer`と恣意の複数の`Consumer`を受け入れたいとする場合、存在型を使えば自由にすることができる。

```swift
struct EventSystem {
    var producers: [any Producer]
    var consumers: [any Consumer]
  
    mutating func add(_ producer: any Producer) {
        self.producers.append(producer)
    }
}
```

しかし、プロデューサとコンシューマをお互いに組み合わせることは難しい。`poll`している際に`Producer`は無特定の関連性のない`Event`を生成するので、Swiftはコンシューマが任意のイベントを受け入れることができないことを(正しく)警告する。解決方法としては、イベントの型を汎用化して`EventSystem`を構築することで、`Producer`と`Consumer`のインスタンスがこの特定のイベントを返すようにする。現状では、これはプロデューサとコンシューマを具体的な型に制約することも意味し、型を固定する必要があるという追加の欠点がある。つまり、アドホックな型消去が再び発生する:

```swift
struct EventSystem<Event> {
    var producers: [AnyProducer<Event>]
    var consumers: [AnyConsumer<Event>]
  
    mutating func add<P: Producer>(_ producer: P) 
        where P.Event == Event { 
        self.producers.append(AnyProducer<Event>(erasing: producer)) 
    }
}
```

この例では、型の安全性のためにかなりの犠牲を払っている。また、プロデューサとコンシューマのために2つの追加の型消去ラッパを維持する必要がある。実際、不足しているのは、プロデューサとコンシューマの型は重要ではないが（存在型）、操作するデータは重要である（ジェネリック制約）ということを表現する機能。 これは、制約された存在型が必要なところ。[SE-0346](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md)の主要な関連型を組みさわせることで、これを実現できる:

```swift
struct EventSystem<Event> {
    var producers: [any Producer<Event>]
    var consumers: [any Consumer<Event>]
  
    mutating func add(_ producer: any Producer<Event>) { 
        self.producers.append(producer) 
    }
}
```

### 解決策

存在型は、主要な関連型に制約を指定する機能を拡張できる。そのような制約のある存在型が現れると、それらは同じ型の要件に変換される。

```swift
protocol P<T, U, V> { }

var xs: [any P<B, N, J>] // 次と同等[any P] where P.T == B, P.U == N, P.V == J
```

## 詳細

存在型の構文は、制約句を受け入れるように更新される。型推論手順が更新され、パラメータ化された存在型の一部として表示されるジェネリックパラメータに推論規則が適用される。

Swiftの型システムとランタイムは、パラメータ化された存在型からパラメータ化されていない存在型へのキャスト、およびその逆のキャスト、および制約された主要な関連型を改良するキャストを受け入れる。存在型へのアップキャストやダウンキャスト、存在型間のキャストなどの際に追加の制約が考慮されるようになる:

```swift
var x: any Sequence<T>
_ = x as any Sequence // いつもtrue
_ = x as! any Sequence<String> // 実行時にSequence.Elementを調べる
```

### 制約されたプロトコル型の等価性

言語は、コードで異なる方法で派生した2つの型が、実際には同じ型である場合を定義する必要がある。原則として、2つの制約付きプロトコル型は、可能な準拠型のセットがまったく同じである場合に限り、同じであるとするのは理にかなっている。残念ながら、このルールは、複雑な技術的理由から、Swiftの型システムでは実用的ではない。これは、互いに論理的に同等であるいくつかの制約されたプロトコル型が、Swiftでは異なる型と見なされることを意味する。

正確なルールはまだ決定されていないが、たとえば、これらのプロトコルの関連型が同じだとわかっていても、`any P＆Q<Int>`が`any P<Int>＆Q`と異なると見なされる可能性がある。ただし、これらの型には同等の論理コンテンツがあるため、両方向で暗黙的に変換される。結果として、これは大きな実用上の困難をもたらすとは思われない。

`T ==Int`であるジェネリックコンテキストでの`any P<Int>`および`any P<T>`など、同じ基本的な「形状」で記述された制約付きプロトコル型の置換は、常に同じ型になる。

### 変性

制約付き存在型の主なユースケースの1つは、Swift標準ライブラリの`Collection`型。標準ライブラリの具体的な`Collection`型タイプには、共変の型強制のサポートが組み込まれている。例えば、

```swift
func up(from values: [NSView]) -> [Any] { return values }
```

一見すると、制約された存在型も変性をサポートする必要があるように思われる:

```swift
func up(from values: any Collection<NSView>) -> any Collection<Any> { return values }
```

しかし、これはかなり技術的に困難であることがわかった。入力コレクションを適切な型の`Array`として再キャストするこの強制の内部の実装では、これは非常に驚くべきことだが、`Array`が常に標準ライブラリのABIに永久に返される。

制約された存在型は、変性に関しては、通常のジェネリック型のように動作する。つまり、不変となり、上記のコードはエラーになる。

### 制約された存在型の共変消去

[SE-0309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)は、`Self`またはその関連型への参照が、プロトコルで定義されたメソッドの戻り値の型など、共変位置にある場合にのみ、存在型の値でメンバを使用できるようにした。このような位置では、関連型が上限まで消去される。上限は、関連型の機能を最も厳密に表す型。たとえば、`Collection`の`first`プロパティの使用を検討する:

```swift
extension Collection {}
    var first: Element? { get }
}

func test(collection: any Collection, stringCollection: any Collection<String>) {
    let x = collection.first       // 前はエラー。 SE-0309によってAny?に消去される
    let y = stringCollection.first // SE-0309によって、Element == StringからString?を生成する
}
```

ただし、`Self`またはその関連型が不変の位置([SE-0309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)で定義)で発生する場合、具体的な型がわからない限り、存在型のメンバを使用することはできない。[SE-0309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)では、次の例を提供している

```swift
var collection: any RangeReplaceableCollection = [1, 2, 3]
// エラー: これは反変位置の関連型Elementを参照しているため
// appendはRangeReplaceableCollectionプロトコル型の値を使えない
// 変わりに準拠制約を使用する
collection.append(4)
```

制約された存在型を使用して、`RangeReplaceableCollection<Int>`には`append`できる。

```swift
var intCollection: any RangeReplaceableCollection<Int> = [1, 2, 3]
collection.append(4) // ⭕️: Element型は具体的なInt
```

ここでの原則は、その関連型が存在型によって具体化されている場合、メンバの関連型の使用は不変であるとは見なされない。これにより、`Element`は任意の`RangeReplaceableCollection<Int>`型によって具象化されているため、上記の`append`を使用できる。さらに、これは、関連型を消去した存在型で、制約された存在型を利用できることを意味する。例えば:

```swift
extension Sequence {
    func eagerFilter(_ isIncluded: @escaping (Element) -> Bool) -> [Element] { ... }
    func lazyFilter(_ isIncluded: @escaping (Element) -> Bool) -> some Sequence<Element> { ... }
}

func doFilter(sequence: any Sequence, intSequence: any Sequence<Int>) {
    let e1 = sequence.eagerFilter { _ in true } // ❌: Elementは不変位置で使われている
    let e2 = intSequence.eagerFilter { _ in true } // ⭕️: [Int]を返す
    let l1 = sequence.lazyFilter { _ in true }    // ❌: Elementは不変位置で使われている
    let l2 = intSequence.lazyFilter { _ in true } // ⭕️: any Sequence<Int>を返す
}
```

[SE-0352](https://github.com/apple/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md)で導入された暗黙的開示にも同じ消去効果がある。

## ABIへの影響

制約付きの存在型は完全に付加的な概念であるため、ABIの安定性に影響なし。

この機能には、既存のOSリリースへの下位互換性も下位展開もできないSwiftランタイムとABIの改訂が必要であることに注意が必要。

## 検討された代替案

関連型に同じ型の要件を導入するために、さまざまな種類のスペルを想像することができる。 たとえば、次のような`where`句ベースのアプローチ:

```swift
any (Collection where Self.Element == Int)
```

このような構文は、読みやすく、コンテキストで使用するのが難しく、他の既存の型や制約で構成するようになると、問題はさらに悪化する。さらに、[SE-0346](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md)の時点でSwiftのジェネリック制約が取っている全体的な方向性と矛盾する。ジェネリック制約構文はこの提案の範囲外であり、将来の方向性として後で説明する。

## 将来の検討事項

### ジェネリック制約

この提案は、[SE-0341](https://github.com/apple/swift-evolution/blob/main/proposals/0341-opaque-parameters.md#constraining-the-associated-types-of-a-protocol)のレビュー中に検討されたジェネリック制約構文を意図的に外している。1つスペルを取ると:

```swift
any Collection<.Index == String.Index>
```

ただし、このような構文が利用できる場合は、制約された存在型に適用されると予想されます。存在型に対するジェネリック制約の可能な設計については、[ここで](https://forums.swift.org/t/generalized-opaque-and-existential-type-constraints/55494)話されている。

### Opaque制約

特に興味深い構造の1つは、不透明(Opaque)型と制約付き存在型の組み合わせ。このコンボにより、特に強力な形式の型抽象化が可能になる。

```swift
any Collection<some View>
```

この型は、`Collection`プロトコルを実装するが、その`Element`型が`View`プロトコルのOpaqueインスタンスである値を表している。現在、Swiftのジェネリックシステムには、不透明（Opaque)型をオペランドとして使用して同じ型の制約を表現する機能がない。

### もっとジェネリックな存在型

既存の主要な関連型に対する制約は、既存の型が表現できる唯一のものではない。Swiftの型システムには、存在型を介して任意の(制約された)型パラメータをスコープ内で開くことができる。これにより、トップレベルの使用法だけでなく、

```swift
any<T: View> Collection<T>
```

ネストしていた場合にも利用できる

```swift
any Collection<any<T: Hashable> Collection<T>>
```

基本的に、プログラムの任意の時点で、任意の形状のジェネリック型に対してアドホックな抽象化を有効にする。


## 参考リンク

### Forums

- [[Pitch] Constrained Existential Types](https://forums.swift.org/t/pitch-constrained-existential-types/56361)
- [SE-0353: Constrained Existential Types](https://forums.swift.org/t/se-0353-constrained-existential-types/56853)
- [[Accepted] SE-0353: Constrained Existential Types](https://forums.swift.org/t/accepted-se-0353-constrained-existential-types/57560)

### プロポーザルドキュメント

- [Constrained Existential Types](https://github.com/apple/swift-evolution/blob/main/proposals/0353-constrained-existential-types.md)
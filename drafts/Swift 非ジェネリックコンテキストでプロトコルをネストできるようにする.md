# Swift 非ジェネリックコンテキストでプロトコルをネストできるようにする(Allow Protocols to be Nested in Non-Generic Contexts)

- [Swift 非ジェネリックコンテキストでプロトコルをネストできるようにする(Allow Protocols to be Nested in Non-Generic Contexts)](#swift-非ジェネリックコンテキストでプロトコルをネストできるようにするallow-protocols-to-be-nested-in-non-generic-contexts)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
    - [解決策](#解決策)
    - [詳細](#詳細)
      - [関連型のマッチング](#関連型のマッチング)
    - [ソース互換性](#ソース互換性)
    - [ABI互換性](#abi互換性)
    - [導入への影響](#導入への影響)
    - [将来の検討事項](#将来の検討事項)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

プロトコルを非ジェネリックな`struct`/`class`/`enum`/`actor`や関数にネストできるようにする。

## 内容

### 動機

例えば、`String.UTF8View`は`struct String`の中にネストした`struct UTF8View`であり、その名前は`String`値のUTF-8コードユニットへのインターフェースとしての目的を明確に伝えている。

しかし、現在のところ、ネストは他の`struct`/`class`/`enum`/`actor`の中の`struct`/`class`/`enum`/`actor`に制限されている。プロトコルはまったくネストできないので、モジュール内では常にトップレベルの型でなければならない。これは残念なことで、開発者が何らかの外側の型にスコープされるプロトコルを表現できるように、この制限を緩和すべき。

### 解決策

ジェネリックでない`struct`/`class`/`enum`/`actor`や、ジェネリックコンテキストに属さない関数内でのプロトコルのネストを許可する。

例えば、`TableView.Delegate`は当然`TableView`に関するデリゲートプロトコルです。開発者は、`TableView`クラスの中にネストして宣言する必要がある:

```swift
class TableView {
  protocol Delegate {
    func tableView(_ tableView: TableView, didSelectRowAt: Int)
  }
}

class DelegateConformer: TableView.Delegate {
  func tableView(_: TableView, didSelectRowAtIndex: Int) {
    // ...
  }
}
```

現在、開発者は`TableViewDelegate`のような複合的な名前をつけることで、ネストによって表現できる自然なスコープを表現している。

追加の利点として、`TableView`のコンテキスト内で、ネストされたプロトコル`Delegate`は、(他のすべてのネストされた型の場合と同様に)短い名前で参照することができる:

```swift
class TableView {
  weak var delegate: Delegate?
  
  protocol Delegate { /* ... */ }
}
```

プロトコルは、非ジェネリックな関数やクロージャの中にネストできる。確かに、このようなプロトコルへの適合もすべて同じ関数内になければならないため、この有用性はやや限定的である。しかし、開発者が関数内で作成するモデルの複雑さを人為的に制限する理由もない。いくつかのコードベース(特筆すべきは、Swiftコンパイラ自身)は、ネストした型を持つ大規模なクロージャを使用しており、プロトコルを使用した抽象化から利益を得ている。

```swift
func doSomething() {

   protocol Abstraction {
     associatedtype ResultType
     func requirement() -> ResultType
   }
   struct SomeConformance: Abstraction {
     func requirement() -> Int { ... }
   }
   struct AnotherConformance: Abstraction {
     func requirement() -> String { ... }
   }
   
   func impl<T: Abstraction>(_ input: T) -> T.ResultType {
     // ...
   }
   
   let _: Int = impl(SomeConformance())
   let _: String = impl(AnotherConformance())
}
```

### 詳細

プロトコルは、`struct`/`class`/`enum`/`actor`がネストできる場所であれば、ジェネリックコンテキストを除いて、どこにでもネストできる。例えば、以下は禁止されている:

```swift
class TableView<Element> {
  protocol Delegate { // Error: プロトコル 'Delegate' をジェネリックコンテキスト内に入れ子にすることはできません
    func didSelect(_: Element)
  }
}
```

ジェネリック関数も同様:

```swift
func genericFunc<T>(_: T) {
  protocol Abstraction { // Error: プロトコル'Abstraction'をジェネリック・コンテキスト内にネストすることはできません。
  }
}
```

そして、ジェネリックコンテキスト内の他の関数にも:

```swift
class TableView<Element> {
  func doSomething() {
    protocol MyProtocol {  // Error: プロトコル'Abstraction'はジェネリックコンテキスト内にネストできません。
    }
  }
}
```

これをサポートするには、以下のどちらかが必要になる:
- ジェネリックプロトコルを導入する
- ジェネリックパラメータを関連する型にマッピングする

どちらもこの提案の対象ではないが、ジェネリックコンテキストをサポートしなくても十分なメリットがあると筆者は感じている。これらのいずれかが、興味深い将来の方向性になることは間違いない。

#### 関連型のマッチング

具象型にネストしている場合、プロトコルは関連型(associatedtype)要件をwitnessしない。

```swift
protocol Widget {
  associatedtype Delegate
}

struct TableWidget: Widget {
  // Widget.Delegateをwitnessしない
  protocol Delegate { ... }
}
```

関連型は、1つの具象型を1つの適合型に関連付ける。プロトコルは、多くの具象型が適合する可能性のある制約型であるため、プロトコルが関連型の要件を証明することに明白な意味はない。

過去に、プロトコルが"associated protocol"機能を得ることができるかどうかについての議論があった。もしそのような機能が導入されることがあれば、関連型の要件が今日ネストした具象型によってwitnessできるのと同じように、関連するプロトコル要件がネストしたプロトコルによってwitnessすることは合理的かもしれない。

### ソース互換性

追加のみ。

### ABI互換性

この提案は、純粋に言語のABIを拡張するものであり、既存の機能を変更するものではない。

### 導入への影響

この機能は、デプロイの制約を受けずに、ソースコード上で自由に導入・未導入を選択できる。
一般的に、親コンテキストからプロトコルを出し入れすることは、ソースを壊す変更である。しかし、この破壊は、新しい名前に`typealias`を提供することで軽減できる。

他のネストした型と同様に、親コンテキストはネストしたプロトコルの修飾された(mangled)名前の一部を形成する。したがって、親コンテキストからプロトコルを出し入れすることは、ABIと互換性のない変更である。

### 将来の検討事項

- プロトコルに他の（プロトコルでない）型をネストできるようにする。
- プロトコル自身が、そのプロトコルに自然にスコープされる型を定義したいと思うことがある。例えば、標準ライブラリの[`FloatingPointRoundingRule`](https://developer.apple.com/documentation/swift/FloatingPointRoundingRule) `enum`型は、[`FloatingPoint`プロトコルの要件](https://developer.apple.com/documentation/swift/floatingpoint/round(_:))で使用され、そのために定義されている。
- ジェネリック型にプロトコルをネストできるようにする

詳細設計のセクションで述べたように、ジェネリック型内のプロトコルのネストできるようにする戦略がすでに潜在的にあり、その表現能力を使用する方法ははっきりとイメージできる。別のトピックとして潜在的なアプローチについて議論するようにコミュニティを招待する。

## 参考リンク

### Forums

- [Pitch: Allow Protocols to be Nested in Non-Generic Contexts](https://forums.swift.org/t/pitch-allow-protocols-to-be-nested-in-non-generic-contexts/65285)
- [SE-0404: Allow Protocols to be Nested in Non-Generic Contexts](https://forums.swift.org/t/se-0404-allow-protocols-to-be-nested-in-non-generic-contexts/66332)

### プロポーザルドキュメント

- [Allow Protocols to be Nested in Non-Generic Contexts](https://github.com/apple/swift-evolution/blob/main/proposals/0404-nested-protocols.md)
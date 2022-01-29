# Swift 存在型(Existential type)とanyキーワード

- [Swift 存在型(Existential type)とanyキーワード](#swift-存在型existential-typeとanyキーワード)
  - [概要](#概要)
  - [内容](#内容)
    - [存在型(Existential type)とは?](#存在型existential-typeとは)
    - [存在型の制限や問題](#存在型の制限や問題)
      - [機能面での制限](#機能面での制限)
      - [パフォーマンス面の問題](#パフォーマンス面の問題)
      - [ジェネリック制約としてのプロトコルと存在型としてのプロトコル](#ジェネリック制約としてのプロトコルと存在型としてのプロトコル)
    - [解決案](#解決案)
    - [anyの使い方](#anyの使い方)
      - [プロトコル、プロトコル合成、それらのメタタイプに使える](#プロトコルプロトコル合成それらのメタタイプに使える)
      - [プロトコル合成以外のAnyとAnyObjectにはanyが不要](#プロトコル合成以外のanyとanyobjectにはanyが不要)
      - [メタタイプ](#メタタイプ)
      - [タイプエイリアスとassociated type](#タイプエイリアスとassociated-type)
    - [Swift6への移行](#swift6への移行)
    - [Swift5.6から導入する理由](#swift56から導入する理由)
    - [将来的な話](#将来的な話)
      - [Existential typeの拡張](#existential-typeの拡張)
      - [プレーンなプロトコル名の転用](#プレーンなプロトコル名の転用)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [関連PR](#関連pr)
    - [その他](#その他)

## 概要

Swift6で存在型(Existential type)を書く際に`any`キーワードが必須になる。その理由や内容について見ていく。

## 内容

### 存在型(Existential type)とは?

ある条件を満たす全ての型を表す型。Swiftではプロトコルを型として使用した場合にがこれに該当する。

例: `Animal`プロトコルに準拠した`Dog`構造体と`Cat`構造体は`Animal`型として扱うことができる。

```swift
protocol Animal {}
struct Dog: Animal {}
struct Cat: Animal {}

let animals: [Animal] = [Dog(), Cat()]
```

存在型には、異なる具体的な型を代入できる。

```swift
var dog: Animal = Dog()
dog = Cat() // ok
```

関連ドキュメント: https://docs.swift.org/swift-book/LanguageGuide/Protocols.html#ID275

### 存在型の制限や問題

存在型を使用する際、下記のような機能面の制限やパフォーマンス面の問題があることを認識する。

#### 機能面での制限

- 型として利用できない場合がある。

下記の場合、プロトコルを型として扱うことはできない。

1. 関連型(associated type)が要件に含まれている
2. メソッド/プロパティ/subscript/イニシャライザの共変ではない位置(引数など)に`Self`への参照が含まれている

※ ただし、この問題は解決される予定。

より詳細は[Swift PAT問題を解消して全てのプロトコルを存在型として使えるように](./Swift%20PAT問題を解消して全てのプロトコルを存在型として使えるように.md)に記載。

関連ドキュメント: https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md

- 型情報が消去されているため、使える機能が制限される。

例えば、下記の2つの`callee`の戻り値の型は同じであることがわからないので加算できない。

```swift
func callee() -> Numeric {
    if Bool.random() {
        return 42
    } else {
        return 42.0
    }
}

func caller() {
    let x = callee() + callee() // ❌ Binary operator '+' cannot be applied to two 'Numeric' operands
}
```

- 存在型はそのプロトコル自身に準拠できない。これはプロトコル自身がそのプロトコルの全ての要件を満たすことができない場合があるため(staticプロパティやイニシャライザなど)。全てを満たすことができる場合もあるが、例えば外部から提供されるプロトコルにプロトコル自身が満たせない要件が追加された場合にsource breakが発生してしまうため、これを防ぐ役割もある。

```swift

protocol P {
    func test()
}

func generic<ConcreteP: P>(p: ConcreteP) {
    p.test()
}

func useExistential(p: any P) {
    generic(p: p) // ❌ Protocol 'P' as a type cannot conform to the protocol itself
}
```

関連ドキュメント: https://github.com/apple/swift/blob/main/userdocs/diagnostics/protocol-type-non-conformance.md

#### パフォーマンス面の問題

- プロトコルは具体的な型を動的に保持するため、動的なメモリが必要になる。たとえ3単語バッファのようなスタックに収まる小さいものでもヒープ領域と参照カウンタを使う。

下記を見てみると、プロトコルは動的な型の情報を保持するためにあらかじめメモリを確保しておく必要がある。(`class`は実際のオブジェクトのメモリアドレスを保持するために必要なメモリが確保される)

```swift
protocol Animal {}
struct Dog: Animal {}
class Cat: Animal {}

let animal: Animal = Dog()
let dog = Dog()
let cat = Cat()

MemoryLayout.size(ofValue: animal) // 40
MemoryLayout.size(ofValue: dog) // 0
MemoryLayout.size(ofValue: cat) // 8
```

関連ドキュメント: https://devstreaming-cdn.apple.com/videos/wwdc/2015/414sklk5h2k3ki3/414/414_building_better_apps_with_value_types_in_swift.pdf

- メソッドのダイナミックディスパッチやポインタの間接参照があるため、コンパイラによる最適化ができない。

下記の例で見てみると、変数`animal`の値は実行時までわからないため、コンパイル時に型を決められない。

```swift

protocol Animal {
    func bark()
}
struct Dog: Animal {
    func bark() { print("bowwow") }
}

class Cat: Animal {
    func bark() { print("meow") }
}

var animal: Animal = Dog()
animal.bark() // bowwow

animal = Cat()
animal.bark() // meow
```

#### ジェネリック制約としてのプロトコルと存在型としてのプロトコル

Swiftでは、ジェネリックの制約にも存在型にもプロトコルを使用し、スペリングも似ている。そのため、ジェネリックを扱う際に両方を混同しがちになる。
さらに、存在型はジェネリックの制約を設定するよりもはるかに簡単に記述できることから、上記のようなコストがあることを意識せずに存在型を使ってしまっている場合がある。

関連ドキュメント:

- [Protocols as Types](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html#ID275)
- [Type Constraints](https://docs.swift.org/swift-book/LanguageGuide/Generics.html#ID186)

### 解決案

そこで、存在型が使用されている場面では、`any`キーワードを付けることを必須にすることで、存在型の誤用を防ぎたい。
エラーにはならないが、Swift5.6から導入予定。

※ Swift5とSwift6以降で挙動が異なる。

- Swift5 モード

```swift
protocol P {}
protocol Q {}
struct S: P, Q {}

let p1: P = S() // ⭕️ この P は存在型
let p2: any P = S() // ⭕️ any P は明示的に存在型

let pq1: P & Q = S() // ⭕️ この P & Q は存在型
let pq2: any P & Q = S() // ⭕️ any P & Q は明示的に存在型
```

- Swift6 モード

```swift
protocol P {}
protocol Q {}
struct S: P, Q {}

let p1: P = S() // ❌ エラー
let p2: any P = S() // ⭕️

let pq1: P & Q = S() // ❌ エラー
let pq2: any P & Q = S() // ⭕️
```

### anyの使い方

#### プロトコル、プロトコル合成、それらのメタタイプに使える

逆に`struct`や`enum`、`tuple`などの名前的型(nominal type)、tupleやOptional、関数型などの構造的型(structural type)、関数、ジェネリックの型パラメータなどには使えない。

```swift
struct S {}
 
let s: any S = S() // ❌ error: 'any' has no effect on concrete type 'S'
 
func generic<T>(t: T) {
  let x: any T = t // ❌ error: 'any' has no effect on type parameter 'T'
}
 
let f: any ((Int) -> Void) = generic // ❌ error: 'any' has no effect on concrete type '(Int) -> Void'
```

#### プロトコル合成以外のAnyとAnyObjectにはanyが不要

`Any`と`AnyObject`はすでに存在型だと明白なので2度手間になる。ただし、付けてもワーニングなどは出ない。(Accepted時に変更された)

```swift
struct S {}
class C {}
 
let value: any Any = S()
let values: [any Any] = []
let object: any AnyObject = C()
 
protocol P {}
extension C: P {}
 
let pObject: any AnyObject & P = C() // ⭕️
```

#### メタタイプ

存在型のメタタイプには`any`を付ける。例えば、`P`に準拠した型自体のメタタイプ(`P.Type`)は`any P.Type`になる。`P`プロコトルのメタタイプ(`P.Protocol`)は、`any P`をかっこで囲んで`(any P).Type`になる。`P`プロコトルのメタタイプの値(`P.self`)は`any P`をかっこで囲んで`.self`を付ける。`(any P).self`

存在型のメタタイプは複数存在するが、プロトコルのメタタイプは一つしか存在しない。

```swift
protocol P {}
struct S: P {}
 
let existentialMetatype: any P.Type = S.self
 
protocol Q {}
extension S: Q {}
 
let compositionMetatype: any (P & Q).Type = S.self
 
let protocolMetatype: (any P).Type = (any P).self
```

#### タイプエイリアスとassociated type

プレーンなプロトコルの名前と同様に、プロトコルのタイプエイリアスもジェネリック制約と存在型の両方で使える。`any`が付いていると明らかに存在型なので、`any P`のタイプエイリアスはジェネリック制約としては使えず、存在型のみに利用できる。

```swift
protocol P {}
typealias AnotherP = P
typealias AnyP = any P
 
struct S: P {}
 
let p2: any AnotherP = S()
let p1: AnyP = S()
 
func generic<T: AnotherP>(value: T) { ... }
func generic<T: AnyP>(value: T) { ... } // ❌
```

Swift6になると、プレーンなプロトコルのタイプエイリアスは関連型として妥当ではなくなり、タイプエイリアスの中で`any`を指定する必要がある。

- Swift6 モード

```swift
protocol P {}
 
protocol Requirements {
  associatedtype A
}
 
struct S1: Requirements {
  typealias A = P // ❌ error: associated type requirement cannot be satisfied with a protocol
}
 
struct S2: Requirements {
  typealias A = any P // ⭕️ okay
}
```

### Swift6への移行

方法はまだ未定だが、migratorで自動で変換されるという記載はある。

段階的な導入として

- Swift5.6から利用可能だが、必要な箇所で`any`がなくてもワーニングは出ない
- 1つ以上先のメジャーリリース※でワーニングを導入して新しい構文の導入を促す
- 最終的に新しいメジャーな言語バージョン(Swift6を想定)では、古い構文(`any`なしの存在型)はエラーになる

※ 5.6や5.7などをメジャーリリースと呼ぶ(5.5.1などをポイントリリースと呼ぶ)

### Swift5.6から導入する理由

[Unlock existential for all protocols](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md) によってExistentialをより多くのコードで使えるようになった。そのためSwift6で不正になるコードを減らすためにもSwift5.6から導入するのが良いと判断。

### 将来的な話

#### Existential typeの拡張

Existentialをそのプロトコル自身に適合できるようにする。こうすることで、型消去の型をわざわざ作成しなくてもextensionで対応できるようになる場合がある。

```swift
extension any Equatable: Equatable { ... }
```

#### プレーンなプロトコル名の転用

存在型のシンタックスを変えることで、プレーンなプロトコル名の利用方法を変えることができるかもしれない。

例えば、プレーンなプロトコル名は、プロトコルの要件を満たす囲まれたコンテキスト(extensionなど)内で、常にジェネリックの型パラメータのシンタックスシュガーとみなすことができる。

```swift
extension Collection { ... }

// 上記と同じと見なせる
extension <Self> Self where Self: Collection { ... }
```

これによって今提案されている他の機能と組み合わせることで、よりコードを簡潔に書けるかもしれない。

例えば、現在のArrayの`append(contentsOf:)`は

```swift
extension Array {
  mutating func append<S: Sequence>(contentsOf newElements: S) where S.Element == Element
}
```

という形だが、associated typeをカッコ内に書けるようになると、もっと簡単に定義できる。

```swift
extension Array {
  mutating func append(contentsOf newElements: Sequence<Element>)
}
```

## 参考リンク

### Forums

- [[Pitch] Introduce existential any](https://forums.swift.org/t/pitch-introduce-existential-any/53520)
- [SE-0335: Introduce existential any](https://forums.swift.org/t/se-0335-introduce-existential-any/53934)
- [[Accepted with modifications] SE-0335: Introduce existential any](https://forums.swift.org/t/accepted-with-modifications-se-0335-introduce-existential-any/54504)

### プロポーザルドキュメント

- [Introduce existential any](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md)

### 関連PR

- [[SE-0335] Enable explicit existential types.](https://github.com/apple/swift/pull/40666)
- [[5.6][SE-0335] Enable explicit existential types.](https://github.com/apple/swift/pull/40804)

- [[5.6][SE-0335] Enable explicit existential types.](https://github.com/apple/swift/pull/40949)

### その他

- [Unlock existential for all protocols](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)
- [Protocols as Types](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html#ID275)
- - [Type Constraints](https://docs.swift.org/swift-book/LanguageGuide/Generics.html#ID186)
- [Building Better Apps with Value Types in Swift](https://www.wwdcnotes.com/notes/wwdc15/414/)
- [Understanding Swift Performance](https://developer.apple.com/videos/play/wwdc2016/416/)
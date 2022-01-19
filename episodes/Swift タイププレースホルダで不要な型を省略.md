# Swift タイププレースホルダで不要な型を省略

収録日: 2022/01/15

- [Swift タイププレースホルダで不要な型を省略](#swift-タイププレースホルダで不要な型を省略)
  - [概要](#概要)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決策](#解決策)
    - [なぜ今出てきたの？](#なぜ今出てきたの)
    - [文法](#文法)
    - [型推論の流れ](#型推論の流れ)
    - [ジェネリック制約](#ジェネリック制約)
    - [ジェネリックパラメータの推論](#ジェネリックパラメータの推論)
    - [関数シグネチャ(入力と出力)には使えない](#関数シグネチャ入力と出力には使えない)
    - [ダイナミックキャスト(ランタイム時のキャスト)は使えるけど](#ダイナミックキャストランタイム時のキャストは使えるけど)
    - [将来的な検討事項](#将来的な検討事項)
      - [ジェネリックの基の型やネストした型でも使えるようにするかの検討](#ジェネリックの基の型やネストした型でも使えるようにするかの検討)
        - [ジェネリックの基の型](#ジェネリックの基の型)
        - [ネストした型](#ネストした型)
      - [属性のタイププレースホルダ](#属性のタイププレースホルダ)
      - [トップレベルのタイププレースホルダ](#トップレベルのタイププレースホルダ)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [その他](#その他)

## 概要

Swiftの型推論は特定のケースでうまく機能しないことがある。そういった場合、開発者側で必要ではない型も明示しなければならない。そこで、Swift5.6よりタイププレースホルダという機能を追加してこの負担を軽減する。

```swift
let losslessStringConverter = Double.init as (String) -> Double?

losslessStringConverter("42") //-> 42.0
losslessStringConverter("##") //-> nil
```

代わりに`_`(アンダースコア)で代用できるようにする

```swift
let losslessStringConverter = Double.init as (String) -> _?
```

## 内容

### 問題点

Swiftの型推論は強力だが、うまく機能しない場合やデフォルトの型をオーバーライドしたい場合がある。

上記で示した`Double.init`は複数のオーバーライドがあるため、コンパイラは自動で型解決できない。

Swiftには型情報を明示する方法がいくつかある。

- 変数に型を指定

```swift
let losslessStringConverter: (String) -> Double? = Double.init
```

- `as`キーワードを使用

```swift
let losslessStringConverter = Double.init as (String) -> Double?
```

- 明示的に型引数を指定

```swift
let dict = try JSONDecoder().decode([String: Int].self, from: data)
```

上記のようなシンプルなものでは良いが、もっと複雑なケースだと問題になる。

```swift
enum Either<Left, Right> {
  case left(Left)
  case right(Right)

  init(left: Left) { self = .left(left) }
  init(right: Right) { self = .right(right) }
}

func makePublisher() -> Some<Complex<Nested<Publisher<Chain<Int>>>>> { ... }
```

`makePublisher`から`Either`を初期化するのは難しい。

```swift
let publisherOrValue = Either(left: makePublisher()) // ❌ Error: generic parameter 'Right' could not be inferred
```

こうする必要がある。

```swift
let publisherOrValue = Either<Some<Complex<Nested<Publisher<Chain<Int>>>>>, Int>(left: makePublisher())
```

この型をエラーメッセージやドキュメントから探して書くことは厳しい。

### 解決策

代わりに`_`(アンダースコア)を使うことでコンパイラが型チェック中に任意の型を満たせるようにする。上記の例は下記のようになる。

```swift
let publisherOrValue = Either<_, Int>(left: makePublisher())
```

`makePublisher`の型は戻り値からわかるので推論できる。

### なぜ今出てきたの？

型推論システムの改善によって、こういった推論ができるようになった。  

関連ドキュメント: https://github.com/apple/swift/blob/main/docs/TypeChecker.md

### 文法

Swiftで型を指定できるところではどこでも使える。

タイププレースホルダを含んだ型の例:

```swift
Array<_>
[Int: _]
(_) -> Int
(_, Double)
_?
```

ただし、トップレベルの型には使用できない(詳しくは[トップレベルのタイププレースホルダ](#トップレベルのタイププレースホルダ)を参照)。

```swift
let percent: _ = 100.0 // ❌ placeholders are not allowed as top-level types
```

ネストした型にも使用できない(詳しくは[ネストした型](#ネストした型)を参照)。

```swift
struct Outer {
    struct Inner {}
    func inner() -> Inner { Inner() }
}

func test(outer: Outer) {
    let result: Outer._ = outer.inner() // ❌
}
```

ジェネリックの基の型にも使用できない(詳しくは[ジェネリックの基の型](#ジェネリックの基の型)を参照)。

```swift
let publisher: _<Int, Error> = Just(0).setFailureType(to: Error.self).eraseToAnyPublisher() // ❌
```

### 型推論の流れ

コンパイラが型チェック時にタイププレースホルダを見つけると、先に型が指定されているところから型を満たしていく。

タイププレースホルダは、型のその部分にコンテキストを提供しないものとして扱われ、部分的なコンテキストが与えられた場合に式の残りの部分で解決される(できないとエラーになる)。事実上、タイププレースホルダはユーザが定義する匿名型と言える。

例えば、

```swift
import Combine

func makeValue() -> String { "" }
func makeValue() -> Int { 0 }

let publisher = Just(makeValue()).setFailureType(to: Error.self).eraseToAnyPublisher()
```

このコードは`makeValue()`の型が決まらないためエラーになる。

型を指定することで解決する。

```swift
let publisher: AnyPublisher<Int, Error> = Just(makeValue()).setFailureType(to: Error.self).eraseToAnyPublisher()
```

しかし、`Error`は`setFailureType(to:)`でわかっているので必要ない。そこで`Error`にタイププレースホルダが使える。

```swift
let publisher: AnyPublisher<Int, _> = Just(makeValue()).setFailureType(to: Error.self).eraseToAnyPublisher()
```

### ジェネリック制約

特定のプロトコルに準拠していることが期待される箇所にも利用できる。

```swift
let dict: [_: String] = [0: "zero", 1: "one", 2: "two"]
```

`Key`のタイププレースホルダは`Hashable`に準拠した型が期待される。現在のタイププレースホルダは、全て具体的な型の推論として利用されるため、`Key`は全て同じ型である必要がある。

```swift
let dict: [_: String] = ["0": "zero", 1: "one", 2: "two"] // ❌　Cannot convert value of type 'String' to expected dictionary key type 'Int'
```

### ジェネリックパラメータの推論

既存のジェネリックパラメータの推論で既に一部は実現できている。例えば、

```swift
let publisher = Just(0) // Just<Int>と推論される
```

タイププレースホルダも下記のように使える。

```swift
let publisher = Just<_>(0)
```

### 関数シグネチャ(入力と出力)には使えない

現在の実装ルールに合わせて引数の戻り値の型を完全に指定する必要がある。たとえプロトコル要件やデフォルト引数などで型が明らかにわかっている場合などでも。

下記はエラー。

```swift

func doSomething(_ count: _? = 0) { ... } // ❌

let count: _? = 0 // ❌

struct Bar<T, U>
where T: ExpressibleByIntegerLiteral, U: ExpressibleByIntegerLiteral {
    var t: T
    var u: U
}

extension Bar {
    func frobnicate() -> Bar {
        return Bar(t: 42, u: 42)
    }
    func frobnicate2() -> Bar<_, _> { // ❌
        return Bar(t: 42, u: 42)
    }
    func frobnicate3() -> Bar {
        return Bar<_, _>(t: 42, u: 42)
    }
    func frobnicate4() -> Bar<_, _> { // ❌
        return Bar<_, _>(t: 42, u: 42)
    }
    func frobnicate5() -> Bar<_, U> { // ❌
        return Bar(t: 42, u: 42)
    }
    func frobnicate6() -> Bar {
        return Bar<_, U>(t: 42, u: 42)
    }
    func frobnicate7() -> Bar<_, _> { // ❌
        return Bar<_, U>(t: 42, u: 42)
    }
    func frobnicate8() -> Bar<_, U> { // ❌
        return Bar<_, _>(t: 42, u: 42)
    }
}
```

### ダイナミックキャスト(ランタイム時のキャスト)は使えるけど

ダイナミックキャストは、`as`とは異なり、`0 as? String`や`[""] is Double`と書くことができるように、キャスト式とキャストタイプの間に固有の関係はない。

今回のプロポーザルで`is`、`as?`、`as!`の使用は禁止しないが、特別に型推論のルールの追加などはないので、たいていの場合は失敗する。`0 as? [_]`など。

### 将来的な検討事項

#### ジェネリックの基の型やネストした型でも使えるようにするかの検討

いくつかの例では、まだ技術的にコンパイラが必要な情報以上に型情報を提供する必要がある。たとえば、ジェネリックの基の型や型の中のネストした型も推論させることは可能。しかし、これらがコードをより明確にするかどうかは懐疑的なため、有用性やトレードオフなどを将来的に検討して導入の可否を決める。

##### ジェネリックの基の型

例えば、

```swift
let publisher: _<Int, _> = Just(makeValue())
    .setFailureType(to: Error.self)
    .eraseToAnyPublisher() // ❌
```

これは、`eraseToAnyPublisher()`から基の型は`AnyPublisher`であることは明白なので、利用できる可能性がある。

##### ネストした型

同様に、型の内部のメンバでも利用できる可能性がある。下記の例では、変数の型の親(`S`)から推論できる。

```swift
struct S {
  struct Inner {}

  func overloaded() -> Inner {  }
  func overloaded() -> Int {  }
}

func test(val: S) {
  let result: S._ = val.overloaded() // 'func overloaded() -> Inner'を読んでいる
}
```

#### 属性のタイププレースホルダ

コンテキストの残りの型から推論できる場合、属性にも利用できる可能性がある。

```swift
let x: @convention(c) _ = { 0 }
```

ただし、現在の型属性の型名の構文フォームに密接につながっているため問題がある。

下記はエラーになる。

```swift
typealias F = () -> Int
let x: @convention(c) F = { 0 } // ❌ @convention attribute only applies to function types
```

こういったユースケースはあまり多く使われないと思うため、将来的な検討事項にしておく。

#### トップレベルのタイププレースホルダ

元々は導入する予定だったがちょっとだけ明確になるだけで、`as`キャストでも役に立たない。

```swift
let x: _ = 0.0 // type of x is inferred as Double
```

メタタイプを引数に渡す場合などで役に立つケースも考えられるが、

```swift
let p: AnyPublisher<Int, Error> = Just<Int>().setFailureType(to: _.self).eraseToAnyPublisher()s
```

例えばライブラリの作者が型情報を明示的に提供することを意図していても、利用側で省略できてしまうことになってしまう。

```swift
self.someProp = try container.decode(_.self, forKey: .someProp)
```

これは実際に利用された後でメリットがあるかないかを見て検討する。

## 参考リンク

### Forums

- [SE-0315: Placeholder types](https://forums.swift.org/t/se-0315-placeholder-types/49801)
- [[Accepted] SE-0315: Placeholder types](https://forums.swift.org/t/accepted-se-0315-placeholder-types/50671)

### プロポーザルドキュメント

- [Type placeholders](https://github.com/apple/swift-evolution/blob/main/proposals/0315-placeholder-types.md)

### その他

- [Type Checker Design and Implementation](https://github.com/apple/swift/blob/main/docs/TypeChecker.md)
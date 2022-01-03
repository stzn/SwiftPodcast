# Swift5.6 タイププレースホルダで不要な型を省略

- [Swift5.6 タイププレースホルダで不要な型を省略](#swift56-タイププレースホルダで不要な型を省略)
  - [概要](#概要)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決策](#解決策)
    - [文法](#文法)
    - [型推論](#型推論)
    - [使い方](#使い方)
      - [Generics制約](#generics制約)
      - [Generics引数の推論](#generics引数の推論)
      - [関数シグネチャ(入力と出力)](#関数シグネチャ入力と出力)
      - [ダイナミックキャスト(ランタイム時のキャスト)](#ダイナミックキャストランタイム時のキャスト)
    - [将来的な検討事項](#将来的な検討事項)
      - [Genericsの基の型やネストした型](#genericsの基の型やネストした型)
  - [参考リンク](#参考リンク)

## 概要

Swiftの型推論は特定の場合にうまく機能しないことがある。そういった場合、開発者側で、全部が必要ではない場合でも明示的に全ての型を指定しなければならない。この負担を軽減する。

```swift
let losslessStringConverter = Double.init as (String) -> Double?

losslessStringConverter("42") //-> 42.0
losslessStringConverter("##") //-> nil
```

代わりに_(アンダースコア)で代用できるようにする

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

- asキーワードを使用

```swift
let losslessStringConverter = Double.init as (String) -> Double?
```

- 明示的に型引数を指定

```swift
let dict = try JSONDecoder().decode([String: Int].self, from: data)
```

もっと複雑なケースだと問題になる

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
let publisherOrValue = Either(left: makePublisher()) // Error: generic parameter 'Right' could not be inferred
```

こうする必要がある。

```swift
let publisherOrValue = Either<Some<Complex<Nested<Publisher<Chain<Int>>>>>, Int>(left: makePublisher())
```

この型をエラーメッセージやドキュメントから探して書くことは厳しい。

### 解決策

代わりに_(アンダースコア)を使うことでコンパイラの型チェック中に任意の型で満たせるようにする。上記の例は下記のようになる。

```swift
let publisherOrValue = Either<_, Int>(left: makePublisher())
```

`makePublisher`の型は戻り値からわかるのでできる。

### 文法

Swiftで型を指定できるところではどこでも使える(変数の型を指定、as を使ったキャスト、型パラメータを明示的に渡す、Genericsの制約など)

下記のようなところで使える。

```swift
Array<_>
[Int: _]
(_) -> Int
(_, Double)
_?
```

### 型推論

コンパイラが型チェック時にタイププレースホルダを見つけると、先に型が指定されているところから型を満たしていく。

タイププレースホルダは、型のその部分にコンテキストを提供しないものとして扱われ、部分的なコンテキストが与えられた場合、式の残りの部分で解決可能である必要がある。事実上、タイププレースホルダはユーザが定義する匿名型である。

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

しかし、`Error`は`setFailureType(to:)`でわかっているので必要ない。

```swift
let publisher: AnyPublisher<Int, _> = Just(makeValue()).setFailureType(to: Error.self).eraseToAnyPublisher()
```

### 使い方

#### Generics制約

特定のプロトコルに準拠していることが期待される箇所にも利用できる場合がある

```swift
let dict: [_: String] = [0: "zero", 1: "one", 2: "two"]
```

Keyのタイププレースホルダは`Hashable`であると期待され、タイププレースホルダは必要なすべての制約を満たすと想定され、イニシャライザがチェックされるまでこれらの制約の検証を遅らせる。

#### Generics引数の推論

Generics引数の推論で既に似たようなことは実現できている。例えば、

```swift
let publisher = Just(0) // Just<Int>と推論される
```

タイププレースホルダも下記のように使える。

```swift
let publisher = Just<_>(0)
```

#### 関数シグネチャ(入力と出力)

今回のプロポーザルでは使用することができない。

下記の例でエラーになる箇所を参照

```swift
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

#### ダイナミックキャスト(ランタイム時のキャスト)

`as`とは異なり、キャストされた式とキャストタイプの間に固有の関係はない。`0 as? String`や`[""] is Double`と書くことができる。

今回のプロポーザルで`is`、`as?`、`as!`の使用は禁止しないが、特別な考慮も行わないのでたいていの場合は失敗する。`0 as? [_]`など。

### 将来的な検討事項

#### Genericsの基の型やネストした型

いくつかの例では、まだ技術的にコンパイラが必要な情報以上に型情報を提供する必要がある。

例えば、上記の`Just`の例では、

```swift
let publisher: _<Int, _> = Just(makeValue()).setFailureType(to: Error.self).eraseToAnyPublisher() // 今のところ❌
```

`eraseToAnyPublisher()`から`AnyPublisher`は型が明白。

同様に、型の内部のメンバも推論可能だができない。

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

これはどっちの`overloaded()`が使われているかがコードから明確かが懐疑的なため、さらにトレードオフなどを議論する必要がある。

## 参考リンク

- https://github.com/apple/swift-evolution/blob/main/proposals/0315-placeholder-types.md

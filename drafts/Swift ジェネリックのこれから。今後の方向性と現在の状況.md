# Swift ジェネリックのこれから。今後の方向性と現在の状況

- [Swift ジェネリックのこれから。今後の方向性と現在の状況](#swift-ジェネリックのこれから今後の方向性と現在の状況)
  - [概要](#概要)
  - [内容](#内容)
    - [ジェネリックとは？](#ジェネリックとは)
    - [ジェネリック Manifest](#ジェネリック-manifest)
    - [用語紹介](#用語紹介)
      - [型レベルの抽象化(ジェネリック)](#型レベルの抽象化ジェネリック)
      - [値レベルの抽象化(存在型)](#値レベルの抽象化存在型)
    - [Generics Manifestで足りないもの、再考の余地があるもの](#generics-manifestで足りないもの再考の余地があるもの)
      - [値レベルの抽象化の制限](#値レベルの抽象化の制限)
        - [問題点](#問題点)
        - [解決案](#解決案)
      - [戻り値で型レベルの抽象化ができない](#戻り値で型レベルの抽象化ができない)
        - [問題点](#問題点-1)
        - [解決策](#解決策)
      - [ジェネリックの書き方が複雑](#ジェネリックの書き方が複雑)
        - [問題点](#問題点-2)
        - [解決策](#解決策-1)
  - [参考リンク](#参考リンク)

## 概要

以前よりSwift ジェネリックの進むべき方向としてジェネリック Manifestが存在していた。現在では、コアな機能の実装は完了している。
Generics Manifestでまだ未実装の部分の実装を進めるにあたって、足りない機能や再考が必要と考えられる機能などについて、ロードマップとして示された。

関連スレッド: https://forums.swift.org/t/improving-the-ui-of-generics/22814

## 内容

※今回は全体の概要を紹介して、個々の具体的な機能については別途

### ジェネリックとは？

プログラミング言語の機能・仕様の一つで、同じプログラムコードで様々なデータ型のデータを処理できるようにするもの。ジェネリックコードを使用すると、定義した要件に応じて、任意の型で機能する柔軟で再利用可能な関数と型を記述できる。重複を避け、その意図を明確で抽象的な方法で表現するコードを書くことができる。

関連ドキュメント: https://docs.swift.org/swift-book/LanguageGuide/Generics.html

### ジェネリック Manifest

Swift 3 リリース時のSwiftのジェネリックの足りない機能や改善点などについてのSwift Core teamやdeveloperの話し合いの結果をまとめたもの。

関連ドキュメント: https://github.com/apple/swift/blob/main/docs/GenericsManifesto.md

### 用語紹介

#### 型レベルの抽象化(ジェネリック)

特定のインスタンスで使用されている特定の型情報を保持しつつ、特定の制約に準拠する任意の型を使って共通の関数または型を使用できる

ジェネリック関数は特定の型を表す**型変数**を使う。

```swift
func foo<T: Collection>(x: T) { ... }
func bar<T: Collection>(x: T, y: T) -> [T] { ... }
```

コンパイル時にTの型情報は解決されるため、それぞれの値(x, y, 戻り値)の型の関係は保持される。上記の例だとx, y, 戻り値は全て同じ型になる。

#### 値レベルの抽象化(存在型)

存在型は、一連の制約に準拠する任意の型の任意の値を保持できる別の型であり、基になる特定の型の情報を消去して抽象化する

存在型を使用すると、さまざまな具体的な型の値を、同じ存在型の値として使用でき、値レベルで基になる型の間の違いを抽象化できる。

同じ存在型の異なるインスタンスは、異なる具体的な型の値を一緒に保持できる

```swift
protocol Animal {}
struct Dog: Animal {}
struct Cat: Animal {}

let animals: [Animal] = [Dog(), Cat()]
```

存在値を変更すると、異なる具体的な型に変更できる。

```swift
var dog: Animal = Dog()
dog = Cat() // ok
```

### Generics Manifestで足りないもの、再考の余地があるもの

`Collection`プロトコルを使って考えてみる。

```swift
func bar(x: Collection, y: Collection) -> [Collection] { ... }
```

※ Selfやassociated typeを要件とするプロトコルは型として使用できないという制限があったが(いわゆるPAT問題)、2020/1現在、全てのプロトコルを存在型は使用できるようになる実装が進んでいる。

関連ドキュメント: https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md

#### 値レベルの抽象化の制限

##### 問題点

型情報を保持するジェネリックとは異なり、値の型同士の関係が保持されていないため、具体的な型を使った機能が使えない(例えば、関数の引数と戻り値が実装レベルで同じだったとしても型情報からはそれがわからない)。そのため、ジェネリックに比べてできることが制限されている。

```swift
// Indexの型がわからないので型安全ではない
var start = x.startIndex

// 別の方に変更できてしまう可能性がある
// start = y.startIndex
var firstValue = x[start] // error

// 同じようにElementの型もわからないので型安全ではない
var first = x.first!
var indexOfFirst = x.index(of: first) // error

```

##### 解決案

- associated typeを指定できるようにする

こうすることで、下記の関数では引数と戻り値のElementの型が同じある制約を設定できる。

```swift
typealias CollectionOf<T> = Collection where Self.Element == T
func bar<T>(x: CollectionOf<T>, y: CollectionOf<T>) -> [CollectionOf<T>] { ... }
```

関連スレッド: https://forums.swift.org/t/pitch-light-weight-same-type-constraint-syntax/52889

- 存在型はそれを制約するプロトコル自身に準拠できるようにする

上記の例ではIndexの型がわからないため、全てのAPIを使用することはできない。一方でIndexを指定すると具体的過ぎて、存在型としての利便性が損なわれてしまう。

そこで存在型にある制約を加えてそのプロトコル自身に準拠できるようにする。

```swift
extension Collection: Collection {
  // Comparableに準拠した任意の方を引数に値を取得する
  subscript(index: Comparable) -> Any {
    // Indexは現在のCollectionのIndexの型と合致していることは事前にわかっているので
    // キャストしてそのCollectionの型にキャストすることができる
    return self[index as! Self.Index]
  }
}
```

- 存在型から具体的な型へ「展開」できるようにする

ローカル変数に再代入する際に、型として利用できるようにする。こうすることで、単一の存在型のassociated typeを使った計算ができる。

※ シンタックスは未定

```swift
let <X: Collection> openedX = x // Xはxの動的な型に紐づく
let start = openedX.startIndex
let first = openedX[start] // X.IndexからX.Elementを取得することができる
```

#### 戻り値で型レベルの抽象化ができない

##### 問題点

ジェネリックを使った場合、呼び出し側で具体的な型を指定することができる。

```swift
func zim<T: P>() -> T { ... }

let x: Int = zim() // T == Int 呼び出し側で型が決まる
let y: String = zim() // T == String 呼び出し側で型が決まる
```

一方で、呼び出し先の実装で具体的な型を抽象化したい場合もある。
例えば、`Collection`の計算に`lazy`を使った場合に`LazySequence`といった型の情報を公開したくないので`Collection`を存在型として使いたい。

```swift
func evenValues<C: Collection>(in collection: C) -> Collection where C.Element == Int {
  return collection.lazy.filter { $0 % 2 == 0 }
}
```

しかし、現状`Collection`は型として使えない(上記でも書いたが、今後使用可能になる予定)。そこでジェネリックを使ってみる。

```swift
func evenValues<C: Collection, Output: Collection>(in collection: C) -> Output
  where C.Element == Int, Output.Element == Int
{
  return collection.lazy.filter { $0 % 2 == 0 }
}
```

これだと呼び出し側で戻り値が決められてしまい、元の意図と違ってくる。

解決するためには型消去(Type eraser)する必要がある。

Standard Libraryには`AnyCollection`が用意されている。

```swift
func evenValues<C: Collection>(in collection: C) -> AnyCollection<Int> where C.Element == Int {
    AnyCollection<Int>(collection.lazy.filter { $0 % 2 == 0 })
}
```

(将来的にはassociated typeを指定できるようになるかもしれない)

```swift
func evenValues<C: Collection>(in collection: C) -> CollectionOf<Int>
  where C.Element == Int
{
  return collection.lazy.filter { $0 % 2 == 0 }
}
```

ただし、これも正確ではなく、呼び出されるたびに異なる`Collection`型を返し、その型情報は失われている。

結局、戻り値で型レベルの抽象化が必要な可能性がある。

##### 解決策

- Swift5.1でOpaque Result Typeが導入された  
関連ドキュメント: https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md

- Swift5.6でさらに使用可能範囲が拡張する  
関連ドキュメント: https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md

※ これは別で取り上げる。この際にリーバースジェネリクスというより広い概念についても紹介する。

#### ジェネリックの書き方が複雑

##### 問題点

Swiftでは、プロトコルを使ってジェネリックも存在型も実現している。そのため、この両方に使い方が不明瞭になりがちであったり、ジェネリックの書き方が存在型に比べて複雑で、不必要な場所で存在型を導入して余計なコストを費やしている場合がある。

例えば、

- ジェネリックのシンタックスはかっこだらけ。型変数に型を指定しなければならない。
- 具体的な実装からジェネリックを使った抽象化をするのが難しい。

```swift
func concatenate(a: [Int], b: [Int]) -> [Int] {
  var result: [Int] = []
  result.append(a)
  result.append(b)
  return result
}

↓

func concatenate<T>(a: [T], b: [T]) -> [T] {
  var result: [T] = []
  result.append(a)
  result.append(b)
  return result
}

↓

func concatenate<A: Collection, B: Collection>(a: A, b: B) -> [A.Element]
  where A.Element == B.Element
{
  var result: [A.Element] = []
  result.append(a)
  result.append(b)
  return result
}
```

##### 解決策

- 引数や戻り値に直接制約を書けるようにしたい  

※ シンタックスはイメージ

```swift
func concatenate<T>(a: some Collection<.Element == T>, b: some Collection<.Element == T>)
  -> some Collection<.Element == T>
```

関連スレッド: https://forums.swift.org/t/discussion-easing-the-learning-curve-for-introducing-generic-parameters/52891

- 存在型を使っていることを明確にして誤用を防ぎたい  
※ これは別で取り上げる

関連スレッド: https://forums.swift.org/t/se-0335-introduce-existential-any/53934/125  

## 参考リンク

- https://forums.swift.org/t/improving-the-ui-of-generics/22814
- https://docs.swift.org/swift-book/LanguageGuide/Generics.html
- https://github.com/apple/swift/blob/main/docs/GenericsManifesto.md
- https://forums.swift.org/t/pitch-light-weight-same-type-constraint-syntax/52889
- https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md
- https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md
- https://forums.swift.org/t/se-0335-introduce-existential-any/53934/125
- https://forums.swift.org/t/discussion-easing-the-learning-curve-for-introducing-generic-parameters/52891

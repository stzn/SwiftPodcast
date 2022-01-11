# Swift ジェネリックのこれから。今後の方向性と現在の状況

収録日: 2022/01/09

- [Swift ジェネリックのこれから。今後の方向性と現在の状況](#swift-ジェネリックのこれから今後の方向性と現在の状況)
  - [概要](#概要)
  - [内容](#内容)
    - [ジェネリックとは？](#ジェネリックとは)
    - [Generics Manifesto](#generics-manifesto)
    - [用語紹介](#用語紹介)
      - [型レベルの抽象化(ジェネリック制約)](#型レベルの抽象化ジェネリック制約)
      - [値レベルの抽象化(存在型)](#値レベルの抽象化存在型)
      - [Swiftのジェネリックにはどちらもプロトコルが使われている](#swiftのジェネリックにはどちらもプロトコルが使われている)
    - [ジェネリックの改善に必要なこと](#ジェネリックの改善に必要なこと)
      - [値レベルの抽象化の制限](#値レベルの抽象化の制限)
        - [問題点](#問題点)
        - [解決案](#解決案)
      - [戻り値で型レベルの抽象化ができない](#戻り値で型レベルの抽象化ができない)
        - [問題点](#問題点-1)
        - [解決策](#解決策)
      - [ジェネリック制約の書き方が複雑](#ジェネリック制約の書き方が複雑)
        - [問題点](#問題点-2)
        - [解決案](#解決案-1)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [関連PR](#関連pr)
    - [その他](#その他)

## 概要

以前よりジェネリックの進むべき方向としてGenerics Manifestoが存在していた。現在では、コアな機能の実装は完了している。
Generics Manifestoでまだ未実装の部分の実装を進めるにあたって、足りない機能や再考が必要と考えられる機能などについて、ロードマップとして示された。

関連スレッド: https://forums.swift.org/t/improving-the-ui-of-generics/22814

## 内容

※今回は全体の概要を紹介して、個々の具体的な機能については別途

※ 下記で登場するコードの多くはイメージで実際のコードとは異なります。

### ジェネリックとは？

プログラミング言語の機能・仕様の一つで、同じプログラムコードで様々なデータ型のデータを処理できるようにするもの。ジェネリックコードを使用すると、定義した要件に応じて、任意の型で機能する柔軟で再利用可能な関数と型を記述できる。重複を避け、その意図を明確で抽象的な方法で表現するコードを書くことができる。

関連ドキュメント: https://docs.swift.org/swift-book/LanguageGuide/Generics.html

### Generics Manifesto

Swift 3 リリース時のSwiftのジェネリックの足りない機能や改善点などについてのSwift Core teamやdeveloperの話し合いの結果をまとめたもの。

関連ドキュメント: https://github.com/apple/swift/blob/main/docs/GenericsManifesto.md

### 用語紹介

#### 型レベルの抽象化(ジェネリック制約)

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

#### Swiftのジェネリックにはどちらもプロトコルが使われている

Swiftでは、型レベルの抽象化、値レベルの抽象化のどちらにもプロトコルが使われている。

※ このことが誤用を招いたり、理解を難しくしているとされ、下記に紹介する問題の一つとなっている。

### ジェネリックの改善に必要なこと

`Collection`プロトコルを使って考えてみる。

```swift
func bar(x: Collection, y: Collection) -> [Collection] { ... }
```

※ 上記のコードは現在はエラーになる。プロトコルは下記に当てはまる場合に型として使用できない(いわゆるPAT問題)

- 関連型(associated type)が要件に含まれている
- メソッド/プロパティ/subscript/イニシャライザの共変ではない位置(引数など)に`Self`への参照が含まれている

しかし、この制限は緩和され、全てのプロトコルを存在型で使用可能できるようになる(実装待ち)。そのため、これが使用可能だとして話を進める。

関連ドキュメント: https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md

#### 値レベルの抽象化の制限

##### 問題点

型情報を保持する型レベルの抽象化とは異なり、値の型同士の関係が保持されていないため、具体的な型を使った機能が使えない(例えば、関数の引数と戻り値が実装レベルで同じだったとしても型情報からはそれがわからない)。そのため、型レベルの抽象化に比べてできることが制限されている。

```swift
// Indexの型がわからないので型安全ではない
var start = x.startIndex

// 別の型に変更できてしまう可能性がある
// start = y.startIndex
var firstValue = x[start] // error

// 同じようにElementの型もわからないので型安全ではない
var first = x.first!
var indexOfFirst = x.index(of: first) // error

```

##### 解決案

そこでいくつかの解決案が出されている(既に実装が進んでいるものもある)

- 関連型を存在型に指定できるようにする

こうすることで、下記の関数では引数と戻り値の`Element`の型が同じある制約を設定できる。

```swift
typealias CollectionOf<T> = Collection where Self.Element == T
func bar<T>(x: CollectionOf<T>, y: CollectionOf<T>) -> [CollectionOf<T>] { ... }
```

関連スレッド: https://forums.swift.org/t/pitch-light-weight-same-type-constraint-syntax/52889

- 存在型はそれを制約するプロトコル自身に準拠できるようにする

上記の例では`Index`の型がわからないため、全てのAPIを使用することはできない。一方で`Index`を指定すると具体的過ぎて、存在型としての利便性が損なわれてしまう。

そこで存在型にある制約を加えてそのプロトコル自身に準拠できるようにする。

```swift
extension any Collection: Collection {
  // Comparableに準拠した任意の型を引数に値を取得する
  subscript(index: Comparable) -> Any {
    // Indexは現在のCollectionのIndexの型と合致していることは事前にわかっているので
    // キャストしてそのCollectionの型にキャストすることができる
    return self[index as! Self.Index]
  }
}
```

これまでは型消去(type eraser)用の`AnyCollection<T>`と`AnyIndex<T>`を使って上記と同じことができたが、それが不要になる。

- 存在型から具体的な型へ「展開」できるようにする

ローカル変数に再代入する際に、具体的な型として再度利用できるようにする。こうすることで、単一の存在型の関連型を使った処理ができる。

```swift
let <X: Collection> openedX = x // Xはxの動的な型に紐づく
let start = openedX.startIndex
let first = openedX[start] // X.IndexからX.Elementを取得することができる
```

#### 戻り値で型レベルの抽象化ができない

※ これは既にSwift5.1のOpaque Return Typesによって一部解決されている。

上記のような問題を解決したとしても値レベルの抽象化でできないことがある。

##### 問題点

例えば、ジェネリック制約を使った場合は、型を抽象化して関数を定義しても、呼び出し側で具体的な型を特定することができる。

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

しかし、現状`Collection`は型として使えない(上記でも書いたが、今後使用可能になる予定)。そこでジェネリック制約を使おうとすることを考える。

```swift
func evenValues<C: Collection, Output: Collection>(in collection: C) -> Output
  where C.Element == Int, Output.Element == Int
{
  return collection.lazy.filter { $0 % 2 == 0 }
}
```

これだと呼び出し側で戻り値が決められてしまい、実装側の意図と戻り値の型が異なる可能性もある。

そこで解決するためには型消去(Type eraser)する必要がある。

Standard Libraryには`AnyCollection`が用意されているので、それを使ってみる。

```swift
func evenValues<C: Collection>(in collection: C) -> AnyCollection<Int> where C.Element == Int {
    AnyCollection<Int>(collection.lazy.filter { $0 % 2 == 0 })
}


//補足: 将来的にはAnyCollectionはいらなくなる予定
func evenValues<C: Collection>(in collection: C) -> CollectionOf<Int>
  where C.Element == Int
{
  return collection.lazy.filter { $0 % 2 == 0 }
}
```

ただし、これらも正確ではなく、呼び出されるたびに異なる`Collection`型を返し、その型情報は失われている。

例えば、下記はいずれも`Int`だとわかっているが、型情報が失われているでの加算ができない。

```swift

func callee() -> Numeric {
    Int(42)
}

func caller() {
    callee() + callee() // ❌ Binary operator '+' cannot be applied to two 'Numeric' operands
}
```

結局、戻り値で型レベルの抽象化が必要な場合がある。

##### 解決策

- Swift5.1でOpaque Result Typeが導入された  
関連ドキュメント: https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md

- Swift5.6でさらに使用可能範囲が拡張する  
関連ドキュメント: https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md

※ これは別で取り上げる。この際にリーバースジェネリクスというより広い概念についても紹介する。

#### ジェネリック制約の書き方が複雑

##### 問題点

例えば、

- 具体的な実装からジェネリック制約を使った抽象化をするのが難しい
- どの制約がどこに対する制約に行なっているのかがわかりづらい(where句)

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

これによって、他の問題を引き起こしている可能性がある。
Swiftでは、プロトコルを使ってジェネリック制約も存在型も実装している。そのため、この両方に使い方が不明瞭になりがちで、ジェネリック制約の書き方が存在型に比べて複雑であることから不必要な場所で存在型を導入して余計なコストを費やしている場合がある。また実装側がジェネリック制約を使っているつもりが、意図しない形で存在型を使用してしまっている可能性もある。

```swift

func foo<T: Collection, U: Collection>(x: T, y: U) -> <V: Collection> V

func foo(x: Collection, y: Collection) -> Collection
```

##### 解決案

- 型変数をなくす

型変数をなくして引数や戻り値に直接「ある条件に準拠する具体的な型」を指定できるようにする。Opaque Result Typesと一緒に導入された`some`キーワードを利用することなどが検討されている。

```swift

func concatenate(a: some Collection, b: some Collection) -> some Collection
```

- 引数や戻り値に直接制約を書けるようにしたい  

```swift
func concatenate<T>(a: some Collection<.Element == T>, b: some Collection<.Element == T>)
  -> some Collection<.Element == T>
```

関連スレッド: https://forums.swift.org/t/discussion-easing-the-learning-curve-for-introducing-generic-parameters/52891

関連PR:  
https://github.com/apple/swift/pull/40715  
https://github.com/apple/swift/pull/40714

- 存在型を使っていることを明確にして誤用を防ぎたい  
※ これは別で取り上げる

関連スレッド:  
https://forums.swift.org/t/se-0335-introduce-existential-any/53934/125  

## 参考リンク

### Forums

- [Improving the UI of generics](https://forums.swift.org/t/improving-the-ui-of-generics/22814)
- [[Pitch] Light-weight same-type constraint syntax](https://forums.swift.org/t/pitch-light-weight-same-type-constraint-syntax/52889)
- [SE-0335: Introduce existential any](https://forums.swift.org/t/se-0335-introduce-existential-any/53934)
- [[Accepted with modifications] SE-0335: Introduce existential any](https://forums.swift.org/t/accepted-with-modifications-se-0335-introduce-existential-any/54504)
- [[Discussion] Easing the learning curve for introducing generic parameters](https://forums.swift.org/t/discussion-easing-the-learning-curve-for-introducing-generic-parameters/52891)

### プロポーザルドキュメント

- [Opaque Result Types](https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md)
- [Structural opaque result types](https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md)
- [Unlock existential for all protocols](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)
- [Introduce existential any](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md)

### 関連PR

- [Structural opaque result types](https://github.com/apple/swift/pull/40710)
- [Named opaque result types](https://github.com/apple/swift/pull/40715)

### その他

- [LanguageGuide Generics](https://docs.swift.org/swift-book/LanguageGuide/Generics.html)
- [GenericsManifesto](https://github.com/apple/swift/blob/main/docs/GenericsManifesto.md)
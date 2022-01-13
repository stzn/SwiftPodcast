# Swift Opaque Result Typeの拡張とリバースジェネリック

- [Swift Opaque Result Typeの拡張とリバースジェネリック](#swift-opaque-result-typeの拡張とリバースジェネリック)
  - [概要](#概要)
  - [内容](#内容)
    - [Opaque Result Typeとは？](#opaque-result-typeとは)
    - [Swift5.6で何が変わる？](#swift56で何が変わる)
      - [構造的位置(structural position)とは?](#構造的位置structural-positionとは)
      - [問題点](#問題点)
    - [解決策](#解決策)
      - [Optionalの書き方](#optionalの書き方)
      - [高階関数(Higher order functions)](#高階関数higher-order-functions)
      - [ジェネリック制約の推論](#ジェネリック制約の推論)
    - [リバースジェネリック(Reverse generics)](#リバースジェネリックreverse-generics)
      - [通常のジェネリック](#通常のジェネリック)
      - [リバースジェネリックの場合](#リバースジェネリックの場合)
      - [通常のジェネリックとリバースジェネリック比較](#通常のジェネリックとリバースジェネリック比較)
        - [ジェネリック型Tについて見てみると、callerとcallee関数の役割が**逆になった(Reverseした)**](#ジェネリック型tについて見てみるとcallerとcallee関数の役割が逆になったreverseした)
        - [ジェネリック型の参照先が異なる](#ジェネリック型の参照先が異なる)
        - [どっちの関数(caller or callee)がジェネリックになるかが異なる](#どっちの関数caller-or-calleeがジェネリックになるかが異なる)
        - [ジェネリック型は具体的な型のプレースホルダなのは共通](#ジェネリック型は具体的な型のプレースホルダなのは共通)
        - [特殊化(Specialization)](#特殊化specialization)
      - [通常のジェネリックとリバースジェネリックの組み合わせ](#通常のジェネリックとリバースジェネリックの組み合わせ)
      - [where句を使った型制約も同じように設定できる(可能性がある)](#where句を使った型制約も同じように設定できる可能性がある)
      - [より複雑な型も同じように扱える](#より複雑な型も同じように扱える)
    - [将来的な話](#将来的な話)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [関連PR](#関連pr)
    - [その他](#その他)

## 概要

Swift5.1で導入されたOpaque Result Typeが構造的位置(structural position)でも使えるようになった。

※ structural positionとは?
`String`や`int`などの全体の型ではなく、ジェネリックの型変数やTuple、関数型の一部などで型を指定する位置を指す

## 内容

### Opaque Result Typeとは？

Opaque Typeとは、具体的な型の情報を保持しつつ、具体的な型からは抽象化された方法で提供される型。  

https://docs.swift.org/swift-book/LanguageGuide/OpaqueTypes.html

Opaque Typeを関数やsubscriptの戻り値、変数の型に使用したものをOpaque Result Typeと呼ぶ。こうすることで、関数の呼び出し側からは抽象化された方法で、関数の実装側で返す値の型を選択できる。

SwiftUIの`View`プロトコルの`body`プロパティなどで使われている。

```swift
var body: some View { ... }
```

メリット

- 呼び出し側に不要な詳細情報が漏れない
- 実装側で実装を変更しても戻り値が抽象化されているため、実装の変更が呼び出し側に影響を与えない
- (現状)associated typeやSelfを要件に使っている場合に存在型(プロトコルを型として使用した型)として使用できない場面でも使用できる

デメリット

- 戻り値は全実行パスで同じ型である必要がある(SwiftUIでは、これを解決するためにiOS14でViewBuilderが導入された)  
https://developer.apple.com/documentation/swiftui/viewbuilder

### Swift5.6で何が変わる？

#### 構造的位置(structural position)とは?

`String`や`Int`などの型全体ではなく、ジェネリックの型変数やTuple、関数型の一部などで型を指定する位置を指す。

https://docs.swift.org/swift-book/ReferenceManual/Types.html

#### 問題点

現在はこの構造的位置にOpaque Result Typeが使えない。

```swift

func f0() -> (some P)? { /* ... */ } // ❌

func f1() -> (some P, some Q) { /* ... */ } // ❌

func f2() -> () -> some P { /* ... */ } // ❌

func f3() -> S<some P> { /* ... */ } // ❌
```

### 解決策

この制限を解除する。

#### Optionalの書き方

`(some P)?`、`(some P)!`と書く。
`some P?`と書くと`some Optional<P>`となってエラーになる。Opaque Typeは`Any`, `AnyObject`、プロトコル合成、加えて/またはbase classに制限されなければならない。

#### 高階関数(Higher order functions)

関数型の引数と戻り値どちらにも使える。

```swift
func f() -> () -> some P
func g() -> (some P) -> ()
```

#### ジェネリック制約の推論

型変数のジェネリック制約が関数のシグニチャの構造的位置で使用される場合、コンパイラは使用されるコンテキストに基づいて型変数のジェネリック制約を暗黙的に制約する。

```swift
struct H<T: Hashable> { init(_ t: T) {} }
struct S<T>{ init(_ t: T) {} }

// 'f<T: Hashable>'と同じ。 `H<T>`は暗黙的に`T: Hashable'と推論される
func f<T>(_ t: T) -> H<T> {
    var h = Hasher()
    h.combine(t) // OK - 'T: Hashable'だとわかる
    let _ = h.finalize()
    return H(0)
}

// 'S<T>'は'T'について何も示していない
func g<T>(_ t: T) -> S<T> {
    var h = Hasher()
    h.combine(t) // ERROR - instance method 'combine' requires that 'T' conform to 'Hashable'
    let _ = h.finalize()
    return S(0)
}
```

Opaque Result Typeでは、この型推論は効かない。

```swift
// ERROR - type 'some P' does not conform to protocol 'Hashable'
func f<T>(_ t: T) -> H<some P> { /* ... */ }
```

### リバースジェネリック(Reverse generics)

Opaque Result Typeは、通常のジェネリックとは異なり、リバースジェネリックと呼ばれている。

リバースジェネリックとは、通常のジェネリクスと概念としては同じだけど方向が、**逆になる(Reverseする)**ジェネリックを指す。

#### 通常のジェネリック

では、何が逆なのか、を理解するために、まず通常のジェネリックについて見てみる。

```swift
// 通常のジェネリック
func callee<T: Numeric>(_ x: T) {
  // ここではxの具体的な型はわからない
  // そのため、Numericプロトコルの要件で定義されたことしかできない
}

func caller() {
  // callee関数をp呼び出す際、callerがTの具体的な型を決める
  // callee関数に渡す型はNumericプロトコルに準拠していれば良い
  // この場合、Tの具体的な型はIntになる
  callee(42) 
}
```

#### リバースジェネリックの場合

では、上記の関数をリバースジェネリックに変換するにはどうすればよいのか。
2つのことが必要になる。

1. 関数の入力と出力を切り替える
2. ジェネリックの型パラメータをリバースジェネリックにする

Opaque Result Typeを使って考えてみる。

```swift

// リバースジェネリック
func callee -> some Numeric {
  // calleeがTの具体的な型を決める
  // conforms to the Numeric protocol. In this case the concrete type
  // この場合、Tの具体的な型はIntになる
  return 42
}

func caller() {
  let x = callee()
  // ここではxの具体的な型はわからない
  // そのため、Numericプロトコルの要件で定義されたことしかできない
}
```

#### 通常のジェネリックとリバースジェネリック比較

##### ジェネリック型Tについて見てみると、callerとcallee関数の役割が**逆になった(Reverseした)**

上記の関数では、

- 通常のジェネリック: **`caller`**関数が具体的な型のIntを使って処理をする。**`callee`**関数はあるNumericプロトコルに準拠した型を使って処理をする。
- リバースジェネリック: **`callee`**関数が具体的な型のIntを使って処理をする。**`caller`**関数はあるNumericプロトコルに準拠した型を使って処理をする。

※ 下記は`caller`関数で具体的な型を決められないため正しく機能しない

```swift

func callee<T: Numeric>() -> T {
    return 42 // Generic parameter 'T' could not be inferred
}

func caller() {
    let x = callee()
}
```

##### ジェネリック型の参照先が異なる

通常のジェネリックは、ジェネリック関数からジェネリックの型を参照することができる。

```swift
// 通常のジェネリック
func callee<T: Numeric>(_ x: T) {
  var y: T // T型の別の変数を宣言する
  y = x    // この関数内では、具体的な型はわからないものの、xとyが同じ型だということはわかるので代入できる
}
```

リバースジェネリックでジェネリック関数からジェネリックの型を参照するには下記のようになる(現状できないのでイメージとして)。

```swift
// リバースジェネリック
func callee() -> some Numeric {
  return 42
}

func caller() {
  let x = callee()
  var y: callee.T // callee.T型の別の変数を宣言する
  y = x  // この関数内では、具体的な型はわからないものの、xとyが同じ型だということはわかるので代入できる
}
```

##### どっちの関数(caller or callee)がジェネリックになるかが異なる

- 通常のジェネリック: 関数はジェネリックの型パラメータに対してジェネリックである(`callee`関数はTに対してジェネリック)
- リバースジェネリック: ジェネリックなのは関数自体ではなく、`caller`がジェネリックになる(`caller`関数は`callee.T`に対してジェネリック)

##### ジェネリック型は具体的な型のプレースホルダなのは共通

どちらの場合もジェネリック型は具体的な型のプレースホルダで、同じコンテキスト内で突然型が変わることはない。

```swift
// 通常のジェネリック
func callee<T: Numeric>(_ x: T, _ y: T) { ... }

func caller() {
  callee(42, 42.0) // error: TはIntとDoubleのいずれかにしかできない  
}
```

リバースジェネリックも同様に戻り値は同じ型にする必要がある

```swift
// リバースジェネリック
func callee() -> some T {
  if Bool.random() {
    return 42
  } else {
    return 42.0 // error: TはIntとDoubleのいずれかにしかできない
  }
}
```

つまり、caller関数は`callee.T`が突然違う型になることはないということに依存している。

```swift
// リバースジェネリック
func caller() {
  let x = callee() + callee() // 両方のリバースジェネリッのcallee関数は
                              // 同じ具体的な型のcallee.Tを返す
                              // そしてNumericに準拠しているため、加算できる
}
```

※ これは存在型(Existential type)との重要な違い。存在型は毎回型が異なる可能性があるため、仮に`callee`関数が存在型を返した場合は、xとyは直接加算できない(動的にキャストする必要がある)。

##### 特殊化(Specialization)

通常のジェネリックの場合、コンパイラは具体的な型がわかる場合、コンパイル時にジェネリック型を具体的な型に置き換える。
リバースジェネリックでも同じだが、ジェネリックなのは`callee`ではなく、callerであるため、コンパイラが十分な情報を持っている場合は、callerが特殊化される。つまり、もしそれぞの関数が別のモジュールに存在する場合、`callee`関数から戻ってくる具体的な型の情報がわからないため、コンパイラは特殊化ができない(インライン化については今回考えない)。

#### 通常のジェネリックとリバースジェネリックの組み合わせ

両方を組み合わせて使うこともできる。

```swift
// Normal and reverse generics
func makeCollection<T>(with element: T) -> some Collection {
  return [element]
}
```

中身を見ていくと、

- `makeCollection`関数は引数で受け取るTの具体的な型はわからない。`T`は`caller`によって決められる。
- `caller`は戻り値の具体的な型がわからない。`makeCollection`関数が決める。
- `caller`は`some Collection`型が`Collection`に準拠していることはわかるため、`count`や`isEmpty`などのAPIを使用できる。

気になる点として、`makeCollection`関数が返す型は、引数に`Int`を渡すと`Int`の配列、引数に`String`を渡すと`String`の配列を返す。これは、上記で話した内容と一致しない。ちょっとあいまいだが、リバースジェネリックの型は通常のジェネリックの型に応じて固定されていると言える。つまり、毎回引数の型が`Int`で呼び出す場合、`makeCollection`関数が返す具体的な型はいつも同じである。
なので下記は機能する。

```swift
let c = makeCollection(with: 5) + makeCollection(with: 10)
```

#### where句を使った型制約も同じように設定できる(可能性がある)

これまで使っていたwhere句でも型制約を設定できるがより複雑になる可能性がある。例えば、上記の関数は、`caller`が引数に渡した型が必ず返ってくる保証はない。これをwhere句で制約を追加する。

```swift
func makeCollection<T>(with element: T) -> some Collection where Collection.Element == T {
  return [element]
}
```

これができると下記ができる。

```swift
let c = makeCollection(with: 5)
let d = c.map { $0 + 10 } // 引数と戻り値の型がIntだとわかっているためOK
```

#### より複雑な型も同じように扱える

リバースジェネリックはもっと複雑な型を返すこともできる。

```swift
// OptionalのCollectionを返す
func makeCollection<T: Numeric>(with element: T) -> (some Collection)? {
  guard element > 0 else { return nil }
  return [element]
}

// 同じ型の2つのCollectionを返す（Tuple）
func makeCollection<T>(with element: T) -> (some Collection, some Collection) {
  return ([element], [element])
}

// Elementが同じCollectionとCollectionの配列を返す
func makeCollection<T>(with element: T) -> (some Collection, some Collection) {
  return ([element] as Set, [[element] as Set])
}
```

異なる型を返すこともできるかもしれない(現状できないのでイメージ)

```swift
func makeCollections<T>(with element: T) -> <C: Collection where .Element == T, D: Collection where .Element == T>(some C, some D) {
  return ([element] as Set, [element])
}
```

関連スレッド: https://forums.swift.org/t/reverse-generics-and-opaque-result-types/21608  
関連PR: https://github.com/apple/swift/pull/40715

### 将来的な話

この拡張は汎用的なリバースジェネリックに向けたステップアップ。

より直感的な書き方ができるように`some`キーワードを引数でも利用できるようにしたり、関連型(associated type)の制約をOpaque Result Typesと一緒に書けるようにすることなどが検討されている。

```swift
func makeCollection(with number: some Numeric) -> some Collection {
  return [number]
}
```

関連スレッド:  
- https://forums.swift.org/t/discussion-easing-the-learning-curve-for-introducing-generic-parameters/


```swift
func concatenate<T>(a: some Collection<.Element == T>, b: some Collection<.Element == T>) -> some Collection<.Element == T>
```

もっと簡単に書く方法なども。

```swift
func concatenate<T>(a: some Collection<T>, b: some Collection<T>) -> some Collection<T>
```

関連スレッド:  

- https://forums.swift.org/t/pitch-light-weight-same-type-constraint-syntax/52889


Opaque Result Typesについて残りのタスクについての記載もある  

関連スレッド: https://forums.swift.org/t/future-work-on-opaque-result-types/50999

## 参考リンク

### Forums

- [Structural opaque result types](https://forums.swift.org/t/se-0328-structural-opaque-result-types/53248)
- [Reverse generics and opaque result types](https://forums.swift.org/t/reverse-generics-and-opaque-result-types/21608)
- [[Pitch] Light-weight same-type constraint syntax](https://forums.swift.org/t/pitch-light-weight-same-type-constraint-syntax/52889)
- [[Discussion] Easing the learning curve for introducing generic parameters](https://forums.swift.org/t/discussion-easing-the-learning-curve-for-introducing-generic-parameters/52891)
- [Future work on opaque result types](https://forums.swift.org/t/future-work-on-opaque-result-types/50999)

### プロポーザルドキュメント

- [Opaque Result Types](https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md)
- [Structural opaque result types](https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md)

### 関連PR

- [Structural opaque result types](https://github.com/apple/swift/pull/40710)

### その他

- [LanguageGuide Opaque Types](https://docs.swift.org/swift-book/LanguageGuide/OpaqueTypes.html)
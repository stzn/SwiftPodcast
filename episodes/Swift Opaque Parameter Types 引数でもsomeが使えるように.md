# Swift Opaque Parameter Types 引数でもsomeが使えるように

収録日: 2022/03/04

- [Swift Opaque Parameter Types 引数でもsomeが使えるように](#swift-opaque-parameter-types-引数でもsomeが使えるように)
  - [概要](#概要)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決策](#解決策)
    - [詳細(現状使えない場所)](#詳細現状使えない場所)
      - [関数型の「消費」ポジションのOpaque parameter](#関数型の消費ポジションのopaque-parameter)
      - [可変個ジェネリクス(Variadic generics)](#可変個ジェネリクスvariadic-generics)
    - [将来的な検討事項](#将来的な検討事項)
      - [プロトコルの関連型(associated types)に制約を加える](#プロトコルの関連型associated-typesに制約を加える)
      - [「消費」ポジションのOpaque typeを使えるようにする](#消費ポジションのopaque-typeを使えるようにする)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

Swiftのジェネリクス構文は、複雑な制約を表現できるように設計されているものの、シンプルな制約を設定したい場合、この制約の指定方法は必要以上に宣言が複雑になってしまう。今回Opaque Result Typeの構文を拡張してこの複雑さを減らす。Swift5.7で導入予定。

## 内容

### 問題点

Swiftのジェネリクス構文は、一般性を考慮して設計されており、関数のさまざまな入力と出力の間で複雑な制約のセットを表現できる。例えば下記の例では、2つのシーケンスから1つの配列を生成する即時連結処理を考えてみる。


```swift
func eagerConcatenate<Sequence1: Sequence, Sequence2: Sequence>(
    _ sequence1: Sequence1, _ sequence2: Sequence2
) -> [Sequence1.Element] where Sequence1.Element == Sequence2.Element
```

関数の宣言では色々なことが行われている:

- `Sequence1`と`Sequence2`としてキャプチャされている呼び出し元で決められた2つの異なる型を引数として受け取る
- それらは`Sequence`プロトコルに準拠していなければならない
- 2つの型のElementは同じ型でなければならない
- この操作の結果はシーケンスのElementの型の配列である

条件を満たせば、さまざまな型で使うことができる:

```swift
eagerConcatenate([1, 2, 3], Set([4, 5, 6]))  // ⭕️ [Int]
eagerConcatenate([1: "Hello", 2: "World"], [(3, "Swift"), (4, "!")]) // ⭕️ [(Int, String)]
eagerConcatenate([1, 2, 3], ["Hello", "World"]) // ❌ [Int]と[String]は違う
```

一方でこのような複雑な制約が不要な場合、この構文はとても負担に感じる。例えば、SwiftUIの2つのViewを水平に組み合わせる処理を考える:

```swift
func horizontal<V1: View, V2: View>(_ v1: V1, _ v2: V2) -> some View {
  HStack {
    v1
    v2
  }
}
```

一度しか使われていないジェネリックパラメータの`V1`と`V2`を宣言するのに多くのボイラープレートが必要で、実態以上に複雑に見える。

戻り値は[Opaque Result Types](https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md)を使うことができるので、戻り値特定の型を隠すことができる。  
Opaque Result Typesについては以前取り上げている  
[Swift Opaque Result Typeの拡張と戻り値の新しい書き方](../episodes/Swift%20Opaque%20Result%20Typeの拡張と戻り値の新しい書き方.md)

### 解決策

Opaque result typeの構文をジェネリックな関数の引数でも使えるようにする(ジェネリックパラメータを用いたボイラープレートはいらなくなる)。

```swift

func horizontal(_ v1: some View, _ v2: some View) -> some View {
  HStack {
    v1
    v2
  }
}
```

`some`キーワードを関数、イニシャライザ、subscript宣言の引数の型にも使えるようにする。`some P`はプロトコル`P`を満たす型であるという制約だけがわかる匿名の型である。これが引数に出てきた場合、匿名のジェネリックパラメータに置き換わる。

```swift
func f(_ p: some P) { }
```

は、下記と同じ。

```swift
func f<_T: P>(_ p: _T)
```

注意が必要な点は、このopaque typeの型は型推論を通して**呼び出し元**で決められる。逆に戻り値に`some`キーワードを使うと、**呼び出し先(実装側)**で決められる。

```swift
f(17)      // ⭕️ opaque typeはInt
f("Hello") // ⭕️ opaque typeはString

let fInt: (Int) -> Void = f       // ⭕️ opaque typeはInt
let fString: (String) -> Void = f // ⭕️ opaque typeはString
let fAmbiguous = f                // ❌ some Pが推論できない
```

任意の構造的位置に複数の`some P`を使うこともできる。[SE-0328](https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md)

```swift
func encodeAnyDictionaryOfPairs(_ dict: [some Hashable & Codable: Pair<some Codable, some Codable>]) -> Data
```

は、下記と同じ。


```swift
func encodeAnyDictionaryOfPairs<_T1: Hashable & Codable, _T2: Codable, _T3: Codable>(_ dict: [_T1: Pair<_T2, _T3>]) -> Data
```

### 詳細(現状使えない場所)

#### 関数型の「消費」ポジションのOpaque parameter

Opaque parameter typeは関数、イニシャライザ、subscript宣言の引数のみに使える。タイプエイリアスや関数型の引数には使えない(関数型の「消費」ポジションと呼ばれている)。

```swift
typealias Fn = (some P) -> Void    // ❌ cannot use opaque types in a typealias
let g: (some P) -> Void = f        // ❌ cannot use opaque types in a value of function type
```

[SE-0328](https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md)で戻り値の関数型の「消費」ポジションのOpaque parameterは禁止された。

```swift
func f() -> (some P) -> Void { ... } // ❌ cannot use opaque type in parameter of function type
```

これは、呼び出し元がこの未知の匿名の型の値を生成する簡単な方法が存在しないため、`f`を使うのがとても難しい。

```swift
let fn = f()
fn(/* どの型の値を渡せばよいのか？ */)
```

同じことが関数型のパラメータの中のOpaque typeも適用される。

```swift
func g(fn: (some P) -> Void) { ... } // ❌ cannot use opaque type in parameter of function type
```

#### 可変個ジェネリクス(Variadic generics)

opaque typeは可変個ジェネリクスに使えない。

```swift
func acceptLots(_: some P...)
```

可変個ジェネリクスは現在別で提案されているが、Swiftが可変個引数のジェネリックを取得した場合、このプロポーザルのセマンティクスが適切ではない可能性があるため、この制限が適用される。

例えば、このプロポーザル自身が提案している方法で下記を推論した場合(可変個ジェネリクスが存在しないとして)、

```swift
func acceptLots<_T: P>(_: _T...)
```

この関数は一つの型しか受け入れられない。

```swift
acceptLots(1, 1, 2, 3, 5, 8)          // ⭕️
acceptLots("Hello", "Swift", "World") // ⭕️
acceptLots("Swift", 6)                // ❌  argument for `some P` could be either String or Int
```

可変個ジェネリクスが導入されると、この構文は暗黙のジェネリックパラメータを一つのジェネリックパラメータのパックとする。

```swift
func acceptLots<_Ts: P...>(_: _Ts...)
```

この場合、この関数は異なる型を受け入れることができる。

```swift
acceptLots(1, 1, 2, 3, 5, 8)          // ⭕️ Ts 6個のInt型を含む
acceptLots("Hello", "Swift", "World") // ⭕️ Ts 3個のString型を含む
acceptLots(Swift, 6)                  // ⭕️ Ts StringとInt型を含む
```

### 将来的な検討事項

#### プロトコルの関連型(associated types)に制約を加える

このプロポーザルの内容は、プロトコルの関連型(associated types)を特定するジェネリック構文で使えるようにする考えとも上手く組み合う。例えば、`Collection<String>` は「Elementが`String`の`Collection`」。このプロポーザルと組み合わせて、もっと簡単に任意の文字列のコレクションを関数の引数に受け取ることができる。

```swift
func takeStrings(_: some Collection<String>) { ... }
```

冒頭で例として出した複雑な`eagerConcatenate` も、このopaque parameterとプロトコルのジェネリック構文を使って、一つのジェネリックパラメータを使ってより簡単に表現できる。

```swift
func eagerConcatenate<T>(
    _ sequence1: some Sequence<T>, _ sequence2: some Sequence<T>
) -> [T]
```

Opaque result typeとも合わせて戻り値も隠すことができる

```swift
func lazyConcatenate<T>(
    _ sequence1: some Sequence<T>, _ sequence2: some Sequence<T>
) -> some Sequence<T>
```

この構文に関しては[[Pitch 2] Light-weight same-type requirement syntax](https://forums.swift.org/t/pitch-2-light-weight-same-type-requirement-syntax/55081)で話されている。

#### 「消費」ポジションのOpaque typeを使えるようにする

引数でも戻り値でも「消費」ポジションのOpaque typeを禁止しているのは、どちらの場合も呼び出し元と呼び出し先が間違った引数の型を選択してしまい、現在の構文だと役に立たないから。これを「消費」ポジションのOpaque typeの方の選択を「反転」させることで可能にすることができると思われる。これを理解するために、opaque result typesを「リバースジェネリクス」と見なして、関数の`->`の後にジェネリックパラメータのリストが指定できて、呼び出し先が方を選択することができるとする。

```swift
func f1() -> some P { ... }
// リバースジェネリクスバージョンに変換すると...
func f1() -> <T: P> T { /* 呼び出し先の実装がTの具体的な型を決める */ }
```

戻り値の関数型の「消費」ポジションのopaque typeの問題は、呼び出し先が具体的な型を決めるので、呼び出し先がそれを判断できないこと。

```swift
func f2() -> (some P) -> Void { ... }
// リバースジェネリクスバージョンに変換すると...
func f2() -> <T: P> (T) -> Void { /* 呼び出し先の実装がTの具体的な型を決める */ }
```

「消費」ポジションにあるopaque typeを`->`の反対側に変換することで、この呼び出し元/呼び出し先の選択を「反転」させることができる。

```swift
// 仮に「消費」ポジションにあるopaque typeを反転して場合
func f2() -> (some P) -> Void { ... }
// 下記のように変換される
func f2<T: P>() -> (T) -> Void { ... }
```

こうすると呼び出し元が`T`を決められるのでより有用。呼び出し先は呼び出し元が決めた任意の型で機能するクロージャを汎用的に提供することができる。

```swift
let fn1: (Int) -> Void = f2() // ⭕️ T == Int
let fn2: (String) -> Void = f2() // ⭕️ T == String
```

関数のパラメータの関数型の「消費」ポジションにあるopaque typeも同じロジックが適用できる。

```swift
func g2(fn: (some P) -> Void) { ... }
```

「通常の」ジェネリクスだと、全然役に立たない

```swift
// 「通常の」ジェネリクスとして変換した場合
func g2<T: P>(fn: (T) -> Void) { /* fnの呼び出しに使うTは何？ */}
```

呼び出し元が`T`を決めるので、呼び出し先がそれを事実上使うことができない。これも「消費」ポジションにあるopaque typeを`->`の反対側に変換してジェネリクスを「反転」させる。

```swift
// 仮に「消費」ポジションにあるopaque typeを反転して場合
func g2(fn: (some P) -> Void) { ... }
// 下記のように変換される
func g2(fn: (T) -> Void) -> <T> Void { ... }
```
`g2`の実装側(呼び出し先)で`T`を決めることができ、`fn`に`T`の型の値を渡すことができるため、適切である。呼び出し先は、`P`に準拠した任意の`T`を受け入れることができるクロージャやジェネリック関数を提供する必要がある。型を明示することはできないが、型推論によって確実に利用できる。

```swift
g2 { x in x.doSomethingSpecifiedInP() }
```

## 参考リンク

### Forums

- [[Pitch] Opaque parameter types](https://forums.swift.org/t/pitch-opaque-parameter-types/54914)
- [SE0341: Opaque Parameter Declarations](https://forums.swift.org/t/se0341-opaque-parameter-declarations/55082)
- [[Accepted with Modifications] SE-0341: Opaque Parameters](https://forums.swift.org/t/accepted-with-modifications-se-0341-opaque-parameters/55397)
- [[Pitch 2] Light-weight same-type requirement syntax](https://forums.swift.org/t/pitch-2-light-weight-same-type-requirement-syntax/55081)で話されている。

### プロポーザルドキュメント

- [Opaque parameter types](https://github.com/apple/swift-evolution/blob/main/proposals/0341-opaque-parameters.md)
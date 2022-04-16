# Swift 引数のデフォルト式からジェネリックパラメータの型を推論する

収録日: 2022/04/09

- [Swift 引数のデフォルト式からジェネリックパラメータの型を推論する](#swift-引数のデフォルト式からジェネリックパラメータの型を推論する)
  - [用語紹介](#用語紹介)
  - [概要](#概要)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決策](#解決策)
    - [詳細](#詳細)
    - [検討された代替案](#検討された代替案)
    - [将来の検討事項](#将来の検討事項)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [関連PR](#関連pr)

## 用語紹介

デフォルト式: 具体的に型指定されたデフォルト引数の値

## 概要

現在、ジェネリックパラメータ型の引数にデフォルト式を使用することはできない。

```swift
func compute<C: Collection>(_ values: C = [0, 1, 2]) { ❌
    ...
}
```

この宣言をコンパイルしようとすると、次のコンパイルエラーが発生する `default argument value of type '[Int]' cannot be converted to type 'C'`。これは、現在のセマンティックルール上、デフォルト式の型は 呼び出し側で指定して推論された全ての`C`の型に対して置換可能ではないため。

この表現度の制限を回避する方法はいくつかあるが、いずれもオーバーロードが必要であり、APIが複雑になる。

```swift
func compute<C: Collection>(_ values: C) { // デフォルト式なしの元の宣言
    ...
}

func compute(_ values: [Int] = [0, 1, 2]) { // 具体的な型を指定したオーバーロード
    ...
}
```

そこで、呼び出し側で明示的な引数が省略された場合に、具体的な型のデフォルト式からジェネリックパラメータの型の推論を可能にする。ただし、デフォルト式を持つ引数に紐づくジェネリックパラメータが、呼び出し側から与えられる暗黙的あるいは明示的な引数によって、他の引数から推論できてしまう場合、デフォルト式はコンパイルエラーになる(型の衝突が起きてしまうと言いたいのだと思っている)。例えば

```swift
func compute<T, U>(_:　T = 42, _: U) where U: Collection, U.Element == T
```

はコンパイルできない。なぜなら2番目の引数から`T`の型を推論することができてしまうからである。


一方で

```swift
func compute<T, U>(_: T = 42, _: U = []) where U: Collection, U.Element == Int
```

は`T`と`U`が独立しているためコンパイル可能。

## 内容

### 問題点

デフォルト式がジェネリックパラメータの具体的な型の特殊化(※)でのみ機能するとしたら、ジェネリックパラメータとデフォルト式の間の相互作用は混乱を招く。(状況によっては)現在の言語機能を使って実現可能だが、ボイラープレートコードと条件付き`extension`に関する知識が必要となる。

※ ジェネリックパラメータに具体的な型を当てはめてクラスや関数をインスタンス化すること

例えば、デフォルトのフラグセットの`Flag`プロトコルとそのコンテナタイプを定義する:

```swift
protocol Flags {
    ...
}

struct DefaultFlags : Flags {
    ...
}
```

次に、フラグのセットを引数に受け入れるイニシャライザを宣言する:

```swift

struct Box<F: Flags> {
    init(dimensions: ..., flags: F) {
        ...
    }
}
```

次に、フラグのセットを引数に受け入れるイニシャライザを持つ型を宣言する:

`Box`を作成するには、呼び出し側は、`Flags`に準拠する型のインスタンスをイニシャライザの呼び出し時に渡す必要がある。`Box`の大部分が特別なフラグを必要としない場合、このAPIの使い勝手は標準よりも悪くなる。`DefaultFlags`型があっても、現状、`flags`パラメータにデフォルト式を提供することはできない。(`flags: F = DefaultFlags()`)。これを行おうとすると、次のエラーが発生する:

```swift
error: default argument value of type 'DefaultFlags' cannot be converted to type 'F'
```

これは、`DefaultFlags`が`Flags`に準拠していても、`F`が`DefaultFlags`の場合にのみ有効で、呼び出し側で推論できる全ての可能な`F`にこのデフォルト値を使用できないために発生する。

このflagsを渡さなくするためには、条件付き`extension`とオーバーロードの組み合わせを介して、具体的な型の`F`にイニシャライザを「特殊化」することができる。

まずは直接`where`句を使う場合から始める:

```swift
struct Box<F: Flags> {
    init(dimensions: ..., flags: F = DefaultFlags()) where F == DefaultFlags {
        ...
    }
}
```

このイニシャライザがあると`Box`のメンバワイズイニシャライザは使えなくなる。

他の手段としては`F`を具体的な`DefaultFlags`にする条件付き`extension`などがある:

```swift
extension Box where F == DefaultFlags {
    init(dimensions: ..., flags: F = DefaultFlags()) {
        ...
    }
}
```

`flags:`なしの`Box`のイニシャライザは、暗黙的なメンバワイズイニシャライザも保持されるようになる。ただし、`init`はオーバーロードされているものの、このアプローチはジェネリックパラメータがメンバ自体に属する状況では機能しない。

例えば、異なるフラグのセットを渡す必要がある`Box`のメソッドがあると考えてみる:

```swift
extension Box {
    func ship<F: Flags>(_ flags: F) {
        ...
    }
}
```

この場合、条件付き`extension`を使用する前述のアプローチは機能しない。ジェネリックパラメータ`F`が`Box`型ではなくメソッド`ship`に関連付けられているからである。この場合に機能する別のトリックがある。オーバーロードを使う。

この場合、新しいメソッドは、`flags:`が具体的な型を持っている必要がある:

```swift
extension Box {
    func ship(_ flags: DefaultShippingFlags = DefaultShippingFlags()) {
        ...
    }
}
```

そのパラメータが宣言に含まれているかどうかによって、機能するものとしないものがあり、使いやすさが欠如している。この不整合を解消するために、API作成者は、コンパイラによって受け入れられるからという理由で存在型を使うという結論に至ることがあるが、必要となる可能性のある全ての結果を認識できない可能性がある:

```swift

extension Box {
    func ship(_ flags: any Flags = DefaultShippingFlags()) {
        ...
    }
}
```

また、`enum`宣言の場合、存在型を使用せずに`flags:`引数にデフォルト式を紐づける方法がない:

```swift
enum Box<F: Flags> {
    // ❌: Default argument value of type 'DefaultFlags' cannot be converted to type 'F'
    case flatRate(dimensions: ..., flags: F = DefaultFlags())
}

extension Box where F == DefaultFlags {
    // ❌: enum 'case' is not allowed outside of an enum
    case flatRate(dimensions: ..., flags: F = DefaultFlags()) 
}
```

要約すると、デフォルト式に関連する表現に制限がある。これは、特定の状況でのみ条件付き`extension`を介して軽減できる。その他の問題には、次のものがある:

1. 条件付き`extension`は型に対してのみ宣言できるため、関数、subscript、または`case`に紐づいたジェネリックパラメータでは機能しない。例えば、`init<T: Flags>(..., flags: F = F()) where F == DefaultFlags`はできない
2. メソッドをオーバーロードする必要がある。これにより、`Box`のAPIが増加し、もしデフォルト式が必要なパラメータの組み合わせ以上のものが存在する場合、オーバーロードのマトリックスができてしまう。例えば、`dimensions`パラメータをジェネリックにして一部の箱の側面のみデフォルト式を設定する場合など
3. Swiftは`enum`の`case`のオーバーロードや`extension`での宣言をサポートしていないため、まったく機能しない
4. 条件付き`extension`やジェネリックパラメータを具体的な型に紐づけるといったノウハウを知っておく必要がある

### 解決策

前述の欠点に対処するために、より簡潔で直感的なシンタックスをサポートすることを提案する。具体的に型指定されたデフォルト式を、ジェネリックパラメータを参照するパラメータに関連付けることができるようになる:

```swift
struct Box<F: Flags> {
    init(flags: F = DefaultFlags()) {
        ...
    }
}

Box() // FはDefaultFlagsと推論される 
Box(flags: CustomFlags()) // FはCustomFlagsと推論される
```

この構文は、デフォルト式に関連付けられている型チェックの方法を修正して、明示的に渡された引数に干渉しない場合に、呼び出し側での型推論を可能にすることで実現できる。

### 詳細

次の場合、デフォルト式から型推論ができる:

1. ジェネリックパラメータが、パラメータの直接的な型、例えば`<T>(_: T = ...)`や、ネストされた位置位置、例えば`<T>(_: [T?] = ...)`で使用されている場合
2. ジェネリックパラメータが、パラメータリスト内の一箇所でのみ使用されている場合。例えば、`<T>(_: T, _: T = ...) `や`<T>(_: [T]?, _: T? = ...)`はできない。なぜならば、型の暗黙的な結合による予期しない動作を回避するために、型の競合を解決することができるのは、明示的に1つの引数のみに使われている場合だからである。(呼び出し側の指定によっては型の競合が発生する)  
    注: 結果の型は、デフォルト式から型推論できるジェネリックパラメータの型を参照することができ、ジェネリックなイニシャライザやジェネリックな`enum`の`case`を宣言していても使用できる
3. デフォルト式から推論できるジェネリックパラメータと、そのパラメータに関連するその他のパラメータが同じデフォルト式から型推論できる場合。例えば、`<T: Collection, U>(_: T = [...], _: U) where T.Element == U`は、`U`がデフォルト式で使用されている`T`と関連がないのでエラーになるが、`<K: Collection, V>(_: [(K, V?)] = ...) where K.Element == V`は、両方のジェネリックパラメータが一つの式に紐づいているため可能
4. デフォルト式が、呼び出し側で推論するために使用される各ジェネリックパラメータに設定された準拠要件、レイアウト、およびその他のジェネリック要件のすべてを満たす型を生成する場合

今回の更新により、`Box`型のイニシャライザと`ship`メソッドの両方を、条件付き`extension`やオーバーロードを必要としない、簡潔でわかりやすい方法で表現できる:

```swift
struct Box<F: Flags> {
    init(dimensions: ..., flags: F = DefaultFlags()) {
        ...
    }

    func ship<F: Flags>(_ flags: F = DefaultShippingFlags()) {
        ...
    }
}
```

また`Box`は、表現度を失うことなく`enum`に変換することもできる:

```swift
enum Box<D: Dimensions, F: Flags> {
    case flatRate(dimensions: D = [...], flags: F = DefaultFlags())
    case overnight(dimensions: D = [...], flags: F = DefaultFlags())
    ...
}
```

呼び出し側では、デフォルト式を持つ引数が指定されていない場合、型チェック中にデフォルト式の型からパラメータの型へ引数間で制約を変換する。これにより、全てジェネリックパラメータの型が常に推論されることが保証される。

```swift
let myBox = Box(dimensions: ...) // FはDefaultFlagsと推論される
myBox.ship() // FはDefaultShippingFlagsと推論される
```

デフォルト式とそれに対応するパラメータの型の間の関連を構築することが重要なのは、単に推論のためだけでなく、戻り値の型(同じジェネリックを参照することが許可されているジェネリックパラメータ型)との間で型の衝突がないことを保証するためでもあることに注目:

```swift
func compute<T: Collection>(initialValues: T = [0, 1, 2, 3]) -> T {
    // 初期値を使った複雑な計算
}

let result: Array<Int> = compute() ✅
// `initialValues`と戻り値の型が同じ型(Array<Int>)なのでOK

let result: Array<Float> = compute() ❌
// `initialValues`(Array<Int>)と戻り値の型(Array<Float>)が一致しないためエラー
```

### 検討された代替案

Generics Manifestoに記載されている[Default generic arguments](https://github.com/apple/swift/blob/main/docs/GenericsManifesto.md#default-generic-arguments)機能と、ここで提案されている型推論のルールと混同しないで欲しい。ジェネリック引数をデフォルトにする機能だけでは、ジェネリックパラメータが含まれる場合に、デフォルト式を使用する一貫した方法を提供するのに十分ではない。この提案で説明されている型推論は、パラメータが型パラメータを参照するときに具体的な型のデフォルト式を使用できるようにし、デフォルト式がデフォルトのジェネリックパラメータ型で機能するかどうかを判断するために必要。つまり、このデフォルトのジェネリックパラメータの機能は、代替アプローチではなく、拡張機能と見なせる。

Swiftフォーラムでは、同様のアプローチがいくつか議論されていた。そのうちの1つは、オーバーロード、条件付き`extension`、および/またはカスタム属性に依存する[[Pre-pitch] Conditional default arguments - #4 by Douglas_Gregor](https://forums.swift.org/t/pre-pitch-conditional-default-arguments/7122/4)があるが、これはmotivationのセクションでざっと説明されているすべての問題を抱えている。この点でデフォルトの式から型推論を許可することは、新しい構文やカスタム属性を導入することなく、すべての状況で機能するはるかにクリーンなアプローチである。


### 将来の検討事項

この提案では、すべてのデフォルトの式が個別に型チェックされるため、推論可能なジェネリックパラメータの使用をパラメータリスト内の単一の場所に制限される。この制限を解除し、すべてのデフォルト式を一緒に型チェックすることができる。つまり、ジェネリックパラメータが異なるデフォルト式から推論できる場合、その型はすべての場所に適合する共通の型になる(このような型を取得するアクションは 型結合と呼ばれる)。この制限を解除することが、ユーザにとって驚き最小の原則に常に準拠するかどうかはすぐにはわからないため、この提案が受け入れられた場合は、別途話し合う必要がある。

問題を説明する最も簡単な例は`test<T>（a:T = 42, b:T = 4.2）-> T`。この宣言は、それぞれ異なる型の可能な呼び出しのマトリックスを作成する。

1. `test()` — 42と4.2の両方に適合する唯一の型は`Double`なのでT = Doubleになる
2. `test(a: 0.0)` — T = Double
3. `test(b: 0)` — T = Int
4. `let _: Int = test()` - Tは`Int`と`Double`に同時にできないため失敗する

## 参考リンク

### Forums

- [[Pitch] Type inference from default expressions](https://forums.swift.org/t/pitch-type-inference-from-default-expressions/55585)
- [SE-0347: Type inference from default expressions](https://forums.swift.org/t/se-0347-type-inference-from-default-expressions/56142)
- [[Accepted] SE-0347: Type inference from default expressions](https://forums.swift.org/t/accepted-se-0347-type-inference-from-default-expressions/56558)

### プロポーザルドキュメント

- [Type inference from default expressions](https://github.com/apple/swift-evolution/blob/main/proposals/0347-type-inference-from-default-exprs.md)

### 関連PR

- [[TypeChecker] Allow inference from default expressions in certain scenarios (under a flag)](https://github.com/apple/swift/pull/41436)
- [[5.7][TypeChecker] SE-0347: Enable type inference from default expressions](https://github.com/apple/swift/pull/42272)
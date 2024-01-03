# Inferring Sendable for methods and key path literals

- [Inferring Sendable for methods and key path literals](#inferring-sendable-for-methods-and-key-path-literals)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
      - [Key Path](#key-path)
    - [提案内容](#提案内容)
      - [関数](#関数)
      - [Key Path](#key-path-1)
    - [詳細](#詳細)
      - [`Sendable`を維持するためのKey Pathの結合機能の拡張](#sendableを維持するためのkey-pathの結合機能の拡張)
    - [ソースの互換性](#ソースの互換性)
    - [ABIの安定性](#abiの安定性)
    - [APIレジリエンス](#apiレジリエンス)
    - [将来的な方向性](#将来的な方向性)
    - [検討された代替案](#検討された代替案)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

このプロポーザルは、値として扱われる関数とKey Pathリテラルに関するいくつかの言語の重箱の隅に焦点を当てたものある。ここでは部分適用メソッドと未適用メソッドについては、`Sendable`であることを推論することを提案する。また、開発者がKey Pathリテラルを`Sendable`かどうかを制御できるようにすることで、[SE-0302](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md#key-path-literals)のKeyPathリテラルのSendableに関する制限を解除することも提案する。目標は、Swift に大きな変更を加えることなく、柔軟性、シンプルさ、人間工学を改善することである。

## 内容

### 動機

メソッドの部分適用やその他の第一級関数[^1]の使い方は、Concurrencyと組み合わせると少々乱暴な面がある。

まず、部分適用を単独で見てみよう。Swiftでは、そのインスタンスの1つを使用してメソッドにアクセスする(が、呼び出さない)式を書いて、メソッドを表す関数値を作成できる。このアクセスは、その(カリー化)引数の1つ、つまりオブジェクトインスタンスへのメソッドの「部分適用」と呼ばれる。

[^1] 第一級関数: 関数を第一級オブジェクトとして扱うことのできるプログラミング言語の性質、またはそのような関数。ここでは関数型の話をしている。 https://ja.wikipedia.org/wiki/%E7%AC%AC%E4%B8%80%E7%B4%9A%E9%96%A2%E6%95%B0

```swift
struct S {
  func f() { ... }
}

let partial: (() -> Void) = S().f 
```

NominalType.method(公称型.メソッド)という表現を使って、メソッドをオブジェクトのインスタンスに部分適用*せず*に参照することを「未適用」と呼ぶ。

`Sendable`に準拠した未適用のメソッドをパラメータとして期待するジェネリックメソッドを作りたいとする。SE-302でクロージャと関数に導入された`@Sendable`属性を使って、クロージャのパラメータにアノテーションを付けられる。

```swift
protocol P: Sendable {
  init()
}

func g<T>(_ f: @escaping @Sendable (T) -> (() -> Void)) where T: P {
  Task {
    let instance = T()
    f(instance)()
  }
}
```

では、メソッドを呼び出し、構造体`S`を渡してみよう。まず、`S`を`Sendable`に準拠させる必要がある。これは、`S`を新しい`Sendable`型の`P`に準拠させることで実現できる。

これで`S`とそのメソッドも`Sendable`になるはずだ。しかし、未適用メソッド`S.f`をジェネリック関数`g`に渡すと、`S.f`が期待している`Sendable`ではないと、`g()`は警告を発する。

```swift

struct S: P {
  func f() { ... }
}

g(S.f) // Converting non-sendable function value to '@Sendable (S) -> (() -> Void)' may introduce data races
```

未適用メソッドを`Sendable`クロージャでラップすることで、これを回避することができる。

```swift
// S.f($0) == S.f()
g({ @Sendable in S.f($0) })
```

しかし、これでは期待通りの動作を得るために多くの手間がかかる。コンパイラは、代わりに`@Sendable`を型シグネチャに保持すべきだ。

#### Key Path

[SE-0302](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md#key-path-literals)では、すべてのKey Pathリテラルを暗黙的に`Sendable`として扱うと明示している。この動作は、Key Pathの値がConcurrencyのドメインを越えて渡されたり、同時実行されるコードに関与したりする場合には妥当だが、同時実行に関連しないコードに対しては制限が多すぎる。

```swift
class Info : Hashable {
  // ユーザに関する情報
}

public struct Entry {}

public struct User {
  public subscript(info: Info) -> Entry {
    // 与えられたInfoに対応するEntryを返す
  }
}

let entry: KeyPath<User, Entry> = \.[Info()]
```

`Sendable`チェックを有効にすると、この例では次のような警告が出る:

```swift
warning: cannot form key path that captures non-sendable type 'Info'
let entry: KeyPath<User, Entry> = \.[Info()]
```

全てのKey Pathリテラルは`Sendable`であるべきなので、Key Pathリテラルの使用は現在チェックされている。実際には、このコードはConcurrencyセーフであり、Key Pathが実際にConcurrencyの境界を越えることはないため、データ競合は発生しない。そのため、このような警告は Swift Concurrencyに不慣れな開発者を混乱させ、型が別のモジュールで宣言されたときに常に対処できるとは限らず、言語の漸進的情報開示(progressive disclosure)の原則に反する可能性がある。

### 提案内容

私たちは、`Sendable`でない値を取り込むことができないメソッドやKey Pathに対して、コンパイラが自動的に`Sendable`を導入することを提案する。これには、`Sendable`型のインスタンスメソッドの部分適用や未適用メソッド、非ローカル関数が含まれる。さらに、`Sendable`でない型のインスタンスメソッドで`@Sendable`を使用することは禁止されるべきである。

#### 関数

関数の場合、`@Sendable`属性は、主に関数が取り込める値の型に影響を与える。しかし、公称型メソッドは、オブジェクトのインスタンスそのもの以外を取り込むことはない。意味的には、メソッドは以下のような関数で表せると考えられる:

```swift
// 公称型の擬似コード宣言
type NominalType {
  func method(ArgType) -> ReturnType { /* メソッド */ }
}

// この2つのグローバル関数に脱糖する(糖衣構文から元に戻す)ことができる：:
func NominalType_method_partiallyAppliedTo(_ obj: NominalType) -> ((ArgType) -> ReturnType) {
  let inner = { [obj] (_ arg1: ArgType) -> ReturnType in
    return NominalType_method(obj, arg1)
  }
  return inner
}
// 実際のメソッド呼び出し
func NominalType_method(_ self: NominalType, _ arg1: ArgType) -> ReturnType {
  /* メソッド本文 */
}
```

したがって、部分適用メソッドが`@Sendable`になる唯一の方法は、内側のクロージャが`@Sendable`である場合であり、つまり公称型が`Sendable`に準拠する場合にのみ成立する。

```swift
type NominalType : Sendable {
  func method(ArgType) -> ReturnType { /* メソッド本文 */ }
}
```

例えば、次のように`Sendable`型を宣言すると、その型の部分メソッドと未適用メソッドの値は`Sendable`に暗黙的に準拠し、次のコードはエラーなしでコンパイルされる。

```swift
struct User : Sendable {
  func updatePassword (new: String, old: String) -> Bool {
    /* パスワード更新*/ 
    return true
  }
}

let unapplied: @Sendable (User) -> ((String, String) → Bool) = User.updatePassword // エラーなし

let partial: @Sendable (String, String) -> Bool = User().updatePassword // エラーなし
```

#### Key Path

Key Pathリテラルは関数に非常に似ており、それが`Sendable`かどうかは、引数に取り込まれる値が`Sendable`かどうかと、参照されるプロパティとsubscriptの分離状態の影響を受ける可能性がある。Key Pathリテラルが常に`Sendable`であることを要求し、`Sendable`ではない型をキャプチャする場合は警告を発するこれまでに代わり、その要求を反転させ、開発者がKey Pathが`Sendable`であることを要求された場合、明示的に`& Sendable`による型合成を使って`Sendable`であることを指定できるようにする。また、文脈上の型が指定されていない場合は、関数と同じく`Sendable`だと型推論する。(型のKey Path階層は`Sendable`ではない)。

新しいプロパティとsubscriptを使用して、先ほどの例の`User`型を拡張し、動作の変化を示す:

```swift
struct User {
  var name: String

  @MainActor var age: Int

  subscript(_ info: Info) -> Entry { ... }
}
```

プロパティ名を参照するKey Pathは、`Sendable`ではない型を捕捉しない。つまり、このようなKey Pathリテラルの型は、`WritableKeyPath<User, String> & Sendable`と推論されるか、または、`& Sendable`合成によって`Sendable`型を持つことが示される:

```swift
let name = \User.name // WritableKeyPath<User, String> **& Sendable**
let name: KeyPath<User, String> & Sendable = \.name // 🟢
```

また、`@Sendable`関数型と`&Sendable` Key Pathの使い分けもできる:

```swift
let name: @Sendable (User) -> String = \.name 🟢
```

**このプロポーザル上のルールでは、Key Pathの型とともに`Sendable`要件を明示的に指定しない宣言はすべて、`Sendable`ではないと扱われる**ことに注意することが重要である(詳細な議論については、「ソースの互換性」のセクションを参照):

```swift
let name: KeyPath<User, String> = \.name // 🟢 だけど Key Pathは**Sendableではない**
```

`Sendable`はマーカープロトコルであるため、`& Sendable`であることが期待されるすべての宣言を、ABIに影響を与えることなく調整できるはずである。

パラメータの型やデフォルト値にKey Pathを使用している既存のAPIは、既存の宣言を`@preconcurrency`としてマークし、適切な位置に`& Sendable`を追加することで、ABIを破壊しない方法で`Sendable`要件を追加できる：


```swift
public func getValue<T, U>(_: KeyPath<T, U>) { ... }
```

は

```swift
@preconcurrency public func getValue<T, U>(_: KeyPath<T, U> & Sendable) { ... }
```

になる。

明示的な`Sendable`のアノテーションは`Sendable`かどうかのチェックを上書きしないので、`Sendable`ではない値をキャプチャしているときに、Key Pathリテラルが`Sendable`であるとするのはこれまで通り正しくない:

```swift
let entry: KeyPath<User, Entry> & Sendable = \.[Info()] 🔴 Info is a non-Sendable type
```

このような`entry`宣言は、`Sendable`チェッカーのチェックに引っかかる:

```swift
warning: cannot form key path that captures non-sendable type 'Info'
```

同じように、グローバルアクターに分離されたプロパティである`age`(すなわち、`\User.age`)を参照するKey Pathは、`Sendable`ではない。

### 詳細

このプロポーザルには、`Sendable`の動作に関する5つの変更が含まれている。

最初の2つは、先ほど説明した部分適用と未適用メソッドに関するものである。

```swift
struct User : Sendable {
  var address: String
  var password: String

  func changeAddress (new: String, old: String) {/* 何かする */ }
}
```

1. `Sendable`型のメソッドへの未適用関数の参照に対する`@Sendable`の推論。

```swift
let unapplied : @Sendable (User)-> ((String, String) -> Void) = User.changeAddress // エラーなし
``

2. `Sendable`型が部分適用されるメソッドに対する`@Sendable`の推論。

```swift
let partial : @Sendable (String, String) -> Void = User().changeAddress // エラーなし
```

この2つのルールには、部分適用と未適用の静的(static)メソッドが含まれるが、部分適用と未適用の可変(mutable)メソッドは含まれない。可変メソッドに対する未適用メソッドの参照は、未定義の動作につながる可能性があるため、Swiftでは許可されていない。これについての詳細は[SE-0042](https://github.com/apple/swift-evolution/blob/main/proposals/0042-flatten-method-types.md)を参照。

3. `Sendable`ではない型のキャプチャとアクター分離プロパティやsubscriptへの参照を持たないKey Pathリテラルは、`&Sendable`要件を持つKey Path型、または`@Sendable`属性を持つ関数型として推論される。

```swift
extension User {
  @MainActor var age: Int { get { 0 } }
}

let ageKP = \User.age
let infoKP = \User.[Info()]
```

`ageKP`の型は、`age`がグローバルアクターに分離されているため、`KeyPath<User, Int>`。同様に、`infoKP`は、subscriptが参照する`Info()`引数が`Sendable`ではない型のため、`Sendable`ではないKey Pathである。

これは、`Sendable`としてマークされていないKey Pathを`Sendable`である値に割り当てることはできないことを意味する。

```swift
let name: KeyPath<User, String> = \.name
let otherName: KeyPath<User, String> & Sendable = \.name 🔴
```

Key Pathと`@Sendable`関数の間の変換は、実際にはKey Path自体が`Sendable`である必要はない。

```swift
let name: @Sendable (User) -> String = \.name 🟢
```

上記の例は認められ、コンパイラによって下記のように変換される:

```swift
let name: @Sendable (User) -> String = { $0[keyPath: \.name] }
```

しかし、`Sendable`ではないsubscriptの引数は、クロージャを`Sendable`にさせない暗黙的に合成されたクロージャによってキャプチャされる可能性があるため、変換できない:

```swift
let value: NonSendable = NonSendable()
let _: @Sendable (User) -> String = \.[value] 🔴
```

これは、`value`が`Sendable`ではなく、Key Pathをラップするコンパイラが合成するクロージャ(`{ $0[keyPath： \value]] }`)が、(`value`をキャプチャしているため)`Sendable`ではないと推論され、`@Sendable`関数型に変換できまない。

同様に、変換がアクター分離のプロパティまたはsubscriptへの参照を持つKey Pathをキャプチャする場合、暗黙的に生成されるクロージャは`Sendable`ではないと推論されない。

Key Pathリテラルが`Sendable`型を要求するパラメータの引数として渡される場合など、Key Pathリテラルは文脈から`Sendable`であると推論できる:

```swift
func getValue<T: Sendable>(_: KeyPath<User, T> & Sendable) -> T {}

getValue(name) // 🟢 パラメータも引数も`Sendable`である
getValue(\.name) // 🟢 パラメータに'& Sendable'を使ってKey Pathリテラルを変換する
getValue(\.[NonSendable()]) // 🔴 Key Pathが`Sendable`ではない型をキャプチャしているため不正

func filter<T: Sendable>(_: @Sendable (User) -> T) {}
filter(name) // 🟢 `@Sendable`を使って`Sendable`のKey Pathを適用している
```

次は

4. 非ローカル関数を参照する際の`@Sendable`の推論

キャプチャした値を保持するクロージャとは異なり、グローバル関数は変数をキャプチャできない。これは所有権(ownership)を持たずに関数から参照されるだけだからである。この点を考慮すれば、グローバル関数をデフォルトで`Sendable`にしない理由はない。この変更は静的グローバル関数も含まる。

```swift
func doWork() -> Int {
  Int.random(in: 1..<42)
}

Task<Int, Never>.detached(priority: nil, operation: doWork) // Converting non-sendable function value to '@Sendable () async -> Void' may introduce data races(現在出るwarning)
```

現在、グローバル関数`doWork`でタスクを開始しようとすると、関数が`Sendable`ではないというエラーが発生するが、これは問題なくコンパイルできるべき。

5. メソッドが属する型が`@Sendable`でない場合に`@Sendable`とすることを禁止する

```swift
class C {
    var random: Int = 0 // randomは可変のため`C`は`Sendable`ではない

    @Sendable func generateN() async -> Int { //error: adding @Sendable to function of non-Senable type prohibited
         random = Int.random(in: 1..<100)
         return random
    }
}

func test(x: C) { x.generateN() }

let num = C()
Task.detached {
  test(num)
}
test(num) // データ競合
```

生成した乱数を可変な値として格納するクラスに、私たちがやりたかった先ほどの作業を移した場合、この作業を担当する関数を`@Sendable`にすることで、データ競合を引き起こす可能性がある。これはコンパイラによって禁止されるべきだ。

このプロポーザルにより、`@Sendable`属性は自動的に決定されるようになるため、関数やメソッドの宣言時に`@Sendable`属性を明示的に記述する必要がなくなる。

#### `Sendable`を維持するためのKey Pathの結合機能の拡張

既存のKey PathAPIは、インスタンスメソッド`appending(...)`を使って2つのKey Pathを結合する方法を提供している。このメソッドのオーバーロードは、さまざまな可変可能性(mutability)を持つKey Path型をパラメータとして取り、期待する可変可能性(読み取り専用、書き込み可能、参照書き込み可能)の新しい「結合」Key Pathを生成する。

ここで提案されたセマンティクスの下では、このメソッドのすべてのオーバーロードは `Sendable`でなくなるが、「ベース」のKey Pathと「追加」されたKey Pathの両方が`Sendable`の場合、それを緩和し、`Sendable`をサポート/継承できるし、それが望ましい。

このようなことは、パラメータに`& Sendable`を利用する`func appending(...)`に新しいオーバーロードを導入して、`Sendable`プロトコルを拡張することで実現できる。例えば:

```swift
extension Sendable where Self: AnyKeyPath {
  @inlinable
  public func appending<Root, Value, AppendedValue>(
        path: KeyPath<Value, AppendedValue> & Sendable
  ) -> KeyPath<Root, AppendedValue> & Sendable where Self : KeyPath<Root, Value> {
    ...
  }
}
```

このオーバーロードは、「ベース」と「追加」のKey Pathが`Sendable`である場合に選択され、新しい`Sendable`に準拠したKey Pathを生成する:

```swift
func makeUTF8CountKeyPath<Root>(from base: KeyPath<Root, String> & Sendable) -> KeyPath<Root, Int> & Sendable {
  // `base`と`\String.utf8.count`は`Sendable`なKey Pathなので、
  // `appending(path:)`も`Sendable`なKey Pathを返す
  return base.appending(path: \.utf8.count) 🟢
}
```

標準ライブラリは、`Sendable`な`appending(...)`を既存の`Sendable`ではない関数との均等に保つために、さまざまな新しいオーバーロードを導入しなければならないだろう。

### ソースの互換性

「提案内容」のセクションで述べたように、明示的な型を持たない既存のプロパティ宣言や変数宣言のいくつかは、その型を変更できるが、推論が変更することによる影響は非常に限定的なものになるはずである。例えば、`Sendable`と推論される関数やKey Pathの値が、`Sendable`を許容するオーバーロードされたAPIに渡された場合にのみ、その影響が見られる:

```swift
func callback(_: @Sendable () -> Void) {}
func callback(_: () -> Void) {}

callback(MyType.f) // `f`が`@Sendable`と推論されると最初の`callback`が選ばれる

func getValue(_: KeyPath<String, Int> & Sendable) {}
func getValue(_: KeyPath<String, Int>) {}

getValue(\.utf8.count) // Key Pathが`& Sendable`ならば、最初の`getValue`が選ばれる
```

このような`callback`と`getValue`の呼び出しは現在あいまいな状態だが、ここで提案されたルールの下では、`f`が`@Sendable`と推論される場合、型チェッカーは`callback`と`getValue`の最初のオーバーロードを解決策として選ぶ。また、`\String.utf8.count`は、単なる`KeyPath<String, Int>`ではなく、`KeyPath<String, Int> & Sendable`と推論される。

### ABIの安定性

メソッドから明示的な`@Sendable`を削除すると、そのメソッドの名前修飾(mangling)[^1]が変更される。`Sendable`が推論されるようになるため、明示的なアノテーションを削除して推論を「採用」する場合は、名前修飾の変更を考慮する必要があるかもしれない。

これは`Sendable`はマーカープロトコルで、透過的に追加することができるためである。

[^1] 名前修飾: サブルーチン(関数)名などに対する内部名を、その表層的な名前のみならず、関数であればその引数の型や返戻値の型などといった意味的な情報を含めて修飾した（manglingした）名前とするもの。内部で名前が被らないように内部的な名前をつけること。
https://ja.wikipedia.org/wiki/%E5%90%8D%E5%89%8D%E4%BF%AE%E9%A3%BE

### APIレジリエンス

影響なし

### 将来的な方向性

この提案では、アクセサは`@Sendable`システムに参加させていない。もし要求があれば、将来の提案でgetterも参加させることは簡単だろう。

### 検討された代替案

Swiftは、`@Sendable`属性で関数宣言を明示的にマークすることを禁止できる。

```swift
/*@Sendable*/ func alwaysSendable() {}
```

しかし、これらの属性は現在許可されているので、これはソースの破壊的変更になるだろう。Swift 6 は、移行を容易にするために `@Sendable`` 属性を削除するfixitを含む可能性があるが、それでもまだ破壊的である。この属性はこの提案の下では無害であり、古いツールでコンパイルする必要があるコードでも時々役に立つ。もし変更する妥当な理由が見つかれば、後で非推奨にすることを検討できる。

## 参考リンク

### Forums

- [pitch](https://forums.swift.org/t/pitch-inferring-sendable-for-methods/66565)
- [review](https://forums.swift.org/t/se-0418-inferring-sendable-for-methods-and-key-path-literals/68999)
- [accepted](https://forums.swift.org/t/accepted-se-0418-inferring-sendable-for-methods-and-key-path-literals/69242)

### プロポーザルドキュメント

- [Inferring Sendable for methods and key path literals](https://github.com/apple/swift-evolution/blob/main/proposals/0418-inferring-sendable-for-methods.md)
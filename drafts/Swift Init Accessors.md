# Init アクセサ

- [Init アクセサ](#init-アクセサ)
  - [概要](#概要)
  - [動機](#動機)
  - [内容](#内容)
  - [詳細](#詳細)
    - [`self`のプロパティを確実に初期化する](#selfのプロパティを確実に初期化する)
    - [メンバーワイズイニシャライザ](#メンバーワイズイニシャライザ)
    - [計算プロパティ上の`init`アクセサ](#計算プロパティ上のinitアクセサ)
    - [readonlyプロパティに対する`init`アクセサ](#readonlyプロパティに対するinitアクセサ)
    - [`init`アクセサとプロパティの初期値](#initアクセサとプロパティの初期値)
  - [ソース互換性](#ソース互換性)
  - [API互換性](#api互換性)
  - [導入の影響](#導入の影響)
  - [検討された代替案](#検討された代替案)
    - ["initializes"と"accesses"のシンタックス](#initializesとaccessesのシンタックス)
  - [将来の方向](#将来の方向)
    - [ローカル変数の`init`アクセサ](#ローカル変数のinitアクセサ)
    - [storageRestrictionsの他の機能への一般化](#storagerestrictionsの他の機能への一般化)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

Initアクセサは、プロパティラッパ(Property Wrapper)のアウトオブライン初期化機能を一般化して、型の計算プロパティ(computed property)が確定的な初期化分析(initialization analysis)に組み込まれるようにし、カスタムの初期化コードで格納プロパティ(stored property)のセットの初期化も併せて行う。

## 動機

Swift は、格納プロパティ、格納ローカル変数、およびプロパティラッパを持つ変数に[確定的な初期化分析](https://en.wikipedia.org/wiki/Definite_assignment_analysis)を適用する。確定初期化は、メモリがアクセスされる前に、すべてのパスでメモリが初期化されることを保証する。Swift のコードで一般的なパターンは、1 つ以上の計算プロパティのバッキングストレージ(backing storage)として 1 つのプロパティを使用することであり、プロパティラッパと付属型マクロ(attached macros)のような抽象化は、このパターンを容易する。このパターンの下では、バッキングストレージは実装の詳細であり、ほとんどのコードは、イニシャライザ(initializer)を含む計算プロパティで動作する。

プロパティラッパは、計算プロパティを経由して、バッキングプロパティのラッパストレージを初期化できる、特別な確定初期化をサポートしており、常に、`self.value = value`の形で、`_value = Wrapper(wrappedValue: value)`の形で、「ラッパプロパティ経由の初期化」を書き直して、バッキングストレージを初期化する:

```swift
@propertyWrapper
struct Wrapper<T> {
  var wrappedValue: T
}

struct S {
  @Wrapper var value: Int

  init(value: Int) {
    self.value = value  // self._value = Wrapper(wrappedValue: value)に書き直される
  }

  init(other: Int) {
    self._value = Wrapper(wrappedValue: other) // バッキングストレージの'_value'を直接初期化しているのでOK
  }
}
```

プロパティラッパの初期化のアドホックな性質と(すべての格納プロパティが初期化される)確定的な初期化パターンが混在しているため、追加の引数を持つプロパティラッパを、別で初期化することができない。さらに、「プロパティラッパのようなマクロ」は、計算プロパティによる初期化をサポートする代わりに、追加されたバッキングストレージ変数を直接初期化しなければならないため、同じイニシャライザの使い勝手を実現できない。例えば、[`@Observable`マクロ](https://github.com/apple/swift-evolution/blob/main/proposals/0395-observability.md)は、プロパティラッパのような変換を適用して、保存プロパティを監視APIによって裏付けされた計算プロパティに変えますが、プログラマが期待するような元のプロパティ名を使ったイニシャライザを書く方法は提供していない:

```swift
@Observable
struct Proposal {
  var title: String
  var text: String

  init(title: String, text: String) {
    self.title = title // error: 全ての格納プロパティが初期化される前に'self'が使われている
    self.text = text // error: 全ての格納プロパティが初期化される前に'self'が使われている
  } // error: 全ての格納プロパティが初期化されずに戻ろうとしている
}
```

## 内容

本提案では、型で定義されている計算プロパティを、指定された0個以上の保存プロパティの初期化を含めた確定的な初期化プロセスに入れるための*`init`アクセサ*を追加し、型のイニシャライザの本文で計算プロパティへの割り当てを可能する:

```swift
struct Angle {
  var degrees: Double
  var radians: Double {
    @storageRestrictions(initializes: degrees)
    init(initialValue) {
      degrees = initialValue * 180 / .pi
    }

    get { degrees * .pi / 180 }
    set { degrees = newValue * 180 / .pi }
  }

  init(degrees: Double) {
    self.degrees = degrees // 'self.degrees'を直接初期化
  }

  init(radiansParam: Double) {
    self.radians = radiansParam // 'radiansParam'を引数として渡して'self.radians'のために`init`アクセサを呼ぶ
  }
}
```

`init`アクセサのシグネチャは、最大2つのタイプの格納プロパティを指定する。これは、そのアクセサによって、(`accesses`経由で)アクセスされるプロパティと(`initializes`経由で)初期化されるプロパティである。`initializes`と`accesses`は`init`アクセサの副作用である。`accesses`効果(effect)は、`init`アクセサ内からアクセスできる他の格納プロパティを指定する(`self`の他の使用は許可されていない)ので、計算プロパティの`init`アクセサが呼び出される前に初期化する必要がある。`init`アクセサは、すべての制御フローパス上で初期化された格納プロパティのそれぞれを初期化する必要がある。上の例の`radians`プロパティは、`accesses`効果は指定しないが、`degree`プロパティを初期化するので、`initializes: degree`のみを指定している。
`accesses`効果はその中身を他の格納プロパティの中に置くことで初期化できる:

```swift
struct ProposalViaDictionary {
  private var dictionary: [String: String]

  var title: String {
    @storageRestrictions(accesses: dictionary)
    init(newValue) {
      dictionary["title"] = newValue
    }

    get { dictionary["title"]! }
    set { dictionary["title"] = newValue }
  }

   var text: String {
    @storageRestrictions(accesses: dictionary)
    init(newValue) {
      dictionary["text"] = newValue
    }

    get { dictionary["text"]! }
    set { dictionary["text"] = newValue }
  }

  init(title: String, text: String) {
    self.dictionary = [:] // 'dictionary'は`init`アクセサからアクセスされる前に初期化されていなければならない
    self.title = title // `title`を'dictionary'に入れるために`init`アクセサを呼ぶ
    self.text = text   // `text`を'dictionary'に入れるために`init`アクセサを呼ぶ
    // いずれかの初期化を省略するとエラーになる
  }
}
```

どちらの`init`アクセサも、`dictionary`にアクセスすることを明文化しており、初期化の一環として、適切なキーを持つ`dictionary`に新しい値を挿入することができる。これにより、型に使用されている保存メカニズムを完全に抽象化することができる。

最後に、`init`アクセサを持つ計算プロパティは、コンパイラによって合成されたメンバワイズイニシャライザで特権を得ます。この提案では、プロパティラッパは、特注の確定的なメンバーワイズイニシャライザをサポートしない。その代わりに、`init(wrapValue:)`を持つプロパティラッパのための糖衣構文は、ラップされたプロパティのための`init`アクセサと、それぞれのバックストレージの代わりに、ラップされた値を含むメンバワイズイニシャライザを含む。「動機」セクションのプロパティラッパのコードは、以下のコードな糖衣構文になる:

```swift
@propertyWrapper
struct Wrapper<T> {
  var wrappedValue: T
}

struct S {
  private var _value: Wrapper<Int>
  var value: Int {
    @storageRestrictions(initializes: _value)
    init(newValue) {
      self._value = Wrapper(wrappedValue: newValue)
    }

    get { _value.wrappedValue }
    set { _value.wrappedValue = newValue }
  }

  // このイニシャライザはコンパイラが生成するメンバワイズイニシャライザと同じ
  init(value: Int) {
    self.value = value  // 'self.value'上の'init`アクセサを呼ぶ
  }
}

S(value: 10)
```

この提案は、マクロが、計算されたプロパティのアウトオブライン初期化を含む、以下のプロパティラッパのようなパターンのモデル化を可能にする:

- 属性引数を持つラップされたプロパティ
- 明示的な格納プロパティにバッキングデータを持つラップされたプロパティ
- 1つの格納プロパティのバッキングデータを持つラップされたプロパティのセット


## 詳細

本提案では、計算プロパティのアクセサリストに記述できる`init`アクセサという新しい種類のアクセサを追加する。`init`アクセサは、文法に以下の生成ルールを追加する:

```
init-accessor -> 'init' init-accessor-parameter
init-accessor -> 'init' init-accessor-parameter[opt] function-body

init-accessor-parameter -> '(' identifier ')'

accessor-block -> init-accessor
```

指定された場合、`init-accessor-parameter`の`identifier`は、初期値を含むパラメータの名前である。提供されない場合、`newValue`という名前のパラメータが自動的に作成される。最小限の`init`アクセサは、パラメータリストを持たず、初期化の効果もない:

```swift
struct Minimal {
  var value: Int {
    init {
      print("init accessor called with \(newValue)")
    }

    get { 0 }
  }
}
```

また、この提案では `init` アクセサブロックのストレージ制限を記述するための新しい `storageRestrictions` 属性を追加しする。この属性は `init` アクセサにのみ使用することができる。 この属性は文法中の以下の生成規則で記述される:

```
attribute ::= storage-restrictions-attribute

storage-restrictions-attribute ::= '@' storageRestrictions '(' storage-restrictions[opt] ')'

storage-restrictions-initializes ::= 'initializes' ':' identifier-list
storage-restrictions-accesses ::= 'accesses' ':' identifier-list

storage-restrictions ::= storage-restrictions-accesses
storage-restrictions ::= storage-restrictions-initializes
storage-restrictions ::= storage-restrictions-initializes ',' storage-restrictions-accesses
```

ストレージ制限属性は、このアクセサによって初期化される格納プロパティのリスト(`storage-restrictions-initializes`の識別子リスト)と、このアクセサによってアクセスされる格納プロパティのリスト(storage-restrictions-accessesの識別子リスト)を含むことができ、それぞれはオプションである:

```swift
struct S {
  var readMe: String

  var _x: Int

  var x: Int {
    @storageRestrictions(initializes: _x, accesses: readMe)
    init(newValue) {
      print(readMe)
      _x = newValue
    }

    get { _x }
    set { _x = newValue }
  }
}
```

アクセサがデフォルトのパラメータ名`newValue`を使用し、格納プロパティを初期化もアクセスもしない場合、このシンタックスは必要ない。

`init`アクセサは、格納プロパティのセットの初期化を含めることができる。入れ込まれた格納プロパティは、その属性の`initializes`パラメータによって指定される。`init`アクセサの本文は、すべての制御フローパスで入れ込まれた格納プロパティを初期化することが要求される。

`init`アクセサは、本文が評価されるときにすでに初期化されている格納プロパティのセットを要求することもでき、これはその属性の`accesses`パラメータを通じて指定される。これらの格納プロパティは、アクセサ本文の中でアクセスすることができる。逆に言うと、`self`の他のプロパティやメソッドは、アクセサ本文の中では利用できず、`self`はオブジェクト全体として利用できない(たとえば、`self`上のメソッドを呼び出すことができない)。


### `self`のプロパティを確実に初期化する

型のイニシャライザ内の代入のセマンティクスは、代入する時点で`self`内のすべてが、すべてのコードパスで初期化されているかどうかに依る。`self`の全てが初期化される前に、`init`アクセサを持つ計算プロパティへ値を代入しようとすると`init`アクセサの呼び出しに書き換えられ、`self`が初期化された後だと、計算プロパティへの代入はsetter呼び出しに書き換えられる。

この提案では、以下の場合、`self`のすべてが初期化される:

- すべての保存されたプロパティがすべてのパスで初期化され、かつ
- `init`アクセサを持つすべての計算プロパティが、すべてのパスで初期化される

`self`の全てが初期化される前に`init`アクセサを持つ計算されたプロパティへ値を代入する場合、計算プロパティの`init`アクセサを呼び出し、その`initializes`句で指定されたすべての格納プロパティを初期化する:

```swift
struct S {
  var x1: Int
  var x2: Int
  var computed: Int {
    @storageRestrictions(initializes: x1, x2)
    init(newValue) { ... }
  }

  init() {
    self.computed = 1 // 'computed'、'x1'と'x2'を初期化する。つまり、'self'は今完全に初期化される
  }
}
```

すべてのパスで初期化されていない計算プロパティへの代入は、`init`アクセサの呼び出しに書き直される:

```swift
struct S {
  var x: Int
  var y: Int
  var point: (Int, Int) {
    @storageRestrictions(initializes: x, y)
    init(newValue) {
	    (self.x, self.y) = newValue
    }
    get { (x, y) }
    set { (x, y) = newValue }
  }

  init(x: Int, y: Int) {
    if (x == y) {
      self.point = (x, x) // 'init'アクセサを呼ぶ
    }

    // 'self.point'はすべてのコードパスで初期化されていない

    self.point = (x, y) // 'init'アクセサを呼ぶ

    // 'self.point'はすべてのコードパスで初期化されている
  }
}
```

`self`のすべてが初期化される前に格納プロパティに代入すると、その格納プロパティは初期化される。`init`アクセサを持つ計算プロパティの`initializes`句に列挙されたすべての格納プロパティが初期化されたとき、その計算プロパティは初期化されたとみなされる:

```swift
struct S {
  var x1: Int
  var x2: Int
  var x3: Int
  @storageRestrictions(initializes: x1, x2)
  var computed: Int {
    init(newValue) { ... }
  }

  init() {
    self.x1 = 1 // 'x1'を初期化する。'x2'と'computed'は初期化されていない。
    self.x2 = 1 // 'x2'を初期化する。'computed'は初期化されていない。
    self.x3 = 1 // 'x3'を初期化する。'self'は完全に初期化された。
  }
}
```

`initializes`にリストされた格納プロパティの少なくとも1つが初期化されているけど`self`が初期化されていないポイントで、計算プロパティへ代入しようとするとエラーとなる。これにより、基となる格納プロパティの二重初期化を防ぐことができる:

```swift
struct S {
  var x: Int
  var y: Int
  @storageRestrictions(initializes: x, y)
  var point: (Int, Int) {
    init(newValue) {
	    (self.x, self.y) = newValue
    }
    get { (x, y) }
    set { (x, y) = newValue }
  }

  init(x: Int, y: Int) {
    self.x = x // 'x'が初期化されている
    self.point = (x, y) // error: `init`アクセサもsetterもここで呼ぶことはできない
  }
}
```

### メンバーワイズイニシャライザ

構造体が独自のイニシャライザを宣言していない場合、構造体の格納されたプロパティに基づいて暗黙のメンバワイズイニシャライザを受け取る(格納プロパティは初期化される必要があるため)。`init`アクセサの多くのユースケースは、1つの計算プロパティを完全に抽象化して、1つの格納プロパティをバッキングストレージとして使用するものである(プロパティラッパのユースケースなど)。プログラマは主に計算プロパティを通じてストレージとやり取りするため、`init`アクセサはストレージの初期化に適したメカニズムを提供する。そのため、メンバワイズイニシャライザのパラメータリストには、initアクセサを持つ計算プロパティと、initアクセサに包含されていない格納プロパティのみが含まれる。

```swift
struct S {
  var _x: Int
  var x: Int {
    @storageRestrictions(initializes: _x)
    init(newValue) {
      _x = newValue
    }

    get { _x }
    set { _x = newValue }
  }

  var y: Int  
}

S(x: 10, y: 100)
```

上記のstruct`S`はコンパイラが合成したイニシャライザを受け取る:

```swift
init(x: Int, y: Int) {
  self.x = x
  self.y = y
}
```

メンバワイズイニシャライザのパラメータは、ソースの順序に従う。しかし、initアクセサが、メンバワイズイニシャライザで先行する格納プロパティに `アクセス` する場合、メンバワイズイニシャライザでパラメータが発生するのと同じ順序で プロパティを初期化することはできない。例えば:


```swift
struct S {
  var _x: Int
  var x: Int {
    @storageRestrictions(initializes: _x, accesses: y)
    init(newValue) {
      _x = newValue
    }

    get { _x }
    set { _x = newValue }
  }

  var y: Int 
}
```

上記のstructのメンバワイズイニシャライザで、プロパティをパラメータと同じ順序で初期化するように記述すると、エラーが発生する：

```swift
init(x: Int, y: Int) {
  self.x = x // エラー
  self.y = y
}
```

したがって、コンパイラは、合成されたメンバワイズイニシャライザにおいて、アクセス句を尊重するように初期化の順序を決める:

```swift
init(x: Int, y: Int) {
  self.y = y
  self.x = x
}
```

この提案の最初のレビューでは、順序外の初期化が驚きを引き起こすという懸念に基づいて、このような場合のメンバワイズイニシャライザを抑制した。しかし、フィールドは独立して初期化され(あるいは、相対的な順序を定義するアクセス関係を持ち)、ここでの副作用は`init`アクセサ自体の副作用に限定されるという事実を考慮すると、任意の差異を観察するためには、初期化中にグローバルな副作用を導入しなければならない。

メンバワイズイニシャライザ子を合成できない場合もある。例えば、ある型に、同じ格納プロパティを初期化する`init`アクセサを持つ複数の計算プロパティが含まれている場合、メンバワイズイニシャライザ子内でどの計算プロパティを使用すべきかは明らかではない。このような場合、メンバワイズイニシャライザは合成されない。

### 計算プロパティ上の`init`アクセサ

`init`アクセサは、計算プロパティで提供することができ、その場合は初期化に使用され、メンバワイズイニシャライザのデフォルト引数として使用される。例えば、以下のように指定する:

```swift
struct Angle {
  var degrees: Double
  
  var radians: Double {
    @storageRestrictions(initializes: degrees)
    init(initialValue) {
      degrees = initialValue * 180 / .pi
    }

    get { degrees * .pi / 180 }
    set { degrees = newValue * 180 / .pi }
  }
}
```

### readonlyプロパティに対する`init`アクセサ

`init`アクセサは、setterを持たないプロパティに提供することができる。このようなプロパティは`let`プロパティと同じように動作し、一度だけ(正確に)初期化され、その後は設定されない:

```swift
struct S {
  var _x: Int

  @storageRestrictions(initializes: _x)
  var x: Int {
    init(initialValue) {
      self._x = x
    }

    get { _x }
  }

  init(halfOf y: Int) {
    self.x = y / 2 // initアクセサはxに対して呼んでいるためOK 
    self.x = y / 2 // xはsetできないためエラー
  }
}
```

### `init`アクセサとプロパティの初期値

`init`アクセサを持つプロパティは初期値を持つことができる。例えば:

```swift
struct WithInitialValues {
  var _x: Int

  var x: Int = 0 {
    @storageRestrictions(initializes: _x)
    init(initialValue) {
      _x = initialValue
    }

    get { ... }
    set { ... }
  }

  var y: Int
}
```

合成されるメンバワイズイニシャライザは、その初期値をデフォルト引数として使用する。次のようになる:

```swift
init(x: Int = 0, y: Int) {
  self.x = x  // _xを初期化するinitアクセサを呼ぶ
  self.y = y
}
```

手動で書かれたイニシャライザでは、初期値は、ユーザが書いたコードの前に、`init`アクセサでプロパティを初期化するために使用される:

```swift
init() {
  // 暗黙的にself.x = 0で初期化されている
  self.y = 10
  self.x = 20 // setterを呼ぶ
}
```

## ソース互換性

`init`アクセサは、新しい構文による追加機能であり、既存のソースコードに影響を与えることはない。

## API互換性

`init`アクセサはモジュール内部からしか呼び出されないので、モジュールのABIの一部ではない。型のイニシャライザが`@inlinable`である場合、`init`アクセサの本体も`inlinable`でなければならない。

## 導入の影響

`init`アクセサは常に定義モジュール内から呼び出されるため、`init`アクセサを導入することはABIで互換を保ったままの変更となる。既存のプロパティに`init`アクセサを追加しても、定義モジュールの外ではソース互換性に影響を与えることはない。ソース上で互換性が保たれない可能性があるのは、合成されるメンバワイズイニシャライザ(新しいエントリが追加された場合)、または型の`init`実装(新しい初期化効果が追加された場合)だけである。

## 検討された代替案

### "initializes"と"accesses"のシンタックス

TBD

## 将来の方向

### ローカル変数の`init`アクセサ

ローカル変数の`init`アクセサは、`init`や`set`への代入の書き換えが`self`の初期化状態に基づいていないため、確定的な初期化に対する意味合いが異なる。ローカル変数のgetterとsetterは、スコープ内の他の任意のローカル変数をキャプチャすることができるため、プロパティへの代入が`init`や`set`に書き換えられる可能性がある同じコードパスの中で、初期化前にエスケープした使用があるかを診断するためにより多くの課題が出てくる。そのため、`init`アクセサを持つローカル変数は、将来導入される可能性のあるものとしている。

### storageRestrictionsの他の機能への一般化

将来的には、`storageRestrictions`属性を一般化して、他の関数にも適用できるようになるかもしれない。例えば、クラス内で共通の初期化関数を実装することができるようになる:

```swift
class C {
  var id: String
  var state: State

  @storageRestrictions(initializes: state, accesses: id)
  func initState() {
    self.state = /* 初期化のためのコードをここに */
  }

  init(id: String) {
    self.id = id
    initState() // idにアクセスしてstateを初期化するのはOK
  }
}
```

原則は`init`アクセサと同じ。関数の実装は、特定の格納プロパティにのみアクセスし、他のプロパティはすべてのパスで初期化するように制限することができます。関数の実装は、特定の格納プロパティにのみアクセスするように制限することができ、すべてのパスに沿って他のプロパティを初期化することができる。

この一般化には、`init`アクセサには関係のない制限が伴う。例えば、`initState`関数は、`state`が初期化された後に呼び出すことはできず()`state`ステートを再初期化してしまうから)、「第一級」関数として使用することもできない:

```swift
  init(id: String) {
    self.id = id
    initState() // idにアクセスしてstateを初期化するのはOK

    initState() // `state`は既に初期化されているためエラー
    let fn = self.initState // 関数値のように扱うことはできない
  }
```


## 参考リンク

### Forums

- [[Pitch] `init` accessors](https://forums.swift.org/t/pitch-init-accessors/64881)
- [SE-0400: Init Accessors](https://forums.swift.org/t/pitch-init-accessors/65583)

### プロポーザルドキュメント

- [Init Accessors](https://github.com/apple/swift-evolution/blob/main/proposals/0400-init-accessors.あ０
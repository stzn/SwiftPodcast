# Swift structは不変(immutable)?

- [Swift structは不変(immutable)?](#swift-structは不変immutable)
  - [概要](#概要)
  - [内容](#内容)
    - [varに割り当てられた場合は変更できるので、不変ではないのでは？](#varに割り当てられた場合は変更できるので不変ではないのでは)
    - [COW(copy-on-write)とは？](#cowcopy-on-writeとは)
      - [仕組み](#仕組み)
      - [独自で定義した型にCOWを活用する方法](#独自で定義した型にcowを活用する方法)
      - [プロパティのdidSet](#プロパティのdidset)
    - [Modify Accessors](#modify-accessors)
      - [背景](#背景)
      - [改善](#改善)
    - [borrow変数](#borrow変数)
      - [背景](#背景-1)
      - [改善](#改善-1)
    - [WWDC(Building Better Apps with Value Types in Swift)](#wwdcbuilding-better-apps-with-value-types-in-swift)
    - [Law of Exclusivityとは？](#law-of-exclusivityとは)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
## 概要

## 内容

Swiftのstructは不変(immutable)と呼ばれているが、実際はvarに割り当てられるとそのメンバを変更できるように見える。では、この不変とはどういう意味なのか？

### varに割り当てられた場合は変更できるので、不変ではないのでは？

Swiftのstructの場合、**セマンティクス上**、値の変更時にコピーが発生して元の値へ影響を与えない。これはobservable effect(※1)という観点での話。structの値はその自身の「スコープを超えた範囲(他の変数への代入時、初期化や関数の引数へ渡す場合など)」ではコピーが発生し、元の値が変更されることはないので不変と言われている。これはstructだけではなく、Swiftの値型(enumなど)に当てはまる。

例えば、下記では、`foo.bar.x`のタイミングで"Got a new bar!"が出力されていて、新しい`Bar`が割り当てられていることがわかる。

```swift
struct Bar {
    var x: Int
    var y: Int
}

struct Foo {
    var bar: Bar {
        didSet { print("Got a new bar!") }
    }
}

var foo = Foo(bar: Bar(x: 1, y: 1))
foo.bar.x = 10
```

※1  
observable effectとは、一般的に「状態の変更」を指す。状態の変更とは例えば下記のようなもの:

- 関数のスコープを超えた変数の変更(グローバルやstatic変数、関数の引数など)
- ファイルやネットワークの読み書きなどのI/O処理
- 画面の表示の更新
　　　　
- [observable effectとは？](http://introtorx.com/Content/v1.0.10621.0/09_SideEffects.html)

ただし、実際はコンパイラが最適化していて、必ず毎回コピーしているわけではない。ただし、これは内部の詳細になるので、必ずこうなると推定するべきものではない。将来的に実装が変わる可能性もある。

https://stackoverflow.com/a/43493749
https://gist.github.com/hamishknight/edbcc3b9cc92158a35488ce28108fe9f#optimising-value-type-parameters-to-pass-by-reference


一方で、明示的に余計なコピーを防ぐ仕組みも存在している。

### COW(copy-on-write)とは？

#### 仕組み

大きな値型をメモリに割り当てたり、引数として渡す場合、メモリ内に紐づくデータの全てをコピーしなければならないため、コピーがパフォーマンスに影響を与える可能性がある。  
この問題を最小限にするために、その値への参照がユニークな場合、変更が発生したとしてもコピーは発生しない。参照がユニークなので、コピーする必要がなく、ただその参照上で変更させること(インプレース変更)ができる。これをCOW(copy-on-write)と呼ぶ。Standard Libraryの`Array`や`Dictionary`などにはCOWが使われている(全ての型に使われているわけではない)。

[Copy-on-Write Representation](https://github.com/apple/swift/blob/main/docs/SIL.rst#copy-on-write-representation)

[Understanding Swift Copy-on-Write mechanisms](https://medium.com/@lucianoalmeida1/understanding-swift-copy-on-write-mechanisms-52ac31d68f2f)

#### 独自で定義した型にCOWを活用する方法

ユニークかどうかを明示的にチェックする方法として[`isKnownUniquelyReferenced(_:)`](https://developer.apple.com/documentation/swift/2429905-isknownuniquelyreferenced)メソッドがある。

これを活用して自身で定義した値型にもCOWを適用することができる。

```swift
final class Ref<T> {
  var val: T
  init(_ v: T) {val = v}
}

struct Box<T> {
    var ref: Ref<T>
    init(_ x: T) { ref = Ref(x) }

    var value: T {
        get { return ref.val }
        set {
          if !isKnownUniquelyReferenced(&ref) {
            ref = Ref(newValue)
            return
          }
          ref.val = newValue
        }
    }
}
```

[Advice: Use copy-on-write semantics for large values](https://github.com/apple/swift/blob/main/docs/OptimizationTips.rst#advice-use-copy-on-write-semantics-for-large-values)

`ref`への参照がユニークである場合はインプレース変更が行なわれ、そうでない場合は新しいインスタンスを設定している。  
上記のリンクに記載のあるように、これは大きい値型を使用している際に、変数への割り当てや関数の引数への受け渡し時にコピーが発生すると、コピーに時間がかかってパフォーマンスに影響を与えるようなケースで利用される。

#### プロパティのdidSet

他にも`willSet`がなく`didSet`のみかつ`oldValue`を参照していない場合はインプレースの変更ができる。
[SE-0268: Refine didSet Semantics](https://forums.swift.org/t/se-0268-refine-didset-semantics/30049)

※ これは`struct`に限った話ではない

また内部的に実装されており、今後公開予定の機能もある。

### Modify Accessors

上記で記載したような瞬間的ではない可変アクセスを実装するための`modify`アクセサも現在提案されている。これを使うとコルーチン(※)でインプレース処理ができるようになる。

※ コルーチン
関数を任意の箇所でストップ、再開できる機能。サブルーチンがエントリーからリターンまでを一つの処理単位とするのに対し、コルーチンはいったん処理を中断した後、続きから処理を再開できる。

#### 背景

`get`/`set`の仕組みは関数の呼び出しと同じ。`get`から戻り値を返す時にコピーが起きる。`set`は値の所有権が譲渡される。

そうするといくつかのオーバーヘッドが生じる。

```swift
struct Foo {
  private var _x: [Int]

  var x: [Int] {
      get { _x }
      set { _x = newValue }
  }
}

foo.x.append(1738)
```

`append`を呼ぶと内部で下記のことが起きる

```swift
var foo_x = foo.get_x()
foo_x.append(1738)
foo.set_x(foo_x)
```

これはCOWのデータ構造の場合、`get`からの戻り値が一時的に格納されることで参照が複数あるため必ずコピーが発生する。

そこで、値の一部にこのオーバーヘッドなしでアクセスできるようにしたい。

#### 改善

`read`と`modify`を使ってコルーチンでインプレース処理できるようにする。

```swift
struct Foo {
    private var _x: [Int]

    var x: [Int] {
        read { yield _x }
        modify { yield &_x }
    }
}
```

通常の関数は戻り値を一度返すと処理の実行が終わるため、戻り値の値は引数とは別で所有権を持たなければならない。  
一方で、コルーチンは、コルーチンが完了して結果を生成した後も実行を継続し、引数は存続し続ける。そのため、コピーなしに生成された値にアクセスすることができ、オーバーヘッドなしに格納プロパティのインプレースの変更をプロパティや`subscript`に行うことができるようになる。  
内部では、`_read`と`_modify`で既に実装されていて、将来的には正式に使えるようにする。

[Modify Accessors](https://forums.swift.org/t/modify-accessors/31872)

### borrow変数

他にもインプレースに変更できるローカル変数も提案されている。

#### 背景

深い(ネストした)オブジェクトグラフを利用している場合、そのグラフ内で深くネストしているプロパティにローカル変数を割り当てたいと思うことは当然ある:

```swift
let greatAunt = mother.father.sister
greatAunt.sayHello()
greatAunt.sayGoodbye()
```

しかし、グローバル変数やクラスインスタンスのような共有された可変状態のオブジェクトの場合、このローカル変数のバインディングはオブジェクトから取り出した値のコピーが必須になる。上記の`mother`や`mother.father`は同じオブジェクトを参照しているプログラムのどこからでも変更が可能であるため、`mother.father.sister`の値は、ローカル関数内の外側からオブジェクトグラフの変更から独立させるために、ローカル変数`greatAunt`にコピーさせなければならない。値型であっても、同じスコープ内で複数回変更が発生する場合は、その時点での値を保存するために、それぞれの値バインディング時に強制的にコピーが発生する。

#### 改善

そこで、そのオブジェクトグラフを参照している変数の存続している間は、オブジェクトグラフ内の値をインプレースで共有できるようにするために、そのバインディングがアクティブな間は、このような変更(コピー)が起きないようにしたい。これを行うには、コピーせずにインプレースで値にバインドする新しい種類のローカル変数バインディングを導入し、インプレースで値にアクセスするために必要なオブジェクトの借用をアサートする。

```swift
// 参照を生成して借用した値の変更を防ぐ
// 右辺に計算プロパティなどが関わると参照できない
ref greatAunt = mother.father.sister // refは仮の名前

greatAunt.sayHello()
mother.father.sister = otherGreatAunt // ❌　エラー。 greatAuntを借用している間は、mother.father.sisterを変更できない
greatAunt.sayGoodbye()
```

クラスインスタンスのプロパティを関数の引数に渡す場合も、呼び出し先でオブジェクトグラフの共有状態が変更される可能性があるため、コピーが発生する。これも借用変数で回避できる。

```swift
print(mother.father.sister) // mother.father.sisterのコピーが発生する

ref greatAunt = mother.father.sister
print(greatAunt) // 既に借用済みなのでコピーは発生しない
```

また、一度のアクセスでオブジェクトグラフの一部への変更を複数回行うことができるのが望ましい。例えば、

```swift
mother.father.sister.name = "Grace"
mother.father.sister.age = 115
```

これは繰り返し記載が必要というだけではなく、内部の値にアクセスする度に`get`/`set`や`read`/`modify`が呼ばれるため効率が悪い。操作間で共有状態の変更が入る可能性があるため、`mather`、`mother.father`、`mother.father.sister`の順にアクセスするシーケンスを2回繰り替えされなければならない。

上記のように、変数のスコープに対して変更されている値への排他的アクセスをアサートするローカル変数を作成し、アクセスシーケンスを繰り返さずにインプレースで変更できるようにする。

```swift
inout greatAunt = &mother.father.sister // inoutは仮の名前
greatAunt.name = "Grace"
mother.father.sister = otherGreatAunt // ❌ エラー。greatAuntが排他的に借用されている間はmother.father.sisterにアクセスできない
greatAunt.age = 115
```

他にも現在の`inout`が現在使えない箇所も拡張して使用可能にしたい。例えば、`enum`内の`switch`文で自身を更新できるようにしたい。

```swift
enum ZeroOneOrMany<T> {
    case zero
    case one(T)
    case many([T])

    mutating func append(_ value: consuming T) {
        switch &self {
        case .zero:
            self = .one(move(value))
        case .one(let oldValue):
            self = .many([move(oldValue), move(value)])
        case .many(inout oldValues):
            oldValues.append(move(value))
        }
    }
}
```

### WWDC(Building Better Apps with Value Types in Swift)

structについてのより詳細は[Building Better Apps with Value Types in Swift(WWDC2015)](../documents/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift.md)にて記載している。

### Law of Exclusivityとは？

同じ変数(※)への2つのアクセスは、両方のアクセスが読み取りでない限り重複することはできない、というルール。

※ グローバル変数、ローカル変数、クラスおよび構造体のプロパティなど、あらゆる種類の可変メモリを意味する。

Swift3ではこれが導入されていなかった。

詳細は[Law of Exclusivity](./../documents/Law%20of%20Exclusivity.md)

## 参考リンク

### Forums

- [Why are structs in Swift said to be immutable?](https://forums.swift.org/t/why-are-structs-in-swift-said-to-be-immutable/55319)
- [Value and Reference Types](https://developer.apple.com/swift/blog/?id=10)
https://www.vadimbulavin.com/value-types-and-reference-types-in-swift/)
- [SE-0268: Refine didSet Semantics](https://forums.swift.org/t/se-0268-refine-didset-semantics/30049)
- [Copy-on-Write Representation](https://github.com/apple/swift/blob/main/docs/SIL.rst#copy-on-write-representation)
- [Understanding Swift Copy-on-Write mechanisms](https://medium.com/@lucianoalmeida1/understanding-swift-copy-on-write-mechanisms-52ac31d68f2f)
- [Enforce Exclusive Access to Memory](https://github.com/apple/swift-evolution/blob/main/proposals/0176-enforce-exclusive-access-to-memory.md)


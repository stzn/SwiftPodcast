# Swift ARCと所有権(Ownership)

- [Swift ARCと所有権(Ownership)](#swift-arcと所有権ownership)
  - [概要](#概要)
  - [内容](#内容)
    - [用語説明](#用語説明)
      - [ARCとは？](#arcとは)
      - [所有権(Ownership)とは？](#所有権ownershipとは)
      - [Object graphとは？](#object-graphとは)
    - [提案されている内容](#提案されている内容)
      - [構文ベースの存続期間](#構文ベースの存続期間)
        - [背景](#背景)
        - [改善](#改善)
      - [move関数による明示的な所有権の移行](#move関数による明示的な所有権の移行)
        - [背景](#背景-1)
        - [改善](#改善-1)
      - [consuming/nonconsumingを使った引数での所有権の移行の管理](#consumingnonconsumingを使った引数での所有権の移行の管理)
        - [背景](#背景-2)
        - [改善](#改善-2)
      - [readとmodifyアクセサによるコルーチン(※)](#readとmodifyアクセサによるコルーチン)
        - [背景](#背景-3)
        - [改善](#改善-3)
      - [nonImplicitCopy属性によるコンパイラの暗黙的なコピーの禁止](#nonimplicitcopy属性によるコンパイラの暗黙的なコピーの禁止)
        - [背景](#背景-4)
        - [改善](#改善-4)
      - [汎用的なnonescaping引数](#汎用的なnonescaping引数)
        - [背景](#背景-5)
        - [改善](#改善-5)
      - [借用変数(Borrow variables)](#借用変数borrow-variables)
        - [背景](#背景-6)
        - [改善](#改善-6)
    - [将来的な話](#将来的な話)
      - [move-only型の導入](#move-only型の導入)
      - [既存のAPIのパフォーマンス改善](#既存のapiのパフォーマンス改善)
  - [参考リンク](#参考リンク)

## 概要

Swiftは日常でアプリケーションを書いている場合に、エンジニアがメモリ管理を気にする必要はない。

しかし、パフォーマンスを重視する(もしくは必要なシステムを開発する)エンジニアにとってARCとSwiftのオプティマイザの挙動は変わりやすくどのようにメモリが管理(retain/release)されるのを予測することはとても難しい。

そこで、ARCモデルを簡単に理解できるようにすること、一方でエンジニアがマニュアルにメモリをコントロールできる幅も広げようとしている。今回はここを中心にSwiftの内部で何が行われているのかを見ていきたい。

## 内容

### 用語説明

#### ARCとは？

自動参照カウント(Automatic Reference Counting)の略。ARCは、Objective-Cオブジェクトのメモリの自動管理を提供するコンパイラ機能。

#### 所有権(Ownership)とは？

所有権は、最終的に値を破棄するコードの責任の所在を指す。所有権システムは、所有権を管理および譲渡するための一連の規則または規則。メモリについての所有権を指すことが多いが、プログラムで使われる他のリソースについての所有権の破棄に関する責任を指すこともある。

関連ドキュメント: https://github.com/apple/swift/blob/main/docs/OwnershipManifesto.md

#### Object graphとは？

オブジェクト指向プログラミングでは、オブジェクトのグループは、別のオブジェクトへの直接参照または一連の中間参照のいずれかを介して、相互の関係を通じてネットワークを形成する。この一連のオブジェクトのグループをオブジェクトグラフ(Object graph)と呼ぶ。

### 提案されている内容

#### 構文ベースの存続期間

Swiftのメモリ管理の基本的な挙動をより予測しやすく安定させるためのもの。表に現れる変更ではない。弱参照やポインタ、外部関数の呼び出しなどの存続期間を構文のスコープに紐づける。

##### 背景

Objective-C ARCでは、どのタイミングでメモリ解放が行われているかが定まっていない。(最後に変数が使われるタイミングなのか、構文スコープの終わりなのか)。例えば、デバッグビルドとリリースビルドで解放タイミングが違うなど。これによってテストでは見逃しやすいバグに繋がる可能性がある。

```swift
class MyController {
  weak var delegate: MyDelegate?

  func callDelegate() {
    _ = delegate!
  }
}

let delegate = MyDelegate(controller)
MyController(delegate).callDelegate()
```

例えば、weak変数の存続期間は最後に使われたタイミングで終わる可能性があり、nilなった後にforce-unwrappedしてクラッシュする可能性もある。

##### 改善

そこで、パフォーマンスが重要なケースでは最適化が必要ではあるが、開発者の直感にも従うように言語のルールを変更した方が筋が通ると考え、変数のスコープの終わりと存続期間を紐づける。そうすると、弱参照へのアクセスやポインタを使用する場合などに、オプティマイザがライフタイムを短くすることを防ぐようにする(deinitialization barriers)。

これによって`withExtendedLifetime`を使うようなケースが減る。C++のstrict scoping modelよりは厳密ではなく、`deinit`の順番は保証しない(が、これは大きい問題ではないと考えられる)。

#### move関数による明示的な所有権の移行

今話した構文ベースの存続期間によってメモリ管理が予測可能なったものの、場合よってはライフタイムを短くする最適化も行いたい。

##### 背景

forumの例を見てみる。配列を保持している`struct`のイニシャライザで、配列の初期値を受け取るとする。値をプロパティに代入後にその値をソートする。このソート時に引数で渡された配列からの参照も残っている場合、配列がユニークに参照されていないため、コピーオンライト(※1)が発生する可能性がある。もちろんオプティマイザは値の最後の使用箇所を検知してコピーフォワード(※2)することができるが、これは保証されていない。

※1 コピーオンライト

複製を要求されても、コピーをした振りをして、とりあえず原本をそのまま参照させるが、原本またはコピーのどちらかを書き換えようとしたときに、それを検出し、その時点ではじめて新たな空き領域を探して割り当て、コピーを実行する

※2 コピーフォワード  
メモリへの参照を割り当て先に転送して元の参照を解放

```swift
struct SortedArray {
    var values: [String]

    init(values: [String]) {
        self.values = values
        // valuesがソートされていることを保証する
        self.values.sort()
    }
}
```

##### 改善

これは時間計算量に影響するため、コピーの転送を明示的に保証できるようにしたい。そこで`move`関数を使う。

```swift
struct SortedArray {
    var values: [String]

    init(values: [String]) {
        // valuesの参照をselfに移動させることでvaluesの参照が唯一であることを保証する
        self.values = move(values)
        // valuesがソートされていることを保証する
        self.values.sort()
    }
}
```

代入時に`move`関数を呼び出すことで、引数の値の存続期間がこのタイミングで終了することが保証できる。

関連スレッド: https://forums.swift.org/t/pitch-move-function-use-after-move-diagnostic/53983

#### consuming/nonconsumingを使った引数での所有権の移行の管理

※ consuming/nonconsumingは仮

##### 背景

関数などの呼び出し間での所有権の転送がどうやって行われているのか不明瞭。これを明示したい。

また、`inout`引数の場合、インプレース(※)で変更するため引数の値を借りなければならないが、他の通常の関数に関しては通常の引数に使用する規則を特定していない。

実際は経験に基づいたいくつかのルールにしたがっている。

- 大抵の通常の関数は引数を借りている
- `init`の引数と`setter`の`newValue`は所有権が譲渡される(`init`と`setter`は値の構築や既存の値の更新に使われることが多いため、余計なコピーやretain/releaseが起きないようにしている)

このルールで良い場合は多いが、ARCやコピーを最小限にするまで、このデフォルトの挙動をオーバーライドできるようにしたい。

※ インプレース(In-place)アルゴリズム

追加の記憶領域をほとんど使わずに行うアルゴリズム

##### 改善

関数に引数を渡す際に、

- 呼び出し元(caller)がその引数の値を呼び出し先(callee)に貸し出し、呼び出し先(callee)は一時的に所有権を引き受けて、呼び出し先(callee)がリターンする際に所有権が呼び出し元(caller)に戻る
- 呼び出し先(callee)に、呼び出し元(caller)の値の所有権を譲渡する。呼び出し先(callee)では使用後にそれを解放するか、他に所有権をさらに転送する責任を持つ
例えば、`Array`の`append`は既存のデータ構造に値を挿入するだけなので、`consuming`をつける

```swift
extension Array {
    mutating func append(_ value: consuming Element) { ... }
}
```

逆に`init`の引数で値の初期化に使用されない引数には`nonconsuming`をつけることができる

```swift
struct Foo {
    var bars: [Bar]

    // nameはlogにしか使われていない。
    // そのため、nonconsumingにすることでcaller側のretainを防ぐ
    init(bars: [Bar], name: nonconsuming String) {
        print("creating Foo with name \(name)")
        self.bars = move(bars)
    }
}
```

関連スレッド: https://forums.swift.org/t/pitch-formally-defining-consuming-and-nonconsuming-argument-type-modifiers/54313

#### readとmodifyアクセサによるコルーチン(※)

※ コルーチン
関数を任意の箇所でストップ、再開できる機能。サブルーチンがエントリーからリターンまでを一つの処理単位とするのに対し、コルーチンはいったん処理を中断した後、続きから処理を再開できる。

##### 背景

`get`/`set`の仕組みは関数の呼び出しと同じ。`get`から戻り値を返す時にコピーが起きる。`set`は値の所有権が譲渡される。

そうするといくつかのオーバーヘッドが生じる。

```swift
struct Foo {
  private var _x: [Int]

  var x: [Int] {
      get { return _x }
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

これはコピーオンライトのデータ構造の場合、`get`からの戻り値が一時的に格納されることで参照が複数あるため必ずコピーが発生する。

そこで、値の一部にこのオーバーヘッドなしでアクセスできるようにしたい。

##### 改善

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
内部では、`_read`と`_modify`で既に実装されている。これを正式に使えるようにする。

関連スレッド: https://forums.swift.org/t/modify-accessors/31872

#### nonImplicitCopy属性によるコンパイラの暗黙的なコピーの禁止

##### 背景

これまでの話してきた機能で、値の存続期間の終了タイミングや、関数間、プロパティやsubscriptアクセス時の所有権の流れをコントロールできるようになる。

その一方で、Swiftは通常は必要に応じて値のコピーを自由に挿入することもできる。この暗黙的なコピーを禁止することで、コンパイラが開発者のコードを最適化して、将来のコンパイラの変更があっても最適化し続けることもできるべき。

##### 改善

そこで明示的に呼び出し元でcopyを実行するように設定できるようにする(暗黙的なコピーを禁止する)。関数の引数に`@nonImplicitCopy`属性を付ける。

※ `@nonImplicitCopy`は仮

```swift
class C {}

func borrowTwice(first: C, second: C) {}
func consumeTwice(first: consuming C, second: consuming C) {}
func borrowAndModify(first: C, second: inout C) {}

func foo(x: @noImplicitCopy C) {
    // OK. nonconsumingなので同じ値を複数回貸し出している
    borrowTwice(first: x, second: x)

    // 引数の両方でconsumeしたいかつxはその後も使用し続けたいため、通常は2回コピーが必要。
    // そのためエラーが発生する
    consumeTwice(first: x, second: x) // ❌　error: copies x, which is marked noImplicitCopy
    // 使用するためには呼び出し元でcopyが必要になる
    consumeTwice(first: copy(x), second: copy(x)) // ⭕️ OK

    // この場合、xをIn-placeで変更するために排他的にアクセスが必要になるため、
    // 最初の不変の引数では値を借りるのではなく、コピーが必要になる。
    borrowAndModify(first: copy(x), second: &x)

    // この時点でxを使うのは最後なので、第2引数でxをmoveすることができる
    consumeTwice(first: copy(x), second: move(x))
}
```

過度のコピーやARCトラフィックを防ぐためにも、必要なところに明示的にコピーを指定するのは有効。

#### 汎用的なnonescaping引数

##### 背景

引数の値を借りる関数(`nonconsuming`引数を持つ通常の関数)で暗黙的なコピーを選択的に防ぐことができることを上記で示したが、明示的にコピーを防ぐこともできる。

これを行うには、引数は事実上`nonescaping`になる。`nonescaping`だと、呼び出し先で渡した値のコピーや保持がどこでも行われず、関数の呼び出し中は存続し続けすることができる。

クロージャにはこの概念が取り入れられている。クロージャの引数はデフォルトで`nonescaping`で、呼び出しの期間を超えて使われる場合は`@escaping`の指定が必要。クロージャを`nonescaping`にするとパフォーマンスや正しさという点でメリットがある。具体的にはスタックに割り当てられ、retain/releaseが発生しない(`escaping`は逆)。また`inout`引数をクロージャのスコープ内で、安全にキャプチャして変更できる。

クロージャ以外でもこのメリットを受けられる。Swiftのオプティマイザは、限られた状況においては、`class`、`Array`、および`Dictionary`をスタックに割り当てることが既にできるが、関数呼び出しでは、通常は引数をエスケープすると想定する必要があるためその能力は制限される。

##### 改善

そこで引数に`@nonescaping`をつけることで、更なる最適化が図れる。

```swift
func foo(x: Int, y: Int, z: Int) {
    // 配列をスタックに割り当てたい
    let xyz = [x, y, z]

    // しかし、これは引数はエスケープされる可能性があるように見える
    print(xyz)
}

// しかし、nonescapingをつけるとスタックに割り当てることができる
func print(_ args: @nonescaping Any...) { }
```

`withUnsafePointer`のようなローレベルのAPIなどの多くのAPIでエスケープしてはならないものがあるが、現状プログラマが正しく実装することに依存している。これらを`nonescaping`にすることで安全に使用できる。

```swift
func withUnsafePointer<T, R>(to: T, _ body: (@nonescaping UnsafePointer<T>) -> R) -> R

let x = 42
var xp: UnsafePointer<Int>? = nil
withUnsafePointer(to: x) { p in
    xp = p // ❌　 エラー!!pはエスケープできない
}
```

#### 借用変数(Borrow variables)

##### 背景

深い(ネストした)オブジェクトグラフを利用している場合、そのグラフ内で深くネストしているプロパティロにローカル変数を割り当てたいと思うことは当然ある:

```swift
let greatAunt = mother.father.sister
greatAunt.sayHello()
greatAunt.sayGoodbye()
```

しかし、グローバル変数やクラスインスタンスのような共有された可変状態のオブジェクトの場合、このローカル変数のバインディングはオブジェクトから取り出した値のコピーが必須になる。上記の`mother`や`mother.father`は同じオブジェクトを参照しているプログラムのどこからでも変更が可能であるため、`mother.father.sister`の値は、ローカル関数内の外側からオブジェクトグラフの変更から独立させるために、ローカル変数`greatAunt`にコピーさせなければならない。値型であっても、同じスコープ内で複数回変更が発生する場合は、その時点での値を保存するために、それぞれの値バインディング時に強制的にコピーが発生する。

##### 改善

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

### 将来的な話

#### move-only型の導入

暗黙的にコピーしない、およびエスケープしない変数は、ARCがコピーを挿入する必要のない方法でのみ変数を使用するようにコンパイラに要求するため、事実上「move-only変数」と呼べる。引数の`consuming`および`nonconsuming`修飾子、アクセサコルーチンの`read`/`modify`、および`ref`変数と`inout`変数を使用すると、ローカル変数は、コピーせずにデータ構造とオブジェクトグラフへの`non-mutating`および`mutating`参照を作成できる。これらは全てmove-only型(=全ての値がコピーできない型)をサポートする方法を明確にしている。

move-only型はARCのオーバーヘッドなしにユニークにリソースを保持することが表現できる(Ownership Manifestより)。つまり、複数のコピーが安全に存在することができない。アトミック変数やロックなどのローレベルの同時並行にアクセスされるプリミティブなど。

このロードマップには含まれていないが、上記の機能が将来への追加への布石となっている。

#### 既存のAPIのパフォーマンス改善

一方でmove-only型がなくても上記の機能を使って、既存の安全なAPI以上にメモリ安全にオーバーヘッドを減らしたAPIを追加できる。

例えば、`public`なイニシャライザがないけれども、clientから`@nonescaping`な引数を使ってのみ利用できる型など(コピーできず、限られた範囲でのみリソースが使われることが保証される)で活用できる。
`UnsafeBufferPointer`と同等に効率的で柔軟な方法で連続したメモリ領域を参照することができる。しかも`ArraySlice`と同じくらい安全に。

```swift
struct BufferView<Element> {
    // no public initializers

    subscript(i: Int) -> Element { read modify }

    subscript<Range: RangeExpression>(range: Range) -> BufferView<Element> {
        @nonescaping read
        @nonescaping modify
    }

    var count: Int { get }
}

extension Array {
    subscript<Range: RangeExpression>(bufferView range: Range) -> BufferView<Element> {
        @nonescaping read
        @nonescaping modify
    }
}

var lastSummedBuffer: BufferView<Int>?

func sum(buffer: @nonescaping BufferView<Int>) -> Int {
    var value = 0

    // ❌  エラー bufferはスコープからエスケープできない
    lastSummedBuffer = buffer

    // Move-only型によってBufferViewをSequenceに準拠させることができるかもしれない
    // それまではindex使ってループする必要がある
    for i in 0...buffer.count {
        value += buffer[i]
    }
}

let primes = [2, 3, 5, 7, 11, 13, 17]
// コンパイラからは配列がエスケープするように見えず、参照カウントなしに配列のBufferViewに渡すことができる
// こうすることで、固定の配列としてスタックに割り当てられ、グローバルな静的配列として最適化できる可能性が高まる
let total = sum(buffer: primes[bufferView: ...])
```

`nonescaping`制約により、`BufferView`は安全でありながら、`UnsafeBufferPointer`と同等のオーバーヘッドしかなく、`BufferViews`を生成する`@nonescaping`コルーチンは、現在Swiftの標準ライブラリで使用されている`withUnsafe...{}`クロージャベースのパターンのより、表現力豊かな代替手段を提供できる。

他にもResourceの使用、破棄をスコープ内で安全に管理できるようにもなる。例えば、

```swift
extension ResourceHolder {
    func withScopedResource<R>(_ body: (ScopedResource) throws -> R) rethrows -> R {
        let scopedResource = setUpScopedResource()
        defer { tearDownScopedResource(scopedResource) }
        try body(scopedResource)
    }
}
```

は、下記に置き換えられる。

```swift
extension ResourceHolder {
    var scopedResource: ScopedResource {
        @nonescaping read {
            let scopedResource = setUpScopedResource()
            defer { tearDownScopedResource(scopedResource) }
            yield scopedResource
        }
    }
}
```

## 参考リンク

- https://forums.swift.org/t/a-roadmap-for-improving-swift-performance-predictability-arc-improvements-and-ownership-control/54206
- https://github.com/apple/swift/blob/main/docs/OwnershipManifesto.md
- https://forums.swift.org/t/pitch-move-function-use-after-move-diagnostic/53983
- https://forums.swift.org/t/pitch-formally-defining-consuming-and-nonconsuming-argument-type-modifiers/54313
- https://forums.swift.org/t/psa-compiler-optimisation-remarks/54152/6
- https://forums.swift.org/t/modify-accessors/31872
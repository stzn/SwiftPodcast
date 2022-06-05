# Swift 暗黙的な存在型の開示

- [Swift 暗黙的な存在型の開示](#swift-暗黙的な存在型の開示)
  - [用語紹介](#用語紹介)
  - [概要](#概要)
  - [内容](#内容)
    - [`any`と`some`間の移動](#anyとsome間の移動)
  - [詳細](#詳細)
    - [いつ存在型を開示するか](#いつ存在型を開示するか)
    - [戻り値の型消去](#戻り値の型消去)
    - [戻り値を型消去する際に「失われる」制約](#戻り値を型消去する際に失われる制約)
    - [関数型のパラメータの反変型消去](#関数型のパラメータの反変型消去)
    - [評価制限の順序](#評価制限の順序)
    - [存在型がプロトコル要件を満たしている場合は開かない（Swift 5の場合）](#存在型がプロトコル要件を満たしている場合は開かないswift-5の場合)
    - [as any P / as! any Pを使った明示的な開示の抑制](#as-any-p--as-any-pを使った明示的な開示の抑制)
  - [ソース互換性](#ソース互換性)
  - [検討された代替案](#検討された代替案)
    - [明示的な存在型の開示](#明示的な存在型の開示)
    - [値に依る存在型の開示](#値に依る存在型の開示)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 用語紹介

- 開示: 存在型の基底型を取り出すこと

## 概要

存在型(Existential Type)は、特定の型を知らずに実行時に変化する値を保持できる。この保持している動的な型を存在型の「基底型」と呼び、その型に準拠した一連のプロトコルか、潜在的にサブクラスのみがその型について知っている。存在型は動的な型の値を表現するのに有用だが、その動的な性質による制限がある。最近の[anyキーワードで存在型を明示](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md)したり、[全てのプロトコルを存在型として利用可能にする](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)機能の追加などによって多くの制限はなくなった。しかし、存在型の根本的な問題は解決していない。というのも、一度存在型として値を受け取った後は、ジェネリクスと一緒に使うのが非常に難しい。開発者はよく`protocol 'P' as a type cannot conform to itself`というエラーに遭遇する

```swift
protocol P {
    associatedtype A
    func getA() -> A
}

func takeP<T: P>(_ value: T) { }

func test(p: any P) {
    takeP(p) // ❌: protocol 'P' as a type cannot conform to itself
}
```

ジェネリクスのシステムとやり取りをする際、Swiftの存在型はちょっとした罠になる。つまり、ジェネリクスから存在型に移行するのは簡単だが、いったん存在型になると、ジェネリクスに戻して使用するのは非常に困難。最悪の場合、多くの階層の関数に戻って、パラメータまたは結果を`any P`からジェネリックな`P`に変更するか、独自の[型消去](https://www.swiftbysundell.com/articles/different-flavors-of-type-erasure-in-swift/)を作成する必要がある。

そこで、存在型の値を「開示」することを可能にし、ジェネリックパラメータをその基底型に紐づけることによって、この存在型の罠に対処する。

存在型の値を「開示」することで、存在型の値でジェネリック関数を呼び出すことができる。これにより、ジェネリック関数は、存在型のボックス自体ではなく、存在型の値の基底型の値として動作し、大幅なリファクタリングなしで存在型の罠から抜け出すことができる。この機能は、存在型のメンバ(`p.getA()`など)にアクセスするときには、言語の機能としてすでに存在している。今回は、その動作を、ほとんど見えないように意図された方法ですべての呼び出し引数に拡張する。つまり、これまで失敗していた(上記の`takeP(p)`のような)ジェネリック関数の呼び出しが成功するようになる。存在型とジェネリクスの間のやり取りをスムーズにすることで、Swiftのコードを簡潔にし、言語をより扱いやすくすることができる。

## 内容

存在型からより強い制約のジェネリクス型に戻すのを簡単にするために、ジェネリックパラメータに渡されるときに存在型の値を暗黙的に「開示」する。この場合、ジェネリック引数は、「ボックス」としての存在型ではなく、存在型の値の基底型を参照する。以下に、`Self`要件を含むプロトコル`Costume`に対して、`Costume`のいくつかのプロパティをチェックするジェネリック関数を書いてみる:

```swift
protocol Costume {
    func withBells() -> Self
    func hasSameAdornments(as other: Self) -> Bool
}

// ⭕️: ベルを追加すると何か変わるかをチェックするジェネリック関数
func hasBells<C: Costume>(_ costume: C) -> Bool {
    return costume.hasSameAdornments(as: costume.withBells())
}
```

これは問題ない。次に、すべてのコスチュームに大きなフィナーレのベルが付いていることを確認する関数を作成する。これは、存在型の値の配列とジェネリック関数の境界で問題が発生する。

```swift
func checkFinaleReadiness(costumes: [any Costume]) -> Bool {
    for costume in costumes {
        if !hasBells(costume) { // ❌: protocol 'Costume' as a type cannot conform to the protocol itself
          return false
        }
    }

    return true
}
```

`hasBells`の呼び出しでは、ジェネリックパラメータ`C`は、`any Costume`型(例えば、基底型が不明な型の値を保持するボックス)、に紐付けられている。そのボックスの型の各インスタンスは実行時に異なる型を持つ可能性があるため、基底型が`Costume`に準拠していたとしても、そのボックス自体は`hasSameAdornments`の要件を満たしていないため、`Costume`に準拠していない(例えば、2つのボックスが同じ基底型を保持していることは保証されていない)。

このプロポーザルでは、存在型の暗黙的な開示を導入する。これにより、存在型の値(たとえば、`any Costume`)を、そのジェネリックパラメータにその基底型を渡せるような場合に使うことができる。たとえば、上記の`hasBell(costume)`の呼び出しは、ジェネリックパラメータ`C`をその特定の`Costume`のインスタンスの基底型に紐付けることに成功する。ループの各反復は、`C`に紐づいた異なる基底型である可能性がある:

```swift
func checkFinaleReadiness(costumes: [any Costume]) -> Bool {
    for costume in costumes {
        if !hasBells(costume) { // ⭕️: Cはanyのボックス内部に保持された実行時にのみわかる型に紐づく
            return false
        }
    }

    return true
}
```

存在型の暗黙的な開示は、動的に型付けされた値を取得し、それをジェネリックパラメータに紐づけることで基底型に名前を付け、動的に型付けされた値からより静的に型付けされた値へと実質遷移できる。この概念は実際には新しいものではない。存在型の値でプロトコルのメンバを呼び出すと、`Self`型が暗黙的に「開示」される。既存の言語機能では、プロトコル拡張のメンバとして`hasBells`の隙間を埋めることができる:

```swift
extension Costume {
    var hasBellsMember: Bool {
        hasBells(self) 
    }
}

func checkFinaleReadinessMember(costumes: [any Costume]) -> Bool {
    for costume in costumes {
            if !costume.hasBellsMember { // 今は⭕️: 'Self'はanyのボックス内部に保持された実行時にのみわかる型に紐づく
            return false
        }
    }

    return true
}
```

こういった意味で考えると、ジェネリック関数の呼び出しのために存在型の暗黙的な開示は、この既存の動作をすべてのジェネリックパラメータに一般化することになる。`hasBellsMember`の例が示すように、この開示する動作をするために、プロトコル拡張でいつでもメンバを書き込むことができるが、これは厳密にはより表現的であるわけではない。このプロポーザルでは、存在型の暗黙的な開示をより一般的にすることによって、より統一され、わかりやすくすることを目的としている。

「準備」チェックの最後の実装を考えてみる。ここでは、ロジックを個別のジェネリック関数`hasBells`に入れずに、ベルのチェックの「コードを開き」たいとする:


```swift
func checkFinaleReadinessOpenCoded(costumes: [any Costume]) -> Bool {
    for costume in costumes {
        let costumeWithBells = costume.withBells() // 戻り値は'any Costume'
        if !costume.hasSameAdornments(costumeWithBells) { // ❌: 'any Costume'は必ずしも同じ型とは限らない
            return false
        }
    }

    return true
}
```

ここで注意すべきことが2つある。まず、`withBells()`メソッドは`Self`型を返す。`any Costume`の値でそのメソッドを呼び出す場合、具体的な結果の型は不明であるため、`any Costume`コスチュームに型消去される(これが`costumeWithBells`の型になる)。次に、次の行で、`hasSameAdornments`を呼び出すと、関数が`Self`型の値を予期しているが、`costume`と`costumeWithBells`の間に静的に型付けされたリンクはないため(どちらも`any Costume`型)、エラーが発生する。 存在型の引数の暗黙的な開示は呼び出し時にのみ発生するため、呼び出しの最後にその効果は型消去される。 この開示の効果を複数の文で持続させるには、`hasBells`の場合と同様に、そのコードをジェネリックパラメータに名前を付けるためにジェネリック関数を抽出する。

### `any`と`some`間の移動

このプロポーザルの興味深い側面の1つは、クライアントコードに大きな影響を与えることなく、`any`のパラメータを`some`のパラメータ([SE-0341](https://github.com/apple/swift-evolution/blob/main/proposals/0341-opaque-parameters.md)で導入)にリファクタリングできること。`some`を使用して、ジェネリックに`hasBells`関数を書き直してみる:

```swift
func hasBells(_ costume: some Costume) -> Bool {
    return costume.hasSameAdornments(as: costume.withBells())
}
```

このプロポーザルを使うと、`any Costume`型の値で`hasBells`を呼び出すことができる:

```swift
func isReadyForFinale(_ costume: any Costume) -> Bool {
    return hasBells(costume) // 存在型の値の暗黙的開示
}
```

静的に入力された`some Costume`から`any Costume`には常に移行できる。このプロポーザルでは、別の方法で、`any Costume`を`some Costume`パラメータに開示することもできる。つまり、`isReadyForFinale`をリファクタリングして、`some`を使って次の方法でジェネリックにすることができる:

```swift
func isReadyForFinale(_ costume: some Costume) -> Bool {
    return hasBells(costume) // ⭕️: `T`はジェネリックパラメータに紐付く
}
```

`isReadyForFinale`の呼び出し側から具体的な型を受け取った場合、`any Costume`で型を「ボックス化」するオーバーヘッドを回避できるようになった。また、`any Costume`を受け取った場合は、`isReadyForFinale`の呼び出し時に存在型を暗黙的に開示する。これにより、すべてのクライアントを同時にジェネリックにすることなく、存在型の操作をジェネリック操作に移行でき、「存在型の罠」から段階的に抜け出すことができる。

## 詳細

基本的に、存在型を開示するとは、存在型のボックスを調べてボックス内に格納されている動的な型を見つけ、その動的な型に「名前」を付けることを意味する。その動的な型名は、どこかのジェネリックパラメータで受け取る必要があるため、静的に推論することができ、その型の値を、呼び出されるジェネリック関数に渡すことができる。このような呼び出しの結果として、その動的な型名を参照する場合もある。その場合、呼び出し後、その動的な型に開示された存在型の値は、開示された型名がユーザから見えないように、再び型消去して存在型に戻す必要がある。これは、既存の言語機能(そのメンバの1つにアクセスするときに存在型の値を開示する)と一致し、これが型システム自体の大きな機能の拡張になってしまうことを防ぐ。

このセクションでは、存在型の開示から、型消去して存在型に戻す方法について詳しく説明する。この変更の内部の詳細は、ユーザには見えないはずであり、今現在の機能では不可能な場所でのみ、ジェネリクスで存在型を使用するための機能として現れる。ただし、動的に型付けされた存在型のボックスから静的に型付けされたジェネリック値への移行は、型の同一性と期待される評価セマンティクスを維持するために慎重に行う必要があるため、多くの内部の実装の詳細が存在する。

### いつ存在型を開示するか

存在型を開示するには、引数(またはソース)は存在型(たとえば、`any P`)または存在型のメタタイプ(たとえば、`any P.Type`)である必要があり、その型が存在型の基底型に直接バインド可能なジェネリックパラメータを含むパラメータ(またはターゲット)に提供される必要がある。これは、たとえば、基底型がジェネリックパラメータに直接バインドできる場合に存在型を開示することができることを意味する:

```swift
protocol P {
    associatedtype A
  
    func getA() -> A
}

func openSimple<T: P>(_ value: T) { }

func testOpenSimple(p: any P) {
    openSimple(p) // ⭕️: 'p'は開示され、'T'は基底型に紐づく
}
```

`inout`パラメータを開示することも可能。ジェネリック関数は基底型で動作し、(たとえば)その上で`mutating`メソッドを呼び出すことができるが、存在型のボックスにアクセスできないため、動的な型を変更することはできない:

```swift
func openInOut<T: P>(_ value: inout T) { }
func testOpenInOut(p: any P) {
    var mutableP: any P = p
    openInOut(&mutableP) // ⭕️: 'mutableP'は開かれ、'T'は基底型に紐づく
}
```

ただし、存在型の値が複数ある場合や値がまったくない場合は、推論される基底型が1つであることを保証する必要があるため、開示することはできない。存在型の引数を開かないようにジェネリックパラメータを複数の場所で使用している例をいくつか示す:

```swift
func cannotOpen1<T: P>(_ array: [T]) { .. }
func cannotOpen2<T: P>(_ a: T, _ b: T) { ... }
func cannotOpen3<T: P>(_ values: T...) { ... }

struct X<T> { }
func cannotOpen4<T: P>(_ x: X<T>) { }

func cannotOpen5<T: P>(_ x: T, _ a: T.A) { }

func cannotOpen6<T: P>(_ x: T?) { }

func testCannotOpenMultiple(array: [any P], p1: any P, p2: any P, xp: X<any P>, pOpt: (any P)?) {
    cannotOpen1(array)         // 配列の各要素は基底型が異なるため、開示することはできない
    cannotOpen2(p1, p2)        // p1とp2の基底型が異なるため、'T'を一貫した型に紐づけることができない
    cannotOpen3(p1, p2)        // 上記と同様にp1とp2の基底型が異なるため、開示することはできない
    cannotOpen4(xp)            // 'X<any P>'には特定の値がないため、開示することはできない
    cannotOpen5(p1, p2.getA()) // 'T'が両方の引数で使われているため、どちらか一方の引数を開示することができない
    cannotOpen6(pOpt)          // nilの可能性があり、基底型ではないかもしれないため、'(any P)?の存在型を開示することはできない
}
```

オプショナルの場合はやや興味深い。`pOpt`が`nil`である可能性があるため、呼び出している`cannotOpen6(pOpt)`が機能しないことは明らか。この場合、`T`に紐付けられる型はない。パラメータがオプショナルの場合、オプショナルではない存在型の引数を開示することを許可することも選択できる:

```swift
cannotOpen6(p1) // 開示することはできるが、それはしないことにした
```

しかし、このプロポーザルでは、この呼び出しを許可するのは不自然であるため、これは許可しない。

上記と同じ制限付きで、存在型のメタタイプの値を開示することもできる。

```swift
func openMeta<T: P>(_ type: T.Type) { }

func testOpenMeta(pType: any P.Type) {
    openMeta(pType) // ⭕️: 'pType'を開示して'T'にその基底型を紐づける
}
```

### 戻り値の型消去

ジェネリック関数の戻り値の型には、ジェネリックパラメータとそれの関連型(associated type)を含めることができる。たとえば、元の値とその関連型のいくつかの値を返すジェネリック関数は次のとおり:

```swift
protocol Q { 
    associatedtype B: P
    func getB() -> B
}

func decomposeQ<T: Q>(_ value: T) -> (T, T.B, T.B.A) {
    (value, value.getB(), value.getB().getA())
}
```

存在型の値を使用して`decomposeQ`を呼び出すと、存在型の値が開示され、`T`はその基底型に紐づけられる。`T.B`と`T.B.A`は、その基底型から派生した型。ただし、呼び出しが完了すると、型`T`、`T.B`、および`T.B.A`は、上限まで型消去される(例: すべてのプロトコル要件を満たす存在型)。たとえば:

```swift
func testDecomposeQ(q: any Q) {
    let (a, b, c) = decomposeQ(q) // aはany Q、bはany P、cはAny
}
```

これは[SE-309で記載されている内容](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md#covariant-erasure-for-associated-types))と一致していて、ここでも同じルールが適用されている。任意のジェネリックパラメータについて、これらの要件をより一般的に次のように言い換えることができる:

ジェネリックパラメータ`T`を、開示している存在型`T`、および`T`の関連型に紐づける場合

- 具体的な型にバインドされていない、かつ
- ジェネリック関数の戻り値の型内の共変位置(covariant)にある

メンバへのアクセスに使用される存在型の一般的なシグネチャに従って、上限まで型消去される。上限は、関連型へのジェネリック制約の存在と種類に応じて、クラス、プロトコル、プロトコル合成、または`Any`のいずれかになる。

`T`または`T`の関連型が戻り値の型の非共変位置(invariant)にある場合、型消去された結果を表す方法がないため、`T`を存在型の値の基底型に紐づけることはできない。これは、前述のように、存在型が開示しないようにするパラメータの型について説明したものと本質的に同じ性質。たとえば:

```swift
func cannotOpen7<T: P>(_ value: T) -> X<T> { /*...*/ }
```

ただし、戻り値は存在型への消去が許可されているため、オプショナル、タプル、さらには配列でも開示できる:

```swift
func openWithCovariantReturn1<T: Q>(_ value: T) -> T.B? { /*...*/ }
func openWithCovariantReturn2<T: Q>(_ value: T) -> [T.B] { /*...*/ }

func covariantReturns(q: any Q){
    let r1 = openWithCovariantReturn1(q)  // ⭕️ 'T'は'q'の基底型に紐づく。結果型は'any P'
    let r2 = openWithCovariantReturn2(q)  // ⭕️ 'T'は'q'の基底型に紐づく。結果型は'[any P]'
}
```

### 戻り値を型消去する際に「失われる」制約

開示された存在型を含む呼び出しの戻り値が型消去されると、戻り値の型に関する一部の情報が存在型で表現できない可能性があるため、上記の「上限」は情報を失う。 たとえば、次の例の`b`の型について考えてみる:

```swift
protocol P {
    associatedtype A
}

protocol Q {
    associatedtype B: P where B.A == Int
}

func getBFromQ<T: Q>(_ q: T) -> T.B { ... }

func eraseQAssoc(q: any Q) {
    let b = getBFromQ(q)
}
```

`T.B`を型消去する場合、最も具体的な上限は「関連型`A`が`Int`である`P`に準拠する型」になる。ただし、Swiftの存在型はそのような型を表現できないため、`b`の型は`any P`になる。

Swiftの存在型は、時間の経過とともに表現度が向上してきている。たとえば、[SE-0353 条件付き存在型](https://github.com/apple/swift-evolution/blob/main/proposals/0353-constrained-existential-types.md)を使用すると、[主要な関連型](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md)のバインディングを含む存在型を表現できる。プロトコル`P`にその機能を採用する場合、最も具体的な上限は次のように表現できる。

```swift
// SE-0353があることを想定
protocol P<A> {
    associatedtype A
}
```

ここで、`b`の型は`any P<Int>`であると予想される。存在型の将来の拡張により、ソースコードを変更しなくても、最も具体的な上限を表現できるようになる可能性がある。暗黙的に開示された存在型で関数を呼び出した後の型消去は、これらの機能が追加されるとより正確になると予想される。

ただし、この種の変更は、ソースの互換性に問題をもたらす。これは、コードが`b`の型に依存するようになる可能性があるため。たとえば、オーバーロードなどにより、`P`の精度が低くなる。

```swift
func f<T: P>(_: T) -> Int { 17 }
func f<T: P>(_: T) -> Double where T.A == Int { 3.14159 }

// ...
func eraseQAssoc(q: any Q) {
    let b = getBFromQ(q)
    f(b)
}
```

より具体的でない上限(`any P`)を使用すると、`f(b)`の呼び出しは、`Int`を返す最初のオーバーロードを選択する。より具体的な上限(`A`が`Int`であることがわかっている`any P`)を使用すると、呼び出し`f(b)`は、`Double`を返す2番目のオーバーロードを選択する。

オーバーロードによって、上限を改善することによるソース互換性への影響は、(たとえば)新しい主要言語バージョンまで上限を一定に保たなければ、完全に排除することはできない。ただし、存在型の制限のために上限が一部の要件を表現できない呼び出しに対して、特定の型の強制を要求することによる影響の軽減を提案する。具体的には、`getBFromQ(q)`の呼び出しは次のように記述する必要がある:

```swift
getBFromQ(q) as any P
```

このように、言語の存在型の表現力が増したことで上限が変更された場合でも、式全体は、常に同じ型の値(`any P`)を生成する。開発者は、Swiftが存在型でわかるすべての情報を完全にキャプチャできるようになったタイミングで、`as any P`を自由に削除できる。

明示的な型強制のこの要件は、このプロポーザルの前に存在していたものを含む、存在型の開示によるすべての型消去にも適用されることに注意。たとえば、`getBFromQ`はプロトコル拡張のメンバとして記述できる。以下のコードには、[SE-0309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)で最初に文法的に正しくなったものと同じ問題(および同じ解決策)がある:

```swift
extension Q {
    func getBFromQ() -> B { ... }
}

func eraseQAssocWithSE0309(q: any Q) {
    let b = q.getBFromQ()
}
```

### 関数型のパラメータの反変型消去

共変型消去はジェネリック関数の戻り値の型に適用されるが、その反対はジェネリック関数の他のパラメータに適用される。これは、開示された存在型に紐づいているジェネリックパラメータを参照する関数型のパラメータに影響する。この場合、上限まで型消去される。たとえば:

```swift
func acceptValueAndFunction<T: P>(_ value: T, body: (T) -> Void) { ... }

func testContravariantErasure(p: any P) {
    acceptValueAndFunction(p) { innerValue in // innerValueは'any P'
    // ... 
  }
}
```

戻り値の型に適用される共変型消去と同様に、この型消去は、動的な型に割り当てられた「名前」を、推論されたクロージャパラメータから型システムを通してユーザに見えないようにする。ジェネリック型のパラメータ`T`は、`any P`に紐づいているように見せかけているが、実際には、その特定の値の基底型に紐づいている。

このルールには1つの例外がある。そのようなパラメータへの引数がジェネリック関数への参照である場合、型消去は発生しない。このような場合、動的な型の「名前」
はこの2番目のジェネリック関数のジェネリックパラメータに直接バインドされ、同じ暗黙的な存在型の開示を再び実行する。これは例を示すとわかりやすい:

```swift
func takeP<U: P>(_: U) -> Void { ... }

func implicitOpeningArguments(p: any P) {
    acceptValueAndFunction(p, body: takeP) // ⭕️: TとUは両方ともpの基底型に紐づく
}
```

この動作は、内部の`_openExistential`のほとんどの挙動と一致する。具体的には、1つの存在型の値を開示し、それをジェネリック関数に渡すことのみをサポートする。`_openExistential`は、準拠要件がない存在型を開示するときに、まだいくつかのユースケースが存在する可能性がある。


### 評価制限の順序

存在型のボックスを開示するには、そのボックスを生成する式を評価してから、そのボックスを開示して、基底型を抽出する必要がある。 式の評価には副作用がある可能性がある。たとえば、次の`getP()`関数を呼び出して、存在型のボックス`any P`の値を生成する場合:

```swift
extension Int: P { }

func getP() -> any P {
    print("getP()")
    return 17
}
```

ここで、存在型の引数を開示したいジェネリック関数について考えてみる:

```swift
func takeP<U: P>(_: U) -> Void { ... }

func acceptFunctionStringAndValue<T: P>(body: (T) -> Void, string: String, value: T) { ... }

func hello() -> String {
    print("hello()")
}

func implicitOpeningArgumentsBackwards() {
    acceptFunctionStringAndValue(body: takeP, string: hello(), value: getP()) // エラーになる。詳細は後ほど
}
```

`value`パラメータの引数を開示するには、`getP()`の呼び出しをまず実行する必要がある。`takeP`のジェネリック型パラメータ`U`は、その存在型のボックスの基底型に紐づいているため、これは、`body`パラメータへの引数を形成する前に行われる必要がある。これは、プログラムが次の順序で副作用を生成することを意味する:

```swift
getP()
hello()
```

ただし、これはSwiftの左から右への評価順序と矛盾する。そこで、こうではなく、存在型の引数の暗黙的な開示に別の制限を追加する。基底型に紐づけられたジェネリック型パラメータが、存在型の引数に対応するものより前の関数パラメータで使用されている場合、存在型の引数を開示することはできない。上記の`implicitOpeningArgumentsBackwards`では、`acceptFunctionStringAndValue`の呼び出しでは、値に先行する`body`パラメーターでもジェネリック型パラメータ`T`が使用されているため、`value`パラメーターへの存在型の引数を開示することができない。これにより、開かれた存在型の引数の前の引数に、基底型が不要になるため、左から右への評価順序が維持される。


### 存在型がプロトコル要件を満たしている場合は開かない（Swift 5の場合）

これまでに示したように、存在値を開示すると、存在型のボックスをジェネリック関数に渡すことに依存していた既存のプログラムの動作が変わる可能性がある。たとえば、パラメータを戻り値の配列に入れる制約のないジェネリック関数に存在型のボックスを渡した場合の結果を考えてみる:

```swift
func acceptsBox<T>(_ value: T) -> Any { [value] }

func passBox(p: any P) {
      let result = acceptsBox(p) // 現在'T'は'any P'に推論される、戻り値の型は`[any P]`
      // 制約のない存在型の開示は'p'の基底型'T'に推論される。戻り値の型は`[T]`
}
```

ここで、存在型のボックスが呼び出しの一部として開かれると、`acceptsBox`の結果の動的な型が変わる。変更自体は小さく、実行時まで検出されないため、ジェネリックパラメータの紐づきに依存する既存のSwiftプログラムで問題が発生する可能性がある。したがって、Swift 5では、このプロポーザルは、存在型自体が対応するジェネリックパラメータの準拠要件を満たす場合に存在型の値の開示を防ぎ、厳密に追加のみの変更を加える。以前に機能した存在型の値を持つジェネリック関数の呼び出しは引き続き機能する。同じセマンティクスを使用するが、以前は機能しなかった呼び出しは存在型を開き、したがって成功するかもしれない。

ジェネリックパラメータが存在型に紐づいている今日のSwiftのほとんどの場合は、`acceptsBox`の`T`ジェネリックパラメータのように、ジェネリックパラメータに準拠要件がないため成功する。ほとんどのプロトコルでは、対応する型を参照する存在型はそのプロトコルに準拠していない。つまり、`any Q`は`Q`に準拠していない。ただし、いくつかの例外がある:

- エラーの存在型は、[SE-0235で指定されているように](https://github.com/apple/swift-evolution/blob/main/proposals/0235-add-result.md#adding-swifterror-self-conformance)`Error`プロトコルに準拠している
- `Q`に`static`要件が含まれていない`@objc`プロトコル`Q`の存在型 `any Q`は、`Q`に準拠する

たとえば、エラーが発生する操作について考えてみる。 `any Error`型の値を渡すと、存在型を開かなくても成功する:

```swift
func takeError<E: Error>(_ error: E) { }

func passError(error: any Error) {
    takeError(error)  // ⭕️ 開かないで成功: 'any Error'は'Error'に準拠しているため、'E'は'any Error'に紐づく
}
```

このプロポーザルは、存在型が上記の結果に従って対応するジェネリックパラメータの準拠要件を満たす場合に存在型の引数を開示しないことにより、上記の呼び出しのセマンティクスを保持する。Swiftが最終的に存在型をプロトコルに準拠させるメカニズムを拡張した場合(たとえば、`any Hashable`が`Hashable`に準拠するように）、これらの準拠を利用したコードは新しく有効になるコードになるため、そのような準拠は暗黙的な開示を抑制しない。暗黙的に開示される。

Swift 6は、いくつかのセマンティクスおよびソースを壊す変更を組み込むことができる主要な言語バージョンの変更になる。Swift 6では、このセクションで説明されている抑制メカニズムは適用されないため、上記の`passBox`の例では、`p`の値を開示し、開示した存在型に`T`を紐づける。これにより、より一貫性のあるセマンティクスが提供され、さらに、`type(of:)`のすべての動作と内部の`_openExistential`操作とも一致する。

### as any P / as! any Pを使った明示的な開示の抑制

何らかの理由で存在型の値の暗黙的な開示を抑制したい場合は、強制または強制キャストを存在型に明示的に書くことができる。たとえば:

```swift
func f1<T: P>(_: T) { }   // #1
func f1<T>(_: T) { }      // #2

func test(p: any P) {
    f1(p)          // pを開示してより具体的な#1を呼ぶ
    f1(p as any P) // pの開示を抑制して、有効な唯一の候補である#2を呼ぶ
}
```

ジェネリック関数が、他の方法では呼び出せない場合に、存在型の暗黙的な開示が起きるように定義されていることを考えると、この抑制メカニズムはSwift 5では頻繁には必要とされないと思われる。暗黙的な開示がより積極的に実行されるSwift 6では、Swift 5のセマンティクスを提供するために使用される。

## ソース互換性

このプロポーザルは、特にSwift 5で、ソースの互換性へのほとんどの影響を回避するために特別に定義されている。以前は形式が正しくなかったジェネリック関数への呼び出し(たとえば、`P`が`any P`に準拠していないために失敗するなど)がうまくいくようになり、既存のコードは以前と同じように動作する。このような変更と同様に、これまでうまく機能していたオーバーロードは引き続き使えるものの、別のオーバーロードが選択されるかもしれない。たとえば:

```swift
protocol P { }

func overloaded1<T: P, U>(_: T, _: U) { } // A
func overloaded1<U>(_: Any, _: U) { }     // B

func changeInResolution(p: any P) {
    overloaded1(p, 1) // これまではBが選ばれたが、このプロポーザル後はAが選ばれる
}
```

このような例は、不正なコードを正しいコードにする機能を抽象的に構築するのは簡単だが、これらの例が実際に問題を引き起こすことはめったにない。

## 検討された代替案

### 明示的な存在型の開示

この提案は、暗黙的に呼び出し側で存在型を開示する。代わりに、存在型を開示する明示的な構文を提供するもできる。例えば、[`some Pへの強制`](https://forums.swift.org/t/pitch-implicitly-opening-existentials/55412/8)など。例えば:

```swift
protocol P {
    associatedtype A
}

func takesP<T: P>(_ value: T) { }

func hasExistentialP(p: any P) {
    takesP(p) // 現在は❌　('any P' does not conform to 'P')。暗黙的開示によって⭕️になる
}
```

これを明示的に開示の場合、以下のようになる:

```swift
func hasExistentialP(p: any P) {
    takesP(p) // 現在は❌　('any P' does not conform to 'P')で、この先も❌
    takesP(p as some P) // 明示的な存在型の開示
}
```

今回の暗黙的開示に比べて、このアプローチには2つの利点がある。1つ目は、これが純粋に付加的で、完全にオプトインの機能であり、ソースコードで見つけやすいこと。2つ目は、開示された存在型が関数の本文全体で存続することができること。これにより、コードを別の(ジェネリック)関数に分解することなく、このプロポーザル最初の方で示した「オープンコード化された」フィナーレチェックを記述できる:

```swift
func checkFinaleReadinessOpenCoded(costumes: [any Costume]) -> Bool {
    for costume in costumes {
        let openedCostume = costume as some Costume //  型は「この時点で開示されたcostume」"
        let costumeWithBells = openedCostume.withBells() // openedCostumeと同じ型を返す
       if !openedCostume.hasSameAdornments(costumeWithBells) { // ⭕️ 同じ型であることがわかっている
        return false
    }
  }

  return true
}
```

`opendCostume`の型は、`as some Costume`式が発生した時点での可変の`costume`の値の動的な型に基づいている。その型は、値が作成されるスコープを「エスケープ」することを許可してはいけない。これは、いくつかの制限を意味する:

- 非静的ローカル変数のみが存在型を開示することができる。他の種類の変数は、動的な型が変更される可能性がある後の時点で参照できてしまう
- 開示された存在型の値は、opaque typeの基になる型が関数に提供された実行時の値に依存するため、opaque result type(`some P`など)を持つ関数から返すことができない

さらに、明示的な開示式があるということは、開かれた存在型がユーザから見える型システムの一部になることを意味する。`openedCostume`の型は、制約(`P`)と式が発生したソースコード内の場所に基づいてのみ推論できる。同じ変数を続けて開示すると2つの異なる型を生成する:

```swift
func f(eq: any Equatable) {
    let x1 = eq as some Equatable
    if x1 == x1 { ... }  // ⭕️

    let x2 = eq as some Equatable
    if x1 == x2 { ... } // ❌: "eq as some Equatable"によってx1とx2で異なる型を生成している
}
```

明示的な開示構文は、今回の暗黙的開示よりも単一の関数においてはより表現力がある。これは、新しいジェネリック関数を導入しなくても、同じ開示された存在型に由来していることが静的にわかるさまざまな値を操作できるため。ただし、この明示性には、言語の表面積の対応する増加が伴う。つまり、明示的な開示を実行する式(`as some P`)だけでなく、これまでユーザに公開されていないコンパイラの実装詳細であった開示される型に対する型システムの概念にまで及。ぶ

対照的に、暗黙的開示は、言語の事実上の表面積を増やすことなく、言語の表現度を向上させる。開示は暗黙的で、開かれた型は実装の詳細のまま。

この「代替案」は、おそらく将来の方向性を示しているかもしれない。このプロポーザルは、明示的に開示する存在型を将来追加することを妨げるものではない。それらが有用であることが証明された場合でも、このプロポーザルで説明されているように、型消去を使用して暗黙的に開示する必要がある。その場合、このプロポーザルの暗黙的な動作は、明示的な構文で記述できる何かを推論するものとして遡及的に理解できる:

```swift
protocol Q { }

protocol P {
  associatedtype A: Q
}

func getA<T: P>(_ value: T) -> T.A { ... }

func unwrap(p: any P) {
    let a = getA(p) // 暗黙的に"getA(p as some P) as any Q"と同じ
}
```

### 値に依る存在型の開示

このプロポーザルの暗黙的開示は、常に特定のジェネリックパラメータ(`T`)の特定のバインディングにスコープされ、その後型消去される。たとえば、これは、同じ存在型に対する同じジェネリック関数の2つの呼び出しが、(静的に)同等だと分からない状態で存在型の値を返すことを意味する。

```swift
func identity<T: Equatable>(_ value: T) -> T { value }
func testIdentity(p: any Equatable) {
    let p1 = identity(p) // p1は型消去された'any Equatable'を取得する
    let p2 = identity(p) // p2は型消去された'any Equatable'を取得する
    if p1 == p2 { ... } // ❌ : p1とp2が同じ型だとわからない
    
    let openedP1: some P = identity(p) // openedP1は呼び出し側で指定した基底型にバインドされたopaque type
    let openedP2: some P = identity(p) // openedP2は呼び出し側で指定した基底型にバインドされたopaque type
    if openedP1 == openedP2 { ... } // ❌ : openedP1とopenedP2が同じ型だとわからない
}
```

開かれた存在型のアイデンティティが存在型の値と紐づくと想像するかもしれない。たとえば、`identity(p)`への2つの呼び出しは、値`p`の基底型に基づいているため、同一のopaque typeを生成する可能性がある。一部のエンティティの(静的な)型は値によって決定されるため、これは依存型(※)の形式。値が変化する可能性がある場合、この形式は崩れる:

※ https://ja.wikipedia.org/wiki/依存型

```swift
func identityTricks(p: any Equatable) {
    let openedP1 = identity(p) // openedP1は'p'の基底型
    let openedP2 = identity(p) // openedP2は'p'の基底型
    if openedP1 == openedP2 { ... } // ⭕️ 両方とも'p'の基底型
    
    var q = p                          // qは'p'の基底型
    let openedQ1: some P = identity(q) // openedQ1は'q'の基底型。つまり`p`
    if openedP1 == openedQ1 { ... }    // ⭕️ 両方とも'p'の基底型

    if condition {
        q = 17   // `q`の基底型が変わる
    }
    
    let openedQ2: some P = identity(q)
    if openedQ1 == openedQ2 { } // ❌ openedQ1の基底型は'p'だが
                                // openedQ2の基底型は'q'でこの時点では型が異なっている
}
```

このアプローチは、型システムに値の追跡を導入するため(この存在型の値はどこで生成されたか？)、はるかに複雑。この時点で、変数への変更がシステムの静的な型に影響を与える可能性がある。


## 参考リンク

### Forums

- [[Pitch] Implicitly opening existentials](https://forums.swift.org/t/pitch-implicitly-opening-existentials/55412)
- [[Pitch #2] Implicitly opening existentials](https://forums.swift.org/t/pitch-2-implicitly-opening-existentials/56360)
- [SE-0352: Implicitly Opened Existentials](https://forums.swift.org/t/se-0352-implicitly-opened-existentials/56557)
- [SE-0352 (second review): Implicitly Opened Existentials](https://forums.swift.org/t/se-0352-second-review-implicitly-opened-existentials/56878)

### プロポーザルドキュメント

- [Implicitly Opened Existentials](https://github.com/apple/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md)
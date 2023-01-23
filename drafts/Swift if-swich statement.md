# if/switch式

- [if/switch式](#ifswitch式)
  - [概要](#概要)
  - [動機](#動機)
  - [詳細](#詳細)
  - [将来の方向性](#将来の方向性)
    - [完全式](#完全式)
    - [`do`式](#do式)
    - [`break`と`continue`のサポート](#breakとcontinueのサポート)
    - [guard](#guard)
    - [複数行ある分岐](#複数行ある分岐)
    - [`Either`](#either)
  - [代替案](#代替案)
    - [現状維持](#現状維持)
    - [他の構文](#他の構文)
  - [ABI stabilityへの影響](#abi-stabilityへの影響)
  - [謝辞](#謝辞)
  - [メモ](#メモ)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

次の場合に、`if`/`switch`文を式(expression)として使用する機能を導入する。

- 関数、プロパティ、およびクロージャから値を返す
- 変数への値の代入
- 変数の宣言

## 動機

Swiftのクロージャには、本文が単一の式の場合に`return`を省略できるぶっきらぼうだけど読みやすい構文がある。[SE-0255:単一式関数からの暗黙的な戻り値](https://github.com/apple/swift-evolution/blob/master/proposals/0255-omit-return.md)は、今回、これを本文が単一の式の関数とプロパティにも拡張した。

この`return`キーワードの省略は、Swiftの何かを達成するために少量のコードを書く(low ceremony)という考えに沿ったものであり、Swiftと同じような多くの「モダン」言語に共通している。ただし、Swiftは、`if`式と`switch`式がサポートされていないという点でそれら同類の言語とは異なる。

これにより`return`キーワードを入れなければならない、いわゆる必要以上のコードを書く「儀式」が必要な場合がある。例えば:

```swift
public static func width(_ x: Unicode.Scalar) -> Int {
    switch x.value {
        case 0..<0x80: return 1
        case 0x80..<0x0800: return 2
        case 0x0800..<0x1_0000: return 3
        default: return 4
    }
}
```

他にもユーザは読みにくい 3 項演算子を使いたくなるかもしれない。

```swift
let bullet = isRoot && (count == 0 || !willExpand) ? ""
    : count == 0    ? "- "
    : maxDepth <= 0 ? "▹ " : "▿ "
```


この種のコードについてはさまざまな意見があるが、この構文が読みづらいと考えるほうが妥当であると多くの人は思うだろう。

別の選択肢は、Swiftの明確な初期化機能を使用すること。

```swift
let bullet: String
if isRoot && (count == 0 || !willExpand) { bullet = "" }
else if count == 0 { bullet = "- " }
else if maxDepth <= 0 { bullet = "▹ " }
else { bullet = "▿ " }
```

この場合は、各分岐で明示的なタイピングと代入の式を追加する儀式を追加する(そしておそらく過度に簡潔な変数名を使いたくなる)だけではなく、型が簡単にわかる場合にのみ有効となる。Opaque型には使用できず、型が複雑にネストされたジェネリックである場合、非常に不便で美しくない。

Swiftにあまり慣れていないプログラマは、この方法を知らない可能性があるため、`var bullet = ""`としたくなるかもしれない。これは、デフォルト値をそのまま使いたくないあらゆる状況で初期化後に必ずオーバーライドされることが保証されないため、バグが発生しやすい。

最後の例として、クロージャを使用して`if`式をシミュレートすることもできる.

```swift
let bullet = {
    if isRoot && (count == 0 || !willExpand) { return "" }
    else if count == 0 { return "- " }
    else if maxDepth <= 0 { return "▹ " }
    else { return "▿ " }
}()
```

これには、`return`に加えて、いくつかのクロージャ式を使った儀式が必要になる。しかし、ここでの`return`はただの儀式的なものではない。外側の関数ではなくクロージャから値を返していることを理解するための認知的負荷が必要となる

このプロポーザルでは、これらの問題をすべて回避する新しい構文が導入されている。

```swift
let bullet =
    if isRoot && (count == 0 || !willExpand) { "" }
    else if count == 0 { "- " }
    else if maxDepth <= 0 { "▹ " }
    else { "▿ " }
```

同様に、`return`を付ける儀式は先ほどの例でもしなくて済むようになる:

```swift
public static func width(_ x: Unicode.Scalar) -> Int {
    switch x.value {
        case 0..<0x80: 1
        case 0x80..<0x0800: 2
        case 0x0800..<0x1_0000: 3
        default: 4
    }
}
```

これらの例は両方とも、[Nate Cook](https://forums.swift.org/t/if-else-expressions/22366/48)と[Michael Ilseman](https://forums.swift.org/t/omitting-returns-in-string-case-study-of-se-0255/24283)による投稿から来ており、この機能によって標準ライブラリコードが大幅に改善される多くの例がドキュメント化されている。

## 詳細

`if`文と`switch`文は、次の目的で式として使用できる:

- 関数、プロパティ、およびクロージャから値を返す (暗黙的または明示的な戻り値のいずれかを使用)。
- 変数への値の代入
- 変数の宣言

もちろん、部分式や関数の引数など、式が現れる場所は他にもたくさんあるが、現時点では対象外で将来の方向性のセクションで議論されている。

`if`または`switch`を式として使用するには、次の基準を満たす必要がある:

**`if`の各分岐、または`switch`の各ケースは、単一の式**

ある分岐が選択された場合、それぞれの式の値が式全体としての値になる。

例えば、単一の式の上にログを出力する行がある場合、既存の(`return`をつける)方法へのフォールバックが必要になるという欠点がある。これは、現在の`return`の省略した際の挙動と一致している。

例外として、分岐が(値を返さずに)リターンするかエラーをスローする場合は、式全体として値を生成する必要がないため、スローまたはリターンの前にその分岐で複数の式が実行される可能性がある。

分岐内では、さらに`if`または`switch`式をネストすることもできる。

**これらの各式は、それぞれの分岐の型チェック時に、同じ型を生成する必要がある**

これには 2 つの利点がある。コンパイラの式の型チェック作業が大幅に簡単になるため、個々の分岐と式全体の両方で簡単に型を推論できる。

あいまいな場合は、より多くの型コンテキストが必要になる。次のコードはコンパイルできない。

```swift
let x = if p { 0 } else { 1.0 }
```

これは、各分岐の型チェックを行うと、`0`は `Int`型で `1.0`は`Double`になるからである。修正方法は各分岐のあいまいさをなくすこと。この場合、`0`を `0.0`に書き換えるか、`0 as Double`のように型のコンテキストを提供する。

下記の例では、各分岐に型のコンテキストを提供することで解決できる場合もある。

```swift
let y: Float = switch x.value {
    case 0..<0x80: 1
    case 0x80..<0x0800: 2.0
    case 0x0800..<0x1_0000: 3.0
    default: 4.5
}
```

この決定は、[SE-0244: Opaque Result Types](https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md) などの他の最近のプロポーザルと一致している。

```swift
// エラー: 関数はOpaque Result Typeの'some Numeric' を宣言しているが、
// 本文内のreturn 文の間で一致する基の型がない
func f() -> some Numeric {
    if Bool.random() {
        return 0
    } else  {
        return 1.0
    }
}
```

このルールに従うと、`nil`リテラルの型を決定するために、宣言の明示的な型のコンテキストが必要になるだろう。

```swift
// ❌
let x = if p { nil } else { 2.0 }
// 型のコンテキストが提供されているので⭕️
let x: Double? = if p { nil } else { 2.0 }
```

もちろん、関数からリターンするとき、または既存の変数に代入するときは、この型のコンテキストが常に提供される。

また、[SE-0326: Enable multi-statement Closure parameter/result type inference](https://github.com/apple/swift-evolution/blob/main/proposals/0326-extending-multi-statement-closure-inference.md)ともこのルールは一致する。

```swift
func test<T>(_: (Int?) -> T) {}

// ❌
test { x in
    guard let x { return nil }
    return x
}

// 型のコンテキストが提供されているので⭕️
test { x -> Int? in
    guard let x { return nil }
    return x
}
```

これは、(`let x = p ? 0 : 1.0`が`x: Double`としてコンパイルされる)三項演算子の動作とは異なる。

この挙動が望ましいとしても、双方向の推論が型チェッカのパフォーマンスに影響するため、現在は実装されていない。これは、分岐が多い場合に特に当てはまる。ただし、この決定は将来再検討される可能性がある。なぜなら、完全な双方向の型推論への切り替えは、理論的にはソースブレークを起こす可能性があるが、おそらく実際にはそうならないためである(どのような例があるか思いつかない)。

また、双方向の推論は、各分岐について個別に推論することを非常に難しくし、予期しない結果につながる場合がある。

```swift
let x = if p {
    [1]
} else {
    [1].lazy.map(expensiveOperation)
}
```

完全な双方向推の論では、`if`　分岐の `Array`によって、`else`分岐の `.lazy.map`が予期せずeagerになってしまう。

このルールの例外の1つは、一部の分岐が `Never`型を生成する場合。他のすべての非 `Never`分岐が同じ型である限りOKとなる。

```swift
// xはInt型、else 分岐の型は気にしない
let x = if .random() {
  1
} else {
  fatalError()
}
```

**`if`文の場合、分岐には `else`を含めなければならない**

このルールは、`if`などを使用した確定初期化(definitive initialization)※および`return`文の現在のルールと一致している。例えば:

※ コンパイラは変数が使用される前に初期化されているということを強制すること

```swift
func f() -> String {
    let b = Bool.random()
    if b == true {
        return "true"
    } else if b == false { // } else { これがあればコンパイルできる
        return "false"
    }
} // 'String'を返すことが期待されるグローバル関数に戻り値がない
```

網羅的な`switch`と同様のロジックが適用された場合、これは将来、全面的(確定初期化、戻り値、および`if`式)に対して再検討される可能性があるが、これは別のプロポーザルで考える。

**式はリザルトビルダ式の一部ではない**

`if`文と`switch`文は、`buildEither`関数を介してリザルトビルダのコンテキストで使用される場合、既に式になっている。このプロポーザルではこの機能を変更しない。

`if`の変数宣言の形式は、リザルトビルダではOK。

**パターンマッチバインディングが、`if`または `case`内で発生する可能性がある**

例えば、次のようにリターンが省略される可能性がある:

```swift
private func balance() -> Tree {
    switch self {
        case let .node(.B, .node(.R, .node(.R, a, x, b), y, c), z, d):
            .node(.R, .node(.B,a,x,b),y,.node(.B,c,z,d))
        case let .node(.B, .node(.R, a, x, .node(.R, b, y, c)), z, d):
            .node(.R, .node(.B,a,x,b),y,.node(.B,c,z,d))
        case let .node(.B, a, x, .node(.R, .node(.R, b, y, c), z, d)):
            .node(.R, .node(.B,a,x,b),y,.node(.B,c,z,d))
        case let .node(.B, a, x, .node(.R, b, y, .node(.R, c, z, d))):
            .node(.R, .node(.B,a,x,b),y,.node(.B,c,z,d))
        default:
            self
    }
}
```

また、Optionalのアンラップに`if let`で使用できる。

```swift
// let x = foo.map(process) ?? someDefaultValue と同じ
let x = if let foo { process(foo) } else { someDefaultValue }
```

## 将来の方向性

このプロポーザルは、最初に示した3つのケースでのみ式を有効にするという狭い範囲で適用している。これは、大多数のユースケースをカバーすることを目的としているためだが、他の多くのユースケースをカバーする拡張機能を今後追加することができる。コミュニティが、現在提案されている機能を実際に使ってみた上で、後のプロポーザルにさらに多くのユースケースが追加される可能性もある。これには、言語のバージョンによってソースブレークが起こるような機能も含まれる。

### 完全式

※ 完全式とは、別の式または宣言子の一部ではない式

この機能が生成できる式の種類は、この機能をパーサに追加する[このコミット](https://github.com/apple/swift/compare/main...hamishknight:express-yourself#diff-7db38bc4b6f7872e5a631989c2925f5fac21199e221aa9112afbbc9aae66a2de)から分かるだろう。

完全式には、ここではプロポーザルされていない、いくつかのかなりお決まりな例が含まれている。

```swift
let x = 1 + if .random() { 3 } else { 4 }
```

しかし、次のようなかなり奇妙なものもある。

```swift
for b in [true] where switch b { case true: true case false: false } {}
```

奇妙な例は、ほとんどの場合「奇妙だが無害」と見なすことができるものの、特にリザルトビルダでは、いくつかのソースブレークを起こすエッジケースがある。

```swift
var body: some View {
    VStack {
        if showButton {
            Button("Click me!", action: clicked)
        } else {
            Text("No button")
        }
        .someStaticProperty
    }
}
```

この場合、式が(現在リザルトビルダでもできない)後置メンバ式を持つことを許可された場合、これを`if`式の修飾子として解析する必要があるのか、新しい式として解析する必要があるのかが、あいまいになる。これはリザルトビルダのみの問題である可能性があるが、パーサにはリザルトビルダの動作を特殊化(specialize)する機能がない。この問題は、現在も発生する可能性がある(これがRegex BuildersにOneが存在する理由)が、このように機能するコードに、新たなあいまいさをもたらす可能性があることに注意。

### `do`式

`do`ブロックも同様に式に変換できる。例えば、次のようになる。

```swift
let foo: String = do {
    try bar()
} catch {
    "Error \(error)"
}
```

### `break`と`continue`のサポート

`return`と同様に、ラベルを中断または継続する文を分岐内で使う場合がある。ここで考慮すべき多くのエッジケースが存在する可能性があり、すべての分岐で変数が初期化されることを保証するために確定初期化の拡張が必要になる可能性がある。

### guard

多くの場合、`guard`でも使用できることへの期待は、`guard`は`if`と同等であるべきという要求とつながる。`guard`の`else`から値を返すことは非常に一般的であり、将来的に次のようなシンタックスシュガーが書ける可能性がある

```swift
guard hasNativeStorage else { nil }
```

これは魅力的だが、`guard`文で`return`を省略できるようにするという、別のプロポーザルで考える。

### 複数行ある分岐

すべての分岐が単一の式でなければならない、という要件は、残念なユーザビリティにつながる。

```swift
let decoded =
  if isFastUTF8 {
    Log("Taking the fast path")
    withFastUTF8 { _decodeScalar($0, startingAt: i) }
  } else
    Log("Running error-correcting slow-path")
    foreignErrorCorrectedScalar(
      startingAt: String.Index(_encodedOffset: i))
  }
```

これは、複数行のクロージャなど他の場合と一致しているが、クロージャから`return`だけが必要な場合と異なり、ユーザがコードを古いメカニズムに戻すリファクタリングをする必要がある。

問題はこれに対する優れた解決策がないこと。Rustなどの他の言語で採用されているアプローチは、スコープの末尾にある式をそのままそのスコープの式の値にする。これは賛否両論で、スタイル上の好みがある。さらに重要なことは、これはSwiftにとってかなり急進的な新しい方向性であり、プロポーザルが出された場合、おそらくそのようなすべてのケース(関数やクロージャの戻り値など)についても検討する必要がある。

あるいは、式の値がこの分岐の値として使用されていることを明示するために、新しいキーワードを導入することもできる(Java は`switch`式でこれに `yield`を使用する)。

### `Either`

前述のように、リザルトビルダでは`if`を使用して、`Either`型を構築できる。これは、分岐内の式が異なる型になる可能性があることを意味する。

これは、リザルトビルダの外部の`if`式でも実行でき、Swiftの強力な新機能となる。ただし、範囲が広く(言語に統合された`Either`型の導入を含む)、別のプロポーザルで検討する必要がある。おそらく、ここでプロポーザルされている、より通常のバージョンにコミュニティが慣れた後になる。

## 代替案

### 現状維持

[一般的に拒否されたプロポーザルのリスト](https://github.com/apple/swift-evolution/blob/main/commonly_proposed.md)には、このプロポーザルの主な内容が含まれている。

> **式としてのif/elseおよびswitch**: これらはサポートすることは概念的に興味深いが、これらを式にすることによって解決される問題の多くは、Swiftの他の方法で既に解決されている。それらを式にすることは大きなトレードオフをもたらし、全体として、これまでのデザインよりも明らかに優れたデザインは見つらなかった。

上記の動機のセクションでは、現在存在する代替案が不十分である理由を概説している。このプロポーザルの範囲が狭い理由の1つは、これらのより困難なトレードオフのいくつかを解決する作業を避けながら、大部分のことへ価値をもたらすこと。

この機能の欠如は、現代プログラミング言語である、というSwiftの[主張](https://www.swift.org/about/)にとって重荷になっている。Swiftは現代プログラミング言語の方針に沿ったものをサポートしていない数少ない現代言語の1つ(Goはもう1つの注目すべき例外)。

### 他の構文

単一の式が戻り値として扱われる現在の暗黙のリターンメカニズムを拡張する代わりに、このプロポーザル提案は `if/switch`式バージョンの新しい構文を導入できる。例えば、

```swift
var response = switch (utterance) {
    case "thank you" -> "you’re welcome";
    case "atchoo" -> "gesundheit";
    case "fire!" -> {
        log.warn("fire detected");
        yield "everybody out!";  // yield = 複数文の分岐の値
    };
    default -> {
		throw new IllegalStateException(utterance)
	}
}
```

同様の提案が[SE-0255: Implicit Returns from Single-Expression Functions](https://forums.swift.org/t/se-0255-implicit-returns-from-single-expression-functions/)でもあった。そこでは、`func sum() -> Element = reduce(0, +)`のような単一式関数の代替構文が議論された。そのときは、コアチームは別の構文を導入するための十分な動機が考えられなかった。

`->`構文を代替として使う主なメリットは、より明示的になることだが、2種類の`switch`構文について事前に知っておく必要があるというトレードオフが伴う。これは、(Javaの例でも示されているような)複数文の分岐の場合に、式の値を明示的に「生成」する方法を提供することと、「最後の式」アプローチを取る、という対立した別の目的に直交しており、問題を解決してないことに注意が必要。

Javaでは、`if`式にこの構文を導入しなかったが、これはSwiftの目標であるため、次のことを暗に意味する:

```swift
let x = 
  if .random() -> 1
  else -> fatalError()
```

ただし、これは複数文の分岐に発展するときに問題を引き起こす。`switch`とは異なり、これらは波括弧(`{}`)を導入する必要があり、`{}`と「これは式だ」という記号(`->`)の両方を置くことになる。

```swift
let x = 
  if .random() -> {
    let y = someComputation()
    y * 2
  } else -> fatalError()
```

JavaやCとは異なり、この「2つ以上の引数に対する括弧」スタイルの`if`は、Swiftには合わない。

式の機能がより多くの種類の文にもたらされた場合、`->`がうまく機能するかどうかも明らかではない。例えば、

```swift
let foo: String = 
  do ->
    try bar()
  catch ns as NSError ->
    "Error \(error)"
```

または、式とリターンを含む混合分岐の場合、

```swift
let x = 
  if .random() -> 1
  else -> return 2
```

将来の方向として完全式が検討される場合、特に単一行の式が必要な場合に、`->`形式はうまく機能しない可能性がある。

```swift
// これは(p ? 1 : 2) + 3
// もしくは p ? 1 : (2 + 3)
let x = if p -> 1 else -> 2 + 3
```

##　ソース互換性

提案されているように、この追加には、到達不能コードに関連するソースの非互換性が1つある。以下は現在コンパイルできるが、`if`文に到達できない(および分岐の値が使用されていない)という警告が表示される。

```swift
func foo() {
  return
  if .random() { 0 } else { 0 }
}
```

しかし、このプロポーザルでは、`if`から`Int`式を返すものとして解析するようになるため、「予期しない非Voidの戻り値がVoid関数に含まれている」というエラーでコンパイルに失敗する。これはさまざまな方法で(`return;`または`return ()`を使用するか、デッドコードを明示的に `#if`で囲むことにより)修正できる。

定数の評価時にコンパイラがデッドコードを無視するようになると、別の同様のケースが発生する可能性がある。

```swift
func foo() -> Int {
  switch true {
  case true:
    return 0
  case false:
    print("unreachable")
  }
}
```

これは現在、`false`のケースが到達不能であることを(おそらくそうすべきだが)警告*しない*が、このプロポーザルが導入された後は、特別な処理を行わないと、`()`と推論され、`Int`型と一致しないという型エラーが発生する。

これらの例はすべてデッドコードなため、言語バージョンでこの変更をゲートしたり、ブレークを回避するために特別な処理を追加したりするのではなく、このソースブレークを受け入れるのが妥当だと思われる。

## ABI stabilityへの影響

なし

## 謝辞

この実装の多くは、[Pavel Yaskevich](https://github.com/xedin)によって行われた土台となる作業、特に[複数文のクロージャ型推論](https://github.com/apple/swift-evolution/blob/main/proposals/0326-extending-multi-statement-closure-inference.md)を可能にするために行われた作業の上に成り立っている。

[Nate Cook](https://forums.swift.org/t/if-else-expressions/22366/48)と[Michael Ilseman](https://forums.swift.org/t/omitting-returns-in-string-case-study-of-se-0255/24283)の両方が、標準ライブラリや他の場所でのユースケースの分析を提供した。また、多くのコミュニティメンバーがこの変更を強く主張しており、最近では[Dave Abrahams](https://forums.swift.org/t/if-else-expressions/22366)が該当する。

## メモ

ここで出てくるDIはDefinitive Initialization(確定初期化)のこと。

Definitive Initialization means the Swift compiler will make sure an instance is fully initialized before the first attempt to read it.

## 参考リンク

### Forums

- [[Pitch] if and switch expressions](https://forums.swift.org/t/pitch-if-and-switch-expressions/61149)
- [SE-0380: `if`and `switch`expressions](https://forums.swift.org/t/se-0380-if-and-switch-expressions/61899)

### プロポーザルドキュメント

- [if and switch expressions](https://github.com/apple/swift-evolution/blob/main/proposals/0380-if-switch-expressions.md)
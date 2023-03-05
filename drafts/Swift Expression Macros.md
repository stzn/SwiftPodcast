# Expression Macros

- [Expression Macros](#expression-macros)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
    - [提案内容](#提案内容)
      - [マクロの引数と戻り値の型チェック](#マクロの引数と戻り値の型チェック)
      - [構文変換](#構文変換)
      - [マクロを別プログラムとして定義](#マクロを別プログラムとして定義)
      - [`stringify`マクロの実装](#stringifyマクロの実装)
    - [詳細](#詳細)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

式マクロは、新しい種類の式でSwiftを展開する方法を提供して、新しいコードを生成するために引数で任意の構文変換を実行することができる。式マクロは、以前は新しい言語機能を導入することでしかできなかった方法でSwiftを展開することを可能にし、開発者がより表現力豊かなライブラリを構築したり、余計な定型文を排除することを支援しする。

## 内容

### 動機

式マクロは、Swiftにマクロを導入するための一般的な動機を示している[マクロのビジョン](https://github.com/apple/swift-evolution/pull/1927)の一部である。特に式は、どこからでも式として呼び出す関数を作成できるため、Swiftがすでに実行時の動作を除くための適切な抽象化を提供している領域である。しかし、`#file`や`#line`のようないくつかのハードコードされた例外を除いて、式はコンパイルされているプログラムのソースコードについて推論したり、修正したりすることはできない。このような使用例では、外部のソース生成ツールが必要になるが、他のツールときれいに統合できることはあまりない。

### 提案内容

このプロポーザルでは、式マクロという概念を導入し、ソースコード中で式として使用され(`#`でマークされ)、式に展開されるようにする。式マクロは、関数と同様にパラメータと戻り値の型を持つことができ、マクロを実際にまず展開する前に、式へのマクロ展開の効果を記述することができる。

実際のマクロ展開は、シンタックスツリーのソース間変換で実装される。つまり、式マクロは、マクロ展開自体のシンタックスツリー(例えば、`#`から始まり最後の引数で終わる)を提供され、それを展開したシンタックスツリーに書き換えることができる。その展開されたシンタックスツリーは、マクロの戻り値の型に対して型チェックが行われる。

簡単な例として、入力として1つの引数を取り、元の引数とその引数のソースコードを含む文字列リテラルの両方を含むタプルを生成する`stringify`マクロを考えてみよう。このマクロは、ソースコードの中で、例えば次のように使用することができる。

```swift
#stringify(x + y)
```

そして次のように展開される。

```swift
(x + y, "x + y")
```

マクロの型のシグネチャは宣言の一部であり、関数とよく似ている。

```swift
@freestanding(expression) macro stringify<T>(_: T) -> (T, String)
```

#### マクロの引数と戻り値の型チェック

マクロの引数は、マクロをインスタンス化する前に、マクロのパラメータ型と照合して型チェックを行う。例えば、マクロ引数`x + y`は型チェックされ、不正な形式(例えば、`x`が`Int`で`y`が`String`)の場合、マクロは決して展開されない。もし正しい形式ならば、ジェネリックパラメータ`T`は`x + y`の結果に推論され、その型はマクロの戻り値の型に引き継がれる。この型チェックモデルには、いくつかの利点がある。

- マクロの実装では、入力として適切に型付けされた引数が保証されるため、マクロに不正なコードが渡されることを心配する必要がない
- ツールは、マクロの引数が他のSwiftコードと同じ規則に従うので、関数のようにマクロを扱うことができ、コード補完やシンタックスハイライトなどのために同じ機能を提供できる
- マクロの展開式は、マクロを展開することなく、部分的に型チェックを行うことができる。これは、型推論中に同じマクロが繰り返し展開されないため、コンパイル時のパフォーマンスを向上させるだけでなく、ツールがマクロ展開を実行しなくても、依然として妥当な結果を得ることができる。
- マクロが展開されると、展開されたシンタックスツリーはマクロの戻り値の型に対して型チェックされる。これは、`#stringify(x + y)`の場合、`x + y`が`Int`型だった場合、展開されたシンタックスツリー`((x + y, "x + y"))`は`(Int, String)`として型検査されることを意味する

マクロ式の型チェックは、呼び出しの型チェックに似ており、型推論情報がマクロの引数から戻り値の型に流れる(その逆も同様)。例えば:

```swift
let (a, b): (Double, String) = #stringify(1 + 2)
```

の場合、整数リテラル`1`、`2`には`Double`型が割り当てられる。


#### 構文変換

マクロ展開は構文操作の一つで、完全なマクロ展開式(例:`#stringify(x + y)`)からなる整ったシンタックスツリーを入力として受け取り、出力としてシンタックスツリーを生成する。出力されたシンタックスツリーは、マクロの戻り値の型に基づいて型チェックされます。

構文変換は、コンパイラの抽象シンタックスツリー(AST)または内部表現(IR)の直接操作など、より構造化されたアプローチよりも多くの利点を持っている。

マクロ展開は、その効果を表現するために完全にSwift言語を使用することができる。文法のある位置にマクロをSwiftソースコードとして書くことができる場合、マクロはそこに展開することができる。
SwiftプログラマはSwiftソースコードを理解しているので、ソースコードに適用したときのマクロの出力について推論することができる。これは、マクロを作成するときと使用するときの両方に役立つ。
マクロを使用するソースコードは、例えば理由付けやデバッグを容易にするため、またはマクロをサポートしない古いSwiftコンパイラで動作するように、「展開」してマクロの使用を排除することができる。
後方互換性の懸念からコンパイラの進化と改善の能力を制限することになるため、コンパイラのASTとIRはクライアントに公開する必要はない。
一方、純粋な構文変換にも、いくつかの欠点がある。

- ソースコードを文字列として扱うことになるので、マクロの実装に(例えば)構文エラーや型エラーが発生しやすく、構文的なマクロ展開はコンパイル時に失敗しやすい。
- 構文マクロの展開は、再分析と再型チェックを行うため、(例えば)ASTやIRを直接操作するアプローチよりも、コンパイル時のオーバーヘッドが大きくなる
- 構文マクロは「衛生的」でない。つまり、マクロ展開の処理方法が展開される環境に依存し、その環境に影響を与える可能性がある

今回の提案のマクロの設計は、これらの問題を軽減することを試みているが、構文でマクロを使用する上で結構根本的な問題である。バランス的には、構文マクロの使いやすさと解釈のしやすさは、これらの問題を凌駕している。

#### マクロを別プログラムとして定義

マクロの定義はシンタックスツリーに対して操作する。大まかに言って、マクロの展開操作を定義する方法は2種類ある:

- *宣言的な変換のセット*: これは、マクロの入力が与えられたときにマクロがどのように展開されるかを定義できる特別な構文で言語を拡張し、コンパイラがそれぞれのマクロ展開に対してそのルールを適用するもの。Cプリプロセッサはこのシンプルな形式を採用しているが、Racketの[パターンベースマクロ](https://docs.racket-lang.org/guide/pattern-macros.html)やRustの[宣言型マクロ](https://doc.rust-lang.org/book/ch19-06-macros.html#declarative-macros-with-macro_rules-for-general-metaprogramming)は、マクロの引数をパターンにマッチさせ、マクロに記述された新しい構文への書き換えを実行する、より高度なルールを提供している。Swiftがこのアプローチを導入するためには、シンタックスツリーをマッチングして書き換えるためのパターン言語を発明する必要がありそうだ。
- *ソースを変換する実行可能なプログラム*: これは、そのプログラムのシンタックスを直接操作するプログラムを実行することを含む。プログラムがどのように実行されるかは、環境に大きく依存する。[Scala 3マクロ](https://docs.scala-lang.org/scala3/guides/macros/macros.html)、JVMを使用することでターゲットコード(生成されるプログラム)とホストコード(コンパイラが動作している場所)を結びつけることができるというメリットがあるが、Rustの[手続き型マクロ](https://doc.rust-lang.org/reference/procedural-macros.html)は、コンパイラが対話する別の木箱に組み込まれている。Swiftがこのアプローチを導入するためには、コンパイラにSwiftコードのための完全なインタプリタを構築するか、Rustと同様のアプローチを取り、コンパイラが対話できる別のプログラムとしてマクロ定義を構築する必要がある。

私たちは後者のアプローチを提案し、マクロ定義は [swift-syntax](https://github.com/apple/swift-syntax/)パッケージを使用して Swiftの構文木を操作する別のプログラムである。式マクロは、`ExpressionMacro`プロトコルに準拠した型として定義される。

```swift
public protocol ExpressionMacro: FreestandingMacro {
    /// 与えられた文脈の中で、与えられた自立型マクロ展開で記述されたマクロを展開し、置換式を生成する。
    static func expansion(
        of node: some FreestandingMacroExpansionSyntax,
        in context: some MacroExpansionContext
    ) async throws -> ExprSyntax
}
```

`expansion(of:in:)`メソッドは、マクロ展開式の構文ノード(例：`#stringify(x + y)`)と、マクロが展開されるコンパイルコンテキストに関する詳細情報を提供する「コンテキスト」を引数として受け取る。そして、変換された構文木を含むマクロの結果を生成する。

`Macro`、`ExpressionMacro`、`MacroExpansionContext`の具体的な内容は、詳細に続く。

#### `stringify`マクロの実装

続けて、`stringify`マクロの実装を説明する。`ExpressionMacro`に準拠した新しい`StringifyMacro`型である。

```swift
import SwiftSyntax
import SwiftSyntaxBuilder
import _SwiftSyntaxMacros

public struct StringifyMacro: ExpressionMacro {
    public static func expansion(
        of node: some FreestandingMacroExpansionSyntax,
        in context: some MacroExpansionContext
    ) -> ExprSyntax {
        guard let argument = node.argumentList.first?.expression else {
            fatalError("compiler bug: the macro does not have any arguments")
        }

        return "(\(argument), \(literal: argument.description))"
    }
}
```

`stringify`マクロが比較的単純であるため、`expansion(of:in:)`関数はかなり小さい。この関数は、構文木からマクロ引数(`#stringify(x + y)`の`x + y`)を抽出し、元の引数を値として、さらにソースコードとして[文字列リテラル](https://github.com/apple/swift-syntax/blob/main/Sources/SwiftSyntaxBuilder/ConvenienceInitializers.swift#L259-L265)で補間して、戻り値のタプル式を形成する。この文字列は、式として解析され(`ExprSyntax`ノードを生成)、マクロ展開の戻り値として返される。これは`SwiftSyntaxBuilder`モジュールによって提供される準引用の単純な形式で、主要な構文ノード(この場合、式のための`ExprSyntax`)を[ExpressibleByStringInterpolation](https://developer.apple.com/documentation/swift/expressiblebystringinterpolation)に準拠させることによって実装されており、既存の構文ノードは、展開したSwiftコードを含む文字列リテラルに補間することができる。

`StringifyMacro`構造体は、先ほど宣言された`stringify`マクロのための実装である。私たちは、何らかのメカニズムによって、ソースコードでこれらを結びつける必要がある。そこで、`=`に続くマクロ宣言内でモジュールと`ExpressionMacro`の型名を命名する組み込みマクロを提供することを提案する。

```swift
@freestanding(expression)
macro stringify<T>(_: T) -> (T, String) =
  #externalMacro(module: "ExampleMacros", type: "StringifyMacro")
```

### 詳細

TBD


## 参考リンク

### Forums

- [A Possible Vision for Macros in Swift](https://forums.swift.org/t/a-possible-vision-for-macros-in-swift/60900)
- [[Pitch] Expression macros](https://forums.swift.org/t/pitch-expression-macros/61499)
- [SE-0382: Expression Macros](https://forums.swift.org/t/se-0382-expression-macros/62090)
- [[Accepted] SE-0382: Expression Macros](https://forums.swift.org/t/accepted-se-0382-expression-macros/63495)

### プロポーザルドキュメント

[Expression Macros](https://github.com/apple/swift-evolution/blob/main/proposals/0382-expression-macros.md)
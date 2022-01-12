# Swift 新しい宣言的な文字列処理APIとより大きなビジョン

- [Swift 新しい宣言的な文字列処理APIとより大きなビジョン](#swift-新しい宣言的な文字列処理apiとより大きなビジョン)
  - [概要](#概要)
  - [内容](#内容)
    - [用語説明](#用語説明)
    - [Swiftの文字列に関わるクラス](#swiftの文字列に関わるクラス)
    - [問題点](#問題点)
      - [既存のAPIを使った場合](#既存のapiを使った場合)
        - [問題点](#問題点-1)
      - [コンシューマパターン](#コンシューマパターン)
        - [改善点](#改善点)
        - [問題点](#問題点-2)
    - [提案内容](#提案内容)
      - [Regexリテラル](#regexリテラル)
        - [Swiftに正規表現を導入するメリット](#swiftに正規表現を導入するメリット)
          - [補足](#補足)
      - [Pattern result builder](#pattern-result-builder)
      - [PatternとRegexの相互補間](#patternとregexの相互補間)
      - [`Collection`アルゴリズム](#collectionアルゴリズム)
    - [将来的な話](#将来的な話)
    - [文字列処理を超えたデータ処理に汎用的なデザインの検討](#文字列処理を超えたデータ処理に汎用的なデザインの検討)
  - [参考リンク](#参考リンク)

## 概要

文字列処理は難しく、Standard Libraryによって提供される現在のAPIでは価値を十分に提供できていない。

そこで高速に簡単にするために、宣言的な文字列処理のAPIをSwiftやStandard Libraryから提供したい。

※ 具体的な詳細はまだ議論中。ここでは全体的な目指す方向を紹介したい。

関連PR: https://forums.swift.org/t/declarative-string-processing-overview/52459

## 内容

### 用語説明

- Unicode: 全世界共通で使えるように、世界中の文字を収録する文字コード規格である。Swiftおよび他のほとんどのプログラミング言語がこの規格に従って文字列を扱っている。つまりこの規格に合わないものは文字とみなされない。
- コードポイント: Unicodeに規定されている文字の一つ一つに割り振られている0から0x10FFFFまでの符号。
- 書記素クラスタ: ユーザが文字として認識するもの。一つのUnicodeのコードポイントが割り当てられているのがほとんどだが、2つ以上で1つの書記素クラスタになることもある(これは拡張書記素クラスタと呼ばれる)。
- Unicodeスカラ値: サロゲートコードポイント以外のすべてのコードポイント。文字が割り当てられているか、将来文字が割り当てられる可能性があるコード ポイント。
- サロゲートコードポイント:0xD800から0xDFFFのコードポイント空間を指し，この空間の値には文字は割り当てられない。

### Swiftの文字列に関わるクラス

- Character: 一つの拡張書記素クラスタ
- String: Characterのコレクション
- Substring: Stringの一部分
- Unicode.Scalar: Unicodeスカラ値 

関連ドキュメント: https://developer.apple.com/documentation/swift/swift_standard_library/strings_and_text

### 問題点

Forumの例を見てみる。

Swiftの`String`は、Unicode data tableと呼ばれる、CSVのように一定の規則に従って書かれているドキュメントのパース処理が事前に必要になっている。
関連リンク: http://www.unicode.org/Public/13.0.0/ucd/auxiliary/GraphemeBreakProperty.txt

#### 既存のAPIを使った場合

このデータをパースする際、既存のAPIだと、汎用的な`Collection`の`split`や`map`や`filter`を使う。

```swift
extension Unicode.Scalar {
  // Try to convert a hexadecimal string to a scalar
  init?<S: StringProtocol>(hex: S) {
    guard let val = UInt32(hex, radix: 16), let scalar = Self(val) else {
      return nil
    }
    self = scalar
  }
}

func graphemeBreakPropertyData(
  forLine line: String
) -> (scalars: ClosedRange<Unicode.Scalar>, property: Unicode.GraphemeBreakProperty)? {
  let components = line.split(separator: ";")
  guard components.count >= 2 else { return nil }

  let splitProperty = components[1].split(separator: "#")
  let filteredProperty = splitProperty[0].filter { !$0.isWhitespace }
  guard let property = Unicode.GraphemeBreakProperty(filteredProperty) else {
    return nil
  }

  let scalars: ClosedRange<Unicode.Scalar>
  let filteredScalars = components[0].filter { !$0.isWhitespace }
  if filteredScalars.contains(".") {
    let range = filteredScalars
      .split(separator: ".")
      .map { Unicode.Scalar(hex: $0)! }
    scalars = range[0] ... range[1]
  } else {
    let scalar = Unicode.Scalar(hex: filteredScalars)!
    scalars = scalar ... scalar
  }
  return (scalars, property)
}
```

##### 問題点

- 読みづらいしい、何度も見返さないと理解できない。
- `index`をハードコードで使っている。force unwrappedもしている。こうするとformatの変更に弱くなっている。
- 入力に対して複数のパスが実行され、プロセスに複数の一時データをメモリに割り当てられる

#### コンシューマパターン

`Collection`のAPIを拡張することで、入力の1回のパスで関連情報を抽出できるようにしてみる。

関連実装: https://github.com/apple/swift-evolution-staging/blob/976ea3a81813f06ec11f00550d4e83f340cf2f7e/Sources/CollectionConsumerSearcher/Eat.swift

```swift
// ... consumer helpers like `eat(exactly:)`, `eat(while:)`, and `peek()` ...

// Try to parse a Unicode scalar off the input
private func parseScalar(_ str: inout Substring) -> Unicode.Scalar? {
  let val = str.eat(while: { $0.isHexDigit })
  guard !val.isEmpty else { return nil }

  // Subtle potential bug: if this init fails, we need to restore
  // str.startIndex. Because of how this is currently called, the bug won't
  // manifest now, but could if the call site is changed.
  return Unicode.Scalar(hex: val)
}

func graphemeBreakPropertyData(
  forLine line: String
) -> (scalars: ClosedRange<Unicode.Scalar>, property: Unicode.GraphemeBreakProperty)? {
  var line = line[...]
  guard let lower = parseScalar(&line) else {
    // Comment or whitespace line
    return nil
  }

  let upper: Unicode.Scalar
  if line.peek(".") {
    guard !line.eat(exactly: "..").isEmpty else {
      fatalError("Parse error: invalid scalar range")
    }
    guard let s = parseScalar(&line) else {
      fatalError("Parse error: expected scalar upperbound")
    }
    upper = s
  } else {
    upper = lower
  }

  line.eat(while: { !$0.isLetter })
  let name = line.eat(while: { $0.isLetter || $0 == "_" })
  guard let prop = Unicode.GraphemeBreakProperty(name) else {
    return nil
  }

  return (lower ... upper, prop)
}
```

##### 改善点

- 左から右にファイルを読めるようになる
- 中間のメモリ割り当てが不要
- 失敗が明示的(force unwrappedなどしない)

##### 問題点

とてもローレベルのコードで扱いに注意が必要。例えば後方参照は手動で処理する必要があるし、パフォーマンスにも影響がある。

### 提案内容

そこでより扱いやすく、保守しやすく、スケーラブルな宣言的APIを提供する。

#### Regexリテラル

TODO: ここは見直しが入って大きく変更される可能性が高いので更新が必要  
https://github.com/apple/swift-experimental-string-processing/issues/63

こういった文字列のパターンマッチやデータを抽出する一般的な方法として正規表現がある。そこで正規表現をパースして静的な型として扱う機能をSwiftに追加する。

Swiftの正規表現ではPCREシンタックス(※)をサポートする。

※ PCRE（Perl Compatible Regular Expressions)
関連リンク: http://pcre.org/current/doc/html/pcre2syntax.html

正規表現はコンパイラによって`Regex`構造体へと推論される。推論できない場合はコンパイルエラー。

文字列の`firstMatch`メソッドにを特定の正規表現を渡してパースする。文字列の中で最初にマッチした`Substring`を返却する。マッチしない場合は`nil`。

キャプチャグループが存在する場合は、一つのグループを一つのの塊として、結果はマッチした文字列全体の`Substring`と、キャプチャグループにマッチした値(Pitchでは仮で`Substring`)のTupleになる。この戻り値の構造は、これまでの正規表現の後方参照の方法が0ではなく1から始まるのに合わせている。Swiftの正規表現の主要な目的の一つに他の正規表現パースツールとの親和性を挙げている。

※ PitchではSubstringで取り急ぎ書かれているが、実際は`Substring`に限られない。戻り値は渡した`RegexProtocol.Match`型

```swift
extension String {
    public func firstMatch<R: RegexProtocol>(of regex: R) -> R.Match?
}
```

使い方としては下記のようになる

```swift
func graphemeBreakPropertyData(
  forLine line: String
) -> (scalars: ClosedRange<Unicode.Scalar>, property: Unicode.GraphemeBreakProperty)? {
  line
    .match(/([0-9A-F]+)(?:\.\.([0-9A-F]+))?\s*;\s(\w+).*/)?
    .captures.flatMap { (l, u, p) in
      guard let property = Unicode.GraphemeBreakProperty(p) else {
        return nil
      }
      let scalars = Unicode.Scalar(hex: l)! ... Unicode.Scalar(hex: u ?? l)!
      return (scalars, property)
    }
}
```

```swift
let regex = /ab(cd*)(ef)gh/
// => Regex<(Substring, Substring, Substring)>
if let match = "abcddddefgh".firstMatch(of: regex) {
  print(match) // => ("abcddddefgh", "cdddd", "ef")
}

let line = "007F..009F    ; Control # Cc  [33] <control-007F>..<control-009F>"
let scalarRangePattern = /([0-9a-fA-F]+)(?:\.\.([0-9a-fA-F]+))?/
// Positions in result: 0 ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
//                      1 ^~~~~~~~~~~~~~     2 ^~~~~~~~~~~~~~
if let match = line.firstMatch(of: scalarRangePattern) {
    print((match.0, match.1, match.2)) // => ("007F..009F", "007F", "009F")
}
```

##### Swiftに正規表現を導入するメリット

- 型としてキャプチャできるので、マッチ後の処理もより使いやすく安全に使用できる
- 他の正規表現に活用していたさまざまな種類の知識やパターンをSwiftで再利用できる

さらに、`ExpressibleByRegexLiteral`と`RegexLiteralProtocol`というプロトコルを用意して、正規表現リテラルのパースをカスタマイズできるようにもする予定(デフォルト実装はStandard Libraryで用意)

`String Interpolation`に似た仕組み  
関連ドキュメント: https://github.com/apple/swift-evolution/blob/master/proposals/0228-fix-expressiblebystringinterpolation.md

関連スレッド: https://forums.swift.org/t/pitch-regular-expression-literals/52820
関連ドキュメント: https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/Evolution/StronglyTypedCaptures.md

###### 補足

これまでのSwiftで正規表現を扱う場合は、`NSRegularExpression`というFoundationのクラスを使っていた。  
関連ドキュメント: https://developer.apple.com/documentation/foundation/nsregularexpression

これはICUというライブラリを使用していたが、パフォーマンスが良くなく依存関係があるので問題視されていた。

Swiftでは、最近これを元のUnicodeのデータを利用するように変更した。  

関連リンク: 
https://unicode-org.github.io/icu/  
http://www.unicode.org/L2/L2001/01339-grapheme-break.htm  

関連実装: https://github.com/apple/swift/pull/37864  

#### Pattern result builder

正規表現は、Unixのコマンドライン引数とテキストエディタの検索フィールドで使用するために作成され、非常に簡潔で、プログラム全体を1行で表すことができる。そのため、より汎用的な目的のプログラミングに使うにはいくつかの不都合がある。

- シンタックスが覚えづらい(\wは"word"なのか"whitespace”なのか?)、読みづらい(..や;はメタ文字で見えづらくなっている)。
- ライブラリのサポートがないので車輪の再発明が起こりやすい。洗練されたパーサーを持つ強力なライブラリがすでに存在する場合、単純な正規表現は、日付や通貨などの一見複雑な形式を解析するためによく間違われる。
- 正規表現は、強力すぎて他と組み合わせて使えない一方で、再帰的な構造を認識するほど強力ではないというやっかいな中間点にある。

Swiftは、簡潔さよりも明快さを重視している。 正規表現は単純なマッチングには最適だが、複雑さが増すにつれて、Swiftとそのライブラリの全機能を発揮できるようにしたい。そこでより多機能な`Pattern`を宣言するためのresult builderを提案する。

```swift
func graphemeBreakPropertyData(
  forLine line: String
) -> (scalars: ClosedRange<Unicode.Scalar>, property: Unicode.GraphemeBreakProperty)? {
  line.match {
    OneOrMore(.hexDigit).capture { Unicode.Scalar(hex: $0) }

    Optionally {
      ".."
      OneOrMore(.hexDigit).capture { Unicode.Scalar(hex: $0) }
    }

    OneOrMore(.whitespace)
    ";"
    OneOrMore(.whitespace)

    OneOrMore(.word).capture(GraphemeBreakProperty.init)

    Repeat(.anyCharacter)
  }?.captures.map { (lower, upper, property) in
    let scalars = lower ... (upper ?? lower)
    return (scalars, property)
  }
}
```

改善点

- 文字クラスや数量詞はなくなり、もっと読みやすくなった。コード補間で見つけやすくもなる。
- 文字列リテラルによって句読点のマッチングがシンプルかつ明確に
- キャプチャグループをインラインで処理できる。ローカルで完結でき、強い片付けなどの改善がされている。  
`Pattern<(Unicode.Scalar, Unicode.Scalar?, GraphemeBreakProperty)>` vs. `Regex<(Substring, Substring?, Substring)>`
※ただし、result builderのグループから`Pattern`の型を推論するには言語の改善が必要
- 失敗した際に早期リターンが可能(これまでの後処理でハンドリングするよりも、余計な処理が減る)

これらは全て通常のSwiftコードで書くことができため、親しんだ方法で扱うことができる。

例えば、`GraphemeBreakProperty`は`enum`のイニシャライザで処理できる。

```swift
enum GraphemeBreakProperty: UInt32 {
  case control = 0
  case extend = 1
  case prepend = 2
  case spacingMark = 3
  case extendedPictographic = 4

  init?(_ str: String) {
    switch str {
    case "Extend":
      self = .extend
    case "Control", "CR", "LF":
      self = .control
    case "Prepend":
      self = .prepend
    case "SpacingMark":
      self = .spacingMark
    case "Extended_Pictographic":
      self = .extendedPictographic
    default:
      return nil
    }
  }
}
```

Patternは既存のFoundationの`Formatter`との互換も備えている。

#### PatternとRegexの相互補間

`Pattern`は通常のSwiftコードのように幅広い用途で使用でき、`Regex`よりも複雑な言語にマッチできるようにサポートできる。一方で、これまでの昔から使われている正規表現構文との親和性や普遍性はない。また、`Regex`リテラルは次に紹介する`Collection`アルゴリズムなどのAPIと連結しやすい。

そこで`Pattern`と`Regex`は次のように相互補間することができると考える

- 既製のパーサの強力なライブラリと同様に、`Pattern`ビルダの中で`Regex`を使えるようにする。これにより、開発者は必要に応じて簡潔な表現(`Regex`)か、より強力で汎用的な構築(`Pattern`)の間を微調整できる。
- `Regex`リテラルを`Pattern`に変換するリファクタリングアクションを追加する。これにより、`Regex`リテラルを使用した高速なプロトタイピングが可能にして、さらにより保守性が高く強力なものへの簡単に変換することが可能になる。

#### `Collection`アルゴリズム

`Collection`に新しいconsumerやsearcherアルゴリズムを追加する。こうすることで`Regex`リテラルが使いやすくなる。

関連実装: https://github.com/milseman/swift-evolution/tree/collections_om_nom_nom
関連スレッド: https://forums.swift.org/t/prototype-protocol-powered-generic-trimming-searching-splitting/29415

例えば、`contains`メソッドは要素の一つしかチェックできない。

```swift
let str = "Hello, World!"
str.contains("Hello") // ❌　error: String型はString.Element(aka 'Character')に変換できない
```

これを下記ができるようにする。

```swift
// The below are all equivalent
str.contains("Hello") || str.contains("Goodbye")
str.contains(/Hello|Goodbye/)
str.contains {
  Alternation {
    "Hello"
    "Goodbye"
  }
}
```

下記のものを追加予定

- `firstRange(of:)`, `lastRange(of:)`, `allRanges(of:)`, `contains(_:)`
- `split(separator:)`
- `trim(_:)`, `trimPrefix(_:)`, `trimSuffix(_:)`
- `replaceAll(_:with:)`, `removeAll(_:)`, `moveAll(_:to:)`
- `match(_:)`, `allMatches(_:)`

### 将来的な話

正規表現パターンを`case`で一致させrて、それらのキャプチャを変数に直接バインドできるようにしたい。

```swift
func parseField(_ field: String) -> ParsedField {
  switch field {
  case let text <- /#\s?(.*)/:
    return .comment(text)
  case let (l, u) <- /([0-9A-F]+)(?:\.\.([0-9A-F]+))?/:
    return .scalars(Unicode.Scalar(hex: l) ... Unicode.Scalar(hex: u ?? l))
  case let prop <- GraphemeBreakProperty.init:
    return .property(prop)
  }
}
```

### 文字列処理を超えたデータ処理に汎用的なデザインの検討

文字列処理はもっと大きく見ればデータ処理の一種。

非同期なイベント処理など他のデータ処理も含めたより汎用的で便利なデザインを考えたい。これはSource stabilityやABI stabilityに関わるので将来にも大きく影響する。

関連ドキュメント: https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/BigPicture.md

## 参考リンク

- https://forums.swift.org/t/declarative-string-processing-overview/52459
- https://developer.apple.com/documentation/swift/swift_standard_library/strings_and_text
- http://www.unicode.org/Public/13.0.0/ucd/auxiliary/GraphemeBreakProperty.txt
- https://github.com/apple/swift-evolution-staging/blob/976ea3a81813f06ec11f00550d4e83f340cf2f7e/Sources/CollectionConsumerSearcher/Eat.swift
- http://pcre.org/current/doc/html/pcre2syntax.html
- https://github.com/apple/swift-evolution/blob/master/proposals/0228-fix-expressiblebystringinterpolation.md
- https://forums.swift.org/t/pitch-regular-expression-literals/52820
- https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/Evolution/StronglyTypedCaptures.md
- https://developer.apple.com/documentation/foundation/nsregularexpression
- https://github.com/apple/swift/pull/37864
- https://github.com/milseman/swift-evolution/tree/collections_om_nom_nom
- https://forums.swift.org/t/prototype-protocol-powered-generic-trimming-searching-splitting/29415
- https://github.com/apple/swift-experimental-string-processing/blob/main/Documentation/BigPicture.md

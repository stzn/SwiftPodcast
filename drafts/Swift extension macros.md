
# Swift extension macros

- [Swift extension macros](#swift-extension-macros)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
  - [提案された内容](#提案された内容)
    - [詳細](#詳細)
      - [マクロで導入されたプロトコル適合とメンバ名の指定](#マクロで導入されたプロトコル適合とメンバ名の指定)
      - [`extension`マクロの適用](#extensionマクロの適用)
      - [`extension`マクロの実装](#extensionマクロの実装)
      - [冗長な適合の抑制](#冗長な適合の抑制)
  - [ソース互換性](#ソース互換性)
  - [ABI互換性](#abi互換性)
  - [適用の影響](#適用の影響)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

この提案は、`conformance`マクロの役割を`extension`マクロの役割として一般化するもので、プロトコルと`where`句に加えて、メンバリストをextensionに追加することができる。


## 内容

### 動機

[SE-0389: Attached Macros](https://github.com/apple/swift-evolution/blob/main/proposals/0389-attached-macros.md)では、マクロが付属されている型のextensionで記述された`where`句を伴って適合したコードに展開される付属型マクロ(attached macros)が導入された:

```swift
@attached(conformance)
macro AddEquatable() = #externalMacro(...)

@AddEquatable
struct S {}

// expands to
extension S: Equatable {}
```

しかし、`conformance`マクロの役割は、それだけでは極めて限定的である。`conformance`マクロはプロトコル名と`where`句の構文を返す機能しか持っていない。プロトコルの要件がメンバを必要とする場合（ほとんどのプロトコルの要件がそうであるように）、それらは別の`member`マクロの役割を通して追加しなければならない。

さらに重要なことは、`conformance`マクロは、マクロがアノテーションされた型のextensionに展開するための唯一の方法であるということである。主要な宣言ではなく型のextensionでメンバを追加できないことは、マクロシステムの重大な制限である。なぜなら、extensionには次のような重要な意味合いがあるからである(ただし、これらに限定されない):

- プロトコルは、extension内でしか要件のデフォルト実装を提供できない
- 型のextensionで追加されたイニシャライザは、コンパイラが合成するイニシャライザを抑制しない
- プロトコルやクラスのextension内の計算プロパティやメソッドは、動的ディスパッチに参加しない

extension機能にはスタイル上の利点もある。extension内のコードは、すべてのメソッドに対するジェネリック要件を繰り返すのではなく、extension自体のジェネリック要件を共有する。

## 提案された内容

この提案では、`conformance`マクロの役割が削除され、`extension`マクロの役割が追加される。`extension`マクロの役割は、`@attached`マクロ属性で使用することができ、マクロがアタッチされる型のextensionでのプロトコルへの適合、`where`句、メンバリストを追加することができる:

```swift
protocol MyProtocol {
  func requirement()
}

@attached(extension, conformances: MyProtocol, names: named(requirement))
macro MyProtocol = #externalMacro(...)

@MyProtocol
struct S<T> {}

// 展開される

extension S: MyProtocol where T: MyProtocol {
  func requirement() { ... }
}
```

生成されるマクロのextensionは、マクロがアタッチされた型のみを拡張しなければならない。また適合や追加するメンバも、`@attached(extension)`属性で前もって指定しなければならない。

### 詳細

#### マクロで導入されたプロトコル適合とメンバ名の指定

SE-0389 は、マクロが他の Swift コードから見える宣言を生成するときは、常に事前に名前を宣言する必要があると述べている。このルールは、拡張マクロにも適用される。つまり下記を指定しなければならない:

- `named`、`prefixed`、`suffixed`、`arbitrary`を使用して指定できるextension内部の宣言。
- extension機能で適合句に記載されるプロトコルの名前。これらのプロトコルは`@attached(conformances:)`属性の`conformances:`リストに指定される。このリストに現れるそれぞれの名前は、適合制約でなければななければならない。適合制約とは:
- プロトコル名
- その基礎となる型が適合制約であるt`typealias`
- エントリがそれぞれ適合制約であるプロトコルのコンポジション

`attached(extension)`に列挙された生成された適合と名前には、以下の制限が適用される:

- `extension`マクロは、`@attached(extension, conformances:)`の`conformances:`リストでカバーされていない適合をプロトコルに追加できない。
- `extension`マクロは、`@attached(extension, names:)`の`names:`リストでカバーされていないメンバーを追加できない。
- `peer`マクロで生成された名前は元の`@attached(extension)`属性ではカバーされないため、`extension`マクロは`peer`マクロがアタッチされたextension機能を導入できない。

#### `extension`マクロの適用

`extension`マクロは名前付型のプライマリ宣言にのみアタッチすることができる。

Swiftはファイルのトップレベルでのみextension宣言を許可する。にもかかわらず、`extension`マクロはネストされた型に適用することができる:

```swift
@attached(extension, conformances: MyProtocol, names: named(requirement))
macro MyProtocol = #externalMacro(...)

struct Outer {
  @MyProtocol
  struct Inner {}
}
```

この状況では、拡張子を含むマクロ展開は、マクロが呼び出された直後ではなく、ファイルのトップレベルに挿入される。上記のコードは次のように展開される:

```swift
struct Outer {
  struct Inner {}
}

extension Outer.Inner: MyProtocol {
  func requirement() { ... }
}
```

Swiftではローカル型にextensionを書く方法がないので、ローカル型に`extension`マクロを適用しようとしてもエラーになる:

```swift
func test() {
  @MyProtocol // error
  struct Local {}
}
```

#### `extension`マクロの実装

`extension`マクロの実装は、`ExtensionMacro`プロトコルに準拠しなければならない:

```swift
/// Describes a macro that can add extensions of the declaration it's
/// attached to.
public protocol ExtensionMacro: AttachedMacro {
  /// Expand an attached extension macro to produce the contents that will 
  /// create a set of extensions.
  ///
  /// - Parameters:
  ///   - node: The custom attribute describing the attached macro.
  ///   - declaration: The declaration the macro attribute is attached to.
  ///   - type: The type to provide extensions of.
  ///   - protocols: The list of protocols to add conformances to. These will
  ///     always be protocols that `type` does not already state a conformance
  ///     to.
  ///   - context: The context in which to perform the macro expansion.
  ///
  /// - Returns: the set of extension declarations introduced by the macro,
  ///   which are always inserted at top-level scope. Each extension must extend
  ///   the `type` parameter.
  static func expansion(
    of node: AttributeSyntax,
    attachedTo declaration: some DeclGroupSyntax,
    providingExtensionsOf type: some TypeSyntaxProtocol,
    conformingTo protocols: [TypeSyntax],
    in context: some MacroExpansionContext
  ) throws -> [ExtensionDeclSyntax]
}
```

結果の配列に含まれる各`ExtensionDeclSyntax`は、`providingExtensionsOf` パラメータを拡張された型として使用する必要がある。例えば、以下のコードでは:

```swift
struct Outer {
  @MyProtocol
  struct Inner {}
}
```

`provideExtensionsOf`の`ExtensionMacro.expansion`に渡される型構文は`Outer.Inner`である。

#### 冗長な適合の抑制

`ExtensionMacro.expansion`の`conformingTo:`パラメータは、`extension`マクロが、元のソースコードにすでに記述されている適合の生成を抑制できる。`conformingTo:`引数の配列には、`@attached(extension conformances:)`の`conformances:`リストから、暗黙の適合やクラスの継承を含め、元のソースコードで型がすでに適合していないプロトコルのみが含まれる。

例えば、次のような`extension`マクロを含むコードを考えてみる:

```swift
protocol Encodable {}
protocol Decodable {}

typealias Codable = Encodable & Decodable

@attached(extension, conformances: Codable)
macro MyMacro() = #externalMacro(...)

@MyMacro
struct S { ... }

extension S: Encodable { ... }
```

`extension`マクロは、`Codable`(`Encodable & Decodable`)への適合を追加できる。構造体`S`は元のソースですでに `Encodable`に適合しているため、`ExtensionMacro.expansion`メソッドは `conformingTo:`` パラメータに`[TypeSyntax(Encodable)]`という引数を受け取る。この情報を使用して、マクロ実装は、`Decodable`に適合する拡張機能のみを追加できる。

## ソース互換性

この提案は、Swift5.9で受け入れられ実装された SE-0389から、`conformance`マクロの役割を削除する。この提案が5.9以降に受け入れられる場合、`conformance`マクロの役割は、適合のみを追加する`extension`マクロのためのシンタックスシュガーとして言語に残る。

## ABI互換性

`extension`マクロは、コンパイル時に通常のSwiftコードに展開され、ABIに影響を与えない。

## 適用の影響

`extension`マクロを使用することによる影響は、プロジェクトでextensionにコードを手動で記述する場合と同じである。

## 参考リンク

### Forums

- [[Pitch] Generalize `conformance` macros as `extension` macros](https://forums.swift.org/t/pitch-generalize-conformance-macros-as-extension-macros/65653)
- [SE-0402: Generalize `conformance` macros as `extension` macros](https://forums.swift.org/t/se-0402-generalize-conformance-macros-as-extension-macros/65965)
- [[Accepted] SE-0402: Generalize `conformance` macros as `extension` macros](https://forums.swift.org/t/accepted-se-0402-generalize-conformance-macros-as-extension-macros/66276)

### プロポーザルドキュメント

- [Generalize conformance macros as extension macros](https://github.com/apple/swift-evolution/blob/main/proposals/0402-extension-macros.md)
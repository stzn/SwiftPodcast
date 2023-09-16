# Swift Member Macro Conformance

- [Swift Member Macro Conformance](#swift-member-macro-conformance)
  - [概要](#概要)
  - [内容](#内容)
    - [提案内容](#提案内容)
    - [詳細](#詳細)
    - [ソース互換性](#ソース互換性)
    - [ABI互換性](#abi互換性)
    - [導入の影響](#導入の影響)
  - [検討された代替案](#検討された代替案)
    - [主要な型定義に影響を与える拡張](#主要な型定義に影響を与える拡張)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

[SE-0402](https://github.com/apple/swift-evolution/blob/main/proposals/0402-extension-macros.md)でのconformanceマクロからextensionマクロへの移行には、extensionマクロが、その型がすでにどのプロトコルに準拠しているか(例えば、スーパークラスが準拠しているとか、どこかに明示的な準拠性が記述されているとか)を知る機能が含まれていた。また、新しい宣言が追加されても、元の型定義ではなく、extensionの一部であることを意味します。これは、(たとえば)新しいイニシャライザがメンバワイズイニシャライザを抑制しないので、一般的に役に立つ。また、プロトコルの準拠をそれ自身のextensionに分割することは、通常、良い形式であると考えられている。

しかし、準拠に使用されるメンバが、元の型定義の一部である必要がある場合もあります。たとえば、以下のような場合:

- non-finalクラスのイニシャライザは、プロトコルの要件を満たすために`required init`である必要がある
- non-finalクラスのオーバーライド可能なメンバ
- 格納プロパティやケースは、主要な型定義にしか存在できない

このような場合、メンバマクロは宣言を生成することができる。しかし、メンバマクロには、どのプロトコルに準拠したメンバを提供すべきかという情報は提供されないので、マクロは、(スーパークラスなどを通じて)すでにプロトコルに準拠している型に、誤って準拠したメンバを追加しようとする可能性がある。このため、ある種のマクロ(`Encodable`プロトコルや`Decodable`プロトコルを実装するマクロなど)が未実装になる可能性がある。


## 内容

### 提案内容

メンバマクロがプロトコルへ準拠するのに適切なメンバを提供できるようにするために、extensionマクロが準拠性を推論するのと同じ能力でメンバマクロを拡張することを提案する。具体的には:

- `member`ロールを指定する`attached`属性は、extensionマクロが提供できる準拠性を指定するのと同じように、興味のあるプロトコルコンフォーマンスのセットを指定できるようにする
- `MemberMacro`　へ準拠した実装の`expansion`操作は、(上記のように)指定されたプロトコルのセットで、その型がまだ準拠していないものを受け取る。
 
この情報によって、マクロは、どのメンバを生成して準拠性を満たすべきかを推論できる。プロトコルへの準拠に関心のあるメンバマクロは、多くの場合、extensionマクロでもあり、メンバマクロとともに動作して完全なコンフォーマンス情報を提供する。

例として、`Decodable`プロトコルと`Encodable`プロトコルでそれぞれ必要とされる`init(from:)`操作と`encode(to:)`操作を提供する`Codable`マクロを考えてみよう。このようなマクロは次のように定義できる:

```swift
@attached(member, conformances: Decodable, Encodable, names: named(init(from:), encode(to:)))
@attached(extension, conformances: Decodable, Encodable, names: named(init(from:), encode(to:)))
macro Codable() = #externalMacro(module: "MyMacros", type: "CodableMacro")
```

このマクロには、`init(from:)`と`encode(to:)`をどこでどのように生成するかについて、いくつかの重要な決定がある:

- 構造体、列挙型、アクター、`final`クラスの場合、`init(from:)`と`encode(to:)`は、準拠性とともに (memberのroleを介して)extensionモジュールに出力されるべき。これは良いスタイルであると同時に、構造体の場合、イニシャライザがメンバ単位のイニシャライザを阻害しないことを保証する。
- non-`final`クラスの場合、`init(from:)`と`encode(to:)`は、サブクラスでオーバーライドできるように、(memberのroleを介して)メインクラスの定義に出力されるべきである。
- スーパークラスから `Encodable`または`Decodable`の準拠性を継承するクラスでは、 `init(from:)` と `encode(to:)` の実装は、クラス階層全体をデコード/エンコードするために、それぞれスーパークラスのイニシャライザとメソッドを呼び出す必要がある。

型に関する既存の構文情報(`final`の有無を含む)を与え、memberとextensionのroleの両方に(ここで提案するように)型が必要とする準拠に関する情報を提供すれば、上記のすべての決定をマクロ実装で行うことができ、あらゆる型を考慮した`Codable`マクロの柔軟な実装が可能になる。

### 詳細

`attached(member,...)`属性の`conformances`引数の仕様は、[SE-0402](https://github.com/apple/swift-evolution/blob/main/proposals/0402-extension-macros.md)で文書化されているextensionマクロの対応する引数の仕様と一致する。

マクロ実装では、`MemberMacro`プロトコルの`expansion`要件は、`extension`マクロと同じプロトコルのセットを受け取る`conformingTo:`引数で補強される。`MemberMacro`プロトコルは以下のように定義される:

```swift
protocol MemberMacro: AttachedMacro {
  /// 一連のメンバを生成する付属マクロ(attached マクロ)を拡張する。
  ///
  /// - Parameters:
  ///   - node: 付属マクロを記述するカスタム属性。
  ///   - declaration: マクロ属性に付属させる宣言。
  ///   - missingConformancesTo: 一連のマクロの準拠セットで宣言され、宣言が明示的に適合しないプロトコルのセット。
  ///     メンバマクロ自身はこれらのプロトコルへの準拠を宣言できないが(extensionマクロだけができる)、
  //      必須イニシャライザや格納プロパティなど、extensionでは記述できない補助宣言を提供できる。
  ///   - context: マクロexpansionが実行されるコンテキスト。
  ///
  /// - Returns: このマクロによって導入されるメンバ宣言のセットで、`attachedTo`宣言の内部に入れ子になる。
  ///
  static func expansion(
    of node: AttributeSyntax,
    providingMembersOf declaration: some DeclGroupSyntax,
    conformingTo protocols: [TypeSyntax],
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax]
}
```

メンバマクロの定義は、準拠性そのものを提供しない。

### ソース互換性

この提案は、`@attached`属性の既存の構文空間を使用するもので、純粋な拡張である。よって影響なし。

### ABI互換性

影響なし。

### 導入の影響

この機能は、配置の制約や、ソースやABIの互換性に影響を与えることなく、ソースコードで自由に採用することができる。また、この機能を導入したマクロの使用は、マクロをその場で展開することによってソースコードから削除することができる。

## 検討された代替案

### 主要な型定義に影響を与える拡張

前述の問題に対するまったく異なるアプローチは、型定義に直接記述されているかのように、型にメンバを追加する拡張の形式を導入することである。例えば

```swift

class MyClass { ... }

@implementation extension MyClass: Codable {
  required init(from decoder: Decoder) throws { ... }
  func encode(to coder: Coder) throws { ... }
}
```

`@implementation` extensionのメンバは、メインの型定義のメンバと同じルールに従う。たとえば、格納プロパティは`@implementation` extensionで定義でき、必須イニシャライザや、オーバーライド可能なメソッド、プロパティ、subscriptも定義できる。deinitとenumのcaseも、有用であれば`@implementation` extensionで定義できる。

それらは元の型と同じソースファイルでのみ定義でき、これらのextensionは追加のジェネリック制約を持てない可能性がある。プロトコルはそれ自体実装を持たないので、`@implementation` extensionをサポートはないかもしれない。

`@implementation` extensionの存在を考えると、この提案のメンバマクロのextensionはもはや必要ないだろう。なぜなら、実装自体を拡張する必要がある場合に、`@implementation` extensionを生成するextensionマクロを使用して望ましい効果を達成できるからである。

この`@implementation` extensionの概念の主な欠点は、型の完全な「形」を見つけるために、型の主要な定義を見ることができなくなることである。その代わりに、その情報は元の型定義と`@implementation` extensionの間に散在することになり、読者は型全体のビューをつなぎ合わせる必要がある。これは、マクロは単一の実体(クラスや構造体の定義など)を見るだけで、その実体の拡張を見ることができないからである。たとえば、この提案で取り上げた`Codable`マクロは、`@implementation` extensionで記述された格納プロパティをエンコードまたはデコードできなくなり、[`Observable`](https://github.com/apple/swift-evolution/blob/main/proposals/0395-observability.md)マクロは、`@implementation` extensionで記述されたプロパティを暗黙的に観察できなくなります。この提案で説明したマクロの欠点に対処するために`@implementation` extensionを使用しようとすることは、結局、同じマクロに対してより大きな問題を引き起こすことになる。

## 参考リンク

### Forums

- [pitch](https://forums.swift.org/t/pitch-member-macros-that-know-what-conformances-are-missing/66590)
- [review](https://forums.swift.org/t/se-0407-member-macro-conformances/66951)

### プロポーザルドキュメント

- [Member Macro Conformances](https://github.com/apple/swift-evolution/blob/main/proposals/0407-member-macro-conformances.md)
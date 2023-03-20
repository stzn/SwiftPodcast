# Swift 付属マクロ(Attached Macros)

- [Swift 付属マクロ(Attached Macros)](#swift-付属マクロattached-macros)
  - [概要](#概要)
  - [内容](#内容)
    - [付属マクロの実装](#付属マクロの実装)
    - [マクロで生成された宣言の命名](#マクロで生成された宣言の命名)
    - [付属マクロの種類](#付属マクロの種類)
      - [ピア(peer)マクロ](#ピアpeerマクロ)
      - [メンバ(Member)マクロ](#メンバmemberマクロ)
      - [アクセサマクロ](#アクセサマクロ)
      - [メンバ属性マクロ](#メンバ属性マクロ)
      - [適合(Conformance)マクロ](#適合conformanceマクロ)
  - [詳細](#詳細)
    - [マクロの役割を構成する](#マクロの役割を構成する)
    - [新たに導入される名称を指定する](#新たに導入される名称を指定する)
    - [マクロが使用・導入する名前の可視性](#マクロが使用導入する名前の可視性)
    - [マクロ展開の順序](#マクロ展開の順序)
    - [許可されている宣言の種類](#許可されている宣言の種類)
  - [ソース互換性](#ソース互換性)
  - [ABI安定への影響](#abi安定への影響)
  - [APIレジリエンス](#apiレジリエンス)
  - [代替案](#代替案)
    - [宣言の拡張ではなく変更する](#宣言の拡張ではなく変更する)
  - [将来の方向性](#将来の方向性)
    - [付属マクロの役割の追加](#付属マクロの役割の追加)
  - [補足](#補足)
    - [`AddCompletionHandler`の実装](#addcompletionhandlerの実装)
  - [参考リンク](#参考リンク)
    - [Example repository](#example-repository)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

付属マクロは、引数の任意の構文変換に基づいて宣言を作成、拡張することによって、Swiftを拡張する方法を提供する。以前は新しい言語機能を導入することでしかできなかった方法が、これによってそれ以外でもSwiftを拡張することを可能にし、開発者がより表現力豊かなライブラリを構築し、不要な定型文を排除することを支援する。

付属マクロは、言語にマクロを導入するための一般的なモチベーションを示すSwiftにおける[マクロのビジョン](https://github.com/apple/swift-evolution/pull/1927)の一部である。付属マクロは、大きな新しいユースケースをカバーする[SE-0382 "式マクロ"](https://github.com/apple/swift-evolution/blob/main/proposals/0382-expression-macros.md)のアイデアと動機に基づき構築される。私たちは、マクロを言語に統合する方法の基本モデルとしてその提案を参照する。式マクロが`#`によって導入される独立したエンティティとして設計されているのに対し、付属マクロはプログラム内の特定の宣言に関連付けられ、それを補強・拡張することができる。これにより、多くの新しい使用例がサポートされ、マクロシステムの表現力が大幅に拡張される。

- トランポリン関数やラッパー関数の作成。例えば、`async`関数のcompletion handlerバージョンを自動的に作成したり、その逆を行ったりすることができる
- フラグを含む列挙型から[OptionSet](https://developer.apple.com/documentation/swift/optionset)を生成し、`OptionSet`プロトコルに適合させたり、メンバ単位のイニシャライザを追加するなど、型の定義に基づいたメンバを作成する
- [SE-0258 プロパティラッパ](https://github.com/apple/swift-evolution/blob/main/proposals/0258-property-wrappers.md)の動作の一部を網羅する、格納プロパティまたはsubscriptのためのアクセサを作成する
- 型のすべての格納プロパティにプロパティラッパを適用するなど、新しい属性で型のメンバを補強する。

この機能のプロトタイプを使用して実装された多くのマクロを含む[サンプルリポジトリ](https://github.com/DougGregor/swift-macro-examples)がある。

## 内容

この提案は、付属マクロを追加するもので、特定の宣言に添付されることから、そう呼ばれている。このマクロは、カスタム属性構文(`@AddCompletionHandler`など)を使用して記述する。この構文は、プロパティラッパ、リザルトビルダ、グローバルアクタを通じて宣言の拡張性をすでに提供している。付属マクロは、付加されている宣言について推論し、1つまたは複数の異なるマクロの役割に基づいて追加、変更することができる。各役割には、メンバの追加、「アクセサ」の作成、宣言に沿った「ピア」の追加など、特定の目的がある。付属マクロは、複数の異なる役割を持つことができるため、異なる役割に対応して複数回展開され、様々な役割を構成することができる。例えば、プロパティラッパをエミュレートする付属マクロは、「ピア」と「アクセサ」の両方の役割を持ち、バッキングストレージプロパティを導入し、そのバッキングストレージプロパティを経由するgetter/setterを合成することができる。マクロの役割の合成については、基本的なマクロの役割を説明された後に詳しく説明する。

式マクロと同様に、付属マクロは`macro`で宣言され、その動作をカスタマイズできるように[型チェックされたマクロ引数](./Swift%20Expression%20Macros.md)を持つ。付属マクロは`@attached`属性で識別され、特定の役割や導入する名前も提供される。例えば、completion handlerを追加するための付属マクロは、次のように宣言される。

```swift
@attached(peer, names: overloaded)
macro AddCompletionHandler(parameterName: String = "completionHandler")
```

このマクロは次のように使われる。

```swift
@AddCompletionHandler(parameterName: "onCompletion")
func fetchAvatar(_ username: String) async -> Image? { ... }
```

このマクロの使用は`fetchAvatar`に付与され、`fetchAvatar`と一緒に名前が「オーバーロード」された*ピア*宣言を生成します。生成される宣言は

```swift
/// マクロ展開で次のものが生成される
func fetchAvatar(_ username: String, onCompletion: @escaping (Image?) -> Void) {
  Task.detached {
    onCompletion(await fetchAvatar(username))
  }
}
```

### 付属マクロの実装

付属マクロはすべて、`AttachedMacro`プロトコルを継承したプロトコルのいずれかに準拠した型として実装される。`Macro`プロトコルと同様に、`AttachedMacro`プロトコルは要件を持たず、マクロの実装を整理するために使用される。各付属マクロの役割は、`AttachedMacro`を継承した独自のプロトコルを持つことになる。

```swift
public protocol AttachedMacro: Macro { }
```

式マクロとの最大の違いは、マクロが考慮すべき関連する構文が複数あること。付属マクロの各実装は、属性(例, `@AddCompletionHandler(parameterName: "onCompletion")`)とマクロが付与された宣言(`func fetchAvatar.`)に対して、マクロの役割に適した新しいコードを返すことができる。例えば、`PeerMacro`は次のように定義されている。

```swift
public PeerMacro: AttachedMacro {
  /// 与えられた属性で記述されたマクロを展開し、それが付属している宣言の「ピア」宣言を生成する
  ///
  /// マクロ拡張は、与えられた宣言と並行して「ピア」宣言を導入することができる
  static func expansion(
    of node: AttributeSyntax,
    providingPeersOf declaration: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) async throws -> [DeclSyntax]
}
```

### マクロで生成された宣言の命名

式マクロとは異なり、付属マクロは新しい宣言を導入することができる。例えば、マクロが`hello`という名前の関数の宣言を提供し、その関数が別のソースファイルから呼び出された場合、これらの宣言はプログラム内の他のコードに影響を与えることがある。このため、マクロが周囲のプログラムに与える影響を理解するために、開発者やツールに対して、より多くの情報を前もって提供することにしている。開発者にとっては、驚きが少なくなり、ツールにとっては、不要なマクロの拡張を避けてコンパイル時間を短縮するために利用できる。

`AddCompletionHandler`マクロは、*オーバーロードされた*名を導入しており、付属する宣言と同じベース名を持つ宣言を生成する。プロパティラッパをエミュレートするマクロでは、`prefixed(_)`を介してストレージ名を指定する。これは、そのマクロがアタッチされる宣言の名前にプレフィックスとして`_`が追加されることを意味する。マクロで生成された名前が伝達されるその他の方法については、「詳細設計」で説明する。

### 付属マクロの種類

#### ピア(peer)マクロ

ピアマクロは、付属する宣言と一緒に新しい宣言を生成する。先ほどの`AddCompletionHandler`マクロはピアマクロである。ピアマクロは、先に示した`PeerMacro`プロトコルに準拠した型によって実装される。`AddCompletionHandlerMacro`の実装は、以下のようになる:

```swift
public struct AddCompletionHandlerMacro: PeerDeclarationMacro {
  public static func expansion(
    of node: CustomAttributeSyntax,
    providingPeersOf declaration: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax] {
    // 非同期関数があることを確認し、その非同期関数名から始まる新しい関数 "completionHandlerFunc "を形成し
    // - 非同期を削除する
    // - 戻り値の型を削除します。
    // - 完了ハンドラのパラメータを追加する
    // - 引数を転送するボディを追加する
    // 新しいピア関数を返す
    return [completionHandlerFunc]
  }
}
```

実装の詳細はAppendixに譲り、完全版は[exampleリポジトリ](https://github.com/DougGregor/swift-macro-examples)にある。

#### メンバ(Member)マクロ 

メンバマクロでは、マクロが付属する型や拡張子に新しいメンバを導入することができる。例えば、`OptionSet`の定義を容易にするために、静的メンバを定義するマクロを書くことができる。例えば:

```swift
@OptionSetMembers
struct MyOptions: OptionSet {
  enum Option: Int {
    case a
    case b
    case c
  }
}
```

このstructは、`rawValue`フィールドと各オプションの静的プロパティの両方を含むように拡張される。

```swift
// 次のように展開される...
struct MyOptions: OptionSet {
  enum Option: Int {
    case a
    case b
    case c
  }
  
  // 以下、合成されたコード
  var rawValue: Int = 0
  
  static var a = MyOptions(rawValue: 1 << Option.a.rawValue)
  static var b = MyOptions(rawValue: 1 << Option.b.rawValue)
  static var c = MyOptions(rawValue: 1 << Option.c.rawValue)
}
```

マクロ自体は、任意のメンバセットを定義するメンバマクロとして宣言される。

```swift
/// structをOptionSetにするために必要なメンバを作成する。
@attached(member, names: named(rawValue), arbitrary) macro OptionSetMembers()
```

`member`の役割は、このマクロが付属宣言の新しいメンバを定義することを指定する。この場合、マクロは`rawValue`という名前のメンバを定義することを知っているが、マクロが定義する静的プロパティの名前を予測する方法がないため、任意に決定された名前のメンバを導入することを示すために`arbitrary`も指定する。

メンバマクロは、`MemberMacro`プロトコルに準拠した型で実装される。

```swift
protocol MemberMacro: AttachedMacro {
  /// 与えられた属性で記述されたマクロを展開し、その属性が付属する与えられた宣言のメンバを生成する
  static func expansion(
    of node: AttributeSyntax,
    providingMembersOf declaration: some DeclGroupSyntax,
    in context: some MacroExpansionContext
  ) async throws -> [DeclSyntax]
}
```

#### アクセサマクロ

アクセサマクロは、マクロがプロパティにアクセサを追加して、格納プロパティを計算プロパティに変えることができる。例えば、格納プロパティに適用して、プロパティ名をキーとするDictionaryにアクセスするマクロを考えてみよう。このようなマクロは、次のように使用することができる。

```swift
struct MyStruct {
  var storage: [AnyHashable: Any] = [:]
  
  @DictionaryStorage
  var name: String
  
  @DictionaryStorage(key: "birth_date")
  var birthDate: Date?
}
```

`DictionaryStorage`付属マクロでは、`MyStruct`のプロパティを以下のように変更し、それぞれにアクセサを追加する。

```swift
struct MyStruct {
  var storage: [String: Any] = [:]
  
  var name: String {
    get { 
      storage["name"]! as! String
    }
    
    set {
      storage["name"] = newValue
    }
  }
  
  var birthDate: Date? {
    get {
      storage["birth_date"] as Date?
    }
    
    set {
      if let newValue {
        storage["birth_date"] = newValue
      } else {
        storage.removeValue(forKey: "birth_date")
      }
    }
  }
}
```

マクロは以下のように宣言することができます。

```swift
@attached(accessor) macro DictionaryStorage(key: String? = nil)
```

アクセサマクロの実装は、以下のように定義される`AccessorMacro`プロトコルに準拠する。

```swift
protocol AccessorMacro: AttachedMacro {
  /// 与えられた属性で記述されたマクロを展開し、その属性が付属する与えられた宣言のアクセサを生成する
  static func expansion(
    of node: AttributeSyntax,
    providingAccessorsOf declaration: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) async throws -> [AccessorDeclSyntax]
}
```

`DictionaryStorage`マクロの実装は、(存在する場合)`key`引数を使用するか、プロパティ名からキー名を導出することで、上記のアクセサ宣言を作成する。プロパティラッパは `self.storage`にアクセスできないため、このマクロの効果は、プロパティラッパでできません。
アクセサマクロは、`@attached(accessors, names: named(willSet))`などのように、名前に`willSet`または`didSet`の少なくとも1つをリストして、オブザーバを生成することを指定できる。このようなマクロはオブザーバを生成するだけで、格納プロパティを計算プロパティに変更することはできない。
名前のリストで`willSet`または`didSet`のいずれかを指定しないアクセサマクロの拡張は、計算プロパティを生成する必要がある。拡張の副作用として、格納プロパティ自体からイニシャライザが削除される。(使用できない場合に)イニシャライザの存在を見つけ出すか、結果に組み込むかは、アクセサマクロの実装に任されている。

#### メンバ属性マクロ

メンバ宣言マクロは、それが適用される型または拡張の中に新しいメンバ宣言を導入することができる。これに対して、メンバ*属性*マクロは、型や拡張の中に明示的に書かれたメンバ宣言に新しい属性を追加して修正することができる。これらの新しい属性は、ピアマクロやアクセサマクロ、ビルトイン属性など、他のマクロを参照することができる。このように、それ自体ではほとんど効果がないため、主に合成の手段である。

メンバ属性マクロは、多くのビルトイン属性が機能するのと同様の動作を、マクロでも提供できるようにし、型または拡張で属性を宣言すると、その属性が各メンバに適用される。例えば、extension上に書かれたグローバルアクタの`@MainActor`は、そのエクステンションの各メンバに適用され(別のグローバルアクタを指定するか、non-isolatedでない限り)、extension上のアクセス指定子はそのextensionの各メンバに適用される、といった具合に。

例として、[SE-0160](https://github.com/apple/swift-evolution/blob/main/proposals/0160-objc-inference.md#re-enabling-objc-inference-within-a-class-hierarchy)でかなり前に紹介された`@objcMembers`属性の動作を部分的に模倣したマクロを定義してみる。私たちの`myObjCMembers`マクロは、メンバ属性マクロである。

```swift
@attached(memberAttribute)
macro MyObjCMembers()
```

このマクロの実装は、そのメンバがすでに`@objc`マクロを持つか、`@nonobjc`を持つ場合を除き、型または拡張子のすべてのメンバに`@objc`属性を追加する。例えば、このようになる

```swift
@MyObjCMembers extension MyClass {
  func f() { }

  var answer: Int { 42 }

  @objc(doG) func g() { }

  @nonobjc func h() { }
}
```

結果、

```swift
extension MyClass {
  @objc func f() { }

  @objc var answer: Int { 42 }

  @objc(doG) func g() { }

  @nonobjc func h() { }
}
```

メンバ属性マクロの実装は、`MemberAttributeMacro`プロトコルに準拠する必要がある。

```swift
protocol MemberAttributeMacro: AttachedMacro {
  /// 与えられたカスタム属性で記述されたマクロを展開し、型のメンバに対して追加の属性を生成する
  static func expansion(
    of node: AttributeSyntax,
    attachedTo declaration: some DeclGroupSyntax,
    providingAttributesOf member: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) async throws -> [AttributeSyntax]
}
```

`expansion`操作では、メンバ属性マクロのスペルに対する属性構文`node`と、その属性が付属する宣言(すなわち、型または拡張)を受け付ける。マクロの実装は、`declaration`のメンバを調べて、どのメンバが追加属性を必要とするかを決定することができる。返されたDictionaryは、それらのメンバから、各メンバに追加されるべき属性にマッピングされる。

#### 適合(Conformance)マクロ

適合マクロは、ある型に新しいプロトコル準拠を導入することを可能にする。これはしばしば、そのプロトコル適合性を満たすのを助けることを目的とする他のマクロと対になるであろう。例えば、先に示した `OptionSetMembers`属性の拡張版で、`OptionSet`への適合も追加することが想像できる。これによって、OptionSetの最小限の実装は次のようになる。

```swift
@OptionSet
struct MyOptions {
  enum Option: Int {
    case a
    case b
    case c
  }
}
```

ここで、`OptionSet`はメンバと適合マクロの両方であり、(`OptionSetMembers`のように)メンバと`OptionSet`への適合を提供する。このマクロは、例えば、次のように宣言されるだろう。

```swift
/// structをOptionSetにするために必要なメンバを作成する
@attached(member, names: named(rawValue), arbitrary)
@attached(conformance)
macro OptionSet()
```

適合マクロの実装は、`ConformanceMacro`プロトコルに準拠する必要がある。

```swift
/// 付属の宣言に適合性を追加できるマクロを記述する。
public protocol ConformanceMacro: AttachedMacro {
  /// 付属の適合マクロを展開し、適合の集合を生成する
  ///
  /// - Parameters:
  ///   - node: 付属マクロを記述するカスタム属性
  ///   - declaration: マクロ属性が付属している宣言
  ///   - context: マクロ展開を行うコンテキスト
  ///
  /// - Returns: 宣言された型が適合するプロトコル型と、オプショナルのwhere句を提供する`(type, where-clause?)`ペアのセット
  static func expansion(
    of node: AttributeSyntax,
    providingConformancesOf declaration: some DeclGroupSyntax,
    in context: some MacroExpansionContext
  ) throws -> [(TypeSyntax, WhereClauseSyntax?)]
}
```

返される配列には、適合性が含まれている。`TypeSyntax`は、囲む型が適合するプロトコルを記述し、オプショナルの`where`句は、適合を条件付きにする追加の制約を提供する。

## 詳細

### マクロの役割を構成する

あるマクロは複数の異なる役割を持つことができ、様々なマクロ機能を構成することができる。各役割は独立して考慮されるため、ソースコード内でマクロを1回使用するだけで、異なるマクロ展開関数が呼び出されることがある。これらの呼び出しは独立しており、同時に起こる可能性もある。例として、プロパティラッパをかなり忠実にエミュレートするマクロを定義してみる。プロパティラッパの提案には、[Clampingプロパティラッパ](https://github.com/apple/swift-evolution/blob/main/proposals/0258-property-wrappers.md#clamping-a-value-within-bounds)の例があります。

```swift
@propertyWrapper
struct Clamping<V: Comparable> {
  var value: V
  let min: V
  let max: V

  init(wrappedValue: V, min: V, max: V) {
    value = wrappedValue
    self.min = min
    self.max = max
    assert(value >= min && value <= max)
  }

  var wrappedValue: V {
    get { return value }
    set {
      if newValue < min {
        value = min
      } else if newValue > max {
        value = max
      } else {
        value = newValue
      }
    }
  }
}

struct Color {
  @Clamping(min: 0, max: 255) var red: Int = 127
  @Clamping(min: 0, max: 255) var green: Int = 127
  @Clamping(min: 0, max: 255) var blue: Int = 127
  @Clamping(min: 0, max: 255) var alpha: Int = 255
}
```

その代わり、マクロとして実装してみよう。

```swift
@attached(peer, prefixed(_))
@attached(accessor)
macro Clamping<T: Comparable>(min: T, max: T) = #externalMacro(module: "MyMacros", type: "ClampingMacro")
```

使用構文はどちらも同じ。マクロとして、`Clamping`はピア(プレフィックスが`_`のバッキングストレージプロパティ)を定義し、さらに(`min`/`max`をチェックするための)アクセサを定義する。ピア宣言マクロは、バッキングストレージの定義を担当する、例えば。

```swift
private var _red: Int = {
  let newValue = 127
  let minValue = 0
  let maxValue = 255
  if newValue < minValue {
    return minValue
  }
  if newValue > maxValue {
    return maxValue
  }
  return newValue
}()
```

これは、`ClampingMacro`が`PeerMacro`に準拠することで実装される。

```swift
enum ClampingMacro { }

extension ClampingMacro: PeerDeclarationMacro {
  static func expansion(
    of node: CustomAttributeSyntax,
    providingPeersOf declaration: DeclSyntax,
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax] {
    // 元の変数と同じ新しい変数宣言を作成する。以下のようになる。
    //   - 名前の前にアンダースコアを付ける
    //   - privateにする
  }
}
```

また、以下のようなアクセサを導入している。

```swift
    get { return _red }

    set(__newValue) {
      let __minValue = 0
      let __maxValue = 255
      if __newValue < __minValue {
        _red = __minValue
      } else if __newValue > __maxValue {
        _red = __maxValue
      } else {
        _red = __newValue
      }
    }
```


これは、`ClampingMacro`が`AccessorMacro`に準拠することで実装される。

```swift
extension ClampingMacro: AccessorMacro {
  static func expansion(
    of node: CustomAttributeSyntax,
    providingAccessorsOf declaration: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) throws -> [AccessorDeclSyntax] {
    let originalName = /* 宣言から取得 */, 
        minValue = /* カスタム属性ノードから取得 */,
        maxValue = /* カスタム属性ノードから取得 */
    let storageName = "_\(originalName)",
        newValueName = context.getUniqueName(),
        maxValueName = context.getUniqueName(),
        minValueName = context.getUniqueName()
    return [
      """
      get { \(storageName) }
      """,
      """
      set(\(newValueName)) {
        let \(minValueName) = \(minValue)
        let \(maxValueName) = \(maxValue)
        if \(newValueName) < \(minValueName) {
          \(storageName) = \(minValueName)
        } else if \(newValueName) > maxValue {
          \(storageName) = \(maxValueName)
        } else {
          \(storageName) = \(newValueName)
        }
      }
      """
    ]
  }  
}
```

この`@Clamping`の定式化には、プロパティラッパのバージョンと比較していくつかの利点がある。バッキングストレージの一部としてminとmaxを格納する必要がなく(したがって、`@Clamping`が存在してもストレージは追加されない)、新しい型を定義する必要もない。さらに重要なのは、異なるマクロの役割を一緒に構成することで、興味深い効果が得られることを実証していることです。

アクセサマクロは関数の意味をなさないし、メンバマクロはプロパティの意味をなさない。マクロが宣言に付属している場合、マクロは宣言に適用される役割に対してのみ展開されます。例えば、`Clamping`マクロが関数に適用された場合、ピアマクロとしてのみ展開され、アクセサマクロとしての役割は無視されます。マクロが宣言に付属していて、その役割のどれもがその種類の宣言に適用されない場合、そのプログラムは不正な形式となる。

### 新たに導入される名称を指定する

マクロが他のSwiftコードに見える宣言を生成するときはいつでも、事前に名前を宣言することが要求されます。すべての名前は、以下の形式を使用して、マクロの役割を宣言する属性の中で指定する必要があります。

- 特定の固定名を持つ宣言: named(<declaration-name>).
- 他の入力から派生したものなど、名前が静的に記述できない宣言：`arbitrary`
- マクロが付属する宣言と同じベース名を持ち、その宣言でオーバーロードされる宣言: overloaded.
- マクロが付属する宣言の名前に接頭辞を付加した名前を持つ宣言：接頭辞(_)。特別な配慮として、$は接頭辞として許容され、マクロは、マクロが接続される宣言の名前から派生して、先頭に$を持つ名前を生成することができます。この切り分けにより、投影値を導入するプロパティラッパーと同様の動作をするマクロが可能になります。
- マクロが付属する宣言の名前に接尾辞を付加した名前を持つ宣言： suffixed(_docinfo).

マクロは、指定された種類で名前がカバーされるか、MacroExpansionContext.createUniqueNameで名前が生成された新しい宣言のみを導入できます。これは、ほとんどの場合 (arbitrary が指定されていない場合)、Swift コンパイラと関連ツールは、マクロを展開することなく、マクロの特定の使用によって導入される名前のセットについて推論できることを保証し、マクロのコンパイル時のコストを削減し、インクリメンタルビルドを改善できる。例えば、コンパイラが_xという名前のルックアップを行うとき、_xという名前の宣言を生成できないマクロの展開を避けることができる。任意の名前を生成できるマクロは、xという名前の宣言に付属する接頭辞(_)を持つマクロのように、常に展開されなければならない。

例えば、OptionSetMembers は rawValue という名前の宣言を生成することを表明しますが、そのようなプロパティがすでに存在することを確認した場合、実装はそうしないことを選択することができます。

### マクロが使用・導入する名前の可視性

マクロはプログラムに新しい名前を導入することができます。これらの名前がプログラムの他の部分、特に他のマクロから見えるかどうか、またいつ見えるかは微妙な設計問題であり、興味深い制約がいくつもある。

まず、マクロの引数は、どのマクロが展開されているかを判断する前に型チェックされるので、そのマクロの展開に依存することはできない。例えば、`@myMacro(x)`と綴られた付属マクロの使用を考えてみよう。`x`という名前の宣言を導入すると、それ自身の引数の型チェックが無効になったり、マクロ展開が`x`を生成しない別の`myMacro`を選択する原因になる可能性がある!

次に、あるマクロ展開の出力(例えば `@myMacro1(x)`)が、別のマクロ展開(例えば`@myMacro2(y)`)の引数に使用される名前(例えば`y`)を導入する場合、マクロを展開する順序によって、結果のプログラムのセマンティクスに影響を与えることがある。Swiftは一般的にプログラム内の宣言の間に順序依存性を持たないようにするため、マクロ展開の順序を導入することは一般的に不可能であり、またそれは望ましいことではない。宣言マクロの名前は任意に指定でき、したがって述語化できないので、マクロによって導入された名前に基づく付随的な順序付けも、役に立たない。

第三に、マクロの展開は、コンパイラ/IDEからマクロを展開する別のプログラムへプログラムの一部を転送するためのシリアライズオーバーヘッドを伴う、比較的高価なコンパイル時操作である。したがって、[マクロは使用部位ごとに一度だけ展開される](https://github.com/apple/swift-evolution/blob/main/proposals/0382-expression-macros.md#macro-expansion)ため、その展開は引数や周囲の他の文や式の型チェックに参加することはできない。例えば、このような(意図的にわかりにくくしている)Swiftのコードを考えてみましょう。

```swift
@attached(peer) macro myMacro(_: Int)
@attached(peer, names: named(y)) macro myMacro(_: Double)

func f(_: Int, body: (Int) -> Void)
func f(_: Double, body: (Double) -> Void)

f(1) { x in
  @myMacro(x) func g() { }
  print(y)
}
```

`@myMacro(x)`マクロの使用は、引数が`Int`か`Double`かによって異なる名前を提供する。最初の`myMacro`では、`print(y)`式は型チェックに失敗する(`y`がない)のに対し、2番目の`myMacro`では生成された`y`が見つかり、成功する。しかし、この方法では、型チェックされる際にマクロを何度も拡張する必要があるため、スケーラビリティに欠ける。さらに、例えばマクロがある時点で型情報を提供され、その(型推論中に展開が行われているために)型情報がまだわからない場合、マクロ展開は不完全な情報を得ることになるかもしれない。

このような問題に対処するため、マクロ拡張によって導入された名前は、マクロ拡張と同じスコープまたはそれを囲むスコープにあるマクロの引数内では表示されないようにした。これは[式マクロのアウトサイドイン展開](https://github.com/apple/swift-evolution/blob/main/proposals/0382-expression-macros.md#macro-expansion)と概念的に似ており、特定のスコープにあるすべてのマクロがトップレベルのスコープで同時に型チェックされ展開されると想像することができる。そして、各型定義とその拡張の中のマクロは、型チェックされ、同時に拡張される。それらは、トップレベルのマクロ拡張によって導入された名前を見ることができるが、他のレベルで導入された名前を見ることはできない。次の注釈付きコード例では、どの名前が見えるかを示している。

```swift
import OtherModule // x, y, zという名前の変数を宣言している

@macro1(x) // OtherModule.xを使い、yという名前の変数を導入
func f() { }

@macro2(y) // OtherModule.yを使い、xという名前の変数を導入
func g() { }

struct S1 {
  @macro3(x) // @macro2で導入されたxを使う、zという名前の変数を導入
  func h()
}

extension S1 {
  @macro4(z) // OtherModule.zを使う
  func g() { }
  
  func f() {
    print(z) // @macro3で導入されたzを使う
  }
}
```

これらの名前ルックアップの規則は、マクロが互いに干渉することなく拡張されるための合理的なスキームを提供するために重要だが、言語に新しい種類の名前シャドーイングを導入している。もし`@macro2`がソースコードに手動で展開された場合、それが生成する宣言`x`が`@macro1(x)`のマクロ引数に見えるようになり、プログラムの動作が変化する。特定のスコープでマクロが導入された名前が、その特定のスコープからのルックアップの意味を変えてしまうような場合、コンパイラは実際にこのようなシャドーイングを検出できるかもしれない。

関数やクロージャのボディでは、マクロ展開によって導入された名前は現在のスコープでは見えないため、先ほどの例では`@myMacro`によって導入された宣言`y`を見つけることができない。

```swift
f(1) { x in
  @myMacro(x) func g() { }
  print(y)     // `@myMacro`で導入されたどの名前も見つけられない。
}
```

したがって、クロージャや関数本体内で使用されるマクロは、`createUniqueName`で生成された名前を使用した宣言のみを導入することができる。これにより、マクロを展開せずに型チェックと推論を行い、その後、マクロを展開してその戻り値の型チェックを独立して行い、それ以上型推論に影響を与えないという、[2段階のマクロチェック](https://github.com/apple/swift-evolution/blob/main/proposals/0382-expression-macros.md#macro-expansion)を維持している。


### マクロ展開の順序

1つの宣言に複数のマクロを適用する場合、あるマクロ展開の出力が他のマクロの入力として扱われると、マクロ展開の順序が結果のプログラムに大きな影響を与えることがある。このようなマクロの構成は非常に強力だが、意外な副作用があることはある。

1. 宣言に属性を書き込む順序が結果に影響する可能性がある
2. マクロの入力がどのようなものであるかについて、明白なsingle source of truthはない。
3. 独立して開発されたマクロが、プログラムを破壊するような形で衝突する可能性がある。

付属マクロのきめ細かい性質は、多くの付属マクロが独立した効果を持つことを意味し、例えば、マクロは別個のメンバマクロ、適合マクロ、およびピアマクロを生成できる。同じ定義に適用される異なるマクロの独立性は、マクロの適用をより予測しやすくし、より効率的な実装を可能にするため、重要な機能であると考えている。

マクロ呼び出しの独立性を確保するために、各マクロ展開は、他のマクロの効果で更新されない、最初に書かれたままの構文ノードを提供する。例えば、先ほどの`OptionSet`マクロを考えてみよう。

```swift
@OptionSet
struct MyOptions {
  enum Option: Int {
    case a
    case b
    case c
  }
} 
```

`OptionSet`マクロは、メンバマクロ(インスタンスプロパティ`rawValue`と静的プロパティ`a`, `b`, `c`を生成する)と適合マクロ(`OptionSet`の適合性を追加する)の両方を持つ。`OptionSet`マクロの実装では、`expansion(of:providingMembersOf:in:)`と`expansion(of:providingConformancesOf:in:)`の2つの異なる展開関数が呼び出される。どちらの場合も、`MyOptions`のシンタックスツリーが中間引数として渡され、どちらの関数も、メンバーの追加または適合性の追加によって、`MyOptions`の定義を拡張する結果を提供する。

どちらの場合も、展開操作は、新しい適合性または追加されたメンバなしで、`MyOptions`の元の定義で提供される。そうすれば、それぞれの展開操作は、同じマクロのための異なる役割であろうと、完全に異なるマクロであろうと、他のものから独立して動作し、展開の順序は問題ではありません。

### 許可されている宣言の種類

マクロは、マクロが展開されるコンテキスト内で構文的および意味的に整った任意の宣言に展開することができるが、いくつかの顕著な例外がある。

- `import`宣言は決してマクロによって生成されることはない。Swiftのツールは、元のソースファイルの単純なスキャンに基づいて`import`宣言を解決する能力に依存している。マクロ展開が`import`宣言を導入することを許可すると、`import`の解決はかなり複雑になる。
- `main`属性を持つ型は、マクロによって生成することができない。Swiftのツールは、マクロを展開することなく、ソースファイルの構文スキャンに基づいて、ソースファイル内のメインエントリーポイントの存在を判断することができるはず
- `extension`宣言は、決してマクロによって生成することができない。`extension`の効果は多岐にわたり、適合やメンバなどを追加することができる。これらの機能は、よりきめ細かい方法で導入することを意図している
- 演算子宣言と`precedencegroup`宣言は、マクロで生成することはできない。なぜなら、既存のコードの優先順位グラフを変更することができ、マクロの展開を見たコードとそうでないコードのセマンティクスに微妙な違いが生じる可能性があるため
- `macro`宣言は、マクロによって生成されることはない
- `IntegerLiteralType`, `FloatLiteralType`, `BooleanLiteralType`, `ExtendedGraphemeClusterLiteralType`, `UnicodeScalarLiteralType`, `StringLiteralType`などのトップレベルのデフォルトのリテラル型のオーバーライドは、決してマクロによって生成することができない

## ソース互換性

影響なし

## ABI安定への影響

影響なし

## APIレジリエンス

影響なし

## 代替案

### 宣言の拡張ではなく変更する

付属宣言マクロの実装には、マクロ展開そのものと付属宣言に関する情報が提供される。実装は、付属される宣言の構文を直接変更することはできません。むしろ、`AttachedDeclarationExpansion`でパッケージ化することによって、追加または変更を指定する必要がある。

別のアプローチとして、マクロ実装がマクロが付属されている宣言を直接変更できるようにすることもできる。これにより、マクロ実装は、マクロが付属されている宣言に影響を与える柔軟性が高まる。しかし、その結果、ユーザが書いた宣言と大きく異なる可能性があり、マクロを展開する前に宣言の意味についてコンパイラが判断することができなくなる。これは、コンパイル時のパフォーマンスに悪影響を及ぼす可能性があり(マクロが実行されるまで、宣言について何も判断できない)、また、マクロが将来、実際の宣言に関する情報(パラメータの型など)にアクセスできなくなる可能性もある。

元の宣言に対する変更で表現されるマクロ実装APIを提供することは可能かもしれないが、許容される変更は、実装によって処理できるものに限定される。たとえば、マクロ実装に提供された構文木とマクロ実装が生成した構文木の「差分」から変更を特定し、コンパイラが許容する変更を返すことができる。

## 将来の方向性

### 付属マクロの役割の追加

この提案を拡張して新しいマクロの役割を提供する方法は数多くある。新しいマクロの役割はそれぞれ、`@attached`属性に新しい役割の種類と、対応するプロトコルを導入する。マクロのビジョン文書には、このような提案が多数ある。

## 補足

### `AddCompletionHandler`の実装

```swift
public struct AddCompletionHandler: PeerDeclarationMacro {
  public static func expansion(
    of node: CustomAttributeSyntax,
    providingPeersOf declaration: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax] {
    // 今のところ関数にのみ対応。イニシャライザも少し手を加えれば扱えるようになる
    guard let funcDecl = declaration.as(FunctionDeclSyntax.self) else {
      throw CustomError.message("@AddCompletionHandler only works on functions")
    }

    // これは非同期関数にのみ意味がある
    if funcDecl.signature.asyncOrReasyncKeyword == nil {
      throw CustomError.message(
        "@AddCompletionHandler requires an async function"
      )
    }

    // completion handlerパラメータを形成
    let resultType: TypeSyntax? = funcDecl.signature.output?.returnType.withoutTrivia()

    let completionHandlerParam =
      FunctionParameterSyntax(
        firstName: .identifier("completionHandler"),
        colon: .colonToken(trailingTrivia: .space),
        type: "(\(resultType ?? "")) -> Void" as TypeSyntax
      )

    // completion handlerパラメータをパラメータパラメータリストに追加
    let parameterList = funcDecl.signature.input.parameterList
    let newParameterList: FunctionParameterListSyntax
    if let lastParam = parameterList.last {
      // 前のリストに末尾のカンマを追加する必要がある
      newParameterList = parameterList.removingLast()
        .appending(
          lastParam.withTrailingComma(
            .commaToken(trailingTrivia: .space)
          )
        )
        .appending(completionHandlerParam)
    } else {
      newParameterList = parameterList.appending(completionHandlerParam)
    }

    let callArguments: [String] = try parameterList.map { param in
      guard let argName = param.secondName ?? param.firstName else {
        throw CustomError.message(
          "@AddCompletionHandler argument must have a name"
        )
      }

      if let paramName = param.firstName, paramName.text != "_" {
        return "\(paramName.withoutTrivia()): \(argName.withoutTrivia())"
      }

      return "\(argName.withoutTrivia())"
    }

    let call: ExprSyntax =
      "\(funcDecl.identifier)(\(raw: callArguments.joined(separator: ", ")))"

    // FIXME: CodeBlockSyntaxをExpressibleByStringInterpolationにして、フルボディをここに置くことができるようにすべき。
    let newBody: ExprSyntax =
      """

        Task.detached {
          completionHandler(await \(call))
        }

      """

    // 新しい宣言から@addCompletionHandler属性を削除する
    let newAttributeList = AttributeListSyntax(
      funcDecl.attributes?.filter {
        guard case let .customAttribute(customAttr) = $0 else {
          return true
        }

        return customAttr != node
      } ?? []
    )

    let newFunc =
      funcDecl
      .withSignature(
        funcDecl.signature
          .withAsyncOrReasyncKeyword(nil)  // async削除
          .withOutput(nil)                 // 戻り値の型を削除
          .withInput(                      // completion handlerパラメータの追加
            funcDecl.signature.input.withParameterList(newParameterList)
              .withoutTrailingTrivia()
          )
      )
      .withBody(
        CodeBlockSyntax(
          leftBrace: .leftBraceToken(leadingTrivia: .space),
          statements: CodeBlockItemListSyntax(
            [CodeBlockItemSyntax(item: .expr(newBody))]
          ),
          rightBrace: .rightBraceToken(leadingTrivia: .newline)
        )
      )
      .withAttributes(newAttributeList)
      .withLeadingTrivia(.newlines(2))

    return [DeclSyntax(newFunc)]
  }
}
```


## 参考リンク

### Example repository

https://github.com/DougGregor/swift-macro-examples

### Forums

- [pitch #1, under the name "declaration macros"](https://forums.swift.org/t/pitch-declaration-macros/62373)
- [pitch #2](https://forums.swift.org/t/pitch-attached-macros/62812)
- [review](https://forums.swift.org/t/se-0389-attached-macros/63165)
- [acceptance](https://forums.swift.org/t/accepted-se-0389-attached-macros/63593)


### プロポーザルドキュメント

- [Attached Macros](https://github.com/apple/swift-evolution/blob/main/proposals/0389-attached-macros.md)

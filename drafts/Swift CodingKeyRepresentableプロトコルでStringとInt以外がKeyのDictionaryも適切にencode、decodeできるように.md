# Swift CodingKeyRepresentableプロトコルでStringとInt以外がKeyのDictionaryも適切にエンコード、デコードできるように

- [Swift CodingKeyRepresentableプロトコルでStringとInt以外がKeyのDictionaryも適切にエンコード、デコードできるように](#swift-codingkeyrepresentableプロトコルでstringとint以外がkeyのdictionaryも適切にエンコードデコードできるように)
  - [概要](#概要)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決方法](#解決方法)
      - [`CodingKeyRepresentable`](#codingkeyrepresentable)
      - [例:](#例)
        - [RawValueがStringのenum](#rawvalueがstringのenum)
        - [独自のstruct](#独自のstruct)
      - [CodingKeyRepresentableに準拠した型のDictionaryへのエンコードの実装](#codingkeyrepresentableに準拠した型のdictionaryへのエンコードの実装)
      - [CodingKeyRepresentableに準拠した型のDictionaryからのデコードの実装](#codingkeyrepresentableに準拠した型のdictionaryからのデコードの実装)
      - [RawValueがStringまたはIntのRawRepresentableのCodingKeyRepresentableにはデフォルト実装を提供](#rawvalueがstringまたはintのrawrepresentableのcodingkeyrepresentableにはデフォルト実装を提供)
      - [内部型の_DictionaryCodingKeyが失敗しないイニシャライザを持つように変更](#内部型の_dictionarycodingkeyが失敗しないイニシャライザを持つように変更)
    - [既存のコードへの影響](#既存のコードへの影響)
    - [その他の検討事項](#その他の検討事項)
      - [標準ライブラリの型をCodingKeyRepresentableに準拠させる](#標準ライブラリの型をcodingkeyrepresentableに準拠させる)
      - [標準ライブラリにAnyCodingKey型を追加](#標準ライブラリにanycodingkey型を追加)
    - [今回採用されなかった案](#今回採用されなかった案)
      - [なぜ既存のCodingKeyを変更(デフォルト実装を追加)して単に型をCodingKeyに直接準拠させないのか？](#なぜ既存のcodingkeyを変更デフォルト実装を追加して単に型をcodingkeyに直接準拠させないのか)
      - [なぜRawRepresentableを見直したり、RawRepresentable where RawValue == CodingKey制約を使わない？](#なぜrawrepresentableを見直したりrawrepresentable-where-rawvalue--codingkey制約を使わない)
      - [なぜCodingKeyのassociated typeを使わない？](#なぜcodingkeyのassociated-typeを使わない)
      - [Encoder/Decoderにワークアラウンドを追加する](#encoderdecoderにワークアラウンドを追加する)
      - [`newtype`の設計を待つ](#newtypeの設計を待つ)
      - [何もしない](#何もしない)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [実装](#実装)

## 概要

Codableを使ってプレーンな`String`や`Int`以外の型を`Dictionary`のキーとして使用すると、期待とは異なる結果になっていた。今回新しく`CodingKeyRepresentable`プロトコルを導入してその問題を解消する。

## 内容

### 問題点

例えば、下記のような`RawValue`が`String`の`enum`を`Dictionary`のキーに指定して`JSONEncoder`でエンコードするとキーバリューのペア(`KeyedContainer`)ではなく配列(`UnkeyedContainer`)になり、`JSONDecoder`でデコードするとエラーになる。

```swift

let json = "{\"key\": \"value\"}"
enum Key: String, Codable {
   case key
}
let jsonData = Data(json.utf8)
let dataToEncode = [Key.key: "value"]
do {
   let decoded = try JSONDecoder().decode([Key: String].self, from: jsonData)

   let encoded = try JSONEncoder().encode(dataToEncode)
   print(String(data: encoded, encoding: .utf8)!) // ①
} catch {
   print(error) // ②
}
```

①エンコード結果
```swift
[ "key", "value" ]
```

②デコード結果
```swift
typeMismatch(Swift.Array<Any>, Swift.DecodingError.Context(codingPath: [], debugDescription: "Expected to decode Array<Any> but found a dictionary instead.", underlyingError: nil))
```
特に

- 上記で示したような`enum`(特に`String`もしくは`Int`を`RawValue`にした`RawRepresentable`)
- `String`のWrapperクラス(例: [Tagged](https://github.com/pointfreeco/swift-tagged))
- `Int8`などの`Int*`

をキーとして使用した場合に多くの人が混乱している。

※ 実際に`String`と`Int`は特別扱いされている。
https://github.com/apple/swift/blob/main/stdlib/public/core/Codable.swift#L5616
しかし、この既存の実装に修正を加えると

1. これまでの動作を破壊してしまい、後方互換性に影響がある(新しいコードは過去のコードをでコードできない、逆も同様)。
2. この動作は標準ライブラリと結びついているので、OSのバージョンによって動作が異なる。


そこで新しく`CodingKeyRepresentable`というプロトコルを提供し、これに準拠した型を`Dictionary`のキーとして使用することで、`KeyedContainer`としてエンコード/デコードできるようにする。

### 解決方法

#### `CodingKeyRepresentable`

`CodingKeyRepresentable`に準拠した型は`CodingKey`として利用できることを示し、`KeyedContainer`にエンコードするためにそれらで定義された`CodingKey`を`Dictionary`にオプトインで使用することができる。

このオプトインは、プロトコルが利用可能なバージョンのSwiftでのみ発生するため、ユーザは状況を完全に制御できる。例えば、現在、独自のワークアラウンドを使用していても、この機能を備えた特定の将来のSwiftバージョンを実行するiOSバージョンのみをサポートすると、独自のワークアラウンドをスキップして、代わりにこの動作に依存できる。

```swift
/// A type that can be converted to and from a coding key.
///
/// With a `CodingKeyRepresentable` type, you can losslessly convert between a
/// custom type and a `CodingKey` type.
///
/// Conforming a type to `CodingKeyRepresentable` lets you opt in to encoding
/// and decoding `Dictionary` values keyed by the conforming type to and from
/// a keyed container, rather than encoding and decoding the dictionary as an
/// unkeyed container of alternating key-value pairs.
@available(SwiftStdlib 5.6, *)
public protocol CodingKeyRepresentable {
  @available(SwiftStdlib 5.6, *)
  var codingKey: CodingKey { get }
  @available(SwiftStdlib 5.6, *)
  init?<T: CodingKey>(codingKey: T)
}
```

https://github.com/apple/swift/blob/4f7f9f5e615f815800d2c802d6daa39c5e5cf9a2/stdlib/public/core/Codable.swift#L5539

#### 例:

##### RawValueがStringのenum

```swift

let json = "{\"key\": \"value\"}"
enum Key: String, Codable, CodingKeyRepresentable {
   case key
}
let jsonData = Data(json.utf8)
let dataToEncode = [Key.key: "value"]
do {
   let decoded = try JSONDecoder().decode([Key: String].self, from: jsonData)
   print(decoded) // ①
   let encoded = try JSONEncoder().encode(dataToEncode)
   print(String(data: encoded, encoding: .utf8)!) // ②
} catch {
   print(error)
}
```

①デコード結果
```swift
[main.Key.key: "value"]
```

②エンコード結果
```swift
{"key":"value"}
```

##### 独自のstruct

```swift
// 標準ライブラリの_DictionaryCodingKeyと同じ
struct _AnyCodingKey: CodingKey {
    let stringValue: String
    let intValue: Int?

    init(stringValue: String) {
        self.stringValue = stringValue
        self.intValue = Int(stringValue)
    }

    init(intValue: Int) {
        self.stringValue = "\(intValue)"
        self.intValue = intValue
    }
}

struct ID: Hashable, CodingKeyRepresentable, Codable {
    static let knownID1 = ID(stringValue: "<some-identifier-1>")
    static let knownID2 = ID(stringValue: "<some-identifier-2>")

    let stringValue: String

    var codingKey: CodingKey {
        return _AnyCodingKey(stringValue: stringValue)
    }

    init?<T: CodingKey>(codingKey: T) {
        stringValue = codingKey.stringValue
    }

    init(stringValue: String) {
        self.stringValue = stringValue
    }
}

let data: [ID: String] = [
    .knownID1: "...",
    .knownID2: "...",
]

let encoder = JSONEncoder()
try String(data: encoder.encode(data), encoding: .utf8)

/*
{
    "<some-identifier-1>": "...",
    "<some-identifier-2>": "...",
}
*/

let decoder = JSONDecoder()
try decoder.decode([ID: String].self, from: encoder.encode(data))

/*
[
    main.ID(stringValue: "<some-identifier-1>"): "...",
    main.ID(stringValue: "<some-identifier-2>"): "..."
]
*/
```

#### CodingKeyRepresentableに準拠した型のDictionaryへのエンコードの実装

```swift
} else if #available(SwiftStdlib 5.6, *), 
          Key.self is CodingKeyRepresentable.Type {
    // Since the keys are CodingKeyRepresentable, we can use the `codingKey`
    // to create `_DictionaryCodingKey` instances.
    var container = encoder.container(keyedBy: _DictionaryCodingKey.self)
    for (key, value) in self {
        let codingKey = (key as! CodingKeyRepresentable).codingKey
        let dictionaryCodingKey = _DictionaryCodingKey(codingKey: codingKey)
        try container.encode(value, forKey: dictionaryCodingKey)
    }
} else {
    // Keys are Encodable but not Strings or Ints, so we cannot arbitrarily
```

https://github.com/apple/swift/blob/4f7f9f5e615f815800d2c802d6daa39c5e5cf9a2/stdlib/public/core/Codable.swift#L5630


#### CodingKeyRepresentableに準拠した型のDictionaryからのデコードの実装

```swift
} else if #available(SwiftStdlib 5.6, *),
          Key.self is CodingKeyRepresentable.Type {
    // The keys are CodingKeyRepresentable, so we should be able to expect
    // a keyed container.
    let container = try decoder.container(keyedBy: _DictionaryCodingKey.self)
    for codingKey in container.allKeys {
        guard let key: Key = keyType.init(codingKey: codingKey) as? Key else {
            throw DecodingError.dataCorruptedError(
                forKey: codingKey,
                in: container,
                debugDescription: "Could not convert key to type \(Key.self)"
            )
        }
        let value: Value = try container.decode(Value.self, forKey: codingKey)
        self[key] = value
    }
} else {
    // Keys are Encodable but not Strings or Ints, so we cannot arbitrarily
    // convert to keys. We can encode as an array of alternating key-value
    // pairs, though.
```

https://github.com/apple/swift/blob/4f7f9f5e615f815800d2c802d6daa39c5e5cf9a2/stdlib/public/core/Codable.swift#L5694

#### RawValueがStringまたはIntのRawRepresentableのCodingKeyRepresentableにはデフォルト実装を提供

このプロポーザルの多くのユースケースで、`CodingKeyRepresentable`に準拠している型は(`RawValue`が`String`または`Int`の)`RawRepresentable`に既に準拠している。そこで、これらのケースで独自実装によって起こる不一致を避けるために、`RawValue`が`String`または`Int`の場合の`RawRepresentable`にデフォルト実装を提供する。

```swift
@available(SwiftStdlib 5.6, *)
extension RawRepresentable
where Self: CodingKeyRepresentable, RawValue == String {
    @available(SwiftStdlib 5.6, *)
    public var codingKey: CodingKey {
        _DictionaryCodingKey(stringValue: rawValue)
    }
    @available(SwiftStdlib 5.6, *)
    public init?<T: CodingKey>(codingKey: T) {
        self.init(rawValue: codingKey.stringValue)
    }
}
```

https://github.com/apple/swift/blob/4f7f9f5e615f815800d2c802d6daa39c5e5cf9a2/stdlib/public/core/Codable.swift#L5548

```swift
@available(SwiftStdlib 5.6, *)
extension RawRepresentable where Self: CodingKeyRepresentable, RawValue == Int {
    @available(SwiftStdlib 5.6, *)
    public var codingKey: CodingKey {
        _DictionaryCodingKey(intValue: rawValue)
    }
    @available(SwiftStdlib 5.6, *)
    public init?<T: CodingKey>(codingKey: T) {
        if let intValue = codingKey.intValue {
            self.init(rawValue: intValue)
        } else {
            return nil
        }
    }
}
```

https://github.com/apple/swift/blob/4f7f9f5e615f815800d2c802d6daa39c5e5cf9a2/stdlib/public/core/Codable.swift#L5560

例えば、下記のように使用できる

```swift
// StringWrapperはRawValue == StringなRawRepresentableに既に準拠しているとする
extension StringWrapper: CodingKeyRepresentable {}
```

#### 内部型の_DictionaryCodingKeyが失敗しないイニシャライザを持つように変更

これは実際失敗することがなく、不要なオプショナルバイディングを減らすことができる.

```swift
/// A wrapper for dictionary keys which are Strings or Ints.
internal struct _DictionaryCodingKey: CodingKey {
    internal let stringValue: String
    internal let intValue: Int?

    internal init(stringValue: String) {
        self.stringValue = stringValue
        self.intValue = Int(stringValue)
    }

    internal init(intValue: Int) {
        self.stringValue = "\(intValue)"
        self.intValue = intValue
    }

    fileprivate init(codingKey: CodingKey) {
        self.stringValue = codingKey.stringValue
        self.intValue = codingKey.intValue
    }
}
```

https://github.com/apple/swift/blob/4f7f9f5e615f815800d2c802d6daa39c5e5cf9a2/stdlib/public/core/Codable.swift#L5509

### 既存のコードへの影響

このプロトコルを取り入れることは追加なので直接は影響がない。

ただし、以前に`Dictionary`のキーとしてエンコードされたT型に準拠させると、アーカイブとの後方互換性が失われる可能性があるため、特別な注意が必要。 新しい型または`Codable`に新しく準拠した型に`CodingKeyRepresentable`を準拠させることは常に安全。

### その他の検討事項

#### 標準ライブラリの型をCodingKeyRepresentableに準拠させる


後方互換性の懸念により、標準ライブラリやFoundationの型をさせることは提案しない。もしエンドユーザのコードが既存の型でこの変換を必要とする場合は、それらの型に代わって準拠するWrapper型を作成することを推奨(例えば、`UUID`を含み、`CodingKeyRepresentable`に準拠して`UUID`を`Dictionary`のキーとして直接使用できるようにする`MyUUIDWrapper`)。

#### 標準ライブラリにAnyCodingKey型を追加

`CodingKeyRepresentable`に準拠する型は`CodingKey`を提供する必要があるので、型の内容から自動で生成できる可能性は高い。これは、初期化時に任意の`String`または`Int`を受け取ることができる一般的なキー型を導入する良い機会かもしれない。

`Dictionary`は既に内部で`_DictionaryCodingKey`を使用している(`JSONEncoder`/`JSONDecoder`には`_JSONKey`、, `PropertyListEncoder`/`PropertyListDecoder`には`_PlistKey`)。そのため、これを汎用化することは役に立つかもしれないと考える。この型の実装は上記で示した`_AnyCodingKey`と同じになる。

### 今回採用されなかった案

#### なぜ既存のCodingKeyを変更(デフォルト実装を追加)して単に型をCodingKeyに直接準拠させないのか？

2つの理由がある。

1. `CodingKey`に既に準拠している場合に、希に動作を変えてしまうリスクがある
2. `CodingKey`の`stringValue`と`intValue`プロパティを露出させる必要があるが、これはエンコード/デコード時にのみ適切。勝手にこれを外部に露出させるのは適切でないと思われる。

https://forums.swift.org/t/codingkeypath-add-support-for-encoding-and-decoding-nested-objects-with-dot-notation/34710/16

https://forums.swift.org/t/codingkeypath-add-support-for-encoding-and-decoding-nested-objects-with-dot-notation/34710/33

#### なぜRawRepresentableを見直したり、RawRepresentable where RawValue == CodingKey制約を使わない？

※ ロスレス変換が必要になるような厳密さは不要である

型が`RawRepresentable`に準拠することは、その対象の型とその基になる`RawValue`型の間でロスレス変換されることを示す。この変換は、多くの場合、対象の型とその基になる表現の間、最も一般的にはraw値を持つ`enum`やオプションセットの間の「標準」変換。

対照的に、`CodingKey`との間の変換は「付属的」なものであり、エンコードおよびデコードプロセス内のみでの表現であることを期待する。`protocol CodingKeyRepresentable: RawRepresentable where RawValue == CodingKey`とした場合に必要とするような、型の標準的な基底表現が`CodingKey`であることを提案（期待）しない。同様に、`CodingKey`以外のraw値で既に`RawRepresentable`の型は、この方法だと準拠できない。今回の機能の大きな推進力は、`Int`および`String`をraw値に持つ`enum`を`Dictionary`の`CodingKey`として参加できるようにすること。

#### なぜCodingKeyのassociated typeを使わない？

※ 型チェックのコストと天秤にかけた場合にメリットが上回らない

<details>
<summary>詳細を知りたい場合はクリック</summary>

これはpitch時に提案されており、下記のようなユースケースでは完全に妥当である。

```swift
enum MyKey: Int, CodingKey {
    case a = 1
    case b = 3
    case c = 5

    var intValue: Int? { rawValue }

    var stringValue: String {
        switch self {
        case .a: return "a"
        case .b: return "b"
        case .c: return "c"
        }
    }

    init?(intValue: Int) { self.init(rawValue: intValue) }

    init?(stringValue: String) {
        guard let rawValue = RawValue(stringValue) else { return nil }
        self.init(rawValue: rawValue)
    }
}

struct MyCustomType: CodingKeyRepresentable {
    typealias CodingKey = MyKey

    var useB = false

    var codingKey: CodingKey {
        useB ? .b : .a
    }

    init?(codingKey: CodingKey) {
        switch codingKey {
        case .a: useB = false
        case .b: useB = true
        case .c: return nil // .c is unsupported
        }
    }
}
```

この提案の分析は、利用側でキー値を引き出すために型消去を行うためのコストがゼロではなく、そのコストが見合わないことを示唆している。https://forums.swift.org/t/pitch-allow-coding-of-non-string-int-keyed-dictionary-into-a-keyedcontainer/44593/9

associated typeは利用側でゼロではないコストがかかるため(例えば、キーの型を使用して`CodingKeyRepresentable`に準拠しているかをチェックする)、associated typeにそのコストに見合うメリットが必要になる。名前が`CodingKeyRepresentable`だが、`CodingKeyRepresentable`と`RawRepresentable`の主な違いは、`RawValue`型の識別子が`RawRepresentable`にとって重要な一方、`CodingKeyRepresentable`はそうでもない。

`CodingKeyRepresentable.codingKey`の利用側(例えば、`Dictionary`)では、キー型の識別子が必ずしも十分に役立つとは思えない:

- `.codingKey`の主な用途は、基になる`String`/`Int`値の即時取得。`Dictionary`はそれらの値をすぐに使用できるように引き出し、元のキーを破棄する
- 非ジェネリックコンテキスト(または`CodingKeyRepresentable`への準拠を前提としないコンテキスト)では、キーの型を意味のある形で取得することはできない。キーキーの値を取得するために型消去したとしても、型付けされたキーを渡すことができない(そして、その苦痛は、型消去を行うには、これを実行したいすべての利用側で実装(車輪の再発明)を行い、それを実行するための別のプロトコルを追加する必要がある。`Optional`の場合は数回実行する必要があり、これは嬉しくない)
- 具体的なキーの型を取得する必要がある場合でも、この機能の主なユースケースは、列挙型ではない動的な値のキーを提供することになると思われる(例えば、`UUID`のような`struct`(実際これはできない)。これらの型の場合、必ずしも`CodingKey`の`enum`を定義できるとは限らない。代わりに、`AnyCodingKey`(定義上識別子を持たない)などのより一般的なキーの型を使用することを推奨する

作成側(例えば`MyCustomType`)では、十分な有用性が必ずしもあるかどうかもわからない。一般に、`CodingKeyRepresentable`の大部分で実際に気にするのはキーの`String`/`Int`の値のみだと思われる。といういのも、これらは動的に初期化されると思われるため(ここでも、`CodingKey.stringValue`からの`UUID`の初期化を考えている—これは任意の`CodingKey`から実行できる)

上記の`MyKey`の例はマイナーなユースケースで、`associatedtype`の制約なしでも表現できる:

```swift
enum MyKey: Int, CodingKey {
    case a = 1, b = 3, c = 5

    // There are several ways to express this, just an example:
    init?(codingKey: CodingKey) {
        if let key = codingKey.intValue.flatMap(Self.init(intValue:)) {
            self = key
        } else if let key = Self(stringValue: codingKey.stringValue) {
            self = key
        } else {
            return nil
        }
    }
}

struct MyCustomType: CodingKeyRepresentable {
    var useB = false

    var codingKey: CodingKey {
        useB ? MyKey.b : MyKey.a
    }

    init?(codingKey: CodingKey) {
        switch MyKey(codingKey: codingKey) {
        case .a: useB = false
        case .b: useB = true
        default: return nil
        }
    }
}
```

これも同様に表現力豊かだと思われ、特に`enum`ではない型を念頭に置いて、associated typeを必要としないことで、大きな損失なしに柔軟性が向上すると考えられる。

</details>

#### Encoder/Decoderにワークアラウンドを追加する

`DictionaryKeyEncodingStrategy`を`JSONEncoder`に追加しようとしてみた。https://github.com/apple/swift/pull/26257

新しいエンコード/デコードの「strategy」を提供することにより、`JSONEncoder`および`JSONDecoder`型で新しい動作へのオプトインを直接表現できるようにするというアイデアがあったが、問題は特定の`Encoder`/`Decoder`のペアだけでなく、すべての問題を修正する必要があると思われる。

#### `newtype`の設計を待つ

[Tagged](https://github.com/pointfreeco/swift-tagged)ライブラリが解決する問題を基本的に解決しようとする`newtype`のデザインについての言及を聞いたことがある。つまり、他のプリミティブ型の周りに型安全なWrapperを作成する。決してこれの専門家ではなく、これがどのように実装されるかはわからないが、`SomeType`が`String`の`newtype`であることがわかった場合、これを使用して、`Dictionary`の新しい実装に`Codable`への準拠に提供できる。この機能は古いバージョンのSwiftには存在しないため(これがSwiftランタイムの変更を必要とする機能である場合)、これを`Dictionary`に追加しても、`Codable`への準拠が動作を損なうことはない。

しかし、これらは非常に多くのifとbutがあり、人々が遭遇しているように見える問題の1つ(`RawRepresentable`のWrapperの問題)を解決するだけで、例えば文字列ベースの列挙型や`Int8`ベースのキーは解決しない。

#### 何もしない

もちろん、エンコード中にこの状況を手動で処理することは可能。

この状況に対応するためのやや邪魔にならない方法は、[CodableKey](https://forums.swift.org/t/bug-or-pebkac/33796/12)で提案されているようにProperty Wrapperを使用すること。

この解決策は各`Dictionary`に適用する必要があり、非常に洗練された回避策。しかし標準ライブラリで修正できる可能性のある問題のワークアラウンドでもある。

Property Wrapperの解決策のいくつかの欠点は、Pitch段階で発覚した:

- `Int8`(またはその他の標準ライブラリの数値の型)をキーとして使用するには、`CodingKey`に準拠している必要がある。この準拠は、例えばSwift Packageなど他の場所での準拠の衝突を防ぐために、標準ライブラリで行う必要がある。そして私見では、これらの型は`CodingKey`への準拠を提供するべきではない。
- 単純にエンコード/デコードするのは簡単ではない。例えば、別の`Codable`型のプロパティではない`Dictionary<Int8, String>`(上記のリンクされた投稿の中の例でも言及されている)。
- すでに定義されているオブジェクトに`Codable`への準拠を追加することはできない。したがって、あるファイルに`Dictionary<Int8, String>`を持つstruct(`MyType`)を定義した場合、`extension MyType: Codable {/ * ... * /}`を別のファイルに単純に配置することはできない。

## 参考リンク

### Forums

- [Dictionarys encoding strategy](https://forums.swift.org/t/dictionarys-encoding-strategy/11973)
- [JSON Encoding / Decoding weird encoding of dictionary with enum values](https://forums.swift.org/t/json-encoding-decoding-weird-encoding-of-dictionary-with-enum-values/12995)
- [Bug or PEBKAC](https://forums.swift.org/t/bug-or-pebkac/33796)
- [Using RawRepresentable String and Int keys for Codable Dictionaries](https://forums.swift.org/t/using-rawrepresentable-string-and-int-keys-for-codable-dictionaries/26899)
- [[Pitch] Allow coding of non-String/Int keyed Dictionary into a KeyedContainer](https://forums.swift.org/t/pitch-allow-coding-of-non-string-int-keyed-dictionary-into-a-keyedcontainer/44593)
- [SE-0320: Coding of non String / Int keyed Dictionary into a KeyedContainer](https://forums.swift.org/t/se-0320-coding-of-non-string-int-keyed-dictionary-into-a-keyedcontainer/50903)
- [SE-0320(2nd review): Coding of non String / Int keyed Dictionary into a KeyedContainer](https://forums.swift.org/t/se-0320-2nd-review-coding-of-non-string-int-keyed-dictionary-into-a-keyedcontainer/51710)
- [[Accepted] SE-0320: Coding of non String / Int keyed Dictionary into a KeyedContainer](https://forums.swift.org/t/accepted-se-0320-coding-of-non-string-int-keyed-dictionary-into-a-keyedcontainer/52057)

### プロポーザルドキュメント

- [Allow coding of non String / Int keyed Dictionary into a KeyedContainer](https://github.com/apple/swift-evolution/blob/main/proposals/0320-codingkeyrepresentable.md)

### 実装

- [Added protocol CodingKeyRepresentable](https://github.com/apple/swift/pull/34458)

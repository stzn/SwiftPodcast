# Swift エンコーディング検証付きStringのイニシャライザ

- [Swift エンコーディング検証付きStringのイニシャライザ](#swift-エンコーディング検証付きstringのイニシャライザ)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
    - [提案](#提案)
    - [詳細](#詳細)
    - [ソース互換性](#ソース互換性)
    - [ABI互換性](#abi互換性)
    - [導入の影響](#導入の影響)
    - [検討された代替案](#検討された代替案)
      - [引数ラベルでエンコーディングを指定するイニシャライザ](#引数ラベルでエンコーディングを指定するイニシャライザ)
      - [`String.init?(validating: some Sequence<Int8>)`が、`some Sequence<CChar>`として、または`CChar`の特定の`Collection`として型付けされたパラメータを取るようにする](#stringinitvalidating-some-sequenceint8がsome-sequenceccharとしてまたはccharの特定のcollectionとして型付けされたパラメータを取るようにする)
    - [将来の方向性](#将来の方向性)
      - [検証失敗に関する情報を含むエラーをスローする](#検証失敗に関する情報を含むエラーをスローする)
      - [入力修復初期化の改善](#入力修復初期化の改善)
      - [正規化オプションの追加](#正規化オプションの追加)
      - [その他](#その他)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

エンコードされた入力を検証し、入力に無効な要素が含まれている場合は`nil`を返す、新しい`String`失敗可能イニシャライザを追加することを提案する。

## 内容

### 動機

`String`型は、整形されたUnicodeテキスト表現を保証する。テキストを表すデータがファイルやネットワーク、その他のソースから受信された場合、それを`String`に格納するかもしれないが、そのデータは最初に検証されなければならない。`String`はすでに、無効な要素を修復してデータを有効なUnicodeに変換する方法を提供しているが、特に信頼できないソースを扱う場合、そのような変換は望ましくないことがよくある。たとえば、JSONデコーダは入力を変換することができない。テキストを表すスパンに無効なUTF-8が含まれる場合、失敗しなければならない。

この機能は、標準ライブラリから直接利用することはできない。既存のパブリックAPIを使用して構成することは可能だが、余分なメモリコピーと割り当てが必要になる。標準ライブラリは、この機能をパフォーマンスがよい方法で実装できるユニークな立場にある。

### 提案

引数が`Unicode.Encoding`に準拠する型パラメータで表されるエンコーディング形式に変換しようとしても無効であった場合、`nil`を返す失敗可能な新しい`String`イニシャライザを追加する。

```swift
extension String {
  public init?<Encoding: Unicode.Encoding>(
    validating codeUnits: some Sequence<Encoding.CodeUnit>,
    as: Encoding.Type
  )
}
```

C言語から取得したデータを処理する場合、UTF-8データが`UInt8`ではなく`Int8`(通常は`CChar`)で表現されることがよくある。このような場合に便利なイニシャライザを提供する:

```swift

extension String {
  public init?<Encoding: Unicode.Encoding>(
    validating codeUnits: some Sequence<Int8>,
    as: Encoding.Type
  ) where Encoding.CodeUnit == UInt8
}
```

`String`には、Cとの相互運用性を考慮した、UTF-8入力の検証用イニシャライザがすでに用意されている。その引数のラベルは、入力がヌル終端C文字列であることを想定していることを伝えておらず、これがエラーの原因となっていた。この前提条件を明確にするために、ラベルを変更することを提案する:

```swift
extension String {
  public init?(validatingCString nullTerminatedUTF8: UnsafePointer<CChar>)

  @available(Swift 5.XLIX, deprecated, renamed: "String.init(validatingCString:)")
  public init?(validatingUTF8 cString: UnsafePointer<CChar>)
}
```

`String.init?(validatingCString:)`とは異なり、`String.init?(validating:as:)`イニシャライザは、埋め込まれているすべての`\0`コードユニットを含む入力全体を変換することに注意。

### 詳細

私たちは、これらの新しいイニシャライザが高性能であることを望んでいる。そのため、これらの実装では、必要なメモリ割り当てとコピーの回数を最小限に抑える必要がある。私たちは、`withContiguousStorageIfAvailable`を活用した`@inlinable`実装によって、検証ケースに対する具体的な(内部的な)コードパスを提供することで、このパフォーマンスを実現する。具体的な内部イニシャライザは、標準ライブラリ内部の関数を呼び出す。

```swift
extension String {
  /// 指定されたエンコーディングに従って渡された一連のコードユニットをコピーして検証することで
  /// 新しい`String`を作成する。
  ///
  /// このイニシャライザは、不正な形式のコードユニットシーケンスを修復しようとはしない。
  /// もし見つかった場合、イニシャライザの結果は`nil`となる。
  ///
  /// 次の例では、2つの異なるシーケンスでこのイニシャライザを呼び出している。
  /// 最初は正しい整形式のUTF-8コードユニットシーケンスで
  /// 次に不正な整形式のUTF-16コードユニットシーケンス、このイニシャライザを呼び出す。
  ///
  ///     let validUTF8: [UInt8] = [67, 97, 0, 102, 195, 169]
  ///     let valid = String(validating: validUTF8, as: UTF8.self)
  ///     print(valid)
  ///     // Prints "Optional("Café")"
  ///
  ///     let invalidUTF16: [UInt16] = [0x41, 0x42, 0xd801]
  ///     let invalid = String(validating: invalidUTF16, as: UTF16.self)
  ///     print(invalid)
  ///     // Prints "nil"
  ///
  /// - Parameters:
  ///   - codeUnits: `String`を符号化する一連のコードユニット
  ///   - encoding: 使用する `Unicode.Encoding` に準拠するもの
  ///
  @inlinable
  public init?<Encoding>(
    validating codeUnits: some Sequence<Encoding.CodeUnit>,
    as encoding: Encoding.Type
  ) where Encoding: Unicode.Encoding

  /// 指定されたエンコーディングに従って、渡された`Int8`のシーケンスをコピーして検証することで、
  /// 新しい`String`を作成する。
  ///
  /// このイニシャライザは、不正な形式のコードユニットシーケンスを修復しようとはしない。
  /// もし見つかった場合、イニシャライザの結果は`nil`となる。
  ///
  /// 次の例では、2つの異なるシーケンスでこのイニシャライザを呼び出す
  /// 最初は整形式のUTF-8コードユニットシーケンスで、
  /// 次に不正なASCIIコードユニットシーケンスで、このイニシャライザを呼び出す。
  ///
  ///     let validUTF8: [Int8] = [67, 97, 0, 102, -61, -87]
  ///     let valid = String(validating: validUTF8, as: UTF8.self)
  ///     print(valid)
  ///     // Prints "Optional("Café")"
  ///
  ///     let invalidASCII: [Int8] = [67, 97, -5]
  ///     let invalid = String(validating: invalidASCII, as: Unicode.ASCII.self)
  ///     print(invalid)
  ///     // Prints "nil"
  ///
  /// - Parameters:
  ///   - codeUnits: `String`を符号化する一連のコードユニット
  ///   - encoding: codeUnits`を`UInt8`としてデコードできる`Unicode.Encoding`に準拠するもの
  @inlinable
  public init?<Encoding>(
    validating codeUnits: some Sequence<Int8>,
    as encoding: Encoding.Type
  ) where Encoding: Unicode.Encoding, Encoding.CodeUnit == UInt8
}
```

```swift
extension String {
  /// 与えられたポインタによって参照されるヌル終端UTF-8データをコピーし、
  /// 検証することによって新しい文字列を作成する。
  ///
  /// このイニシャライザは、不正な形式のUTF-8コードユニットシーケンスを修復しようとはしない。
  /// もし見つかった場合、イニシャライザの結果は`nil`となる。
  ///
  /// 以下の例では、2つの異なる`CChar`シーケンスへのポインタを指定してこのイニシャライザを呼び出している。
  /// 最初のシーケンスは整形式のUTF-8コードユニットシーケンスで、
  /// 2番目のシーケンスは末尾に不正なUTF-8コードユニットシーケンスがある。
  ///
  ///     let validUTF8: [CChar] = [67, 97, 102, -61, -87, 0]
  ///     validUTF8.withUnsafeBufferPointer { ptr in
  ///         let s = String(validatingCString: ptr.baseAddress!)
  ///         print(s)
  ///     }
  ///     // Prints "Optional("Café")"
  ///
  ///     let invalidUTF8: [CChar] = [67, 97, 102, -61, 0]
  ///     invalidUTF8.withUnsafeBufferPointer { ptr in
  ///         let s = String(validatingCString: ptr.baseAddress!)
  ///         print(s)
  ///     }
  ///     // Prints "nil"
  ///
  /// - Parameter nullTerminatedUTF8: A pointer to a null-terminated UTF-8 code sequence.
  @_silgen_name("sSS14validatingUTF8SSSgSPys4Int8VG_tcfC")
  public init?(validatingCString nullTerminatedUTF8: UnsafePointer<CChar>)
  
  @available(*, deprecated, renamed: "String.init(validatingCString:)")
  @_silgen_name("_swift_stdlib_legacy_String_validatingUTF8")
  @_alwaysEmitIntoClient
  public init?(validatingUTF8 cString: UnsafePointer<CChar>)
}
```
### ソース互換性

この提案は、定義上ソースと互換性のある追加を中心に構成されている。

この提案には、1つの関数の名前を`String.init?(validatingUTF8:)`から`String.init?(validatingCString:)`に変更することが含まれている。既存の関数名は非推奨となり、警告が表示される。fixitは、リネームされたバージョンの関数への簡単な移行をサポートする。

### ABI互換性

この提案は、ABIに新しい関数を追加するものである。

改名された関数は既存のABIエントリーポイントを再利用するため、ABI互換の変更となる。

### 導入の影響

新しい標準ライブラリのバージョンが必要

### 検討された代替案

#### 引数ラベルでエンコーディングを指定するイニシャライザ

最も一般的なケースでの利便性と発見しやすさのために、私たちは当初、引数ラベルの一部としてUTF-8入力エンコーディングを指定するイニシャライザを提案しました：

```swift
extension String {
  public init?(validatingAsUTF8 codeUnits: some Sequence<UTF8.CodeUnit>)
}
```

レビュアーと言語運営グループは、このイニシャライザはあまり重要な改善をもらたさず、これが目指す発見しやすさの問題は、ツールの改善によって解決するのが最善であると考えた。

#### `String.init?(validating: some Sequence<Int8>)`が、`some Sequence<CChar>`として、または`CChar`の特定の`Collection`として型付けされたパラメータを取るようにする

この検証するイニシャライザを`Sequence<CChar>`で定義すると、`CChar`が`Int8`ではなく`UInt8`にタイプエイリアスされるプラットフォームでは、コンパイル時に曖昧さが生じる。レビューされた提案では、`UnsafeBufferPointer<CChar>`で定義することが提案された。この問題の実際の根源は、`CChar`が独立した型ではなくタイプエイリアスであることである。この点を考慮し、レビュー期間中および言語運営グループでの議論により、このイニシャライザは`Sequence<Int8>`を使用して再定義されることになった。これにより、ソースコードレベルでの`CChar`対U`Int8`の相互運用性の問題が解決され、曖昧さを排除して柔軟性をできるだけ保つことができる。

### 将来の方向性

#### 検証失敗に関する情報を含むエラーをスローする

バイトストリームをデコードする際に、検証失敗の詳細を取得することは、問題を診断するために有用である。私たちはこの機能を提供したいが、現在の入力検証機能はそれに適していない。これは将来の改良点として残されている。

#### 入力修復初期化の改善

標準ライブラリには入力値を修復して初期化する用のイニシャライザが1つしかなく、発見しにくいという問題がある。UTF-8エンコーディングに特化した、より発見しやすいバージョンを追加することができる。

#### 正規化オプションの追加

文字列を正規化することはしばしば望ましいが、標準ライブラリはそのためのパブリックAPIを公開していない。正規化を行うイニシャライザや、正規化を行うmutating関数を追加することができる。

#### その他

- `Sequence<UnicodeScalar>`から`String`を生成する（失敗しない）イニシャライザを追加する
- 入力検証に特化したAPIを追加する

## 参考リンク

### Forums

- [pitch](https://forums.swift.org/t/66206)
- [review](https://forums.swift.org/t/se-0405-string-initializers-with-encoding-validation/66655)
- [acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0405-string-initializers-with-encoding-validation/67134)

### プロポーザルドキュメント

- [String Initializers with Encoding Validation](https://github.com/apple/swift-evolution/blob/main/proposals/0405-string-validating-initializers.md)
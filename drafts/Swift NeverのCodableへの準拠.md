# Swift NeverのCodableへの準拠

- [Swift NeverのCodableへの準拠](#swift-neverのcodableへの準拠)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
    - [提案](#提案)
    - [詳細](#詳細)
    - [ソース互換性](#ソース互換性)
    - [ABI互換性](#abi互換性)
    - [導入の影響](#導入の影響)
  - [検討された代替案](#検討された代替案)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

`Codable`として知られる`Encodable`および`Decodable`プロトコルに準拠するように`Never`を拡張する。

## 内容

### 動機

Swiftは、`Codable`メンバを持つ任意の型のための`Codable`への準拠性を合成することができる。ジェネリック型は、多くの場合、この`Either`型のように、ジェネリックパラメータを制約することによって、この合成された準拠性に参加する

```swift
enum Either<A, B> {
    case left(A)
    case right(B)
}

extension Either: Codable where A: Codable, B: Codable {}
```

この方法では、両方のジェネリックパラメータが`Codable`である`Either`インスタンスは、`Either<Int, Double>`のように、それ自体が`Codable`になります。しかし、`Never`は`Codable`ではないので、`Either<Int, Never>`のような型をエンコードまたはデコードするのはまったく問題ないにもかかわらず、`Never`をパラメータの1つとして使用すると、Conditional Conformanceできなくなる。

### 提案

標準ライブラリは、`Never`型に`Encodable`と`Decodable`の準拠性を追加する。

### 詳細

`Encodable`への準拠性は単純で、`Never`インスタンスを持つことは不可能であるため、`encode(to:)`メソッドは単に空にすることができる。

`Decodable`プロトコルは、`init(from:)`イニシャライザを必要とするが、これは明らかに`Never`インスタンスを生成できない。無効な入力をデコードしようとすることは、プログラマのエラーではないので、致命的なエラーは不適切である。その代わりに、デコードが試みられると、実装は`DecodingError.typeMismatch`エラーを投げる。

### ソース互換性

既存のコードがすでに`Codable`の準拠性を宣言している場合、そのコードは警告を出す。たとえば、プロトコル`Encodable`に対する`Never`の準拠性は、型のモジュール`Swift`ですでに宣言されている。

`Never`のインスタンスを構築することはできないので、新しい準拠性は既存の準拠性と異なるべきではありません。

### ABI互換性

影響なし

### 導入の影響

新しい準拠にはavailabilityアノテーションが追加される。

## 検討された代替案

この提案の以前のイテレーションでは、`Decodable`イニシャライザからスローされるエラーとして`DecodingError.dataCorrupted`を使用していた。このエラーは通常、不正なJSONなどの構文エラーに使用されるため、`typeMismatch`エラーが代わりに使用されるようになった。この目的のためにカスタムエラー型を作成することもできるが、`typeMismatch`はすでにドキュメント化されたAPIとして存在し、開発者がエラーを理解するために必要な情報を提供しているため、ここではその使用が適切であるとした。

## 参考リンク

### Forums

- [pitch](https://forums.swift.org/t/pitch-conform-never-to-codable/64056)
- [review](https://forums.swift.org/t/se-0396-conform-never-to-codable/64469)
- [acceptance](https://forums.swift.org/t/accepted-se-0396-conform-never-to-codable/64848)

### プロポーザルドキュメント

- [Conform `Never` to `Codable`](https://github.com/apple/swift-evolution/blob/main/proposals/0396-never-codable.md)
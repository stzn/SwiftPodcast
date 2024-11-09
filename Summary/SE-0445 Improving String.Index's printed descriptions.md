# Improving String.Index's printed descriptions

バージョン: Swift 6.1

## 内容

`String.Index`のデバッグ出力を改善する。

`CustomDebugStringConvertible`に準拠させる。

現状の出力は何を表しているのかわかりにくい。

```swift
let string = "👋🏼 Helló"

print(string.startIndex) // ⟹ Index(_rawBits: 15)
print(string.endIndex) // ⟹ Index(_rawBits: 983047)
print(string.utf16.index(after: string.startIndex)) // ⟹ Index(_rawBits: 16388)
print(string.firstRange(of: "ell")!) // ⟹ Index(_rawBits: 655623)..<Index(_rawBits: 852487)
```

これが以下のように改善される。

```swift
let string = "👋🏼 Helló"

// どのエンコーディングでもオフセット0なのでanyとなっている
print(string.startIndex) // ⟹ 0[any] 
print(string.endIndex) // ⟹ 15[utf8]
// UTF-8エンコーディングのオフセット0から1進んだ位置
// 👋🏼はUTF-16では D83D DC4Bなので、DC4Bを指す
print(string.utf16.index(after: string.startIndex)) // ⟹ 0[utf8]+1 
print(string.firstRange(of: "ell")!) // ⟹ 10[utf8]..<13[utf8]
```

注意: 上記はあくまで例であって、Swiftのバージョンアップによって変わることがある(上記は[Swift5.8で導入されたLLDBの出力フォーマット](https://github.com/swiftlang/llvm-project/pull/5515))。目的はわかりやすい出力をすることなのでそこまで問題はない。

## 補足

- `String`の`Index`は文字列の先頭からのオフセットを表し、文字列のエンコーディングに応じてUTF-8またはUTF-16コードユニットを参照する。(Swiftの文字列はUTF-8エンコーディングを使用しているが、Objective-Cからブリッジされた文字列(`NSString`)はUTF-16エンコーディングを使用している場合もある)

- `CustomDebugStringConvertible`への準拠自体はバックデプロイできないため、OSのバージョン制限がある


## プロポーザルリンク

- [https://github.com/swiftlang/swift-evolution/blob/main/proposals/0445-string-index-printing.md](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0445-string-index-printing.md)

## Forums

- [acceptance](https://forums.swift.org/t/accepted-with-modifications-se-0445-improving-string-index-s-printed-descriptions/75108)

- [review](https://forums.swift.org/t/se-0445-improving-string-indexs-printed-descriptions/74643)

- [pitch](https://forums.swift.org/t/improving-string-index-s-printed-descriptions/57027)

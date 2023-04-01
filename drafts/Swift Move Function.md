# 変数バインディングのライフタイムを終了させる`consume`演算子

- [変数バインディングのライフタイムを終了させる`consume`演算子](#変数バインディングのライフタイムを終了させるconsume演算子)
  - [概要](#概要)
  - [動機](#動機)
  - [提案内容: `consume`演算子](#提案内容-consume演算子)
  - [詳細](#詳細)
  - [ソース互換性](#ソース互換性)
  - [ABIへの影響](#abiへの影響)
  - [APIレジリエンスへの影響](#apiレジリエンスへの影響)
  - [代替案](#代替案)
  - [将来の検討事項](#将来の検討事項)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要


`consume`は、特定のローカル`let`、ローカル`var`、または関数のパラメータのライフタイムを終了させ、`consume`後の使用時にコンパイラに診断を出させることでこれを強制します。これにより、パフォーマンスや正しさのために値の所有権を転送することに依存しているコードで、その要件をコンパイラと人間の読者に伝えることができます。たとえば、次のようになります:

```swift
useX(x) // ローカル変数xで何かする

// xのライフタイムを終了し、yのライフタイムが始まる
let y = consume x // [1]

useY(y) // ローカル変数yで何かする
useX(x) // エラー、xのライフタイムは[1]で終了している


// yのライフタイムを終わらせ、現在値を破棄する
_ = consume y // [2]
useX(x) // エラー、xのライフタイムは[1]で終了している
useY(y) // エラー、yのライフタイムは[2]で終了している
```
## 動機

Swiftは、参照カウントとコピーオンライトを使用して、開発者が値セマンティクスを持つコードを書くことができ、通常はパフォーマンスやメモリ管理についてあまり気にならないようにしている。しかし、パフォーマンスに敏感なコードでは、開発者は、COWデータ構造の一意性を制御し、言語実装やソースコードへの変更に対してこの先も保証される方法で、retain/releaseの呼び出しを減らすことができるようにしたいと考えている。次のような例を考えてみる。

```swift
func test() {
  var x: [Int] = getArray()
  
  // xに値が追加される。この時点で、xが一意であることがわかる。このプロパティを保持したい
  x.append(5)
  
  // xの現在値を他の関数に渡し、その一意性を利用して、さらに効率的に変異させることができる
  doStuffUniquely(with: x)

  // xを新しい値にリセットする。古い値はもう使わないので、呼び出し元から一意に参照されていた可能性がある
  x = []
  doMoreStuff(with: &x)
}

func doStuffUniquely(with value: [Int]) {
  // `value` への最後の参照を受け取った場合、コピー回数を増やさずに効率よく更新できるようにしたい
  var newValue = value
  newValue.append(42)

  process(newValue)
}
```

上記の例では、変数`x`に値が蓄積され、`doStuffUniquely(with:)`に渡され、受け取った値にさらに変更が加えられている。`doStuffUniquely`はもはや値をそのまま使用しないので、呼び出し元が`x`の値の所有権を`doStuffUniquely`に転送し、配列バッファの不要なretain/release、配列内容の不要なCOWを回避できるようにする必要がある。`doStuffUniquely`は、そのパラメータをローカルの可変変数に移動して一意のバッファを代わりに変更できるはず。コンパイラが自動的にこれらの最適化を行うこともできるが、最適な結果を得るためには、多くの分析が必要。プログラマは、このような一連の最適化が行われることを保証し、最適化の妨げとなるようなコードを書いた場合にはエラーを出力したいと思うかもしれない。

Swift-evolution pitch threads:
    https://forums.swift.org/t/pitch-move-function-use-after-move-diagnostic
    https://forums.swift.org/t/selective-control-of-implicit-copying-behavior-take-borrow-and-copy-operators-noimplicitcopy/60168


## 提案内容: `consume`演算子

`consume`は、**静的なライフタイムにバインディングされたものの現在の値**を消費する。このバインディングは、エスケープされていないローカル`let`、エスケープされていないローカル`var`、または関数パラメータで使用できるが、プロパティラッパーやget/set/read/modify/などへのアクセサには適用されない。そして、現在値がローカルで再び使用されないことをコンパイラが保証する。もし、そのような使い方がされた場合、コンパイラはエラーを出力する。前の例を修正して、`doStuffUniquely(with:)`に渡すときに、`consume`を使って`x`の現在の値のライフタイムを明示的に終了させることができる:

```swift
func test() {
  var x： [Int] = getArray()
  
  // xに値が追加される。この時点で、xが一意であることがわかる。このプロパティを保持したい
  x.append(5)
  
  // 現在のxの値を別の関数に渡す
  doStuffUniquely(with: consume x)

  // 古い値はもう使わないので、xを新しい値にリセットする。
  x = []
  doMoreStuff(with: &x)
}
```
`consume x`演算子の構文は、提案されている[所有権修飾](https://forums.swift.org/t/borrow-and-take-parameter-ownership-modifiers/59581)パラメータの構文`(x: consume T)`を意図的に反映している。`doStuffUniquely(with:)`は、`consume`パラメータ修飾子と組み合わせて、値を変更するためにパラメータを自身のローカル変数に移動する際に、パラメータの一意性を保持するために`consume`演算子を使用できる:

```swift
func doStuffUniquely(with value: consume [Int]) {
  // `value`の残った最後の参照を受け取った場合、さらにコピーを発生させることなく、効率的に更新できるようにしたい
  var newValue = consume value
  newValue.append(42)

  process(newValue)
}
```

テスト関数では、再割り当て前の`x`の最終値が明示的に`doStuffUniquely(with:)`に渡され、その時点で呼び出し先が値の一意の所有権を受け取り、呼び出し元が古い値を再び使用できないことを保証している。`doStuffUniquely(with:)`の内部では、ローカル変数`newValue`を初期化するために不変の`value`パラメータのライフタイムを終了し、割り当てがコピーを引き起こさないことを保証している。さらに、将来のメンテナンス担当者がこの所有権移転の連鎖を断ち切るような形でコードを修正した場合、コンパイラはエラーを出力する。例えば、`x`が消費された後、再代入される前に、メンテナンス担当者が`x`をさらに使用した場合、エラーが表示される:

```swift
func test() {
  var x: [Int] = getArray()
  x.append(5)
  
  doStuffUniquely(with: consume x)

  // エラー: consume後に'x'が使われた
  doStuffInvalidly(with: x)

  x = []
  doMoreStuff(with: &x)
}
```
同様に、メンテナンス担当者が`newValue`を初期化するために`doStuffUniquely`の内部で元の`value`パラメータが`consume`された後にアクセスしようとすると、エラーが発生する:

```swift
func doStuffUniquely(with value: consume [Int]) {
  // `value` への最後の参照を受け取った場合、より多くのコピーを発生させることなく効率的に更新できるようにしたい
  var newValue = consume value
  newValue.append(42)

  process(newValue)
}
```

`consume`は、ローカルの`let`の不変バインディングのライフタイムを終了させることもできる。また、`consume`は値ではなくバインディングを操作することに注意が必要。定数`x`と同じ値を持つ別のローカル定数`other`を宣言した場合、次のように`x`から値を消費した後も`other`を使用することができる:

```swift
func useX(_ x: SomeClassType) -> () {}

func f() {
  let x = ...
  useX(x)
  let other = x   // otherはxのライフタイムを延ばすために使われる新しいバインディング
  _ = consume x // xのライフタイムが尽きる
  useX(other)     // otherをここで使う・・・問題なし
  useX(other) // otherをここで使う・・・問題なし
}
```

また、`x`とは別に`other`を消費し、両方の変数に対して別々の診断を得ることもできる:

```swift
func useX(_ x: SomeClassType) -> () {}

func f() {
  let x = ...
  useX(x)
  let other = x
  _ = consume x
  useX(consume other)
  useX(other) // エラー: consume後に'other'が使われた
  useX(x) // エラー: consume後に'x'が使われた
}
```

`inout`関数パラメータも`consume`することができる。しかし、`inout`パラメータの最終値は呼び出し元に戻されるため、呼び出し元が戻る前に再割り当てする必要がある。そのため、`buffer`が戻る時点で値を持っていないため、これはエラーとなる:

```swift
func f(_ buffer: inout Buffer) { // error: consume後'buffer'が再割り当てされていない!
  let b = consume buffer         // note: ここでconsume
  b.deinitialize()
  // ... コードを書く ...
}                                // note: inout引数`buffer`を再割り当てせずに返す。
```

しかし、以下のコードを書くことで、`buffer`を再初期化することができる:

```swift
func f(_ buffer: inout Buffer) {
  let b = consume buffer
  b.deinitialize()
  // ... コードを書く ...
  // 関数終了前に`buffer`を再初期化したので、コンパイラは満足している
  buffer = getNewInstance()
}
```

`defer`は`consume`の後に`inout`や`var`を再初期化するために使うこともできる。だから、こうも書ける:

```swift
func f(_ buffer: inout Buffer) {
  let b = consume buffer
  // 終了する前にバッファが再初期化されていることを確認する
  defer { buffer = getNewInstance() }
  try b.deinitializeOrError()
  // ... コードを書く ...
}
```

## 詳細

実行時、`consume x`は式`x`と同様に`x`にバインディングされた現在の値として評価される。しかし、コンパイル時には、`consume`の存在により、引数の所有権は、指定された時点のバインディングからある場所に移される。そのため`consume`から到達可能なバインディングのその後の使用はすべてエラーとなる。`consume`するオペランドは、静的ライフタイムを持つバインディングへの参照であることが要求される。現在、静的ライフタイムを持つバインディングとして参照できる宣言の種類は次の通り:
- 直前の関数のローカル`let`定数
- 直前の関数のローカル`var`変数
- 直前の関数のパラメータの1つ、または
- `mutating`または`__consuming`メソッド内の`self`パラメータ

静的ライフタイムを持つバインディングは、以下の要件も満たす必要がある:
- `@escaping`クロージャやネストした関数でキャプチャできない
- プロパティラッパーには適用できない
- `get`、`set`、`didSet`、`willSet`、`_read`、`_modify`のようなアクセサに付けることはできない
- `async let`であってはならない

`consume`で使用できるオペランドのセットの拡張の可能性については、Future Directionsで説明する。静的ライフタイムを持つバインディングを参照しないオペランドで`consume`を使用するとエラーになる。
検証オペランドが与えられた場合、`consume`は、それが消費された後にバインディングへの他の参照が存在しないことを強制する。解析はフローに依存するので、条件付きで値のライフタイムを終了させることができる:

```swift
if condition {
  let y = consume x
  // ここでもうxは使えない！
  useX(x) // !ERROR！consume後に使用している
} else {
  // ここではxを使うことができる
  useX(x) // OK
}
// しかし、ここでxを使うことはできない
useX(x) // !ERROR！consume後に使用している
```

バインディングが`var`の場合、解析ではさらに、`var`を条件付きで再初期化し、再初期化によって使うことができる位置も決まる。次のような例を考えてみよう:

```swift
if condition {
  _ = consume x
  // ここでもうxは使えない！
  useX(x) // !ERROR！consume後に使用している
  x = newValue
  // しかし、xに新しい値を再代入したことで、varを再び使える
  useX(x) // OK
} else {
  // このパスでは消費されなかったので、ここでまだxを使うことができる
  useX(x) // OK
}
// `if`分岐でxを再初期化し、`else`分岐でxが消費されることはなかったので、ここでもxを使用することができる
useX(x) // OK
```

上記の例では、trueのブロックと`if`ブロックの後のコードの両方で`x`を使用することができる。これは、`if`を通過する両方の経路で`x`が有効な値で終了してから処理を行うためである。
`inout` パラメータの場合、解析は`var`の場合と同じだが、関数からのすべての終了(`return`または`throw`)がパラメータの使用とみなされることを除く。したがって、正しいコードは、`inout`パラメータが`consume`された後に再割り当てする必要がある。
結果を使用せずにバインディングで`consume`を使用すると、未使用の非`Void`結果を返す関数の呼び出しと同じように警告が発生する。値を使用せずに「ドロップ」するには、`consume`を明示的に`_`に代入することができる。

## ソース互換性

`consume`はコンテキストキーワードとして動作する。`consume`という名前の関数を呼び出す既存のコードとの干渉を避けるため、`consume`するオペランドは別の識別子で始まり、識別子または`postfix`式で構成されていなければならない:

```swift
consume x // OK
consume [1, 2, 3] // `consume`という名前のプロパティへの添え字アクセスであり、consume操作ではない
consume (x) // 関数 `consume` を呼び出し，consume操作は行わない
consume x.y.z // 構文的にはOK(ただし、x.y.zは現在意味論的に検証されていない)
consume x[0] // 構文的にはOK(ただし、x[0]は現在、意味論的に検証されていない)
consume x + y // (consume x) + y として解析する
```

## ABIへの影響

なし

## APIレジリエンスへの影響

なし

## 代替案

TBD

## 将来の検討事項

TBD

## 参考リンク

### Forums

- [pitch](https://forums.swift.org/t/pitch-move-function-use-after-move-diagnostic/53983)
- [first review](https://forums.swift.org/t/se-0366-move-function-use-after-move-diagnostic/59202)
- [returned for revision](https://forums.swift.org/t/returned-for-revision-se-0366-move-operation-use-after-move-diagnostic/59687)
- [second review](https://forums.swift.org/t/se-0366-second-review-take-operator-to-end-the-lifetime-of-a-variable-binding/61021)
- [third review](https://forums.swift.org/t/combined-se-0366-third-review-and-se-0377-second-review-rename-take-taking-to-consume-consuming/61904)
- [acceptance](https://forums.swift.org/t/accepted-se-0366-consume-operator-to-end-the-lifetime-of-a-variable-binding/62758)

### プロポーザルドキュメント

[consume operator to end the lifetime of a variable binding](https://github.com/apple/swift-evolution/blob/main/proposals/0366-move-function.md)

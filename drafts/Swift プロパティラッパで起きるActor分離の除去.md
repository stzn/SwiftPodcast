# プロパティラッパで起きるActor分離の除去

<!-- 最後にTable of Contentsを入れる -->

## 概要

[SE-0316: Global Actor](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md)では、`@MainActor`のようなアノテーションを導入して、型、関数、プロパティを特定のGlobal Actorに分離するようにした。また、Global Actorの分離を推論する方法について、さまざまなルールが導入された。そのルールの1つが以下である:

> Global Actorまたは`nonisolated`のいずれかを明示的にアノテーションしていない宣言は、いくつかの異なる場所からGlobal Actorの分離を推論することができる:
> 
> [...]
> 
> - Global Actorで修飾された`wrappedValue`を持つラップされたインスタンスプロパティを含む構造体やクラスは、そのプロパティラッパからActor分離を推論する:
> ```swift
> @propertyWrapper
> struct UIUpdating<Wrapped> {
>   @MainActor var wrappedValue: Wrapped
> }
> 
> struct CounterView { // @UIUpdatingから@MainActorが推論される
>   @UIUpdating var intValue: Int = 0
> }
> ```

この提案は、Swift 6言語モードでコンパイルする際に、この推論ルールを削除することを提案している。上記の例を考えると、Swift6では`CounterView`はもはや`@MainActor`分離を推論しないだろう。

## 動機

この特定の推論ルールは、多くのユーザーにとって驚きであり、明らかではない。一部の開発者は、Actor分離が適用されるとき、開発者にとって明白でないため、Swift Concurrencyモデルを理解するのに苦労している。何かが推論されるとき、それはユーザには見えず、それが理解を難しくしている。これは、AppleのSwiftUIフレームワークによって導入されたプロパティラッパを使用するときに頻繁に生じるが、そのframeworkに限定されるものではない。例えば、以下のようなものがある:

### SwiftUIを使った例

```swift
struct MyView: View {
  // `StateObject`はMainActorに分離された`wrappedValue`を持つ
  @StateObject private var model = Model()
  
  var body: some View {
    Text("Hello, \(model.name)")
      .onAppear { viewAppeared() }
  }

  // このメソッドは`＠MainActor`と推論される
  func viewAppeared() {
    updateUI()
  }
}

@MainActor func updateUI() { /* 何かする */ }
```

上記のコードは問題なくコンパイルできる。しかし、`@StateObject`を`@State`に変更すると、エラーが発生する:

```swift
-   @StateObject private var model = Model()
+   @State private var model = Model()
```

```swift
  func viewAppeared() {
    // error: nonisolatedな同期コンテキスト上で
    // MainActorに分離されたグローバル関数'updateUI()'を呼ぶ
    updateUI()
  }
```

`@StateObject`の`var model`を`@Stateのvar model`に変更すると、`viewAppeared()`がコンパイルを停止してしまう。あるプロパティの宣言を変更することで、*兄弟関数*のコンパイルが停止するのは当然といえば当然である。実際、1つのプロパティラッパを変更することで、`MyView`型全体の分離も変更された。

### SwiftUIを使用しない例

この問題は、SwiftUIに限ったことではありません。例えば、以下のようなものです:

```swift
// DBライブラリを使ったプロパティラッパ
@propertyWrapper
struct DBParameter<T> {
  @DatabaseActor public var wrappedValue: T
}

// `@DBParameter`を使ったことで`@DatabaseActor`に分離されていると推論される
struct DBConnection {
  @DBParameter private var connectionID: Int

  func executeQuery(_ query: String) -> [DBRow] { /* 実装はここ */ }
}


// 他のファイル

@DatabaseActor
func fetchOrdersFromDatabase() async -> [Order] {
  let connection = DBConnection()

  // 'connection'は`DatabaseActor`に分離されているため
  // ここで'await'が必要ない
  connection.executeQuery("...")
}
```

`DBConnection.connectionID`のプロパティラッパを削除すると、`DBConnection`の推論されたActor分離がなくなり、その結果、`fetchOrdersFromDatabase`がコンパイルに失敗する原因となる。**privateプロパティへの変更が、全く別のファイルでコンパイルエラーを引き起こすというのは、Swiftでは前例がないことである。**(プロパティラッパからそれらの含む型への)Actor分離の上方推論は、型内のprivateプロパティの効果について、もはや局所的に推論することができないことを意味する。その代わりに、「不気味な［気味の悪い］遠隔作用 」が発生する。

※ アインシュタインが量子もつれ(非局所相関)を評した言葉

### 実際に問題が発生する？

この動作は、コミュニティでかなり多くの混乱を引き起こしている。例えば、[このツイート](https://twitter.com/teilweise/status/1580105376913297409?s=61&t=hwuO4NDJK1aIxSntRwDuZw)、[このブログ記事](https://oleb.net/2022/swiftui-task-mainactor/)、そして[この全体のSwift Forumsのスレッド](https://forums.swift.org/t/reconsider-inference-of-global-actor-based-on-property-wrappers/60821)を見て欲しい。この1つの特定の呼びかけは、この推論がActor分離が意図した範囲を超えて「ウイルス的」になっているため、いくつかのケースでSwift Concurrencyを導入することを難しくしたと述べている[この投稿](https://forums.swift.org/t/reconsider-inference-of-global-actor-based-on-property-wrappers/60821)から来ている:

```swift
class MyContainer {
     let contained = Contained() // error: nonisolatedな同期コンテキスト上でMainActorに分離されたイニシャライザ'init()'を呼んでいる
}

class Contained {
    @OnMainThread var i = 1
}
```

`@OnMainThread`の作者は、このプロパティラッパを作成し、特定のプロパティがMainThreadに分離されていることを宣言することを意図しました。しかし、プロパティラッパ内で`@MainActor`を使用すると、それを含んだ型全体まで予期せず分離してしまうため、それを強制することができない。

このプロパティラッパに基づく上方推論がなぜ最初に提案されたのかは不明である。Global Actorのレビューでは、この点に関する議論は見当たらなかった。私たちは、それが最初にロールアウトしたときにSwiftUIの`@ObservedObject`と容易にやり取りすることを意図していたかもしれないと推論できます。しかし、それは実際に何かを大幅に容易にしたか明らかではない。より詳しく言うと、それは、型に単一のアノテーションを書くことから私たちを救うだけで、そのアノテーションによる損失は、[最小驚きの原則](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)違反になる。

## 内容

提案はシンプルである。Swift6の言語モードでは、型内で使用されるプロパティラッパは、その型のActor分離に影響しない。私たちは単にこの推論ステップを完全に無効にする。

Swift5言語モードでは、分離は現在と同様に推論し続ける。新しい動作は、`-enable-upcoming-feature DisableActorIsolationFromPropertyWrapperUsage`コンパイラフラグを使用して確認できる。

## 詳細

[ActorIsolationRequest.getIsolationFromWrappers()](https://github.com/apple/swift/blob/85d59d2e55e5e063c552c15f12a8abe933d8438a/lib/Sema/TypeCheckConcurrency.cpp#L3618)は、本提案で説明するAcotr分離の推論を実装している。その関数は、Swift6言語モードで実行されるとき、または上記のコンパイラフラグが渡されるとき、推論を生成しないように調整される。

## ソース互換性

この変更は、推論されたActor分離に依存していたコードが存在する可能性があるため、ソースの互換性を破る可能性をもたらす。そのようなコードは、現在のソースと互換性のある方法で、必要なGlobal Actorを明示的にアノテートすることができます。例えば、ある型が現在`@MainActor`の分離を持つと推論されている場合、ソース互換性の破壊を避けるために、今すぐその型に分離を明示的に宣言できる。(「検討された代替案」の警告に関する注記を参照すること)。

ソースの非互換性をライブラリの作者がソース互換性のある方法で緩和できるケースもあるかもしれない。例えば、AppleがSwiftUIの`View`プロトコルを`@MainActor-`分離にすることを選んだ場合、すべての準拠する型は、特定のプロパティラッパの使用に基づいて矛盾なく分離されるのではなく、一貫してMainActorに分離されるだろう。この提案は、この緩和が可能かもしれないことを指摘するだけで、それが必要であるかどうかについての推奨はしていない。

### ソースの互換性評価

この変更の実用的な影響を決定する努力で、私はこれらの変更を含むmacOSツールチェーンを使用して、(Swiftソース互換性ライブラリや他の場所から)さまざまなオープンソースのSwiftプロジェクトを評価した。私は、提案された変更の結果として、実際のソースの非互換性の事例を見つけなかった。ほとんどのオープンソースプロジェクトは、プロパティラッパを全く使用しないライブラリだが、私は、プロパティラッパを使用し、この変更によって影響を受ける可能性のあるいくつかのプロジェクトを特に探し出そうとした。その結果、以下のようなものがあった:

Project | Outcome | Notes
---|---|---
[ACNHBrowserUI](https://github.com/Dimillian/ACHNBrowserUI) | 完全に互換性あり | SwiftUIのプロパティラッパを使用している
[AlamoFire](https://github.com/Alamofire/Alamofire) | 完全に互換性あり | カスタムのプロパティラッパを使用しているが、Actor分離はしていない
[Day One (Mac)](https://dayoneapp.com) | 完全に互換性あり | SwiftUIのプロパティラッパを使用している. (オープンソースではない)
[Eureka](https://github.com/xmartlabs/Eureka) | 完全に互換性あり | プロパティラッパを使用していない
[NetNewsWire](https://github.com/Alamofire/Alamofire) | 完全に互換性あり | SwiftUIのプロパティラッパを使用している
[swift-nio](https://github.com/apple/swift-nio) | 完全に互換性あり | プロパティラッパを使用していない
[SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON) | 完全に互換性あり | プロパティラッパを使用していない
[XcodesApp](https://github.com/RobotsAndPencils/XcodesApp) | 完全に互換性あり | SwiftUIのプロパティラッパを使用している

上記の全ては、デフォルトで `Swift Concurrency Checking`の設定が**Minimal**だった。私がConcurrencyチェックレベルを**Targeted**に変更したとき、上記のすべては、提案された変更の有無にかかわらず、エラーなしでコンパイルし続けた。

Concurrencyチェックレベルを**Complete**に変更したところ、*今回提案した変更の有無にかかわらず*、上記のプロジェクトのほとんどでコンパイルエラーが発生した。今回提案した変更は、「Completeチェック」で若干の*追加の*エラーを引き起こした可能性があるが、そうでなければソース互換性があったはずのプロジェクトでソース互換性を破壊することはなかった。

## ABI Stabilityへの影響

型のActor分離は実行時の呼び出し規約に一切反映されないため、この変更はABI上安定している。

## APIレジリエンスへの影響

影響なし。

## 検討された代替案

### Swift5のプロパティラッパベースの推論について警告を出す

特定のケースでは、将来のSwiftリリースでコードが無効になる警告を作成する。(例えば、これはSwift 6でのSwift Concurrencyに計画されている変更で行われる)。私はこの方向性に沿ってSwift5の言語モードに警告を追加することを検討した:

```swift
// ⚠️ Warning: `MyView`は`@StateObject`を使用しているため、'@MainActor'分離を使用すると推論される。
// この推論はSwift 6では無くなる予定である。
//
// `@MainActor`を型に追加するとこの警告は消える。
struct MyView: View {
  @StateObject private var model = Model()

  var body: some View {
    Text("Hello")
  }
}
```

しかし、私は2つの問題を発見した:

1. これは、Swift6の言語モードの下で壊れないコードでさえ、多くの警告を発生させるだろう
2. 型を*分離することなく*、この警告を黙らせる方法がない。もし私が実際に型が分離されることを望んでいないなら、それを表現する方法がない。あなたはnonisolatedな型を宣言することはできない:

```swift
nonisolated // 🛑 Error: 'nonisolated'修飾子はこの宣言に適用できない。
struct MyView: View {
  /* ... */
}
```

ユーザがSwift6の新しい動作に合わせた方法で警告を黙らせることができないことを考えると、ここで警告を出すのは不適切だと思われる。

## 参考リンク

### Forums

- [[Pitch] Stop inferring actor isolation based on property wrapper usage](https://forums.swift.org/t/pitch-stop-inferring-actor-isolation-based-on-property-wrapper-usage/63262)
- [SE-0401: Remove Actor Isolation Inference caused by Property Wrappers](https://forums.swift.org/t/se-0401-remove-actor-isolation-inference-caused-by-property-wrappers/65618)

### プロポーザルドキュメント

- [Remove Actor Isolation Inference caused by Property Wrappers](https://github.com/apple/swift-evolution/blob/main/proposals/0401-remove-property-wrapper-isolation.md)
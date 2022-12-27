# Swift将来追加予定の言語改善への段階的な適用

- [Swift将来追加予定の言語改善への段階的な適用](#swift将来追加予定の言語改善への段階的な適用)
  - [概要](#概要)
  - [言語バージョンとツールバージョン](#言語バージョンとツールバージョン)
  - [導入バージョン](#導入バージョン)
  - [内容](#内容)
    - [プロポーザルでそれぞれの機能の識別子を定義する](#プロポーザルでそれぞれの機能の識別子を定義する)
    - [ソース上での機能の検知](#ソース上での機能の検知)
    - [実験的な機能の採用](#実験的な機能の採用)
  - [ソース互換性](#ソース互換性)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

Swift6では、以前の言語モード(Swift4.xおよびSwift5.x)ではデフォルトで有効にできなかったソース互換性への影響がたくさんある、言語の多くの改善が積み上がっている。これらの改善は、Swift6言語モードの背後にあるSwiftコンパイラに既に実装されているが、Swift6が言語モードとして利用可能になるまでユーザはアクセスできない。これらの改善を、より早く利用できるようにすることを検討すべき理由がいくつかある。

- 開発者は、Swift6が利用可能になるまで待たずに、これらの改善によるメリットをすぐに得たいと考えている
- これらの変更をSwift6より前に開発者が利用できることで、より多くの知見が得られ、必要に応じてSwift6をさらに改善することができる
- Swift6で変更されるものが積み上がると、モジュールによっては移行が面倒になる可能性があり、Swift4.x/5.xの内にこれらの言語の変更を1つずつ適用することで、移行をスムーズにすることができる

いくつかの提案では、移行に有利なソリューションを提供するオーダーメイドのソリューションがすでに導入されている。
- [SE-0337](https://github.com/apple/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md)は、Swift4.x/5.xで`Sendable`関連のチェックの警告を有効にするために`-warn-concurrency`フラグを追加した
- [SE-0354](https://github.com/apple/swift-evolution/blob/main/proposals/0354-regex-literals.md)は`-enable-bare-slash-regex`フラグを追加して`/.../`をそのまま使った正規表現構文を有効にします。
- プロポーザルの一部ではなかったが、[SE-0335](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md)の議論には、すべての存在型で`any`を必須にするコンパイラフラグの要件が含まれていた。

これらはすべて、既存のSwift4.x/5.xコードを、来たるSwift6での改善に事前に組み入れるという同じ趣向を持っている。

このプロポーザルでは、ソース互換性の理由でSwift6まで適用されない機能の断片を意図的かつ明示的に導入する。これは、Swift4.x/5.xコードベースでそのメリットを享受し、Swift6言語モードへの移行をスムーズにするために、Swift6の機能を1つずつ段階的に採用するための、直接的なパスを構築する。開発者は、新しいコンパイラフラグ`-enable-upcoming-feature X`を使用して、そのモジュールで、`X`という名前の特定の機能を有効にすることができる(複数可)。そして、次のメジャー言語バージョンに移行すると、`X`はその言語バージョンに含まれているため、このコンパイラフラグは無効になる。このように、upcoming feature flagは、次の主要なSwift言語バージョンまでしか有効化されず、その後消去されるため、言語を互換性のない独自の言語バージョンにフォークすることはない。

## 言語バージョンとツールバージョン

「Swiftバージョン」には 2 つの関連する異なる種類のバージョンがあり、便宜上一緒に扱うことがよくあるが、このプロポーザルでは両方のバージョンがそれぞれ別々に関係している。

- Swiftツールバージョン: コンパイラ自体のバージョン番号。例えば、Swift5.6コンパイラは 2022 年 3 月に導入された
- Swift言語バージョン: ソース互換性を提供する言語バージョン。例えば、Swiftバージョン5は、Swiftツールバージョン5.6でサポートされている最新の言語バージョン

Swiftツールは、複数のSwift言語バージョンをサポートしている。最近のすべてのバージョン(Swiftツール5.0以降)は、複数のSwift言語バージョンをサポートしており、現在は4、4.2、および5の3つのみ。ツールが進化しても、一つの言語バージョン内でソース互換性のない変更を行わないようにしている。これは、Swiftエボリューションのプロセスにも反映されている。既存のソースコードセマンティクスを変更したり、無効にしたりするプロポーザルは、通常、既存の言語モードでは受け入れられない。多くのプロポーザルは、既存の言語モード内でSwift言語を拡張する。例えば、`async/await`は、Swiftツール5.5で利用できるようになったが、すべての言語バージョン(4、4.2、5)でも利用できる。

このプロポーザルには、新しいSwift言語バージョン(例:6)まで導入を待機しているソース互換性のない変更が含まれる。Swiftツール6.0は、Swift6を正式に使用できる最初のツールになるだろう。そして、引き続きSwift4、4.2、および5をサポートするだろう。Swift4、4.2、および5。Swiftツール6.0または6.1などを使用するために、コードをSwift6に移行する必要はない。また、Swift6で作成されたコードは、Swift4、4.2、または5で作成されたコードと相互運用できる。

## 導入バージョン

Swift5.8

## 内容

新しいコンパイラフラグ`-enable-upcoming-feature X`を導入する(`X`は有効にする機能の名前)。各プロポーザルでは、`X`が何かを明示するため、その機能を有効にする方法は明白になっている。例えば、SE-0274 は`ConciseMagicFile`を使用できるようにしたため、`-enable-upcoming-feature ConciseMagicFile`を設定すると、セマンティクスの変更を有効にする。もちろん、複数の`-enable-upcoming-feature`フラグをコンパイラに渡して、複数の機能を有効にすることもできる。

認識できないupcoming featureに関してコンパイラは無視されるだろう。これにより、古いツールは、新しい機能を採用したSwiftコード用の新しいツールと同じコマンドラインを使用できるが、古いツールを引き続き使用するための適切なワークアラウンドがある。古いコンパイラでもコードを適切に解釈できるため、これが可能な場合もあれば、[ソースコード内の機能を検出する方法](https://github.com/apple/swift-evolution/blob/main/proposals/0362-piecemeal-future-features.md#feature-detection-in-source-code)が必要な場合もある。これについては、後のセクションで説明する。

一部の言語バージョンでは、すべてのupcoming featureがデフォルトで有効になっている。`-enable-upcoming-feature X`が指定されていて、言語バージョンがデフォルトで機能`X`を有効にしている場合、コンパイラはエラーを生成する。こうすることで、いつ機能が利用可能になるかが明確になり、以前の言語バージョンから進化し、機能を少しずつ採用し、その後、新しい言語バージョンに移行すると、プロジェクトとマニフェストがきれいに片付く。

### プロポーザルでそれぞれの機能の識別子を定義する

機能識別子を定義する新しいオプションフィールドを使用して、[Swiftプロポーザルのテンプレート](https://github.com/apple/swift-evolution/blob/main/proposal-templates/0000-swift-template.md)を修正する。

- **Feature identifier**:`UpperCamelCaseFeatureName`

Swift6まで部分的または全体的に延期している次のプロポーザルを、下記の機能識別子で修正する:

- [SE-0274 "Concise magic file names"](https://github.com/apple/swift-evolution/blob/main/proposals/0274-magic-file.md)(`ConciseMagicFile`)は、Swift6まで`#file`のセマンティクスの変更を延期する。この機能を有効にすると、`#file`が`#filePath`ではなく`#fileID`を意味するように変更される
- [SE-0286"Forward-scan matching for trailing closures"](https://github.com/apple/swift-evolution/blob/main/proposals/0286-forward-scan-trailing-closures.md)(`ForwardTrailingClosures`)は、Swift6まで、後続クロージャの「後方スキャンマッチング」ルールの削除を延期する。この機能を有効にすると、後方スキャンマッチングルールが削除される
- [SE-0335 "Introduce existential any"](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md)(`ExistentialAny`)は、Swift6まで、すべての存在型に`any`を使用する要件を延期する。この機能を有効にすると、すべての存在型に`any`が必要になる
- [SE-0337 "Incremental migration to concurrency checking"](https://github.com/apple/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md)(`StrictConcurrency`)は、Swift6までConcurrencyモデルの一部のチェックを延期する(Swift5.xでは、このフラグを使用すると警告を発する)。この機能を有効にすると、完全なConcurrencyチェックを実行する`-warn-concurrency`と同等の意味を持つ
- [SE-0352 "Implicitly Opened Existentials"](https://github.com/apple/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md)(`ImplicitOpenExistentials`)は、Swift5.xで十分に機能しているコードのセマンティクスを変更したくなかったため、Swift6で「存在型の暗黙的開示」をより多くのケースに拡張する。この機能を有効にすると、これらの追加のケースで「存在型の暗黙的開示」が実行される
- [SE-0354 "RegexLiterals"](https://github.com/apple/swift-evolution/blob/main/proposals/0354-regex-literals.md)(`BareSlashRegexLiterals`)は、Swift6まで`/.../`正規表現リテラル構文の導入を延期する。この機能を有効にすることは、`-enable-bare-regex-syntax`と同等で、`/.../`正規表現リテラル構文が利用可能になる。このプロポーザルとSE-0354が同じリリースで導入される場合、このプロポーザルのアプローチをサポートして`-enable-bare-regex-syntax`は完全に削除できる

###　Swift Package Managerのサポート

SwiftPMのターゲットに、必要なupcoming featureを指定できる必要がある。`SwiftSetting`のAPIを拡張して、upcoming featureを有効にする。

```swift
extensionSwiftSetting {
    public static func enableUpcomingFeature(
        _ name: String,
        _ condition: BuildSettingCondition? = nil
    )->SwiftSetting
}
```

SwiftPM は、この設定を使用してモジュールをビルドするときに、そこにリストされているupcoming featureを`-enable-upcoming-feature`フラグを介してコンパイラに渡す。これに依存する他のターゲットは、ビルド時に機能を渡す必要はない。というのも、upcoming featureの影響範囲はモジュールの境界を越えない

新しい機能がコンパイラに追加されるたびにSwiftPMのマニフェスト形式を変更する必要がないように、機能は文字列として提供される。パッケージの作成者は、バージョン管理された新しいマニフェストを作成しなくても、古いツールを引き続きサポートしながら、upcoming featureを追加できる

### ソース上での機能の検知

新しい機能を導入する場合、その機能が利用できない古いツールでもコードをコンパイルしたいというのはよくあること。このためには、`-enable-upcoming-feature`または適切な言語バージョンを有効にすることによって、機能が有効になっているかどうかを確認する方法が必要になる。

Swiftの`#if`に`hasFeature(X)`チェックを明示的にできるように拡張する必要がある。これは、識別子`X`を持つ機能が利用可能な場合は常に`true`と評価される。特定の機能をチェックする必要があるコードは、次のように`#if hasFeature`を使用できる。

```swift
#if hasFeature(ImplicitOpenExistentials)
    f(aCollectionOfInts)
#else
    f(AnyCollection<Int>(aCollectionOfInts))
#endif
```

`hasFeature(X)`チェックは機能の有無を示すが、古いコンパイラ自体は機能が不明であっても`#if`分岐を解析しようとする。この機能(`ImplicitOpenExistentials`)は新しい構文を追加しないため問題ないが、構文を追加する他の機能では、さらに何かが必要になる場合がある。`hasFeature`は[SE-0212](https://github.com/apple/swift-evolution/blob/main/proposals/0212-compiler-version-directive.md)で導入した`compiler`ディレクティブと組みわせることができる

```swift
#if compiler(>=5.7)&& hasFeature(BareSlashRegexLiterals)
let regex= /.../
#else
let regex= try NSRegularExpression(pattern: "...")
#endif
```

`hasFeature`自体はこのプロポーザルより前のツールでは理解されないため、上記のコードには問題がある。そのため、上記のコードは`hasFeature`の導入より前のSwiftコンパイラではコンパイルできない。次のように`hasFeature`チェックをネストすることで、この問題を回避することができる(Swift5.7 で`hasFeature`が導入されたと仮定)。

```swift
#if compiler(>=5.7)
  #if hasFeature(BareSlashRegexLiterals)
  let regex= /.../
  #else
  let regex= #/.../#
  #endif
#else
let regex= try NSRegularExpression(pattern: "...")
#endif
```

最悪の場合、これには、`hasFeature`の導入前のSwiftバージョンで動作する必要があるライブラリのコードと重複してしまうが、コンパイラは処理可能になり、時間の経過とともにその制限はなくなる。

今後のどんなupcoming featureに対しても、`#if`構文のこの問題を回避するために、左側が`compiler(>=5.7)`や`swift(>=6.0)`などの`#if`本文の解析を不可能にして、右側の項が式全体の結果を決定する必要がない場合、コンパイラは`&&`または`||`の右側にある「呼び出し」構文を解釈するべきではない。例えば、将来`#if hasAttribute(Y)`のようなものを導入した場合、次の式が使用できる。

```swift
#if compiler(>=5.8)&& hasAttribute(Sendable)
...
#endif
```

(`hasAttribute`をサポートすると想定した)Swift5.8 以降のコンパイラでは、完全な条件が評価されます。それ以前のSwiftコンパイラ(つまり、このプロポーザルをサポートしているが`hasAttribute`のような新しい機能が存在していないバージョン)では、`&&`または`||`の後のコードを式として解析するが、評価されないためコンパイラはこの`#if`条件をエラーにしない。

### 実験的な機能の採用

コンパイラの言語機能は、開発時に「実験的」フラグに隠れた状態でステージングされるのが一般的。これは、通常アドホックな方法で行われ、機能が最終的にリリースされる前にフラグを削除する。ただし、この実験的な機能モデルをもっと活用するべき。そこで、機能が開発中の場合は、新しいフラグ`-enable-experimental-feature X`または対応するSwiftPM`enableExperimentalFeature`で有効にできる機能識別子を提供する。

実験的な機能はまだ不安定であると考えられており、リリースされたコンパイラでは利用できないはず。ただし、実験的な機能とupcoming featureを導入する方法を統一することで、同じステージングの仕組みを利用することができる。つまり、機能を有効にし、ソースコードで簡単に試すことができる。機能がサポートされている完全な言語機能に「卒業」した場合、`hasFeature`はその機能に対して`true`を返せる。また、機能の一部が次の主要な言語バージョンまで延期された場合、`-enable-upcoming-feature`も機能上手く働く。

## ソース互換性

言語自体については、`hasFeature`が唯一の追加であり、ユーザ定義可能な関数がない制約された構文空間(`#if`)でのみ利用される。したがって、新しいコンパイラが既存の適切な形式のコードをエラーにするという従来の意味でのソース互換性の問題ない。

SwiftPM の場合、`enableUpcomingFeature`および`enableExperimentalFeature`関数を`SwiftSetting`に追加することは、1度、マニフェストファイルフォーマットを分ける必要がある。これらの関数を採用し、`enableUpcomingFeature`および`enableExperimentalFeature`の導入よりも前のバージョンのツールをサポートしたいパッケージは、バージョン管理されたマニフェスト(`Package@swift-5.6.swift`など)を使用して、新しいツールバージョンの機能の導入が可能。`enableUpcomingFeature`と`enableExperimentalFeature`が追加されても、追加の機能を導入するためにマニフェストファイルの別のコピーは必要ない。

## 参考リンク

### Forums

- [Piecemeal adoption ofSwift6improvements inSwift5.x](https://forums.swift.org/t/piecemeal-adoption-of-swift-6-improvements-in-swift-5-x/57184)
- [SE-0362: Piecemeal adoption of future language improvements](https://forums.swift.org/t/se-0362-piecemeal-adoption-of-future-language-improvements/58384)
- [[Accepted] SE-0362: Piecemeal adoption of future language improvements](https://github.com/apple/swift-evolution/blob/main/proposals/0362-piecemeal-future-features.md)

### プロポーザルドキュメント

- [Piecemeal adoption of upcoming language improvements](https://github.com/apple/swift-evolution/blob/main/proposals/0362-piecemeal-future-features.md)

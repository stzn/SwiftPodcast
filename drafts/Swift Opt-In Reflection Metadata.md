# Swift Opt-In Reflection Metadata

- [Swift Opt-In Reflection Metadata](#swift-opt-in-reflection-metadata)
  - [概要](#概要)
  - [内容](#内容)
    - [理由](#理由)
    - [提案内容](#提案内容)
      - [Case Study1](#case-study1)
      - [Case Study2](#case-study2)
      - [条件付きおよび強制キャスト(`as? Reflectable`、`as! Reflectable`、`is Reflectable`)](#条件付きおよび強制キャストas-reflectableas-reflectableis-reflectable)
      - [Swift6での挙動の変化](#swift6での挙動の変化)
      - [標準ライブラリでの挙動の変化](#標準ライブラリでの挙動の変化)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

既存のメカニズムを改善し、リフレクションメタデータを使用する APIを提供して、Swift リフレクション メタデータの安全性、効率性、機密性を高める

## 内容

### 理由

Swiftコンパイラが出力するメタデータは2種類ある

- コアメタデータ(型のメタデータのレコード、nominalタイプの記述子など) OSが参照するための情報
- リフレクションメタデータ(リフレクションメタデータの編集記述子など) 名前などの文字列情報

コアメタデータは常に出力する必要があり、使用されていないことが明らかになった場合にのみ削除できます。(この種のメタデータは、このプロポーザルの影響を受けない。)一方、リフレクションメタデータには、宣言フィールドに関するオプションの情報(名前と型への参照)が含まれている。Swiftランタイムはこのメタデータを使用しないため、そのような型がリフレクションを使用する APIに渡されない場合、出力をスキップする場合がある。

現在、出力可能なメタデータの出力を部分的に有効にする方法や、APIが水面下でリフレクションメタデータを使っているかどうかを認識する方法がない。さらに、出力を完全に無効にするコンパイラフラグが存在する。

今のところ、次の2つの内の1つの方法がとれる

- 念のためリフレクションを完全に有効にする
- どのAPIがメタデータを使用しているかを推測して、そのAPIを使っているモジュールに対してのみ有効にする

これらのオプションには両方とも欠陥がある。最初のものは、リフレクションメタデータがバイナリサイズの増加につながり、生成されたコードの機密性に影響を与える可能性がある。2 つ目は、推測が間違っている場合、多くの APIはブラックボックスであり、実行時にアプリが期待どおりに動作しない可能性があるため、安全ではない。

さらに、APIはリフレクションメタデータを異なる方法で使用できる。`print`、`debugPrint`、`dump`などはリフレクションを無効にしても機能するが、出力は制限される。SwiftUIなどの他のものはこれに依存しており、リフレクションメタデータが欠落している場合は正しく機能しない。前者にもメリットがありますが、このプロポーザルの主な焦点は後者にあります。

開発者は、Swiftモジュールのリフレクションメタデータを誤ってオフにすることがあり、そのモジュールでリフレクション APIが使用されている場合、コンパイル時に警告が表示されない。このようなモジュールを含むアプリは、実行時に期待どおりに動作しないため、そのようなバグに気付いたとしても、リフレクションメタデータまで追跡するのが難しい場合がある。例えば、SwiftUIの実装では、ユーザモジュールからのリフレクションメタデータを使用して、状態が変化したときにビュー階層の再レンダリングをトリガーする。なんらかの理由で、ユーザモジュールがメタデータの生成を無効にしてコンパイルされた場合、状態を変更してもその動作はトリガーされず、状態と表示されている内容の間に矛盾が生じ、そのような APIはコンパイル時ではなく実行時の問題になるため、安全性が低下する。

一方、現在そのAPIが使用されているかどうかを静的に判断する方法がないため、リフレクションメタデータは使用されていなくても過剰にバイナリに保持される場合がある。これは、Dead Code Elimination LLVMを経由して改善することで、未使用のリフレクションメタデータの量を制限する試みがあったが、多くの場合、フルタイプのメタデータによって参照されるため、依然としてバイナリに保持される。この結果、リフレクションメタデータが削除できない。これによって、バイナリサイズが不必要に大きくなったり、リバースエンジニアリングを簡単にされてしまう可能性を高める。

ここに静的コンパイルチェックを導入すると、リフレクションメタデータを実行時に利用できるようにする要件をSwiftに追加して、前述の問題の両方を解決することに役立つ可能性がある。

### 提案内容

リフレクションAPIが使用されている場合に、型チェッカとIRGenへリフレクションメタデータがバイナリに保持されるように伝えると、問題を実行時からコンパイル時に移行することができる。

これを実現するために、新しいマーカープロトコル`Reflectable`を導入する。まず、APIの開発者は、関数の一般的な要件を通じてリフレクションメタデータへの依存関係を表現することができるようになる。これにより、APIがより安全になる。第2に、IRGenの間、コンパイラは、`Reflectable`プロトコルに明示的に準拠する型のリフレクションメタデータのシンボルを選択的に出力できるようになる。

#### Case Study1

SwiftUI:

```swift

protocol SwiftUI.View: Reflectable {}  
class NSHostingView<Content> where Content : View {  
    init(rootView: Content) { ... }  
}
```

ユーザモジュール:

```swift

import SwiftUI  
  
struct SomeModel {}  
  
struct SomeView: SwiftUI.View {  
    var body: some View {          
        Text("Hello, World!")  
            .frame(...)      
    }  
}  
  
window.contentView = NSHostingView(rootView: SomeView())
```

`SomeView`のリフレクションメタデータは、`Reflectable`プロトコルに暗黙的に準拠しているため出力されるが、`SomeModel`のリフレクションメタデータは出力されない。ユーザモジュールがリフレクションメタデータを無効にしてコンパイルすると、コンパイラはエラーを出力する。

#### Case Study2

Framework:

```swift

public func foo<T: Reflectable>(_ t: T) { ... }
```

ユーザモジュール:

```swift

struct Bar: Reflectable {}  
foo(Bar())
```

`Reflectable`プロトコルに明示的に準拠しているため、`Bar`のリフレクションメタデータが出力される。`Reflectable`に準拠していない場合、`Bar`型のインスタンスを関数`foo`で使用することはできない。ユーザモジュールがリフレクションメタデータを無効にしてコンパイルされると、コンパイラはエラーを出力する。

#### 条件付きおよび強制キャスト(`as? Reflectable`、`as! Reflectable`、`is Reflectable`)

また、`Reflectable` プロトコへの条件付きキャストと強制キャストもできるように提案する。これは、型に関連するリフレクションメタデータが実行時に利用できる場合にのみ成功する。これにより、開発者はリフレクションメタデータが存在するかどうかを明示的に確認し、それに基づいてコードを適宜分岐できる。

```swift

public func conditionalUse<T>(_ t:  T) {
    if let _t = t as? Reflectable { // リフレクションメタデータを利用する
    } else { // デフォルト実装に戻る }
}

public func forceUse<T>(_ t:  T) {
    debugPrint(t as! Reflectable) // リフレクションメタデータが無効の場合クラッシュ
}

public func testIsReflectable<T>(_ t:  T) -> Bool {
    return t is Reflectable // リフレクションメタデータが有効の場合はTrue
}
```

#### Swift6での挙動の変化

Swift6以降では、ユーザエクスペリエンスの一貫性と安全性を確保するために、デフォルトでこのオプトインモードを有効にすることを提案する。ただし、新しいフラグ (`-enable-full-reflection-metadata`)でフルリフレクションが有効になっていない場合、`Reflectable`プロトコルに準拠していないすべての型のリフレクションメタデータの出力はスキップされる。これにより、`Reflectable`に準拠するために監査されなかったが、リフレクションAPIを使用しているコードの動作が変更される可能性がある。

例えば、`dump`、`debugPrint`、`String(describing:)`などの標準ライブラリのAPIの出力内容は制限される。ライブラリの作成者は、Swift6用にAPIを準備し、API で`Reflectable`に関する要件を満たす必要がある。

`-reflection-metadata-for-debugger-only`と`-disable-reflection-metadata`といったリフレクションメタデータを欠落させる可能性のあるコンパイラのオプションを非推奨にすることも提案する。Swift6以降では、デフォルトのオプトインモードを優先してこれらの引数を無視する。

#### 標準ライブラリでの挙動の変化

Swiftの`Mirror(reflecting:)`はリフレクションメタデータにアクセスする唯一の公式な方法で、他のすべてのAPIは内部でそれを使用している。必要に応じてリフレクションを使用したくない開発者に制限を課すことになるため、`Mirror`型に`Reflectable`の制約を追加することは意図的に提案していない。リフレクションメタデータの存在が必須の場合、`Reflectable`プロトコルの要件は、呼び出し関数のシグネチャで表現する必要がある。

## 参考リンク

### Forums

- [SE-0379: Opt-In Reflection Metadata](https://forums.swift.org/t/se-0379-opt-in-reflection-metadata/61714)
- [Returned for revision)](https://forums.swift.org/t/returned-for-revision-se-0379-opt-in-reflection-metadata/62390)

### プロポーザルドキュメント

[Swift Opt-In Reflection Metadata](https://github.com/apple/swift-evolution/blob/main/proposals/0379-opt-in-reflection-metadata.md)
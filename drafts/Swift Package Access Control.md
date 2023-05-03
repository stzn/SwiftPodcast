# Swift Package Access Control

- [Swift Package Access Control](#swift-package-access-control)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
  - [プロポーザル内容](#プロポーザル内容)
  - [詳細設計](#詳細設計)
    - [`package`キーワード](#packageキーワード)
    - [宣言側](#宣言側)
    - [呼び出し側](#呼び出し側)
    - [パッケージ名](#パッケージ名)
    - [パッケージシンボルの配布](#パッケージシンボルの配布)
    - [パッケージのシンボルと@inlinable](#パッケージのシンボルとinlinable)
    - [サブクラスとオーバーライド](#サブクラスとオーバーライド)
  - [将来の検討事項](#将来の検討事項)
    - [サブクラスとオーバーライド](#サブクラスとオーバーライド-1)
    - [パッケージプライベートモジュール](#パッケージプライベートモジュール)
    - [パッケージ内のグループ化](#パッケージ内のグループ化)
    - [最適化](#最適化)
  - [ソース互換性](#ソース互換性)
  - [ABI安定への影響](#abi安定への影響)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

本プロポーザルでは、新しいアクセス修飾子として`package`を導入する。現在、他のモジュールにあるシンボルにアクセスするためには、そのシンボルを`public`と宣言する必要がある。しかし、シンボルが`public`であると、パッケージ内、パッケージ外のどのモジュールからもアクセスできてしまい、好ましくない場合がある。このようなシンボルの可視範囲をより制御できるようにするために、新しいアクセス修飾子が必要である。

## 内容

### 動機

最も基本的なレベルでいえば、すべてのSwiftプログラムは関数、型、変数などの宣言の集まりである。原則として、これより上位のすべてのレベルは任意のものであり、これらの宣言を1つのファイルにまとめ、コンパイルして実行することができる。現実には、Swiftプログラムは別々のファイル、ディレクトリ、ライブラリなどに整理される。各レベルでのこの整理は、コード内の関係性と、コードの開発方法に関するプログラマの判断を反映している。

Swiftは、モジュールというレベルを認識している。モジュールは、独立したインターフェースと循環依存のない依存関係を持つライブラリの最小単位であり、名前空間とアクセス制御の両方でSwiftが認識することが合理的。ファイルは、それ以下の最小のグループであり、関連する宣言を収集するために使用される。そのため、アクセス制御を尊重することも合理的である。

Swift Package Managerによって表現されるパッケージは、コードの分配単位である。一部のパッケージは単一のモジュールしか含まず、パッケージのコードを複数のモジュールに分割することは役に立つことが頻繁にある。例えば、あるモジュールに内部のヘルパーAPIが含まれている場合、これらのAPIをユーティリティモジュールに分割して、他のモジュールやパッケージで再利用することができる。

しかし、Swiftはモジュールレベル以上のコードの組織化を認識していないため、パッケージ内で完全に内部的なAPIを作成することはできない。パッケージ内の他のモジュールから使用できるようにするためには、APIを公開する必要があるが、これによりパッケージの外部からも使用できるようになってしまう。これにより、クライアントがAPIに望ましくない依存関係を形成することができる。また、ビルドされたモジュールはAPIをエクスポートする必要があるため、コードサイズやパフォーマンスに否定的な影響を与える。

たとえば、以下は、クライアントが依存するパッケージ内のユーティリティAPIにアクセスできるシナリオである。クライアントアプリは、実行可能ファイルまたはXcodeプロジェクトである可能性がある。クライアントアプリは、`Game`と`Engine`の2つのモジュールを含む`gamePkg`というパッケージに依存しています。

`gamePkg`の`Engine`モジュール:

```swift
public struct MainEngine {
public init() { ... }
// 意図的なpublic
public var stats: String { ... }
// Gameからのみアクセスされるようにpublicにした意図的ではないヘルパー関数
public func run() { ... }
}
```

`gamePkg`の`Game`モジュール:

```swift

import Engine

public func play() {
MainEngine().run() // 同じパッケージ内であるためrunにアクセスできる。これは意図した通り
}
```

`appPkg`のクライアントアプリ:


```swift
import Game
import Engine

let engine = MainEngine()
engine.run() // アプリからrunにアクセスできるが、これは意図しないアクセスである
Game.play()
print(engine.stats) // 意図した通りにstatsにアクセスできる
```


上記のシナリオでは、`App`は`Engine`(`gamePkg`内のユーティリティモジュール)をインポートし、パッケージ外で使用されることが意図されていないヘルパーAPIに直接アクセスできてしまう。

このような意図しないパッケージAPIの公開を許可することは特に問題である。なぜなら、パッケージはコードの分配単位であり、Swiftは、明確に定義されたインターフェースを持つモジュールにプログラムを分割することを奨励するため、アクセス制御でモジュールの境界を強制する。しかし、このように分割された場合でも、密接に関連するモジュールを密接に関連する(または同じ)人々が作成することは珍しくない。このようなモジュール間のアクセス制御は、関心の分離を促進する目的を果たすが、モジュールのインターフェースを修正する必要がある場合、それは通常、単一のコミットなどで簡単に調整できる。しかし、パッケージでは、コードを単一の小規模組織よりもはるかに広く共有できる。そのため、パッケージ間の境界はしばしばプログラマ間の重要な違いを表し、APIの変更に関する調整がより困難になる。たとえば、オープンソースパッケージの開発者は、ほとんどのクライアントを知らないことが一般的であり、そのようなパッケージでは既存のAPIをメジャーバージョンリリースでのみ削除することが標準的な推奨事項である。したがって、プログラマがこれらのパッケージ間の境界を強制できるようにすることが特に重要である。

##  プロポーザル内容

私たちの目標は、アクセス制御の側面からパッケージを単位として認識する仕組みをSwiftに導入することである。我々は、`package`と呼ばれる新しいアクセス修飾子を導入することによって、これを行うことを提案する。`package`アクセス修飾子は、シンボルを定義するモジュールの外からアクセスすることを許可するが、同じパッケージ内の他のモジュールからしかアクセスできない。これにより、パッケージ間の境界を明確に設定することができる。

## 詳細設計

### `package`キーワード

`package`はアクセス修飾子として導入される。`package`は文脈キーワードであるため、`package`という名前の既存の宣言は引き続き機能する。これは、同じく文脈キーワードとして追加された`open`の先例に倣ったものである。たとえば、次のようなものが許可される:

```swift
package var package: String {...}
```

### 宣言側

`package`キーワードは、宣言場所で追加される。上記のシナリオを使用すると、ヘルパーAPIの`run`は、新しいアクセス修飾子で次のように宣言することができる:

`Engine`モジュール:

```swift

public struct MainEngine {
    public init() { ...  }
    public var stats: String { ...  }
    package func run() { ...  }
}
```

`package`アクセス修飾子は、既存のアクセス修飾子が使用できる場所であればどこでも使用できる(例：`class`、`struct`、`enum`、`func`、`var`、`protocol`など)。

Swiftでは、特定の場所(関数のシグネチャなど)で使用される宣言は、その宣言に含まれる宣言では、少なくともそのアクセスレベルと同じアクセスレベルでなければならない。この規則によって、`package`は、`open`および`public`よりもアクセスしにくく、`internal`、`fileprivate`、および`private`よりもアクセスしやすい。たとえば、`public`関数はパラメータや戻り値に`package`型を使用することができず、`package`関数はパラメータや戻り値に`internal`型を使用することができない。同様に、`@inlinable` `public`関数はその実装でパッケージ宣言を使用できず、`@inlinable` `package`関数はその実装で`internal`宣言を使用できない。


### 呼び出し側

`Game`モジュールは`Engine`と同じパッケージなので、ヘルパーAPIの`run`にアクセスすることができる。


`Engine`モジュール:

```swift

import Engine

public func play() {
    MainEngine().run() // // 同じパッケージ内のpackageシンボルであるため、`run`にアクセスできる
}
```

ただし、パッケージ外のクライアントがヘルパーAPIにアクセスしようとしても、それは許可されない。

```swift

import Game
import Engine

let engine = MainEngine()
engine.run() // Error: cannot find `run` in scope
```

### パッケージ名

言語としてのSwiftは、ビルドシステムに依存してパッケージの境界を定義する。コンパイラは、2つのモジュールが同じパッケージ名でビルドされた場合、同じパッケージに属するとみなすが、これは単なるUnicode文字列である。パッケージ名はソースレベルでは公開されないので、「パッケージ」に固有である限り、その正確な内容は重要ではない。

新しいフラグ`-package-name`は、次のようにコマンドライン呼び出しに引き渡される。

```swift
swiftc -module-name Engine -package-name gamePkg ...
swiftc -module-name Game -package-name gamePkg ...
swiftc -module-name App -package-name appPkg ...
```

`Engine`モジュールをビルドすると、そのパッケージ名`gamePkg`がビルドインターフェイスに記録される。`Game`モジュールをビルドするとき、そのパッケージ名`gamePkg`と`Engine`モジュールのビルドインターフェイスに記録されているパッケージ名が比較され、一致するので、`Game`モジュールは`Engine`モジュールのパッケージ宣言にアクセスすることができる。`App`をビルドする場合、そのパッケージ名`appPkg`は`gamePkg`と異なるので、`Engine`モジュールと`Game`モジュールのどちらのパッケージシンボルにもアクセスすることはできない。

`-package-name`が与えられていない場合、`package`アクセス修飾子は使えない。`package`アクセスを使用しないSwiftコードは、`-package-name`を渡す必要がなくビルドを継続する。パッケージ名なしでビルドされたモジュールは、他のモジュールと同じパッケージにあるとみなされることはない。

ビルドシステムは、パッケージ名がユニークであることを保証するために最善の努力をする必要がある。Swiftパッケージマネージャは、すでにすべてのパッケージに対してパッケージID文字列の概念を持っている。この文字列は一意であることが検証されており、すでにパッケージ名として機能しているので、SwiftPMは自動的にそれを受け渡すことになる。Bazelのような他のビルドシステムは、パッケージ名のために新しいビルド設定を導入する必要があるかもしれない。一意である必要があるため、衝突を避けるために逆DNS名を使用することがある。

ターゲットをパッケージ境界から除外する必要がある場合、マニフェスト内の新しい`packageAccess`設定を使用して、以下のように行うことができます：

```
.target(name: "Game", dependencies: ["Engine"], packageAccess: false)
```
`packageAccess`設定はデフォルトで`true`に設定され、ターゲットは`-package-name PACKAGE_ID`(`PACKAGE_ID`はマニフェストのパッケージ識別子) でビルドされます。`packageAccess`が`false`に設定されている場合、ターゲットをビルドするときに`-package-name`が渡されないため、ターゲットはパッケージシンボルにアクセスすることができず、本質的にパッケージの外側のクライアントであるかのように動作する。これは、パッケージ内のサンプルアプリやブラックボックステストターゲットに便利です。

### パッケージシンボルの配布

Swiftフロントエンドがソースから直接`.swiftmodule`ファイルを構築するとき、ファイルはパッケージ名とモジュール内のパッケージ宣言のすべてを含む。Swiftフロントエンドがソースから`.swiftinterface`ファイルをビルドするとき、ファイルはパッケージ名を含むが、セカンダリの`.package.swiftinterface`ファイルにパッケージ宣言が含まれることになる。Swiftフロントエンドがパッケージ名を含む`.swiftinterface`ファイルから`.swiftmodule`ファイルを構築するが、対応する`.package.swiftinterface`ファイルがない場合、`.swiftmodule`にこれを記録し、このファイルが同じパッケージの他のモジュールを構築するために使用されるのを防ぐ。

### パッケージのシンボルと@inlinable

`package`タイプは`@inlinable`にすることができます。`inlinable public`と同様に、`@inlinable package`の本文内ですべてのシンボルが使用できるわけではない。それらは`open`、`public`、または`@usableFromInline`でなければならない。`usableFromInline`属性は、`internal`宣言の他に`package`にも適用できる。これらの属性付きシンボルは、`@inlinable public`または`@inlinable package`宣言の本文(同じパッケージ内の任意の場所で定義されているもの)で使用できる。`internal`シンボルと同様に、`@usableFromInline`または`@inlinable`を持つパッケージ宣言は、モジュールの公開の`.swiftinterface`に格納される。

以下はその例:

```swift
func internalFuncA() {}
@usableFromInline func internalFuncB() {}

package func packageFuncA() {}
@usableFromInline package func packageFuncB() {}

public func publicFunc() {}

@inlinable package func pkgUse() {
    internalFuncA() // Error
    internalFuncB() // OK
    packageFuncA() // Error
    packageFuncB() // OK
    publicFunc() // OK
}

@inlinable public func publicUse() {
    internalFuncA() // Error
    internalFuncB() // OK
    packageFuncA() // Error
    packageFuncB() // OK
    publicFunc() // OK
}
```

### サブクラスとオーバーライド

Swiftのアクセス制御は、通常、異なる種類の使用を区別していない。たとえば、プログラムが型にアクセスできる場合、それはプログラマに幅広い権限を与えます。たとえば、型名はほとんどの場所で使用でき、型の値は借用、コピー、破棄でき、型のメンバは(型自身のアクセス制御の限界まで)アクセスできる、など。これは、アクセス制御がカプセル化を強制し、コードの将来的な進化を可能にするためです。広範な権限が付与されるのは、通常、より厳しく制限することはその目的に適わないからです。

ただし、2つの例外があります。1つ目は、Swiftは`var`と`subscript`が読み取り専用のアクセスよりも変更するアクセスをより厳しく制限することができる。これは、セッターのために別のアクセス修飾子を書くことによって実現できます。たとえば、`private(set)`など。2つ目は、Swiftがクラスとクラスメンバーが通常の参照よりも厳しくサブクラス化とオーバーライドを制限できることです。これは、`open`の代わりに`public`を書くことができる。これらの権限を個別に制限できることで、カプセル化と進化を促進することができます。

セッターのアクセスレベルは、メインのアクセス制御とは別の修飾子を書くことで制御されるため、構文は自然に`package(set)`とできるように拡張されます。しかし、サブクラス化とオーバーライドは、主要なアクセス修飾子として特定のキーワード(`public`または`open`)を選択することで制御されるので、構文は同じように`package`に拡張されない。このプロポーザルでは、クラスとクラスメンバに対して、`package`それ自体が何を意味するかを決定しなければならない。また、`package`だけではカバーできないオプションをサポートするか、将来の可能性として残しておくかを決めなければならない。

ここでは、現在の各アクセスレベルを持つシンボルが、どこで使用または上書きできるかを示すマトリックスを示す。

<thead>
<tr>
<th></th>
<th>Accessible Anywhere</th>
<th>Accessible in Module</th>
</tr>
</thead>
<tbody>
<tr>
<th>Subclassable Anywhere</th>
<td align="center">open</td>
<td align="center">(illegal)</td>
</tr>
<tr>
<th>Subclassable in Module</th>
<td align="center">public</td>
<td align="center">internal</td>
</tr>
<tr>
<th>Subclassable Nowhere</th>
<td align="center">public final</td>
<td align="center">internal final</td>
</tr>
</tbody>
</table>

新しいアクセス修飾子として`package`を使用すると、マトリックスは次のように変更される。

<table>
<thead>
<tr>
<th></th>
<th>Accessible Anywhere</th>
<th>Accessible in Package</th>
<th>Accessible in Module</th>
</tr>
</thead>
<tbody>
<tr>
<th>Subclassable Anywhere</th>
<td align="center">open</td>
<td align="center">(illegal)</td>
<td align="center">(illegal)</td>
</tr>
<tr>
<th>Subclassable in Package</th>
<td align="center">?(a)</td>
<td align="center">?(b)</td>
<td align="center">(illegal)</td>
</tr>
<tr>
<th>Subclassable in Module</th>
<td align="center">public</td>
<td align="center">package</td>
<td align="center">internal</td>
</tr>
<tr>
<th>Subclassable Nowhere</th>
<td align="center">public final</td>
<td align="center">package final</td>
<td align="center">internal final</td>
</tr>
</tbody>
</table>


このプロポーザルは、`package`単独では、定義モジュールの外でのサブクラス化やオーバーライドを許可すべきではないという立場をとっている。これは`public`の挙動と一致しており、`package`を既存の特権アクセス制御のルールに則ったものである。また、`public`クラスやメソッドの通常の最適化モデルを`package`クラスやメソッドにも適用することができ、サブクラス化やオーバーライドが行われない場合は暗黙のうちに`final`にすることができるし、新たに「package全体の最適化」という構築モードを必要としない。

しかし、この選択では、上の表で`?`でマークされた2つの組み合わせを綴る方法がない。これらは設計や実装がより複雑になるため、「将来の検討事項」で説明します。

## 将来の検討事項

### サブクラスとオーバーライド

上のマトリックスから`?(a)`と`?(b)`で示された実体は、どちらもパッケージ内のクロスモジュールにアクセスしサブクラス化する必要がある(パッケージ内で`open`)。唯一の違いは、(b)はパッケージの外からシンボルを隠し、(a)は外から見えるようにすることである。(a)の使用例は稀だが、シンボルの見え方を除けば(b)と基本的な流れは同じであるべきである。

解決策として、特定のアクセスの組み合わせに対して新しいキーワードを導入する(例：`packageopen`)、`open`をアクセス修飾できるようにする(例：`open(package)`)、アクセス修飾子を特定の目的で修飾できるようにする(例：`package(override)`)などが考えられる。

### パッケージプライベートモジュール

モジュール全体が、それを提供するパッケージに対してプライベートであることを意図していることがある。これを直接表現できるようにすると、ユーティリティモジュールをパッケージの外に完全に隠すことができるようになり、モジュールの存在に対する不要な依存を避けることができる。また、ビルドシステムがパッケージ内のモジュールを自動的に名前空間化することができ、異なるパッケージのユーティリティモジュールが(`Utility`などの)名前を共有している場合や、パッケージの複数のバージョンを同じプログラムに組み込む必要がある場合に、明示的な[モジュールエイリアス](https://github.com/apple/swift-evolution/blob/main/proposals/0339-module-aliasing-for-disambiguation.md)の必要性を低減することができるようになる。

### パッケージ内のグループ化

このプロポーザルの基本的な言語設計は、関連するモジュールのグループであれば、どのようなグループでも機能しますが、SPMでこの設計を適用すると、SPMパッケージごとに単一のそのようなグループしかできません。複雑なSPMパッケージの開発者は、1つのパッケージの中に複数のアーキテクチャの「レイヤー」があることに気づき、パッケージをレイヤーの中だけで適用できるようにしたいと思うことがあります。論理的には、各レイヤーをそれぞれのパッケージに入れることはある程度理にかなっています。現実的には、異なるSPMパッケージが別々のリポジトリに存在し、独立してバージョン管理されなければならないため、このようにパッケージを分割することは、開発プロセスに膨大な量の余分な複雑さをもたらすことになり、気軽に行うべきことではない。

SPMが1つのパッケージリポジトリの中で複数のレイヤーをサポートするように進化するためには、いくつかの合理的な方法がある。たとえば、`.target`に`group`パラメータを追加することによって、マニフェスト内でターゲットをグループ化することができる。このプロポーザルの以前のバージョンはこれをプロポーザルし、`packageAccess:`除外機能を設計した。しかし、これはすべてのレイヤーの詳細を一緒に混在させる大規模で複雑なマニフェストにつながる傾向がある。非常に異なるアプローチは、リポジトリ内でサブパッケージの作成を許可し、それぞれが独自のマニフェストを持つようにすることである。SPMは、これらのサブパッケージを、単一のリポジトリとバージョンを共有する論理的に独立したユニットとして扱う。これらは独立したマニフェストで記述されるため、異なるパッケージのように感じられ、パッケージへのアクセスがそれらの中でスコープされることは理に適っている。

### 最適化

モジュールをレジリエントにするライブラリエボリューションを有効にしても、パッケージはレジリエンスドメインとして扱われることがある。Swiftフロントエンドは、同じパッケージで定義されたモジュールは常に一緒に再構築され、それらの間のレジリエントなABI境界を必要としないと仮定する。これは、ABIレジリエントなコード生成によって導入されたパフォーマンスとコードサイズのオーバーヘッドの必要性を排除し、またnon-`frozen enum`のための`@unknown default`のような言語要件を排除している。

デフォルトでは、`package`シンボルは最終的なライブラリ/実行可能ファイルでエクスポートされる。静的にリンクされたライブラリの`package`シンボルを隠すことができるビルド設定を導入することは、コードサイズとビルド時間の最適化に役立つと思われる。

## ソース互換性

影響なし

## ABI安定への影響

パッケージ内の個別に構築されたモジュール間の境界は、依然として潜在的なABIの境界である。`package`シンボルのABIは`public`シンボルのABIと変わらないが、将来的にはimage内で解決できる`package`シンボルをエクスポートしないオプションを追加することが検討されるかもしれない。


## 参考リンク

### Forums

- [New Access Modifier: package](https://forums.swift.org/t/new-access-modifier-package/61459)
- [SE-0386: `package` access modifier](https://forums.swift.org/t/se-0386-package-access-modifier/62808)
- [[Second review] SE-0386: `package` access modifier](https://forums.swift.org/t/second-review-se-0386-package-access-modifier/64086)


### プロポーザルドキュメント

- [New access modifier: `package`](https://github.com/apple/swift-evolution/blob/main/proposals/0386-package-access-modifier.md)
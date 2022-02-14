# Swift Concurrencyの段階的な導入について考える

収録日: 2022/02/05

- [Swift Concurrencyの段階的な導入について考える](#swift-concurrencyの段階的な導入について考える)
  - [概要](#概要)
  - [用語](#用語)
  - [内容](#内容)
    - [`Sendable`チェックの適用が難しい箇所](#sendableチェックの適用が難しい箇所)
      - [ライブラリを全てを遡ってアノテーションを追加する](#ライブラリを全てを遡ってアノテーションを追加する)
      - [Swift Concurrency未対応のモジュールに`Sendable`チェックを適用する](#swift-concurrency未対応のモジュールにsendableチェックを適用する)
    - [解決策](#解決策)
      - [導入に至るまでの想定ワークフロー](#導入に至るまでの想定ワークフロー)
      - [実現するために必要な機能](#実現するために必要な機能)
    - [詳細](#詳細)
      - [リカバリ動作](#リカバリ動作)
      - [Concurrencyチェックモード](#concurrencyチェックモード)
      - [名前的型宣言の`@preconcurrency`属性](#名前的型宣言のpreconcurrency属性)
      - [`Sendable`準拠の状態](#sendable準拠の状態)
      - [`@preconcurrency`と`Sendable`プロトコル](#preconcurrencyとsendableプロトコル)
      - [`@preconcurrency`属性と`import`宣言](#preconcurrency属性とimport宣言)
    - [ソースの互換性](#ソースの互換性)
    - [ABI安定性への影響](#abi安定性への影響)
    - [APIレジリエンスへの影響](#apiレジリエンスへの影響)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [関連PR](#関連pr)

## 概要

Swift5.5で、値が安全にTaskやactorの境界を跨いで安全に受け渡しができることを示す`Sendable`プロトコルや、例えばMainActorと適切に同期するためのglobal actorなど、データ競合を排除するためのメカニズムが導入された。しかし、Swift5.5だと、まだSwift Concurrencyに対応していないモジュールとやり取りするのがあまりにも骨が折れる作業であることがわかっていたため、この`Sendable`やMainActorの全ての使用も強制はしていない。開発者がSwift Concurrencyをサポートしたり、Swift Concurrency未対応の他のモジュールとやり取りするためのコードのマイグレーションを支援して、データ競合を排除するためにSwiftのエコシステムにスムーズなパスを提供する。

## 用語

- Sendable: 同時並行処理を行うTaskやactor間で安全に受け渡しができることを示すプロトコル
- データ競合: 複数スレッドから同じメモリの可変状態に書き込み/読み込み処理が同時に起きること
- Task: Swift Concurrencyの一連のasync関数などを実行する単位
- actor: 同時並行処理でデータ競合を起こさずに内部の可変状態を安全に扱えるようにした参照型
- global-actor: グローバルな状態を一つのactorの内部状態として分離するためのactor。プロセスに一つのインスタンスしか存在しない。複数の異なる型やメソッドなどに適用でき、安全にglobal-actorのエグゼキュータ内でデータ競合を起こさずに安全に同時並行処理で扱うことができる。

## 内容

Swift Concurrencyは同時並行プログラミング内でデータ競合を排除するための分離された状態を提供するメカニズムを探している。主なメカニズムは、`Sendable`チェック。Taskやactorの境界を跨いでデータを送信するAPIの入力値は`Sendable`に準拠していなければならない。コンパイラは、型が`Sendable`に準拠していて、`Sendable`に準拠した型のみが含まれているかどうかをチェックして安全性を確認する(ただし、型の作者が`@unchecked`を付与していてnon-`Sendable`な型でも安全に使えるように明示的に実装している場合を除く)。

Swiftの既存のコードや、CやObjective-Cのコードの同時並行処理は、`Sendable`チェック(コンパイル)が理解できるような方法では実装されていない。しかし、それがSwift Concurrencyに対応するのを待たなくても引き続き使い続けたい。

### `Sendable`チェックの適用が難しい箇所

#### ライブラリを全てを遡ってアノテーションを追加する

多くの既存のAPIは、これまでおこなってきた同時並行処理をSwift Concurrencyが理解できるような形に対応させる必要があるものの、それをコンパイラに伝えることができていない。例えば、UIKitのメソッドやプロパティはメインスレッドで実行されるべきだが、`@MainActor`属性導入以前は、コンパイラに伝えることができず、ドキュメントの記載や実装内のアサーションでのみしかその安全性を開発者に伝えることができなかった。

そのため、多くのモジュールはどこにSwift Concurrencyのアノテーションを追加するかを決めるためにAPI全体をチェックする必要がある。さらに、もし今あるツールでそれを実行しようとすると、source breakingが間違いなく発生するだろう。例えば、あるメソッドに`@MainActor`をつけると、Swift Concurrency未対応のプロジェクトは、正しく使っているのにコンパイラにMainActorで実行していることを伝える手段がないため、このメソッドは使えなくなる。

いくかの場合だと、ABI breakingも起こす可能性がある。例えば、関数型の`@Sendable`属性やジェネリックパラメータへの`Sendable`の制約は、`Sendable`への準拠が呼び出し規約に影響がない場合でも(余計なwitness tableのパラメータがいらないなど)、マングリングされた関数名に組み込まれてしまう。型チェック中にこの制約を適用するためのメカニズムが必要であるものの、あたかも`Sendable`が存在しないようにコードは生成される。

つまり、必要なのは、

- Swift Concurrency対応済のモジュールをインポートするSwift Concurrency未対応コードへの「互換モード」
- Swift Concurrencyによってシグネチャが変わってしまった場合に、「互換モード」への特別な対応のために宣言にマークを付ける方法

#### Swift Concurrency未対応のモジュールに`Sendable`チェックを適用する

モジュールがSwift Concurrencyのアノテーションを追加するのを待つのは時間がかかると想定される。そのため、`Sendable`チェックの適用を開始する前に全てのモジュールが更新されるまで待つのは現実的ではない。

モジュールがインポートされている側へ、未対応なアノテーションへのワークアラウンドが必要。これは、インポートされた宣言を微調整するか、エラーを無視するようにコンパイラに使えるかのどちらかが必要である。ただ、どんな方法にせよ、あまり面倒なことにはしたくない。例えば`Sendable`として扱って欲しいnon-`Sendable`型の全部の変数にマークを付けるなどはとても辛い。

また、最終的にモジュールでSwift Concurrencyに対応した時にも、何が起きたのかを特別に注意しなければならず、利用側で間違った同時並行処理の挙動が想定されていた場合はそれを明らかにする必要がある。

例えば、

1. `Geomerty`モジュールから`Point`型をインポートする
2. `Geomerty`モジュールがSwift Concurrency未対応のタイミングで`Sendable`チェックを適用する
3. 公開されている情報から`Point`型はおそらく`Sendable`であると判断する
4. `Sendable`チェックを抑制する(`@unchecked`を付けるなど)
5. `Geomerty`モジュールの著者が後で`Point`型の実装を見て、同時並行処理で渡されると安全ではないと判断し、non-`Sendable`としてマークする

この状況でモジュールの最新バージョンを使ってアプリをリビルドした場合どうなるだろうか？

理想を言うと、Swiftはこのバグを抑制し続けるべきではない。結局`Geomerty`モジュールのチームはnon-`Sendable`としてマークしているため、インポートした側の推測よりも信頼できるものである。一方で、この変更はリグレッションではないので、利用側のプロジェクトがリビルドできなくなるべきでもない。これは現状の改善で。最新の`Geomerty`モジュールはバグを追加したわけではなく、既にバグがあったということが明らかになっただけ。バグは診断される方が隠れているよりも良い。

しかし、Swiftがこのバグに対してエラーなどを出力すると、昨日ビルドできたものができなくなるため、`Geomerty`モジュールのアップデートを延期したり、モジュールの著者にモジュールのアップデートを遅らせるようにプレッシャーをかけるかもしれない。そこで、安全だと想定していたものがあとで間違っていたとわかった場合は、Swiftはエラーではなくワーニングを出して、ビルドはできるもののバグに気づくようにするべき。

つまり、

- 特定の宣言やモジュールにSwift Concurrencyのアノテーションが足りない場合にエラー診断を抑制するメカニズム
- Swift Concurrencyのアノテーションが追加された後はこの診断を復活させるが、ワーニングのみを出力するルール

が必要になる。

### 解決策

#### 導入に至るまでの想定ワークフロー

このSwift Concurrencyのアノテーションの適用(特に`Sendable`チェック)をサポートする機能群を導入する。これらの機能は、Swift Concurrency適用のための下記のワークフローを達成するために設計されている。

1. Swift Concurrencyの機能(`async`/`await`や`actor`など)を適用する(Swift6モードでのビルドや、Swift5系に`-warn-concurrency`フラグを追加したビルドをする)ことで、Concurrencyチェックを有効にする。こうすると、Concurrencyチェックに違反した際は、新しくエラーやワーニングが発生する。
2. この問題の解決を開始する。他のモジュールからの型が関わっている場合、fix-itが`@preconcurrency import`という特殊なインポートが提案される。これはエラーやワーニングを抑制する。
3. 一度問題が解決したら、より広い範囲のビルドに変更を統合する。
4. 将来のある時点で、インポートしているモジュールが`Sendable`の準拠や他のConcurrencyアノテーションを追加して更新するとする。その場合に、名前的型宣言に`@preconcurrency`を付けることでConcurrency未対応のコードでsource breakingを起こさずに対応できる。また、`@preconcurrency import`の中で新しく追加された制約によって違反が発生した場合、この間違いはワーニングで教えてくれる。これは潜在的なバグなので直す。
5. このバグを直したら(あるいはバグはないかもしれないが)、`@preconcurrency import`は不要だというワーニングが現れるので`@preconcurrency`を消す。この時点からこのモジュールに関連した`Sendable`チェックのエラーは`@preconcurrency import`の利用が提案されず、Swift6モードではエラーになってビルドができなくなる。

#### 実現するために必要な機能

これを達成するために、下記の機能が必要になる:

- Swift6モードでは、全てのコードで`Sendable`の準拠不足や他のConcurrency関連の違反は実装ミスとして全部エラーになる。Swift6よりも古い言語バージョンでは、`-warn-concurrency`フラグを追加することでこのエラーをワーニングにする。
- 名前的型(nominal type)に`@preconcurrency`属性を適用すると、その宣言はConcurrencyチェックが行われるように更新されたことを示す。そのため、Concurrencyチェックに違反する場合でも、Swift5モードでは引き続き利用できるように、コンパイラがConcurrency適用前のバイナリと相互運用できるように調整を行う
- `import`宣言に`@preconcurrency`属性が適用されている場合は、そのモジュールからインポートしたnon-`Sendable`型が、`Sendable`として使用できないまたは制約を満たしていないことが明白に宣言されているにも関わらず、`Sendable`が必須の箇所で使われていた場合にのみ診断するようコンパイラに伝える。ただし、この場合はエラーではなくワーニングが発生するに留まる。

### 詳細

#### リカバリ動作

このプロポーザルで、エラーをワーニングとして出力したり、ワーニングを抑制する場合は、コンパイラが下記のように振る舞うことで「リカバリ」したことを意味する。

- 名前的型が`Sendable`に準拠していないように振る舞う
- 関数型に`@Sendable`やglobal-actorアノテーションが付いていないように振る舞う

#### Concurrencyチェックモード

Swiftは、次の2つのConcurrencyチェックモードの内のいずれかを備えていると説明できる

- 厳密(Strict)モード: `Sendable`準拠不足やglobal-actorアノテーションは診断される。Swift6モードの場合、これらは基本的にエラーになる。Swift5モードでは、`@preconcurrency import`を通して使われている名前的型の宣言の場合、ワーニングになる
- 最小(Minimal)モード: `Sendable`準拠不足やglobal-actorアノテーションはワーニングで診断される。名前的型の宣言に`@preconcurrency`属性を適用した場合は、特殊な効果が発生して多くの診断を抑制する

トップレベルスコープでのConcurrencyチェックモードは:

- Swift6モード以降でコンパイルされた場合と、それよりも古い言語バージョンで`-warn-concurrency`フラグが使われている場合、またはファイルがモジュールインターフェイスとして解析された場合、厳密モードになる
- それ以外は最小モードになる

子レベルスコープでのConcurrencyチェックモードは:

- 親レベルスコープのConcurrencyチェックモードが最小モードで子レベルスコープが次の条件に当てはまる場合:
  - global-actor属性を明示しているクロージャ
  - `async`や`@Sendable`なクロージャやオートクロージャ(ただし、親のモードが最小モードの場合は`@Sendable`と推論されるかどうかが影響を与える可能性があることに注意)
  - 明示的に`nonisolated`キーワードやglobal-actor属性が付与されている宣言
  - `async`や`@Sendable`がマークされた関数、メソッド、イニシャライザ、アクセサ(get/set)、変数、subscript
  - `actor`
  - それ以外は親スコープと同じ

実装詳細: 子スコープが厳密モードか最小モードかを決定するロジックは`swift::contextRequires厳密ConcurrencyChecking()`に実装されている。インポートされたC言語の宣言は最小モードになる。

#### 名前的型宣言の`@preconcurrency`属性

これらのConcurrency上での挙動に対応するためには、著者は既存の宣言を変更せねばならず、Concurrency未対応コードのsource breakingやこれまでにコンパイルしていたバイナリと相互運用している場合にABI-breakingを起こす可能性がある。特に注意が必要なのは

- `@Sendable`やglobal-actor属性を関数型に追加する
- ジェネリック構文に`Sendable`制約を追加する
- global-actor属性を宣言に追加する

名前的型宣言に適用されると、`@preconcurrency`属性はSwift Concurrencyに完全に対応する前からその宣言が存在していたことを示し、コンパイラはsource breakingやABI breakingを起こさないようにするべき。これは`enum`、`enum`の`case`、`struct`、`class`、`actor`、`protocol`、`var`、`let`、`subscript`、`init`または`func`宣言に使える。

名前的型宣言に`@preconcurrency`属性が使われていると:

- Concurrencyの機能が全く使われていないように名前がマングリングされる
- 利用者側が最小モード内で使った場合は、コンパイラはこの不一致の診断を全て抑制する
- ABIチェックはダイジェスト(※)を生成する時にこれらの機能を削除する

※ ダイジェストは、ハッシュ関数（1方向関数）を用いて、ある長さを持つデータを固定長のデータに変換したもの

また、Objective-Cの宣言は、常に`@preconcurrency`属性がついているようにインポートされる。

次の例は、Main Actor上でしか実行できない関数で別のTaskから提供されたクロージャを実行することを想定している。

```swift
@MainActor func doSomethingThenFollowUp(_ body: @Sendable () -> Void) {
    // 何かする
    Task.detached {
      // 何か他のことをする
      body()
  }
}
```

この関数は、`@MainActor`や`@Sendable`のない、Concurrency以前から存在していた可能性がある。これらのアノテーションを追加後、以前動いていたコードはエラーが発生する。

```swift
class MyButton {
    var clickedCount = 0

    func onClicked() { // 常にシステムがMain Threadから呼び出す
        doSomethingThenFollowUp { // ❌　ERROR: cannot call @MainActor function outside the main actor
            clickedCount += 1 // ❌ ERROR: captured 'self' with non-Sendable type `MyButton` in @Sendable closure
        }
    }
}
```

しかし、`@preconcurrency`を`doSomethingThenFollowUp`宣言に追加すると、`@MainActor`や`@Sendable`が削除されてエラーがなくなり、Concurrency対応以前と同じ型推論がされるように調整される。これは厳密モードと最小モードで下記のような違いがある:

```swift
func minimal() {
    let fn = doSomethingThenFollowUp // 型は (()-> Void) -> Void
}

func strict() async {
    let fn = doSomethingThenFollowUp // 型は @MainActor (@Sendable ()-> Void) -> Void
}
```

#### `Sendable`準拠の状態

ある型は次の3つの`Sendable`準拠の状態のいずれかであると説明できる:

- 明白な`Sendable`: 事実上`Sendable`に準拠している場合。明示的に`Sendable`が宣言されている、もしくは[SE-0302](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)のルールに基づいて準拠が推論される場合のいずれかが該当する
- 明白なnon-`Sendable`: 型に対して`Sendable`準拠が宣言されているがそれが使用できないか制約を満たしていない場合、または型が厳密モードが適用される範囲で宣言されていた場合(※)
- 暗黙的なnon-`Sendable`: 型に対して`Sendable`準拠が宣言されていない場合

※ もしモジュールがSwift6モードやSwift5系+`-warn-concurrency`フラグを指定している場合、全ての型は明白に`Sendable`か明白にnon-`Sendable`になることを意味している

型に`Sendable`として利用できない制約を作成して、明白なnon-`Sendable`にできる。例えば、

```swift
@available(unavailable, *)
extension Point: Sendable { }
```
このような準拠は暗黙的に型が`Sendable`に準拠するケースを抑制する。

#### `@preconcurrency`と`Sendable`プロトコル

一定数の既存のプロトコルは、全て`Sendable`である必要がある。そのようなプロトコルをConcurrencyに対応させる際、`Sendable`プロトコルをおそらく継承する。しかし、そうするとこのプロトコルに準拠した既存の型は`Sendable`だと想定されるので壊れてしまう。この問題は標準ライブラリの`Error`と`CodingKey`プロトコルに影響があるため[SE-0302](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md#thrown-errors)でも記載されていた。

```swift
protocol Error: /* 新しく追加された → */ Sendable { ... }

class MutableStorage {
  var counter: Int
}
struct ProblematicError: Error {
  var storage: MutableStorage // ❌　error: Sendable struct ProblematicError has non-Sendable stored property of type MutableStorage
}
```

これを解決するために、SE-0302では`Error`プロトコルの`Sendable`継承に関して下記の方法に従っている:

> 移行を簡単にするために、`Error`を通した`Sendable`準拠の問題はSwift6モード未満ではワーニングにダウングレードする

今回は、`@preconcurrency`属性が付けられ、`Sendable`プロトコルを継承する全てのプロトコルに適用するために、この`Error`と`CodingKey`プロトコルへの特別ルールを置き換え、これらの2つの標準ライブラリのプロトコルは`@preconcurrency`を使用するように変更になる。

```swift
@preconcurrency protocol Error: Sendable { ... }
@preconcurrency protocol CodingKey: Sendable { ... }
```

#### `@preconcurrency`属性と`import`宣言

`@preconcurrency`属性は`import`宣言にも適用することができ、そのモジュールからインポートした型が起こすConcurrencyチェック違反の厳しさを軽減することをコンパイラに伝える。これはConcurrency未対応のモジュールをインポートする際に活用できる。そうした場合、`Sendable`にする必要があるすべての型にアノテーションが付けられたときにコンパイラが後で教えてくれる。また、そのモジュールに関して誤った想定が修正されるまで、プロジェクトをコンパイルし続けるための一時的なエスケープハッチとしても機能する。

importに`@preconcurrency`属性を付けると、事実上次のルールが適用される:

- もし`Sendable`が必要なところに**暗黙的な**non-`Sendable`が使われていた場合
    - `@preconcurrency import`を通してその型が使われている場合は、Swift6未満の場合診断は抑制され、Swift6以降はワーニングが出力される
    - それ以外は、診断は通常通りに出力されるが、別の診断として`@preconcurrency import`がワークアラウンドとして提案される

- もし`Sendable`が必要なところに**明白な**non-`Sendable`が使われていた場合
    - `@preconcurrency import`を通してその型が使われている場合は、Swift6以降でもワーニングで出力される
    - それ以外は、診断は通常通りに出力される

- もし`@preconcurrency`属性が「未使用」な場合(※)、それを削除する提案がワーニングとして出力される

※ たとえば、お互いに影響を与える型をインポートする`@preconcurrency import`のペアの1つを削除することを推奨するのに十分なほど診断が改良できるかどうかわからないため「未使用」をこれ以上具体的に定義できない

### ソースの互換性

このプロポーザル自体がsource breakingを防ぐためのもの。`@preconcurrency`を活用することで最小モードでのsource breakingを防げるはず。そして依存しているモジュールがConcurrency未対応で厳密モードを適用していても、`@preconcurrency import`でConcurrencyチェックを弱めることができるため、ソースの互換性を保持できる。

### ABI安定性への影響

`@preconcurrency`それ自体はABIを変更しない。もし既にABIに影響を与えるようなConcurrencyの機能を宣言に適用している場合は、ABI breakingを起こすかもしれない。一方で、そういった機能が
`@preconcurrency`と同時にもしくは適用後に導入される場合には、ABI breakingは起きない。

`Sendable`は、追加のメタデータを出力したり、渡す必要のあるwitness tableを持ったり、呼び出し規約やABIの他のほとんどの部分に影響を与えたりしないように設計されているため、`Sendable`準拠適合エラーを無効にする`@preconcurrency`の戦術は現在のABIと互換性がある。名前マングリングにのみ影響する。

今回のプロポーザルでは、他にABIへの影響はない。

### APIレジリエンスへの影響

名前的型の`@preconcurrency`はモジュールインターフェイスへの出力が必要になる。これは事実上、レジリエンスを損なうような方法でAPIを進化させる機能である。

`import`宣言の`@preconcurrency`は、モジュールインターフェイスに出力する必要はない。モジュールインターフェイスは、Concurrencyチェックがワーニングである厳密モードを使用するため、足りない準拠を許容するのに十分な「余裕」がある。(いつものように、モジュールインターフェイスをコンパイルすると、デフォルトでワーニングが表示されない。）

## 参考リンク

### Forums

- [[Pitch] Staging in `Sendable` checking](https://forums.swift.org/t/pitch-staging-in-sendable-checking/51341)
- [[Pitch 2] Staging in Sendable checking](https://forums.swift.org/t/pitch-2-staging-in-sendable-checking/52413)
- [[Pitch #3] Incremental migration to concurrency checking](https://forums.swift.org/t/pitch-3-incremental-migration-to-concurrency-checking/53610)
- [[Pitch #4] Incremental migration to concurrency checking](https://forums.swift.org/t/pitch-4-incremental-migration-to-concurrency-checking/54263)
- [SE-0337: Incremental Migration to Concurrency Checking](https://forums.swift.org/t/se-0337-incremental-migration-to-concurrency-checking/54570)

### プロポーザルドキュメント

- [Incremental migration to concurrency checking](https://github.com/apple/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md)

### 関連PR

- [Add @_predatesConcurrency attribute for declarations.](https://github.com/apple/swift/pull/40220)
- [Rename @_predatesConcurrency to @preconcurrency.](https://github.com/apple/swift/pull/40680)
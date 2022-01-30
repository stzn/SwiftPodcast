# Swift Concurrencyの段階的な導入について考える

- [Swift Concurrencyの段階的な導入について考える](#swift-concurrencyの段階的な導入について考える)
  - [概要](#概要)
  - [内容](#内容)
    - [適用が難しい箇所](#適用が難しい箇所)
      - [ライブラリを遡ってアノテーションを追加する](#ライブラリを遡ってアノテーションを追加する)
      - [Swift Concurrency未対応のモジュールに`Sendable`チェックを適用する](#swift-concurrency未対応のモジュールにsendableチェックを適用する)
    - [解決策](#解決策)
    - [詳細](#詳細)
      - [リカバリ動作](#リカバリ動作)
      - [Concurrencyチェックモード](#concurrencyチェックモード)
      - [名前付型宣言の`@preconcurrency`属性](#名前付型宣言のpreconcurrency属性)
      - [`Sendable`準拠の状態](#sendable準拠の状態)
      - [`@preconcurrency`と`Sendable`プロトコル](#preconcurrencyとsendableプロトコル)
      - [`@preconcurrency`属性と`import`宣言](#preconcurrency属性とimport宣言)
    - [Source互換性](#source互換性)
    - [ABI安定性への影響](#abi安定性への影響)
    - [APIレジリエンスへの影響](#apiレジリエンスへの影響)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [関連PR](#関連pr)

<!-- 最後にTable of Contentsを入れる -->


## 概要

Swift5.5で、値が安全にTaskやactorの境界を跨いで安全に受け渡しができることを示す`Sendable`プロトコルや、例えばMainActorと適切に同期するためのglobal actorなど、データ競合を排除するためのメカニズムが導入された。しかし、Swift5.5だと、まだSwift Concurrencyに対応していないモジュールとやり取りするのがあまりにも骨が折れる作業であることがわかっていたため、この`Sendable`やMainActorの全ての使用も強制はしていない。開発者がSwift Concurrencyをサポートしたり、Swift Concurrency未対応の他のモジュールとやり取りするためのコードのマイグレーションを支援して、データ競合を排除するためにSwiftのエコシステムにスムーズなパスを提供する。

## 内容

Swift Concurrencyは同時並行プログラミング内でデータ競合を排除するための分離された状態を提供するメカニズムを探している。主なメカニズムは、`Sendable`チェック。Taskやactorの境界を跨いでデータを送信するAPIの入力値は`Sendable`に準拠していなければならない。つまり、安全に遅れる型は準拠が宣言されていて、コンパイラは`Sendable`に準拠した型のみが含まれているかどうかをチェックする((ただし、型の作者がnon-`Sendable`な型でも安全に使えるように実装していることを明示している場合を除く)。

Swiftは既に既存のコードがたくさんあり、CやObjective-Cのコードとやり取りも行っており、それらの同時並行処理は、`Sendable`チェックが理解(コンパイル)できるような方法では実装されていない。しかし、それが対応を待つのではなく、引き続き使い続けたい。

### 適用が難しい箇所

#### ライブラリを遡ってアノテーションを追加する

多くの既存のAPIはこれまでおこなってきた同時並行処理を正式にSwift Concurrencyが理解できるように対応させる必要があるものの、これまでのところそれをコンパイラに伝えることができていない。例えば、UIKitのメソッドやプロパティはメインスレッドで実行されるべきだが、`@MainActor`属性導入以前は、コンパイラに伝えることができず、ドキュメントの記載や実装内のアサーションでのみしか実現できなかった。

そのため、多くのモジュールはどこにSwift Concurrencyのアノテーションを追加するかを決めるためにAPIを包括的に監視する必要がある。さらに、もし今あるツールでそれを実行しようとすると、source breakが間違いなく発生するだろう。例えば、あるメソッドに`@MainActor`をつけると、Swift Concurrency未対応のプロジェクトは、正しく使っているのにコンパイラにMainActorで実行していることを伝える手段がないため、このメソッドは使えなくなる。

いくかの場合だと、ABI breakも起こす可能性がある。例えば、関数型の`@Sendable`属性やジェネリックパラメータへの`Sendable`の制約は、`Sendable`への準拠は呼び出し規約に影響がない場合でも(余計なwitness tableのパラメータがいらないなど)、マングリングされた関数名に組み込まれてしまう。型チェック中にこの制約を適用するためのメカニズムが必要であるものの、あたかも`Sendable`が存在しないようにコードは生成される。

つまり、必要なのは、

- Swift Concurrency対応済のモジュールをインポートするSwift Concurrency未対応コードへの「互換モード」の正式な仕様
- Swift Concurrencyによってシグネチャが変わってしまったので、この「互換モード」で特別な対応が必要な時に宣言にマークを付ける方法

#### Swift Concurrency未対応のモジュールに`Sendable`チェックを適用する

ライブラリがSwift Concurrencyのアノテーションを追加するのを追いかけるのは時間がかかると思われる。`Sendable`チェックを適用を開始する前に全てのライブラリが更新されるまで待つのは現実的ではない。

モジュールはインポートされている側へ不完全なアノテーションへのワークアラウンドが必要。これは、インポートされた宣言を微調整するか、エラーを無視するようにコンパイラに使えるかのどちらかである。ただ、どんな方法にせよ、あまり面倒なことにはしたくない。例えば`Sendable`として扱って欲しいnon-Sendable型の全部の変数にマークを付けるなどはとても辛い。

また、最終的にライブラリでSwift Concurrencyに対応した時にも何が起きたのかに特別な注意しなければならず、利用側で間違った同時並行処理の挙動が想定されていた場合はそれを明らかにする。

例えば

1. `Geomerty`モジュールから`Point`型をインポートする
2. `Geomerty`モジュールがSwift Concurrency未対応のタイミングで`Sendable`チェックを適用する
3. 公開されている情報から`Point`型はおそらく`Sendable`であると判断する
4. `Sendable`チェックを抑制する(`@unchecked`を付けるなど)
5. `Geomerty`モジュールの著者が後で`Point`型の実装を見て、同時並行処理で渡されると安全ではないと判断し、non-`Sendable`としてマークする

この状況でライブラリの最新バージョンを使ってアプリをリビルドした場合、どうなるだろうか？

理想を言うと、Swiftはこのバグを抑制し続けるべきではない。結局は`Geomerty`モジュールのチームはnon-`Sendable`としてマークしているため、推測よりも信頼できる。一方で、この変更は、リグレッションではないので、利用側のプロジェクトがリビルドできなくなるべきでもない。最新の`Geomerty`モジュールはバグを追加したわけではなく、既にバグがあったということが明らかになっただけ。これは現状の改善です。バグが診断される方がバグが隠れているよりも良い。

しかし、Swiftがこのバグに対してエラーなどを出力すると、昨日ビルドできたものができなくなるため、`Geomerty`モジュールのアップデートを延期したり、ライブラリの著者にライブラリのアップデートを遅らせるようにプレッシャーをかけるかもしれない。そこで、安全だと想定していたものがあとで間違っていたとわかった場合は、Swiftはエラーではなくワーニングを出して、ビルドはできるもののバグに気づくようにするべき。

つまり、

- 特定の宣言やモジュールにSwift Concurrencyのアノテーションが足りない場合は、エラー診断を抑制するメカニズム
- Swift Concurrencyのアノテーションが追加された後はこの診断を復活させるが、ワーニングのみを出力するルール

ルールが必要になる。

### 解決策

このSwift Concurrencyのアノテーションの適用(特に`Sendable`チェック)をサポートする機能群を導入する。これらの機能は、Swift Concurrency適用のための下記のワークフローを達成するために設計されている。

- Swift Concurrencyの機能(async/awaitやactorなど)を適用する(Swift6モードや`-warn-concurrency`フラグを追加する)ことで、Concurrencyチェックを有効にする。こうすると、新しいワーニングやエラーが発生するConcurrencyチェックに違反した際は、新しくワーニングやエラーが発生する。
- この問題を解決を開始する。他のモジュールからの型が関わっている場合、fix-itが`@preconcurrency import`はという特殊なインポートが提案される。これはワーニングを抑制する。
- 一度問題が解決したら、より広い範囲のビルドに変更を統合する。
- 将来のある時点で、インポートしているモジュールが`Sendable`の準拠や他のConcurrencyアノテーションを追加して更新されるかもしれない。その場合かつもし新しく追加された制約に違反していた場合、この間違いをワーニングで教えてくれる。これは潜在的なバグなので直そう。
- このバグを直したら、あるいはバグはないかもしれないが、`@preconcurrency import`は不要だというワーニングが現れるので`@preconcurrency`を消そう。この時点からこのモジュールに関連した`Sendable`チェックのエラーは`@preconcurrency import`の利用が提案されず、Swift6モードではエラーになってビルドができなくなる。

これを達成するために、下記の機能が並行に必要になる

- Swift6モードでは、全てのコードで`Sendable`の準拠不足や他のConcurrency関連の違反は実装ミスとして全部エラーになる。Swift6よりも古い言語バージョンでは、`-warn-concurrency`フラグを追加することでこのエラーをワーニングにする。
- 名前付型に`@preconcurrency`属性を適用すると、その宣言はConcurrencyチェックが行われるように更新されたことを示す。そのため、Concurrencyチェックに違反する場合でもコンパイラはSwift5モードでも利用できるように、Concurrency適用前のバイナリと相互運用できるようにする
- import宣言に`@preconcurrency`属性を適用する場合は、そのモジュールからインポートしたnon-`Sendable`型が、その型が`Sendable`として使用できないまたは満たされていない制約があることを明示的に宣言しているときのみ、`Sendable`が必須の箇所で使われていた場合のみ診断するようにコンパイラに伝える。この場合はエラーではなくワーニングが発生するに留まる。

### 詳細

#### リカバリ動作

このプロポーザルで、エラーをワーニングとして出力したり、抑制する場合は、コンパイラが下記のように振る舞うことでリカバリしたことを意味する。

- 名前付型は`Sendable`に準拠していない
- 関数型に`@Sendable`やglobal-actorアノテーションが付いていない

#### Concurrencyチェックモード

Swiftの全てスコープで、次の2つのConcurrencyチェックモードの内のいずれかを備えていると説明できる

- Strict Concurrencyチェック: `Sendable`準拠不足やglobal-actorアノテーションは診断される。Swift6モードの場合、これらは基本的にエラーになる。Swift5モードでは、`@preconcurrency import`を通して見える名前付型の宣言の場合、ワーニングになる
- Minimal Concurrencyチェック: `Sendable`準拠不足やglobal-actorアノテーションはワーニングで診断される。名前付型の宣言に`@preconcurrency`属性を適用した場合は、特殊な効果が発生して多くの診断を抑制する

トップレベルスコープでのConcurrencyチェックモードは:

- Swift6以降でコンパイルされた場合と、それよりも古い言語バージョンで`-warn-concurrency`フラグが使われている場合、またはファイルがモジュールインターフェイスとして解析された場合、Strictになる
- それ以外はMinimalになる

子レベルスコープでのConcurrencyチェックモードは:

- 親レベルスコープのConcurrencyチェックモードがMinimalで子レベルスコープで次の条件がtrueである場合:
  - global-actor属性を明示しているクロージャ
  - `async`や`@Sendable`なクロージャやオートクロージャ(ただし、親のモードがMinimalモードの場合は`@Sendable`と推論されるかどうかに影響を与える可能性があることに注意)
  - 明示的にnonisolatedやglobal-actor属性を持つ宣言
  - `async`や`@Sendable`がマークされた関数、メソッド、イニシャライザ、アクセサ(get/set)、変数、subscript
  - `actor`
  - それ以外は親スコープと同じ

実装詳細: 子スコープがStrictかMinimalかを決定するロジックは`swift::contextRequiresStrictConcurrencyChecking()`に実装されている。インポートされたC言語の宣言はMinialになる。

#### 名前付型宣言の`@preconcurrency`属性

これらのConcurrency上での挙動に対応するためには、著者は既存の宣言を変更せねばならず、Concurrency未対応コードのsorce breakingやこれまでにコンパイルしていたバイナリと相互運用している場合にABI-breakingを起こす可能性がある。特に

- `@Sendable`やglobal-actor属性を関数型に追加
- ジェネリック構文に`Sendable`制約を追加
- global-actor属性を宣言に追加

名前付型宣言に適用されると、`@preconcurrency`属性はSwift Concurrencyに完全に対応する前からその宣言が存在していたことを示し、コンパイラはsourceやABI-breakingを起こさないようにするべき。これは`enum`、`enum`の`case`、`struct`、`class`、`actor`、`protocol`、`var`、`let`、`subscript`、`init`または`func`宣言に使える。

名前付型宣言に`@preconcurrency`属性が使われていると:

- Concurrencyの機能が全く使われていないように名前がマングリングされる
- 利用者側がMinimal Concurrencyチェックモード内で使った場合は、コンパイラはこの不一致の診断を全て抑制する
- ABIチェックはダイジェスト(※)を生成する時にこれらの機能を削除する

※ ダイジェストは、ハッシュ関数（1方向関数）を用いて、ある長さを持つデータを固定長のデータに変換したもの

Objective-Cの宣言は、常に`@preconcurrency`属性がついているようにインポートされる。

Main Actor上でしか実行できない関数で別のタスクから提供されたクロージャを実行することを想定する

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

しかし、`@preconcurrency`を`doSomethingThenFollowUp`宣言に追加すると、`@MainActor`や`@Sendable`が削除され、エラーは除外され、Concurrency以前と同じ型推論がされるように調整される。これはStrictとMinimalモードで違いが見える:

```swift
func minimal() {
    let fn = doSomethingThenFollowUp // 型は (( )-> Void) -> Void
}

func strict() async {
    let fn = doSomethingThenFollowUp // 型は @MainActor (@Sendable ( )-> Void) -> Void
}
```

#### `Sendable`準拠の状態

ある型は次の3つの`Sendable`準拠の状態を備えていると説明できる:

- 明白な`Sendable`: 明示的に`Sendable`が宣言されている、もしくはルールに基づいて準拠が推論される場合
- 明白なnon-`Sendable`: 型に対してSendable準拠が宣言されているが、それが利用できないか、型が満たさない制約がある場合、または型が厳密な同時実行性チェックを使用するスコープで宣言されている場合。(※)
- 暗黙的なnon-`Sendable`: 型に対してSendable準拠が宣言されていない場合

※ もしモジュールがSwift6モードから`-warn-concurrency`フラグを指定している場合、全ての型は明白に`Sendable`か明白にnon-`Sendable`になる。

型は`Sendable`準拠にunavailableを付けることで明白なnon-`Sendable`にできる。

```swift
@available(unavailable, *)
extension Point: Sendable { }
```

#### `@preconcurrency`と`Sendable`プロトコル

いくつかの既存のプロトコルは全て`Sendable`である必要がある型を記述している。そのようなプロトコルがConcurrencyに対応する際、`Sendable`プロトコルをおそらく継承するだろう。しかし、そうするとこのプロトコルに準拠した既存の型は`Sendable`だと想定されるので壊れるだろう。この問題は標準ライブラリの`Error`と`CodingKey`プロトコルに影響があるためSE-0302でも記載されていた。

```swift
protocol Error: /* 新しく追加 → */ Sendable { ... }

class MutableStorage {
  var counter: Int
}
struct ProblematicError: Error {
  var storage: MutableStorage // ❌　error: Sendable struct ProblematicError has non-Sendable stored property of type MutableStorage
}
```

これを解決するために、SE-0302では`Error`プロトコルに`Sendable`を追加した

この移行を簡単にするために、`Error`を通した`Sendable`準拠の問題はSwift6モード未満ではワーニングにダウングレードすることになっていた。
今回は、`@preconcurrency`属性が付けられ、`Sendable`プロトコルを継承する全てのプロトコルに適用するために、この`Error`と`CodingKey`プロトコルへの特別ルールを置き換え、これらの2つの標準ライブラリのプロトコルは`@preconcurrency`を使用するようになる。

```swift
@preconcurrency protocol Error: Sendable { ... }
@preconcurrency protocol CodingKey: Sendable { ... }
```

#### `@preconcurrency`属性と`import`宣言

`@preconcurrency`属性は`import`宣言にも適用することができ、そのモジュールからインポートした型が起こるConcurrencyチェック違反の強さを軽減することをコンパイラに伝える。これはConcurrency未対応のモジュールをインポートする際に活用できる。そうした場合、`Sendable`にする必要のあるすべての型にアノテーションが付けられたときにコンパイラが通知してくれる。また、そのモジュールに関して誤った想定が修正されるまで、プロジェクトをコンパイルし続けるための一時的なエスケープハッチとしても機能する。

importに`@preconcurrency`属性を付けと、事実上次のルールになる:

- もし`Sendable`が必要なところに暗黙的なnon-`Sendable`が使われていた場合
    - `@preconcurrency import`を通してその型が見える場合は、Swift6未満の場合は診断は抑制され、Swift6以降はワーニングが出力される
    - それ以外は、診断は通常通りに出力されるが、別の診断として`@preconcurrency import`がワークアラウンドとして提案される

- もし`Sendable`が必要なところに明白non-`Sendable`が使われていた場合
    - `@preconcurrency import`を通してその型が見える場合は、Swift6未満の場合はワーニング、Swift6以降はエラーが出力される
    - それ以外は、診断は通常通りに出力される

- もし`@preconcurrency`属性が「未使用」な場合(※)、それを削除する提案がワーニングとして出力される

※ たとえば、お互いに影響を与える型をインポートする`@preconcurrency import`のペアの1つを削除することを推奨するのに十分なほど診断が改良できるかどうかわからないため「未使用」をこれ以上具体的に定義できない

### Source互換性

このプロポーザル自体がsource breakを防ぐためのもの。`@preconcurrency`を活用することでMinimalモードでのsource breakを防げるはず。そして依存しているモジュールがConcurrency未対応でも、Strictモードを適用していても`@preconcurrency import`でConcurrencyチェックを弱めることができるため、Source互換性を保持できる。

### ABI安定性への影響

`@preconcurrency`はABIを変更しない。もし既にConcurrencyを適用しているコードに追加した場合は、ABIを破壊するだろう。一方で,
`@preconcurrency`と同時にもしくは後から付ける場合は、ABI破壊は起きない。

`Sendable`は、追加のメタデータを出力したり、渡す必要のあるwitness tableを持ったり、呼び出し規約やABIの他のほとんどの部分に影響を与えたりしないように設計されているため、`Sendable`準拠適合エラーを無効にする`@preconcurrency`の戦術は現在のABIと互換性がある。名前マングリングにのみ影響する。

今回のプロポーザルでは、他にABIへの影響はない。

### APIレジリエンスへの影響

名前付型宣言の`@preconcurrency`はモジュールインターフェイスへの出力が必要になるだろう。これは事実上、レジリエンスを損なうような方法でAPIを進化させる機能である。

`import`宣言の`@preconcurrency`は、モジュールインターフェイスに出力する必要はない。モジュールインターフェイスは、ConcurrencyチェックがワーニングであるStrictモードを使用するため、足りない準拠を許容するのに十分な「余裕」がある。(いつものように、モジュールインターフェイスをコンパイルすると、デフォルトでワーニングが表示されない。）

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
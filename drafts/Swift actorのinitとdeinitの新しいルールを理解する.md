# Swift actorのinitとdeinitの新しいルールを理解する

- [Swift actorのinitとdeinitの新しいルールを理解する](#swift-actorのinitとdeinitの新しいルールを理解する)
  - [用語](#用語)
    - [`actor`](#actor)
    - [global-`actor`](#global-actor)
    - [Sendable](#sendable)
    - [その他](#その他)
  - [概要](#概要)
  - [内容](#内容)
    - [`actor`のnon-asyncイニシャライザと全てのデイニシャライザの前提知識](#actorのnon-asyncイニシャライザと全てのデイニシャライザの前提知識)
    - [現在の問題点](#現在の問題点)
      - [`non-async`のイニシャライザの`self`のアクセスの制限が厳しすぎる](#non-asyncのイニシャライザのselfのアクセスの制限が厳しすぎる)
      - [deinitでデータ競合を起こすリスクがある](#deinitでデータ競合を起こすリスクがある)
      - [格納プロパティのデフォルト値を設定する式の`actor`分離](#格納プロパティのデフォルト値を設定する式のactor分離)
      - [イニシャライザの委譲](#イニシャライザの委譲)
    - [解決策](#解決策)
      - [非委譲イニシャライザの場合](#非委譲イニシャライザの場合)
        - [Flow-sensitiveな`actor`の分離(GAITを除く`actor`の非委譲イニシャライザ)](#flow-sensitiveなactorの分離gaitを除くactorの非委譲イニシャライザ)
          - [`isolated self`のイニシャライザ](#isolated-selfのイニシャライザ)
          - [nonisolated selfのイニシャライザ](#nonisolated-selfのイニシャライザ)
        - [GAITs(global-`actor`に分離された型)](#gaitsglobal-actorに分離された型)
        - [`actor`イニシャライザのルールを簡単にまとめると](#actorイニシャライザのルールを簡単にまとめると)
      - [委譲イニシャライザ](#委譲イニシャライザ)
        - [シンタックス](#シンタックス)
        - [Isolation](#isolation)
      - [Sendability](#sendability)
      - [デイニシャライザ](#デイニシャライザ)
      - [global-`actor`分離と`actor`インスタンスメンバ](#global-actor分離とactorインスタンスメンバ)
        - [余分な分離の削除](#余分な分離の削除)
      - [Swift5.5からのsource breaking](#swift55からのsource-breaking)
      - [検討したが却下になった考え](#検討したが却下になった考え)
        - [selfが完全に初期化された後はnonisolationにする](#selfが完全に初期化された後はnonisolationにする)
        - [nonisolated selfのイニシャライザ内のプロパティへのアクセスにawaitをつける](#nonisolated-selfのイニシャライザ内のプロパティへのアクセスにawaitをつける)
        - [`actor`のasyncデイニシャライザ](#actorのasyncデイニシャライザ)
        - [委譲イニシャライザのみ`Sendable`引数を必須にする](#委譲イニシャライザのみsendable引数を必須にする)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [関連PR](#関連pr)
    - [その他](#その他-1)

## 用語

### `actor`

同時並行処理中でもデータ競合(※)を起こさずに内部の可変状態を安全に扱えるようにした参照型。

※ データ競合とは、複数スレッドから同じメモリの可変状態に書き込み/読み込み処理が同時に起きること。

例えば、下記の例は`increment()`メソッド内の`value`の更新タイミング次第で出力される値が変わり、1と1や2と2など予期せぬ値になることがある。

```swift
class Counter {
    var value = 0
    func increment() -> Int {
        value += 1
        return value
    }
}
let counter = Counter()
Task.detached { print(counter.increment()) } // データ競合
Task.detached { print(counter.increment()) } // データ競合
```

`actor`の場合、データ競合が起こらずに1と2または2と1になる。

```swift
`actor` Counter {
    var value = 0
    func increment() -> Int {
        value += 1
        return value
    }
}
let counter = Counter()
Task.detached { print(counter.increment()) } // データ競合しない
Task.detached { print(counter.increment()) } // データ競合しない
```

※ データ競合と競合状態

似ているものに「競合状態」がある。これは、各スレッドの処理の実行順序が不定で、同じ入力しても出力結果が変わってしまう状態を指す。`actor`はデータ競合を防ぐが、設計次第で競合状態を起こす可能性はある。

### global-`actor`

グローバルな状態を一つの`actor`の内部状態として分離するための`actor`。プロセスに一つのインスタンスしか存在しない。複数の異なる型やメソッドなどに適用でき、安全にglobal-`actor`のエグゼキュータ内でデータ競合を起こさずに安全に同時並行処理で扱うことができる。

### Sendable

同時並行に使用しても安全であることを示すプロトコル。引数などで受け渡される際にコピーが発生する。準拠するための条件は複数あるが、例えば、``actor``はデータ競合を防ぐ仕組みなっているため全て準拠している。また、publicではない`Sendable`に準拠したメンバのみで構成された`struct`や`enum`は自動で準拠される。`class`は`final`かつ`Sendable`に準拠したメンバのみで構成され、かつそれらが全てimmutable(`let`)であれば`Sendable`を明示的に記載することで準拠できる。
### その他

- 分離: `actor`によってデータ競合から守られている(他から分離されている)状態
- 委譲: 他のメソッドやクラスなどに処理を任せること。例:イニシャライザの中で別のイニシャライザを呼ぶイニシャライザを委譲イニシャライザと呼ぶ
- 中断: 英語ではsuspension。async関数の中で処理の実行を一度中断して、その処理を行なっていたスレッドで他の処理が実行できるようにすること。`await`キーワードが使われる
- 中断点: 中断されるポイント
- エグゼキュータ(executor): async関数を実行するオブジェクト。`actor`は内部に一度に一つの処理を実行するシリアルエグゼキュータを持つ
- ホップ: ある`actor`のエグゼキュータに移動する(飛ぶ)こと。こうすることで`actor`に分離されている処理はこのエグゼキュータで実行される
- GAIT: Global-`actor` isolated typesの略。global-`actor`に分離された型。例:@MainActorの付いたクラスなど

## 概要

Swift5.5で導入された`actor`に分離された状態の生成や解放の過程においていくつかの側面(リスク)が見逃されていた。この提案では、`actor`の定義を強化して、`actor`に分離された状態がいつ開始/終了するかや`init`/`deinit`でできることを明確にする。Swift5.6で導入予定。

## 内容

### `actor`のnon-asyncイニシャライザと全てのデイニシャライザの前提知識

`actor`の`non-async`イニシャライザと全てのデイニシャライザは`actor`コンテキスト外で実行される。`actor`の`non-async`イニシャライザには`await`なしで任意のスレッドからアクセスでき、デイニシャライザは参照カウンタが0になった時点で任意のスレッドから呼び出される。そのため、`actor`のエグゼキュータにホップすることができず、データ競合を起こす可能性がある(逆にasyncイニシャライザは`actor`のエグゼキュータにホップすることができるため、データ競合を防ぐことができる)。

### 現在の問題点

#### `non-async`のイニシャライザの`self`のアクセスの制限が厳しすぎる

`actor`はデータ競合を防ぐために、内部の状態の更新は`actor`内で行われる必要がある。そのために、`await`することで`actor`のコンテキストにホップしなければならない場合がある。

しかし、`non-async`のイニシャライザは`await`してホップすることができないため、データ競合を防ぐことができない。

```swift
`actor` Clicker {
    var count: Int
    func click() { self.count += 1 }

    init(bad: Void) {
        self.count = 0
        // `actor`コンテキストにホップすることができない

        Task { await self.click() }

        self.click() // 💥 ここで変更できてしまうとTask内とデータ競合する

        print(self.count) // 💥 1 または 2が出力される(不定)
    }
}
```

これを防ぐためにSwift5.5では`self`をクロージャでキャプチャしたり、クロージャ内でインスタンスメソッドを呼び出すとワーニングを出していた(= Swift6ではエラーになる予定だった)。しかし、`self`使うこと自体が全て危険というわけではない。

例えば、下記の`self.announce()`は`nonisolated`で`actor`に分離された状態であるかどうかとは無関係にも関わらずエラーになる。

```swift
`actor` Clicker {
    var count: Int
    func click() { self.count += 1 }
    nonisolated func announce() { print("performing a click!") }

    init(ok: Void) {
        self.count = 0
        Task { await self.click() }
        self.announce() // Swift 5.5では❌だが、データ競合は起きない。 ⚠️This use of `actor` 'self' can only appear in an async initializer
    }
}
```

#### deinitでデータ競合を起こすリスクがある

`deinit`でもデータ競合やselfの(余計な)延命を起こす可能性がある。

下記の例で考えてみる。

```swift
`actor` Clicker {
    var count: Int = 0

    func click(_ times: Int) {
        for _ in 0..<times {
            self.count += 1
        }
    }

    deinit {
        let old = count
        let moreClicks = 10000

        Task { await self.click(moreClicks) } // ❌ deinit以降も　selfを延命させている可能性がある

        for _ in 0..<moreClicks {
            self.count += 1 // 💥 ここで変更がTask内とデータ競合する可能性がある
        }

        assert(count == old + moreClicks) // 💥 データ競合によって失敗するかもしれない
    }
}
```

これがGAITの場合はもっと複雑になる。GAITは他のインスタンスとエグゼキュータを共有している`actor`(※)に近い。`actor`やGAITのエグゼキュータがそのインスタンスだけで排他的に所有されているわけではない場合に、non-Sendableのプロパティにアクセスすると安全ではない。

※ 他のインスタンスとエグゼキュータを共有している`actor`は、(まだ導入されていない)Custom Executorについての言及。

```swift
class NonSendableAhmed {
    var state: Int = 0
}

@MainActor
class Maria {
    let friend: NonSendableAhmed

    init() {
        self.friend = NonSendableAhmed()
    }

    init(sharingFriendOf otherMaria: Maria) {
        // friendはnon-Sendableだが、
        // MariaがMainActorに守られた型で
        // otherMariaも全てMainActorからアクセスされているためOK
        self.friend = otherMaria.friend
    }

    deinit {
        friend.state += 1
        // 💥 deinitはMainActorに守られていないため
        // この変更は内部のNonSendableAhmedに他のスレッドと同時にアクセスされる可能性があるため
        // データ競合が起こる可能性がある
    }
}

func example() async {
    let m1 = await Maria()
    let m2 = await Maria(sharingFriendOf: m1)
    doSomething(m1, m2)
}
```

二つの`Maria`はエグゼキュータを共有しているので、同じnon-Sendableな可変状態を参照することができ、`deinit`はエグゼキュータから`nonisolated`なことから、これにアクセスすると同時並行なアクセスが発生してしまう。

また、`deinit`が呼ばれた後に`self`の参照カウンタが増えた場合の挙動はundefinedなため、ランダムにクラッシュが発生するリスクがある。(これは通常の`class`でも同じことが発生する)

#### 格納プロパティのデフォルト値を設定する式の`actor`分離

現在は、`class`、`struct`、`enum`ではglobal-`actor`に分離されたプロパティを保持できる。プログラマがデフォルト値を設定する式を指定している場合、その値はそれぞれの型の`non-async`非委譲イニシャライザの中で計算される。

問題なのは、それらがそれぞれのglobal-`actor`のエグゼキュータ上で実行されるため、不可能な制約が生成できてしまう。

```swift

@MainActor func getStatus() -> Int { /* ... */ }
@PIDActor func genPID() -> ProcessID { /* ... */ }

class Process {
    @MainActor var status: Int = getStatus()
    @PIDActor var pid: ProcessID = genPID()

    init() {} // 問題:このinit内はどの`actor`で分離されている？
}
```

この例はSwift5.5でエラーにならないが、`status`と`pid`が別のglobal-`actor`に分離されているため、`non-async`なイニシャライザ内で一つの`actor`に分離できない。また、`non-async`なので、適切な`actor`にホップすることもできず、結局、`getStatus`も`getPID`も適切なエグゼキュータで実行できない状態になっている。

#### イニシャライザの委譲

Swiftでは全てのnominal型(`class`や``struct``などの名前のある型)はイニシャライザの委譲(他のイニシャライザに残りの初期化を任せることができる機能)をサポートしている。classは継承があるため他よりも複雑で、委譲するイニシャライザには`convenience`修飾子をつけなければならない。値型は継承がないためよりシンプル。任意の`init`で委譲可能だが、その場合は全実行パスで委譲または`self`への割り当てが必要になる。

```swift
`struct` S {
    var x: Int
    init(_ v: Int) { self.x = v }
    init(b: Bool) {
        if b {
            self.init(1)
        } else {
            self.x = 0 // ❌　'self' used before 'self.init' call or assignment to 'self'
        }
    }
}
```

`actor`は継承をサポートしておらず、`convenience`修飾子をつけるかどうかも明記していなかったが、Swift5.5では委譲する際に
`convenience`修飾子が必須になっている。

### 解決策

#### 非委譲イニシャライザの場合

`actor`やGAITの非委譲イニシャライザはその型の全ての格納プロパティを初期化しなければならない。

##### Flow-sensitiveな`actor`の分離(GAITを除く`actor`の非委譲イニシャライザ)

Swift5.5では、「escapingする`self`の使用制限」(escaping-use restriction)が行われていた。escapingする使用とは、

- クロージャでの`self`のキャプチャ
- `self`内のメソッドや計算プロパティを呼ぶ
- 任意の種類の引数(値としてや`autoclosure`や`inout`)に`self`を渡す

がある。しかし、この制限は過剰で、このescapingする使用制限を削除する。代わりに、よりシンプルなルールを提案。

まずイニシャライザを分離のされ方に応じて二つに分類する。

1. `nonisolated self`参照を保持するイニシャライザ
   - `non-async`イニシャライザ
   - global-`actor`に分離されたイニシャライザ
   - `nonisolated`キーワードを持つイニシャライザ

2. `isolated self`参照を保持するイニシャライザ
   - `async`イニシャライザ

###### `isolated self`のイニシャライザ

`async`イニシャライザは`self`が完全に初期化された後、`self`に分離された状態あると考えられ、`actor`のコンテキストにホップが即座に行われる。この`actor`のエグゼキュータへホップする位置を選択することで、`async`イニシャライザ全体で`self`の分離状態を保つことができる。つまり、`self`をescapeして使用する前に、`actor`のコンテキストへのホップは完了している。

ホップするポイントは中断点であると認識することが大事である。`self`の初期化が完了するタイミングは複数存在するので、中断が起きる可能性があるポイントは複数ある。

```swift
`actor` Bob {
    var x: Int
    var y: Int = 2
    func f() {}
    init(_ cond: Bool) async {
        if cond {
            self.x = 1 // 初期化完了
        }
        self.x = 2 // 初期化完了

        f() // ⭕️ `actor`のエグゼキュータ上にいる
    }
}
```

`Bob.init`に明示的に中断点(`await`)をマークしようとする場合の問題点は、プログラマがそれを追跡するのが困難で、シンプルなリファクタリングでも中断点を同じ位置に保つことができないこと。例えば、デフォルト値を追加したり消したり、新しい格納プロパティを追加することで、ホップが発生する箇所に大きく影響するかもしれない。

上記の例をちょっと変えてみる。

```swift
`actor` EvolvedBob {
    var x: Int
    var y: Int
    func f() {}
    init(_ cond: Bool) async {
        if cond {
            self.x = 1
        }
        self.x = 2
        self.y = 2 // 初期化完了

        f() // ⭕️ エグゼキュータ上にいる
    }
}
```

違いは`y`のデフォルト値を消しただけだが、中断点は大きく変わっている。もし`await`が必要な場合、この中断点が移動した理由をプログラマに説明するのが難しい。

まとめると、プログラマがこのポイントを明示する代わりに、`self`が一度初期化したら暗黙的にホップするようにする。理由として:

- 継続的に中断点を見つけて更新するのはプログラマにストレス
- プロパティへのシンプルな値の格納が処理の中断を引き起こすのは言語側の実装詳細であり、プログラマに説明するのが難しい
- 処理の中断を明示するメリットが低い。中断が発生するまで`self`への参照はユニークであることはわかっているので、`actor` reentrancy(※)が発生することがない
- 暗黙的に処理の中断が発生するメリットがデメリットを超えると判断された`async let`のような[前例](https://github.com/apple/swift-evolution/blob/main/proposals/0317-async-let.md#requiring-an-awaiton-any-execution-path-that-waits-for-an-async-let)が存在する

※ `actor` reentrancy
`actor`上で中断が発生した際に、他の`actor`のタスクをエグゼキュータ上で実行できること

関連ドキュメント: https://github.com/apple/swift-evolution/blob/main/proposals/0306-`actor`s.md#`actor`-reentrancy

これらの暗黙のホップの効果は、プログラマから見ると、`async`イニシャライザにルールが追加されていないように見えること。他の`async`メソッドと同じように、イニシャライザは全体で`actor`に分離された状態になっているように見なせる。ほとんどのケースでイニシャライザにホップが挿入されるFlow-sensitiveなポイントは実装詳細として無視することができる。ただし例外としては下記のようなものもある:
```swift
`actor` OddActor {
    var x: Int
    init() async {
        let name = Thread.current.name
        self.x = 0 // 初期化完了
        assert(name == Thread.current.name) // 失敗する可能性あり
    }
}
```

`OddActor.init`の呼び出し側は、呼び出し時に`await`が必要なため、中断しないとは想定しないはず。

###### nonisolated selfのイニシャライザ

このカテゴリには`non-async`イニシャライザや`self`とは異なったコンテキスト(global-`actor`)に分離したイニシャライザを含んでいる。`actor`のメソッドと異なり、同期する必要のある`actor`インスタンスは存在しないため、`actor`の`non-async`イニシャライザに`await`は必要ない。さらに、もし安全ならば、同期なしで格納プロパティにアクセスすることができる。

ただし、格納プロパティにアクセスするには`actor`のインスタンスをまず立ち上げる(初期化を完了させる)必要がある。これは`self`への参照のアクセスが排他的であるということに依存しており、より弱い形式の分離であると見なされる。もし`self`がイニシャライザをescapeして使用される場合、このユニークさを保証するためには時間コストのかかる分析をしなければならない。そこで、直接格納プロパティへアクセスする以外に`self`を使っている期間は`self`に`nonisolated`な状態に変化(衰退)する。(Flow-Sensitive `actor`分離(※))。これは特定の制御フローで一度起きたらイニシャライザの最後まで保たれる。

※ データフロー解析を使って制御フローの位置に応じて`actor`の分離状態を変えること。
https://ja.wikipedia.org/wiki/データフロー解析

下記に`nonisolated`へ衰退する例を示す。

1. 引数に`self`を渡す(以下を含む)
   - `self`のメソッドを呼ぶ
   - プロパティラッパを使ったものを含めた`self`の計算プロパティへアクセスする
   - 監視プロパティ(つまり、`didSet`や`willSet`持つプロパティ)をトリガーする
2. クロージャやautoclosureで`self`をキャプチャする
3. `self`を他の変数に割り当てる(コピーする)

具体的なコードで見てみる。

```swift
class NotSendableString { /* ... */ }
final class Address: Sendable { /* ... */ }
func greetCharlie(_ charlie: Charlie) {}

`actor` Charlie {
    var score: Int
    let fixedNonSendable: NotSendableString
    let fixedSendable: Address
    var me: Charlie? = nil

    func incrementScore() { self.score += 1 }
    nonisolated func nonisolatedMethod() {}

    init(_ initialScore: Int) {
        self.score = initialScore
        self.fixedNonSendable = NotSendableString("Charlie")
        self.fixedSendable = NotSendableString("123 Main St.")

        if score > 50 {
            nonisolatedMethod() // ✅ selfのnonisolatedな使用
            greetCharlie(self)  // ✅ selfのnonisolatedな使用
            self.me = self      // ✅ selfのnonisolatedな使用
        } else if score < 50 {
            score = 50
        }

        assert(score >= 50) // ❌ selfのnonisolatedな使用後に、可変な分離された格納プロパティへアクセスできない

        _ = self.fixedNonSendable // ❌ selfのnonisolatedな使用後に、non-Sendableなプロパティへアクセスできない
        _ = self.fixedSendable

        Task { await self.incrementScore() } // ✅ selfのnonisolatedな使用
    }
}
```

この例のポイントは、`if else`で複数の制御フローがイニシャライザに導入されていること。
最初の条件分岐の中で複数回`self`の`nonisolated`な使用をしている。一方の分岐(`else if`)ではまだ`score`への読み書きはできる。一方で`assert`の時点では、ここに到達できるブロックの一つで、`self`が`nonisolated`に使用されているため、`self`は`nonisolated`とみなされる。

結果的に、`self`が`nonisolated self`になった後は、アクセス可能な唯一の格納プロパティは`let`で宣言された`Sendable`なもののみとなる。それ以外の格納プロパティへ不正にアクセスした場合は、分離の衰退が発生した不正アクセスよりも前に`self`を使用した場所の一つを指摘する。これは制御フローの観点で、プログラム上のどこに現れるかではない(不正アクセスより下に記述されていたとしても実行順序が先の場合はそこのポイントが指摘される)。

例えば、`defer`を使った`Charlie.init`の別の例を見て考えてみる。

```swift
init(hasADefer: Void) {
    self.score = 0
    defer {
        print(self.score) // ❌ selfのnonisolatedな使用後に、分離された可変の格納プロパティへアクセスできない
    }
    Task { await self.incrementScore() } // note: selfのnonisolatedな使用
}
```

`self.score`の出力をイニシャライザの最後に遅らせている。しかし、`defer`の実行前に`self`がクロージャ内でキャプチャされているため、`self.score`の値の読み込みはいつもデータ競合から安全であるとは限らない。そのため、これはエラーになる。

不正なプロパティアクセスが目に見える形で`nonisolated`に衰退する例として`for`ループがある。

```swift
init(hasALoop: Void) {
    self.score = 0
    for i in 0..<10 {
        self.score += i     // ❌ selfのnonisolatedな使用後に、分離された可変の格納プロパティへアクセスできない
        greetCharlie(self)  // note: selfのnonisolatedな使用
    }
}
```

イテレーションの最初のループしか完全ではないので、この`for`ループでは、`self.score`の変更にもエラーにしなければならない。それ以降のイテレーションではこのループの外側で同時並行にアクセスされる可能性があるため、安全ではない。

**他の例**

`non-async`イニシャライザ以外にも、global-`actor`に分離されていたり、`nonisolated`の付いたイニシャライザも`nonisolated self`を持つ可能性がある。

下記の例を考えてみる。

```swift
func printStatus(_ s: Status) { /* ... */}

`actor` Status {
    var valid: Bool

    // isolatedメソッド
    func exchange(with new: Bool) {
        let old = valid
        valid = new
        return old
    }

    // isolatedメソッド
    func isValid() { return self.valid }

    // awaitを使ってisolatedメソッドを呼び出したnonisolated selfイニシャライザ
    @MainActor init(_ val: Bool) async {
        self.valid = val

        let old = await self.exchange(with: false) // note: selfのnonisolatedな使用
        assert(old == val)

        _ = self.valid //❌ selfのnonisolatedな使用後に、可変なisolatedの格納プロパティへアクセスできない

        let isValid = await self.isValid() // ✅ OK

        assert(isValid == false)
    }
}
```

`await`できる場合、`nonisolated self`を持つイニシャライザから`self`に分離されたメソッドの呼び出しが許可されていることに注目。この呼び出しは、`nonisolated self`の使用とみなされ、格納プロパティへのアクセス以外で最初の`self`へのアクセスになる。その後は、`non-async`イニシャライザと同じように`init`内での格納プロパティへのアクセスはできなくなる。このイニシャライザは`async`なので技術的には`Sendable`な`self.valid`の値を`await`できるが、この状況でのアクセスを禁止することを選んだ。(詳細は[nonisolated selfのイニシャライザ内のプロパティへのアクセスにawaitをつける](#nonisolated-selfのイニシャライザ内のプロパティへのアクセスにawaitをつける)を参照)

##### GAITs(global-`actor`に分離された型)

GAITの`nonisolated`イニシャライザは、`non-async`イニシャライザと同じ状況で、エグゼキュータの保護なしにインスタンスを立ち上げなければならない。つまり、データ競合を生み出す可能性がある。

```swift
@MainActor
class RequiresFlowIsolation<T>
where T: Sendable, T: Equatable {

    var item: T

    func mutateItem() { /* ... */ }

    nonisolated init(with t: T) {
        self.item = t
        Task { await self.mutateItem() }
        self.item = t   // 💥 Task内とデータ競合
    }
}
```

これを防ぐためにGAITの`nonisolated`イニシャライザにもFlow-sensitive `actor`分離を適用する。

分離されたイニシャライザの場合、GAITはイニシャライザの呼び出し前に既に`actor`に分離できている。なぜなら、エグゼキュータはstaticインスタンスなので、GAITインスタンスの未初期化のメモリを割り当てるより前に存在しているから。そのため、GAITのisolatedなイニシャライザは、初期化開始前に正しいエグゼキュータで実行されるためにも呼び出し側で`await`が必要になる。そしてこのエグゼキュータはイニシャライザがリターンするまで確保されているため、GAITの分離されたイニシャライザでは分離された格納プロパティ間でのデータ競合の危険がない。

※ これはSwiftで定義された`class`にのみ適用され、`MainActor`に分離されたObjective-Cからインポートされた`class`はデータ競合が起きないと想定されるため適用されない。


```swift
@MainActor
class ProtectedByExecutor<T: Equatable> {
    var item: T

    func mutateItem() { /* ... */ }

    init(with t: T) {
        self.item = t
        Task { self.mutateItem() }  // ✅ Taskを生成する際に正しいエグゼキュータ上にいる
        assert(self.item == t) // ✅ エグゼキュータを確保しているため、常にtrue
    }
}
```

`nonisolated`な格納プロパティを持つGAITは、データ競合を防ぐために、既存の`Sendable`の制限に頼っている。

##### `actor`イニシャライザのルールを簡単にまとめると

- `async`イニシャライザは`actor`に分離されている。全ての格納プロパティを初期化した時点で`actor`のエグゼキュータにホップして`await`なしで安全に格納プロパティにアクセスできる
- `non-async`イニシャライザは`actor`に分離されていない。全ての格納プロパティを初期化後、格納プロパティへアクセスできるが、一度格納プロパティ以外で`self`にアクセスした場合、イニシャライザ内のそれ以降では、`self`に`nonisolated`な状態になるため、可変な格納プロパティや`let`でもNonSendableな格納プロパティにアクセスできない
- `actor`型のglobal-`actor`に分離されたイニシャライザや`nonisolated`の`async`イニシャライザなどの例外ケースでは、`non-async`イニシャライザと同じカテゴリになる

[テストコード](https://github.com/apple/swift/blob/main/test/Concurrency/flow_isolation.swift)
を見るとわかりやすいかもしれない。

#### 委譲イニシャライザ

ここでは`actor`型とGAITsの委譲イニシャライザに関するシンタックスとルールを定義する。

##### シンタックス

`actor`は参照型だが、委譲イニシャライザに関しては既存の値型と同じ基本的なルールに従う。

1. イニシャライザの本文で`self.init`の呼び出しがあった場合、それは委譲イニシャライザとして扱われる。`convenience`キーワードはいらない
2. 委譲イニシャライザは、`self`が使用される前に全ての実行パスで`self.init`を呼び出さなければならない

`actor`は継承がないため、`class`の複雑さを減らすことができる。GAITの場合は委譲イニシャライザを定義する際は、`class`と同じシンタックスを使う。

##### Isolation

非委譲イニシャライザと多くが同じで、`actor`の委譲イニシャライザも`isolated self`または`nonisolated self`の場合がある。どっちに分類されるかは完全に同じ(`non-async`イニシャライザは`nonisolated self`など)。

しかし、格納プロパティを必ず初期化する必要がないため、委譲イニシャライザはよりシンプルなルールを持つ。つまり、Flow-sensitive `actor`分離を使うのではなく、通常の関数と同様に`self`の分離状態はイニシャライザ内で一貫している。途中でnonisolatedに変化する、ということではなく、`nonisolated self`の場合は、終始`nonisolated`であるということ。

#### Sendability

`actor`の外側から`actor`のイニシャライザに値を渡す時、この値は`Sendable`でなければならない。
そのため、新しいインスタンスを初期化している中で、Sendabilityという観点での`actor`の「境界」は、イニシャライザのうちの一つへの最初の呼び出しから始まる。
このルールによって、プログラマは新しい`actor`インスタンスを生成する際に、正しく`Sendable`の値を扱うように強制される。基本的に、プログラマが`actor`のnon-Sendableな格納プロパティを初期化する場合は二つの選択肢しかない。

```swift
class NotSendableType { /* ... */ }
`struct` Piece: Sendable { /* ... */ }

`actor` Greg {
    var ns: NotSendableType

    // 選択肢1: 引数がSendableなためどこからでも呼び出せるイニシャライザ
    init(fromPieces ps: (Piece, Piece)) {
        self.ns = NotSendableType(ps)
    }

    // 選択肢2: 引数がnon-Sendableなため、他のイニシャライザへ委譲することしかできないイニシャライザ
    init(with ns: NotSendableType) {
        self.init(fromPieces: ns.getPieces())
    }
}
```

上記で示したようにnon-Sendableな格納プロパティを持つ`actor`を構築することが**できる**。しかし、そのデータを`actor`のインスタンスに格納するためには、`Sendable`な部分から新しいインスタンスを生成するべき。`actor`のイニシャライザ内に入ると、他のイニシャライザに委譲する時や他のメソッドを呼び出す時など、non-Sendableな値は自由に渡すことができる。

```swift
class NotSendableType { /* ... */ }
`struct` Piece: Sendable { /* ... */ }

`actor` Gene {
    var ns: NotSendableType

    init(_ ns: NotSendableType) {
        self.ns = ns
    }

    init(with ns: NotSendableType) async {
        self.init(ns) // ✅ 他のイニシャライザへ委譲する際にnon-Sendableを渡すのはOK
        someMethod(ns)  // ✅ イニシャライザからメソッドの呼び出し時もOK
    }

    init(fromPieces ps: (Piece, Piece)) async {
        let ns = NotSendableType(ps)
        await self.init(with: ns) // ✅ 他のイニシャライザへ委譲する際にnon-Sendableを渡すのはOK
    }

    func someMethod(_: NotSendableType) { /* ... */ }
}

func someFunc() async {
    let ns = NotSendableType()
    _ = Gene(ns) // ❌ non-Sendableな値は`actor`の境界をまたがって渡せない
    _ = await Gene(with: ns) // ❌ non-Sendableな値は`actor`をまたがって渡せない
    _ = await Gene(fromPieces: ns.getPieces()) // ✅ (Piece, Piece) はSendableなのでOK

    _ = await SomeGAIT(isolated: ns) // ❌ non-Sendableな値は`actor`の境界をまたがって渡せない
    _ = await SomeGAIT(secondNonIso: ns)  // ❌ non-Sendableな値は`actor`の境界をまたがって渡せない
}
```

GAITの場合、nonisolatedのイニシャライザは同じルールが適用される。つまり、そのようなイニシャライザに外部から入る際は、渡される値は`Sendable`でなければならない。globalではない`actor`との違いは:

1. 最初のイニシャライザの呼び出し先が既に同じglobal-`actor`に分離されている場合、`Sendable`に縛る必要はない(いつも通り)
2. `nonisolated`のイニシャライザからglobal-`actor`に分離されたイニシャライザに委譲された場合は、`Sendable`でなければならない

2番目の違いは`nonisolated`かつ`async`のイニシャライザからGAITに分離されたイニシャライザに委譲された場合にのみ適用される。

```swift
@MainActor
class SomeGAIT {
    var ns: NotSendableType

    init(isolated ns: NotSendableType) {
        self.ns = ns
    }

    nonisolated init(firstNonIso ns: NotSendableType) async {
        await self.init(isolated: ns) // ❌ non-Sendableな値は`actor`の境界をまたがって渡せない
    }

    nonisolated init(secondNonIso ns: NotSendableType) async {
        await self.init(firstNonIso: ns)  // ✅
    }
}
```

上記の例は、`nonisolated`属性を削除することでイニシャライザの分離コンテキストが一致する(同じGAITに分離されている)ため解決する。

#### デイニシャライザ

Swift5.5では、`actor`とGAITは`deinit`の中で二つの異なる種類のデータ競合を起こす可能性がある。一つは他の`Task`と`self`の参照を共有していること(`Task`のイニシャライザ内で`self`のメソッドを呼び出すなど)に関してで、もう一つはエグゼキュータを共有していることに関連している。

最初の問題の解決法は、`nonisolated self`に対するFlow-sensitive分離を`deinit`にも適用する。`deinit`は、インスタンスの立ち上げの代わりに片付けや破壊をする目的の`non-async`非委譲イニシャライザと事実上同じでであるため、`nonisolated self`を持つ。特に`deinit`は`self`へのユニークな参照から始まるため、`nonisolated self`へ衰退するルールに完全に適している。これは`actor`とGAITの両方に適用される。

二つ目の問題の解決方法は、`deinit`は`Sendable`な`self`の格納プロパティにしかアクセスできないようにする。より具体的には、`self`への参照がユニークで`nonisolated self`に衰退していない場合でも、`actor`やGAITの`Sendable`な格納プロパティにしかアクセスできなくする。イニシャライザは呼び出し側の分離と`Sendable`引数のチェックをしているので、この制限は`init`には不要。`deinit`はいつどこから呼ばれるかがわからないため、この余分な負担が必要になってくる。事実上、non-Sendableな`actor`に分離された状態は、内部で`actor`からその状態の`deinit`を呼び出すことでしか非初期化できない。

下記にこの新しいルールをコードで示す。

```swift
`actor` A {
    let immutableSendable = SendableType()
    var mutableSendable = SendableType()
    let nonSendable = NotSendableType()

    init() {
        _ = self.immutableSendable  // ✅ ok
        _ = self.mutableSendable    // ✅ ok
        _ = self.nonSendable        // ✅ ok

        f(self) // nonisolated selfに衰退する

        _ = self.immutableSendable  // ✅ ok
        _ = self.mutableSendable    // ❌ immutableでなければならない
        _ = self.nonSendable        // ❌ Sendableでなければならない
    }


    deinit {
        _ = self.immutableSendable  // ✅ ok
        _ = self.mutableSendable    // ✅ ok
        _ = self.nonSendable        // ❌ Sendableでなければならない

        f(self) // nonisolated selfに衰退する

        _ = self.immutableSendable  // ✅ ok
        _ = self.mutableSendable    // ❌ immutableでなければならない
        _ = self.nonSendable        // ❌ Sendableでなければならない
    }
}
```

`init`と`deinit`の違いは、`deinit`は`Sendable`プロパティにしかアクセスできないが、`init`は`nonisolated self`に衰退する前はnon-Sendableプロパティへアクセスできる。

#### global-`actor`分離と`actor`インスタンスメンバ

格納プロパティの型がglobal-`actor`に分離されている場合の主な問題は、そのデフォルト値を設定する式もそのglobal-`actor`に分離されているという点。プロパティはそれぞれのglobal-`actor`に分離されているため、複数の`actor`の分離が発生し、不可能な`actor`分離の要件が生まれてしまう。`non-async`非委譲イニシャライザに必要な分離状態は、デフォルト値を持つ格納プロパティに適用されるすべての分離を統一しなければならない。なぜならば`non-async`イニシャライザはホップできず、関数は二つのglobal `actor`に分離させることができないため。Swift5.5では、この不可能な要件が受け入れられている(正しくない)。

これを解決するために、名前付(nominal)型メンバの格納プロパティのデフォルト値を設定する式の分離状態を削除する。その代わりにそれらは型システムから`nonisolated`とみなされる。それらのプロパティを初期化するため分離が必要な場合は、常に`init`を定義し、適切な分離状態を提供することができる。

グローバルやstaticの格納プロパティは、デフォルト値を設定する式の分離はそのプロパティ自身の分離と引き続き合致する。このルールは下記のような例で必要になる。

```swift
@MainActor
var x = 20

@MainActor
var y = x + 2
```

※ Xcode 13.3 Beta2から、global-`actor`に分離された格納プロパティにデフォルト値を設定すると、Swift5ではワーニング、Swift6ではエラーになる。

```swift
class Party {
    @MainActor var a = 1
    @MainActor var members: [PartyMember] = partyGenerator()
    //                                      ^~~~~~~~~~~~~~~~
    // warning: expression requiring global `actor` 'MainActor' cannot
    //          appear in default-value expression of property 'members'
}
```

[Xcode 13.3 Release Notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-13_3-release-notes#Swift)

##### 余分な分離の削除

global-`actor`に分離されている格納プロパティは、型のインスタンスの格納プロパティが占有しているストレージへの安全な同時並行アクセスを提供している。例えば、問題点のところで例で出てきた`pid`など`actor`に分離された格納プロパティの場合、`p.pid.reset()`は`p`から`pid`を読み取る時のメモリアクセスのみ守られ、その後ろの`reset()`は守られない。そのため値型(`struct`や`enum`など)では、global-`actor`に分離された格納プロパティは意味をなさない。値型の中の格納プロパティのストレージの変更は、タスク間でシェアされることがないため、デフォルトで同時並行処理間で安全に扱える。例えば、`Sendable`クロージャ内で可変変数をキャプチャすることはできない:

```swift
@MainActor
struct StatTracker {
    var count = 0

    mutating func update() {
        count += 1
    }
}

var st = StatTracker()
Task { await st.update() } // ❌ error: mutation of captured var 'st' in concurrently-executing code
```

結局、そのstruct内部の格納プロパティがglobal-`actor`に守られているかどうかに関わらず、structのメモリを同時並行に変更する手段はない。インスタンスが共有できるかどうかは`var`に紐づいているかどうか次第で、許されている共有の種類もコピーによるものだけである。そのstruct内部の参照型の変更は、その参照型自身に適用されている`actor`に分離が必須になる。言い換えると、`class`型を含んだ格納プロパティにglobal-`actor`の分離を適用しても、その`class`のメンバへ同時並行にアクセスすることができてしまう。
そこで、こういったプロパティにアクセスする際に、分離しなければならないという要件を削除する。つまり、これらへのアクセスに`await`は必要なくなる。

[global `actor`のプロポーザル](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-`actor`s.md#detailed-design)では、`actor`はglobal-`actor`の格納プロパティを保持できないと明記されているが、Swift5.5ではこれが強制されていない。これは強制されるべき(例えば、`actor`ストレージは`actor`インスタンスに一貫して分離されるべき)。こうすると、スレッド間のfalse sharing(※)の可能性を減らすことができる。具体的には、常に`actor`インスタンスで使用しているメモリへの書き込みアクセスは一つのスレッドのみになる。

※ 共有していないデータをプロセッサのキャッシュ上の同一ラインで共有してしまう状況
https://en.wikipedia.org/wiki/False_sharing

#### Swift5.5からのsource breaking

今回の変更によってコードの変更が必要な点など挙げる。

変更不要、もしくは自動修正可能な点

- (警告の出されずに)Swift5.5で定義した`init`はより厳密なルール下にあるので変更の必要はない
- `actor`イニシャライザの`convenience`修飾子は無視されるかfix-itが出力される
- (値型の)通常の格納プロパティの余分なglobal-`actor`分離は無視されるかfix-itが出力される

source breakが起きる点

- `actor`やGAITの`deinit`で使える機能が制限される [参考PR](https://github.com/apple/swift/pull/41305)
- GAITの分離されていない一連の`init`では、データ競合を防ぐために使用できる機能が制限される
- `actor`のglobal-`actor`に分離された格納プロパティは禁止される
- `actor`に分離された格納プロパティのメンバのデフォルト値は`nonisolated`とみなされる

GAITの場合はSwiftで定義された`class`にのみ適用され、`MainActor`に分離されたObjective-Cからインポートされた`class`はデータ競合が起きないと想定されるため、適用されない。

#### 検討したが却下になった考え

##### selfが完全に初期化された後はnonisolationにする

言語に新しい概念を追加することを避けるために、`self`が完全に初期化された後は`nonisolation`にするようにしたくなる。しかし、制御フローは完全に`self`を初期化するフローのスコープから、完全に`self`を初期化する「かもしれない」他のスコープにまでまたがるため、このルールだけだとルールだけだと、イニシャライザがデータ競合を起こすかどうがを決めるのに十分ではない。

このルールが破られる二つの例を示す。

```swift
`actor` CounterExampleActor {
    var x: Int

    func mutate() { self.x += 1 }

    nonisolated func f() {
        Task { await self.mutate() }
    }

    init(ex1 cond: Bool) {
        if cond {
            self.x = 0
            f()
        }
        self.x = 1 // condがtrueの場合、データ競合の可能性あり
    }

    init(ex2 max: Int) {
        var i = 0
        repeat {
            self.x = i // 最初のイテレーション後、データ競合の可能性あり
            f()
            i += 1
        } while i < max
    }
}
```

Swiftでは`self`が完全に初期化された後はすぐに自由に利用できる。そのため、`nonisolation`を`self`のそれぞれの使用に紐づけると、データ競合(`f`は`actor`を変更するタスクの中で`self`をescapeすることができ、イニシャライザは格納プロパティが同期されていない状態でアクセスする`f`から戻ってきた後も継続する)を許しても、上記の二つのイニシャライザは受け入れるべきである。

今回のプロポーザルのFlow-sensitive分離では、上記の二つの例はエラーになる。`nonisolation`になる起源は`f()`の呼び出しで明確になっているため、プログラマはコードを修正できる。

ここで`f()`の呼び出しを削除した場合はどうなるだろうか？プロポーザルの分離ルールだと、安全なので受け入れられる。`nonisolation`になる要素がない。もし`self`が完全に初期化された後、すぐに`nonisolation`になってそれがイニシャライザの終わりまで継続される場合、`f`が呼ばれなくても、上記のイニシャライザは不必要に拒否されるだろう。

##### nonisolated selfのイニシャライザ内のプロパティへのアクセスにawaitをつける

`nonisolated self`を持つイニシャライザでは、selfのnonisolatedな使用後、格納プロパティへのアクセスは禁止される。`non-async`イニシャライザでは、適切な`actor`のコンテキストへホップすることができないため、これを回避する方法はない。しかし、`nonisolated self`を持つ`async`イニシャライザの場合はホップすることができる。

```swift
`actor` AwkwardActor {
    var x: SomeClass
    nonisolated func f() { /* ... */ }

    nonisolated init() async {
        self.x = SomeClass()
        let a = self.x
        f()
        let b = await self.x // SomeClassはSendableである必要がある
        print(a + b)
    }
}
```

これは実現可能であるものの、このプロポーザルではこれに反対している。

Flow-sensitive分離での`async`なプロパティへのアクセスをサポートすることによるメリットよりも、新しい混乱を生み出す可能性の方が高い。詳細を知らないプログラマがこのコードを読んだ際、`await`は不要に見え、全ての関数に適用されている分離のルールへの理解を困難にするかもしれない。しかし、この特定の種類の`nonisolated self`を持つ`async`イニシャライザが、読み手のプログラマへ、関数の途中で分離が衰退することを示す唯一のポイントでもある。

有効なSwiftコードで、この関数の途中で分離が衰退することが見えるという点が上記のコードを拒否する理由。今回のプロポーザルでは、`nonisolated self`を持つ`non-async`なイニシャライザの場合、分離からの衰退後の格納プロパティへのアクセスは禁止されている。これらのイニシャライザの正しい公式は、分離の衰退を見えなくすることで、読み手に何も気がつかなくさせること。コードを変更するときのみ、この分離の衰退の概念に気づくことが妥当。ただし、この概念も今回のプロポーザル説明するために使用される単なるツール。プログラマは、イニシャライザで`self`をエスケープした後は、格納プロパティへのアクセスできないことだけ覚えておけばよい。

##### `actor`のasyncデイニシャライザ

インスタンスの衰退前に`actor`やGAITのエグゼキュータで`deinit`から同期することができないことへのworkaroundとして、`deinit`の本文を`Task`で囲む方法がある。実際これで`non-async`な`deinit`でも、あたかも本文が`async`であるかのように振る舞える。呼び出し側がasyncコンテキストにあるとは限らないため、`deinit`をasyncにすることはできない。

ここでの主なリスクは、Swift現在の実装では、`deinit`から`self`の参照をエスケープして`deinit`完了後も参照を保持した場合(もし`deinit`がasyncであるならばこれは可能でなければならない。)の挙動が不定であるということ。もう一つの選択肢は`deinit`内でブロッキングをすることだが、Swift Concurrencyはブロッキングを避けるように設計されている。

##### 委譲イニシャライザのみ`Sendable`引数を必須にする

イニシャライザを委譲することは、概念的に初期化中に`actor`に渡されたnon-`Sendable`な値を持つ新しいインスタンスを構築するのに良い場所である。これはnonisolatedな`self`の委譲イニシャライザからのみ任意のコンテキストからnon-`Sendable`の値を受け入れることができる、というこにしなければ機能しない。また、このイニシャライザの委譲状態を型のインターフェイスとして公開しなければならなくなる。例えば、`convenience`のようなアノテーションが必須になる。`convenience`の必要性を削除する方が、特定のケースの委譲イニシャライザにのみnon-`Sendable`な値を許すよりも選ばれた。これにはいくつかの理由がある:

- `Sendable` とイニシャライザのルールが3つの要素(イニシャライザの委譲状態、selfの分離、呼び出し側のコンテキスト)に依存することになり複雑になる。通常の`Sendable`のルールは2つにのみ依存している
- `convenience`をたった一つのユースケースのために必須とすることはあまり価値がない
- ファクトリパターンを使用するstatic関数は、イニシャライザの必要性を任意のコードから呼び出し可能なnon-`Sendable`引数に置き換えることができる

## 参考リンク

### Forums

- [SE-0327: On Actors and Initialization](https://forums.swift.org/t/se-0327-on-`actor`s-and-initialization/53053)

- [SE-0327 (Second Review): On Actors and Initialization](https://forums.swift.org/t/se-0327-second-review-on-`actor`s-and-initialization/54093)

### プロポーザルドキュメント

- [On Actors and Initialization](https://github.com/apple/swift-evolution/blob/main/proposals/0327-`actor`-initializers.md)

### 関連PR

- [[SE-327] Implement Flow-sensitive `actor` isolation for `actor` inits](https://github.com/apple/swift/pull/40028)
- [[SE-327] Remove redundant global-`actor` isolation](https://github.com/apple/swift/pull/40868)
- [[SE-327] Reimplement: make initializing expressions for member stored properties nonisolated](https://github.com/apple/swift/pull/40908)
- [[SE-327] Remove need for convenience for delegating initializers of an `actor`](https://github.com/apple/swift/pull/41083)

### その他

- [Actor reentrancy](https://github.com/apple/swift-evolution/blob/main/proposals/0306-`actor`s.md#`actor`-reentrancy)
- [false sharing](https://en.wikipedia.org/wiki/False_sharing)
- [データフロー解析](https://ja.wikipedia.org/wiki/データフロー解析)

# Swift actorの初期化/非初期化のルールとなぜを理解する

- [Swift actorの初期化/非初期化のルールとなぜを理解する](#swift-actorの初期化非初期化のルールとなぜを理解する)
  - [用語](#用語)
  - [概要](#概要)
  - [内容](#内容)
    - [actorとは？](#actorとは)
    - [global-actorとは？](#global-actorとは)
    - [actorのイニシャライザの前提知識](#actorのイニシャライザの前提知識)
    - [現在の問題点](#現在の問題点)
      - [`non-async`のイニシャライザの`self`のアクセスの制限が厳しすぎる](#non-asyncのイニシャライザのselfのアクセスの制限が厳しすぎる)
      - [deinitでデータ競合を起こすリスクがある](#deinitでデータ競合を起こすリスクがある)
      - [格納プロパティのactor分離](#格納プロパティのactor分離)
      - [イニシャライザの委譲](#イニシャライザの委譲)
    - [解決策](#解決策)
      - [非委譲イニシャライザの場合](#非委譲イニシャライザの場合)
        - [Flow-sensitive actor分離](#flow-sensitive-actor分離)
          - [`isolated self`のイニシャライザ](#isolated-selfのイニシャライザ)
          - [nonisolated selfのイニシャライザ](#nonisolated-selfのイニシャライザ)
        - [GAITs(global-actorに分離された型)](#gaitsglobal-actorに分離された型)
      - [委譲イニシャライザ](#委譲イニシャライザ)
        - [シンタックス](#シンタックス)
        - [Isolation](#isolation)
      - [Sendability](#sendability)
        - [委譲とSendable](#委譲とsendable)
      - [デイニシャライザ](#デイニシャライザ)
      - [global-actor分離とactorインスタンスメンバ](#global-actor分離とactorインスタンスメンバ)
      - [冗長な分離の削除](#冗長な分離の削除)
      - [Swift5.5からのsource breaking](#swift55からのsource-breaking)
      - [検討したが却下になった考え](#検討したが却下になった考え)
        - [selfが完全に初期化された後はnonisolationにする](#selfが完全に初期化された後はnonisolationにする)
        - [nonisolated selfのイニシャライザ内のプロパティへのアクセスにawaitをつける](#nonisolated-selfのイニシャライザ内のプロパティへのアクセスにawaitをつける)
        - [actorのasyncデイニシャライザ](#actorのasyncデイニシャライザ)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)
    - [関連PR](#関連pr)
    - [その他](#その他)

## 用語

- actor-isolated(isolation): actorによってデータ競合から守られている(他から分離されている)状態
- executor: actorの処理を実行するオブジェクト(actorだけではなくSwift ConcurrencyのTaskは全て、あるexecutor上で実行される)
- hop: actorのコンテキストに移動する(飛ぶ)こと
- GAIT: Global-actor isolated types(global-actorで分離された型)

## 概要

Swift5.5で導入されたactorに分離された状態の生成や衰退の過程には、いくつかの微妙なな側面を見逃していた。この提案では、actorの定義を強化して、actorに分離された状態がいつ開始/終了するかや`init`/`deinit`でできることを明確にする。

## 内容

### actorとは？

同時並行処理中でも内部の可変状態でデータ競合(※)を起こさずに安全に扱えるようにした参照型。

※ データ競合とは、複数スレッドから同じメモリの可変状態に書き込み/読み込み処理が同時に起きること。

例えば、下記の例は`increament()`メソッド内の`value`の更新タイミング次第で出力される値が変わり、1と1や2と2など予期せぬ値になることがある。

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

actorの場合、データ競合が起こらずに1と2または2と1になる。

```swift
actor Counter {
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

よく間違えやすいものとして競合状態がある。これは、各スレッドの処理の実行順序が不定で、同じ入力しても出力結果が変わってしまう状態を指す。actorはデータ競合を防ぐが、設計次第で競合状態を起こす可能性はある。

### global-actorとは？

グローバルな状態を一つのactorとして分離するactor。複数の異なる型やメソッドなどに適用でき、安全にglobal-actorのexecutor内で安全に同時並行に扱うことがある。

### actorのイニシャライザの前提知識

actorのイニシャライザはactorに分離されていない。これは、`self`が初期化するまで他からアクセスできないので格納プロパティへアクセスは安全に行うことができる。そのため、`non-async`なイニシャライザは、`await`なしで呼び出せる。

### 現在の問題点

#### `non-async`のイニシャライザの`self`のアクセスの制限が厳しすぎる

actorはデータ競合を防ぐために、内部の状態の更新はactor内で行われる必要がある。そのために、`await`することでactorのコンテキストに移動(以降、hopと呼ぶ)しなければならない場合がある。

しかし、`non-async`のイニシャライザは`await`してhopすることができないため、データ競合を防ぐことができない。

```swift
actor Clicker {
    var count: Int
    func click() { self.count += 1 }

    init(bad: Void) {
        self.count = 0
        // actorコンテキストにhopすることができない

        Task { await self.click() }

        self.click() // 💥 ここで変更できてしまうとTask内とデータ競合する

        print(self.count) // 💥 1 または 2が出力される(不定)
    }
}
```

これを防ぐためにSwift5.5では`self`をクロージャでキャプチャしたり、クロージャ内でインスタンスメソッドを呼び出すとワーニングを出していた(= Swift6ではエラーになる予定だった)。しかし、`self`使うこと自体が全て危険というわけではない。

例えば、下記の`self.announce()`は`nonisolated`でactorに分離された状態であるかどうかとは無関係にも関わらずエラーになる。

```swift
actor Clicker {
    var count: Int
    func click() { self.count += 1 }
    nonisolated func announce() { print("performing a click!") }

    init(ok: Void) {
        self.count = 0
        Task { await self.click() }
        self.announce() // Swift 5.5では❌だが、データ競合は起きない。 ⚠️This use of actor 'self' can only appear in an async initializer
    }
}
```

#### deinitでデータ競合を起こすリスクがある

`deinit`でもデータ競合やselfの(余計な)延命を起こす可能性がある。

下記の例で考えてみる。

```swift
actor Clicker {
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

これがGAITの場合はもっと複雑になる。GAITは他のインスタンスとexecutorを共有しているactor(※)に近い。actorやGAITのexecutorがそのインスタンスだけで排他的に所有されているわけではない場合に、non-Sendableのプロパティにアクセスすると安全ではない。

※ 他のインスタンスとexecutorを共有しているactorは、まだ導入されていないがCustom Executorについての言及。

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

二つの`Maria`はexecutorを共有しているので、同じnon-Sendableな可変状態を参照することができ、`deinit`は`executor`から`nonisolated`なことから、これにアクセスすると同時並行なアクセスが発生してしまう。

また、`deinit`が呼ばれた後に`self`の参照カウンタが増えた場合の挙動はundefinedなため、ランダムにクラッシュが発生するリスクがある。(これは通常の`class`でも同じことが発生する)

#### 格納プロパティのactor分離

現在は、`class`、`struct`、`enum`ではglobal-actorに分離されたプロパティを保持できる。プログラマがデフォルト値を指定している場合、その値はそれぞれの型の非委譲イニシャライザの中で計算される。

問題なのは、それらがそれぞれのglobal-actorのexecutor上で実行されるため、不可能な制約が生成できてしまう。

```swift

@MainActor func getStatus() -> Int { /* ... */ }
@PIDActor func genPID() -> ProcessID { /* ... */ }

class Process {
    @MainActor var status: Int = getStatus()
    @PIDActor var pid: ProcessID = genPID()

    init() {} // 問題:このinit内は何で分離されている？
}
```

この例はSwift5.5でエラーにならないが、`status`と`pid`が別のglobal-actorで実行されているため、`non-async`なイニシャライザ内で一つのactorに分離できない。また、`non-async`なので、適切なactorにhopすることもできず、結局、`getStatus`も`getPID`も適切なexecutorで実行できない状態になっている。

#### イニシャライザの委譲

Swiftでは全てのnominal型(`class`や`struct`などの名前のある型)はイニシャライザの委譲(他のイニシャライザに残りの初期化を任せることができる機能)をサポートしている。classは継承があるため他よりも複雑で、委譲するイニシャライザには`convenience`修飾子をつけなければならない。値型は継承がないためよりシンプル。任意の`init`で委譲可能だが、その場合は全実行パスで委譲または`self`への割り当てが必要になる。

```swift
struct S {
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

actorは継承をサポートしておらず、`convenience`修飾子をつけるかどうかも明記していなかったが、Swift5.5では委譲する際に
`convenience`修飾子が必須になっている。

### 解決策

#### 非委譲イニシャライザの場合

actorやGAITの非委譲イニシャライザはその型の全ての格納プロパティを初期化しなければならない。

##### Flow-sensitive actor分離

※ ここからはactorの非委譲イニシャライザに関する内容で、GAITは除外。

Swift5.5では、「escapingでの使用制限」(escaping-use restriction)が行われていた。escapingでの使用とは、

- クロージャでの`self`のキャプチャ
- `self`内のメソッドや計算プロパティを呼ぶ
- 任意の種類の引数(値としてや`autoclosure`や`inout`)に`self`を渡す

があり、こういった使用は禁止されていた。この制限は過剰で、このescaping使用制限を削除する。代わりに、よりシンプルなルールを提案。

まずイニシャライザをisolationの度合いに応じて二つに分類する。

1. `nonisolated self`
   - `non-async`
   - global-actor isolated
   - `nonisolated`

2. `async`イニシャライザ(`isolated self`参照を保持している)

###### `isolated self`のイニシャライザ

`async`イニシャライザは`self`が完全に初期化後、`self`がisolatedな状態あると考えられ、actorのコンテキストにhopする。このactorのコンテキストにhopする位置を選択することで、`async`イニシャライザ全体で`self`がisolatedな状態を保つことができる。こうすと`self`をescapeする前に、actorのコンテキストにhopは完了している。

hopはsuspension pointであると認識することが大事である。`self`の初期化が完了するタイミングは複数存在するので、suspensionが起きる可能性があるポイントが複数ある。

```swift
actor Bob {
    var x: Int
    var y: Int = 2
    func f() {}
    init(_ cond: Bool) async {
        if cond {
            self.x = 1 // 初期化完了
        }
        self.x = 2 // 初期化完了

        f() // ⭕️ executor上にいる
    }
}
```

`Bob.init`のsuspension pointに明示的に`await`をマークしようとする場合の問題点は、プログラマがそれを追跡するのが困難で、シンプルなリファクタリングでもsuspension pointを同じ位置に保つことができない。例えば、デフォルト値を追加したり消したり、新しい格納プロパティを追加することで、hopが発生する箇所に大きく影響するかもしれない。

上記の例をちょっと変えてみる。

```swift
actor EvolvedBob {
    var x: Int
    var y: Int
    func f() {}
    init(_ cond: Bool) async {
        if cond {
            self.x = 1
        }
        self.x = 2
        self.y = 2 // 初期化完了

        f() // ⭕️ executor上にいる
    }
}
```

違いは`y`のデフォルト値を消しただけだが、suspension pointは大きく変わっている。もし`await`が必要な場合、このsuspension pointが移動した理由をプログラマに説明するのが難しい。

まとめると、プログラマがこのポイントを明示する代わりに、`self`が一度初期化したら暗黙的にhopするようにする。

- 継続的にsuspension pointを見つけて更新するのはプログラマにストレス
- プロパティへのシンプルな値の格納がsuspensionを起動するのは実装詳細でプログラマに説明するのが難しい
- suspensionを明示するメリットが低い。suspensionが発生するまで`self`への参照はユニークであることはわかっているので、actor reentrancy(※)が発生することがない
- 暗黙的にsuspensionが発生するメリットがデメリットを超えると判断された`async let`の前例が存在する

※ actor reentrancy
actor上でsuspensionが発生した際に、他のactorのタスクをexecutor上で実行できること

関連ドキュメント: https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md#actor-reentrancy

これらの暗黙のhopの効果は、プログラマから見ると、`async`イニシャライザにルールが追加されていないように見えること。他の`async`メソッドと同じように、イニシャライザは全体でactorに分離された状態になっているように見なせる。ほとんどのケースでイニシャライザにhopが挿入されるflow-sensitiveなポイントは実装詳細として無視することができる。ただし例外としては下記のようなものもある。

```swift
actor OddActor {
    var x: Int
    init() async {
        let name = Thread.current.name
        self.x = 0 // 初期化完了
        assert(name == Thread.current.name) // 失敗する可能性あり
    }
}
```

`OddActor.init`の呼び出し側は、呼び出し時に`await`が必要なため、suspensionが起きないとは想定しないはず。

###### nonisolated selfのイニシャライザ

このカテゴリには`non-async`、または`self`とは異なったコンテキストで分離して実行されるイニシャライザを含んでいる。メソッドと異なり、同期する必要のあるactorインスタンスは存在しないため、actorの`non-async`イニシャライザに`await`は必要ない。また、このカテゴリのイニシャライザは同期なしで格納プロパティでアクセスすることができる。

格納プロパティにアクセスするにはactorのインスタンスを立ち上げる必要がある。これは`self`の参照アクセスが排他的であるという弱い分離の形式が想定されている。`self`がイニシャライザをescapeする場合に、このユニークさは時間のかかる分析なしでは保証できない。

そこで、Flow-Sensitive actor分離(※)を利用する。この`self`の分離は、直接格納プロパティへアクセスする以外の`self`の使用は、全て`nonisolated`へ衰退する。これは特定の制御フローで一度起きたらイニシャライザの最後まで保たれる。

※ データフロー解析を使って制御フローの位置に応じてactorの分離状態を変えること。
https://ja.wikipedia.org/wiki/データフロー解析

下記に`nonisolated`へ衰退する例を示す。

1. 引数に`self`を渡す(以下を含む)
   - `self`のメソッドを呼ぶ
   - プロパティラッパを使ったものを含めた`self`の計算プロパティへアクセスする
   - 監視プロパティ(つまり、`didSet`や`willSet`持つプロパティ)をトリガーする
2. クロージャやautoclosureで`self`をキャプチャする
3. `self`をメモリに保存する

具体的なコードで見てみる。

```swift
class NotSendableString { /* ... */ }
final class Address: Sendable { /* ... */ }
func greetCharlie(_ charlie: Charlie) {}

actor Charlie {
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
            nonisolatedMethod() // ✅ nonisolated selfの使用
            greetCharlie(self)  // ✅ nonisolated selfの使用
            self.me = self      // ✅ nonisolated selfの使用
        } else if score < 50 {
            score = 50
        }

        assert(score >= 50) // ❌ nonisolated selfの使用後に、可変な分離された格納プロパティへアクセスできない

        _ = self.fixedNonSendable // ❌ nonisolated selfの使用後に、non-Sendableなプロパティへアクセスできない
        _ = self.fixedSendable

        Task { await self.incrementScore() } // ✅ nonisolated selfの使用
    }
}
```

この例のポイントは、`if else`で複数の制御フローがイニシャライザに導入されていること。
最初の条件分岐の中で複数回`self`の`nonisolated`な使用をしている。一方の分岐(`else if`)ではまだ`score`への読み書きはできる。一方で`if else`の後の`assert`の時点では、この地点に到達できるブロックの一つで、`self`の`nonisolated`な使用が行われているため、`self`は`nonisolated`とみなされる。

結論として、`self`が`nonisolated self`になった後は、アクセス可能な唯一の格納プロパティは`let`で宣言された`Sendable`なもののみとなる。それ以外の格納プロパティへ不正にした場合は、分離の変化が発生した不正アクセスよりも前に`self`を使用した場所の一つを指摘する。この早期というのは制御フローの観点で、プログラム上のどこに現れるかではない(不正アクセスより下に記述されていたとしても実行順序が先の場合はそこのポイントが指摘される)。

例えば、`defer`を使った`Charlie.init`の別の例を見て考えてみる。

```swift
init(hasADefer: Void) {
    self.score = 0
    defer {
        print(self.score) // ❌ nonisolated selfの使用後に、分離された可変の格納プロパティへアクセスできない
    }
    Task { await self.incrementScore() } // note: nonisolated selfの使用
}
```

`self.score`の出力をイニシャライザの最後に遅らせている。しかし、`defer`の実行前に`self`がクロージャ内でキャプチャされているため、`self.score`の値の読み込みはいつもデータ競合から安全であるとは限らない。そのため、これはエラーになる。

不正なプロパティアクセスが目に見える形で`nonisolated`に衰退する例として`for`ループがある。

```swift
init(hasALoop: Void) {
    self.score = 0
    for i in 0..<10 {
        self.score += i     // ❌ nonisolated selfの使用後に、分離された可変の格納プロパティへアクセスできない
        greetCharlie(self)  // note: nonisolated selfの使用
    }
}
```

イテレーションの最初のループしか完全ではないので、この`for`ループでは、`self.score`の変更にもエラーにしなければならない。それ以降のイテレーションではこのループの外側で同時並行にアクセスされる可能性があるため、安全ではない。

**他の例**

`non-async`イニシャライザ以外にも、global-actor isolatedや`nonisolated`なイニシャライザも`nonisolated self`になる可能性がある。

下記の例を考えてみる。

```swift
func printStatus(_ s: Status) { /* ... */}

actor Status {
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

        let old = await self.exchange(with: false) // note: nonisolated selfの使用
        assert(old == val)

        _ = self.valid //❌ nonisolated selfの使用後に、可変なisolatedの格納プロパティへアクセスできない

        let isValid = await self.isValid() // ✅ OK

        assert(isValid == false)
    }
}
```

`await`できる場合、`nonisolated self`なイニシャライザから分離されたメソッドの呼び出しが許可されていることに注目。この呼び出しはnonisolated selfの使用とみなされる。これが格納プロパティへのアクセス以外で最初の`self`へのアクセスになる。その後は、`non-async`イニシャライザと同じように`init`内での格納プロパティへのアクセスはできなくなる。このイニシャライザは`async`なので技術的には`Sendable`な`self.valid`の値を`await`できるが、この状況でのアクセスを禁止することを選んだ。(詳細は[nonisolated selfのイニシャライザ内のプロパティへのアクセスにawaitをつける](#nonisolated-selfのイニシャライザ内のプロパティへのアクセスにawaitをつける)を参照)

##### GAITs(global-actorに分離された型)

GAITの`nonisolated`イニシャライザは、`non-async`イニシャライザと同じ状況で、executorの保護なしにインスタンスを立ち上げなければならない。つまり、データ競合を生み出す可能性がある。

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

これを防ぐためにGAITの`nonisolated`イニシャライザにもflow-sensitive actor分離を適用する。

isolatedなイニシャライザの場合、GAITはイニシャライザの呼び出し前にactorに分離できる。なぜなら、executorはstaticインスタンスなので、GAITインタンスの未初期化のメモリを割り当てるより前に存在しているから。そのため、GAITのisolatedなイニシャライザは、初期化開始前に正しいexecutorで実行されるためにも`await`が必要になる。そしてこのexecutorはイニシャライザがリターンするまで確保されているため、GAITのisolatedなイニシャライザではisolatedな格納プロパティ間でのデータ競合の危険がない。

```swift
@MainActor
class ProtectedByExecutor<T: Equatable> {
    var item: T

    func mutateItem() { /* ... */ }

    init(with t: T) {
        self.item = t
        Task { self.mutateItem() }  // ✅ Taskを生成する際に正しいexecutor上にいる
        assert(self.item == t) // ✅ executorを確保しているため、常にtrue
    }
}
```

`nonisolated`な格納プロパティを持つGAITは、データ競合を防ぐために、既存の`Sendable`の制限に頼っている。

#### 委譲イニシャライザ

ここではactor型とGAITsの委譲イニシャライザに関するシンタックスとルールを定義する。

##### シンタックス

actorは参照型だが、委譲イニシャライザに関しては既存の値型と同じ基本的なルールに従う。

1. イニシャライザの本文で`self.init`の呼び出しがあった場合、それは委譲イニシャライザとして扱われる。`convenience`キーワードはいらない
2. 委譲イニシャライザは、`self`が使用される前に全ての実行パスで`self.init`を呼び出さなければならない

actorは継承がないため、`class`の複雑さを減らすことができる。GAITの場合は委譲イニシャライザを定義する際は、`class`と同じシンタックスを使う。

##### Isolation

非委譲イニシャライザと多くが同じで、actorの委譲イニシャライザも`isolated self`または`nonisolated self`の場合がある(non-asyncイニシャライザは`nonisolated self`)。しかし、格納プロパティを初期化する必要がないため、委譲イニシャライザは本文中に存在するものに関するよりシンプルなルールを持つ。つまり、flow-sensitive actor分離ではなく、通常の関数と同様に`self`の分離状態はイニシャライザ内で一貫している(isolationの衰退が起きない)。

#### Sendability

actorの委譲イニシャライザとGAITの全てのイニシャライザは、他の関数と同じSendable引数に関するルールに従う。つまり、関数が分離された状態の場合、actorをまたがった呼び出しの場合、引数は`Sendable`プロトコルへの準拠が必須になる。

actorの非委譲イニシャライザでは、`self`にflow-sensitive分離が適用されているかどうかに関わらず、`Sendable`という観点では分離された状態だとみなされる。なぜならこれらのイニシャライザがインスタンス立ち上げの際にactorの格納プロパティにアクセスできるからである。

この二つのルールによって、プログラマは新しいactorインスタンスを生成する際に、安全に`Sendable`の値を扱うことができる。基本的に、プログラマはactorのnon-Sendableな格納プロパティを初期化する場合は二つの選択肢しかない。

```swift
class NotSendableType { /* ... */ }
struct Piece: Sendable { /* ... */ }

actor Greg {
    var ns: NonSendableType

    // 選択肢1: 非委譲イニシャライザはSendableな値しか受け取れず
    // それを使って新しいnon-Sendableな値を構築する
    init(fromPieces ps: (Piece, Piece)) {
        self.ns = NonSendableType(ps)
    }

    // 選択肢2: nonisolated selfな委譲イニシャライザはnon-Sendableを受け取ることができ
    // Sendableな部分を非委譲イニシャライザに渡すことができる
    init(with ns: NonSendableType) {
        self.init(fromPieces: ns.getPieces())
    }
}
```

上記で示したようにnon-Sendableな格納プロパティを持つactorを構築することができる。しかし、そのデータの`Sendable`な部分から新しいインスタンスを生成することができなければならない。上記の二つの選択肢は、非委譲イニシャライザからその部分を直接受け取るか、non-Sendableな値を受け取った委譲イニシャライザに頼るという方法を提供する。

こうすることで、非委譲イニシャライザ内でnon-Sendableの完全に新しいインスタンスを構築することをプログラマに強制することができる。

##### 委譲とSendable

全ての委譲イニシャライザが任意の呼び出しでnon-Sendableの値を受け取れる訳ではない。non-Sendableを安全に渡せるかどうかは呼び出し側の分離状態次第になる。

例えば、`async`委譲イニシャライザは`isolated self`である。そのため、委譲後にも格納プロパティにアクセスできる。もしnon-Sendableな値を呼び出し側の分離状態を考慮せずに使えるようにしてしまった場合、non-Sendableの不正な共有が起きる可能性がある。

```swift
class NotSendableType { /* ... */ }
struct Piece: Sendable { /* ... */ }

actor Gene {
    var ns: NonSendableType?

    init(with ns: NonSendableType) async {
        self.init()
        self.ns = ns
    }

    init(fromPieces ps: (Piece, Piece)) async {
        let ns = NonSendableType(ps)
        await self.init(with: ns) // ✅ OK
        assert(self.ns == ns)
    }
}

func someFunc(ns: NonSendableType) async {
    let ns = NonSendableType()
    _ = await Gene(with: ns) // ❌ non-Sendableな値はactorをまたがって渡せない

    _ = await Gene(fromPieces: ns.getPieces())
}
```

`Gene.init(with:)`と`Gene.init(fromPieces:)`はどちらも`isolated self`の委譲イニシャライザである。違いは`init(with:)`はnon-Sendableの引数を受け取っていて、`init(fromPieces:)`はSendableな引数のみを受け取れる。`someFunc`から`init(with:)`を呼び出すことはactorをまたがることになるため安全ではない。一方で`init(fromPieces:)`から`init(with:)`を呼び出すことは、同じ分離コンテキスト内なので安全である。

#### デイニシャライザ

Swift5.5では、actorとGAITは`deinit`の中で二つの異なる種類のデータ競合を起こす可能性がある。一つは他のタスクと`self`の参照を共有していることに関してで、もう一つはexecutorを共有していることに関連している。

最初の問題の解決法は、`nonisolated self`に対するflow-sensitive分離を`deinit`にも適用する。`deinit`は、事実上立ち上げの代わりに片付けや衰退をする目的の`non-async`で非委譲イニシャライザであるため、`nonisolated self`になる。特に`deinit`は`self`へのユニークな参照から始まるため、`nonisolated self`へ衰退するルールに完全に適している。これはactorとGAITの両方に適用される。

二つ目の問題の解決方法は、`deinit`は`Sendable`な`self`の格納プロパティにしかアクセスできないようにする。より具体的には、`self`がユニークな参照で`nonisolated self`に変わっていない場合、actorやGAITの`Sendable`な格納プロパティにしかアクセスできなくする。イニシャライザは呼び出し側で分離状態かどうかと`Sendable`引数のチェックをしていることがわかっているので、この制限は`init`には不要。`deinit`はいつどこから呼ばれるかがわからないため、この余分な負担が必要になってくる。事実上、non-Sendableなactorに分離された状態は、内部でactorからその状態の`deinit`を呼び出すことでしかデイニシャライズできない。

下記のこの新しいルールをコードで示す。

```swift
actor A {
    let immutableSendable = SendableType()
    var mutableSendable = SendableType()
    let nonSendable = NonSendableType()

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

#### global-actor分離とactorインスタンスメンバ

格納プロパティの型がglobal-actorに分離されている場合の主な問題は、そのデフォルト値もそのglobal-actorに分離されているという点。プロパティは個々にglobal-actorに分離されているため、型として一つのactorに分離できないという不可能なactor分離の要件が生まれてしまう。非委譲の`non-async`イニシャライザに必要な分離状態は、デフォルト値を持つ格納プロパティに適用されるすべての分離状態と融合しなければならなくなる。なぜならば`non-async`イニシャライザはhopできず、関数を二つのglobal actorに分離させることができない。Swift5.5では、この不可能な要件が受け入れられている。

これを解決するために、nominal型の格納プロパティのデフォルト値に適用された分離コンテキストを削除する。その代わりにそれらは型システムから`nonisolated`とみなされる。それらのプロパティを初期化するために分離コンテキストが必要な場合は、常に`init`を定義し、適切な分離コンテキストを提供することができる。

グローバルやstaticの格納プロパティは、デフォルト値の分離コンテキストはそのプロパティの分離コンテキストが引き続き適用される。このルールは下記のような例で必要になる。

```swift
@MainActor
var x = 20

@MainActor
var y = x + 2
```

#### 冗長な分離の削除

global-actorに分離されている格納プロパティは、その値を保持している型のインスタンスのストレージへの安全な同時並行アクセスを提供している。例えば`pid`はactorに分離された格納プロパティの場合、`p.pid.reset()`は`p`から`pid`を読み取る時のみメモリは守られていて、その後の`reset()`は安全に呼ばれない。そのためstructやenumといった値型では、global-actorに分離された格納プロパティは意味をなさない。値型の格納プロパティのストレージの変更はCOW(Copy on write)のお陰でデフォルトでは安全に同時並行に行える。そこで、これらのプロパティへのアクセスは分離する要件は削除する。つまり、これらの格納プロパティの読み書きに`await`は必要ない。

[global actorのプロポーザル](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md#detailed-design)では、actorはglobal-actorの格納プロパティを保持できないと明記されているが、Swift5.5ではこれが強制されていない。これは強制されるべき(例えば、actorストレージはactorインスタンスに一貫して分離されるべき)。こうすると、スレッド間のfalse sharing(※)の可能性を減らすことができる。具体的には、常にactorインスタンスで使用しているメモリへの書き込みアクセスは一つのスレッドのみになる。

※ 共有していないデータをプロセッサのキャッシュ上の同一ラインで共有してしまう状況
https://en.wikipedia.org/wiki/False_sharing

#### Swift5.5からのsource breaking

今回の変更によって変更が必要な点など挙げる。

変更不要、もしくは自動修正可能な点

- Swift5.5で定義した`init`はより厳密なルールがあるので変更の必要はない
- actorイニシャライザの`convenience`修飾子は無視されるかfix-itが出力される
- (値型の)通常の格納プロパティの余分なglobal-actor分離は無視されるかfix-itが出力される

source breakが起きる点

- actorやGAITの`deinit`で使える機能が制限される
- GAITの`nonisolated self`な`init`では、データ競合を防ぐために使用できる機能が制限される
- actorのglobal-actorの格納プロパティは禁止される
- actorに分離可能な格納プロパティのメンバのデフォルト値は`nonisolated`とみなされる

GAITの場合はSwiftで定義された`class`にのみ適用され、`MainActor`に分離されたObjective-Cからインポートされた`class`はデータ競合が起きないと想定される。

#### 検討したが却下になった考え

##### selfが完全に初期化された後はnonisolationにする

言語に新しい概念を追加することを避けるために、`self`が完全に初期化された後は`nonisolation`にするようにしたくなる。しかし、制御フローは完全に`self`を初期化するフローのスコープから、完全に`self`を初期化する「かもしれない」他のスコープにまでまたがるため、このルールだけだとルールだけだと、イニシャライザがデータ競合を起こすかどうがを決めるのに十分ではない。

このルールが破られる二つの例を示す。

```swift
actor CounterExampleActor {
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

Swiftでは`self`が完全に初期化された後はすぐに自由に利用できる。そのため、`nonisolation`を`self`のそれぞれの使用に紐づけると、データ競合(`f`はactorを変更するタスクの中で`self`をescapeすることができ、イニシャライザは格納プロパティが同期されていない状態でアクセスする`f`から戻ってきた後も継続する)を許しても、上記の二つのイニシャライザは受け入れるべきである。

今回のプロポーザルのflow-sensitive分離では、上記の二つの例はエラーになる。`nonisolation`になる起源は`f()`の呼び出しで明確になっているため、プログラマはコードを修正できる。

ここで`f()`の呼び出しを削除した場合はどうなるだろうか？プロポーザルの分離ルールだと、安全なので受け入れられる。`nonisolation`になる要素がない。もし`self`が完全に初期化された後、すぐに`nonisolation`になってそれがイニシャライザの終わりまで継続される場合、`f`が呼ばれなくても、上記のイニシャライザは不必要に拒否されるだろう。

##### nonisolated selfのイニシャライザ内のプロパティへのアクセスにawaitをつける

`nonisolated self`のイニシャライザでは、nonisolated selfの使用後、格納プロパティへのアクセスは禁止される。`non-async`イニシャライザでは、適切なactorのコンテキストへhopすることができないため、これを回避する方法はない。しかし、`nonisolated self`の`async`イニシャライザの場合は、hopすることができる。

```swift
actor AwkwardActor {
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

このようなflow-sensitive分離をサポートすることによるメリットよりも、新しい混乱を生み出す可能性の方が高い。詳細を知らないプログラマがこのコードを読んだ際、`await`は不要に見え、関数全体がisolatedされているかどうかがわからなくなる可能性がある。しかし、この特定の種類の`nonisolated self`かつ`async`イニシャライザが、読み手のプログラマへ、関数の途中で分離が衰退することを示す唯一のポイントになる。

有効なSwiftコードで、この関数の途中で分離が衰退することが見えるという点が上記のコードを拒否する理由。今回のプロポーザルでは、`nonisolated self`かつ`non-async`なイニシャライザの場合、分離衰退後の格納プロパティへのアクセスは禁止されている。これらのイニシャライザの正しいルールは、分離の衰退を見えなくすることで、カジュアルな読み手は何も異常に気がつかない。コードを変更するときのみ、この分離の衰退の概念は適切になる。ただし、この概念も今回のプロポーザル説明するために使用される単なるツールです。 プログラマは、イニシャライザで`self`をエスケープした後は、格納プロパティへのアクセスできないことだけ覚えておけばよい。

##### actorのasyncデイニシャライザ

インスタンスの衰退前にactorやGAITのexecutorで`deinit`から同期することができないことへのworkaroundとして、`deinit`の本文を`Task`で囲む方法がある。実際これで`non-async`な`deinit`でも、あたかも本文が`async`であるかのように振る舞える。呼び出し側がasyncコンテキストにあるとは限らないため、`deinit`をasyncにすることはできない。

ここでの主なリスクは、Swift現在の実装では、`deinit`から`self`の参照をエスケープして`deinit`完了後も参照を保持した場合(もし`deinit`がasyncであるならばこれは可能でなければならない。)の挙動が不定であるということ。もう一つの選択肢は`deinit`内でブロッキングをすることだが、Swift Concurrencyはブロッキングを避けるように設計されている。

## 参考リンク

### Forums

- [SE-0327: On Actors and Initialization](https://forums.swift.org/t/se-0327-on-actors-and-initialization/53053)

- [SE-0327 (Second Review): On Actors and Initialization](https://forums.swift.org/t/se-0327-second-review-on-actors-and-initialization/54093)

### プロポーザルドキュメント

- [On Actors and Initialization](https://github.com/apple/swift-evolution/blob/main/proposals/0327-actor-initializers.md)

### 関連PR

- [[SE-327] Implement flow-sensitive actor isolation for actor inits](https://github.com/apple/swift/pull/40028)
- [[SE-327] Remove redundant global-actor isolation](https://github.com/apple/swift/pull/40868)

### その他

- [Actor reentrancy](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md#actor-reentrancy)
- [false sharing](https://en.wikipedia.org/wiki/False_sharing)
- [データフロー解析](https://ja.wikipedia.org/wiki/データフロー解析)

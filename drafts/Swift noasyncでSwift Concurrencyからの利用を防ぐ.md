# Swift noasyncでSwift Concurrencyからの利用を防ぐ

- [Swift noasyncでSwift Concurrencyからの利用を防ぐ](#swift-noasyncでswift-concurrencyからの利用を防ぐ)
  - [概要](#概要)
  - [導入バージョン](#導入バージョン)
  - [内容](#内容)
    - [問題点](#問題点)
    - [解決策](#解決策)
  - [置換API](#置換api)
  - [さらに設計詳細](#さらに設計詳細)
  - [既存ソースへの影響](#既存ソースへの影響)
  - [代替案](#代替案)
    - [伝播](#伝播)
      - [暗黙的にunavailabilityを伝播させる](#暗黙的にunavailabilityを伝播させる)
      - [明示的にunavailabilityを示すようにする](#明示的にunavailabilityを示すようにする)
    - [別の属性](#別の属性)
  - [将来の検討事項](#将来の検討事項)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

Swift Concurrencyの登場によって中断(suspend)前のスレッドと異なるスレッドでTaskを再開(resume)できるようになった。そのため、スレッドローカルなストレージやlock、mutex、セマフォなどは中断点をまたいだ使用はされるべきではない。

```swift
func badAsyncFunc(_ mutex: UnsafeMutablePointer<pthread_mutex_t>, _ op : () async -> ()) async {
    // ...
    pthread_mutex_lock(mutex)
    await op()
    pthread_mutex_unlock(mutex) // ❌ 別のスレッドでunlockするかもしれない
    // ...
}
```

もし上記の例が異なるスレッドで再開した場合、`pthread_mutex_unlock`は`op`実行前のスレッドと同じスレッドで実行されなければならないため、挙動は不定である。

そこで、今回、`@available`を拡張して`noasync`属性種別を追加して、asyncコンテキストから直接使用できなくする。

## 導入バージョン

Swift5.7

## 内容

### 問題点

Swift Concurrencyの登場によって、異なるスレッドでタスクの中断(suspend)と再開(resume)ができるようになった。これはリソースの計算などに大変有用だが、これを信用しているプログラマをびっくりさせる嫌な落とし穴がある。一つは`pthread_mutex_t`を異なるスレッドでunlockすることによって挙動が不定になることであり、lockしたスレッドは簡単に予期せぬデッドロックを起こしやすく、中断点をまたがるスレッドローカルのストレージの読み書きはデバッグしづらい意図せぬ挙動になるかもしれない。

### 解決策

そこで`@available`を拡張して`noasync`属性種別を受け入れるようにする。これは多くの宣言に適用できるが、`deinit`には使えない。これは明示的に呼ばれるものではなく、どこからでも呼び出せなければならないからである。

```swift
@available(*, noasync)
func doSomethingNefariousWithNoOtherOptions() { }

@available(*, noasync, message: "use our other snazzy API instead!")
func doSomethingNefariousWithLocks() { }

func asyncFun() async {
    // Error: doSomethingNefariousWithNoOtherOptions is unavailable from
    //        asynchronous contexts
    doSomethingNefariousWithNoOtherOptions()

    // Error: doSomethingNefariousWithLocks is unavailable from asynchronous
    //        contexts; use our other snazzy API instead!
    doSomethingNefariousWithLocks()
}
```

この`noasync`属性は直近のasyncコンテキストのAPIの使用しか防ぐことができない。つまり、使用不可能なAPIをsyncコンテキストでラップして呼び出した場合はエラーにならない。これは特定の方法限定でasyncコンテキストで安全に使える場合に活用できる。下記の例では、pthread mutextをクリティカルセクション(※)でラップして使用する例を示している。

(※) [クリティカルセクション](https://e-words.jp/w/クリティカルセクション.html)

```swift
func goodAsyncFunc(_ mutex: UnsafeMutablePointer<pthread_mutex_t>, _ op : () -> ()) async {
    // pthread_mutex_lockは他の関数にラップされているのでエラーにならない
    with_pthread_mutex_lock(mutex, do: op)
}

func with_pthread_mutex_lock<R>(
    _ mutex: UnsafeMutablePointer<pthread_mutex_t>,
    do op: () throws -> R) rethrows -> R {
    switch pthread_mutex_lock(mutex) {
        case 0:
            defer { pthread_mutex_unlock(mutex) }
            return try op()
        case EINVAL:
            preconditionFailure("Invalid Mutex")
        case EDEADLK:
            fatalError("Locking would cause a deadlock")
        case let value:
            fatalError("Unknown pthread_mutex_lock() return value: '\(value)'")
    }
}
```

上記は、lockが中断点をまたいでいないため、`pthread_mutex_lock`と`pthread_mutex_unlock`の安全なラッパである。は`op`実行前のスレッドと同じスレッドで実行されなければならない。ただし、これが当てはまるには、クリティカルセクションの操作がsyncな必要がある。次のスニペットは、syncクロージャを使用して使用できない関数を呼び出し、属性によって提供される保護を回避している:

```swift
@available(*, noasync)
func pthread_mutex_lock(_ lock: UnsafeMutablePointer<pthread_mutex_t>) {}

func asyncFun(_ mutex : UnsafeMutablePointer<pthread_mutex_t>) async {
    // ❌　pthread_mutex_lockはasyncコンテキストで使用できない
    pthread_mutex_lock(mutex)

    // ⭕️　pthread_mutex_lockはasyncコンテキストから呼ばれていない
    _ = { unavailableFun(mutex) }()

    await someAsyncOp()
}
```

## 置換API

場合によっては、安全な代替手段を提供することが可能。`with_pthread_mutex_lock`は、pthread mutexのlockとunlockをラップする安全な方法を提供例。

その他の場合、特定のactorからのAPIを使用しても安全な場合がある。例えば、スレッドローカルのストレージを使用するAPIは、一般にasync関数での使用は安全ではないが、MainActorはMainThreadのみを使用するため、MainActor上の関数は安全。

使用できないAPIには引き続きそのようにアノテーションを付ける必要があるが、そのような操作のためのactorの拡張として代替関数を実装できる:

```swift
@available(*, noasync, renamed: "readIDFromMainActor()", message: "use readIDFromMainActor instead")
func readIDFromThreadLocal() -> Int { }

@MainActor
func readIDFromMainActor() -> Int { readIDFromThreadLocal() }

func asyncFunc() async {
    // ❌　どのスレッドにいるのかわからない
    let id = readIDFromThreadLocal()

    // ⭕️ MainThread上だとわかっている。
    // 中断によってMainActorへと移動する。
    let id = await readIDFromMainActor()
}
```

以下の例に示すように、sync APIをactorに制限することも同様できる。syncの`save`関数はpublic APIの一部であるため、source breakを発生させずに`DataStore` actorにそのまま移動させることはできない。代わりに、`noasync` availability属性のアノテーションを付けることができる。`DataStore.save`は、元のsyncの`save`関数の薄いラッパである。`save`をasyncコンテキストからの呼び出すには、`DataStore` actorを介してのみ実行でき、協調スレッドプールが元の`save`関数と結びついていないことを保証する。元の`save`関数は、以前と同じようにsyncコンテキストで引き続き使用できる。

```swift
@available(*, noasync, renamed: "DataStore.save()")
public func save(_ line: String) { }

public actor DataStore { }

public extension DataStore {
    func save(_ line: String) {
        save(line)
    }
}
```

## さらに設計詳細

使用できない関数がasyncコンテキストから使用されていないことの確認は、ゆるく行われる。asyncコンテキストから直接呼び出された使用できない関数のみが診断される。これにより、async関数の本文を再帰的に型チェックして、asyncコンテキストから暗黙的に使用可能かどうかを判断したり、適切にアノテーションが付けられていることを確認したりする必要性を避ける。

型チェッカはsync関数から診断する必要はないが、完全に省略することはできない。syncコンテキスト内でasyncコンテキストを宣言することが可能であり、この場合は診断する必要がある。

```swift
@available(*, noasync)
func bad2TheBone() {}

func makeABadAsyncClosure() -> () async -> Void {
    return { () async -> Void in
        bad2TheBone() // ❌ asyncコンテキストからは使用できない
    }
}
```

## 既存ソースへの影響

Swift3と4由来のコードはこの属性がないので影響はない。

後にこの属性を追加して変更されるAPIを現在使っているasyncコードに影響する。移行を容易にするために、この属性はSwift 5.6で警告を発し、Swift 6では完全なエラーになることを提案。誰かが本当に危険な動作を望んでいて、境界線での生活を楽しんでいる場合、上記のようにsyncクロージャでAPIをラップすることで診断を簡単に回避できる。

## 代替案

### 伝播

最初の議論は、次の3つの設計を含め、availabilityがどのように伝播するかに焦点を当てた:

- 暗黙的にunavailabilityを伝播させる
- 明示的にunavailabilityを示すようにする
- 薄いunavailabilityチェックを行う

最終的な決定は、薄いチェックを行うこと。暗黙的チェックと明示的チェックはどちらもパフォーマンスコストが高く、関数に別の色を追加するため、はるかに多くの考慮が必要。

この属性は、かなり限定された一連の特殊なユースケースで使用されることが期待されている。目標は、コンパイラのパフォーマンスに劇的な影響を与えることなく、ある程度の保護を提供すること。

#### 暗黙的にunavailabilityを伝播させる

暗黙的にunavailabilityを伝播させると、unavailability関数を呼び出した関数にもそのままunavailabilityが適用される。これにより、開発者のオーバーヘッドが最小になり、使用できない関数を誤って間接的に使用することがなくなる。

```swift
@available(*, noasync)
func blarp1() {}

func blarp2() {
    // 暗黙的にblarp2はunavailableになる
    blarp1()
}

func asyncFun() async {
    // ❌ blarp2はblarp1を呼び出しているので、暗黙的にasyncコンテキストから使用できない
    blarp2()
}
```

残念ながら、これを計算するには非常にコストがかかり、宣言がasyncコンテキストから使用可能かどうかを判断するために、特定の関数が呼び出すすべての関数の本文を型チェックする必要がある。関数宣言を決定するために関数本文の部分的な型チェックでさえ必要とすることは、法外にコストがかかり、特にインクリメンタルコンパイルのパフォーマンスに悪影響を及ぼす。

使用できない関数が含まれている場合でも、asyncコンテキストから使用できることがわかっている特定の関数のチェックを無効にするには、追加の属性が必要になる。安全であるが「使用できない」関数の例は、上記の`with_pthread_mutex_lock`がある。


#### 明示的にunavailabilityを示すようにする

この設計は、現在のavailabilityとほぼ同じように動作する。使用できない関数を使用するには、呼び出し元の関数にunavailability属性で明示的にアノテーションを付ける必要がある。そうしないと、エラーが発生する。

暗黙的なunavailabilityの伝播と同様に、関数に安全でないAPIが含まれている可能性がある一方で、asyncコンテキストで安全に使用できる方法の場合でもそれらを使用することを示す追加の属性が必要になる。

この設計の利点は、安全でないAPIが明示的に正しく処理され、バグが回避されること。さらに、async関数の型チェックは適度に実行可能であり、async関数によって呼び出されるすべてのasync関数の本文を再帰的に型チェックする必要はない。

残念ながら、すべてのsync関数に正しくアノテーションが付けられるようにするには、すべてのasync関数の本文を走査する必要がある。これにより、開発者のオーバーヘッドが高くなるので、暗黙的なavailabilityチェックとの利点と逆。

### 別の属性

使用できないAPIにアノテーションを付けるために、`@unavailableFromAsync`というスペルの別の属性を使用することを検討した。検討した結果、`@available`属性の機能の多くを再実装する必要があることが明らかになった。

`@unavailableFromAsync`からavailabilityの種類への変わった経緯は次のとおり。

- 特定のAPIは、プラットフォームごとに実装が異なる場合があるため、asyncコンテキストで安全に使用できる方法で実装される場合もあれば、そうでない場合もある
- APIは現在、asyncコンテキストでの使用には安全ではない方法で実装されている可能性があるが、将来的には安全になる可能性がある
- これを`@available`とマージすることで、メッセージ、名前の変更、およびシリアル化を含むものを自由に使用できる

マージする場合の課題は、主に、このモードと他のavailabilityモードの検証モデルの違いに焦点が当たる。上で説明したように、`noasync`はゆるいチェックであり、使用できない関数を使用しているAPIにもアノテーションを付ける必要はない。他のavailabilityチェックでは、availability情報を伝播する必要がある。

## 将来の検討事項

カスタムエグゼキュータは、将来の機能として言語の一部になるように提案されている。APIをカスタムエグゼキュータに制限することは、そのAPIをactorに制限することと同じ。違いは、置換APIを提供するactorが、`unownedExecutor`を目的のカスタムエグゼキュータでオーバーライドすること。

(将来的に期待される)構文の一部を試してみると、この保護は次の例のようになる:

```swift
protocol IOActor : Actor { }

extension IOActor {
    nonisolated var unownedExecutor: UnownedSerialExecutor {
        return getMyCustomIOExecutor()
    }
}

@available(*, noasync, renamed: "IOActor.readInt()")
func readIntFromIO() -> String { }

extension IOActor {
    // IOActor置換API
    func readInt() -> String { readIntFromIO() }
}

actor MyIOActor : IOActor {
    func printInt() {
        // ⭕️ IOActor上のsyncコンテキスト
        print(readInt())
    }
}

func print(myActor : MyIOActor) async {
    // ⭕️ readIntFromIOはIOActorのエグゼキュータからのみ呼び出される
    print(await myActor.readInt())
}
```

`IOActor`は、その`unownedExecutor`を特定のカスタムのIOエグゼキュータでオーバーライドし、`readIntFromIO`関数への呼び出しをラップするsyncの`readInt`関数を提供している。`noasync`availability属性は、`readIntFromIO`が一般的にasyncコンテキストから使用できないことを保証する。`readInt`が呼び出されると、カスタムIOエグゼキュータを使用する`MyIOActor`へホップが発生する。

## 参考リンク

### Forums

- [Unavailable From Async Attribute](https://forums.swift.org/t/pitch-unavailability-from-asynchronous-contexts/53877)
- [SE-0340: Unavailable From Async Attribute](https://forums.swift.org/t/se-0340-unavailable-from-async-attribute/54852)
- [[Accepted] SE-0340: Unavailable From Async Attribute](https://forums.swift.org/t/accepted-se-0340-unavailable-from-async-attribute/55356)

### プロポーザルドキュメント

- [Unavailable From Async Attribute](https://github.com/apple/swift-evolution/blob/main/proposals/0340-swift-noasync.md)

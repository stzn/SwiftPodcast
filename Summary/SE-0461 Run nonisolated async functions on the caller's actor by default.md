# デフォルトでnonisolated async 関数を呼び出し元のアクター上で実行する

バージョン:TBU

Upcoming Feature フラグ: `AsyncCallerExecution`

## 内容

nonisolated async 関数は、デフォルトで呼び出し元のアクターで実行されるようになる(アクターから隔離したい場合は明示的に指定が必要であり、呼び出し元アクターで実行することを明示することも可能)。

例えば、下記の `performAsync` 関数は、MyActorの隔離外で実行されるため、エラーが発生していた。

```swift
class NotSendable {
  func performSync() { ... }
  func performAsync() async { ... }
}

actor MyActor {
  let x: NotSendable
  func call() async {
    x.performSync() // 🟢
    await x.performAsync() // ❌
  }
}
```

この変更により、`performAsync` 関数は `MyActor` のアクターで実行されるため、エラーが発生しなくなる。

明示的に指定する場合は、`@execution` 属性を使用する。

`@execution(concurrent)`: 呼び出し元のアクターとは異なる隔離領域(nonisolated)で実行される。この変更が入る前のデフォルトの挙動。
`@execution(caller)`: 呼び出し元のアクターで実行される。何も指定しない場合は、これがデフォルトになるため、書かなくてもよい。

```swift
class NotSendable {
  func performSync() { ... }
  
  @execution(caller)
  func performAsync() async { ... }

  @execution(concurrent)
  func alwaysSwitch() async { ... }
}

actor MyActor {
  let x: NotSendable

  func call() async {
    x.performSync() // 🟢 MyActor と同じ隔離領域
    await x.performAsync() // 🟢　MyActor と同じ隔離領域
    await x.alwaysSwitch() // ❌ nonisolated
  }
}
```

<details>
<summary>`@execution(caller)` 属性の使用例</summary>

```swift
class NotSendable { ... }

func useAsValue(_ ns: NotSendable) async { ... }

@MainActor let global: NotSendable = .init()

@execution(caller)
func runOnActor(ns: NotSendable) async {}

@MainActor
func callSendableClosure() async {
  // closure の型は @Sendable @execution(caller) (NotSendable) -> Void
  let closure = runOnActor(ns:) 

  let ns = NotSendable()
  await closure(ns) // 🟢 MainActor 上で実行
  await closure(global) // 🟢 MainActor 上で実行
}

callSendableClosure(useAsValue)
```

</details>

<details>
<summary>`@execution(concurrent)` 属性の使用例</summary>

```swift

struct S: Sendable {
  @execution(concurrent)
  func alwaysSwitch() async { ... }
}

class NotSendable { ... }
@execution(concurrent)
func alwaysSwitch(ns: NotSendable) async { ... }

actor MyActor {
  let ns: NotSendable = .init()

  func callConcurrent() async {
    await alwaysSwitch(ns: ns) // ❌　ns は MyActor の隔離領域だか、alwaysSwitch は concurrent なので MyActor とは異なる隔離領域(nonisolated)で実行される

    let disconnected = NotSendable()
    await alwaysSwitch(ns: disconnected) // 🟢
  }
}
```

</details>

### Unstructured Task

Unstructured Task は明示的に指定しない限り、アクター上では実行されない。これは nonisolated 同期/ async 関数で同じ。isolated パラメータを持つ関数は、明示的に指定する必要がある。

```swift
class NotSendable {
  var value = 0
}

@execution(caller)
func createTask(ns: NotSendable) async {
  Task {
    // このタスクは createTask と同じアクター隔離領域では実行されない
    ns.value += 1 // ❌
  }
}
```

### #isolationマクロ

暗黙的な `isolated` パラメータとして展開される。

```swift
nonisolated func printIsolation() async {
  let isolation = #isolation
  print(isolation) // Optional(Swift.MainActor)
}

@main
struct Program {
  // 暗黙的に MainActor に隔離されている
  static func main() async throws {
    await printIsolation()
  }
}
```

つまり、isolatedパラメータのデフォルト引数に `#isolation` マクロを指定すると、`@execution(caller)` 関数から呼び出された際は同じアクター上で実行される

```swift
class NotSendable { ... }

func explicitIsolationInheritance(
  ns: NotSendable,
  isolation: isolated (any Actor)? = #isolation
) async { ... }

@execution(caller)
func printIsolation(ns: NotSendable) async {
  await explicitIsolationInheritance(ns: ns) // 🟢 caller のアクター上で実行される
}
```

nonisolated 同期関数や `@execution(concurrent)` 関数の場合、`#isolation` マクロは `nil` になる


### 関数の変換

関数の変換は隔離状態を変更できる。これは、元の関数を非同期で必要に応じて呼び出す新しい隔離状態を持つクロージャのように考えられる。

```swift
@globalActor actor OtherActor { ... }

func convert(
  closure: @OtherActor () -> Void
) {
  let mainActorFn: @MainActor () async -> Void = closure
  // ↑は↓と同じ
  let mainActorEquivalent: @MainActor () async -> Void = {
    await closure()
  }
}
```

隔離境界を越える場合、引数と戻り値は `Sendable` でなければならない。

```swift
class NotSendable {}
actor MyActor {
  var ns = NotSendable()

  func getState() -> NotSendable { ns }
}

func invalidResult(a: MyActor) async -> NotSendable {
  // ❌ 呼び出し元が MyActor とは限らず、アクターの外からアクターの内部状態(ns)にアクセスできてしまう
  let grabActorState: @execution(caller) () async -> NotSendable = a.getState 

  return await grabActorState()
}
```

隔離境界を越えなければこの制限はない。

```swift
class NotSendable {}

@execution(caller)
nonisolated func performAsync(_ ns: NotSendable) async { ... }

@MainActor
func convert(ns: NotSendable) async {
  // 🟢　performAsync は MainActor 上で実行される
  let runOnMain: @MainActor (NotSendable) async -> Void = performAsync

  await runOnMain(ns)
}
```

<details>
<summary>関数変換早見表</summary>

隔離境界を越える関数変換では、引数と戻り値の型が `Sendable`、かつ変換先の関数型は async でなければならない。nonisolated の同期関数と `@execution(caller)` が付いた nonisolated async 関数の変換ルールは同じで、表では「非隔離」としている。

| 変換前の隔離領域            | 変換後の隔離領域             | 隔離境界を越えるか？ |
|--------------------------|----------------------------|------------------|
| 非隔離              | アクター隔離             | いいえ               |
| 非隔離              | `@isolated(any)`           | いいえ               |
| 非隔離              | `@execution(concurrent)`   | はい              |
| アクター隔離           | アクター隔離             | はい              |
| アクター隔離           | `@isolated(any)`           | いいえ               |
| アクター隔離           | 非隔離                | はい              |
| アクター隔離           | `@execution(concurrent)`   | はい              |
| `@isolated(any)`         | アクター隔離             | はい              |
| `@isolated(any)`         | 非隔離                | はい              |
| `@isolated(any)`         | `@execution(concurrent)`   | はい              |
| `@execution(concurrent)` | アクター隔離             | はい              |
| `@execution(concurrent)` | `@isolated(any)`           | いいえ               |
| `@execution(concurrent)` | 非隔離                | はい              |

</details>

#### 非Sendable関数型の場合

一度に1つの隔離領域のみがその関数を参照でき、関数の呼び出しは同時に発生してはいけない。これはregion-based isolation のチェックで強制される。

非 `Sendable` 関数がアクター隔離関数に変換される場合、関数値自体は、非 `Sendable` 関数のキャプチャと共にアクターの領域にマージされる。

```swift
class NotSendable {
  var value = 0
}

@execution(caller)
func convert(closure: () -> Void) async {
  let ns = NotSendable()
  let disconnectedClosure = {
    ns.value += 1
  }
  let valid: @MainActor () -> Void = disconnectedClosure // 🟢　MainActor に隔離される
  await valid()

  let invalid: @MainActor () -> Void = closure // ❌ closure が呼び出し元のアクターと MainActor の両方から同時に呼び出される可能性がある
  await invalid()
}
```

元の関数がアクターとは異なる隔離領域で呼び出される必要がある場合、非 `Sendable` 関数型をアクター隔離型に変換することはできない。

```swift
@execution(caller)
func convert(
    fn1: @escaping @execution(concurrent) () async -> Void,
) async {
    // ❌ fn1 は concurrent なので
    // MainActor とは異なる隔離領域(nonisolated)で実行される可能性がある
    let fn2: @MainActor () async -> Void = fn1 

    await withDiscardingTaskGroup { group in
      group.addTask { await fn2() }
      group.addTask { await fn2() }
    }
}
```

一般的に、アクターに隔離された関数型から nonisolated 関数型へ変換する場合は、nonisolated 関数型が任意の隔離領域から呼び出される可能性があるため隔離境界を越える。また、変換がアクター上で発生し、新しい関数型が非 `Sendable` でない場合、その関数はアクターからしか呼び出せないため、新しい関数型はそのアクターに隔離され、関数変換できる。

```swift
class NotSendable {}

@MainActor class C {
  var ns: NotSendable

  func getState() -> NotSendable { ns }
}

func call(_ closure: () -> NotSendable) -> NotSendable {
  return closure()
}

@MainActor func onMain(c: C) {
  // c.getState: @MainActor () -> NotSendable から closure: () -> NotSendable への変換
  // closure は MainActor に隔離される
  // result は MainActor に隔離される
  let result = call(c.getState)
}
```
### エグゼキュータの切り替え

async 関数は、関数に入るときや他の async 関数を呼び出した後に、実装内でエグゼキュータを切り替える。同期関数にはエグゼキュータを切り替える機能がない。同期関数の呼び出しが隔離境界を越える場合、その呼び出しは、async コンテキストで行われ、エグゼキュータの切り替えは、呼び出し元で発生させなければならない。

`@execution(concurrent)` の付いた async 関数は、デフォルトのエグゼキュータ(global concurrent executor)に切り替わり、その他すべての async 関数は、呼び出し元のアクターのエグゼキュータに切り替わる

<details>
<summary>`@execution(concurrent)` 属性の使用例</summary>

```swift
@MainActor func runOnMainExecutor() async {
  // MainActor のエグゼキュータに切り替わる

  await runOnGenericExecutor()

  // MainActor のエグゼキュータに切り替わる
}

@execution(concurrent) func runOnGenericExecutor() async {
  // デフォルトのエグゼキュータに切り替わる

  await Task { @MainActor in
    // MainActor のエグゼキュータに切り替わる

    ...
  }.value

  // デフォルトのエグゼキュータに切り替わる
}
```

</details>

`@execution(caller)` 関数は、デフォルトのエグゼキュータに切り替えるのではなく、呼び出し元のアクターのエグゼキュータに切り替わる

<details>
<summary>`@execution(concurrent)` 属性の使用例</summary>

```swift
@MainActor func runOnMainExecutor() async {
  // MainActorのエグゼキュータに切り替わる
  ...
}

class NotSendable {
  var value = 0
}

actor MyActor {
  let ns: NotSendable = .init()

  func callNonisolatedFunction() async {
    await inheritIsolation(ns)
  }
}

// @execution(caller) が付いているのと同じ
nonisolated func inheritIsolation(_ ns: NotSendable) async {
  // 呼び出し元のエグゼキュータに切り替わる

  await runOnMainExecutor()

  // 呼び出し元のエグゼキュータに切り替わる

  ns.value += 1
}
```

</details>

ほとんどの呼び出しでは、すでに呼び出し元のエグゼキュータ上で実行されているため、関数に入るときの切り替えはない。

Task Executor Preference は、nonisolated async 関数が実行される隔離領域を指定するために引き続き使用できる。ただし、カスタムエグゼキュータを持つアクターから nonisolated async 関数が呼び出された場合、Task Executor Preference は適用されない。そうでなければ、コードはデータ競合のリスクがある。これは、Task Executor Preference がカスタムエグゼキュータを持つアクターに隔離されたメソッドには適用されず、nonisolated async メソッドにアクターから可変状態が渡される可能性があるためである。

### asyncとしてインポートされるObjective-C関数

Objective-C からインポートされ、[SE-0297](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md)に適合する nonisolated 関数は、暗黙的に `@execution(caller)` としてインポートされる。この変更により、MainActor から Objective-C クラスの非同期関数を呼び出した際に発生する多くの既存のデータ競合の安全性問題が解消される。この変更は、並行性問題のエラーがなくなり、コードの実行時の挙動には影響を与えないため、upcoming feature フラグに依存しないしません。

### ソース互換性

nonisolated async 関数のセマンティクスが変わる。現在は `@execution(concurrent)` がデフォルトだが、今後は `@execution(caller)` がデフォルトになる。これは、(暗黙的、明示的どちらでも)nonisolatedで宣言された関数と関数値に適用される。

この変更によって、実装内で `@execution(concurrent)` 関数を呼び出し、アクターの隔離領域内で非 `Sendable` な状態を渡す場合、ソース互換性にわずかな影響を与える可能性がある。また、MainActor で実行されることを避けるために nonisolated async 関数を利用している場合に、MainActor で実行されるようになってパフォーマンスが低下する可能性がある。

この変更は、`AsyncCallerExecution` フラグの有効無効に関係なく有効なコードが書けるため、モジュール間でSE-0338と今回の変更が混ざり、実行セマンティクスを理解するのが難しくなる。そこで、`AsyncCallerExecution` フラグが有効でないモジュールの nonisolated async 関数にいずれの属性も指定されていない場合、コンパイラが警告を発するようにする。

古いバージョンの Swift ツールをサポートする必要があるパッケージは、`#if hasAttribute(execution)`を使用して警告を抑制し、Swift 5.8 に導入された hasAttribute 以前のツールバージョンとの互換性を維持できる。

```swift
#if hasAttribute(execution)
@execution(concurrent)
#endif
public func myAsyncAPI() async { ... }
```

## 補足

- この変更は既存のコードの挙動を変える可能性があるため、Upcoming Feature Flagを使用して導入される
- `concurrent` とは、「アクターと並行して実行される」という意味が込められている
- `@execution` 属性を付けることができるのは（暗黙的または明示的な）`nonisolated` 関数のみ。グローバルアクター、`isolated`、`@isolated(any)`と併用しようとするとエラーになる
- 同期関数には `@execution` 属性を付けることはできない(今後の機能拡張の可能性あり)

### SE-0338の意図

[SE-0338](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0338-clarify-execution-non-actor-async.md)は、nonisolated async 関数がアクターとは別の隔離領域で実行されるようにしたプロポーザル。その主な理由は、意図せずアクターをブロックすることを防ぐため。例えば、MainActor は UI スレッドであり、ブロックするとアプリケーション全体がフリーズする可能性がある。

例えば、

```swift
func runHeavyTask() async {
  for i in 0...1000000 {
    await Task.sleep(10_000_000_000)
    print(i)
  }
}

@MainActor
func runOnMainActor() async {
  await runHeavyTask() // メインアクター上で実行される
}
```
こうした事態を防ぐために、nonisolated async 関数は、アクターとは別のエグゼキュータで実行されるようになっていた。

### クロージャの隔離の推論(今回は変更なし)

クロージャの隔離は以下の2つの要因で決まる

- クロージャのコンテキスト型が `@Sendable` または `sending` であるかどうか(ある場合は、nonisolated として推論される)
- 上記ではない場合、クロージャの推論される隔離は、包含するコンテキストと同じである。隔離された値を明示的にキャプチャしない場合、nonisolated として推論される

```swift
class NotSendable { ... }

@MainActor
func closureOnMain(ns: NotSendable) async {
  let syncClosure: () -> Void = {
    // MainActor に隔離されていると推論される
    print(ns) // 🟢 MainActor に隔離されている
  }

  // MainActor 上で実行される
  syncClosure()

  let asyncClosure: (NotSendable) async -> Void = {
    // MainActor に隔離されていると推論される
    print($0)
  }

  // MainActor 上で実行される
  // 🟢 MainActor に隔離されているため、nsを渡すことができる
  await asyncClosure(ns)
}
```

</details>

### ABI互換性

呼び出し元のアクターで実行するセマンティクスを採用することは、呼び出し元のアクターをパラメータとして渡す必要があるため、ABI の変更となる。ただし、Concurrency ライブラリの多くのAPIでは、`isolated` パラメータと `#isolation` を使用して同様の変更を段階的に導入しており、このような動作を採用したい互換性を維持したいライブラリに対して、この変換を自動的に行うツールを提供できる可能性がある。

例えば、nonisolated async 関数が public な ABI で、この変更を含む Swift ランタイムのバージョンより前から利用可能な場合、コンパイラは関数に対して2つの別々のエントリーポイントを生成できる(元の関数の実装が inlinable な場合にのみ可能)。

```swift
@_alwaysEmitIntoClient
public func myAsyncFunc() async {
  // 元の実装
}

@execution(concurrent)
@_silgen_name(...) // 元のシンボル名を維持するため
@usableFromInline
internal func abi_myAsyncFunc() async {
  // 既存のコンパイル済みコードは、
  // この関数への呼び出しを常にデフォルトのエグゼキュータ上で実行し続ける
  await myAsyncFunc()
}
```

### 実装時の注意点

`@execution(caller)` 関数は暗黙的なアクターのパラメータを受け入れる必要がある。これは、アクター隔離された関数に `@execution(caller)` を追加すること、または関数を `@execution(concurrent)` から `@execution(caller)` に変更することは、互換性のない変更であるということである。

## プロポーザルリンク

- [Run nonisolated async functions on the caller's actor by default](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)

## Forums

- [Pitch](https://forums.swift.org/t/pitch-inherit-isolation-by-default-for-async-functions/74862)

- [Review](https://forums.swift.org/t/se-0461-run-nonisolated-async-functions-on-the-callers-actor-by-default/77987)

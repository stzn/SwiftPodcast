# Swift Taskのイニシャライザとselfキャプチャの関係

- [Swift Taskのイニシャライザとselfキャプチャの関係](#swift-taskのイニシャライザとselfキャプチャの関係)
  - [概要](#概要)
  - [内容](#内容)
    - [Taskイニシャライザの定義](#taskイニシャライザの定義)
    - [クロージャ内のself](#クロージャ内のself)
      - [メモリリーク](#メモリリーク)
      - [インスタンスの解放遅延](#インスタンスの解放遅延)
    - [Taskのイニシャライザはメモリリークを起こさない？](#taskのイニシャライザはメモリリークを起こさない)
    - [Taskでもインスタンスの解放遅延問題は起きる](#taskでもインスタンスの解放遅延問題は起きる)
    - [キャンセルして処理が行われないようにする](#キャンセルして処理が行われないようにする)
    - [無限ループでTaskのイニシャライザ内でもメモリリークが発生することがある。](#無限ループでtaskのイニシャライザ内でもメモリリークが発生することがある)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [その他](#その他)
## 概要

Taskのイニシャライザの引数に渡すクロージャ内では、これまでのクロージャとは異なり、`class`などの参照型では`self`を明示的に記載する必要がない。Taskのイニシャライザはどういう仕組みになっているのか、また使用する際に気をつけるべきなどを改めて考えてみる。

## 内容

### Taskイニシャライザの定義

Taskの優先順位を決める`priority`と`@Sendable async`なクロージャを引数に取る

```swift
@available(SwiftStdlib 5.1, *)
extension Task where Failure == Error {
    @discardableResult
    @_alwaysEmitIntoClient
    public init(
        priority: TaskPriority? = nil,
        @_inheritActorContext @_implicitSelfCapture operation: __owned @Sendable @escaping () async throws -> Success
    ) { ... }
}
```

https://github.com/apple/swift/blob/main/stdlib/public/Concurrency/Task.swift#L463

- `@discardableResult`

戻り値が必要ないfire-and-forget形式のものは戻り値を気にしなくて良い。戻り値を取得したり、キャンセルしたい場合に生成したインスタンスを活用できる。

- `@_alwaysEmitIntoClient`

クライアントコードの関数の本文に強制的に埋め込まれる。ABIを壊さずに実装を入れ替えることができる。
https://github.com/apple/swift/blob/967a8b439f8dbd4580f652e378bb246e6eddb3c8/docs/ReferenceGuides/UnderscoredAttributes.md#_alwaysemitintoclient

- `@_inheritActorContext`

この属性を指定された`@Sendable async`なクロージャ内は、定義された位置を基に周りのactorのコンテキスト(どのactor上で実行されるのかの情報)を継承する

下記の例で`onlyOnMainActor`に`await`が必要かどうかを見てみる。

```swift
@MainActor func onlyOnMainActor() { }

func acceptAsyncSendableClosure<T>(_: @Sendable () async -> T) { } // @_inheritActorContextなし
func acceptAsyncSendableClosureInheriting<T>(@_inheritActorContext _: @Sendable () async -> T) { }

@MainActor func testCallFromMainActor() {
    acceptAsyncSendableClosure {
        // global関数'onlyOnMainActor()'をそのactorのコンテキストの外から呼び出すことは暗黙的asyncになる
        onlyOnMainActor() // ❌ expression is 'async' but is not marked with 'await'
    }

    acceptAsyncSendableClosure {
        await onlyOnMainActor() // ⭕️
    }

    acceptAsyncSendableClosureInheriting {
        onlyOnMainActor() // ⭕️
    }

    acceptAsyncSendableClosureInheriting {
        await onlyOnMainActor() // ⚠️ no 'async' operations occur within 'await' expression
    }
}
```

https://github.com/apple/swift/blob/967a8b439f8dbd4580f652e378bb246e6eddb3c8/docs/ReferenceGuides/UnderscoredAttributes.md#_inheritactorcontext

- `@_implicitSelfCapture`

`self`が参照型であってもこの属性が指定されたクロージャ内では`self`がキャプチャされることを明示しなくても良い

```swift
class C {
    func f() {}
    func g(_: @escaping () -> Void) {
        g({ f() }) // ❌ error: call to method 'f' in closure requires explicit use of 'self'
    }
    func h(@_implicitSelfCapture _: @escaping () -> Void) {
        h({ f() }) // ⭕️
    }
}
```

https://github.com/apple/swift/blob/967a8b439f8dbd4580f652e378bb246e6eddb3c8/docs/ReferenceGuides/UnderscoredAttributes.md#_implicitselfcapture

- `__owned`

メモリ管理に使われるキーワードで、呼び出し側から引数の所有権が譲渡される(retainする)

### クロージャ内のself

`class`などの参照型の中で、クロージャを使用し、その内部で`self`(=`class`のインスタンス)を参照すると強参照になる。これがいくつか問題になる場合がある。
例えば、ある数字のリストをタップすると選んだ数字を表示するページに遷移するシンプルなアプリを作る。ボタンを押すとアラートを表示する。

<img alt="画面の基本動作" src="../images/Swift%20Taskのイニシャライザとselfキャプチャの関係/basic_behavior.gif" width="160" height="284" />

#### メモリリーク

`class`のプロパティにクロージャを割り当て、`self`をそのクロージャの内部で使用する(インスタンスのプロパティにアクセスしたり、メソッドを呼び出す)と、`self`を強参照し強参照サイクルが発生する。
クロージャは`class`と同様に参照型。クロージャをプロパティに割り当てると、そのクロージャの参照を割り当てていることになる。

例えば、下記の`DetailViewController`では、クロージャの参照を保持する`action`プロパティの中で、`number`プロパティにアクセスしていることで強参照が発生している。そのため、このクラスの`deinit`が呼ばれることがない。

```swift
class DetailViewController: UIViewController {
    ...

    private var number: Int
    private var action: ((Int) -> Void)?

    init(number: Int) {
        self.number = number
        super.init(nibName: nil, bundle: nil)

        // ↓↓↓↓↓↓↓↓↓強参照サイクル↓↓↓↓↓↓↓↓↓↓↓↓
        self.action = { number in
            self.number = number
        }
    }
    deinit {　print("deinit")　}
}
```

これを解消するためには、`self`をキャプチャリストを使って弱参照にする必要がある。

```swift
// ↓↓↓↓↓↓↓↓↓強参照サイクルにはならない↓↓↓↓↓↓↓↓↓↓↓↓
self.action = { [weak self] number in
    self?.number = number
}
```

#### インスタンスの解放遅延

メモリリークは発生しないものの、実装次第で問題になるケースもある。

例えば、Goボタンをタップした際に、`DetailViewController`閉じる動作を入れる。

```swift
final class DetailViewController: UIViewController {
    @objc private func onTap(_: UIButton) {
        dismiss(animated: true)
        DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
            self.showAlert()
        }
    }
}
```

こうするとアラートは表示されるだろうか？

<img alt="期待していないアラート表示" src="../images/Swift%20Taskのイニシャライザとselfキャプチャの関係/unexpected_show_alert.gif" width="160" height="284" />

画面を閉じてもアラートが表示される。クロージャ内で`self`を強参照して、このクロージャを抜けるまで`self`の参照が残っているため。

これが仮にとても重い処理で大量のデータ通信や処理が必要だった場合、不要にユーザのリソースを消費することになる。さらに、これが仮にデータベースの更新など副作用を発生するものであった場合は、見えずらい不具合につながる可能性もある。

これを解消するには、`self`の強参照をやめる。

```swift
final class DetailViewController: UIViewController {
    @objc private func onTap(_: UIButton) {
        ...
        DispatchQueue.main.asyncAfter(deadline: .now() + 2) { [weak self] in
            self?.showAlert()
        }
    }
}
```

### Taskのイニシャライザはメモリリークを起こさない？

`Task`の場合、本文は即時に実行されてnon-escapingと見なされるため、`self`の参照も本文内で完結するためメモリリークは起こらない。

https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#implicit-self

### Taskでもインスタンスの解放遅延問題は起きる

一方で上記の`DispatchQueue`と同様に、参照が残ることによる問題は生じる。先ほどの例を`Task`に置き換えて考えてみる。

```swift
@objc private func onTap(_: UIButton) {
    dismiss(animated: true)
    Task {
        try? await Task.sleep(nanoseconds: NSEC_PER_SEC * 2)
        showAlert()
    }
}
```

<img alt="Task期待していないアラート表示" src="../images/Swift%20Taskのイニシャライザとselfキャプチャの関係/task_unexpected_show_alert.gif" width="160" height="284" />

予想通り`self`はクロージャを抜けるまで強参照され、アラートが表示される。

では、`self`を弱参照でキャプチャするとどうなるだろうか？

```swift
@objc private func onTap(_: UIButton) {
    dismiss(animated: true)
    Task { [weak self] in
        guard let self = self else { return }
        try? await Task.sleep(nanoseconds: NSEC_PER_SEC * 2)
        self.showAlert()
    }
}
```

<img alt="TaskでSelfを弱参照しても期待していないアラート表示" src="../images/Swift%20Taskのイニシャライザとselfキャプチャの関係/task_with_weak_self_unexpected_show_alert.gif" width="160" height="284" />

この場合でもアラートが表示される。これはSwift Concurrencyがsuspendからresumeした際に必要な変数を内部で保持しているため、`self`の参照が存在し続けている。

これを解消するためには、`self`をOptionalバインディングをせずに、弱参照のまま使用する。

```swift
Task { [weak self] in
    try? await Task.sleep(nanoseconds: NSEC_PER_SEC * 2)
    self?.showAlert()
}
```

こうすることでアラートが呼ばれなくなる。

### キャンセルして処理が行われないようにする

しかし、これは`Task`がキャンセルされている訳ではなく、場合によっては処理が継続する可能性もある。例えば`self`のメソッドの呼び出しではない場合など。

そこで画面が非表示になるタイミングでキャンセル処理を行うことで、`showAlert`が呼ばれないようにする。

```swift

final class DetailViewController: UIViewController {
    private var task: Task<Void, Never>?
    ...
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        task?.cancel()
    }

    ...

    @objc private func onTap(_: UIButton) {
        dismiss(animated: true)
        Task {
            do {
                try await Task.sleep(nanoseconds: NSEC_PER_SEC * 2)
                self.showAlert()
            } catch {
                print(error) // CancellationError()
            }
        }
    }
}
```

`viewWillAppear`で`cancel`が呼ばれることで、`sleep`メソッドが`CancellationError`をスローするため、`catch`句に入り、`showAlert`は呼ばれない。

### 無限ループでTaskのイニシャライザ内でもメモリリークが発生することがある。

`Task`のイニシャライザ自体はメモリリークを起こさないが、内部の実装によってメモリリークにつながるケースがある。

例えば、下記のように`Notification`を待つケースを考えてみる。

```swift
private extension Notification.Name {
    static let numbers = Notification.Name("numbers")
}

final class DetailViewController: UIViewController {
    override func viewDidLoad() {
        ...
        Task {
            for await _ in NotificationCenter.default.notifications(named: .numbers) {
                showAlert()
            }
        }
    }
}
```

この場合、`Task`内部のforループは永遠に終わることなく`self`を強参照し続けるため、画面を`dismiss`しても`deinit`は呼ばれない。

この場合は`viewWillDisappear`などでキャンセルを行う必要がある。(`deinit`での`cancel`は呼ばれない)

```swift

final class DetailViewController: UIViewController {
    override func viewDidLoad() {
        ...
        task = Task {
            for await _ in NotificationCenter.default.notifications(named: .numbers) {
                showAlert()
            }
        }
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        task?.cancel()
    }
}
```

## 参考リンク

### Forums

### その他

- [Memory management when using async/await in Swift](https://www.swiftbysundell.com/articles/memory-management-when-using-async-await)

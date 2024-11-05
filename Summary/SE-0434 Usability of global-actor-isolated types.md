# Usability of global-actor-isolated types

バージョン: Swift 6.0
Upcoming feature: GlobalActorIsolatedTypesUsability

## 内容

グローバルアクターに隔離された型で使いづらい部分があったのでルールを変えてもっと使いやすくしよう。

新しいルール: 
- グローバルアクターに隔離された値型の`Sendable`型の格納プロパティ(Stored Property)は、`(unsafe)`を使用せずに`nonisolated`として宣言できる
- グローバルアクターに隔離された値型の`Sendable`型の格納プロパティは、そのプロパティを定義するモジュール内で`nonisolated`として扱われる
- グローバルアクターに隔離された関数とクロージャには、`@Sendable`が推論される。
- グローバルアクターに隔離されたクロージャは、`@Sendable`であっても、`Sendable`でない値をキャプチャできる
- グローバルアクターに隔離されたサブクラスは、`nonisolated`かつ`Sendable`でないクラスを継承することができるが、`Sendable`にはできない

注意: 他のモジュールで定義された格納プロパティは、将来的に計算プロパティ(Computed Property)に変更され、データ競合を起こす可能性があるため、ルールは変わらない。

例:

```swift

// Before

@MainActor struct S {
  // (unsafe)を付けないとエラー
  // xはSendableなので(unsafe)を付ける必要がない
  nonisolated(unsafe) var x = 0
}

extension S: Equatable {
  static nonisolated func ==(lhs: S, rhs: S) -> Bool {
    return lhs.x == rhs.x
  }
}

// After

@MainActor struct S {
  // nonisolatedを推論してくれるの付けなくても良い
  // nonisolatedを付けると外部のモジュールからも同期的にアクセスできる
  var x = 0
}
```

```swift

// Before

func test(globallyIsolated: @escaping @MainActor () -> Void) {
  Task {
    // @Sendableにしないとエラー
    // 実際は@MainActorに隔離されているので@Sendableでなくても良いはず
    await globallyIsolated()
  }
}

// After: エラーにならない

```

```swift

// Before

class NonSendable {}

func test(ns: NonSendable) async {
  let closure = { @MainActor in
    // nsはnonisolatedかつSendableではないのでエラー
    // ただし、@MainActorに隔離されているのでエラーになる必要はない
    print(ns)
  }

  await closure()
}

// After: エラーにならない

```

```swift

// Before

class NotSendable {}

// @MainActorに隔離されたサブクラスはnonisolatedのクラスを継承できない
// 暗黙的にSendableになるため、親クラスの可変状態に複数の隔離からアクセスできてデータ競合が発生する可能性がある
// しかし、Sendableではなくても使いたい場面はある
@MainActor
class Subclass: NotSendable {} 

// After: SubclassはSendableに推論されなくなったのでエラーにならない

```

## 補足

`@Sendable`ではないグローバルアクターに隔離された関数型は`@Sendable`に推論されるようになるため、既存のコードに影響が出る可能性がある。

## プロポーザルリンク

- [https://github.com/swiftlang/swift-evolution/blob/main/proposals/0434-global-actor-isolated-types-usability.md](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0434-global-actor-isolated-types-usability.md)

## Forums

- [acceptance](https://forums.swift.org/t/accepted-se-0434-usability-of-global-actor-isolated-types/72743)

- [review](https://forums.swift.org/t/se-0434-usability-of-global-actor-isolated-types/71187)

- [pitch](https://forums.swift.org/t/pitch-usability-of-global-actor-isolated-types/70799)

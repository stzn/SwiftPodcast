# Allow nonisolated to prevent global actor inference

バージョン: Swift6.1

## 内容

1. グローバルアクター隔離への推論を打ち消すために、`nonisolated`が使える型を増やす
2. より多くの型の格納プロパティに`nonisolated`が使えるようにする


### 1. `class`、`struct`、`enum`、`protocol`、`extension`に`nonisolated`が使えるようになる

これまでグローバルアクター隔離の推論を打ち消すのはとても大変だった。

```swift
@MainActor
protocol P {
  var x: Int { get }
}

struct S {}

// すべてのメソッドやプロパティに`nonisolated`をつける必要がある
extension S: P {
  nonisolated var x: Int { get { 1 } }
  nonisolated func test() { print(x) }
}
```

型自体のグローバルアクター隔離を打ち消すのはもっと複雑。

```swift
@MainActor
protocol GloballyIsolated {}

@FakeGlobalActor
protocol RemoveGlobalActor {}

protocol RefinedProtocol: GloballyIsolated, RemoveGlobalActor {} // 'RefinedProtocol'はどのグローバルアクター隔離されるかわからないからnonisolatedになる
```

これを改善。

例

```swift
@MainActor
protocol GloballyIsolated {}

struct S: GloballyIsolated {} // 暗黙的にグローバルアクター隔離される

nonisolated struct S: GloballyIsolated {} // グローバルアクター隔離されない

nonisolated protocol P: GloballyIsolated {} // グローバルアクター隔離されない
```

```swift
struct X {}

nonisolated extension GloballyIsolated {
  var x: NonSendable { .init() }
  func implicitlyNonisolated() {}
}

struct C: GloballyIsolated { // `C`自体はグローバルアクター隔離
  nonisolated func explicitlyNonisolated() {
    let _ = x // OK
    implicitlyNonisolated() // OK
  }
}
```

<details>

<summary>もう少し複雑な例
</summary>


```swift
nonisolated class K: GloballyIsolated {
  var x: NonSendable
  init(x: NonSendable) {
    self.x = x // OK, 'x' is non-isolated
  }
} 

nonisolated struct S: GloballyIsolated {
  var x: NonSendable
  init(x: NonSendable) {
    self.x = x // 'x'はnon-isolatedなのでOK
  }
} 

nonisolated enum E: GloballyIsolated {
  func implicitlyNonisolated() {}
  init() {}
}

struct TestEnum {
  nonisolated func call() {
    E().implicitlyNonisolated() // OK
  }
}
```

</details>

注意: 明示的にグローバルアクターが宣言されている場合や`isolated`パラメータを使っている場合は`nonisolated`を使うことはできない。

```swift
@MainActor
nonisolated struct Conflict {} // 複数の隔離が混在しているため判断できない
```

### 2. より多くの型の格納プロパティに`nonisolated`が使えるようにする

#### 2.1 Sendableではない型の格納プロパティ

元々暗黙的に`nonisolated`がついていたが明示的につけることができるようになった。

```swift
class MyClass {
  nonisolated var x: NonSendable = NonSendable() // OK
}
```

補足: `MyClass`は`Sendable`ではないため複数の隔離ドメインから同時にアクセスできない。そして`Sendable`ではない`nonisolated`な格納プロパティも1つの隔離ドメインからしかアクセスできない。なので安全。

### 2.2 `Sendable`な値型の可変な格納プロパティ

[SE-0434](./SE-0434%20Usability%20of%20global-actor-isolated%20types.md)でモジュール内の`Sendable`な`var`プロパティはnonisolatedとして扱われるようになったが、今回は、`Sendable`な値型のすべての格納プロパティがnonisolatedとして扱われるようになった。

```swift
protocol P {
  @MainActor var y: Int { get }
}

struct S: P {
  var y: Int // モジュール内では'nonisolated'として扱われる
}

struct F {
  nonisolated func getS(_ s: S) {
    let x = s.y // OK
  }
}
```

明示的に`nonisolated`をつけると定義したモジュール外からもnonisolatedとして同期的にアクセスできる。

```swift
// In Module A
public protocol P {
  @MainActor var y: Int { get }
}

public struct S: P {
  public nonisolated var y: Int // 'y'は明示的にnonisolated
}
```

補足: 値型が隔離ドメインを超える際はコピーが渡されるため、格納プロパティが`Sendable`であればデータ競合がなく安全。(コピー同士は影響を与えることはない)

### これはできない

#### Sendableな型のSendableではないプロパティ

```swift
@MainActor
struct InvalidStruct /* 暗黙的にSendable */ {
  nonisolated let x: NonSendable // ❌
}
```

理由: `Sendable`な型は隔離ドメインを超えてアクセスできるため、`Sendable`ではないプロパティが他の隔離ドメインからアクセスされデータ競合が発生する可能性がある(参照型)。

#### Sendableなclassのvarプロパティ

```swift
@MainActor
final class InvalidClass /* implicitly Sendable */ {
  nonisolated var test: Int = 1 // ❌
}
```

理由: `Sendable`な型は隔離ドメインを超えてアクセスできるため、nonisolatedな
プロパティは他の隔離ドメインから同期的にアクセスできてしまい、データ競合が発生する可能性がある。

## 補足

既存のコードに変更が必要になる場合がある。

```swift
class C: GloballyIsolated {} // 暗黙的に`Sendable`
```
はグローバルアクターに隔離されて暗黙的に`Sendable`になるが、


nonisolatedをつけると`Sendable`でなくなる。

```swift
nonisolated class C: GloballyIsolated // `Sendable`ではない
```

もし`Sendable`であることに依存している場合は、コードの修正が必要になる。

## プロポーザルリンク

- [https://github.com/swiftlang/swift-evolution/blob/main/proposals/0449-nonisolated-for-global-actor-cutoff.md](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0449-nonisolated-for-global-actor-cutoff.md)

## Forums

- [acceptance](https://forums.swift.org/t/accepted-se-0449-allow-nonisolated-to-prevent-global-actor-interference/75539)

- [review](https://forums.swift.org/t/se-0449-allow-nonisolated-to-prevent-global-actor-inference/75116)

- [pitch](https://forums.swift.org/t/pitch-allow-nonisolated-to-prevent-global-actor-inference/74502)

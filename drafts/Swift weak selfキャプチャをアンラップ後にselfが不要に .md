# Swift クロージャでweak selfキャプチャしたselfをアンラップ後、selfの記載が不要に 

- [Swift クロージャでweak selfキャプチャしたselfをアンラップ後、selfの記載が不要に](#swift-クロージャでweak-selfキャプチャしたselfをアンラップ後selfの記載が不要に)
  - [概要](#概要)
  - [内容](#内容)
    - [現在の問題点](#現在の問題点)
    - [これからできるようになること](#これからできるようになること)
    - [詳細](#詳細)
      - [暗黙的な`self`を可能に](#暗黙的なselfを可能に)
    - [ネストしたクロージャの場合](#ネストしたクロージャの場合)
    - [既存コードへの影響](#既存コードへの影響)
    - [ABI stabilityへの影響](#abi-stabilityへの影響)
    - [代替案](#代替案)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

`self`がキャプチャリストに書かれていた場合に`self`は省略できるようになっている。これを`weak self`で`self`をアンラップした後は明示的に書かなくてもよくする。


```swift
class ViewController {
    let button: Button

    func setup() {
        button.tapHandler = { [weak self] in
            guard let self else { return }
            dismiss()
        }
    }

    func dismiss() { ... }
}
```

## 内容

### 現在の問題点

クロージャ内に明示的な`self`が必要なのは、思わぬところで循環参照(retain cycle)を起こしてしまうことを避けるためである。SE-0269で、クロージャのキャプチャリストに記載していて、循環参照が起きそうにない場合は、`self`は明示的に記載しなくてよいように、このルールは緩和された。

```swift
button.tapHandler = { [self] in
    dismiss()
}
```

しかし、`weak self`に関しては、将来的な方向性として未決のままであった。

```swift
button.tapHandler = { [weak self] in
    guard let self else { return }
    self.dismiss()
}
```

これは、すでに明示的に`self`がキャプチャされており、コードの作者が`self`を明示することを利用者に要求するのは、そこまで重要ではない。これは一貫性がない状態であり、`weak self`キャプチャを使うと、クロージャの本文に不必要なノイズが入ってくることになる。


### これからできるようになること

`weak self`で`self`をアンラップした後は明示的に書かなくてもよくする。

```swift

class ViewController {
    let button: Button

    func setup() {
        button.tapHandler = { [weak self] in
            guard let self else { return }
            dismiss()
        }
    }

    func dismiss() { ... }
}
```

### 詳細

#### 暗黙的な`self`を可能に


以下の形式でオプショナルはアンラップでき、`self`がオプショナルではない範囲で、暗黙的な`self`を可能にする。

```swift
button.tapHandler = { [weak self] in
  guard let self else { return }
  dismiss()
}

button.tapHandler = { [weak self] in
  guard let self = self else { return }
  dismiss()
}

button.tapHandler = { [weak self] in
  if let self {
    dismiss()
  }
}

button.tapHandler = { [weak self] in
  if let self = self {
    dismiss()
  }
}

button.tapHandler = { [weak self] in
  while let self {
    dismiss()
  }
}

button.tapHandler = { [weak self] in
  while let self = self {
    dismiss()
  }
}
```

強参照や`unowned`参照の場合と同様に、`weak self`の場合も、クロージャ内の`self`のプロパティやメソッド呼び出し対して、コンパイラが`self.`を追加してくれる。

`self`をアンラップする前だと、エラーとなる。

```swift
button.tapHandler = { [weak self] in
    // error: explicit use of 'self' is required when 'self' is optional,
    // to make control flow explicit
    // fix-it: reference 'self?.' explicitly
    dismiss()
}
```

### ネストしたクロージャの場合

ネストしたクロージャはし、あいまいな循環参照の原因となる可能性があるため、より慎重に処理する必要がある。例えば、次のコードのコンパイルが許可されている場合、暗黙の`self.bar()`呼び出しにより、隠れた循環参照が導入される可能性がある。

```swift
couldCauseRetainCycle { [weak self] in
  guard let self else { return }
  foo()

  couldCauseRetainCycle {
    bar()
  }
}
```

[SE-0269](https://github.com/apple/swift-evolution/blob/main/proposals/0269-implicit-self-explicit-capture.md)に従って、`[weak self]`クロージャ内でネストした追加のクロージャは、暗黙的な`self`自己を使用するために、`self`を再度明示的にキャプチャする必要がある。

```swift
// ❌
couldCauseRetainCycle { [weak self] in
  guard let self else { return }
  foo()

  couldCauseRetainCycle {
    // error: call to method 'method' in closure requires 
    // explicit use of 'self' to make capture semantics explicit
    bar()
  }
}

// ⭕️
couldCauseRetainCycle { [weak self] in
  guard let self else { return }
  foo()

  couldCauseRetainCycle { [weak self] in
    guard let self else { return }
    bar()
  }
}

// ⭕️
couldCauseRetainCycle { [weak self] in
  guard let self else { return }
  foo()

  couldCauseRetainCycle {
    self.bar()
  }
}

// ⭕️
couldCauseRetainCycle { [weak self] in
  guard let self else { return }
  foo()

  couldCauseRetainCycle { [self] in
    bar()
  }
}
```

同様に、[SE-0269](https://github.com/apple/swift-evolution/blob/main/proposals/0269-implicit-self-explicit-capture.md)に従って、暗黙的な`self`は、`self`のオプショナルバインディングが、明確かつ排他的にキャプチャした`self`を指している場合にのみ、許可される。

```swift
button.tapHandler = { [weak self] in
    guard let self = self ?? someOptionalWithSameTypeOfSelf else { return }

    // error: call to method 'method' in closure requires explicit use of 'self' 
    // to make capture semantics explicit
    method() // この時点のselfは、キャプチャしたselfとは変わってしまっているため❌
}
```

### 既存コードへの影響

問題なし

### ABI stabilityへの影響

なし

### 代替案

アンラップ前に暗黙的な`self` をサポートすることもできた。

```swift
button.tapHandler = { [weak self] in
    dismiss() // as in `self?.dismiss()`
}
```

ただし、これにより、暗黙的な制御フローが、実質追加される。`dismiss()`は、`self`が`nil`でない場合にのみ実行され、実行されない可能性があることを示すものが何もない。`?.dismiss()`のように、オプショナルチェーンを意味する新しいスペルを作成することもできるが、それは既存の`self?.dismiss()`スペルよりも有意に優れているわけではない。

## 参考リンク

- [SE-0269 Increase availability of implicit self in @escaping closures when reference cycles are unlikely to occur](https://github.com/apple/swift-evolution/blob/main/proposals/0269-implicit-self-explicit-capture.md)

### Forums

- Allow implicit self for weak self captures, after self is unwrapped](https://forums.swift.org/t/allow-implicit-self-for-weak-self-captures-after-self-is-unwrapped/54262)

### プロポーザルドキュメント

- [Allow implicit self for weak self captures, after self is unwrapped](https://github.com/apple/swift-evolution/blob/main/proposals/0365-implicit-self-weak-capture.md)

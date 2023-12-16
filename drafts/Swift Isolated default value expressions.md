# Isolated default value expressions

- [Isolated default value expressions](#isolated-default-value-expressions)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
    - [プロポーザル](#プロポーザル)
    - [詳細](#詳細)
      - [デフォルト値の分離要件の推論](#デフォルト値の分離要件の推論)
      - [クロージャ](#クロージャ)
      - [制限](#制限)
    - [デフォルト値の分離要件の実施](#デフォルト値の分離要件の実施)
      - [デフォルト引数値](#デフォルト引数値)
      - [引数の評価](#引数の評価)
      - [格納プロパティの初期値](#格納プロパティの初期値)
    - [イニシャライザにおける格納プロパティの分離](#イニシャライザにおける格納プロパティの分離)
      - [分離された格納プロパティを分離の境界を越えて初期化する](#分離された格納プロパティを分離の境界を越えて初期化する)
      - [コンパイラが合成したイニシャライザにおけるデフォルト値の分離](#コンパイラが合成したイニシャライザにおけるデフォルト値の分離)
  - [ソース互換性](#ソース互換性)
    - [ABI互換性](#abi互換性)
  - [検討された代替案](#検討された代替案)
    - [全てのデフォルト初期化式から分離を取り除く](#全てのデフォルト初期化式から分離を取り除く)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

デフォルト値式はデフォルト引数とデフォルトの格納プロパティ値に対して使うことができる。格納プロパティに関するルールはデータ競合を引き起こし、デフォルト引数の値に関するルールは過度に制限的で、デフォルト値式を使用できる場所によってルールが矛盾しており、アクターの分離モデルを理解しにくくしています。このプロポーザルでは、デフォルト値式のアクター分離ルールを統一し、データ競合を排除し、デフォルト値の分離を安全に行えるようにすることで表現力を向上させる。

## 内容

### 動機

格納プロパティの初期値に関する現在のアクター分離ルールでは、データ競合が発生する。例えば、以下のコードは現在有効である。

```swift
@MainActor func requiresMainActor() -> Int { ... }
@AnotherActor func requiresAnotherActor() -> Int { ... }

class C {
  @MainActor var x1 = requiresMainActor()
  @AnotherActor var x2 = requiresAnotherActor()

  nonisolated init() {} // okay???
}
```

上記のコードでは、`@MainActor`分離関数と`@AnotherActor`分離関数の両方を同期的に呼び出す`nonisolated`の`init`によって、どのコンテキストでも`C()`のインスタンスを初期化できるため、アクターの分離チェックに違反し、`requiresMainActor()`と`requiresAnotherActor()`をそれぞれのアクター上の他のコードと同時に実行できる。

同様に、デフォルトの引数値に対する現在のアクター分離ルールはデータ競合を認めていないが、デフォルトの引数値は常に`nonisolated`であり、これは過度に制限的である。このルールでは、プログラマがMainActorからしか呼び出されない `@MainActor`分離関数のデフォルト引数値で `@MainActor`分離な呼び出しをを禁止している。例えば、以下のコードは完全に安全であっても無効である:

```swift
@MainActor class C { ... }

@MainActor func f(c: C = C()) { ... } // error

@MainActor func useFromMainActor() {
  f()
}
```

### プロポーザル

デフォルト値式を許可して、デフォルト値を使用するためには呼び出し側が分離要件を満していなければならないことを提案する。isolated要件はデフォルト値式から推測される。これまで通り、呼び出し側が分離要件を満たしていない場合、引数または格納プロパティに対して明示的に`await`を記述しなければならない。このルールにより、格納プロパティのデフォルト値の分離要件が満たされないため、上記の格納プロパティの例は`nonisolated init`の時点で無効となる。`f`のデフォルト引数に対する`@MainActor`の分離要件は呼び出し側によって満たされるため、このルールは上記のデフォルト引数の例も有効にする。

これらの規則により、上記の格納プロパティの例は、`nonisolated init`では無効となる:

```swift
@MainActor func requiresMainActor() -> Int { ... }
@AnotherActor func requiresAnotherActor() -> Int { ... }

class C {
  @MainActor var x1 = requiresMainActor()
  @AnotherActor var x2 = requiresAnotherActor()

  nonisolated init() {} // error: 'self.x1'と'self.x2'初期化されていない
}
```

`requiresMainActor()`と `requiresAnotherActor()`を `await` で明示的に呼び出せば、問題は解決する:

```swift
class C {
  @MainActor var x1 = requiresMainActor()
  @AnotherActor var x2 = requiresAnotherActor()

  nonisolated init() async {
    self.x1 = await requiresMainActor()
    self.x2 = await requiresAnotherActor()
  }
}
```

このルールにより、上記のデフォルト引数の例もデフォルト引数とそれを囲む関数はどちらも`@MainActor`で分離されているため、有効になる。

### 詳細


#### デフォルト値の分離要件の推論

デフォルト値式は常に同期的に評価される。そのため、式の評価中に行われる呼び出しもすべて同期的でなければならない。呼び出された側が分離されている場合、デフォルト値式が同期的に呼び出されるためには、既に同じ分離ドメイン内になければならない。そのため、与えられたデフォルト値式に対して、推論される分離はその部分式に必要な分離となります。例えば

```swift
@MainActor func requiresMainActor() -> Int { ... }

@MainActor func useDefault(value: Int = requiresMainActor()) { ... }
```

上記のコードでは、デフォルト値は`@MainActor`に分離された`requiresMainActor()`を呼び出すため、`value`のデフォルト引数は`@MainActor`の分離を必要とする。

#### クロージャ

クロージャのアクター分離はクロージャを呼び出す時にのみ適用される。アクター分離されたクロージャは、クロージャ本文がその分離ドメイン内で同期的に呼び出しを行うことを可能にする。アクター分離が明示的にアノテートされていないデフォルト値式のクロージャリテラルの場合、推論されるクロージャの分離は、同期呼び出しするクロージャ本文の全ての呼び出された側の分離を統合したものになります。例えば:

```swift
@MainActor func requiresMainActor() -> Int { ... }

@MainActor func useDefaultClosure(
  closure: () -> Void = {
    requiresMainActor()
  }
) {}
```

上記の `useDefaultClosure` 関数のデフォルト引数はクロージャリテラルである。クロージャ本文は `@MainActor` で分離された関数を同期的に呼び出すので、クロージャ自体も `@MainActor` で分離されていなければならない。

デフォルト引数のクロージャリテラルをアクターインスタンスに分離する唯一の方法は、分離パラメータで明示的に分離を記述することであることに注目してほしい。推論アルゴリズムは、以下の2つの特性によって分離をアクターインスタンスと判断することはありません：

1. アクターインスタンスに分離するためには、クロージャはそれ自身の(明示的な)分離されたパラメータを持つか、そのクロージャを囲むコンテキストから分離されたパラメータを取り込まなければならない。
2. デフォルト引数のクロージャリテラルは値をキャプチャできない。

#### 制限

* 関数や型自体がアクターに分離されている場合、そのデフォルト値式に必要な分離は、同じアクター分離を共有しなければならない。例えば、`@MainActor`に分離された関数は `@AnotherActor` に分離されたデフォルト引数を持つことはできない。分離されたデフォルト値と`nonisolated`なデフォルト値を混ぜても常に問題ないことに注意してほしい
* 関数や型が `nonisolated` である場合、デフォルト値式の分離は `nonisolated` でなければならない

### デフォルト値の分離要件の実施

#### デフォルト引数値

デフォルト引数式の分離要件は呼び出し側で強制される。呼び出し側が必要な分離領域にない場合、デフォルト引数は非同期に評価され、明示的に `await` でマークされなければならない。例えば:

```swift
@MainActor func requiresMainActor() -> Int { ... }

@MainActor func useDefault(value: Int = requiresMainActor()) { ... }

@MainActor func mainActorCaller() {
  useDefault() // okay
}

func nonisolatedCaller() async {
  await useDefault() // okay

  useDefault() // error: `await`をつけなければならない
}
```

上記の例では、`useDefault` は `@MainActor` に分離されたデフォルト引数を持っている。デフォルト引数は `@MainActor` に分離された呼び出し側から同期的に評価することができるが、`@MainActor` の外から呼び出す場合は `await` を付けなければならない。これらのルールは、アクター分離関数を呼び出すセマンティクスから既に外れていることに注意して欲しい。

#### 引数の評価

ある呼び出しに対して、引数の評価は以下の順序で行われる:

1. 明示的なR値引数の左から右への評価
2. デフォルト引数および公式アクセス引数(get)の左から右への評価

例えば:

```swift
nonisolated var defaultVal: Int { print("defaultVal"); return 0 }
nonisolated var explicitVal: Int { print("explicitVal"); return 0 }
nonisolated var explicitFormalVal: Int {
  get { print("explicitFormalVal"); return 0 }
  set {}
}

func evaluate(x: Int = defaultVal, y: Int = defaultVal, z: inout Int) {}

evaluate(y: explicitVal, z: &explicitFormalVal)
```

出力結果は

```
explicitVal
defaultVal
explicitFormalVal
```

明示的な引数リストとは異なり、分離されたデフォルト引数は、呼び出された側の分離ドメインで評価されなければならない。そのため、引数の値のいずれかが呼び出された側の分離を必要とする場合、引数の評価は以下の順序で行わる:

1. 明示的なR値引数の左から右への評価
2. 公式アクセス引数の左から右への評価
3. 呼び出された側の分離領域へのhop
4. デフォルト引数の左から右への評価

例えば

```swift
@MainActor var defaultVal: Int { print("defaultVal"); return 0 }
nonisolated var explicitVal: Int { print("explicitVal"); return 0 }
nonisolated var explicitFormalVal: Int {
  get { print("explicitFormalVal"); return 0 }
  set {}
}

@MainActor func evaluate(x: Int = defaultVal, y: Int = defaultVal, z: inout Int) {}

nonisolated func nonisolatedCaller() {
  await evaluate(y: explicitVal, z: &explicitFormalVal)
}
```

`nonisolatedCaller()`の呼び出し結果は、

```
explicitVal
explicitFormalVal
defaultVal
```

#### 格納プロパティの初期値

格納プロパティのデフォルトのイニシャライザ式に対する分離要件は、イニシャライザ本文に適用される。もし `init` がイニシャライザ式の分離と一致しない場合、その格納プロパティの初期化は `init` の先頭では行われない。代わりに、格納プロパティは `init` の本文で明示的に初期化する必要がある。例えば:

```swift
@MainActor func requiresMainActor() -> Int { ... }
@AnotherActor func requiresAnotherActor() -> Int { ... }

class C {
  @MainActor var x1: Int = requiresMainActor()
  @AnotherActor var x2: Int = requiresAnotherActor()

  nonisolated init() {} // error: 'self.x1' and 'self.x2' aren't initialized

  nonisolated init(x1: Int, x2: Int) { // okay
    self.x1 = x1
    self.x2 = x2
  }

  @MainActor init(x2: Int) { // okay
    // 'self.x1' はデフォルト値の'requiresMainActor()'から取得する
    self.x2 = x2
  }
}
```

上の例では、パラメータを指定しない `nonisolated init()` は `self.x1` と `self.x2` を初期化しないので無効である。デフォルトのイニシャライザ式は異なるアクタの分離を必要とするので、これらの値は `nonisolated` イニシャライザでは使用できない。他の2つのイニシャライザは有効である。

### イニシャライザにおける格納プロパティの分離

#### 分離された格納プロパティを分離の境界を越えて初期化する

分離された格納プロパティを分離の境界を越えて初期化することは無効である:

```swift
class NonSendable {}

class C {
  @MainActor var ns: NonSendable

  init(ns: NonSendable) {
    self.ns = ns // error: passing non-Sendable value 'ns' to a MainActor-isolated context.
  }
}
```

分離されていないコンテキストから `MainActor`に分離されたプロパティ `self.ns` を初期化すると、分離の境界を越えて `Sendable` でない値を渡すことになるので、上記のコードは `Sendable` 保証に違反します。このようなデータ競合を防ぐために、このプロポーザルでは、グローバルアクターから分離された格納プロパティを初期化する `init` も、そのグローバルアクターから分離されている必要がある。

このルールはデフォルト値に特化したものではないが、コンパイラが合成したイニシャライザにおけるデフォルト値の振る舞いを指定するために必要であることに注目してほしい。

#### コンパイラが合成したイニシャライザにおけるデフォルト値の分離

structの場合、格納プロパティのデフォルトのイニシャライザ式は、コンパイラが合成するメンバワイズイニシャライザのデフォルトの引数値として使用される。デフォルトのイニシャライザ式はまた、コンパイラが生成する引数なしイニシャライザを持つstructやclassの合成された `init()` 本文でも使用される。

型の格納プロパティで `Sendable` でないものがアクターから分離されている場合、または分離されているデフォルトのイニシャライザ式がアクターから分離されている場合、コンパイラが合成するイニシャライザもアクターから分離されていなければならなし。例えば:

```swift
class NonSendable {}

@MainActor struct MyModel {
  // structのアノテーションから@MainActorと推論される
  var value: NonSendable = .init()

  /* コンパイラが合成するメンバワイズイニシャライザは@MainActorである
  @MainActor
  init(value: NonSendable = .init()) {
    self.value = value
  }
  */
}
```

型の格納されているプロパティがどれも `Sendable` でなく、アクターに分離されておらず、デフォルトのイニシャライザ式もすべてアクターの分離を必要としない場合、コンパイラが合成したイニシャライザは `nonisolated` となる。例えば:

```swift
@MainActor struct MyView {
  // structのアノテーションから@MainActorと推論される
  var value: Int = 0

  /* コンパイラが合成するイニシャライザは'nonisolated'

  nonisolated init() {
    self.value = 0
  }

  nonisolated init(value: Int = 0) {
    self.value = value
  }
  */

  // structのアノテーションから@MainActorと推論される
  var body: some View { ... }
}
```

これらのルールにより、コンパイラが合成したイニシャライザ内のデフォルト値式が常に有効であることが保証される。デフォルト値式がアクター分離を必要とする場合、それを囲むイニシャライザは常に同じアクター分離を共有します。2つのデフォルト値式が異なるアクター分離を必要とする場合はエラーとなる。異なるアクターにされた分離2つの異なるデフォルト値式を使用して型のインスタンスを初期化する場合は、`await`を明示的にマークして `async` イニシャライザで行う必要がある。

## ソース互換性

格納プロパティの初期値に対するアクターの分離ルールは、データ競合を排除するために、Swift 5 モードで現在受け入れられているものよりも厳しくなる。格納プロパティの分離ルールは、`IsolatedDefaultValues` upcoming featureフラグの下でステージングされる。

### ABI互換性

影響なし

## 検討された代替案

### 全てのデフォルト初期化式から分離を取り除く

SE-0327 は当初、格納プロパティのデフォルトのイニシャライザ式を常に `nonisolated` に変更することを提案していた。しかし、この変更は `@MainActor` で分離された型が、他の `@MainActor` で分離された型のイニシャライザを呼び出すデフォルト値を持つ格納プロパティを持つというよくあるパターンに従った多くのコードに影響を与えたため、実装した後にリバートされた。場合によっては、`@MainActor`型のイニシャライザを`nonisolated`にすることも可能だが、そのような場合の多くは、イニシャライザの本文で`@MainActor`に分離された型のプロパティや関数にアクセスしてしまう。

## 参考リンク

### Forums

- [[Pitch] Isolated default value expressions](https://forums.swift.org/t/pitch-isolated-default-value-expressions/67714)
- [SE-0411: Isolated default value expressions](https://forums.swift.org/t/se-0411/68065)
- [[Accepted] SE-0411: Isolated Default Value Expressions](https://forums.swift.org/t/accepted-se-0411-isolated-default-value-expressions/68806)

### プロポーザルドキュメント

- [Isolated default value expressions](https://github.com/apple/swift-evolution/blob/main/proposals/0411-isolated-default-values.md)
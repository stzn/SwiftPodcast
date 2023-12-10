# Swift Typed throws

- [Swift Typed throws](#swift-typed-throws)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
      - [`Result`や`Task`よりも少ないエラー情報を伝える](#resultやtaskよりも少ないエラー情報を伝える)
      - [`throws`と`Result`または`Task`は相互変換が不可能](#throwsとresultまたはtaskは相互変換が不可能)
      - [`Result`は命令型言語の`throws`の代替にはならない](#resultは命令型言語のthrowsの代替にはならない)
        - [アプローチ1: 結果の連結](#アプローチ1-結果の連結)
        - [アプローチ2： すべてのチェーン/マッピングポイントでアンラップ／switch／ラップする](#アプローチ2-すべてのチェーンマッピングポイントでアンラップswitchラップする)
      - [存在型のエラーにはオーバーヘッドが発生する](#存在型のエラーにはオーバーヘッドが発生する)
    - [提案内容](#提案内容)
      - [catchブロック内の特定の型](#catchブロック内の特定の型)
      - [`any Error` または `Never`をスローする](#any-error-または-neverをスローする)
      - [`rethrows`の代わり](#rethrowsの代わり)
      - [型付きエラーはいつ使うべきか](#型付きエラーはいつ使うべきか)
    - [詳細](#詳細)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

Swift のエラー処理モデルは、エラーをスローすることによって関数とクロージャから抜け出せることに注目してもらうために `throws` とマークできる。エラーの値自体は、常に任意のエラーに型消去される。このアプローチは、エラーを汎用的に処理することを奨励し、ほとんどのコードにとっては十分なデフォルトのままである。しかし、すべてのエラーを処理することが可能または望ましかったり、型消去のコストが法外であるような限定的な場所では、より正確なエラーの型付けができないためこの型消去の挙動が相応しくないところがある。

このプロポーザルでは、関数やクロージャが特定の具象型のエラーだけを投げるように指定する機能を導入する。

## 内容

### 動機

Swiftは、セマンティクスについて明示的であることと、特定のAPIに適用される制約を伝えるために型を使用することで知られている。その観点から、すべてのスローされたエラーが任意のエラー型であるという事実は、異常なように感じる。しかし、これは、エラーは汎用的　に伝播してレンダリングされるが、網羅的に処理されることはまれであり、違う型に時間の経過とともに変化しやすいという、元のエラー処理の[理論的根拠](https://github.com/apple/swift/blob/main/docs/ErrorHandlingRationale.md)で示された見解を反映している。
特定のスローされたエラー型をスローしたいという要望は、Swift のフォーラムで繰り返し出てきた。以下は、型付きスローのいくつかの形を要求するフォーラムのスレッドのほんの一部である:

[[Pitch N+1] Typed throws]()
[Typed throw functions]()
[Status check: typed throws]()
Precise error typing in Swift
Typed throws
[Pitch] Typed throws
Type-annotated throws
Proposal: Allow Type Annotations on Throws
Proposal: Allow Type Annotations on Throws
Proposal: Typed throws
Type Inferencing For Error Handling (try catch blocks)

ある意味で、Swiftは、その`Failure`パラメータで特定のスローされたエラー型をキャプチャする標準ライブラリの`Result`型の導入で、型付きスローへの道を歩み始めた。そのパターンは、`Task`型と他のConcurrency APIでも模倣された。`Result`や`Task`のような型と言語のエラー処理システムの間の情報の損失は、型付きスローの導入の部分的な動機となります。

型付きスローは、クライアントがエラーを網羅的に処理する必要がある場合にも有効である。これは、クライアントと同じモジュールやパッケージから来るか、事実上スタンドアロンであり、(例えば)他の低レベルライブラリからのエラーを通過するように変わる可能性が低いライブラリから来るためである。また、型付きスローは既存の`rethrows`に代わるより柔軟な選択肢として、引数からエラーを伝播させるが、それ自身はエラーを生成しないジェネリックコードにメリットをもたらす。最後に、型付きスローは、実存型(`any Error``)に関連するオーバーヘッドを回避するため、より効率的なコードが書ける可能性をもたらす。

型付きスローがSwiftに導入されたとしても、既存の(型付けされていない)`throws`は、ほとんどのSwiftコードのためのより良いデフォルトのエラー処理メカニズムのままである。["型付きスローを使用するタイミング"](https://github.com/apple/swift-evolution/blob/main/proposals/0413-typed-throws.md#when-to-use-typed-throws)のセクションでは、型付きスローが使用されるべき状況について説明する。

#### `Result`や`Task`よりも少ないエラー情報を伝える

下記のエラー型があると想定して

```swift
enum CatError: Error {
    case sleeps
    case sitsAtATree
}
```

```swift
func callCatOrThrow() throws -> Cat
```
を
```swift
func callCat() -> Result<Cat, CatError>
```
や
```swift
func callFutureCat() -> Task<Cat, CatError>
```
と比較すると、`throws`はなぜ猫があなたのところに来ないのかの情報を伝えられない。

#### `throws`と`Result`または`Task`は相互変換が不可能

`throws`が`Result`や`Task`よりも少ない情報しか持たないという事実は、`throws`への変換が型情報を失うことを意味し、これは明示的なキャストによってのみ回復できる:

```swift
func callAndFeedCat1() -> Result<Cat, CatError> {
    do {
        return Result.success(try callCatOrThrow())
    } catch {
        // そもそもエラーの型情報がなくなっているためコンパイルできない
        return Result.failure(error)
    }
}
```

```swift
func callAndFeedCat2() -> Result<Cat, CatError> {
    do {
        return Result.success(try callCatOrThrow())
    } catch let error as CatError {
        // コンパイルできる
        return Result.failure(error)
    } catch {
        // コンパイラが網羅性を検証できないためコンパイルできない。
        // ここでは何を返すべきだろうか？
        return Result.failure(error)
    }
}
```

#### `Result`は命令型言語の`throws`の代替にはならない

`Result`で明示的なエラーを使用することは、コードベースに大きな影響を与える。例外処理メカニズム("goto catch")が(`throws`のように)言語に組み込まれていないため、例外処理メカニズムをドメインロジックと混ぜ合わせながら、自分でそれを行う必要がある。

##### アプローチ1: 結果の連結

関数型(つまりモナド)の方法で`Result`を使用する場合、`map`、`flatMap`および同様の演算子を広範囲に使用する必要がある。

例は、[Question/Idea: Improving explicit error handling in Swift (with enum operations) - Using Swift - Swift Forums.](https://forums.swift.org/t/question-idea-improving-explicit-error-handling-in-swift-with-enum-operations/35335)から引用している。

```swift
struct SimpleError: Error {
    let message: String
}

struct User {
    let firstName: String
    let lastName: String
}

func stringResultFromArray(_ array: [String], at index: Int, errorMessage: String) -> Result<String, SimpleError> {
    guard array.indices.contains(index) else { return Result.failure(SimpleError(message: errorMessage)) }
    return Result.success(array[index])
}

func userResultFromStrings(strings: [String]) -> Result<User, SimpleError>  {
    return stringResultFromArray(strings, at: 0, errorMessage: "Missing first name")
        .flatMap { firstName in
            stringResultFromArray(strings, at: 1, errorMessage: "Missing last name")
                .flatMap { lastName in
                    return Result.success(User(firstName: firstName, lastName: lastName))
            }
    }
}
```

これが例外を書く関数型の方法だが、Swiftはそれを快適に扱うのに十分な関数的な構成要素を提供していない([Haskellのdo記法](https://en.wikibooks.org/wiki/Haskell/do_notation)と比較して)。

##### アプローチ2： すべてのチェーン/マッピングポイントでアンラップ／switch／ラップする

すべての結果を`switch`してアンラップし、値やエラーを再び`Result`にラップすることもできる。

```swift
func userResultFromStrings(strings: [String]) -> Result<User, SimpleError>  {
    let firstNameResult = stringResultFromArray(strings, at: 0, errorMessage: "Missing first name")
    
    switch firstNameResult {
    case .success(let firstName):
        let lastNameResult = stringResultFromArray(strings, at: 1, errorMessage: "Missing last name")
        
        switch lastNameResult {
        case .success(let lastName):
            return Result.success(User(firstName: firstName, lastName: lastName))
        case .failure(let simpleError):
            return Result.failure(simpleError)
        }
        
    case .failure(let simpleError):
        return Result.failure(simpleError)
    }
}
```

`flatMap`演算子の実装を何度も書くことになるので、最初のアプローチよりもさらにボイラープレートだ。

#### 存在型のエラーにはオーバーヘッドが発生する

型なしスローは、未知の型の値をサポートする必要があるため、コードサイズ、ヒープ割り当て、実行パフォーマンスにおいて、[いくつかの必要なオーバーヘッド](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md)を発生させる`any Error`という存在型を持っている。[Embedded Swift](https://forums.swift.org/t/embedded-swift/67057)でサポートされるような制約のある環境では、これらのオーバーヘッドのために、既存の型なしスローメカニズムがこれらの環境で使用できないように、存在型は使えないかもしれない。

### 提案内容

汎用的に、単一の特定のエラー型をスローできるようにする。

```swift
func callCat() throws(CatError) -> Cat {
  if Int.random(in: 0..<24) < 20 {
    throw .sleeps
  }
  // ...
}
```

この関数は`CatError`のインスタンスのみをスローできる。これは、すべてのエラーをスローする側へ文脈に沿った型情報を提供するので、型なしスローでは必要だったより冗長な`CatError.sleeps`の代わりに``.sleeps`と書くことができる。関数から他の種類のエラーを投げようとするとエラーになる:

```swift
func callCatBadly() throws(CatError) -> Cat {
  throw SimpleError(message: "sleeping")  // error: SimpleError cannot be converted to CatError
}
```

関数全体を通して特定のエラー型を維持することは、一貫して`try`を使うことができるため、`Result`を使った場合よりもはるかに簡単になる。

```swift
func stringFromArray(_ array: [String], at index: Int, errorMessage: String) throws(SimpleError) -> String {
    guard array.indices.contains(index) else { throw SimpleError(message: errorMessage) }
    return array[index]
}

func userResultFromStrings(strings: [String]) throws(SimpleError) -> User  {
    let firstName = try stringFromArray(strings, at: 0, errorMessage: "Missing first name")
    let lastName = try stringFromArray(strings, at: 1, errorMessage: "Missing last name")
    return User(firstName: firstName, lastName: lastName)
}
```

エラー処理のメカニズムを見せなくし、ドメインロジックがより明確に見えるようになる。

#### catchブロック内の特定の型

型付きスローでは、スローする関数に`Result`と同じエラー型の情報が含まれるため、両者の変換が容易になる:

```swift
func callAndFeedCat1() -> Result<Cat, CatError> {
    do {
        return Result.success(try callCat())
    } catch {
        // エラーは`CatError`なので、今はコンパイルできる
        return Result.failure(error)
    }
}
```

`catch`ブロック内の暗黙の`error`変数は、具象型`CatError`に推論されることに注目してください。

`do`文が異なる具象型のエラーを投げる可能性がある場合、または型なしスローを使用する関数への呼び出しが含まれる場合、`catch`ブロックは`any Error`型を受け取る:

```swift
func callKids() throws(KidError) -> [Kid] { ... }

do {
  try callCat()
  try callKids()
} catch {
  // errorの型は今と同じように'any Error'型となる
}
```

`do..catch`宣言でキャッチされるエラー型は、`do`ブロックの本文内のさまざまなスローする側から推論される。この型は、`do`ブロック自体の`throws`句で明示的に指定することができる:

```swift
do throws(CatError) {
  if isDaylight && foodBowl.isEmpty {
    throw .sleeps   // CatError.sleepsと同等
  }
  try callCat()
} catch let myError {
   // myErrorはCatError型
}
```

ある具象型のエラーを別の具象型に変換する必要がある場合は、同じ種類のエラーを発生させる呼び出しのシーケンスごとに、`do...catch`ブロックを使用する:

```swift
func firstNameResultFromArray(_ array: [String]) throws(FirstNameError) -> String {
    guard array.indices.contains(0) else { throw FirstNameError() }
    return array[0]
}

func userResultFromStrings(strings: [String]) throws(SimpleError) -> User  {
    do {
        let firstName = try stringFromArray(strings, at: 0, errorMessage: "Missing first name")
        return User(firstName: firstName, lastName: "")        
    } catch {
        // errorは`FirstNameError`なので`SimpleError`にマッピングしている
        throw SimpleError(message: "First name is missing")
    }
}
```

#### `any Error` または `Never`をスローする

型付きスローは型なしスローとスローしない関数の両方を一般化する。`any Error`をスローする型として指定した関数:
```swift
func throwsAnything() throws(any Error) { ... }
```
は型なしスロー関数の

```swift
func throwsAnything() throws { ... }
```
と等しい。

同様に、`Never`をスロー型として指定した関数:

```swift
func throwsNothing() throws(Never) { ... }
```
は
```swift
func throwsNothing() { }
```
と等しい。

つまり、スローしない関数をスローする関数に変換したり、具象型をスローする関数を`any Error`をスローする関数に変換したりすることができる。

#### `rethrows`の代わり

`Never`かもしれない汎用的なエラーパラメータをスローする能力は、`rethrows`では不可能ないくつかのパターンを安全に表現可能にする。例えば、意味合い上`rethrow`するがエラーをスローしない関数を考えてみよう:

```swift
/// ツリー内で特定の`predicate`にマッチするノードの数を数える
func countNodes(in tree: Node, matching predicate: (Node) throws -> Bool) rethrows -> Int {
  class MyNodeVisitor: NodeVisitor {
    var error: (any Error)? = nil
    var count: Int = 0
    var predicate: (Node) throws -> Bool

    init(predicate: @escaping (Node) throws -> Bool) {
      self.predicate = predicate
    }
    
    override func visit(node: Node) {
    	do {
        if try predicate(node) {
          count = count + 1
        }
      } catch let localError {
        error = error ?? localError
      } 
    }
  }
  
  return try withoutActuallyEscaping(predicate) { predicate in
    let visitor = MyNodeVisitor(predicate: predicate)
    visitor.visitTree(node)
    if let error = visitor.error {
      throw error // error: これは`predicate`の結果のスローではない
    } else {
      return visitor.count
    }
  }
}
```

コードを見ると見渡してみると、`MyNodeVisitor.error`は`predicate`がエラーをスローした結果としてのみ設定されるので、このコードは意味的にはrethrowsのルールを満たします。しかし、Swiftコンパイラの`rethrows`チェックは、そのような分析を行うことができないので、この関数はコンパイルエラーになる。`rethrows`の制限は、少なくとも2つのピッチで"unsafe"または"unchecked"の`rethrows`のバリエーションを追加し、これを実行時にチェックするルールに変えていた。

型付きスローは納得のいく代替案を提供する。つまり、クロージャ引数のエラー型をジェネリックパラメータに取り込み、それを一貫して使用できる。これは、`map`のようなクロージャ引数からエラーを`rethrows`するジェネリックコードで正確な型付きエラー情報を保持するのに便利である。

```swift
extension Collection {
  func map<U, E: Error>(body: (Element) throws(E) -> U) throws(E) -> [U] {
    var result: [U] = []
    for element in self {
      result.append(try body(element))
    }
    return result
  }
}
```

`CatError`を投げるクロージャが与えられると、この形の`map`は`CatError`をスローする。スローしないクロージャが与えられると、`E`は`Never`になるので、`map`はエラーをスローしない。

このアプローチは`countNodes`の例にも適用できる。

```swift

/// ツリー内で特定の`predicate`にマッチするノードの数を数える
func countNodes<E: Error>(in tree: Node, matching predicate: (Node) throws(E) -> Bool) throws(E) -> Int {
  class MyNodeVisitor<E>: NodeVisitor {
    var error: E? = nil
    var count: Int = 0
    var predicate: (Node) throws(E) -> Bool

    init(predicate: @escaping (Node) throws(E) -> Bool) {
      self.predicate = predicate
    }
    
    override func visit(node: Node) {
    	do {
        if try predicate(node) {
          count = count + 1
        }
      } catch let localError {
        error = error ?? localError // `error`は`E?`型、`localError`は`E`型なのでOK 
      } 
    }
  }
  
  return try withoutActuallyEscaping(predicate) { predicate in
    let visitor = MyNodeVisitor(predicate: predicate)
    visitor.visitTree(node)
    if let error = visitor.error {
      throw error // errorの型はEで、この関数の外でスローされる可能性のあるエラー型はEなのでOK！
    } else {
      return visitor.count
    }
  }
}
```

型付きスローは我々の問題をエレガントに解決していることに注目してほしい。というのも、`E`型の値をスローする側なら何でも受け入れられるからである。クロージャ引数がスローしない場合、`E`は`Never`と推論され、(動的に)そのインスタンスは生成されない。

#### 型付きエラーはいつ使うべきか

型付きスローは、関数がスローするエラー型を厳密に指定することができるが、それによってその関数の実装の変更を制約してしまう。さらに、エラーは、通常は伝播されるかレンダリングされるが、網羅的に処理されることはないので、Swiftに型付きスローが追加されたとしても、型なしスローの方がほとんどの場面でよい選択である。以下の状況でのみ型付きスローを考慮すること:

1. 常にエラーを処理したいモジュールやパッケージ内のコードにとどまる範囲にあれば、それは純粋に実装の詳細であり、そのエラーを処理するのは妥当である
2. 独自のエラーをスローせず、ユーザコンポーネントからのエラーを通過させるだけのジェネリックコード。標準ライブラリには、`map`のような`rethrows`関数であれ、`Task`や`Result`のような`Failure`型を捕捉する場合であれ、このようなが多数含まれている
3. 制約のある環境(例えば、Embedded Swift)で使用されることを意図している依存関係のないコードや、メモリを割り当てることができないコードでは、それ自身のエラーしかスローしない

実装がスローできるエラーの型が唯一のものだったとしても、型付きスローを使用する誘惑に抵抗してください。例えば、指定されたファイルからバイトを読み込むする操作を考えてみよう:

```swift
public func loadBytes(from file: String) async throws(FileSystemError) -> [UInt8]  // 型なしスローを使うべき
```

内部的には、`FileSystemError`をスローするファイルシステムライブラリを使用しており、それをそのまま再発行している。しかし、エラーが常に`FileSystemError`になるように指定しているということは、このAPIの今後の変化を妨げるかもしれない。例えば、ファイル名が他のスキーマにマッチする場合、このAPIが他のソース(例えば、ネットワーク接続やデータベース)からのバイトの読み込みをサポートし始めるかもしれない。しかし、それらの他のライブラリからのエラーは`FileSystemError`インスタンスではないので、`loadBytes(from:)`で問題となる。他のライブラリからのエラーを`FileSystemError`に変換する必要があるか(それが可能であれば)、より一般的なエラー型(または型なしスロー)にしてAPIの元々の契約を破棄する必要がある。

このセクションは将来的に[API設計ガイドライン](https://www.swift.org/documentation/api-design-guidelines/)に追加される。

### 詳細

TBD

## 参考リンク

### Forums

- [SE-0413: Typed throws](https://forums.swift.org/t/se-0413-typed-throws/68507)

### プロポーザルドキュメント

- [0413-typed-throws](https://github.com/apple/swift-evolution/blob/main/proposals/0413-typed-throws.md)
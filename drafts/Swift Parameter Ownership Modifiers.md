# `borrowing`と`consuming`パラメータ所有権修飾子

- [`borrowing`と`consuming`パラメータ所有権修飾子](#borrowingとconsumingパラメータ所有権修飾子)
  - [概要](#概要)
  - [内容](#内容)
  - [詳細](#詳細)
  - [ソース互換性](#ソース互換性)
  - [ABIの安定性への影響](#abiの安定性への影響)
  - [APIレジリエンスへの影響](#apiレジリエンスへの影響)
  - [代替案](#代替案)
  - [関連する方向性](#関連する方向性)
    - [暗黙のコピーに対する呼び出し側の制御](#暗黙のコピーに対する呼び出し側の制御)
      - [`consume`演算子](#consume演算子)
    - [`borrow`演算子](#borrow演算子)
    - [move-only型、コピー不可の値、および関連する機能](#move-only型コピー不可の値および関連する機能)
    - [`set`/`out`パラメータ規約](#setoutパラメータ規約)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

開発者が、関数が不変パラメータを受け取る際に使用する所有権の規約を明示的に選択できるようにするため、新しいパラメータ`borrowing`と`consuming`修飾子を提案する。これにより、関数の呼び出しに必要なARC呼び出しやコピーの回数を減らすことでパフォーマンスの微調整が可能になり、関数がmove-onlyの値を消費するかどうかを指定するmove-only型に必要な前提機能を提供する。

## 内容

Swift は、参照がカウントされるオブジェクトのライフタイムを管理するために、自動参照カウント(ARC)を使用する。関数呼び出しで呼び出し元から呼び出し先に値でオブジェクトを渡すときに、コンパイラがメモリの安全性を維持するために使用する2つの一般的な規約がある:

呼び出し先はパラメータを**借りることができる**。呼び出し元は、その引数オブジェクトが呼び出し中は生き続けることを保証し、呼び出し先がそれを解放する必要はない(ただし、それ自身が実行する追加的なretainのバランスをとることを除く)。
呼び出し先はパラメータを**消費できる**。呼び出し元は、パラメータを解放するか、その所有権を他の場所に渡すか、その責任を負うことになる。呼び出し元が引数の所有権を放棄したくない場合は、引数を保持し、呼び出し元が余分な参照カウントを消費できるようにする必要がある。
これらの2つの規約は値型として一般化される。つまり「retain」が値の独立したコピーになり、「release」がコピーの破壊とメモリ割り当て解除になる。イニシャライザやプロパティセッタは、別の値を構築または更新するためにそれらのパラメータを使用する可能性が高いので、それらのパラメータを消費したほうが、構築した新しい値に所有権を転送するため、より効率的であると思われる: デフォルトでは、Swiftのコードの典型的な動作として知られたいくつかのルールに基づいて使用する規約を選択する。その他の関数は、デフォルトでパラメータを借りるようになっているが、これは、ほとんどの場面でこの方が効率的であることが分かっているからである。

これらの選択は一般的にうまくいくが、常に最適というわけではない。オプティマイザは「関数シグネチャ最適化」をサポートしており、ARCのトラフィック全体を削減できる可能性があると判断した場合、関数が使用する規約を変更することができるが、これを自動化できる状況は限定されている。所有権の規約はパブリックAPIのABIの一部となるため、ABI安定化ライブラリで一度確立されると変更することができない。また、オプティマイザは、non-finalクラスメソッドやプロトコル要件などのポリモーフィックインターフェースを最適化しようとはしない。このような状況でプログラマがデフォルトと異なる動作を望む場合、現在のところその方法はない。

将来的には、Swiftに[所有権を追加するための進行中のプロジェクト](https://forums.swift.org/t/manifesto-ownership/5212)の一環として、最終的にmove-onlyの値や型を持つことになる。move-only型はコピーされる能力を持っていないので、2つの規約の間の区別は、API契約の重要な部分になる: move-onlyの値を借りる関数は、値を一時的に使用し、ファイルハンドルから読み取るように、それ以降の使用のために有効なままにしておくのに対し、move-onlyの値を消費する関数は、ファイルハンドルを閉じるように、それを消費してそれ以降の使用を防止している。これらのタイプでは、パラメータ規約の暗黙の選択に頼るだけでは十分ではない。

##　提案内容

パラメータの所有権の規約を開発者が直接コントロールできるように、2つの新しいパラメータ修飾子`borrowing`と`consuming`を導入する。

## 詳細

`borrowing`と`consuming`は、パラメータ型宣言の内部でコンテキストキーワードとなる。これらは、`inout`修飾子と同じ場所で使用することができ、また互いに`inout`とは排他的である。`func`、`subscript`、`init`宣言では、以下のように表示される:

```swift
func foo(_: borrowing Foo)
func foo(_: consuming Foo)
func foo(_: inout Foo)
```
クロージャの中で

```swift
bar { (a: borrowing Foo) in a.foo() }
bar { (a: consuming Foo) in a.foo() }
bar { (a: inout Foo) in a.foo() }
```

関数型では

```swift
let f: (borrowing Foo) -> Void = { a in a.foo() }
let f: (consuming Foo) -> Void = { a in a.foo() }
let f: (inout Foo) -> Void = { a in a.foo() }
```

また、メソッドは`consuming`または`borrowing`修飾子を使用して、`self`パラメータの所有権を消費すること、または借りることをそれぞれ示すことができる。これらの修飾子は、互いに、また既存の `mutating` 修飾子とも排他的である：

```swift
struct Foo {
  consuming func foo() // `consuming` self
  borrowing  func foo() // `borrowing ` self
  mutating func foo() // `inout` セマンティクスで self を変更する。
}
```

`consuming`は、非エスケープ型クロージャのパラメータに適用することはできず、その性質上、常に貸し出される:

```swift
// ERROR: 非エスケープ化クロージャを `consume` することはできません。
func foo(f: consuming () -> ()) {
}
```

`consuming`または`borrowing`パラメータでは、影響を受ける宣言に引数を渡すための呼び出し元の構文に影響を与えず、`consuming`または`borrowing`は、メソッド呼び出しの`self`の適用に影響を与えません。典型的なSwiftのコードでは、これらの修飾子を追加、削除、または変更しても、ソースを破壊するような影響はない。(ソースを壊す原因となる方法でこれらの修飾子と相互作用するかもしれない、現在または近い将来に検討されている他の言語機能との相互作用については、以下の「関連する方向性」を参照)。

プロトコル要件は、`consuming` と `borrowing` を使用することもでき、修飾子は、ジェネリックインターフェースが要件を呼び出すために使用する規約に影響を与える。コピー可能な型のパラメータに異なる規約を使用する実装でも、要件は満たされる場合もある:

```swift
protocol P {
  func foo(x: consuming Foo, y: borrowing Foo)
}

// 以下は妥当な適合である:

struct A: P {
  func foo(x: Foo, y: Foo)
}

struct B: P {
  func foo(x: borrowing Foo, y: consuming Foo)
}

struct C: P {
  func foo(x: consuming Foo, y: borrowing Foo)
}
```

また、関数値を暗黙のうちに関数型に変換することで、コピー可能な型のパラメータの規約を不特定、`borrowing`、`consuming`の間で変更することができる:

```swift
let f = { (a: Foo) in print(a) }

let g: (borrowing Foo) -> Void = f
let h: (consuming Foo) -> Void = f

let f2: (Foo) -> Void = h
```

関数やクロージャのボディ内部では、`consuming`パラメータを変更することができ、`consuming func`メソッドの`self`パラメータも変更することができる。これらの変更は、関数自身が所有権を持つ値に対して行われ、呼び出し元にまだ存在する可能性のある値のコピーには影響を与えない。このため、所有権移転後の値の一意性を利用して、値の効率的なローカルの変更を行うことが容易になる:

```swift
extension String {
  // 可能ならば`self`を別のStringに追加する。
  consuming func plus(_ other: String) -> String {
    // 所有する `self` のコピーを、可能であれば一意性を利用してインプレースで変更する。
    self += other
    return self
  }
}

// これは O(n^2) ではなく O(n) で償却される！
let helloWorld = "hello ".plus("cruel ").plus("world")
```

## ソース互換性

現在の言語では、パラメータに`consuming`または`borrowing`を追加しても、ソースの互換性に影響を与えることはありません。呼び出し側は通常通り関数を呼び出し続けることができ、関数本体はすでにそうしているようにパラメータを使用することができます。パラメータに`consuming`または`borrowing`修飾子を持つメソッドは、異なる修飾子を持つプロトコル要件を満たすために依然として使用することができます。コンパイラは、期待される規約を維持するために、必要に応じて暗黙のコピーを導入します。これにより、API作成者は、`consuming`と`borrowing`のアノテーションを使用して、実装のコピー動作を微調整することができ、クライアントがアノテーション付きAPIを使用するために所有権を意識することを強制することはありません。ソースのみのパッケージは、クライアントを破壊することなく、時間の経過とともにコピー可能な型にこれらのアノテーションを追加、削除、または調整することができます。

これは、move-only型、「暗黙のコピーなし」の値やスコープ、式中の`consuming`演算子や`borrowing`演算子など、コンパイラが暗黙のうちに値をコピーする能力を制限する機能を導入すると変化します。パラメータ規約を変更すると、呼び出しを実行するためにコピーが必要になる場合があります。コピー不可能な値を`consuming`パラメータの引数として渡すと、その値のライフタイムが終了し、消費された後にその値を再び使用することはできない。

## ABIの安定性への影響

`consuming`または`borrowing`はABIレベルの呼び出し規約に影響し、ABI安定化ライブラリを破壊的に変更することなく変更することはできない(ただし、コピーが`memcpy`と同等で破壊が無意味な「トリビア型」については、`consuming`または`borrowing`もトリビア型のパラメータには実用的な影響を与えない)。

## APIレジリエンスへの影響

`consuming`または`borrowing`は、ABI安定化ライブラリのABIを破壊しますが、ソースレベルのAPIに最小限の影響を抑えることを意図している。コピー可能な型を使用する場合、APIにこれらのアノテーションを追加または変更しても、既存のクライアントには影響しないようにする必要がある。

## 代替案

TBD

## 関連する方向性

### 暗黙のコピーに対する呼び出し側の制御

パフォーマンスに敏感なコードが呼び出しの動作についてアサーションを行えるようにするために、我々が検討している呼び出し側演算子がいくつかある。これらは、`consuming`と`borrowing`パラメータ修飾子に密接に関連しているため、それらの名前を共有している。この一連の機能のより深い議論については、Swiftフォーラムの[暗黙のコピー動作の選択的制御](https://forums.swift.org/t/selective-control-of-implicit-copying-behavior-take-borrow-and-copy-operators-noimplicitcopy/60168)のスレッドも参照してください。

#### `consume`演算子

現在SE-0366としてレビュー中だが、変数のスコープが終了する前に、変数のライフタイムを明示的に終了させる演算子があると便利です。これにより、最適化や曖昧なARCオプティマイザのルールに依存することなく、コンパイラが最後に使用した時点で、確実に変数の値を破棄したり、所有権を移したりすることができます。変数のライフタイムが`consume`パラメータの引数で終了する場合、コピーなしで呼び出し先に所有権を移すことができる:

```swift
func consume(x: consuming Foo)

func produce() {
  let x = Foo()
  consume(x: consume x)
  doOtherStuffNotInvolvingX()
}
```

### `borrow`演算子

特に、グローバル変数や静的変数、escapingクロージャのキャプチャ、クラスの格納プロパティなど、共有可変状態を扱う場合、理論的にはコピーすることが可能なだけなのに、コンパイラがデフォルトでコピーしてしまうという状況が存在する。特に、グローバル変数やスタティック変数、escapingクロージャキャプチャ、クラスの格納プロパティなど、共有可変状態を扱う場合、コンパイラはこのようにして、可変の排他性の法則に抵触しないようにする。以下の例では、`callUseFoo()`がコピーを渡す代わりに`borrowing`によって`global`を`useFoo`に渡した場合、`useFoo`の内部で`global`が変更されると、動的排他性の失敗(排他性のチェックが無効の場合はUB)が発生する:

```swift
var global = Foo()

func useFoo(x: borrowing Foo) {
  //  ここでは `global`への排他的なアクセスが必要
  global = Foo()
}

func callUseFoo() {
  // callUseFooは`useFoo`がグローバルにアクセスするかどうか分からないので、
  // 必要以上に共有アクセスを課すのは避けるため、値のコピーを渡すことにする’。これは:
  useFoo(x: global)

  // のようにコンパイルする:

  /*
  let globalCopy = copy(global)
  useFoo(x: globalCopy)
  destroy(globalCopy)
   */
}
```

コンパイラは、共有可変状態への書き込みが干渉する可能性がないことを決定的に証明するのは難しいので、`useFoo`が証明されれば理論的には防御コピーを排除できるかもしれないが、実際にはそうすることはほとんどない。開発者は、プログラムが呼び出し中に同じオブジェクトやグローバル変数を変更しようとしないことを知っていて、このコピーを抑制したいと思うかもしれない。明示的な`borrowing`演算子を使えば、これができます:

```swift
var global = Foo()

func useFooWithoutTouchingGlobal(x: borrowing Foo) {
  /* global not used here */
}

func callUseFoo() {
  //  プログラマは `useFooWithoutTouchingGlobal`が
  // `global`に触れないことを知っているので、コピーせずに渡したい
  useFooWithoutTouchingGlobal(x: borrow global)
}
```

`useFooWithoutTouchingGlobal`が、呼び出し元が`global`を借りている間に実際に`global`を変更しようとした場合、排他性の失敗が発生する。

### move-only型、コピー不可の値、および関連する機能

暗黙のうちにコピーできない値では、`consuming`と`borrowing`の区別がより重要で顕著になります。今後は、値がコピーできないmove-only型や、特定の変数やスコープに対して選択的にコンパイラの暗黙のコピー動作を抑制する属性も導入する予定です。値を借用する操作では、同じ値を継続して使用することができますが、値を消費する操作では、値を破壊して継続して使用することができなくなります。このため、move-onlyのパラメータに使用される規約は、操作後も値が使用可能かどうかに直接影響するため、そのAPI契約の中でより重要な部分となる:

```swift
moveonly struct FileHandle { ... }

// ファイルハンドルをオープンする操作。新しいFileHandleの値を返す
func open(path: FilePath) throws -> FileHandle

// 開いているファイルハンドルに対して操作し、開いたままにする操作
// FileHandleを借用する
func read(from: borrowing FileHandle) throws -> Data

// ファイルハンドルを閉じて使用不可能にする操作。ファイルハンドルを消費する
func close(file: consuming FileHandle)

func hackPasswords() throws -> HackedPasswords {
  let fd = try open(path: "/etc/passwd")
  // `read` は fd を借用するので、この後使い続けることができる
  let contents = try read(from: fd)
  // `close` は fd を消費する。
  close(fd)

  let moreContents = try read(from: fd) // コンパイルエラー: consumeの後に使用する。

  return hackPasswordData(contents)
}
```

プロトコル要件がmove-only型のパラメータを持つ場合、サンクでは暗黙のうちにコピーできないため、実装メソッドの対応するパラメータの所有権規約を正確に一致させる必要がある:

```swift
protocol WritableToFileHandle {
  func write(to fh: borrowing FileHandle)
}

extension String: WritableToFileHandle {
  // error: プロトコルの要件を満たしていない
  // move-onlyの `FileHandle` 型のパラメータと一致しない。
  /*
  func write(to fh: consuming FileHandle) {
    ...
  }
   */

  // これはOK:
  func write(to fh: borrowing FileHandle) {
    ...
  }
}
```

現在、言語内のすべてのジェネリック型と存在型はコピー可能であると仮定されているが、move-only型をプロトコルに適合させ、ジェネリック関数で使用できるようにするためには、コピー可能性をオプション制約にする必要があります。そうなれば、プロトコル要件における所有権修飾子の厳格なマッチングの必要性は、コピー可能であることを要求されないジェネリックパラメータ型、関連型、存在型、および適合型にコピー可能であることを要求しないプロトコルにおける`Self`型にも及ぶことになるであろう。

### `set`/`out`パラメータ規約

`inout`パラメータは引数に排他的にアクセスできるため、他のコードを気にすることなく、現在の値を変更したり置き換えたりすることができる。これとは対照的に、`borrowing`パラメータは引数への共有アクセスを取得し、複数のコード片がコピーせずに同じ値を共有することができる。`consuming`パラメータは、値を消費して何も残らないが、初期化されていない引数を受け取り、新しい値を入力するという逆の規約のパラメータアナログはまだありません。C#やObjective-Cを含む多くの言語は、「Distributed Objects」機能で使用する場合、このための`out`パラメータ規約を持っており、Valプログラミング言語では、これを`set`と呼んでいる。

この時点までのSwiftでは、戻り値は、関数が呼び出し元に値を返すための好ましいメカニズムであった。この提案は、ある種の`out`パラメータを追加することを提案していないが、将来の提案では可能性がある。


## 参考リンク

### Forums

- []()

### プロポーザルドキュメント

- [borrowing and consuming parameter ownership modifiers](https://github.com/apple/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)

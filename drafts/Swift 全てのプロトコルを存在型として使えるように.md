# Swift 全てのプロトコルを存在型として使えるように

- [Swift 全てのプロトコルを存在型として使えるように](#swift-全てのプロトコルを存在型として使えるように)
  - [概要](#概要)
  - [内容](#内容)
    - [問題点](#問題点)
      - [型消去ラッパ](#型消去ラッパ)
    - [解決策](#解決策)
    - [使用できない場合](#使用できない場合)
    - [コンパイラチェック](#コンパイラチェック)
    - [不適合な存在型](#不適合な存在型)
    - [既知の実装を伴った関連型](#既知の実装を伴った関連型)
    - [関連型の共変消去](#関連型の共変消去)
    - [将来的な話](#将来的な話)
      - [Standard Libraryの`AnyHashable`と`AnyCollection`などの実装を簡単にする](#standard-libraryのanyhashableとanycollectionなどの実装を簡単にする)
      - [存在型の「強調」を抑える](#存在型の強調を抑える)
      - [存在型の展開](#存在型の展開)
      - [存在型のプロトコルへの自己準拠](#存在型のプロトコルへの自己準拠)
      - [anyキーワード](#anyキーワード)
      - [存在型に制約を与える](#存在型に制約を与える)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

これまでSwiftのプロトコルを型として使用できない場合があった。プロトコルを型として使用とした時に、下記のようなエラーを見たことがあるかもしれない。

```
Protocol can only be used as a generic constraint because it has 'Self' or associated type requirements.
```

この制限を緩和して、全てのプロトコルを存在型として使えるようにする。

## 内容

プロトコルを型として使う場合、その型は存在型と呼ばれる。

存在型は「入れ物」のようなもので、ある制約を満たす任意の型を動的にいつでも入れ替えることができる。同じ存在型の値に、同じ制約を満たす型の具体的な型情報を抽象化して、存在型として同じように使うことができる。

※ これとは反対の`some`キーワードのついた型は

- ある制約を満たしたある特定の型を表す
- 異なる同じ制約を満たした型に再代入できない

このドキュメントでは、存在型の値、存在型として使われているプロトコルやプロトコルを組み合わせたものを全て「存在型」と呼ぶ。
また、関連型(associated type)の要件（宣言）と関連型(依存メンバ型)を区別する(例えば、`Self.Element`と`Self.SubSequence.Element`は同じ関連型の要件を示す別の関連型)。

### 問題点

コンパイラは、下記の条件を除いてプロトコルを型として使うことを許可している。

1. 関連型(associated type)が要件に含まれている
2. メソッド/プロパティ/subscript/イニシャライザの共変ではない位置(引数など)に`Self`への参照が含まれている

```swift
// 'Identifiable'は関連型を要件に持つ
public protocol Identifiable {
  associatedtype ID: Hashable

  var id: ID { get }
}

// 'Equatable'は反変の位置でSelfを参照する操作メソッドwお要件に持つ
public protocol Equatable {
  static func == (lhs: Self, rhs: Self) -> Bool
}
```

1番目の条件が、関連型のメタデータを動的に復帰できないプロトコルのwitness tableの不完全な実装が残っているからである。
一方で2番目の条件では、違反した場合に同じようにエラーになるが、個々のメンバへのアクセスを型安全にする意図がある。次の例で考えてみる。

```swift
protocol P {
  func foo() -> Self
  
  func bar(_: Self)
}
```

存在型の値のメンバにアクセスするためには、そのプロトコルのコンテキストから外れてそのメンバの型を綴る能力が必要になる。
存在型の値の動的な`Self`が表すものは型消去されている。`P`型の`foo`から`P`へ型消去された共変の`Self`型の結果を返すのは型安全になる。一方で`bar`に型消去された値を渡すと型に合わない型を渡してしまう可能性がある。
プロトコルの要件に反して、プロトコル拡張(extension)には、プロトコル全体で利用可能かどうかを遡って判断できない。この場合、2番目の条件に違反していると、メンバへのアクセス失敗とされる。

```swift
protocol P {}
extension P {
  func method(_: Self) {}
}

func callMethod(p: P) {
  p.method // error: member 'method' cannot be used on value of protocol type 'P'; use a generic constraint instead
}
```

こういった`bar(_:Self)`や関連値の要件のみでインターフェイスの残りの部分について言う事はできない。上記の条件に当てはまらないプロトコルの有用な機能も存在する。これが使えないことが直感に反した実装の穴であると思われる。

例えば、下記の`Animal`は関連型のIDが`String`だと事前に定義されているため、完全に具体的な実装になるが使うことができない。

```swift
protocol Animal: Identifiable where ID == String {}
extension Animal {
  var name: String { id }
}
```

このセマンティック不一致があることで既存のプロトコルを存在型として使えなくなることを恐れてプロトコルを改善するプロセスの妨げにもなる。

#### 型消去ラッパ

型消去の実装は、単純に`Any`やクロージャで実際の値を隠す代わりに、コンパイラの最適化や将来的なABIとの互換性を考慮した方法で書くこともできる。
存在型に直接アクセスできない要件の場合、プロトコル拡張メソッドを介して呼び出しを転送し、内部の値を開いて、プロトコル拡張内からプロトコルインターフェイスに完全にアクセスすることができる。

```swift
protocol Foo {
    associatedtype Bar

    func foo(_: Bar) -> Bar
}

private extension Foo {
    // Forward to the foo method in an existential-accessible way, asserting that
    // the '_Bar' generic argument matches the actual 'Bar' associated type of the
    // dynamic value.
    func _fooThunk<_Bar>(_ bar: _Bar) -> _Bar {
        assert(_Bar.self == Bar.self)
        let result = foo(unsafeBitCast(bar, to: Bar.self))
        return unsafeBitCast(result, to: _Bar.self)
    }
}

struct AnyFoo<Bar>: Foo {
    private var _value: Foo

    init<F: Foo>(_ value: F) where F.Bar == Bar {
        self._value = value
    }

    func foo(_ bar: Bar) -> Bar {
        return self._value._fooThunk(bar)
    }
}

func f(_ f: Foo) {
    f.foo(1 as! Foo.Bar) // ❌　Associated type 'Bar' can only be used with a concrete type or generic parameter base
}

func f(_ f: AnyFoo<Int>) {
    f.foo(1) // ⭕️
}
```

### 解決策

任意のプロトコルを存在型として使えるようにして、プロトコルの要件と拡張(extension)にまたがって、個々のメンバへのアクセスへの制限を行うようにする。さらに、関連型がわかっている場合にメンバへのアクセスできるようにする。

```swift
protocol IntCollection: RangeReplaceableCollection where Self.Element == Int {}

let array: IntCollection = [3, 1, 4, 1, 5]

array.append(9) // ⭕️ Self.ElementはIntだとわかっている
```

関連値がただ存在することでメンバへのアクセスが制限されるという制限はなくなるが、`Self`をルートに持つ関連型は、`Self`にされる制限と同じ制限が発生する。

例えば、共変位置でのの`Self`は既に利用可能だが、

```swift
protocol Copyable {
  func copy() -> Self
}

func test(_ c: Copyable) {
  let x = c.copy() // ⭕️ xはCopyable型
}
```

これを共変位置の関連型にも拡張する。

```swift
func test(_ collection: RandomAccessCollection) {
  // func dropLast(_ k: Int = 1) -> SubSequence
  let x = collection.dropLast() // ⭕️xはRandomAccessCollection型
}
```

まとめると、プロトコルとプロトコル拡張のメンバ(メソッド/プロパティ/subscript/イニシャライザ)は存在型として使える。ただし、下記は除く

関連するメンバの型が、基となる型のコンテキストから見える時に、共変以外の位置に`Self`や`Self`をルートに持つ関連型を参照している場合

※ これらは共変と見なされる

- 戻り値の型の中の関数型
- いずれかの要素型の中のタプル型
- `Wrapped`型の`Optional`
- `Element`型の`Array`
- `Value`型の`Dictionary`

### 使用できない場合

全ての要件で存在型の値にアクセスできるわけではない。例えば、`==`は`Equatable`型の2つ値に使うことはできない。これは2つの型が合致するとは限らないからである。

```swift
let lhs: Equatable = "Paul"
let rhs: Equatable = "Alex"

lhs == rhs ❌

if let ownerName = lhs as? String, let petName = rhs as? String {
  print(ownerName == petName) ✅ // false
}
```

### コンパイラチェック

存在型の値で互換性のないメンバを呼び出すと、問題の簡潔な説明と、（該当する場合）プロトコルインターフェイスへのフルアクセスを取得するための一般的なアプローチの提案を含むエラーがトリガーされる。よくあるケースとして、存在型の基が関数またはサブルーチンパラメータへの参照の場合は、それをジェネリックパラメータに変換するfix-itが含まる（一部のローカルコンテキストではジェネリック関数が許可されていないため、該当する場合のみ）

```swift

extension Sequence {
  public func enumerated() -> EnumeratedSequence<Self> {
    return EnumeratedSequence(_base: self)
  }
}

func printEnumerated(s: Sequence) {
  // error: member 'enumerated' cannot be used on value of type protocol type 'Sequence'
  // because it references 'Self' in invariant position; use a conformance constraint
  // instead. [fix-it: printEnumerated(s: Sequence) -> printEnumerated<S: Sequence>(s: S)]
  for (index, element) in s.enumerated() {
    print("\(index) : \(element)")
  }
}

let collection: RangeReplaceableCollection = [1, 2, 3]
// error: member 'append' cannot be used on value of protocol type 'RangeReplaceableCollection'
// because it references associated type 'Element' in contravariant position; use a conformance
// constraint instead.
collection.append(4)
```

ただし、特定の型を参照していることを指摘するエラーの出力は今回の範囲外。これは非常に複雑。

```swift
struct G<T> {}

protocol P {
  associatedtype A
  associatedtype B
  
  func method() -> B
}

protocol Q: P where B == G<A> {}
```

プロトコルに同じ型の制約があるため、結果の型が示唆するように、`Q`型の値に対する`method`の呼び出しを妨げる関連型が実際には`B`ではなく`A`であることに注目

### 不適合な存在型

この制限によって不思議な副作用が発生する。準拠できない存在型が拡がる。

例えば、2つの無関係なプロトコルが同じ関連型を持ち、別の具体的な型で制約を設定した場合

```swift
protocol P1 {
  associatedtype A
} 
protocol P2: P1 where A == Int {}

protocol Q1 {
  associatedtype A
}
protocol Q2: Q1 where A == Bool {}

func foo(_: P2 & Q2) {

}
```

このコードは、事実上死んでいて、型を安全に保つことができる。将来的にはSwiftは区別する方法を提供するかもしれないが、今のところはコンパイラチェックでワーニングを出すくらいを提案する程度に留める。

### 既知の実装を伴った関連型

既知の実装とは、特定の存在型のジェネリックシグネチャの下で関連型が具体的な型に制約されている場合を指す。既知の実装は2つの具体例がある。

1. 明示的な同じ型の制約 例: `A == Int`
2. superclassの制約の要件のような他の要件経由でわかる関連型要件への実際の実装

```swift
class Class: P {
  typealias A = Int
}

protocol P {
  associatedtype A
}
protocol Q: P {
  func takesA(arg: A)
}

func testComposition(arg: Q & Class) {
  arg.takesA(arg: 0) // ⭕️ AはIntだとわかっている
}
```

既知の実装を持つ`Self`ルートの関連型への参照はメンバへのアクセスの妨げにならない。

### 関連型の共変消去

メンバにアクセスする際、`Self`ルートの関連型が「既知の実装を持っていない、かつアクセスするメンバの型の中の共変の位置に現れている」場合、
メンバにアクセスするために使われた存在型のジェネリックシグネチャごとにその上位の制約へと消去される。
上位の制約は、関連型のジェネリック制約の存在と種類によってclass、プロトコル、プロトコル合成、`Any`のいづれかになる。そのため、これらの関連型への参照があっても存在値のメンバにアクセスすることができる。

### 将来的な話

#### Standard Libraryの`AnyHashable`と`AnyCollection`などの実装を簡単にする

型消去ラッパで紹介したような方法を使ってStandard Libraryの型消去の実装をもっと簡単にする。

#### 存在型の「強調」を抑える

ジェネリック制約を使うべきところで存在型使われていることがしばしばある。これはパフォーマンスだけではなく、開発者が本当は存在型を使うことを意図してしないからという面もある。コンパイラが最適化をしてくれることもあるが、ジェネリック制約と存在型の2種類の抽象化には大きな違いがある(型レベルの抽象化と値レベルの抽象化)。値レベルの抽象化は型の異なる値間の型レベルの区別を排除し、独立した存在値間の型の関係を維持できない。

#### 存在型の展開

関数にジェネリック引数として渡されたときに、存在型を自動展開することでプロトコルに自己準拠させる。通常のインスタンスはopaque型に展開される。

#### 存在型のプロトコルへの自己準拠

`AnyHashable`に`init(_ box: Hashable)`を追加して混乱の軽減や使いやすさの修正をする。このイニシャライザはワークアラウンドとして扱われ、存在型の自己展開が利用可能になったらdeprecatedになる。

#### anyキーワード

`some P`の対義として明示的に存在型を表す`any P`を導入する。

詳細は[Swift 存在型(Existential type)とanyキーワード](/drafts/Swift%20存在型(Existential%20type)とanyキーワード.md#swift-存在型existential-typeとanyキーワード)

#### 存在型に制約を与える

例えば、下記のような。

```swift
let collection: any Collection<Self.Element == Int> = [1, 2, 3].
```

## 参考リンク

### Forums

- [SE-0309: Unlock existential types for all protocols](https://forums.swift.org/t/se-0309-unlock-existential-types-for-all-protocols/47515)

### プロポーザルドキュメント

- [Unlock existential for all protocols](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)

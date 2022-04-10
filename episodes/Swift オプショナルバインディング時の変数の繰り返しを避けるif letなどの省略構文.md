# Swift オプショナルバインディング時の変数の繰り返しを避けるif letなどの省略構文

収録日: 2022/04/02

- [Swift オプショナルバインディング時の変数の繰り返しを避けるif letなどの省略構文](#swift-オプショナルバインディング時の変数の繰り返しを避けるif-letなどの省略構文)
  - [概要](#概要)
  - [内容](#内容)
    - [これまで方法](#これまで方法)
    - [これからできるようになること](#これからできるようになること)
    - [詳細](#詳細)
      - [暗黙のselfとの関係](#暗黙のselfとの関係)
    - [将来的な検討事項](#将来的な検討事項)
      - [オプショナルキャスティング](#オプショナルキャスティング)
      - [borrow変数との関係](#borrow変数との関係)
      - [オブジェクトのネストされたメンバのアンラップ](#オブジェクトのネストされたメンバのアンラップ)
    - [検討された代替案](#検討された代替案)
      - [if foo](#if-foo)
      - [if unwrap foo](#if-unwrap-foo)
      - [if let foo?](#if-let-foo)
      - [if foo != nil](#if-foo--nil)
      - [if var fooは許可しない](#if-var-fooは許可しない)
    - [最終的な決定に関する記載](#最終的な決定に関する記載)
      - [一般的に拒否された変更をレビューにしたことについて](#一般的に拒否された変更をレビューにしたことについて)
      - [他の構文について](#他の構文について)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)

## 概要

これまで値が存在するオプショナル変数をアンラップする場合、新しい変数を`if let foo = foo { ... }`という形で行うのは、かなりよく使われる方法であった。このパターンでは、参照される識別子を2回繰り返す必要があり、これにより、特に長い変数名を使用する場合に、これらが冗長になる場合がある。このような変数をシャドーングするときに簡略構文を導入する:

```swift
let foo: Foo? = ...

if let foo {
    // `foo` is of type `Foo`
}
```

## 内容

### これまで方法

特に長い変数名の重複を減らすと、コードの読み書きの両方がよりしやすくなる。例えば、`someLengthyVariableName`と`anotherImportantVariable`をアンラップするこの文は、読みのがかなり面倒(そして、間違いなく書くのも面倒だったと思われる):

```swift
let someLengthyVariableName: Foo? = ...
let anotherImportantVariable: Bar? = ...

if let someLengthyVariableName = someLengthyVariableName, let anotherImportantVariable = anotherImportantVariable {
    ...
}
```

これに対処するための1つのアプローチは、より短く、説明の少ない名前を使用すること:

```swift
if let a = someLengthyVariableName, let b = anotherImportantVariable {
    ...
}
```

ただし、このアプローチではアンラップされた変数がどれについてのものなのかの明確さが落ちてしまう。短い変数名を推奨する代わりに、エルゴノミクス的に説明的な変数名を使用できるようにする必要がある。

### これからできるようになること

そこで、代わりに右辺の式を省略し、コンパイラがその名前で値が存在する変数を自動的にシャドーイングできるようにすると、これらのオプショナルバインディングははるかに冗長でなくなり、読み/書きが非常に簡単になる:

```swift
let someLengthyVariableName: Foo? = ...
let anotherImportantVariable: Bar? = ...

if let someLengthyVariableName, let anotherImportantVariable {
    ...
}
```

これは、オプションバインディングの既存の構文に対する自然な拡張である。

### 詳細

[optional-binding-condition](https://docs.swift.org/swift-book/ReferenceManual/Statements.html#grammar_optional-binding-condition)のSwiftの文法を拡張する。

現在は、

```
optional-binding-condition → let pattern initializer | var pattern initializer
```

だが、

```
optional-binding-condition → let pattern initializer<sub>opt</sub> | var pattern initializer<sub>opt</sub>
```
に更新される。

これは、すべての条件付き制御フロー文に適用される:

```swift
if let foo { ... }
if var foo { ... }

else if let foo { ... }
else if var foo { ... }

guard let foo else { ... }
guard var foo else { ... }

while let foo { ... }
while var foo { ... }
```

コンパイラは、シャドーイングされている変数を参照するイニシャライザを生成する:

```swift
if let foo { ... }
```
は

```swift
if let foo: Foo = foo { ... }
```

に変換される(内部で)。

この新しく導入されるパターンは、評価される式と新しく定義されたオプショナルではない識別子の両方を提供する。
既に存在している先例としては、クロージャのキャプチャリストがある:

```swift
let foo: Foo
let closure = { [foo] in // `foo`は式とクロージャ内で定義された新しい変数の識別子の両方になる 
    ...
}
```

このため、有効な識別子はこの構文の形のみとなる。次の例は無効:

```swift
if let foo.bar { ... } // ❌ アンラップの条件が妥当な識別子ではない
       ^               // fix-it: insert `<#identifier#> = `
```

#### 暗黙のselfとの関係

既存のオプショナルバインディングと同様に、この新しい構文は、`self`のオプショナルメンバをアンラップするための暗黙的な`self`参照をサポートする。例えば、下記は可能:

```swift
struct UserView: View {
    let name: String
    let emailAddress: String?

    var body: some View {
        VStack {
            Text(user.name)
        // `if let emailAddress = emailAddress { ... }`と同等
        // `self.emailAddress`をアンラップしている
            if let emailAddress {
                Text(emailAddress)
            }
        }
    }
}
```

### 将来的な検討事項

#### オプショナルキャスティング

この新しい構文の拡張は、オプショナルのキャストの省略形をサポートすること。例えば:

```swift
if let foo as? Bar { ... }
```

は

```swift
if let foo = foo as? Bar { ... }
```

と同等である。このプロポーザルには含まれないが、将来追加されるのは妥当な機能である。


#### borrow変数との関係

[A roadmap for improving Swift performance predictability](https://forums.swift.org/t/a-roadmap-for-improving-swift-performance-predictability-arc-improvements-and-ownership-control/54206#borrow-variables-7)で、コピーを作成せずに(排他的アクセスを強制することにより)既存の変数を「借用」する変数を作成するための新しい`ref`や`inout`の導入について説明されている。`let`/`var`との一貫性を保つために、これら新しい変数へのオプショナルバインディング条件をサポートすることはおそらく理にかなっている。

```swift
if ref foo = foo {
  // `foo`がnilでないならば、借用されてアンラップされた「不変」の変数として利用できる
}

if inout foo = &foo {
  // `foo`がnilでないならば、借用されてアンラップされた「可変」の変数として利用できる
}
```

```swift
if ref foo {
  // `foo`がnilでないならば、借用されてアンラップされた「不変」の変数として利用できる
}

if inout &foo {
  // `foo`がnilでないならば、借用されてアンラップされた「可変」の変数として利用できる
}
```

#### オブジェクトのネストされたメンバのアンラップ

このプロポーザルではできない。例えば:

```swift
if let foo.bar { ... } // ❌
```

将来、このタイプの構文をサポートするためのいくつかオプションがある。

1つのアプローチは、内部スコープでアンラップされた変数の識別子名を自動的に合成すること。例えば、`let foo.bar`が、`bar`または`fooBar`という名前の新しいオプショナルではない変数を生成する。

別のアプローチは、将来的に導入予定のborrow変数の`ref`および`inout`に対してこれを許可すること([A roadmap for improving Swift performance predictability](https://forums.swift.org/t/a-roadmap-for-improving-swift-performance-predictability-arc-improvements-and-ownership-control/54206#borrow-variables-7))。これらには、基盤となるストレージへのコンパイラによる排他的アクセスが含まれるため、技術的には、内部スコープに一意の識別子名は必要ない。これにより、新しい変数やコピーなしでオブジェクトのメンバをアンラップできるようになる。例えば:

```swift
// mother.father.sisterはオプショナル

if ref mother.father.sister {
  // mother.father.sisterはオプショナルではなく「不変」
}

if inout &mother.father.sister {
  // mother.father.sisterはオプショナルではなく「可変]
}
```

### 検討された代替案

#### if foo

この機能の可能な限り短いスペルは、`if foo`の条件をただむき出しにすることだろう。ただし、このスペルは、オプショナルアンラッピング条件と`Bool`条件の間にあいまいさを生じ、混乱を招く/直感的でない状況につながる可能性がある。

```swift
let foo: Bool = true
let bar: Bool? = false

if foo, bar {
  // would succeed
}

if foo == true, bar == true {
  // would fail
}
```
この曖昧さを回避するために、オプショナルバインディングのためのある種の別の構文が必要。

#### if unwrap foo

別のオプションは、この目的のために新しいキーワードまたはシンボルを導入すること。例えば、`if unwrap foo`、`if foo?`や`if have foo`。
`unwrap foo`のような完全に新しい構文を導入する主な利点は、オプショナルバインディング条件が実際にどのように機能するかについてのセマンティクスを再検討する機会が得られること。 現在、オプショナルバインディング条件は常に値のコピーを作成する。パフォーマンスの観点からは、コピーの代わりに*借用*する方が効率的。

[A roadmap for improving Swift performance predictability](https://forums.swift.org/t/a-roadmap-for-improving-swift-performance-predictability-arc-improvements-and-ownership-control/54206#borrow-variables-7)では、潜在的な将来の`ref`(不変の借用)および`inout`(可変の借用)について説明している。`let`/`var`との一貫性を保つために、これらの新しい変数のオプショナルバインディング条件をサポートすることはおそらく理にかなっている。

```swift
if ref foo = foo {
  // `foo`がnilでないならば、借用されてアンラップされた「不変」の変数として利用できる
}

if inout foo = &foo {
  // `foo`がnilでないならば、借用されてアンラップされた「可変」の変数として利用できる
}
```

この新しい短縮構文は、`if let`の省略形ではなく、`if ref`の省略形にすることができる。これにより、一般的にパフォーマンスが向上し、コピーの代わりに借用を使用するようにユーザを誘導することができる(借用の形式のみが短縮の糖衣構文を受け取るため)。

ただし、借用の主な欠点は、借用した変数への排他的アクセスが必要になること。メモリ排他性違反は、場合によってはコンパイラエラーを引き起こすが、より複雑な場合にはランタイムエラーとして現れることもある。 例えば:

```swift
var x: Int? = 1

func increment(by number: Int) {
    x? += number
}

if ref x = x {
    increment(by: x)
}
```

`increment(by:)`は、`if ref x = x`のオプショナルバインディング条件によって既に借用されている間に、`x`の値を変更しようとするため、ランタイム時にトラップする。

借用変数が言語に追加されると、Swiftのどこかに`ref x`または`inout x`が表示されることは、コードの排他性要件に関する重要な視覚的マーカとして機能する。一方、`if unwrap x`のような新しい構文は、変数が借用されていることを明示的に示さない。これにより、予期しない排他性違反にユーザが驚かされる可能性があり、混乱を招くコンパイル時エラーやランタイムクラッシュが発生する可能性がある。

借用変数は非常に便利だが、それらを採用することは、パフォーマンスと概念上のオーバーヘッドの間のトレードオフ。借用はコストが低いが、概念的なオーバーヘッドは高くなる。コピーはコストが高くなる可能性があるが、余計なことを考えなくても常に期待どおりに機能する。このトレードオフを考えると、この省略構文は、ユーザをどちらかに制限するのではなく、ユーザがコピーを実行するか借用を実行するかを選択する方法を提供するのが理にかなっている。

さらに、既存のオプショナルバインディング条件との一貫性を保つために、この新しい省略形は、不変変数と可変変数の区別をサポートする必要がある。コピーと借用の区別と組み合わせると、通常の変数と同じオプショナルのセットが得られる:

```swift
// 今回のプロポーザルに含まれる:
if let foo { /* foo is an immutable copy */ }
if var foo { /* foo is a mutable copy */ }

// 将来的に追加される:
if ref foo { /* foo is an immutable borrow */ }
if inout &foo { /* foo is a mutable borrow */ }
```

これらの概念の構文はすでにあるので、表現力が低い(例えば、使用可能な選択肢のサブセットのみをサポートする)、明示性が低い(例えば、ユーザが この新しい省略形は、コピーまたは借用を実行するかを覚えておかなければならないなど)新しい構文を作成するよりも既存の構文を再利用するべき。

#### if let foo?

別のオプションは`?`を含めること。`if let foo?`を使用して、これがオプショナルアンラップしていることを明示的に示す。これは、既存の`case let foo?`パターンマッチング構文を示している。

`if let foo = foo`(このための最も一般的な既存の構文)は、明示的な`?`なしでオプショナルアンラップする。これは、オプショナル値の存在を示しますために、オプショナルバインディングが`?`なしでも十分に明確であることを意味する。この場合、追加の`?`は、`if let foo`の省略形に厳密にはおそらく必要ない。

`if let foo?`と`case let foo?`の対称性は素晴らしいものの、`if let foo = foo`との一貫性がより重要であり、同じ文内でより頻繁に使われる:

```swift
// 一貫
if let user, let defaultAddress = user.shippingAddresses.first { ... }

// 一貫していない
if let user?, let defaultAddress = user.shippingAddresses.first { ... }
```

さらに、`?`シンボルを使用すると、`if let foo：Foo = foo`のように明示的な型アノテーションをサポートするのが難しくなる。`if let foo: Foo`は、既存の文法で自然なもの。これが追加の`?`でどのように機能するかがあまり明確ではない。`if let foo?: Foo`はおそらく最も理にかなっているが、既存の言語構造とは一致しない。

#### if foo != nil

やや一般的な提案の1つは、`nil`チェック(`if foo！= nil`など)で変数を内部スコープでアンラップできるようにすること。Kotlinはこのタイプの構文をサポートしている。

```swift
var foo: String? = "foo"
print(foo?.length) // "3"

if (foo != null) {
    // `foo`はオプショナルではない
    print(foo.length) // "3"
}
```

Kotlinのこのパターンは、新しい変数を定義しない。内部スコープ内の既存の変数の型を変更するだけ。したがって、内側のスコープの変更は、外側のスコープにも影響を与える。

```swift
var foo: String? = "foo"

if (foo != null) {
    print(foo) // "foo"
    foo = "bar"
    print(foo) // "bar"
}

print(foo) // "bar"
```

これは、内部スコープで新しい個別の変数を定義するSwiftのオプショナルバインディング条件(`if let foo = foo`)とは異なる。これはSwiftのオプショナルバインディング条件の特徴であるため、省略構文では、新しい変数が宣言されていることを十分に明確にする必要がある。

#### if var fooは許可しない

`if var foo = foo`は、`if foo = foo`よりもそこまで一般的ではないため、この省略構文で`var`をサポートしないことを選択できる可能性があった。

`var`のシャドーイングは、`let`のシャドーイングよりも混乱する可能性がある。`var`は新しい可変変数を導入し、新しい変数への変更は元のオプショナル変数と共有されない。一方、`if var foo = foo`がすでに存在し、`if var foo`が既存の構文よりも混乱を招いたり、明確でないとは思われない。

`let`と`var`は言語の相互に入れ替え可能であるため、ここでも同様である必要がある。`if var foo`ができないことは、既存のオプショナルバインディング条件構文と一貫性がない。もし`let`を使用しない別の何かを使っているとしたら、`var`を除外するのが妥当かもしれませんが、ここでは`let`を使用しているため、`var`も許可する必要がある。

### 最終的な決定に関する記載

#### 一般的に拒否された変更をレビューにしたことについて

このトピックは[commonly-rejected list ](https://github.com/apple/swift-evolution/blob/main/commonly_proposed.md)の載っていた。これは、Swift 3で言語が急速に変更される中、Swift Evolutionプロセスの非常に早い段階で、2016年に追加された。このリストは、導入部分に記載されているように、特定のアイデアについての議論を完全に禁止するものではない。

> これは、頻繁に提案されているが受け入れられる可能性が低いSwift言語への変更のリスト。この分野で何かを追求することに興味がある場合は、私たちがすでに行った議論を理解してほしい。これらのトピックを取り上げるには、「これが本当に欲しい」または「これは他の言語に存在し、そこで気に入った」と言うだけでなく、ディスカッションに新しい情報を追加することが期待される。

Swiftコードでの`if let`および`guard let`の普及、この定型文に対処するためのより大きなSwiftコミュニティからの長年にわたる継続的な要求、および強力なサポートピッチを備えた適切な提案を考えると、SE-0345はこの基準を満たしている。この承認の一環として、リストは更新され、このエントリは削除される。

#### 他の構文について

プロポーザルに対する回答の大部分は、`if let x = x`ボイラープレートを排除するための糖衣構文を提供することに賛成であり、議論の多くは代替構文にフォーカスしていた。コアチームはこれらの代替案を評価し、提案された構文を承認した。

最も議論された代替構文は、`if let x?`だった。これは、[if let foo?]()で扱われている。この構文は、シャドーイングの側面ではなく、`if let`のオプショナルアンラッピングの側面を強調し、`if let`自体よりもパターンマッチング(`if case let`を介して)をより強調している。[提供された考察](https://forums.swift.org/t/se-0345-if-let-shorthand-for-shadowing-an-existing-optional-variable/55805/138)では、`?`の代わりに`unwrap`構文が示されている:

> `unwrap`キーワードの概念、または`?`のようなシンボルの必要性は、構文にアンラッピングが発生していることを示すものがないため、既存の`if let foo = bar`またはシャドーイングバージョンの`if let foo = foo`が新規参入者にとってすぐには明確ではないという事実に由来ししていると思う。したがって、既存のオプショナルバインディング構文は、最初に見たときに非常に不明瞭であると言える。

コアチームは、`if let`のオプショナルアンラッピングの挙動がSwiftの新規参入者にとって障害となる可能性があることに同意する。ただし、その設計上の決定はSwiftに深く根付いているため、変更される可能性は低く、1つの小さな場所でその問題に対処しようとすると(例えば、`let x?`を`let x = x`の糖衣構文として提供するなど)、改善するのではなく、さらに混乱させることに繋がりかねない。

## 参考リンク

### Forums

- [SE-0345: `if let` shorthand for shadowing an existing optional variable](https://forums.swift.org/t/se-0345-if-let-shorthand-for-shadowing-an-existing-optional-variable/55805)
- [[Accepted] SE-0345: `if let` shorthand for shadowing an existing optional variable](https://forums.swift.org/t/accepted-se-0345-if-let-shorthand-for-shadowing-an-existing-optional-variable/56364)
- [Feedback on proposal acceptances](https://forums.swift.org/t/feedback-on-proposal-acceptances/56411/28)

### プロポーザルドキュメント

- [if let shorthand for shadowing an existing optional variable](https://github.com/apple/swift-evolution/blob/main/proposals/0345-if-let-shorthand.md)
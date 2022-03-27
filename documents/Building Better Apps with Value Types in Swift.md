# Building Better Apps with Value Types in Swift(WWDC2015)

- [Building Better Apps with Value Types in Swift(WWDC2015)](#building-better-apps-with-value-types-in-swiftwwdc2015)
  - [概要](#概要)
  - [内容](#内容)
    - [参照セマンティクス](#参照セマンティクス)
      - [Cocoa(Touch)とObjective-Cのコピー](#cocoatouchとobjective-cのコピー)
    - [Immutability(不変性)](#immutability不変性)
      - [可変を排除する](#可変を排除する)
      - [Cocoa(Touch)の不変性(コピーが不要なもの)](#cocoatouchの不変性コピーが不要なもの)
    - [値セマンティクス](#値セマンティクス)
      - [値型の合成](#値型の合成)
      - [値型と値は区別される](#値型と値は区別される)
    - [Temperatureの値セマンティクス版](#temperatureの値セマンティクス版)
      - [必要な時に可変で、不要な時は不変](#必要な時に可変で不要な時は不変)
      - [競合状態からの解放](#競合状態からの解放)
      - [コピーのパフォーマンスは？](#コピーのパフォーマンスは)
    - [値型の実用例](#値型の実用例)
      - [Circle](#circle)
      - [Polygon](#polygon)
      - [Diagramは両方を含む](#diagramは両方を含む)
      - [Diagramの作成](#diagramの作成)
      - [ダイアグラムを使う](#ダイアグラムを使う)
      - [DiagramをDrawableに準拠させる](#diagramをdrawableに準拠させる)
    - [値型と参照型を混ぜる](#値型と参照型を混ぜる)
      - [値型を含んだ参照型](#値型を含んだ参照型)
      - [参照型を含んだ値型](#参照型を含んだ値型)
      - [不変の参照は問題なし](#不変の参照は問題なし)
      - [不変の参照と`Equatable`](#不変の参照とequatable)
      - [可変オブジェクトへの参照](#可変オブジェクトへの参照)
      - [Copy-on-write](#copy-on-write)
      - [Copy-on-writeの活用](#copy-on-writeの活用)
      - [Polygonからパスを形成する](#polygonからパスを形成する)
      - [ユニークな参照のSwiftオブジェクト](#ユニークな参照のswiftオブジェクト)
      - [値型でundoを実装する](#値型でundoを実装する)
    - [まとめ](#まとめ)
  - [参考リンク](#参考リンク)

## 概要

Swiftは強力な構造体を形成する表現豊かな第一級の値型をサポートしている。参照型と値型の違いや、可変性やスレッドセーフといった共通の課題を値型がどのように洗練された方法で解決するかを学ぶ。

## 内容

### 参照セマンティクス

Swiftでは`class`を使って参照セマンティクスを表現する。

```swift
class Temperature {
    var celsius: Double = 0
    var fahrenheit: Double {
        get { return celsius * 9 / 5 + 32 }
        set { celsius = (newValue - 32) * 5 / 9 }
    }
}
```

これを家のサーモスタットに設定する。

```swift
let home = House()
let temp = Temperature()
temp.fahrenheit = 75
home.thermostat.temperature = temp
```

次に夕食のために何かを焼くためにオーブンを温める。

```swift
temp.fahrenheit = 425
home.oven.temperature = temp
home.oven.bake()
```

こうするとサーモスタットの温度も425°Cになって大変なことになる。

これは参照セマンティクスの`Temperature`オブジェクトを意図せずに共有してしまっている結果。オブジェクトグラフは下記のようになっている。

![参照型の問題](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/reference_temperature.png)

これを防ぐためにはコピーが必要になる。

```swift
// ...
temp.fahrenheit = 75
home.thermostat.temperature = temp.copy()

temp.fahrenheit = 425
home.oven.temperature = temp.copy()
```

後ろのコピーは必要ないが、次回同じ問題を起こさないためにもあったほうが良い。これは「防御的コピー」と呼ばれ、新しい`Temperature`が必要な全てのオブジェクトでしなければならない。

```swift
class Oven {
    private var _temperature: Temperature = Temperature(celsius: 0)

    var temperature: Temperature {
        get { return _temperature }
        set { _temperature = newValue.copy() }
    }
}
```

これはパフォーマンスのロスにつながり、コピーを忘れた際にいつもバグが起きる可能性がある。

#### Cocoa(Touch)とObjective-Cのコピー

Cocoa(Touch)は全体でコピーを必須としている

- `NSCopying`でオブジェクトをコピーするように実装されている(`NSString`, `NSArray`, `NSDictionary`, `NSURLRequest`など全てコピーが必須)

防御的コピーはCocoa(Touch)とObjective-Cは全体に浸透している

- `NSDictionary`は将来の変更がハッシュ値に影響しないようにkeysはコピーされる
- `copy`属性は代入時に防御的コピーを行う

### Immutability(不変性)

#### 可変を排除する

関数型プログラミングには不変な参照セマンティクスが存在する。これは、意図せぬ副作用など可変の参照セマンティクスが起こす多くの問題を解決する。

一方で不変な参照セマンティクスにもいくつかの問題がある。

- 可変が可能な世界から考えると、扱いづらいインターフェイスを定義することになるかもしれない
- 最終的にはステートフルなCPUやキャッシュ、メモリなどにマップしなければならないがマシンモデルに効率的にマップできない

`Temperature`を不変にすると下記のようになる

```swift
class Temperature {
    // celsiusはあるタイミングの値を表す。不変。
    let celsius: Double = 0

    var fahrenheit: Double {
        return celsius * 9 / 5 + 32
    }

    init(celsius: Double) { self.celsius = celsius }
    init(fahrenheit: Double) { self.celsius = (fahrenheit - 32) * 5 / 9 }
}
```

これは扱いづらいインターフェイスを繋がる。10°Cを足したい場合を考えてみる。可変であった場合、

```swift
// 可変
home.oven.temperature.fahrenheit += 10.0
```

これが不変だと

```swift
// 不変
let temp = home.oven.temperature
home.oven.temperature = Temperature(fahrenheit: temp.fahrenheit + 10.0)
```

これは扱いづらい上にヒープ上に他のオブジェクトを割り当てるためパフォーマンスも落ちる。

もっと重要なのは、これは完全は不変ではない。新しい温度を設定する時、まだ`oven`を変更している。本当に不変の場合は、新しい温度を作成して、オーブンを作成して、家を作成して...とさらに扱いづらくなる。


<details>
<summary>他にも問題となる例がある</summary>

エラスムスの振る舞い

可変バージョン

```swift
func primes(n: Int) -> [Int] {
    var numbers = [Int](2..<n)

    for i in 0..<n - 2 {
        guard let prime = numbers[i] where prime > 0 else { continue }
        for multiple in stride(from: 2 * prime - 2, to: n - 2, by: prime) {
        numbers[multiple] = 0
        }
    }
    return numbers.filter { $0 > 0 }
}
```

不変バージョン(filterが全ての値を毎回走路するのでパフォーマンス的に良くない)

```swift
func sieve(numbers: [Int]) -> [Int] {
    guard !numbers.isEmpty else { return [] }
    let p = numbers[0]
    return [p] + sieve(numbers[1..<numbers.count]).filter { $0 % p > 0 }
}
```
</details>
<br/>

#### Cocoa(Touch)の不変性(コピーが不要なもの)

多くの不変なクラスが存在する:

- `NSDate`, `NSURL`, `UIImage`, `NSNumber`など
- 安全性が改善されており、これらはコピーが不要

一方で同様にデメリットもある。Objective-Cではホームディレクトリから初めてパスのコンポーネントを繰り返し追加したい場合、それぞれのイテレーションで新しい`NSURL`を作成しなければならないかもしれない。

```objective-c
NSURL *url = [[NSURL alloc] initWithString: NSHomeDirectory()];
NSString *component;
while ((component = getNextSubdir())) {
    // ヒープ上に新しいオブジェクトを作成、全ての文字列データをコピー、古いオブジェクトを破棄
    url = [url URLByAppendingPathComponent: component];
}
```
より良い方法としては、コンポーネントを配列に追加して、新しい`NSURL`を最後に一つ作成する。しかし、完全に不変にしたい場合、全てのイテレーションで新しい配列を作成しなければならない。まだ計算量は`O(n^2)`である。

Cocoaでこれを行う適切な方法は可変を利用すること。`NSMutableArray`に全てのコンポーネントを保持して、最後に`NSURL`を一つ作成すると、計算量は`O(n)`になる。

```objective-c
NSMutableArray<NSString *> *array = [NSMutableArray array];
[array addObject: NSHomeDirectory()];
NSString *component;
while ((component = getNextSubdir())) {
    [array addObject: component];  // 可変の配列オブジェクト
}
```

**不変は良いこと**: 参照セマンティクスで何が起きているのかをわかりやすくするが、全体で完全に不変にすることはできない。

### 値セマンティクス

別のアプローチをしてみる。正しく実行すると使いやすいので可変が好ましいが、問題はデータの共有。

値型はこれを解決する。ある値型の変数の変更は、他の変数に決して影響を与えない。

```swift
var a: Int = 17
var b = a
assert(a == b)
b += 25
print("a = \(a), b = \(b)")  // Prints "a = 17, b = 42"
```

値型は数字や`CGPoint`などの基本的な型に使われている。`b`の変更が`a`に影響を与えることは決して期待しない。

より複雑な型でも同じ原理を使っている

#### 値型の合成

- Swiftの「基本的な」型は全て値型(`Int`, `Double`, `String`)
- Swiftのコレクションも値型(`Array`, `Set`, `Dictionary`)

値型のみで構成されているstruct、enum、tupleも値型になる。値型の世界で表現豊かな抽象を構築するのはとても簡単。


#### 値型と値は区別される

等価性は変数の値によって成立する。その変数のアイデンティティやどのメモリアドレスあるのかではない。

これは数字などの基本的な型では完全に理解できる。

```swift
var a: Int = 5
var b: Int = 2 + 3
assert(a == b)  // True
```

これはコレクションのようなもう少し複雑な型にも拡張できる。

```swift
var a: [Int] = [1, 2, 3]
var b: [Int] = [3, 2, 1].sort(<)
assert(a == b)  // True
```

### Temperatureの値セマンティクス版

```swift
struct Temperature {
    var celsius: Double = 0
    var fahrenheit: Double {
        get { return celsius * 9 / 5 + 32 }
        set { celsius = (newValue - 32) * 5 / 9 }
    }
}
```

残念なことに、このstructを使うとエラーになr。

```swift
let temp = Temperature()
temp.fahrenheit = 75  // ERROR: cannot assign to property: 'temp' is a 'let' constant
```

`let`を`var`に変更することで全てがうまくいく。`thermostat`も`oven`もそれぞれの`Temperature`インスタンスの値を持っている。シェアがなく、struct内にインライン化しているため、メモリ利用やパフォーマンスの面でも良い。

![値型の解決](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/valuetype_temperature.png)

#### 必要な時に可変で、不要な時は不変

Swiftでは可変を示すキーワードがあるので、値セマンティクスとうまく協働している。

`let`は値が決して変わらないことを意味する。

```swift
let numbers = [1, 2, 3, 4, 5]
```

`var`は他に影響を与えずに値を更新できることを意味する(データ競合なし)。

```swift
var strings = [String]()
for x in numbers {
    strings.append(String(x))
}
```

#### 競合状態からの解放

スレッドの境界をまたいで値型を渡すことは、その型の競合状態を心配する必要がないことを意味する。

参照セマンティクスの配列の場合、下記のコードは、両方のスレッドで同じ配列を共有しているため競合状態を起こす。値セマンティクスの配列の場合、毎回「ロジック上」はコピーを取得するので、それぞれのスレッドは自身の配列を持つことになる。

```swift
var numbers = [1, 2, 3, 4, 5]
scheduler.processNumbersAsynchronously(numbers)

for i in 0..<numbers.count {
    numbers[i] = numbers[i] * 1
}
scheduler.processNumbersAsynchronously(numbers)
```

#### コピーのパフォーマンスは？

値セマンティクスのコピーはとても軽量(`O(1)`)。

- ローレベルの基本的な型は定数時間(`Int`、`Double`...)
    - 数バイトのコピーだけが必要でプロセッサの内部で行われる
- 値型のstruct、enum、タプルも定数時間(`CGPoint`...)
    - 固定数のフィールドなので`O(1)`
    - それぞれの「構築ブロック」のコピーは定数時間、なので全体のオブジェクトのコピーも定数時間
- より発展系のデータ構造はcopy-on-writeを使う(`String`、 `Array`、`Set`、`Dictionary`など)
    - コピーには、固定数の参照カウント操作が含まれる
    - コピーは変更が発生した時点でのみ起こる。これは「舞台裏の共有」でロジック上の共有ではない。「必要な時」は別々の値をして扱われる

**値セマンティクスはシンプルで効率が良い**
- 異なる変数は、ロジック上別々
- 必要な時に可変
- コピーは軽量

### 値型の実用例

円やポリゴンなどのシンプルな値型からダイアグラムを構築する。

#### Circle

```swift
struct Circle {
    var center: CGPoint
    var radius: Double
}
```

![struct Circle](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/circle.png)

#### Polygon

```swift
struct Polygon {
    var corners = [CGPoint]()
}
```

![struct Polygon](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/polygon.png)


#### Diagramは両方を含む

`Circle`と`Polygon`をプロトコル`Drawable`に準拠させる。

![CircleとPolygonをDrawableに準拠させる](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/drawable.png)

```swift
protocol Drawable {
  func draw()
}

extension Polygon: Drawable {
    func draw() {
        let ctx = UIGraphicsGetCurrentContext()
        CGContextMoveToPoint(ctx, corners.last!.x, corners.last!.y)
        for point in corners {
        CGContextAddLineToPoint(ctx, point.x, point.y)
        }
        CGContextClosePath(ctx)
        CGContextStrokePath(ctx)
    }
}

extension Circle: Drawable {
    func draw() {
        let arc = CGPathCreateMutable()
        CGPathAddArc(arc, nil, center.x, center.y, radius, 0, 2 * π, true)
        CGContextAddPath(ctx, arc)
        CGContextStrokePath(ctx)
    }
}
```

#### Diagramの作成

```swift
struct Diagram {
    var items = [Drawable]()

    mutating func addItem(item: Drawable) {
        items.append(item)
    }

    func draw() {
        for item in items {
        item.draw()
        }
    }
}
```

#### ダイアグラムを使う

```swift
var doc = Diagram()
doc.addItem(Polygon())
doc.addItem(Circle())
var doc2 = doc
doc2.items[1] = Polygon(corners: points)
```

![代入時にタイアグラムはコピーされる](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/diagram.png)

#### DiagramをDrawableに準拠させる

```swift
struct Diagram: Drawable {}
```

新しい`Diagram`を最初の`Diagram`に追加する。

```swift
var doc = Diagram()
doc.addItem(Polygon())
doc.addItem(Circle())
doc.addItem(Diagram())
```

![新しいDiagramの追加](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/add_new_diagram.png)


自身の`items`配列にダイアグラムを追加することもできる

```swift
var doc = Diagram()
doc.addItem(Polygon())
doc.addItem(Circle())
doc.addItem(doc)
```

![Diagramインスタンス自身を追加](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/add_existing_diagram.png)

なぜ無限ループが起きないのか？

```swift
func draw() {
    for item in items {
        item.draw()
    }
}
```

参照セマンティクスであったら、`items`のイテレートで`draw`を呼び出し、さらに自身の`draw`を呼び出して、無限ループが発生していた。

しかし、値セマンティクスを使っているので、`items`配列に自身が存在しないため、元の`Diagram`とは違う`Diagram`の完全に新しいコピーを取得する。

![値セマンティクスのDiagramインスタンスを追加](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/valuesemantics_diagram.png)

### 値型と参照型を混ぜる

#### 値型を含んだ参照型

値型は一般的にオブジェクトの「基本」データとして使われる:

```swift
class Button: Control {
    var label: String
    var enabled: Bool
    // ...
}
```

#### 参照型を含んだ値型

値型のコピーは参照を共有する:

```swift
struct ButtonWrapper {
    var button: Button
}
```

値セマンティクスを維持するためには特別な考慮が必要:

- 参照型のオブジェクトの変更をどう扱う？
- 参照型のアイデンティティは等価性にどう影響を与えるのか？

#### 不変の参照は問題なし

参照型`UIImage`をラップした`Image`を作成する:

```swift
struct Image: Drawable {
    var topLeft: CGPoint
    var image: UIImage
}
```

同じ`UIImage`オブジェクトの参照を示す2つの`Image`を作成する:

```swift
var image = Image(
    topLeft: CGPoint(x: 0, y: 0),
    image: UIImage(imageNamed: "San Francisco")!
)
var image2 = image
```

![UIImageを共有した2つのImage](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/image_shared_uiimage.png)

[`UIImage`は不変](#cocoatouchの不変性)なので問題が発生しない。`image`と共有した`UIImage`を変更される心配がない。

```
Because image objects are immutable, you can’t change their properties after creation. 
```
https://developer.apple.com/documentation/uikit/uiimage


#### 不変の参照と`Equatable`

`Image`を`Equatable`に準拠させる:

```swift
extension Image: Equatable {}
```

上記の例において、`image`と`image2`は等価である。

ではビットマップの異なる`UIImage`を設定した場合はどうなるか。

![UIImageを共有した2つのImage](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/image_separate_uiimage.png)

異なる参照を持っているが、同じbitmapであるため等価と見なされる。

#### 可変オブジェクトへの参照

可変な参照型`UIBezierPath`で全体を構成する`BezierPath`を作成する。もし2つの`BezierPath`が同じ参照を共有する場合は問題になる。

![BezierPath](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/bezierPath.png)


```swift
struct BezierPath: Drawable {
    var path = UIBezierPath()

    var isEmpty: Bool {
        return path.empty
    }

    // 問題発生の可能性あり
    func addLineToPoint(point: CGPoint) {
        path.addLineToPoint(point)
    }
}
```

`path`が参照型なので`mutating`キーワードは不要なことに注意。値型を使っているとしても値セマンティクスを維持することができていないため、良くない状況にある。

#### Copy-on-write

上記の状況を修正するには、Copy-on-writeを使う。

- 参照されるオブジェクトの無制限の変更は値セマンティクスを破壊する
- mutatingではない操作とmutating操作を分ける
    - mutatingではない操作は常に安全
    - mutating操作は最初にコピーしなければならない

#### Copy-on-writeの活用

値セマンティクスを維持するために、読み取りと書き込みのアクセサを分ける必要がある

```swift
struct BezierPath: Drawable {
    private var _path = UIBezierPath()

    var pathForReading: UIBezierPath {
        return _path
    }

    var pathForWriting: UIBezierPath {
        mutating get {
        _path = _path.copy() as! UIBezierPath
        return _path
        }
    }

    var isEmpty: Bool {
        return pathForReading.empty
    }

    mutating func addLineToPoint(point: CGPoint) {
        pathForWriting.addLineToPoint(point)
    }
}
```

このバージョンだと、変更が発生しない限り、`UIBezierPath`の参照をシェアする。

```swift
var path = BezierPath()
var path2 = path
if path.empty { print("Path is empty") }
```

![2つのBezierPathでUIBezierPathシェア](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/bezierpath_share_uibezierpath.png)

...そして、変更する時に、まずコピーを作成するため`path2`に影響を与えない:

```swift
path.addLineToPoint(CGPoint(x: 10, y: 20))
```

![変更時に2つのBezierPathで別々のUIBezierPathになる](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/bezierpath_separate_uibezierpath.png)

#### Polygonからパスを形成する

不変のアルゴリズム:

```swift
extension Polygon {
    var path: BezierPath {
        var result = BezierPath()
        result.moveToPoint(corners.last!)
        for point in corners {
            // 毎ループごとに`BezierPath`がコピーされる!
            result.addLineToPoint(point)
        }
        return result
    }
}
```

これはパフォーマンスに影響が出るかもしれない。より良い実装は注意深く変更を使う:

```swift
extension Polygon {
    var path: BezierPath {
        var result = UIBezierPath() // 参照型
        result.moveToPoint(corners.last!)
        for point in corners {
            // 毎ループは変更
            result.addLineToPoint(point)
        }
        // 最後に値型を作成する
        return BezierPath(path: result)
    }
}
```

#### ユニークな参照のSwiftオブジェクト

Swiftはオブジェクトがユニークに参照されているかどうかを伝えることができるため、不要なコピーを避けることができる。Standard Libraryでは、この特徴を応用的に使っている。

```swift
struct MyWrapper {
    var _object: SomeSwiftObject
    var objectForWriting: SomeSwiftObject {
        mutating get {
        if !isUniquelyReferencedNonObjc(&_object) {
            _object = _object.copy()
        }
        return _object
        }
    }
}
```

※ 現在は`isKnownUniquelyReferenced`に置き換わっており、このコードは古い。[Remove NonObjectiveCBase and isUniquelyReferenced](https://github.com/apple/swift-evolution/blob/master/proposals/0125-remove-nonobjectivecbase.md)


**値型と参照型を混ぜる**
- 値セマンティクスを維持するために、特別な配慮がいる
- Swiftの参照型をラップする際、Copy-on-writeを使うと効率的に値セマンティクスを維持できる

#### 値型でundoを実装する

ダイアグラムとダイアグラムの配列を作成する。毎回の変更時、ダイアグラムの配列に`doc`を追加する。

```swift
var doc = Diagram()
var undoStack = [Diagram]()
undoStack.append(doc)

doc.addItem(Polygon())
undoStack.append(doc)

doc.addItem(Circle())
undoStack.append(doc)
```

![undostack](../images/Building%20Better%20Apps%20with%20Value%20Types%20in%20Swift/undostack.png)


現在、`undo_stack`は3つの異なる`Diagram`のインスタンスを持っている。値セマンティクスを使っているので、同じ`Diagram`への参照は存在しない。

アプリ上だと、過去の状態をリストで記録したヒストリー機能として活用できる。`undo_stack`のインデックスを検索することで簡単に前後に移動できる。

Adobe Photoshopはこの仕組みを使っている(現在は不明)。

### まとめ

- 参照セマンティクスで予期せぬ変更をするとバグに繋がる
- 値セマンティクスは可変を示す表現性と不変であるという安全性を提供することで、これらの問題を解決する　
## 参考リンク

- [WWDC NOTES](https://www.wwdcnotes.com/notes/wwdc15/414/)
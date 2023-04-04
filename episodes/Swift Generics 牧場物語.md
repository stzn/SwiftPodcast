# Swift Generics 牧場物語

収録日: 2022/06/19

- [Swift Generics 牧場物語](#swift-generics-牧場物語)
  - [SwiftにおけるGenericsとは？](#swiftにおけるgenericsとは)
  - [抽象化するとは？](#抽象化するとは)
  - [牧場を例にSwiftのGenericsを使った抽象化について考えてみる](#牧場を例にswiftのgenericsを使った抽象化について考えてみる)
  - [動物たちの共通点と違う点を探す](#動物たちの共通点と違う点を探す)
  - [ポリモーフィズムとは？](#ポリモーフィズムとは)
  - [ポリモーフィズムには色々なパターンがある](#ポリモーフィズムには色々なパターンがある)
    - [オーバーロード(Overloading)](#オーバーロードoverloading)
    - [サブタイプ(Subtype)](#サブタイプsubtype)
    - [パラメータ化(Parametric)](#パラメータ化parametric)
  - [サブタイプを使ってみる](#サブタイプを使ってみる)
  - [パラメータ化を使ってみる](#パラメータ化を使ってみる)
  - [プロトコルで型の「機能」を表現する](#プロトコルで型の機能を表現する)
  - [プロトコルとは？](#プロトコルとは)
  - [Genericコードを書いてみる](#genericコードを書いてみる)
  - [someを使って書き換える(Swift5.7の新機能)](#someを使って書き換えるswift57の新機能)
  - [opaque typeは引数にも戻り値にも使える](#opaque-typeは引数にも戻り値にも使える)
  - [opaque typeの型推論](#opaque-typeの型推論)
  - [opaque result typeの型推論](#opaque-result-typeの型推論)
  - [Farmにsomeを使う](#farmにsomeを使う)
  - [複数回opaque typeを参照する場合は型パラメータを使う](#複数回opaque-typeを参照する場合は型パラメータを使う)
  - [feedメソッドを実装する](#feedメソッドを実装する)
  - [anyを使ってすべての動物にエサを与える](#anyを使ってすべての動物にエサを与える)
  - [型消去とは？](#型消去とは)
  - [feedAllメソッドにanyを使ってみる](#feedallメソッドにanyを使ってみる)
  - [someを使ってanyの箱を開ける(Swift5.7)](#someを使ってanyの箱を開けるswift57)
  - [some とanyの違いまとめ](#some-とanyの違いまとめ)
  - [デフォルトではsomeを使う](#デフォルトではsomeを使う)
  - [ここまでのまとめ](#ここまでのまとめ)
  - [より応用的な例](#より応用的な例)
  - [anyキーワードとassociatedtypeを持ったプロトコルの関係について](#anyキーワードとassociatedtypeを持ったプロトコルの関係について)
  - [すべての動物から物を生産する](#すべての動物から物を生産する)
  - [生産ポジションではassociatedtypeの制約に基づいて型の上限まで型情報を消去する(Swift5.7)](#生産ポジションではassociatedtypeの制約に基づいて型の上限まで型情報を消去するswift57)
  - [消費ポジションのassociatedtypeにanyは使えない](#消費ポジションのassociatedtypeにanyは使えない)
  - [associatedtypeの型消去の挙動は既存の`Self`を返すプロトコルメソッドに似ている](#associatedtypeの型消去の挙動は既存のselfを返すプロトコルメソッドに似ている)
  - [ここまでのまとめ](#ここまでのまとめ-1)
  - [opaque result typeを使ったちょうど良いカプセル化を行う方法](#opaque-result-typeを使ったちょうど良いカプセル化を行う方法)
  - [opaque result typeを使う](#opaque-result-typeを使う)
  - [primary associated typesを使う(Swift5.7)](#primary-associated-typesを使うswift57)
  - [同じ型要件を設定して具象型の関係性を抽象化してモデリングする方法](#同じ型要件を設定して具象型の関係性を抽象化してモデリングする方法)
  - [エサを作って収穫する](#エサを作って収穫する)
  - [抽象化する](#抽象化する)
  - [それぞれの関係について考えてみる](#それぞれの関係について考えてみる)
  - [型同士の関係が保証されていない](#型同士の関係が保証されていない)
  - [プロトコルの定義が抽象的すぎる](#プロトコルの定義が抽象的すぎる)
  - [応用例のまとめ](#応用例のまとめ)

## SwiftにおけるGenericsとは？

コードの成長に応じて増す複雑さに対応するために必須な、Swiftで抽象コードを書くための土台となる機能

## 抽象化するとは？

具体的な実装詳細と概念を分けること

## 牧場を例にSwiftのGenericsを使った抽象化について考えてみる

```swift
// Cow(牛): 干し草を食べる
struct Cow {
  func eat(_ food: Hay)  { ... }
}

// Hay(干し草): 牛のえさ
struct Hay {
  // 干し草を作るアルファルファを育てるstaticメソッド
  static func grow() -> Alfalfa {
    Alfalfa()
  }
}

// Alfalfa: アルファルファ: 干し草の原料
struct Alfalfa {
  // インスタンスから干し草を収穫する
  func harvest() -> Hay {
    Hay()
  }
}

// Farm(牧場): 牛に干し草をあげる
struct Farm {
    func feed(_ animal: Cow) { ... }
}
```

ここに、もっと動物を増やす。


```swift
// Horse(馬): Carrot(ニンジン)を食べる
struct Horse {
  func eat(_ food: Carrot)  { ... }
}

// Chicken(ニワトリ): Grain(穀物)食べる
struct Chicken {
  func eat(_ food: Grain)  { ... }
}
```

牧場でこの動物たちを育てたい。


`Farm`の`eat`をオーバーロードしてみる。

```swift

struct Farm {
    func feed(_ animal: Cow) { ... }
    func feed(_ animal: Horse) { ... }
    func feed(_ animal: Chicken) { ... }
}
```

これだと、動物が増えるたびに似たメソッドを増やさなくてはならない。

## 動物たちの共通点と違う点を探す

別のアプローチとして動物たちの共通点と違う点を考えてみる。

共通点: みんな何かしらの食べ物を食べる(`eat`)  
違う点: それぞれ食べる物が違う+食べ方も違うかもしれない

```swift
struct Cow {
    func eat(_ food: Hay)  {
        // もぐもぐ食べる
    }
}

struct Horse {
    func eat(_ food: Carrot)  {
        // むしゃむしゃ食べる
    }
}

struct Chicken {
    func eat(_ food: Grain)  {
        // つついて食べる
    }
}
```

そこで、動物に応じた`eat`メソッドを`feed`メソッドから呼び出せるような抽象コードを書く。

これをポリモーフィズムと呼ぶ。

## ポリモーフィズムとは？

異なる型で異なる振る舞いができるように抽象化すること。少ないコードでも使われ方によって複数の挙動を実現することができる。

## ポリモーフィズムには色々なパターンがある

ポリモーフィズムには3つのパターンがある

### オーバーロード(Overloading) 

さっき出てきた`Farm`が複数の`feed`を持つようなもの。「その場しのぎの抽象化」と呼ばれ、本当の意味での解決になっていない。結局コードがどんどん増える。

### サブタイプ(Subtype) 

実行時にスーパータイプで定義されたコードが呼ばれた際、そのサブタイプに応じて挙動を変える。`class`のサブクラスなど。

### パラメータ化(Parametric)

型パラメータを使って、一つの小さなコードを異なる型で動くようにする。`Generics`がこれに当たる。

## サブタイプを使ってみる

下記の例を考えてみる。

```swift
class Animal {
    func eat(_ food: ???)  { fatalError("Subclass must implement `eat`") }
}

class Cow: Animal {
    override func eat(_ food: Hay)  {}
}

class Horse: Animal {
    override func eat(_ food: Carrot)  {}
}

class Chicken: Animal {
    override func eat(_ food: Grain)  {}
}
```

`Animal`というスーパークラスを定義。それぞれの動物`struct`を`class`に変える。これらは`Animal`を継承しての`eat`メソッドを`override`しなければならない。`Animal`型の`eat`メソッドを呼ぶと具象型の`eat`メソッドが呼ばれる。

この時`Animal`の`eat`メソッドの`food`引数の型は何だろうか？また、他にも色々問題がある。

- 他のインスタンスと状態を共有する必要がないのに参照型である
- `override`が必要、実装忘れると実行時にcrashする
- もっと大きな問題として、クラス階層では、スーパークラスでサブクラスが異なる型の食べ物を食べるということが表現できない。方法としては、`Animal`の`food`の引数を`Any`にすることはできるが、サブクラスで正しい型を受け取ったかどうかを実行時に判断しなければならない。新しいバグを生むリスクがある

## パラメータ化を使ってみる

`Animal`に型パラメータを追加する。そうすると型安全になる。しかしこれはちょっと不自然。なぜならば食べることは`Animal`の本来の目的ではないのに、`Animal`を参照する全箇所で型パラメータの型を指定しなければならない。もっと型パラメータが増えると指定するパラメータも増えてもっと面倒になる。

根本的な問題は、`class`がデータ型であるということ。スーパークラスを複雑にして、具象型に関する抽象的なアイデアを表現しようとしている。

## プロトコルで型の「機能」を表現する

代わりに、抽象化した形で型の「機能」を表現できるような言語構造が必要。

動物たちには2つの共通の機能がある

- 特定の種類の食べ物と関係がある
- その食べ物を食べる

Swiftではプロトコルを使って、この2つの機能を持つインターフェイスを構築できる。

## プロトコルとは？

そのプロトコルに準拠する型の「機能」を記述できる抽象化ツール。具体的な実装詳細と抽象を分離できる。

ここで`Animal`プロトコルを作成する。

```swift
protocol Animal {
    associatedtype Food
    func eat(_ food: Food)
}
```

特定の種類の食べ物との関係は`associatedtype`で表現している。`associatedtype`は型パラメータと同様に具象型のプレースホルダになる。`associatedtype`は特定のプロトコルに準拠することで`associatedtype`を特定のものにしている。これによって特定の動物のインスタンスは必ず同じ種類の食べ物と関係があることが保証されている。

その食べ物を食べるという行為は`eat`メソッドで表現している。引数にその動物に特定の食べ物を受け取る。`Animal`プロトコルに準拠する型はこのメソッドを実装しなければならない。

具象型の宣言や`extension`でプロトコルへの準拠を指定できる。`class`以外にも`struct`や`enum`、`actor`にも使える。

`associatedtype`の`Food`は具象型のメソッドからコンパイラが推論できる。typealiasで明示的に指定もできる。

## Genericコードを書いてみる

`Farm`の`feed`メソッドで`Animal`プロトコルを使う。


```swift
struct Farm {
    func feed<A: Animal>(_ animal: A) { ... }
}
```

型パラメータを使うと呼び出されたときに具象型に変わる(実際はコンパイル時)。型パラメータには自由に名前を付けられる。型パラメータはメソッド内で何度も参照できる。

型の横に`Animal`プロトコルへの準拠を指定する。

もしくは`where`句も使える。

```swift
struct Farm {
    func feed<A>(_ animal: A) where A: Animal { ... }
}
```

## someを使って書き換える(Swift5.7の新機能)

これは強力だが、多くのジェネリック関数では必要ない。特に複数回参照する必要がない場合など、このシンタックスは余分に複雑に見える。
そこでSwift5.7では、同じことを`some`を使って表現できるようになった。

```swift
struct Farm {
    func feed(_ animal: some Animal) { ... }
}
```

`some`は特定の型を示すキーワード。この場合、特定の型は`Animal`プロトコルに準拠しなければいけないことを表す。
こちらの方が余分なシンタックスが減ってより直感的に書くことができる。

SwiftUIを使ったことがある場合は`body`プロパティの戻り値に`some View`とあるのを見たことがあるかもしれない。これは戻り値が`View`プロトコルに準拠する特定の型を返すことを表している。具象型については何も知らない。このような型をopaque typeと呼ぶ。opaque typeの内部の型はその値の範囲で固定され、この値にアクセスする際は同じ型であることが保証されている。型パラメータも`some`キーワードもopaque typeを定義している。

```swift
some Animal
<T: Animal>
```

## opaque typeは引数にも戻り値にも使える

opaque typeは引数にも戻り値にも使え、現れる位置によって、誰がその型を決め、誰に抽象化された形で見えるのかが変わる。これは関数の→の左右で分かれる。

```swift
     Parameter position  | Result position
func getValue(Parameter) -> Result
```

型パラメータの場合は常に呼び出し側にあるので、呼び出し側が決める。基本的にはopaque typeに値を与えた側が内部の具象型を決める。

## opaque typeの型推論

もう少し掘ってみる。ローカル変数にopaque typeを使用する場合、代入の右側の値から内部の値を推論する。初期値は必要。

```swift
let animal: some Animal = Horse()
```

その変数のスコープで型は固定される。

```swift
var animal: some Animal = Horse()
anima = Chicken() ❌
```

メソッドの引数の場合は、その引数のスコープ内で固定されていればよく、違う型を引数に入れることができる。

```swift
func feed(_ animal: some Animal)

feed(Horse())
feed(Chicken())
```

※ 引数に`some`が使えるようになったのはSwift5.7から。

## opaque result typeの型推論

戻り値のopaque result typeは実装側が返す値から型が推論される。
この値はどこからでも使われる可能性があるため、この値のスコープはグローバルになる。つまり、全実行パスで同じ型を返す必要がある。

```swift
// ❌
func makeView(for farm: some Farm) -> some View {
    if condition {
        return FarmView(farm)
    } else {
        return EmptyView()
    }
}
```

※ SwiftUIの`ViewBuilder`は内部の型を同じにして解決している。

```swift
@ViewBuilder
func makeView(for farm: some Farm) -> some View {
    if condition {
        FarmView(farm)
    } else {
        EmptyView()
    }
}
```

## Farmにsomeを使う

`feed`メソッドでは、引数以外で参照しているところがないので`some`を使うことができる。

```swif
struct Farm {
    func feed(_ animal: some Animal) { ... }
}
```

## 複数回opaque typeを参照する場合は型パラメータを使う

例えば、`Animal`プロトコルにもう一つ`associatedtype`(`Habitat`)を追加する。
※ Habitat=住まい

```swift
protocol Animal {
    associatedtype Food
    associatedtype Habitat
    func eat(_ food: Food)
}
```

動物ごとに特定の住まいを建築したい。

```swift
struct Farm {
    func feed(_ animal: some Animal) { ... }
    func buildHome<A>(_ animal: A) -> A.Habitat where A: Animal { ... }
}
```

この場合、戻り値の型は特定の動物の型に依存しており、引数と戻り値に型パラメータ`A`を使う必要がある。

ジェネリック型でも型パラメータを使うことが多い。

```swift
struct Silo<Material> {
    private var storage: [Material]

    init(storage materials: [Material]) {
        self.storage = materials
    }
}
```

ストアドプロパティに使うとメンバワイズイニシャライザでも使うことになる。型パラメータを使うとこの型を参照する時に型の指定が必要になるが、opaque typeはこのジェネリック型の使い型を明確にするために必須である。

```swift
var hayStorage: Silo<Hay>
```

## feedメソッドを実装する

エサを表す`AnimalFeed`プロトコルと`Corp`プロトコルを追加する。

```swift
protocol AnimalFeed {
    associatedtype CropType: Crop where CropType.Feed == Self
    static func grow() -> CropType
}

protocol Crop {
    associatedtype Feed: AnimalFeed where Feed.CropType == Self
    func harvest() -> Feed
}

protocol Animal {
    associatedtype Feed: AnimalFeed
    ...
}
```

※ `AnimalFeed`と`Crop`の詳細は[より応用的な例](#より応用的な例)以降で紹介。ここでは特定の動物の型からその動物のエサを生産することができることが示されている。

エサの`associatedtype`から育てる作物の型にアクセスするのに`some Animal`引数が使える。

```swift
struct Farm {
    func feed(_ animal: some Animal) {
        let crop = type(of: animal).Feed.grow()
        let produce = crop.harvest()
        animal.eat(produce)
    }
}
```

これは、動物の型が決まっているので、複数のメソッドを超えて、作物やエサ、動物の関係をコンパイラが知ることができる。不正な型を渡そうとしてもコンパイラが防いでくれる。

## anyを使ってすべての動物にエサを与える

今度は動物みんなにまとめてエサを与えたい。そこで`Array`を受け取る`feedAll`メソッドを追加する。

`Element`が`Animal`プロトコルに準拠している必要があることはわかるが、異なる動物を一つの配列で扱いたい。`some`の場合は、内部の型がすべて同じでないといけないので、今回のケースでは使えない。


![someは同じ型しか扱えない](../images/Swift%20Generics%20牧場物語/some_all_same_type.png)


そこで`any`を使う。

```swift
any Animal
```

`any`はプロトコルに準拠する異なる型を動的に扱うことができる。`some`と同じでプロトコルへの準拠の要件の前に現れる。`any`自体が一つの型で動的に異なる動物を内部に保持できる。これを使うことで値型のサブタイプを実現できる。

`any`は特別なストレージをメモリに持つ。箱のようなものをイメージするとわかりやすい。

![anyは箱のイメージ](../images/Swift%20Generics%20牧場物語/any_box.png)

値が小さければ箱の中に直接その値を収めるが、値が大きいとその値へのポインタが入る。

![anyは値が小さければ箱に直接収めるが大きいとポインタを格納する](../images/Swift%20Generics%20牧場物語/any_box2.png)

このように異なる型を同じものとして扱うことを型消去(type eraser)と呼ぶ。

## 型消去とは？

コンパイル時に具象型は消去され、実行時にしか内部の型の情報はわからなくなる。上記の図の例だと外から見ると同じ型だけど、実際の動物の型は異なる。

型消去は異なる型の値の型レベルの違いをなくす。実際は異なる型でも同じ型として交換可能。

## feedAllメソッドにanyを使ってみる

```swift
struct Farm {
    func feedAll(_ animals: [any Animal]) {}
}
```

※ `associatedtype`を持つプロトコルに`any`キーワードが使えるようになったのはSwift5.7から(実はSwift5.6から`any`自体は使える)

ここで問題が発生する。

`feedAll`の中で`eat`メソッドを使いたいがエラーになる。これは型消去で型レベルでの違いを消去しているため、特定の型と`associatedtype`との関係も消えてしまっているため。

```swift
struct Farm {
    func feedAll(_ animals: [any Animal]) {
        for animal in animals {
            animal.eat(...) // エサの型が特定できない
        }
    }
}
```

## someを使ってanyの箱を開ける(Swift5.7)

解決するためには動物と食べ物の関係を戻してあげる必要がある。そのために再び`some`を使う。`any Animal`の`eat`を呼ぶ代わりに、`some Animal`を受け取る`feed`メソッドを呼ぶ。

```swift
struct Farm {
    func feed(_ animal: some Animal) {
        let crop = type(of: animal).Feed.grow()
        let produce = crop.harvest()
        animal.eat(...)
    }

    func feedAll(_ animals: [any Animal]) {
        for animal in animals {
            feed(animal)
        }
    }
}
```

`some`と`any`は違う型だが、`some Animal`引数に`any Animal`を直接渡すことで、コンパイラが`any`の箱を開けて、`any Animal`のインスタンスを`some Animal`に変換してくれる。

![anyとsomeは違う型](../images/Swift%20Generics%20牧場物語/any_some.png)


この引数のスコープ内では型がわかるため、内部の型のメソッドや`associatedtype`などにアクセスできる。

こうすることで`any`と`some`の間をスムーズに移動できる。

必要なときに柔軟なストレージを選択できると同時に、関数のスコープ内で内部の実際の型を再び扱うことができるため便利。また、ほとんどの場合、箱の中身を開くことはコンパイラが自動でやるため、考える必要はない。

## some とanyの違いまとめ

|    |  some  |  any  |
| ---- | ---- | ---- |
|  内部の型は固定  |  ⭕️  |  ❌  |
|  型同士の関係を保証  |  ⭕️  |  ❌  |

## デフォルトではsomeを使う

`any`は箱の分、メモリやパフォーマンスに影響がある。まずは`some`を使い、`any`は意図的に必要な場合のみに使う。

## ここまでのまとめ

- 具体的な実装から始める
- 機能が増えていくにつれて異なる型で重複があることに気が付く
- そこから共通の機能を特定し、プロトコルでその抽象化する
- `some`と`any`を使用して抽象コードを記述し、より表現力のあるコードが書けるようになった
- `some`をデフォルトにする

## より応用的な例

ここからは、具象型を抽象化して型同士の関係をモデリングするための良い応用テクニックを学ぶ。

大きく分けて3つ紹介する。

- 戻り値の型消去を例に`any`キーワードと`associatedtype`を持ったプロトコルの関係について
- opaque result typeを使ったちょうど良いカプセル化を行う方法
- 同じ型要件を設定して具象型の関係性を抽象化してモデリングする方法

## anyキーワードとassociatedtypeを持ったプロトコルの関係について

時間は進み、大事にエサをあげて育てた動物たちは、物を産むことができるようになった。

先ほどの`Animal`プロトコルに加えて、新しく`Food`プロトコルと、`Food`に準拠する`Egg`と`Milk`を追加する。

```swift
protocol Animal {}
struct Chicken: Animal {
    func produce() -> Egg
}
struct Cow: Animal {
    func produce() -> Milk
}

protocol Food {}
struct Egg: Food {}
struct Milk: Food {}
```

`Chicken`は卵を産み、`Cow`には牛乳を産む。これを抽象化するために、`Animal`に`produce`メソッドを追加する。

```swift
protocol Animal {
    associatedtype CommodityType: Food

    func produce() -> CommodityType
}
```

`produce`メソッドは異なる型を返す必要があるので`associatedtype`(CommodityType)を使って`Food`に準拠する特定の型を返すように宣言している。

図にすると下記のようになる。

![AnimalとCommodityType](../images/Swift%20Generics%20牧場物語/animal_commoditytype.png)
![ChickenとEgg](../images/Swift%20Generics%20牧場物語/chicken_egg.png)
![CowとMilk](../images/Swift%20Generics%20牧場物語/cow_milk.png)

`Self`は`Animal`に準拠する具象型で、`Food`に準拠する`CommodityType`型と関係がある。`Chicken`は`Animal`に準拠して`Food`に準拠する`Egg`と関係がある。`Cow`は`Milk`と関係がある。

## すべての動物から物を生産する

牧場にはさまざまな動物がいるので、すべての動物から物をまとめて生産することができるようにする。

```swift
struct Farm {
    var animals: [any Animal]
    func produceCommodities() -> [any Food] {
        animals.map { animal in 
            animal.produce() 
        }
    }
}
```

内部では`Animal`を`map`して`produce`メソッドでそれぞれの物を生産している。

シンプルに見えるが、型消去によって`any`の内部の型情報は消えてしまっている。この場合、この`animal`が生産する`associatedtype`の具象型はわからない状態になっている。

## 生産ポジションではassociatedtypeの制約に基づいて型の上限まで型情報を消去する(Swift5.7)

そこでコンパイラはこれらの型を`associatedtype`の制約に基づいた上限の型にまで型消去する。

プロトコルのメソッドの戻り値の型に`associatedtype`が出てきた場合、「生産ポジション」と呼ばれる。今回の場合だと、すべて`Food`に準拠していることはわかるため、`any Animal`の`produce`メソッドの戻り値の型は`any Food`になる。

![any Animalとany Food](../images/Swift%20Generics%20牧場物語/anyAnimal_anyFood.png)

例えば、`any Animal`が`Cow`の場合、`Cow`は`Milk`を返すが、`Milk`は`any Food`という箱の中に入っており、この`any Food`が`Animal`の`associatedtype`の`CommodityType`の上限であることはわかる。これは`Animal`に準拠するすべての型で安全に扱える。

## 消費ポジションのassociatedtypeにanyは使えない

一方で、`associatedtype`がメソッドやイニシャライザの引数に現れた場合を考える。`eat`メソッドがこれに当たる。`Animal`の`associatedtype`の`FeedType`が引数の位置にあるが、これを「消費ポジション」と呼ぶ。

ここに`any`は使えない。これは[feedAllメソッドにanyを使ってみる](#feedallメソッドにanyを使ってみる)で紹介した通り、別の型を入れてしまう可能性があるため、型消去した上限の型を具象型が必要な引数に渡すのは安全ではない(=コンパイル時に保証できない)。

## associatedtypeの型消去の挙動は既存の`Self`を返すプロトコルメソッドに似ている

この`associatedtype`の型消去の挙動は、既に存在している言語機能にとても似ている。

```swift
protocol Cloneable: AnyObject {
    fun clone() -> Self
}
```

`any Cloneable`の値からこのメソッドを呼んだ場合、戻り値の`Self`は上限にまで型消去される。`Self`型の上限はプロトコル自身なので新しい`any Cloneable`の値を戻す。

![AnimalとCommodityType](../images/Swift%20Generics%20牧場物語/any_cloneable.png)

Swift5.7からは`associatedtype`を持つプロトコルでも同じことができるようになった。

## ここまでのまとめ

- `associatedtype`を持つプロトコルでも`any`でもサポートするようになった
- 生産ポジションに`associatedtype`を使うと、上限まで型消去される

## opaque result typeを使ったちょうど良いカプセル化を行う方法

ここまで関数のインプットを抽象化することの効果を見てきたが、関数のアプトプットを抽象化する場合にも大変効果的。具体的な戻り値の型を抽象化して変更に強い形にする。

新しい希望として、節約のためにエサはお腹が空いた動物だけにあげたい。

`Animal`に`isHungry`プロパティを追加。`hungryAnimals`はプロパティに切り出して`filter`で`true`のものだけを抽出して新しい`any Animal`を作成する。

```swift
extension Farm {
    var hungryAnimals: [any Animal] {
        animals.filter { $0.isHungry }
    }

    func feedAnimals() {
        for animal in hungryAnimals {
            ...
        }
    }
}
```

`feedAnimals`メソッドの中でこの`Array`をループしているが、この`Array`は関数内で完結して一回しか使われない。今の形だと、もし大量に動物がいた場合、一時メモリを大量に消費して効率がよくない。

そこで`lazy`を使う。

```swift
extension Farm {
    var hungryAnimals: LazyFilterSequence<[any Animal]> {
        animals.lazy.filter(\.isHungry)
    }
}
```

しかし、`LazyFilterSequence`でラップされているため、型が変わってしまう。呼び出し側にとってこれは不要な情報で不必要に実装の詳細が見えてしまっている。

## opaque result typeを使う

そこでopaque result typeの`some Collection`とすることで、`Collection`に準拠する特定の型として利用できる。

```swift
extension Farm {
    var hungryAnimals: some Collection {
        animals.lazy.filter { $0.isHungry }
    }

    func feedAnimals() {
        // (some Collection).Element
        for animal in hungryAnimals {
            ...
        }
    }
}
```

しかし、こうすると、内部の`Element`の情報も消えてしまい、`any anyAnimal`ということを知らないとただループするだけしかできなくなってしまう。抽象化をしすぎてしまっている。

## primary associated typesを使う(Swift5.7)

そこでSwift5.7で登場したprimary associated typesを使って、この`some Collection`に制約を追加する。

```swift
some Collection<any Animal>
```

この機能によってopaque result typeに型パラメータを指定できるようになった。こうすると`Element`が`any Animal`だとわかってちょうど良い抽象化ができる。

```swift
extension Farm {
    var hungryAnimals: some Collection<any Animal> {
        animals.lazy.filter { $0.isHungry }
    }

    func feedAnimals() {
        // any Animal
        for animal in hungryAnimals {
            ...
        }
    }
}
```

primary associated typeを使うには宣言の中で設定する必要がある。

```swift
protocol Collection<Element>: Sequence {
    associatedtype Element
    ...
}
```

標準ライブラリでは多くの型が対応済。独自の型でも定義できる。2つ以上のprimary associated typeも指定できる。

`Collection`の`Element`のように、呼び出し側が型を決めるようなものに使うと効果的。逆に`Iterator`などのような実装の詳細を見せたくないものには使わない方が良い。

ジェネリクスの`Array`や`Set`と`Collection`のprimary associated typesは対応していて、ここで、`Collection`の`Element`のprimary associated typesは、`Array`および`Set`の`Element`ジェネリックパラメータで実装されていることがわかる。これは、両方とも`Collection`に準拠する標準ライブラリによって定義された2つの具象型。

このprimary associated typeは`any`にも使うことができる。

```swift
any Collection<Element>
```

Swift5.7以前ではtype eraserと呼ばれる`AnyXX`といった独自の型を定義して、ジェネリクスと組み合わせる必要があったが、Swift5.7で制約付きの存在型として言語機能になった。

例えば、さっきの`lazy`で`lazy`にするかしないかの選択をしたい場合、`some Collection`だと同じ型しか返せない。

```swift
extension Farm {
    var hungryAnimals: some Collection<any Animal> { // ❌
        if isLazy {
            return animals.lazy.filter { $0.isHungry }
        } else {
            return animals.filter { $0.isHungry }
        }
    }
}
```

これを`any`にすることで動的に型を返すことができる。

```swift
extension Farm {
    var hungryAnimals: any Collection<any Animal> { // ⭕️
        if isLazy {
            return animals.lazy.filter { $0.isHungry }
        } else {
            return animals.filter { $0.isHungry }
        }
    }
}
```

## 同じ型要件を設定して具象型の関係性を抽象化してモデリングする方法

抽象化すると複数の異なる型をまとめて扱えるので便利だが、その分複雑になるところもある。opaque typeを扱う上で大事なのは、プロトコル間で使われている型同士の関係を明確にしてコンパイラに伝えてあげること。

## エサを作って収穫する

エサを食べる前に、そのエサを作って収穫することにした。買うより安くて節約になる。動物たちは食べるものが違うので、それぞれ適切な作物を作って収穫する必要がある。

```swift
protocol Animal {
    associatedtype FeedType: AnimalFeed
    func eat(_: FeedType)
}
```

例えば、牛の場合、まず干し草を育てる必要がある。そのためにアルファルファを植えて、収穫して干し草に加工する。すると牛が干し草を食べることができる。

```swift
struct Cow: Animal {
    func eat(_: Hay) { ... }
}

struct Hay: AnimalFeed {
    static func grow() -> Alfalfa { ... }
}

struct Alfalfa: Crop {
    func harvest() -> Hay { ... }
}

let cow = Cow()

let alfalfa = Hay.grow()
let hay = alfalfa.harvest()

cow.eat(hay)
```

ニワトリの場合は、スクラッチを育てる必要がある。そのためにキビを植えて収穫してスクラッチに加工する。するとニワトリがそれを食べることができる。  
※ スクラッチ: 飼育場の地面の上か、干し草と藁の寝床で与えられる穀物のブレンド

```swift
struct Chicken: Animal {
    func eat(_: Scratch) { ... }
}

struct Scratch: AnimalFeed {
    static func grow() -> Miller { ... }
}

struct Miller: Crop {
    func harvest() -> Scratch { ... }
}

let chicken = Chicken()

let miller = Scratch.grow()
let scratch = miller.harvest()

chicken.eat(scratch)
```

## 抽象化する

この２つを抽象化してあらゆる動物を`feedAnimal`メソッドで扱いたい。`feedAnimal`は消費ポジションに`associatedtype`を持つ`eat`メソッドを使いたいので、`some Animal`を引数の型として`any Animal`の箱の中身を開く`feedAnimal`メソッドを呼ぶ。

```swift
extension Farm {
    func feedAnimals() {
        // any Animal
        for animal in hungryAnimals {
            feedAnimal(animal)
        }
    }
    private func feedAnimal(_ animal: some Animal) {
    }
}
```

`Crop`プロトコルを定義する。これは`AnimalFeed`に準拠して`FeedType`という`associatedtype`を持ち`harvest`メソッドでその`FeedType`を返す。

```swift
protocol Crop {
    associatedtype FeedType: AnimalFeed
    func harvest() -> FeedType
}
```

これは、エサを育てるための作物を表現している。

逆に`AnimalFeed`プロトコルには`CropType`という`associatedtype`を追加してそれを返す`static`な`grow`メソッドを定義する。

```swift
protocol AnimalFeed {
    associatedtype CropType: Crop
    static func grow() -> CropType
}
```

これはエサを作るために必要な作物を育てる。

## それぞれの関係について考えてみる

プロトコルは暗黙的に`Self`型を持つ。`AnimalFeed`は`Crop`プロトコルに準拠する`CropType`を`associatedtype`に持つ。`CropType`はさらに`AnimalFeed`に準拠する`FeedType`という`associatedtype`を持つ。こうして無限ループが発生して型が定まらない。

![型関係の無限ループ](../images/Swift%20Generics%20牧場物語/infinite_type_relation_loop.png)
![型関係の無限ループ2](../images/Swift%20Generics%20牧場物語/infinite_type_relation_loop2.png)

`Crop`に関しても同じ。

![型関係の無限ループ3](../images/Swift%20Generics%20牧場物語/infinite_type_relation_loop3.png)
![型関係の無限ループ4](../images/Swift%20Generics%20牧場物語/infinite_type_relation_loop4.png)

## 型同士の関係が保証されていない

これは具象型の時と同じような型同士の関係ができているのかを確認してみる。

```swift
extension Farm {
    private func feedAnimal(_ animal: some Animal) {
        let crop = type(of: animal).FeedType.grow()
        let feed = crop.harvest()
        animal.eat(feed)
    }
}
```

`grow`メソッドは`AnimalFeed`の`static`メソッドで、`AnimalFeed`に準拠する型の値ではなく、`AnimalFeed`に準拠する型から直接呼び出す必要がある。しかし、`AnimalFeed`に準拠する型の名前を書く必要があるが、現状わかっているのは、`Animal`に準拠する特定の型の値だけ。この値から型を取得できる。これは`Animal`に準拠する型が`AnimalFeed`に準拠する`FeedType`と関係を持つことから`FeedType`の`grow`メソッドを呼び出す。では、戻り値はその`FeedType`の`associatedtype`の`Crop`に準拠する`CropType`になる。ではこの`CropType`の`harvest`メソッドの戻り値の型は何だろう？

![間違った型関係](../images/Swift%20Generics%20牧場物語/wrong_type_relation.png)
![間違った型関係2](../images/Swift%20Generics%20牧場物語/wrong_type_relation2.png)

`harvest`メソッドは、`Crop`プロトコルに準拠する`associated type`の`FeedType`を返す。今回だと`(some Animal).FeedType.CropType.FeedType`になる。

![間違った型関係3](../images/Swift%20Generics%20牧場物語/wrong_type_relation3.png)
![間違った型関係4](../images/Swift%20Generics%20牧場物語/wrong_type_relation4.png)

しかし、この型は間違っていて、`eat`メソッドの引数に欲しいのは`(some Animal).FeedType`で`(some Animal).FeedType.CropType.FeedType`ではない。つまり、型の関係が正しくない。

![間違った型関係5](../images/Swift%20Generics%20牧場物語/wrong_type_relation5.png)

結果エラーになる。

```swift
Cannot convert value of type '(some Animal).FeedType.CropType.FeedType' to expected argument type '(some Animal).FeedType'
```

これらのプロトコルの定義は、ある種類のエサから始めてこの作物を育てて収穫し、最初に指定した種類のエサが戻ってくる(これが動物が食べることを期待しているもの)、という関係を保証できていない。

## プロトコルの定義が抽象的すぎる

具象型同士の関係も期待通りに正しくモデリングできていない。

牛で考えてみる。干し草を育てる時にアルファルファを得る。アルファルファを収穫すると、干し草を得る。

![干し草とアルファルファの関係](../images/Swift%20Generics%20牧場物語/hay_alphalfa.png)
![干し草とアルファルファの関係2](../images/Swift%20Generics%20牧場物語/hay_alphalfa2.png)
![干し草とアルファルファの関係3](../images/Swift%20Generics%20牧場物語/hay_alphalfa3.png)

今の状態だと、もし間違ってアルファルファではなくスクラッチを返したとしても、`AnimalFeed`と`Crop`の関係が成り立ってしまう。

![干し草とアルファルファの関係4](../images/Swift%20Generics%20牧場物語/hay_alphalfa4.png)

牛にニワトリのエサを与えてしまう。

今一度`Animal`プロトコルを見てみる。

```swift
protocol AnimalFeed {
    associatedtype CropType: Crop
    static func grow() -> CropType
}

protocol Crop {
    associatedtype FeedType: AnimalFeed
    func harvest() -> FeedType
}
```

ここでの問題は、ある意味`associatedtype`が多すぎること。これらの`associatedtype`のうちの2つが実際には同じであるという事実を示す必要がある。これにより間違った具象型がプロトコルに準拠することを防ぐことができ、`feedAnimal`メソッドに必要な保証を提供する。`where`句で同じ型の要件を記述することで、これらの`associatedtype`間の関係を表すことができる。

```swift
protocol AnimalFeed {
    associatedtype CropType: Crop
        where CropType.FeedType == Self
    static func grow() -> CropType
}

protocol Crop {
    associatedtype FeedType: AnimalFeed
        where FeedType.CropType == Self
    func harvest() -> FeedType
}
```

同じ型要件は、2つの異なるもしくはネストした`associatedtype`が、実際には同じ具象型でなければならないという静的な保証を示す。上記のように同じ型要件を追加すると、`AnimalFeed`プロトコルに準拠する具象型に制約を与える。ここでは`Self.CropType.FeedType`が`Self`と同じであることを宣言した。`AnimalFeed`に準拠する各具象型には、`Crop`に準拠する`CropType`がある。ただし、この`CropType`の`FeedType`は、`AnimalFeed`に準拠する他の型と同時に、元の`AnimalFeed`と同じ具象型。こうすることでネストした`associatedtype`で無限ループが起きる代わりに、すべての関係を関連する`associatedtype`の1つのペアにまとめることができた。

![正しい型同士の関係](../images/Swift%20Generics%20牧場物語/right_type_relation.png)
![正しい型同士の関係2](../images/Swift%20Generics%20牧場物語/right_type_relation2.png)
![正しい型同士の関係3](../images/Swift%20Generics%20牧場物語/right_type_relation3.png)

`Crop`に関しても同様。

![正しい型同士の関係4](../images/Swift%20Generics%20牧場物語/right_type_relation4.png)
![正しい型同士の関係5](../images/Swift%20Generics%20牧場物語/right_type_relation5.png)

`feedAnimal`メソッドをもう一度見てみる。

```swift
extension Farm {
    private func feedAnimal(_ animal: some Animal) {
        let crop = type(of: animal).FeedType.grow()
        let feed = crop.harvest()
        animal.eat(feed)
    }
}
```

`some Animal`から始まり、`AnimalFeed`プロトコルに準拠するその動物の作物の種類がわかる。この作物を育てると、動物のエサになる作物が得られる。今この作物を収穫すると、さらにネストした`associatedtype`を取得する代わりに、この動物が期待する作物を正確に取得し、動物は正しいエサを食べることができる。

![正しい型同士の関係6](../images/Swift%20Generics%20牧場物語/right_type_relation6.png)

これまでをまとめて見てみると、

![型同士の関係まとめ](../images/Swift%20Generics%20牧場物語/type_relation_summary.png)

ニワトリとスクラッチとキビ
牛と干し草とアルファルファ

という具象型同士の関係をプロトコルで表現することができている。

データモデルを理解することにより、同じ型要件を使うことで、これらの異なるネストした`associatedtype`間が同じであることを定義できる。ジェネリックコードは、プロトコルの複数の要件へ複数の呼び出しを連鎖させるときに、これらの関係に依存する場合がある。

## 応用例のまとめ

今回は下記のことを見てきた。

- いつ型消去が安全に行われるのか、いつ型同士の関係が保証されている必要があるか
- 必要な型情報の保持と実装詳細の隠蔽の間のちょうど良いバランスの見つけ方
- `associatedtype`を表すプロトコル全体で、同じ型要件を使用して、一連の具象型間の型関係を識別および保証する方法

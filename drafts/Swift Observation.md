# Swift Observation

- [Swift Observation](#swift-observation)
  - [概要](#概要)
  - [動機](#動機)
    - [既存技術](#既存技術)
      - [KVO](#kvo)
      - [Combine](#combine)
  - [提案内容](#提案内容)
  - [詳細](#詳細)
    - [`Observable`プロトコル](#observableプロトコル)
    - [マクロとの統合](#マクロとの統合)
    - [`TrackedProperties`](#trackedproperties)
    - [`ObservationRegistrar`](#observationregistrar)
    - [`ObservationTracking`](#observationtracking)
    - [`ObservedChanges`と`ObservedValues`](#observedchangesとobservedvalues)
  - [SDKインパクト（SwiftUIのインタラクションをプレビューするもの）](#sdkインパクトswiftuiのインタラクションをプレビューするもの)
  - [ソース互換性](#ソース互換性)
  - [ABI安定性への影響](#abi安定性への影響)
  - [APIレジリエンスへの影響](#apiレジリエンスへの影響)
    - [APIの位置](#apiの位置)
  - [今後の方向性](#今後の方向性)
  - [検討した代替案](#検討した代替案)
    - [関連システム](#関連システム)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

レスポンシブなアプリを作るには、基礎となるデータが変更されたときに表示を更新する機能が必要になることがよくある。オブザーバ(Observer)パターンでは、オブザーバのリストを保持し、特定または一般的な状態の変化を通知することができる。これは、オブジェクトを直接結合せず、潜在的な複数のオブザーバに更新を暗黙的に分配できるという利点がある。監視可能(Observable)なオブジェクトは、そのオブザーバに関する特定の情報を必要としない。

この設計パターンは、多くの言語で知られた方法であり、Swiftは、堅牢で型安全で、実行可能な実装を提供するチャンスがある。この提案は、監視可能な参照が何であるか、オブザーバは何に準拠する必要があるか、そして型とそのオブザーバの間の接続を定義する。


## 動機

Swiftには、すでに監視のためのいくつかのメカニズムがある。これらは、key-value observing(KVO)と`ObservableObject`を含むが、各々に制限がある。KVOは`NSObject`のサブクラスとしか使用できず、`ObservableObject`は、Darwinプラットフォームに制限され、現在のSwiftの同時実行機能(Concurrency)を使用しないCombineを使用する必要があある。これらの既存のシステムから経験を取ることによって、我々は、`NSObject`から継承するものだけでなく、すべてのSwift参照型に適用されるより一般的に有用な機能を構築し、`async`/`await`のような言語機能からの利点を活用してクロスプラットフォームで動作させることができる。

既存のシステムは、多くの動作や特性を正しく理解している。しかし、安全性、性能、表現力のバランスをより良くすることができる領域がいくつもある。たとえば、依存する変更を独立したトランザクションにグループ化することは一般的なタスクだが、これはCombineを使う場合は複雑で、KVOでは未サポートである。実際には、オブザーバは、トランザクションがどのように解釈されるかを指定する機能を備えたトランザクションへのアクセスを望んでいる。

アノテーションは、何が監視可能かを明確にするが、煩雑になることもある。たとえば、Combineでは、型が`ObservableObject`に準拠するだけでなく、監視対象となる各プロパティに`@Published`としてマークする必要がある。さらに、計算プロパティ(computed property)は直接監視することができない。現実には、監視可能な型に監視されないフィールドが存在することは珍しいことである。

この提案を通して、KVOとCombineの両方を参照することで、どのような能力がメリットで、新しいアプローチに取り入れることができるのか、より強固な方法で解決するにはどのような欠点がある可能性があるのか、を説明する。

### 既存技術

#### KVO

Objective-CのKey-Value Observingは、このモデルによく対応していますが、`NSObject`を継承するクラス階層に限定されている。APIはイベントのインターセプトしか提供しないので、変更の通知は`willSet`イベントと`didSet`イベントの間にある。KVOはイベントの粒度に大きな柔軟性を持つが、合成性には欠ける。KVOのオブザーバは`NSObject`を継承し、発生した変更を追跡するためにObjective-Cのランタイムに依存しなければならない。KVOのインターフェイスは、より近代的なSwiftの強く型付けされたKeyPathを利用するために更新されているにもかかわらず、水面下ではまだ文字列で型付けされている。

#### Combine

Combineの`ObservableObject`は、`willSet`/`didSet`イベントの最先端で変更を生成し、すべての値は、値が設定される**前に**配信される。これはSwiftUIでは有用だが、SwiftUI以外で使うには制限があり、その制限に初めて遭遇した開発者が驚くことがある。`ObservableObject`はまた、変更イベントと対話するために、すべての監視されたプロパティを`@Published`としてマークする必要がある。ほとんどの場合、この要件はすべてのプロパティに適用され、開発者にとっては冗長である。`ObservableObject`に準拠した型を書く人は、各プロパティを繰り返し(ほとんど真価を発揮しないまま)アノテートする必要がある。結局のところ、これは、どのプロパティが監視対象か、またはそうでないかについて考えるのにうんざりする。

## 提案内容

体系化されたオブザーバーパターンは、以下の機能をサポートする必要がある。

- 型を監視可能な(observable)ものとしてマークする
- 監視可能な型のインスタンス内の変化を追跡する
- その変化を別の場所(例えば別の型)から監視し、活用する

また、設計・実装は以下の条件を満たす必要がある。

- 監視可能な型は(どれが監視対象かを考える煩わしさを伴わずに)注釈をつけやすいこと
- アクセス制御を尊重すること
- 監視可能な機能を採用することで、最小限の努力で始められること
- 高度な機能は、より複雑なシステムで必要に応じて徐々に開示されること
- 監視は、一度に複数の被監視者を扱えること
- 監視は、他のプロパティを参照する計算プロパティを扱うことができること
- 監視は、外部ストレージへの取得と設定の両方を処理する計算プロパティを扱うことができること
- 監視の完全性は、単一のオブジェクトだけでなく、一塊のトランザクションで機能する必要がある

このようなパターンを実装するためのプロトコル、型、マクロを含む`Observation`という新しい標準ライブラリモジュールを提案する。

主に、型は`@Observable`マクロアノテーションを使用することで、それ自身を監視可能と宣言することができる。

```swift
@Observable public final class MyObject {
    public var someProperty: String = ""
    public var someOtherProperty: Int = 0
    fileprivate var somePrivateProperty: Int = 1
}
```

`@Observable`マクロは、`Observable`プロトコルへの準拠を宣言・実装するもので、監視を処理するための拡張メソッドのセットを含んでいる。`Observable`プロトコルは、変更されたプロパティと対話する3つの方法をサポートしている。イテレーションの中で特定の値の変更を追跡する、特定のアクター分離の中で変更を追跡する、そしてUIと相互運用する。最初の2つのシステムは非UI用途を意図しているが、SwiftUIなどのユーザーインターフェースライブラリは、データの変更がビューに流れることを可能にするために最後のシステムを使用できる。

最も単純なケースでは、クライアントは、与えられたインスタンスのための特定のプロパティへの変更を監視するために、`values(for:)`メソッドを使用することができる。

```swift
func processChanges(_ object: MyObject) async {
    for await value in object.values(for: \.someProperty) {
        print(value)
    }
}
```

これにより、`Observable`型のユーザーは、特定の値やインスタンス全体に対する変更を、変更イベントの非同期シーケンスとして監視することができる。`values(for:)`メソッドは、特定の1つのプロパティに対する変化のみを提供するため、型の安全性を提供する。

```swift
object.someProperty = "hello" 
// await中のループで "hello "を表示する。
object.someOtherProperty += 1
// 何も表示されない
```

`Observable`オブジェクトは、トランザクションでグループ化して変更を提供することもでき、これは中断点(suspension point)間で行われたすべての変更を統合しする。トランザクションは、提供するアクター、またはデフォルトでMainActorに分離して配信される。

```swift
func processTransactions(_ object: MyObject) async {
    for await change in objects.changes(for: [\.someProperty, \.someOtherProperty]) {
        print(myObject.someProperty, myObject.someOtherProperty)
    }
}
```

`ObservableObject`や`@Published`とは異なり、`@Observable型`のプロパティは、個別に監視対象としてマークする必要はない。代わりに、すべての格納プロパティ(stored property)は暗黙のうちに監視対象となる。

読み取り専用の計算プロパティの場合、staticな`dependencies(of:)`メソッドを追加して、監視の一部として追加のKeyPathを要求することができる。これは、KVOがKeyPathへの効果を持つ追加のKeyPathを提供するために使用する機構と同様である。

```swift
extension MyObject {
    var someComputedProperty: Int { 
        somePrivateProperty + someOtherProperty
    }

    nonisolated static func dependencies(
        of keyPath: PartialKeyPath<Self>
    ) -> TrackedProperties<Self> {
        switch keyPath {
        case \.someComputedProperty:
            return [\.somePrivateProperty, \.someOtherProperty]
        default:
            return [keyPath]
        }
    }
}
```

変更を監視するためのアクセスはすべてKeyPathによって行われるため、`public`や`private`といった可視性のキーワードによって、監視できるものとできないものとが決まる。KVOとは異なり、これは特定のスコープでアクセス可能なメンバーのみが監視できることを意味する。この事実は設計にも反映されており、トランザクションは`TrackedProperties`インスタンスとして表現され、変更されたKeyPathの問い合わせは可能だが、それをイテレーションすることはできない。

```swift
// ✅ `someProperty`は一般に公開されている。
object.changes(for: \.someProperty)
// ❌ `somePrivateProperty` は `private`アクセスに制限されている。
object.changes(for: \.somePrivateProperty) 
// ✅ `someComputedProperty`は見えるが、`somePrivateProperty`は返される`TrackedProperties`インスタンスにアクセスできない。
object.changes(for: \.someComputedProperty) 
```
## 詳細

`Observable`プロトコル、`@Observable`マクロ、および一握りのサポート型が`Observation`モジュールを構成している。後述するように、この設計により、このモジュールの利用者は簡単なケースでは簡潔で分かりやすい構文を使用でき、必要に応じて実装の詳細を完全に制御できるようになる。

### `Observable`プロトコル

監視のためのコアプロトコルは`Observable`である。`Observable`準拠型は、変更を登録することで何が監視可能かを定義し、特定のアクターに分離されたトランザクションと個々のプロパティへの変更の非同期シーケンスを提供する。

```swift
protocol Observable {
    /// 指定されたプロパティに対する変更トランザクションの非同期シーケンスを返す
    nonisolated func changes<Isolation: Actor>(
        for properties: TrackedProperties<Self>,
        isolatedTo: Isolation
    ) -> ObservedChanges<Self, Isolation>

    /// 指定されたキーパスに対する変更の非同期シーケンスを返す
    nonisolated func values<Member: Sendable>(
        for keyPath: KeyPath<Self, Member>
    ) -> ObservedValues<Self, Member>

    /// 指定されたキーパスを表す追跡されたプロパティのセットを返す
    nonisolated static func dependencies(
        of keyPath: PartialKeyPath<Self>
    ) -> TrackedProperties<Self>
}
```

これらのプロトコル要件は、準拠する型によって、手動または以下に説明する`@Observable`マクロを使用して実装する必要があります。最初の2つのメソッドは、`ObservationRegistrar`(後述)を使用して変更を追跡し、変更とトランザクションの非同期シーケンスを提供する。

これらのプロトコル要件に加えて、`Observable`型は、監視可能なプロパティへの各アクセスと変更を追跡するという*意味上の*要件を実装しなければならない。この追跡は、`@Observable`マクロを使用することによっても提供され、または登録者の`access`と`withMutation`メソッドを使用して手動で実装することができる。

`Observable`プロトコルは、MainActorに分離されたトランザクションへの便利なアクセス、または単一のキーパスだけを追跡する拡張メソッドも実装している。


```swift
extension Observable {
    /// 指定されたアクターに分離された、指定されたプロパティの変更トランザクションの非同期シーケンスを返します。
    nonisolated func changes<Member, Isolation: Actor>(
        for keyPath: KeyPath<Self, Member>,
        isolatedTo: Isolation
    ) -> ObservedChanges<Self, Delivery>
        
    /// 指定されたプロパティに対する変更トランザクションの非同期シーケンスを、MainActorに分離して返す
    nonisolated func changes(
        for properties: TrackedProperties<Self>
    ) -> ObservedChanges<Self, MainActor.ActorType>
      
    /// 指定されたキーパスに対する変更トランザクションの非同期シーケンスを、メインのアクターに分離して返します。
    public nonisolated func changes<Member>(
        for keyPath: KeyPath<Self, Member>
    ) -> ObservedChanges<Self, MainActor.ActorType>
    
    // デフォルトの実装では、`[keyPath]`を返す
    public nonisolated static func dependencies(
        of keyPath: PartialKeyPath<Self>
    ) -> TrackedProperties<Self>
}
```

すべての`Observable`型は、それ自身の排他性、スレッドセーフ、および/または分離を正しく処理しなければならない。たとえば、2つのプロパティがある種のリンクされた不変条件を持つ場合、そのアトミック性を提供するのは作者の責任である。たとえば、`ImageDescription`という名前の監視可能な型で、`width`と`height`プロパティがロックした状態で更新される必要がある場合、作者は中断点なしに両方のプロパティを割り当てなければならない。

```swift
@Observable class ImageDescription {
    // ...
    
    func BAD_updateDimensions() async {
        self.width = await getUpdatedWidth()
        self.height = await getUpdatedHeight    
    }

    func GOOD_updateDimensions() async {
        let (width, height) = await (getUpdatedWidth(), getUpdatedHeight())
        self.width = width
        self.height = height
    }
}
```

同様に、特定のプロパティの値を監視するシーケンスは、`Observable`型自体が`Sendable`である場合にのみ`Sendable`となる。


### マクロとの統合

実装をできるだけ簡単にするために、`@Observable`マクロは`Observable`プロトコルへの適合性を自動的に合成し、アノテーションされた型を監視可能な型に変換する。`Observable`マクロは、次のような処理を行います。

- `Observable`プロトコルへの準拠を宣言する
- 必要な`Observable`メソッドの要件を追加する
- registrarのプロパティを追加する
- アクセス追跡のための抽象ストレージを追加
- すべての格納プロパティを計算プロパティに変更
- プロパティのイニシャライザを追加(必要ならばデフォルト値も)

マクロで生成されるコードはすべて手動で書くことができるので、よりきめ細かい制御が必要な場合には、開発者は独自の実装を書くことができる。

```swift
@Observable final class Model {
    var order: Order?
    var account: Account?
  
    var alternateIconsUnlocked: Bool = false
    var allRecipesUnlocked: Bool = false
  
    func purchase(alternateIcons: Bool, allRecipes: Bool) {
        alternateIconsUnlocked = alternateIcons
        allRecipesUnlocked = allRecipes
    }
}
```

上記の例のマクロを展開するとこうなる。

```swift
final class Model: Observable {
    internal let _$observationRegistrar = ObservationRegistrar<Model>()
    
    public nonisolated func changes<Isolation: Actor>(
        for properties: TrackedProperties<Model>, 
        isolatedTo: Delivery
    ) -> ObservedChanges<Model, Isolation> {
        _$observationRegistrar.changes(for: properties, isolation: isolation)
    }

    public nonisolated func values<Member: Sendable>(
        for keyPath: KeyPath<Model, Member>
    ) -> ObservedValues<Model, Member> {
        _$observationRegistrar.changes(for: keyPath)
    }

    private struct _$ObservationStorage {
        var order: Order?
        var account: Account?
  
        var alternateIconsUnlocked: Bool
        var allRecipesUnlocked: Bool
    }
  
    private var _$observationStorage: _$ObservationStorage

    init(order: Order? = nil, account: Account? = nil, alternateIconsUnlocked: Bool = false, allRecipesUnlocked: Bool = false) {
        _$observationStorage = _$ObservationStorage(order: order, account: account, alternateIconsUnlocked: alternateIconsUnlocked, allRecipesUnlocked: allRecipesUnlocked)
    }
  
    var order: Order? {
        get { 
            _$observationRegistrar.access(self, keyPath: \.order)
            return _$observationStorage.order
        }
        set {
            _$observationRegistrar.withMutation(self, keyPath: \.order) {
                _$observationStorage.order = newValue
            }
        }
    }
  
    var account: Account? {
        get { 
            _$observationRegistrar.access(self, keyPath: \.account)
            return _$observationStorage.account
        }
        set {
            _$observationRegistrar.withMutation(self, keyPath: \.account) {
                _$observationStorage.account = newValue
            }
        }
    }

    var alternateIconsUnlocked: Bool {
        get { 
            _$observationRegistrar.access(self, keyPath: \.alternateIconsUnlocked)
            return _$observationStorage.alternateIconsUnlocked
        }
        set {
            _$observationRegistrar.withMutation(self, keyPath: \.alternateIconsUnlocked) {
                _$observationStorage.alternateIconsUnlocked = newValue
            }
        }
    }

    var allRecipesUnlocked: Bool {
        get { 
            _$observationRegistrar.access(self, keyPath: \.allRecipesUnlocked)
            return _$observationStorage.allRecipesUnlocked
        }
        set {
            _$observationRegistrar.withMutation(self, keyPath: \.allRecipesUnlocked) {
                _$observationStorage.allRecipesUnlocked = newValue
            }
        }
    }
  
    nonisolated static func dependencies(
        of keyPath: PartialKeyPath<Self>
    ) -> TrackedProperties<Self> {
        [keyPath]
    }
    
    func purchase(alternateIcons: Bool, allRecipes: Bool) {
        alternateIconsUnlocked = alternateIcons
        allRecipesUnlocked = allRecipes
    }
}
```

プロパティがデフォルト値を持たない場合、イニシャライザの対応する引数はデフォルト値を持たない。つまり、次の例では、`init(a: Int, b: Int = 3)`というイニシャライザがマクロ合成されている。

```swift
@Observable final class InitializationSample {
    var a: Int
    var b: Int = 3
}
```

メンバワイズイニシャライザとバッキングストレージは一緒に生成されるため、イニシャライザはそのストレージを初期化することができる。ユーザ定義のイニシャライザは、プロパティを直接初期化しようとするのではなく、生成されたメンバーワイズイニシャライザを呼び出す必要がある。

マクロの制限により、すべてのフィールドは型情報を持たなければならない。この制限があると、型システムが推論された型を検出するメカニズムを将来的に追加した際に、この制限を解除できるようにしたいからである。

```swift
@Observable final class InitializationSample {
    var a: Int
    var b = 3 // これは、エラーを出す: "@Observableでは、プロパティに型アノテーションが必要です。bには、非推論型がありません"
}
```

` willSet`と`didSet`のプロパティオブザベーションを持つプロパティがサポートされる予定である。ここでは`PropertyExample`型の`@Observable`マクロを紹介する。

```swift
@Observable final class PropertyExample {
    var a: Int {
        willSet { print("will set triggered") }
        didSet { print("did set triggered") }
    }
    var b: Int = 0
    var c: String = ""
}
```

...はプロパティaを以下のように変換する。

```swift
var a: Int {
    get {
        _$observationRegistrar.access(self, keyPath: \.a)
        return _$observationStorage.a 
    }
    set {
        print("will set triggered")
        _$observationRegistrar.withMutation(of: self, keyPath: \.a) {
            _$observationStorage.a = newValue
        }
        print("did set triggered")
    }
}
```

`dependencies(of:)`要件の生成された実装は、型の格納プロパティのすべてのリストを作成し、任意の計算プロパティの依存関係のセットを返す。上記の`PropertyExample`型の場合、その実装は次のようになる。

```swift
nonisolated static func dependencies(
    of keyPath: PartialKeyPath<Self>
) -> TrackedProperties<Self> {
    let storedProperties: [PartialKeyPath<Self>] = [\.b, \.c]
    switch keyPath {
    case \.a:
        return TrackedProperties(storedProperties + [keyPath])
    default:
        return [keyPath]
    }
}
```

`@Observable`マクロは、実装が既に存在する場合はその実装を省略するため、必要であれば型の作者はより選択的な実装を提供することができる。

### `TrackedProperties`

型への変更を監視するとき、公に見えないメンバーへの関連した副作用があるかもしれません。`TrackedProperties`タイプは、内部キーパスが、その可視性を超えて公開されることなく、トランザクションに含めることができる。

```swift
public struct TrackedProperties<Root>: ExpressibleByArrayLiteral, @unchecked Sendable {
    public typealias ArrayLiteralElement = PartialKeyPath<Root>
        
    public init()
    
    public init(_ sequence: some Sequence<PartialKeyPath<Root>>)
      
    public init(arrayLiteral elements: PartialKeyPath<Root>...)
      
    public func contains(_ member: PartialKeyPath<Root>) -> Bool
      
    public mutating func insert(_ newMember: PartialKeyPath<Root>) -> Bool
  
    public mutating func remove(_ member: PartialKeyPath<Root>)
}

extension TrackedProperties where Root: Observable {
    public init(dependent: TrackedProperties<Root>)
}
```

### `ObservationRegistrar`

`ObservationRegistrar`は、変更を追跡し、アクセス権を提供するためのデフォルトのストレージである。`@Observable`マクロは、汎用化された機能として、これらのメカニズムを処理するための記録係を合成する。デフォルトでは、この記録係はスレッドセーフであり、コンテナが潜在的にそうであるように`Sendable`でなければならない。したがって、すべてのアクションに対して分離して処理するように設計されなければならない。

```swift
public struct ObservationRegistrar<Subject: Observable>: Sendable {
    public init()
      
    public func access<Member>(
        _ subject: Subject, 
        keyPath: KeyPath<Subject, Member>
    )
      
    public func willSet<Member>(
        _ subject: Subject, 
        keyPath: KeyPath<Subject, Member>
    )
      
    public func didSet<Member>(
        _ subject: Subject, 
        keyPath: KeyPath<Subject, Member>
    )
      
    public func withMutation<Member, T>(
        of subject: Subject, 
        keyPath: KeyPath<Subject, Member>, 
        _ mutation: () throws -> T
    ) rethrows -> T
      
    public func changes<Isolation: Actor>(
        for properties: TrackedProperties<Subject>, 
        isolatedTo: Isolation
    ) -> ObservedChanges<Subject, Isolation>
      
    public func values<Member: Sendable>(
        for keyPath: KeyPath<Subject, Member>
    ) -> ObservedValues<Subject, Member>
}
```

`access`メソッドと`withMutation`メソッドは、トランザクションのアクセスを識別する。これらのメソッドは、`ObservationTracking`システムにアクセスを登録し、オブザーバ用に登録されたトランザクションへの変更を特定する。

### `ObservationTracking`

スコープ付き監視を提供するために、`ObservationTracking`型は、与えられたスコープ内のプロパティへのアクセスをキャプチャし、それらのプロパティのいずれかに最初の変更があったときに呼び出すメソッドを提供する。このAPIは、UIとのインタラクションを構築することができる主要なメカニズムである。特にこれは、`var body: some View`スコープのレンダリングで、特定のプロパティアクセスを与えられたViewの更新を提供するためにSwiftUIによって使用される。より詳細な情報は、「SDKの影響セクション」で後ほど展開する。

```swift
public struct ObservationTracking {
    public static func withTracking<T>(
        _ apply: () -> T, 
        onChange: @autoclosure () -> @Sendable () -> Void
    ) -> T
}
```

`withTracking`メソッドは2つのクロージャを取り、`apply`クロージャ内のプロパティにアクセスすると、その特定のインスタンスのプロパティが`onChange`クロージャに通知される変更に参加する。

```swift
@Observable final class Car {
    var name: String
    var awards: [Award]
}

let cars: [Car] = ...

func render() {
    ObservationTracking.withTracking {
        for car in cars {
            print(car.name)
        }
    } onChange {
        scheduleRender()
    }
}
```

上の例では、`render`関数が各carの`name`プロパティにアクセスしている。そして、いずれかのcarの`name`が変わると、最初の変更時に`onChange`クロージャが呼び出される。しかし、あるcarにawardが追加された場合は、`onChange`は呼び出されない。この設計は、SwiftUIのような暗黙の監視追跡を必要とする用途をサポートし、Viewが関連する変更の場合のみに応答して更新されることを保証する。

### `ObservedChanges`と`ObservedValues`

この2つの非同期シーケンスは、それぞれ`TrackedProperties`インスタンスに基づくトランザクションへのアクセスと、key pathに基づく変更へのアクセスを提供する。この2つのシーケンスは、わずかに異なるセマンティクスを持っている。どちらも、主に非UIシステムでの使用を意図している。

`ObservedChanges`シーケンスは、`Observable`型の`changes(for:isolatedTo:)`を呼び出し、`TrackedProperties`インスタンスとして1つまたは複数のkey pathを渡した結果である。引数に渡した変更を分離するアクター(またはデフォルトのMainActor)は、変更がどのように結合されるかを決定する。この分離アクター上の中断ポイント間の変更は、1つのプロパティであれ、複数のプロパティであれ、1つのトランザクションイベントとして提供されるだけである。シーケンスの要素は`ObservedChange`インスタンスで、*特定の*プロパティが変更されたかどうかを問い合わせることができる。監視された型が`Sendable`の場合、更新されたプロパティへのアクセスを簡単にするために、`ObservedChange`は監視されている主体も含む。

一方、`ObservedValues`シーケンスは、`values(for:)`を呼び出した結果であり、監視するために単一のkey pathを渡す。分離アクターを参照して変更を結合する代わりに、`ObservedValues`は、イテレーション中の中断時に結合された変更値を提供する。イテレータは`Sendable`ではないので、その動作は暗黙のうちに現在のアクターに変更を分離する。

```swift
public struct ObservedChange<Subject: Observable>: @unchecked Sendable {
    public func contains(_ member: PartialKeyPath<Subject>) -> Bool
}

extension ObservedChange where Subject: Sendable {
    public var subject: Subject { get }
}

/// ObservedChangesの非同期シーケンス
public struct ObservedChanges<Subject: Observable, Delivery: Actor>: AsyncSequence {
    public typealias Element = ObservedChange<Subject>
  
    public struct Iterator: AsyncIteratorProtocol {
        public mutating func next() async -> Element?
    }
    
    public func makeAsyncIterator() -> Iterator
}

extension ObservedChanges: @unchecked Sendable where Subject: Sendable { }
@available(*, unavailable)
extension ObservedChanges.Iterator: Sendable { }

/// ObservedValuesの非同期シーケンス
public struct ObservedValues<Subject: Observable, Element: Sendable>: AsyncSequence {  
    public struct Iterator: AsyncIteratorProtocol {
        public mutating func next() async -> Element?
    }
    
    public func makeAsyncIterator() -> Iterator
}

extension ObservedValues: @unchecked Sendable where Subject: Sendable { }
@available(*, unavailable)
extension ObservedChanges.Iterator: Sendable { }
```

`ObservedChanges`は、変更が行われ、利用可能な中断ポイントが分離によって満たされたときに値を出力する。これは、与えられたアクターが実行可能な境界まで、変更をトランザクションでグループ化できることを意味する。一般的な使用例としては、この`AsyncSequence`を、このアクターに分離されていると期待されているアクターと同じアクターで繰り返し実行する。

```swift
@Observable final class ChangesExample {
    @MainActor var someField: Int
    @MainActor var someOtherField: String
}

@MainActor
func handleChanges(_ example: ChangesExample) {
    Task { @MainActor in
        for await change in example.changes(for: [\.someField, \.someOtherField]) {
            switch (change.contains(\.someField), change.contains(\.someOtherField))
            case (true, true):
                print("Changed both properties")    
            case (true, false):
                print("changed integer field")
            case (false, true):
                print("changed string field")
            default:
                fatalError("this case will never happen")
            }
        }
    }
}
```

MainActorの伝播には2つの目的がある。監視対象への安全なアクセスを可能にすることと、同じアクター上の変更の結合である。監視の対象が`Sendable`でない場合、タスクの作成は、アイテムが「アクター境界を越えて送信されている」というConcurrencyの警告を発する。これは`ObservedChanges`非同期シーケンスにも当てはまり、`Sendable`への準拠は監視対象が`Sendable`に準拠していることが条件となる。これは、間違って異なるアクター上で変更がイテレートされる可能性に関する警告を発することを意味する(そして、将来のSwiftリリースでは、これはエラーになる)。

変更を結合する仕組みは、その監視が分離しているアクターの中断に基づく。変更がイテレートされているとき、ある`Observable`内の追跡対象のプロパティの変更は、その監視を分離しているアクターが中断されてから再開されるまでの間は集め続けられる。つまり、この一連の変更の境界は、任意の変更のセット(`willSet`)の最初から、分離しているアクターの中断により始まるトランザクションが終わるまで、を意味する。もし`changes`のイテレーションの範囲が、たとえば上記のようにある特定のアクターに限定されるなら、それらの変更はその前の領域と繋がっていることを意味する。監視対象がアクターの分離に違反していなければ、値はそのアクターのスケジューリング内で次に利用可能なスロットへ反映さるはずである。要するに、MainActorがあれば、発生した変更は実行ループの次のティックまで集められ、その非同期コンテキストでその一連の変更を使用できるようになる。

一方、`ObservedValues`非同期シーケンスは、値の変更を待ち、最後のイテレーション(またはイテレーションの開始点)と中断の間に利用可能な最新の値を出力する。`ObservedChanges`に対する`Sendable`の要件と同様に、`ObservedValues`の非同期シーケンスは条件付きである。つまり、監視対象の主体が`Sendable`である場合のみ、値の`AsyncSequence`は`Sendable`となる。ただし、監視される値のタイプも`Sendable`でなければならないという、もう1つ追加要件がある。

```swift
@Observable final class ValuesExample {
  @MainActor var someField: Int
}

@MainActor
func processValues(_ example: ValuesExample) {
  Task { @MainActor in
    for await value in example.values(for: \.someField) {
      print(value)
    }
  }
}
```

上記の例では、`someField`プロパティがMainActorで連続して変更され、その間に中断がない場合、設定される値の間にawaitしているcontinuationは再開できない。つまり、最新の値のみが出力されることになる。イテレーション処理が別のアクターで行われている場合、イテレータの次の関数を呼び出す間に発生する可能性のある変更も結合される。

注目すべきは、型を`Observable`にしても、個々の値や値のグループへ自動的にアトミック性が提供されないということである。このような変更は、監視される型の実装者が管理する必要があります。この管理は、`Sendable`への準拠性を宣言する際に、型が説明する必要のある責務の1つです。


## SDKインパクト（SwiftUIのインタラクションをプレビューするもの）

`ObservableObject`のような既存のシステムを使用すると、開発者が本当にSwiftUIに深い見解を持っていない限り、驚くような多くのエッジケースがある。監視を公式化することは、理解するために必要な異なるシステムの複雑さを減らすことによって、これらのエッジケースをかなりアプローチしやすくすることができる。

以下は、Frutaのサンプルアプリから引用し、わかりやすくするために修正したものである。

```swift
final class Model: ObservableObject {
    @Published var order: Order?
    @Published var account: Account?
    
    var hasAccount: Bool {
        return userCredential != nil && account != nil
    }
    
    @Published var favoriteSmoothieIDs = Set<Smoothie.ID>()
    @Published var selectedSmoothieID: Smoothie.ID?
    
    @Published var searchString = ""
    
    @Published var isApplePayEnabled = true
    @Published var allRecipesUnlocked = false
    @Published var unlockAllRecipesProduct: Product?
}

struct SmoothieList: View {
    var smoothies: [Smoothie]
    @ObservedObject var model: Model
    
    var listedSmoothies: [Smoothie] {
        smoothies
            .filter { $0.matches(model.searchString) }
            .sorted(by: { $0.title.localizedCompare($1.title) == .orderedAscending })
    }
    
    var body: some View {
        List(listedSmoothies) { smoothie in
            ...
        }
    }
} 
```

`@Published`は、オブジェクトの変更に参加する各フィールドを識別するが、どのフィールドが変更されたかの区別はしない。これは、SwiftUIの観点から、`order`の変更は`hasAccount`を使用するものにも影響を与えることを意味し、残念ながら、追加のレイアウト、レンダリング、更新が作成されることを意味する。本提案のAPIは、`@Published`の繰り返しを減らすだけでなく、SwiftUIのビューコードも単純化することができる！
前の例は、次のように書くことができる。

```swift
@Observable final class Model {
    var order: Order?
    var account: Account?
    
    var hasAccount: Bool {
        return userCredential != nil && account != nil
    }
    
    var favoriteSmoothieIDs: Set<Smoothie.ID> = []
    var selectedSmoothieID: Smoothie.ID?
    
    var searchString: String = ""
    
    var isApplePayEnabled: Bool = true
    var allRecipesUnlocked: Bool = false
    var unlockAllRecipesProduct: Product?
}

struct SmoothieList: View {
    var smoothies: [Smoothie]
    var model: Model
    
    var listedSmoothies: [Smoothie] {
        smoothies
            .filter { $0.matches(model.searchString) }
            .sorted(by: { $0.title.localizedCompare($1.title) == .orderedAscending })
    }
    
    var body: some View {
        List(listedSmoothies) { smoothie in
            ...
        }
    }
} 
```

たとえば、ビュー内のアクセスの追跡監視は、配列、オプション、あるいはカスタムタイプに適用することができる。これは、開発者がより簡単にSwiftUIを利用することができる新しい興味深い方法を開くものである。

これはSwiftUIの潜在的な将来の方向性だが、本提案の一部ではない。


## ソース互換性

本提案は追加的なものであり、既存のソースコードに影響を与えるものではない。

## ABI安定性への影響

本提案は追加的なものであり、既存のABIの安定性に影響を与えるものではない。しかし、関数のインライン化や、この機能のバックポートには影響がある。変更イベントの通知にパフォーマンスが重要であると判断された場合、メソッドはインライン化可能であるとマークされる。

型を監視対象以外から`@Observable`に変更することは、プロパティを格納から計算に変更するのと同じABI的影響を与える(これはABIを破壊するものではない)。`@Observable`を削除すると、計算プロパティから格納プロパティに移行するだけでなく、準拠性も削除される(これはABIの破壊)。

## APIレジリエンスへの影響

本提案は追加的なものであり、既存のAPIレジリエンスに影響を与えることはない。`@Observable`を採用する型は、APIの契約を破ることなく、`@Observable`を削除することはできない。

### APIの位置

このAPIは、Swift言語の一部でありながら標準ライブラリの外にあるモジュールに収容される予定である。このモジュールを使用するには、`import Observation`を使用する必要がある(プレビューの`import _Observation`でも暫定的に使っている)。

## 今後の方向性

最初の実装では、2層以上のコンポーネントを持つキーパスの変更に追従しなかった。たとえば、`\.account`のようなキーパスは動作するが、`\.account.name`は動作しない。この機能は、標準ライブラリがキーパスのコンポーネントを繰り返し処理するメカニズムを提供するようになれば、すぐに可能になる。まだこれを決定する方法がないため、複数のコンポーネントを持つキーパスは、いかなる変更も監視されない。

また、将来的な機能拡張の焦点として、監視可能な`actor`のサポートが挙げられる。この場合、現在アクターには存在しない、キーパスのための特別な処理が必要になる。

格納プロパティ宣言に型を含めるという現在の要件は、マクロがプロパティにセマンティックな型情報を提供できるようになれば、将来的になくなる可能性がある。この場合、`var a = 3`のようなプロパティ宣言が可能になる。

将来のマクロ機能では、`@Observable`マクロによって生成される`dependencies(of:)`実装を改良して、各計算プロパティが依存する格納プロパティをより正確に選択できるようになる可能性がある。

最後に、可変長ジェネリック(Variadic generics)が利用できるようになれば、複数のキーパスを`AsyncSequence`として監視するメカニズムが追加される可能性もある。

## 検討した代替案

以前の検討では、トランザクションを定義する代わりに、オブザーバーとして直接will/didイベントを使用した。これは、より直接的ではあるが、依存するメンバー間に必要な同期をサポートするための適切なメカニズムが提供できなかった。トランザクションの構築は、APIを使用する開発者が、will/didイベントの観点だけで考えるのではなく、必要なトランザクションのモデルを検討することを促すために、余計複雑になると判断された。

別の設計では、コールバックスタイルのオブザーバータイプを構築するために使用できる`Observer`が含まれていた。これは`AsyncSequence`アプローチに取って代わられ、廃止された。

`ObservedChange`型は、条件付きで`Sendable`のみとし、すべてのケースで監視対象へのアクセスを許可することで、`Sendable`要件を緩和することができたが、値へのアクセスは1つのプロパティで行うのが一般的であると考えられているため、`AsyncSequence`がその役割をほとんど果たし、与えられたアクターで複数のフィールドにアクセスする必要がある場合、イテレーションは監視可能な主体への弱参照で行うことができる。

### 関連システム

- [Swift Combine.ObservableObject](https://developer.apple.com/documentation/combine/observableobject/)
- [Objective-C Key Value Observing](https://developer.apple.com/documentation/objectivec/nsobject/nskeyvalueobserving?language=objc)
- [C# IObservable](https://learn.microsoft.com/en-us/dotnet/api/system.iobservable-1?view=net-6.0)
- [Rust Trait rx::Observable](https://docs.rs/rx/latest/rx/trait.Observable.html)
- [Java Observable](https://docs.oracle.com/javase/7/docs/api/java/util/Observable.html)
- [Kotlin observable](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-delegates/observable.html)

## 参考リンク

### Forums

- [SE-0395: Observability](https://forums.swift.org/t/se-0395-observability/64342)

### プロポーザルドキュメント

- [Observation](https://github.com/apple/swift-evolution/blob/main/proposals/0395-observability.md)
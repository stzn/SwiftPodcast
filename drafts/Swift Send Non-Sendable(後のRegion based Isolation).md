# Swift Send Non-Sendable

<!-- 最後にTable of Contentsを入れる -->

## 概要

Swift Concurrencyはメモリを様々なアクターやタスクの"分離ドメイン"に分割する。異なるドメインに分離された計算は同時に実行することができるので、データ競合を防ぐためには、複数のドメインから同時にアクセスできる変更可能な状態がないことが重要である。Swiftの型システムは、分離ドメイン間で通信される深く不変な値への参照のみを許可することで、この分離特性を保証する。深く不変な値だけが安全に`Sendable`プロトコルに準拠することができ、`Sendable`プロトコルに準拠した値だけが分離ドメインを横断することができる。

残念ながら、分離ドメインを横断する全ての値に対して`Sendable`の適合性を要求することは、実際には非常に制限的である。例えば、可変オブジェクトは、あるアクターが構築してから別のアクターに送信することはできない。任意の型の値が、データ競合を発生させることなく安全に送信できるかどうかを決定するフローセンシティブな解析は、このパターンを可能にし、Swiftの並行処理の表現力を大きく向上させる。

現在実験的な機能として利用可能な `SendNonSendable`パスは、そのようなフローセンシティブな分析を実装している。関数スコープ内の全ての`NonSendable`な値を追跡し、分離ドメイン間で送信されることを許可するが、送信された時に"転送された(transferred)"とマークする。転送された値は以後アクセスできなくなる。これにより、送信側ドメインと受信側ドメインでの同時アクセスに起因する潜在的なデータ競合が確実に回避される。この解析の核心は、互いにエイリアスまたは参照し合う可能性のある値を静的な「リージョン」にグループ化することである。値を送信すると、その値自身と、そのリージョン内の他のすべてのエイリアスまたは参照する値が転送されるはずである。

このパスにより、データ競合の自由度や人間工学を犠牲にすることなく、Swiftの並行処理でより柔軟なプログラミングが可能になる。


## 内容

### 動機

次のコードは、そのような状況を示している：

- `NearestNeighbors`は循環が存在するため、安全に`Sendable`にすることができないクラスである
- `NearestNeighbors.init`は非常に高価な処理である
- 最終的に、`NearestNeighbors`インスタンスは`addToDisplay`呼び出しによってユーザに表示される必要がある

このコードは、`NearestNeighbors`クラスを使って位置データをモデル化し、表示するための合理的な試みである

```swift

// データセット内の最近傍の`numNeighbors`と各点を紐付ける位置データの表現
// この戻り値、繋がった点のクラスターが生成され、それぞれの1つのルートが`rootPoints`に格納される
class NearestNeighbors { 
  class DataPoint {
    // この点の`numNeighbors`、つまり最近傍を指す
    var nearestPoints : [DataPoint]
    ...
  }
  
  let numNeighbors : Int
  var rootPoints : [DataPoint]
  ...
}
	
// データセット `data` の各点を`numNeighbors`個の最近傍にある点と紐付け、最近傍グラフを作成する
// 高価な操作
func computeNearestNeighbors(data : LocationData, numNeighbors : Int = 10) -> NearestNeighbors { ... }

// 最近傍グラフをユーザーに表示する
// UI 操作 - `@MainActor` でなければならない
@MainActor func addToDisplay(neighbors : NearestNeighbors) { ... }

// 位置情報を取得し、最近傍グラフを作成し、ユーザーに表示する
func computeAndDisplayData(data : LocationData) async {
  let neighbors = computeNearestNeighbors(data: data)
  await addToDisplay(neighbors) // warning: passing argument of non-sendable type 'NearestNeighbors' into @MainActor-isolated context may introduce data races
}
```

残念ながら、現在のSwiftのstrict concurrencyは、このコードを許可しない。`computeAndDisplayData`はnonisolatedな関数で、`addToDisplay`は`@MainActor` isolatedなので、`await addToDisplay(neighbors)`の呼び出しは分離ドメインを横断し、したがってNonSendableな値を渡すことは許可されていない。コンパイラがこのコードを受け入れるようにするには、3つの選択肢がある:

- strict concurrencyチェックを無効にする(望ましくない)
- `NearestNeighbors`を`Sendable`にする(`let`バインディングのみで循環グラフを作成できないため、不可能)。
- `computeAndDisplayData`を`@MainActor-isolated`にする(近傍グラフの計算中にメインスレッドがハングしてしまうので望ましくない)。

このように、安全性(データ競合の自由度)とパフォーマンスのどちらかを犠牲にすることなく、このグラフのような`NearestNeighbors`オブジェクトを構築し、表示する方法は今のところない。

### 提案内容

人間のプログラマにとって、関数`computeAndDisplayData`は明らかに安全です。Swift Concurrencyは、MainActorが`neighbors`へのアクセスを巡って`computeAndDisplayData`の残りと競合する可能性があるため、`neighbors`が分離ドメインを越えてMainActorに送られるのを防ぐ。しかし、`computeAndDisplayData`は呼び出しの後に再び`neighbors`にアクセスすることはなく、`neighbors`をエスケープすることも許さない。必要なのは、次のような安全な関数の違いを見分けるパスである:

```swift
// この関数は安全である
func computeAndDisplayData(data : LocationData) async {
  let neighbors = computeNearestNeighbors(data: data)
  await addToDisplay(neighbors)
  // `neighbors`には再びアクセスしない
}
```

```swift
// この関数は安全ではない
func computeAndDisplayData(data : LocationData) async {
  let neighbors = computeNearestNeighbors(data: data)
  await addToDisplay(neighbors)
  neighbors.addNewData(receiveNewData()) // `neighbors`に競合アクセス
}
```

もしくは

```swift
// この関数は安全ではない
func computeAndDisplayData(data : LocationData) async -> NearestNeighbors {
  let neighbors = computeNearestNeighbors(data: data)
  await addToDisplay(neighbors)
  return neighbors // `neighbors`をエスケープ
}
```

現在のSwiftのSendableチェックはフローに依存しないので、実装が分離を越えて送信された後、NonSendableな値にアクセスするかどうかを決定することができません。このため、分離を越えてNonSendableな値を送信することを全面的に禁止せざるを得ない。

これが、現在`-enable-experimental-feature SendNonSendable`として提供されている`SendNonSendable`パスの開発の動機となっている。`SendNonSendable`パスは、NonSendableな値を線形に扱います。つまり、分離を越えて送信されるまでは自由に使用することができ、その時点で転送されたものとして扱われ、それ以降の使用でチェックされる。この線形チェックにより、`SendNonSendable`パスは、上記の`computeAndDisplay`の安全なバージョンはデータ競合なしであると判断し、それについて投げられるSendableチェックを抑制し、安全でないバージョンは競合が発生する可能性があると判断し、潜在的な競合について有益な診断を発することができる。

#### リージョン(リージョン)

上記の安全でない例は、単一の値である`neighbors`が分離を越えて送信され、その後アクセスまたはエスケープされるため、明らかに競合している。また、送信された値へのアクセスがキープされた値へのアクセスと競合する可能性があるという、少しわかりにくい競合が発生する例もある。

```swift
func computeAndDisplayLargest(datasets : [LocationData]) async {
  var listOfGraphs : [NearestNeighbors]
  for data in datasets {
    listOfGraphs.append(computeNearestNeighbors(data : data))
  }
  
  let largestGraph = listOfGraphs.max(by: { $0.numRootPoints() > $1.numRootPoints() })
  let smallestGraph = listOfGraphs.min(by: { $0.numRootPoints() > $1.numRootPoints() })
  
  await addToDisplay(largestGraph)
  
  smallestGraph.decreaseClusterSize(1)
}
```

この関数では、`largestGraph`は関数に渡されたデータセットのリストから生成された最大の`NearestNeighbor`グラフを表示しようとする。分離して送信され、その後にアクセスされる単一の値はないように見える。しかし、`largestGraph`と`smallestGraph`が別個の値である保証はなく、同じ値である可能性もある！つまり、`smallestGraph.decreaseClusterSize(1)`を介した`smallestGraph`へのアクセスと変更は、`await addToDisplay(largestGraph)`をMainActorに付与された`largestGraph`へのアクセスと競合する可能性がある。

厳密な線形性は、明らかなエイリアシング(例えば、let x = y)を禁止することで、エイリアシングがそもそも発生しないようにすることでこれを解決し、その戻り値、`min`や`max`のような関数の実装が、データ構造内の他の場所にまだ存在するオブジェクトへの第一級の参照を提供することを防ぐ。要するに、厳密な線形性は「一度に1つの第一級参照」のセマンティクスを強制する。これは不自然なプログラミングモデルであり、特定のAPIの実装を妨げるだけでなく、作業も厄介で面倒("How to write a doubly linked list in Rust "でググってみてください)。

`SendNonSendable`パスが、厳密な値レベルの線形性の代わりに実装することを選んだ解決策は、*線形リージョン*である。リージョンとは、互いにエイリアスや参照する可能性のある値の集まりのことで、別々のリージョンにある2つの値が互いにエイリアスや参照*できない*ように分割される。分離ドメインを超える呼び出しのような、値の所有権を転送するような操作では、リージョン全体の所有権を一度に転送する。例えば、上記の`computeAndDisplayLargest`関数では、値`listOfGraphs`、`largestGraph`、`smallestGraph`は全て同じリージョンとみなされる。`SendNonSendable`コードは、`smallestGraph.decreaseClusterSize(1)`へのアクセスが、`smallestGraph`のリージョンが既に別の分離に転送されているため、潜在的に不正であることを認識することができる。

要約すると、提案された`SendNonSendable`パスは、データ競合の自由度を維持しながら、NonSendableな値を分離を越えて送信することを可能にするという目標を以下の方法で達成した:

- 全てのプログラムポイントでスコープ内の全ての**NonSendableな値**を追跡し、任意にエイリアス化された値を含むことが許されているが、互いにエイリアス化されないことが保証されているリージョンに分割する
  - 例えば、`listOfGraphs`、`largestGraph`、`smallestGraph`がすべて上記の同じリージョンにあることを追跡する
- **NonSendableな値**が分離を横断するプログラムポイントでは、その値を含むリージョンを転送する
  - 例えば、`addToDisplay`の呼び出しの後に、`largestGraph`を含むリージョンが転送されたことをマークする
- 既に転送されたリージョン内の**NonSendableな値**へのアクセス時に診断を生成する
  - 例えば、上記の`smallestGraph`へのアクセスは、その値が転送されたリージョンにあるため禁止する

**NonSendableな値**に注目していることに注意。Sendableな型の値はこのパスでは無視され、現在実装されている型チェックの対象となる。

### 詳細

この提案は、Swiftに新しい構文を追加するものではなく、型を変更するものでもない。既存の動作に対する唯一の変更は、分離ドメインを横断すると判断された呼び出しにNonSendableな引数がある場合に発行される`non_sendable_call_argument`診断の発行を防ぐことである。この診断を先取りすることで、上記のようにnonisolatedコンテキストからMainActor-isolatedコンテキストへといったように、分離ドメインを越えてNonSendableNonSendableな値を渡すことができるようになる。

`TypeCheckConcurrency`のSemaパスは、`non_sendable_call_argument`診断を出す代わりに、`ApplyExprs`を、それが分離を越えるアプリケーションを表しているかどうかで装飾するようになった。`SendNonSendable`必須SILパスは、競合を発生させる可能性のある、分離を越えるアプリケーションのサブセットに関する診断を発行する責任を負う。

そこで重要なのは、SILパスがどのように分離を横断するアプリケーションが競合を引き起こす可能性があるかをどのように判断するかということです。

#### 高レベルな: リージョンに対するシンプルなルール

関数本文内のNonSendableな値はすべて追跡され、どのリージョンに存在するかを決定する。値の初期化に関するいくつかの簡単なルール:

- NonSendableな値のイニシャライザがNonSendableな引数を取る場合、イニシャライザが戻った後、すべての引数のリージョンがマージされ、単一の共有リージョンになる。初期化された値もそのリージョンに存在する
- イニシャライザに引数が渡されなかったり、Sendableな引数のみが渡されたりした場合は、既存の値と共有されていない新しいリージョンが、初期化された値のために作成される

これらのリージョンは純粋に静的な抽象化であり、新しいリージョンを割り当てるということは、そのリージョンに対して新しい識別子を選択することだけを意味し、何らかの動的な割り当てを実行するわけではないということを覚えておくことが重要である。以下のコードは、リージョン作成の動作を示している。

```swift
class NonSendable {
  var x : SendableType
  var y : OtherNonSendableType?
  
  init(_ x : SendableType, _ y : OtherNonSendableType? = .none) {
    self.x = x; self.y = y
  }
}

// リージョン分割開始: 何もなし

let x = SendableType() // xはSendableなのでリージョンを持っていない
let y = OtherNonSendableType() // yは新しいリージョンを取得
let ns1 = NonSendable(x) // ns1は新しいリージョンを取得
let ns2 = NonSendable(x, y) // ns2はyと同じリージョンを取得

// リージョン分割終了: {ns1}, {y, ns2}
```

一般的な関数は、実際には同じセマンティクスを示す:NonSendableな引数は共通のリージョンにマージされ、NonSendableな戻り値はそのリージョンか、NonSendableな引数がない場合は新しいリージョンから得られる。これは、さらなる情報がなければ、呼び出された関数が、渡されたNonSendable引数間の参照を作成すると仮定しなければならないため、必要なことである。

```swift
func compareBoxContents(_ box0 : Box, _ box1 : Box) {
  if (box0.contents == box1.contents) {
    print("the same")
  } else {
    print("different")
  }
}

func reassignBoxContents(_ box0 : Box, _ box1 : Box) {
  box0.contents = Contents()
  box1.contents = box0.contents
}

// 初期化時、box0 と box1 は別々のリージョンを占める
let (box0, box1) = Box(), Box()

compareBoxContents(box0, box1) // この呼び出しは、box0とbox1のリージョンをマージする

// これはbox0のリージョンを転送している
Task {
  box0.contents.increment()
}

// これはbox1が(box0と同じリージョンで)既に転送されているのでエラー
Task {
  box1.contents.increment()
}
```

この関数規約は上図に示されている。上記のコードは、`compareBoxContents`の呼び出しが`box0`と`box1`の間にエイリアシングや参照を導入しないので、実際には完全に安全だが、関数規約は、`compareBoxContents`と同じ静的型を持つが、`box0`と`box1`の内容の間にエイリアシングを導入する`reassignBoxContents`のような実装に関して安全でなければならない。もし`reassignBoxContents`が上記のコードに配置されていたら、2つのタスクの生成は確かにデータ競合を引き起こすので、コードは安全でないとマークされなければならない。

#### 要約: リージョンに対するシンプルなルール 

要約すると、リージョンは、クラスのようなNonSendableな値が、NonSendableな引数なしで初期化されるときに「生成」される。リージョンは以下のように拡張される:

- `let y = x.f`: `x`のNonSendable型フィールドを読み込むと、`x`と同じリージョンに値`y`が生成される
- `let y = x`: エイリアス`y`を作成すると、`x`と同じリージョンに値`y`が生成される
- `y.f = x`: NonSendableな値`y`からNonSendableな値`x`への参照を作成し、それらのリージョンをマージする
- `z = { ... x ... y ...}`: クロージャ内でNonSendable値`x`と`y`をキャプチャすると、それらのリージョンがマージされ、`x`と`y`と同じリージョンに新しいクロージャ`z`が生成される
- `if/for/while/switch`(制御フロー): もし`x`と`y`が関数の先行基本ブロックで同じリージョンにあれば、その基本ブロックのすべての後続ブロックで同じリージョンにマージされる

この戻り値、分離ドメイン間でNonSendableな値が送信された時点で、送信までの制御フローパスの1つだけでも、その値からエイリアスされたり参照されたりする可能性のある他の全ての値が、その値と同じリージョンにあるとみなされ、"transferred"とマークされ、送信後にアクセスできなくなる。

#### 分離ドメインの横断

上で概説した関数規約は、呼び出し元と同じ分離ドメインで同期的に実行される関数呼び出しに適用される。分離ドメインをまたがる呼び出しは少し異なるセマンティクスを持つ。単にマージされるのではなく、分離ドメインをまたがる呼び出しに対する全てのNonSendableな引数のリージョンも同様に転送されます。これが必要な理由は、分離ドメインを越えて値を渡す2つのタイプの呼び出しで異ななる。

##### アクター間での呼び出し

NonSendableな値がアクター入力呼び出し、つまりアクター自身の分離以外の分離からのアクターメソッド呼び出しに渡されると、その値はアクターのストレージに"エスケープ"される可能性がある。そのため、(必然的に非同期の)呼び出しが戻った後でも、アクターは呼び出し元がより多くのコードを実行するのと同時に新しいリクエストを処理することができる。元々渡された値は、そのストレージを通してアクターにアクセス可能になったので、データ競合を防ぐために呼び出し元がアクセスすることを禁止しなければならない。したがって、アクター入力呼び出しは引数を転送する必要がある。次のコードはこれを示している:

```swift
actor NationalPark {
  var visitors : [Visitor]
  
  func admitVisitor(_ visitor : Visitor) {
    visitors.append(visitor)
  }
  
  func admitAndGreetVisitor(_ visitor : Visitor) {
    // この呼び出しは分離を超えないので、
    // `visitor`を転送しない。単に`self`のリージョンにマージする
    admitVisitor(visitor) // 呼び出し 1 - 安全
    
    // なので、`visitor`へのアクセスは許可されている
    visitor.greet()
  }
}

func visitParks(_ parks : [NationalPark], _ visitorName : String) async {
  let visitor = Visitor(visitorName)
  
  for park in parks {
    // この呼び出しは`park`アクターに入っているため、
    // `visitor`を転送するので
    // ループ内での`visitor`のアクセスはエラーである
    await park.admitVisitor(visitor) // 呼び出し 2 - エラー
  }
}
```

呼び出し1とラベル付けされたアクターメソッド`admitAndGreetVisitor`内の`admitVisitor`の呼び出しは、その引数を転送しないことに注意。これは、この呼び出しがアクターを横断した呼び出しではないため、他のアクターが引数への参照を保持するリスクがないためである。

一方、呼び出し2とラベル付けされたnonisolated関数`visitParks`における `admitVisitor`の呼び出しは、アクターに入るため、引数を転送する。このため、このコードは安全ではなく、実際にエラーが発生する。もしこのコードが許可されていれば、複数のアクターがストレージから同じ`visitor`オブジェクトを同時に参照し、競合を起こす可能性がある。

##### Task生成

`Task.init`や`Task.detached`のように新しい`Task`を作成する特殊な関数や、`MainActor.run`のように渡されたクロージャを呼び出し元と同時に実行させる関数も、引数を転送しなければならない。特に、引数はクロージャであるため、クロージャに取り込まれた値はそのクロージャと同じリージョンにあり、それを転送することでそれらの値も転送される。この転送により、呼び出し元と渡されたクロージャの実行側との間で、それらの値に関する競合を防ぐことができる。これにより、以下のような無茶なコードも表現できなくなる:

```swift
func visitParksParallel(_ parks : [NationalPark], _ visitorName : String) {
  let visitor = Visitor(visitorName)
  
  for park in parks {
    Task {
      await park.admitVisitor(visitor) // エラーになる - クロージャの生成で`visitor`を転送する
    }
  }
}
```

そのため、`Task.init`、`Task.detached`、`MainActor.run`のような、渡されたクロージャを呼び出し元とは異なる分離の下で実行したり、呼び出し元と同時に実行したりする特殊な関数の`SendNonSendable`解析を通知する必要がある。これには2つのアプローチが考えられる:

1. そのような関数のリストをハードコードする - 可能だが、バッドプラクティスとみなされる可能性が高い
2. そのような関数が定義されているソースレベルで`@IsolationCrossing`のようなアノテーションを付けてマークする

さらに、望ましい表現レベルを達成するためには、これらの関数の引数の型に関するソースレベルの`@Sendable`アノテーションを削除する必要がある。`SendNonSendable`パスの導入がなければ、`Task.detached`のような関数は、`@Sendable`引数のみを取るようにすることで、データ競合フリーになる。これにより、NonSendableな値を取り込むクロージャがこれらの関数に渡されることを完全に防ぐことができる(そのようなクロージャは必然的に`@Sendable`になるため)。`SendNonSendable`パスを実装することで、Sendableなクロージャをこれらの関数に自由に渡すことができるようになるが、NonSendableなクロージャが全面的に禁止されるわけではなく、むしろ分離横断呼び出しに渡される他の全てのNonSendableな値と同じように、フローセンシティブなリージョンベースのチェックの対象となる。この変更(これらの関数の引数シグネチャから`@Sendable`を削除)は必要です。

#### 診断

`SendNonSendable`パスの診断を生成する最も簡単な方法は、値がアクセスされるが、パスによって転送リージョンにあることが分かっている各コード側を記録することである。これらの診断戻り値は次のようになる:

```swift
func giveBoxToActor(a : MyActor) async {
  let b = Box()
  
  await a.accessBox(b)
  
  if (b.contents >= 7) { // warning: NonSendable型の`Box`にここでアクセスしているが、潜在的な競合を生成するため、上記の同時実行されるコード上で送信できない
		....
}
```

このパスの完全に正しいセマンティクスをこの診断スタイルで実装することもできる。つまり、パスがエラーを発見した場合に診断を投げることもできるが、このスタイルの診断はプログラマを混乱させる可能性がある。エラーが発見された箇所ではないが、実際に分離を越える送信が行われた箇所の方が、警告を表示する場所としてはずっと論理的であると思われる。というのも、分離を越えるような呼び出しは、並行性、そして並行通信が行われる場所であることが容易にわかっているからである。しかし、転送されたリージョンの値がアクセスされる場所は、Swift言語の有効なASTノードである可能性がある。高いレベルでは、プログラマが、同時並行通信が実行されるポイントで送信される値について最も注意深く考えることを期待するのは合理的であると思われる。したがって、そのようなポイントで送信された値と、関数の後半でアクセスされた前リージョンの値の組み合わせから競合が発生するが、値が送信されたポイントを強調表示することで、プログラマは、並行通信が実行されるポイントに主にデバッグの努力を集中し続けることができる。この理由に従い、`SendNonSendable`の実装では、代わりに以下のような診断を出力する:

```swift
func giveBoxToActor(a : MyActor) async {
  let b = Box()
  
  await a.accessBox(b) // warning: 呼び出し元でNonSendable型の'Box'引数をnonisolatedコンテキストからアクター分離コンテキストに渡すとこの関数の後の方のアクセスと競合する可能性がある(1箇所アクセスがある)

  if (b.contents >= 7) { // note: ここのアクセスが競合を起こす可能性がある
		....
}
```

送信された値のリージョン内の値にアクセスするポイントが複数ある場合は、それらすべてが表示される:

```swift
func compareListElems(a : MyActor) async {
  let myList : [NonSendableElem] = genList()
  
  let elem0 = myList.min(by : {$0 < $1})
  let elem1 = myList.max(by : {$0 < $1})
  
  await a.displayElem(elem0) // warning: 呼び出し元でNonSendable型の'NonSendableElem'引数をnonisolatedコンテキストからアクター分離コンテキストに渡すと、この関数の後の方のアクセスと競合する可能性がある(2箇所アクセスがある)
  
  print("min: \(elem0)") // note: ここのアクセスが競合を起こす可能性がある
  print("max: \(elem1)") // note: ここのアクセスが競合を起こす可能性がある
}
```

このパスで発行される診断には、もう1つのカテゴリーがある。上のセクションで説明したように、デフォルトの関数規約では、関数は(`self`を含む)引数のリージョンを転送しないと仮定されている。そのため、`self`のNonSendableな値と同じリージョンにあるNonSendableな値や、NonSendableな引数を渡す分離横断呼び出しはエラーとなる。

```swift
func passToActor(a : MyActor, v : NonSendableValue) async {
  await a.foo(v) // warning: この関数の`self`もしくはNonSendableな引数を他のスレッドに渡すと呼び出し元とデータ競合を起こす可能性がある
}
```

診断メッセージが示すように、この警告が必要なのは、関数規約で、非分離横断呼び出しに引数が渡された後も、NonSendableな引数を使い続けるコードが許されているからである。

```swift
func genAndPassTo(a : MyActor) async {
  let v = NonSendableValue()
  
  // この呼び出しはvを転送"しない"
  await passToActor(a, v)
  
  // ここでのアクセスは許可されている
  print(v)
}
```

上記の関数`passToActor`に型チェックをさせるには、転送スタイルのアノテーションが必要である。以下の引数の転送のセクションを参照してほしい。

## ソース互換性

以前に有効で、適切に型付けされたすべてのSwiftコードは、依然として有効で、適切に型付けされたSwiftコードである。

## ABI互換性

`MainActor.run`のような関数のシグネチャを変更する可能性があることを除けば、現在予定されているABIの変更はない。将来的には、リージョン分割のより一般的な操作をサポートするために、関数のシグネチャに情報を追加する必要があるかもしれない。例えば、関数の戻り値が新しいリージョンや引数のリージョンから来ることを保証したり、クラスメソッドへの2つの引数が`self`とは異なるリージョンから来ることを許可したりする。

## 導入の影響

`SendNonSendable`パスが導入された場合、Swiftの並行処理を広く使用するライブラリの開発を促進するだろうが、分離ドメイン間で通信される全ての型に`Sendable`を強制することはありえない。これは大きな表現力の勝利だが、ライブラリが書かれる方法を根本的に変える可能性がある。

## 将来の方向性

### 引数の転送

現在の`SendNonSendable`の実装では、関数は引数を転送することができない。これは許容されるプログラミングパターンを大きく制限する。他の分離ドメインに値を送るには、その値はローカルで初期化されていなければならない。`self`のストレージから他のスレッドに安全に値を送ることを許可することは`iso`フィールド拡張(後述)[#iso]の主題であり、少し複雑である。必要なのは、関数のシグネチャに`transferring`アノテーションを追加し、関数本文の終わりまでに特定の引数が転送される可能性があることを示すだけである。最も簡単な例として、以下のコードに`transferring`アノテーションの動作を示す:

```swift
func passToActorTransferring(a : MyActor, transferring v : NonSendableValue) async {
  await a.foo(v)
}

func genAndPassTransferring(a : MyActor) async {
  let v = NonSendableValue()
  
  // 呼び出しでvを転送する
  await passToActorTransferring(a, v)
  
  // ここでのアクセスを許可されない
  print(v)
}
```

以前の関数`passToActor`(上で定義)[#passtoactor]とは異なり、`passToActorTransferring`は型チェックを行うようになったが、`genAndPassToTransferring`は行わない。

`transferring`パラメータはごく自然なプログラミングパターンである。これがなければ、NonSendableな値がドメイン間を1ホップすることしかできない。また、実装も非常に簡単で、この提案が受け入れられる前に実装されているかもしれない。この機能の追加に関する最大の困難は、Swift言語に存在する既存の`transferring`アノテーションとの人間工学的な相互作用である。既存の`transferring`キーワードは、コピー不可能な型に焦点を当て、参照カウントとdeallocationを処理するための所有権の規約を指定する。これは、`SendNonSendable`によって必要とされる`transferring`の考え方に関連しており(今のところ、それを`region-transferring`と呼ぶことにしよう)。実際、パラメータが`region-transferring`であるどんなケースでも、それはまた(既存のSwiftの、コピー不可能な意味での)`transferring`であるべきである。残念ながら、逆は成り立たない。例えば、イニシャライザとセッタへのパラメータは、通常、`transferring`だが、それらは`region-transferring`であるべきではない。その理由は、`SendNonSendable`のリージョンベースの型チェックが、セッタやイニシャライザに値を渡すことで、所有権が移されたという事実を追跡できるからである。したがって、そのようなパラメータを`region-transferring`としてマークしないことで、セッタやイニシャライザに渡された後でも、セットされた値や初期化された値が転送されない限り、使用することができる。もしそのようなメソッドのパラメータがすべて`region-transferring`であれば、たとえ`self`が転送されなくても、メソッドの引数は呼び出し後にアクセスできなくなる。

呼び出し元では転送は強制され、関数本文内では第3の分離リージョンに渡すことで引数を再び転送する自由が得られる。このことは、アノテーションとして公開することが有用であることのさらなる証拠となる。

この拡張のセマンティクスを具体的に説明すると、`transferring`パラメータは以下のようになる:

- 呼び出し元が分離横断しているかどうかに関わらず、呼び出し元でNonSendableな引数のリージョンを転送する(呼び出し元が分離横断している場合にのみ引数のリージョンを転送する非`transferring`パラメータとは対照的)。
- 関数本文への入口で使用されるリージョン分割において、`self`や他の全ての引数とは別のリージョンが割り当てられる

技術的には、2つの`transferring`パラメータが互いに同じリージョンから来たと仮定できるようにすることで、上記のセマンティクスの2番目のポイントを拡張した、さらに一般的なアプローチがあることに注意してほしい。この機能がなければ、`transferring`パラメータは常に、呼び出し元の他のすべての引数とは別のリージョンにあることが分かっている値を渡さなければならない。この機能があれば、互いにエイリアスである可能性のある2つの値、または互いに参照している可能性のある2つの値を、2つの`transferring`引数に渡すことができる。このさらなる拡張の利点は、`transferring`そのものよりも具体的でないため、すぐに導入されることはないだろう。

### 新しい戻り値を返す

関数シグネチャをより表現豊かにするもう1つの方法は、その戻り値である。現在の`SendNonSendable`では、分離を越える関数からNonSendableな値を返す方法はない。実際、`TypecheckConcurrency`から発生する、分離境界を越えてNonSendableな戻り値を得るために生成される診断も抑制されていません。このため、以下のようなコードは不可能である:
  
```swift
func generateFreshPerson(_ name : String, _ age : Int, _ ancestryMap : AncestryMap) -> Person {
  let person = Person(name, age)
  if (ancestryMap.containsChild(person)) { ... /* do some logic */ }
  
  return person
}
```

このコードは基本的に、イニシャライザを余分なロジックでラップし、NonSendableな戻り値を呼び出し元へ新しいリージョンで返すことを目的としている。残念ながら、現在の`SendNonSendable`パスでは、その戻り値を使用することはできない。有用な拡張は、以下のような場合に、NonSendableな戻り値を持つ関数が、その戻り値を分離横断の呼び出し元からアクセスできるようにすることである:

- アクターメソッドでないメソッドでは、それらのメソッドへの分離横断呼び出しの戻り値は常に呼び出し元が利用できる。
  - デフォルトでは、戻り値は`self`やその他の引数(`transferring`引数(上記参照)[#transferringargs]を除く)と同じリージョンで提供される
  - 関数のシグネチャで型に`fresh`キーワードが指定されている場合(例: `func generateFreshPerson(...) -> fresh Person`)、戻り値は新しいリージョンで提供される
- アクターメソッドの場合:
  - 戻り値が`fresh`キーワードでアノテートされていない場合、アクター分離されたストレージと同じリージョンにあるため、分離横断の呼び出し元からはアクセスできない
  - 戻り値が`fresh`でアノテートされている場合、非アクターメソッドと同様に新しいリージョンで呼び出し元に提供される

`SendNonSendable`パスの実行は、`fresh`なNonSendableな戻り値として使用されるすべての値が、`self`およびすべての引数のリージョンとは異なるリージョンから確かに来ることを確認することも含むようになる。

#### `iso`フィールド



## 参考リンク

### Forums

- []()

### プロポーザルドキュメント

- []()
# Swift DiscardingTaskGroups

- [Swift DiscardingTaskGroups](#swift-discardingtaskgroups)
  - [概要](#概要)
  - [内容](#内容)
    - [導入理由](#導入理由)
    - [提案内容](#提案内容)
    - [APIサーフェス](#apiサーフェス)
  - [詳細](#詳細)
    - [結果の破棄](#結果の破棄)
    - [エラーの伝播とグループのキャンセル](#エラーの伝播とグループのキャンセル)
  - [将来の検討事項](#将来の検討事項)
    - [エラーハンドリング](#エラーハンドリング)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

`Discarding[Throwing]TaskGroup`というStructured ConcurrencyのTaskGroupに新しい型を導入することを提案する。これは、`TaskGroup`に似ているが、子タスクの結果を即座に破棄する。これは、HTTPや他の種類のRPCサーバーの最上位層で行われるループなど、潜在的に無限に実行され続けるタスクのグループで利用することに特化している。

## 内容

### 導入理由

タスクグループは、Structured Concurrencyの構成要素であり、Swiftランタイムがタスクをグループとして関連付けることができる。これにより、キャンセルの自動伝播、エラーの正確な伝播、適切に定義したタスクの有効期間の保証、プログラミングツールへの診断情報の提供、などの強力な機能が有効に活用できる。

SE-0304で導入されたタスクグループのバージョンは、これらすべての機能をすでに提供している。しかし、戻り値をタスクグループのユーザに伝達する機能も提供しており、一部のユースケースでは予期しない制限を与えている。

何かというと、タスクグループのユーザは、子タスクの戻り値を取得できるため、タスクグループは少なくとも完了した子タスクの結果を保持する。実際は、`Task` オブジェクト全体を保持する。このデータは、ユーザがタスクグループの結果を消費するAPI(`next()`メソッドまたはタスクグループを`for await in`などでループさせる)の1つを介して消費するまで保持される。

この結果、タスクグループは無限に実行される可能性がある場合には適していない。ユースケースとしては、待機しているソケットに受け入れられたコネクションを管理するケースがある。このようなタスクの簡単な例は次のとおり:

```swift
try await withThrowingTaskGroup(of: Void.self) { group in
    while let newConnection = try await listeningSocket.accept() {
        group.addTask {
            handleConnection(newConnection)
        }
    }
}
```

上記で記述したとおり、このタスクグループは、待機しているソケットが終了するかエラーをスローするまで、すべての子タスクの`Task`オブジェクトがメモリ上に残ったままになる。これが実行時間の長いサーバ用に作成されたものである場合、このタスクグループが数日間存続し、何千もの`Task`オブジェクトがメモリ上に残ったままになる可能性は十分にある。安定したサーバの場合、これにより、最終的にプロセスがメモリ不足になってOSによって強制終了されることになる。

タスクグループの現在の実装では、この問題を回避する実用的な方法は提供されていない。タスクグループは (正確には)`Sendable`ではないため、完了したタスクの結果の消費も、新しい作業の送信も、別のタスクで行うことはできない。

この無限のメモリ消費を回避するための最も自然な試みは、完了したタスクの結果をときどき破棄すること。 例は次のとおり:

```swift
try await withThrowingTaskGroup(of: Void.self) { group in
    // タスクグループにmaxConcurrencyまでタスクを追加する
    for _ in 0..<maxConcurrency {
	    guard let newConnection = try await listeningSocket.accept() else {
	        break
	    }

	    group.addTask { handleConnection(newConnection) }
	}

    // 1つタスクを消費したら1つ追加することを繰り返す
	while true {
	    _ = try await group.next()
	    guard let newConnection = try await listeningSocket.accept() else {
	        break
	    }
	    group.addTask { handleConnection(newConnection) }
	}
}
```

これは実行可能ですが、ユーザは`maxConcurrency`の値を決定する必要がある。しかし、事前に決定するのが非常に難しいことはよくある。実際には、ユーザは推測して、値が大きすぎる(メモリが浪費される)か、小さすぎる(システムが十分に活用されない)かのどちらかになる傾向がある。`TaskGroup`の最大同時実行数を制限するための戦略を策定することには価値があるが、この問題はすでに十分に複雑であるため、これは別で議論するのが良いだろう。

### 提案内容

新しい`DiscardingTaskGroup`および`ThrowingDiscardingTaskGroup`型、(`withDiscardingTaskGroup`および`withThrowingDiscardingTaskGroup`を呼び出すことで取得される)を追加することを提案する。これらのグループは、通常の`TaskGroups` 実装と多少似ているが、次の重要な点で異なる:

1. `[Throwing]DiscardingTaskGroup`は、子タスクの`Task`が完了すると、それを自動的にクリーンアップする
2. `[Throwing]DiscardingTaskGroup`には`next()`メソッドがなく、`AsyncSequence`にも準拠していない

名前が示すように、子タスクの*結果を常に破棄しているため*、これらは`ChildTaskResult`を引数に持たず`Void`であると想定される。

`[Throwing]DiscardingTaskGroup`のキャンセルとエラーの伝播は、これまでのタスクグループと同じように機能するが、明示的に`next()`を使用して子タスクのエラーを「再びスロー」することができないため、破棄するタスクグループは *最初に発生したエラー*を再びスローしてグループをキャンセルすることにより、この動作を暗黙的に処理する必要がある。

`[Throwing]DiscardingTaskGroup`は、`[Throwing]TaskGroup`と同じように、Structured Concurrencyのプリミティブであり、`[try] await with[Throwing]DiscardingTaskGroup { body }`の本文が戻る前に、送信されたすべてのタスクを自動的に待機する。

### APIサーフェス

```swift

public func withDiscardingTaskGroup<GroupResult>(
    returning returnType: GroupResult.Type = GroupResult.self,
    body: (inout DiscardingTaskGroup) async -> GroupResult
) async -> GroupResult { ... }

public func withThrowingDiscardingTaskGroup<GroupResult>(
    returning returnType: GroupResult.Type = GroupResult.self,
    body: (inout ThrowingDiscardingTaskGroup<Error>) async throws -> GroupResult
) async throws -> GroupResult { ... }
```

また、`next()`および関連する機能がないことを除いて、ほとんどが`TaskGroup`のAPIをそのまま反映している。

```swift

public struct DiscardingTaskGroup {

    public mutating func addTask(
        priority: TaskPriority? = nil,
        operation: @Sendable @escaping () async -> Void
    )

    public mutating func addTaskUnlessCancelled(
        priority: TaskPriority? = nil,
        operation: @Sendable @escaping () async -> Void
    ) -> Bool

    public var isEmpty: Bool

    public func cancelAll()
    public var isCancelled: Bool
}
@available(*, unavailable)
extension DiscardingTaskGroup: Sendable { }

public struct ThrowingDiscardingTaskGroup<Failure: Error> {

    public mutating func addTask(
        priority: TaskPriority? = nil,
        operation: @Sendable @escaping () async throws -> Void
    )

    public mutating func addTaskUnlessCancelled(
        priority: TaskPriority? = nil,
        operation: @Sendable @escaping () async throws -> Void
    ) -> Bool

    public var isEmpty: Bool

    public func cancelAll()
    public var isCancelled: Bool
}
@available(*, unavailable)
extension DiscardingThrowingTaskGroup: Sendable { }
```

## 詳細

### 結果の破棄

名前が示すように、`[Throwing]DiscardingTaskGroup`はその子タスクの結果をすぐに破棄し、結果を生成した子タスクを解放する。これにより、HTTPやRPCサーバーなどのループを受け入れる、効率的で「永久に実行される」リクエストが可能になる。

具体的には、このプロポーザルの動機に示されている最初の例は、次のように`[Throwing]DiscardingTaskGroup`を使用して安全に表現できる。

```swift
// リークなし!
try await withThrowingDiscardingTaskGroup() { group in
    while let newConnection = try await listeningSocket.accept() {
        group.addTask {
            handleConnection(newConnection)
        }
    }
}
```

このコードは、前に示した`withThrowingTaskGroup`バージョンとは異なり、タスクがメモリ上に残らないため安全であり、そのような無限ループをハンドリングするために推奨される方法である。

### エラーの伝播とグループのキャンセル

タスクグループのエラーのスローは、子タスクがスローした可能性のあるエラーを明らかにするために、`next()`(または`waitForAll()`)がエラーをスローすることと、エンドユーザがこのように子タスクの結果を消費するということに依存している。`ThrowingTaskGroup`の場合、次のように、明示的に結果(および失敗)を収集し、それらに反応することが可能。

```swift
try await withThrowingTaskGroup(of: Void.self) { group in 
  group.addTask { try boom() }
  group.addTask { try boom() }
  group.addTask { try boom() }

  try await group.next() // 最初に発生したどんなエラーも再度スローする
} // 本文からエラーがスローされたので、グループと残りのタスクはすぐにキャンセルされる 
```

上記のスニペットは、`withThrowingTaskGroup`クロージャ本文から、`try await next()`(または`try await group.waitForAll()`)を通して、子タスクからエラーが伝播する単純なケースを示している。エラーがクロージャ本文からスローされるとすぐに、グループはグループ自体と残りのすべてのまだ実行を開始していない子タスクを暗黙的にキャンセルし、最後に保留中のすべてのタスクの完了を待機する。

このパターンは、`ThrowingDiscardingTaskGroup`では使用できない。というのも、`DiscardingTaskGroup`では、結果を集めるメソッドを使用できない。`DiscardingTaskGroup`の一般的なユースケースを適切にサポートするためには、一つの子タスクが失敗すると、グループ自体とすべての同じ階層の他の子タスクを暗黙的かつ即座にキャンセルする必要がある。

つまり、ある子タスクの失敗を検知すると全て子タスクを暗黙的に即時に消費して、失敗を自動的に「再スロー」すると見なせる。よって、エラーは次のように `withThrowingDiscardingTaskGroup`メソッドからも再スローされる:

```swift
try await withThrowingDiscardingTaskGroup() { group in 
    group.addTask { try boom(1) }
    group.addTask { try boom(2) }
    group.addTask { try boom(3) }
    // 最初に発生した失敗が収集され、保持され、終了時にメソッドから再スローされる
}
```

つまり、`[Throwing]DiscardingTaskGroup`は、失敗処理の「one for all, all for one」パターンに従う。1つの子タスクが失敗するとすぐに、グループと同じ階層にある子タスクがキャンセルされる。

この動作を防止するには、次の2つの方法がある:

- 子タスクはエラーをスローすることが許可されず、`withDiscardingTaskGroup`を使用して他の方法でエラーを処理する
- 子タスク内に通常の`do {} catch {}`エラー処理ロジックを含め、再スローのみを行う

エラーを処理して対処するための`DiscardingTaskGroup`に特別な方法を導入するのではなく、通常のSwiftのコードパターンに寄せるべきであり、Structured Concurrencyのプリミティブの正しいアプローチであると感じている。ただし、必要に応じて、将来的に「複数のエラーを処理するreducer」を導入する可能性がある。

## 将来の検討事項

### エラーハンドリング

このpitchの過程で、「最初のエラーのみをスローする」パターンは柔軟性が不十分である可能性がある、という多くの懸念が提起された。コミュニティメンバーは、必要に応じてエラーをフィルタ、収集、または破棄するために使用できる、ある種のエラーフィルタ機能に特に関心を持っていた。

プロポーザルの著者は、このAPIサーフェスを最初のバージョンに導入すると、`[Throwing]DiscardingTaskGroup`がかなり複雑になると感じている。このエラーフィルタ機能を導入するには、提案されたAPIサーフェスが、不必要な認知的負荷を追加することなく、必要なユースケースに対応できるという確信が必要である。また、`try/catch`を使用して処理できる機能と、新しいエラーフィルタ関数を必要とする機能の境界線がどこにあるのか、完全には明確ではない。

その結果、プロポーザルの著者は、一般的に必要になる実際の例が得られるまで、ここで何かを実装することを延期することを選択した。ある種のエラーフィルタを持つことは価値がある可能性が高く、そういった機能を実装する余地は残しておくものの、今のところこのプロポーザルは比較的小さい範囲に止めておく。


## 参考リンク

### Forums

- [[Pitch]TaskPool](https://forums.swift.org/t/pitch-task-pools/61703)
- [SE-0381: discardResults for TaskGroups](https://forums.swift.org/t/se-0381-discardresults-for-taskgroups/62072)

### プロポーザルドキュメント

- [discardResults for TaskGroups](https://github.com/apple/swift-evolution/blob/main/proposals/0381-task-group-discard-results.md)
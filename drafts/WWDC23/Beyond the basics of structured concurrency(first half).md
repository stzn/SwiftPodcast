# Structured Task vs Unstructured Task

- Structured Task(async let, TaskGroup)
  - タスクは宣言されたスコープの終わりまで生存する
- Unstructured Task(Task {}, Task.detached {})

**使えるときはいつでもStructured Taskのほうが好ましい**


# Task hierarchy

- Structured Taskはタスクの親子関係(ツリー構造)を作成する

# Task cancellation

- このツリー内のタスクでキャンセルが伝播される
- Unstructured Taskは明示的にcancelメソッドを呼ぶ必要がある
- キャンセルは協調的なのですぐにタスクがストップするわけではない。"isCancelled"フラグが立つだけ。
- キャンセルは競合する(=キャンセルチェックのタイミングまでに処理がキャンセルされているかどうかは場合による)
- なので、重い処理の前にはまだ処理が必要か確認するためにキャンセルチェックをするべき
- キャンセルチェックは同期的なので、キャンセル対応が必要な関数は、次の処理が同期か非同期か関係なく処理を継続する前にチェックするべき
- タスク実行中の場合はisCancelledやcheckCancellationメソッドは役に立つ
- タスクがsuspend中や実行されていない場合はwithTaskCancellationHandlerが役に立つ

キャンセルが起きるシナリオ

```swift
actor Cook {
    func handleShift<Orders>(orders: Orders) async throws
       where Orders: AsyncSequence,
             Orders.Element == Order {

        for try await order in orders {
            let soup = try await makeSoup(order) // ① makeSoup関数でキャンセルを検知してエラーをthrowする
            // ...
        } // ②次のOrderを待っている間(suspend中)にキャンセルが起きる
    }
}
```

②の場合はタスクが実行されていないため、isCancelledやcheckCancellationが使えない。

タスクがsuspend中や実行されていない場合はwithTaskCancellationHandlerが役に立つ。
多くのAsyncSequencesは、シーケンスを止める目的でステートマシンを利用している。


```swift
public func next() async -> Order? {
    return await withTaskCancellationHandler {
        let result = await kitchen.generateOrder()
        guard state.isRunning else { // まだ実行中か値を出力し続ける必要があるかを判定
            return nil
        }
        return result
    } onCancel: {
        state.cancel() // タスクがキャンセルされたら同期的にシーケンスを止める(isRunningをfalseにする)
    }
}
```

CancelHandlerは即時実行され、bodyとはスレッドが異なる可能性があるため、stateは共有可変状態となり、データ競合を防ぐ必要がある。

Actorはカプセル化された状態を保護するのに適しているが、ステートマシンの個々のプロパティを変更したり読み取ったするため、今回Actorは適切ではない。
さらに、Actorに対する操作の実行順序を保証することはできないので、キャンセルが最初に実行されることを保証できない。何か別のものが必要。

※ 今回はSwiftのAtomicsパッケージからAtomicsを使った。ディスパッチキューやロックでもOK。

このメカニズムは、データ競合を避けながら、共有可変状態を同期させることを可能にし、一方で、CancelHandlerで非構造化タスクを導入することなく、実行中のstateをキャンセルできるようにする。

```swift
private final class OrderState: Sendable {
    let protectedIsRunning = ManagedAtomic<Bool>(true)
    var isRunning: Bool {
        get { protectedIsRunning.load(ordering: .acquiring) }
        set { protectedIsRunning.store(newValue, ordering: .relaxed) }
    }
    func cancel() { isRunning = false }
}
```

# Task Priority

## 優先順位とは何か、そしてなぜ気にする必要があるのか

- 優先順位とは、与えられたタスクの緊急度をシステムに伝えるための方法
- ある種のタスク、例えばボタンが押されたらすぐに反応するようなタスクは、すぐに実行しないとアプリがフリーズしたように見えてしまう
- 一方、サーバからコンテンツをprefetchするような他のタスクは、誰にも気づかれずにバックグラウンドで実行することができる


## 優先順位の逆転とは何か

- 優先順位の逆転とは、優先順位の高いタスクが、優先順位の低いタスクの結果を待っているときに起こる
- デフォルトでは、子タスクは親タスクから優先度を引き継ぐので、仮に親タスクが優先度mediumで実行されているとすると、すべての子タスクも優先度mediumで実行される

## 優先順位のエスカレート

- 優先順位の高いタスクの結果を待つと、タスクツリー内のすべての子タスクの優先順位がエスカレートする
- タスクグループの次の結果を待つと、グループ内のすべての子タスクがエスカレートすることに注意。なぜなら、どのタスクが次に完了する可能性が高いか分からないからである
- Concurrencyのランタイムは、優先順位キューを使用してタスクをスケジューリングするため、優先順位の高いタスクが優先順位の低いタスクの前に実行されるように選択される
- タスクは、そのライフタイムの残りの期間、エスカレーションされた優先順位を維持する
- エスカレーションを取り消すことはできない

## 最大実行数の制限

- タスクを作りすぎると他の作業をするスペースがなくなってしまう
- 最初に最大実行数だけaddTaskして、最初のタスクの結果を得たら次のタスクをaddTaskする。

```swift
func chopIngredients(_ ingredients: [any Ingredient]) async -> [any ChoppedIngredient] {
    return await withTaskGroup(of: (ChoppedIngredient?).self,
                               returning: [any ChoppedIngredient].self) { group in
        // 同時並行に素材を切る
        let maxChopTasks = min(3, ingredients.count)
        for ingredientIndex in 0..<maxChopTasks {
            group.addTask { await chop(ingredients[ingredientIndex]) }
        }

        // 切った野菜を集める
        var choppedIngredients: [any ChoppedIngredient] = []
        var nextIngredientIndex = maxChopTasks
        for await choppedIngredient in group {
            if nextIngredientIndex < ingredients.count {
                group.addTask { await chop(ingredients[nextIngredientIndex]) }
                nextIngredientIndex += 1
            }
            if let choppedIngredient {
                choppedIngredients.append(choppedIngredient)
            }
        }
        return choppedIngredients
    }
}
```

```swift
withTaskGroup(of: Something.self) { group in
    for _ in 0..<maxConcurrentTasks {
        group.addTask { }
    }
    while let <partial result> = await group.next() {
        if !shouldStop { 
            group.addTask { }
        }
    }
}
```

##　DiscardingTaskGroup

- 戻り値を返さない完了した子タスクの結果を保持し続けない
- 自動でリリースするのでcancelを呼ぶ必要がない
- ある子タスクでエラーがthrowされると他の子タスクは自動でキャンセルされる

↓は完了した子タスクの結果を保持しているため、効率がよくない

```swift
func run() async throws {
    try await withThrowingTaskGroup(of: Void.self) { group in
        for cook in staff.keys {
            group.addTask { try await cook.handleShift() }
        }

        group.addTask {
            // 閉店までレストランを開き続ける
            try await Task.sleep(for: shiftDuration)
        }

        try await group.next()
        // 実行中のシフトをキャンセルする
        group.cancelAll()
    }
}
```

DiscardingTaskGroupを使って効率よくする

```swift
func run() async throws {
    try await withThrowingDiscardingTaskGroup { group in
        for cook in staff.keys {
            group.addTask { try await cook.handleShift() }
        }

        group.addTask { // 閉店までレストランを開き続ける
            try await Task.sleep(for: shiftDuration)
            throw TimeToCloseError()
        }
    }
}
```
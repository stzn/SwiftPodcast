# Allow TaskGroup's ChildTaskResult Type To Be Inferred

バージョン: Swift 6.1

## 内容

`TaskGroup`と`ThrowingTaskGroup`の`ChildTaskResult`の型を推論できるようにする。

```swift

// 現状(of: Message.self)が必要
let messages = await withTaskGroup(of: Message.self) { group in
  for id in ids {
    group.addTask { await downloadMessage(for: id) }
  }

  var messages: [Message] = []
  for await message in group {
    messages.append(message)
  }
  return messages
}
```

```swift

// (of: Message.self)が不要になる
let messages = await withTaskGroup { group in
  for id in ids {
    group.addTask { await downloadMessage(for: id) }
  }

  var messages: [Message] = []
  for await message in group {
    messages.append(message)
  }
  return messages
}
```

## 補足

`ChildTaskResult`にデフォルト引数を設定することで推論できるようにした。

```swift
public func withTaskGroup<ChildTaskResult, GroupResult>(
    of childTaskResultType: ChildTaskResult.Type, // ← デフォルト引数が追加された
    returning returnType: GroupResult.Type = GroupResult.self, 
    body: (inout TaskGroup<ChildTaskResult>) async -> GroupResult
) async -> GroupResult where ChildTaskResult : Sendable
```

これは[SE-0326 Enable multi-statement closure parameter/result type inference](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0326-extending-multi-statement-closure-inference.md)のおかげ。

ただし、型推論はトップダウンで行われるため、`ChildTaskResult`のジェネリック引数の推論は、`group`を使用する最初の文に依存する。そのため、最初の`group`の使用が`group.addTask(...)`ではない場合に、コンパイルエラーが発生する可能性がある。

> SE-0327より:
> 最初の`return`文がクロージャの戻り値の型を定義する(型推論はコンテキスト情報がない場合にのみ行われる)。異なる型を生成する`return`文が存在する場合はエラーとなる。

```swift
// ❌ コンパイルエラー
await withTaskGroup { group in 
    // `addTask(...)`が最初の文ではないためコンパイルできない。
    group.cancelAll() 

    for id in ids {
      group.addTask { await logMessageReceived(for: id) } 
    }
}
```

こういったケースの場合は、これまで通り`ChildTaskResult`を指定することで解決できる。

```swift
await withTaskGroup(of: Void.self) { group in 
   ...
}
```

また`addChildTask`から異なる型の戻り値を返すことはできない。

```swift
await withTaskGroup { group in
    group.addTask { await downloadMessage(for: id) }
    group.addTask { await logMessageReceived(for: id) } // Cannot convert value of type 'Void' to closure result type 'Message'
}
```

明示的に型を指定しても同様。

```swift
await withTaskGroup(of: Void.self) { group in
    group.addTask { await downloadMessage(for: id) } // Cannot convert value of type 'Message' to closure result type 'Void'
    group.addTask { await logMessageReceived(for: id) }
}
```

## プロポーザルリンク

- [https://github.com/swiftlang/swift-evolution/blob/main/proposals/0442-allow-taskgroup-childtaskresult-type-to-be-inferred.md](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0442-allow-taskgroup-childtaskresult-type-to-be-inferred.md)

## Forums

- [acceptance](https://forums.swift.org/t/accepted-se-0422-allow-taskgroups-childtaskresult-type-to-be-inferred/73747)

- [review](https://forums.swift.org/t/se-0442-allow-taskgroups-childtaskresult-type-to-be-inferred/73397)

- [pitch](https://forums.swift.org/t/allow-taskgroups-childtaskresult-type-to-be-inferred/72175)

## 実装PR

- [https://github.com/apple/swift/pull/74517](https://github.com/apple/swift/pull/74517)

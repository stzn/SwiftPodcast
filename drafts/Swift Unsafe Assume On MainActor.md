
# Swift Unsafe Assume On MainActor

- [Swift Unsafe Assume On MainActor](#swift-unsafe-assume-on-mainactor)
  - [概要](#概要)
  - [内容](#内容)
    - [動機](#動機)
    - [提案](#提案)
    - [詳細](#詳細)
  - [ソース互換性](#ソース互換性)
  - [ABI stabilityへの影響](#abi-stabilityへの影響)
  - [APIレジリエンスへの影響](#apiレジリエンスへの影響)
  - [代替案](#代替案)
  - [参考リンク](#参考リンク)
    - [Forums](#forums)
    - [プロポーザルドキュメント](#プロポーザルドキュメント)


## 概要

Swiftの`MainActor`は、プログラムの「メインスレッド」の分離ドメインを表すGlobal Actorです。スレッドの安全性と優先度を確保するために、メインスレッドの概念とその上で関数を実行する必要性は、Swift Concurrency以前から存在していた。既存のフレームワークがSwift Concurrencyを採用し始めると、それらの関数に`@MainActor`のアノテーションを付けて、検証のためにその分離をコンパイラに知らせることができる。

しかし、関数に`@MainActor`アノテーションを 1 つだけ追加すると、その呼び出し元に新しい要件が生じる。`@MainActor`関数のすべての呼び出し元は、`@MainActor`自体に分離されているか、メインスレッドで呼び出し先を確実に実行するために呼び出しを`await`する必要がある。

Concurrencyを使用するようにアップグレードする予定の既存のフレームワークの場合、これらの要件により、`@MainActor`を使用して既存のコードパスを断片的に移行することが難しくなる。 特に、これらのコードパスは、メインスレッドで実行されることが既に動的に認識されている。 たとえば、今`MainActor`で実行されている関数の呼び出しの直前に、メインスレッド上にいるかどうかのアサーションを既に行っている場合がある。

これらのアップグレードを容易にするために、`unsafeAssumeOnMainActor`と呼ばれる新しいユーティリティ関数がSwift標準ライブラリ用に提案することで、プログラマが`MainActor`での実行の動的保証から静的保証への移行境界を定義するのに活用できるようにする。

## 内容

### 動機

あなたがSwift開発者で、既存のコード ベースを移行してSwift Concurrencyを使用していると想像してほしい。明らかに `@MainActor`を付ける必要がある関数に遭遇する。それはその効果に対するアサーションさえ行っている:

```swift
func updateUI(_ diffs: [Difference]) {
  dispatchPrecondition(.onQueue(.main))
  // ...
}
```

アノテーションを追加するとアサーションを削除できるが、コンパイラは `updateUI`のいくつかの呼び出し元でエラーを発生させる。それらのいくつかを解決した後、特にこのエラーであなたの動きを一時停止させる。

```swift
@MainActor func updateUI(_ diffs: [Difference]) { /* ... */ }

public extension SingleUpdate {
  func apply() {
    generateDiffs(from: rawContent, on: DispatchQueue.main) { diffs in 
      updateUI(diffs)
//    ^ error: call to main actor-isolated global function 'updateUI' in a synchronous nonisolated context
    }
  }

  public func generateDiffs(from: Content, 
                            on: DispatchQueue,
                            withCompletion completion: ([Difference]) -> ()) {
    // 指定されたDispatchQueueでCompletionハンドラを呼び出す
  }
}
```

問題は、`generateDiffs`に渡されるクロージャーが`@MainActor`ではないことである。`@MainActor`をクロージャリテラルにマークすると、`generateDiffs`に渡されたときに`@MainActor`ではないという別の診断が得られる。コードベースにはゼロ診断ポリシーがある。Completionハンドラが呼び出されるスレッドはその`on`引数によって決定されるため、パラメータの型は`@MainActor`クロージャを受け入れるものとしてマークできない。しかし、これらすべてが正しく機能することはわかっている。`updateUI`関数は常にメインスレッドで終了する。

広く使用され、複雑であるため、今すぐ`generateDiffs`をリファクタリングするにはプロジェクトが大きすぎる。唯一のオプションは、`@MainActor`の`updateUI`への追加をrevertすることです。`updateUI`の他のユーザを更新するために行われた進行状況は、コンパイラによってチェックされなくなるか、revertする必要がある。

### 提案

動機付けとなるこの例の中心的な問題は、`MainActor`で実行されることがわかっている呼び出しに対して安全でないエスケープハッチがないことだが、`MainActor`で実行されるという事実はまだコンパイラに伝えられていない。そこで、標準ライブラリに新しい関数を追加して、その機能を提供する。

```swift
@MainActor func updateUI(_ diffs: [Difference]) { /* ... */ }

public extension SingleUpdate {
  func apply() {
    generateDiffs(from: rawContent, on: DispatchQueue.main) { diffs in 
      unsafeAssumeOnMainActor {
        updateUI(diffs)
      }
    }
  }

  // ...
}
```

`unsafeAssumeOnMainActor`関数は、`@MainActor`クロージャを受け入れるnonisolatedな非同期関数である。まず、`unsafeAssumeOnMainActor`が実行時にテストを実行して、`MainActor`で呼び出されたかどうかを確認する。もし間違っていた場合、プログラムは診断メッセージを出力し、クロージャの実行を続行するか、実行を中止することができる。

### 詳細

```swift
/// 実行時テストを実行して、この関数がMainActor上で呼び出されたかどうかを確認する
/// 次に、操作が呼び出され、その結果が返される。
///
/// - 注意：
/// 実行時チェックが失敗した場合でもオペレーションは引き続きMainActorから呼び出される可能性があるため
/// この操作は安全ではない!
/// 環境変数`SWIFT_UNEXPECTED_EXECUTOR_LOG_LEVEL`を次のように設定することで
/// チェック失敗時の挙動を次のように制御できる:
///
/// - 0 はチェックの失敗を無視する
/// - 1 は警告のみをログに記録する (デフォルト)
/// - 2 は致命的なエラーを意味する
///
/// `0` 以外のモードの場合、メッセージは標準エラーに出力される
///
@available(*, noasync)
public func unsafeAssumeOnMainActor<T>(
  debugFileName: String = #file,
  debugLineNum: Int = #line,
  _ operation: @MainActor () throws -> T) rethrows -> T
```

`debugFileName`および`debugLineNum`のデフォルト引数は、呼び出し元が提供する必要はない。これらは、`MainActor`上にあるという仮定が失敗した場合に、実行時に快適なログメッセージを提供するのに役立つ。例えば:

```swift
func notMainActor() { // これが 1 行目であると仮定
  dispatchPrecondition(.notOnQueue(.main))

  // 実行時に、次のようなメッセージが出力される:
  //
  // warning: data race detected: @MainActor function at example.swift:8 was 
  // not called on the main thread
  unsafeAssumeOnMainActor {
    updateUI([])
  }
}
```

`unsafeAssumeOnMainActor`は、非同期コンテキストでは使用できないとマークされている。これらのコンテキストは常に `@MainActor`クロージャの呼び出しを待機できるためである。

## ソース互換性

影響なし。

## ABI stabilityへの影響

この関数は、Swift Concurrencyをサポートしているすべてのプラットフォームを対象とする場合に使用できるように、バックデプロイできる。

## APIレジリエンスへの影響

影響なし。

## 代替案

`unsafeAssumeOnActor`などの機能を任意のActor isolated関数に対して追加するか検討された。`MainActor`のみに焦点を当てる理由は、Swift Concurrency以前には、メインスレッドを表す`MainActor`を除いて、他の「Actor」が存在しなかったためである。Swift Concurrencyがすでに存在するコードにエスケープハッチを提供することは、この提案の目標ではない。

## 参考リンク

### Forums

- [[Pitch] Unsafe Assume on MainActor](https://forums.swift.org/t/pitch-unsafe-assume-on-mainactor/63074)

### プロポーザルドキュメント

- [Unsafe Assume On MainActor](https://github.com/kavon/swift-evolution/blob/624564f1eb326c80c219d329d7f79a00b8fdf819/proposals/NNNN-assume-on-mainactor.md)
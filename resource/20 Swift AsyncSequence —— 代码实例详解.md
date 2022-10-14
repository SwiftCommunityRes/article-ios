> [AsyncSequence explained with Code Examples](https://www.avanderlee.com/concurrency/asyncsequence/)



`AsyncSequence`是并发性框架和[SE-298](https://github.com/apple/swift-evolution/blob/main/proposals/0298-asyncsequence.md)提案的一部分。它的名字意味着它是一个提供异步、顺序和迭代访问其元素的类型。换句话说：它是我们在Swift中熟悉的常规序列的一个异步变体。



就像你不会经常创建你的自定义序列一样，我不期望你经常创建一个自定义的`AsyncSequence`实现。然而，由于与[AsyncThrowingStream和AsyncStream](https://www.avanderlee.com/swift/asyncthrowingstream-asyncstream/)等类型一起使用，你很可能不得不与异步序列一起工作。因此，我将指导你使用`AsyncSequence`实例进行工作。



## 什么是 AsyncSequence?

`AsyncSequence`是我们在Swift中熟悉的`Sequence`的一个异步变体。由于它的异步性，我们需要使用` await `关键字，因为我们要处理的是异步定义的方法。如果你没有使用过`async/await`，我鼓励你阅读我的文章：[Swift 中的async/await ——代码实例详解](https://www.avanderlee.com/swift/async-await/)。



值可以随着时间的推移而变得可用，这意味着一个`AsyncSequence`在你第一次使用它时可能不包含也可能包含一些，或者全部的值。



重要的是要理解`AsyncSequence`只是一个协议。它定义了如何访问值，但并不产生或包含值。`AsyncSequence`协议的实现者提供了一个`AsyncIterator`，并负责开发和潜在地存储值。



## 创建一个自定义的 AsyncSequence

为了更好地理解`AsyncSequence`是如何工作的，我将演示一个实现实例。然而，在定义你的`AsyncSequence`的自定义实现时，你可能想用`AsyncStream`来代替，因为它的设置更方便。因此，这只是一个代码例子，以更好地理解`AsyncSequence`的工作原理。


下面的例子沿用了原始提案中的例子，实现了一个计数器。这些值可以立即使用，所以对异步序列没有太大的需求。然而，它确实展示了一个异步序列的基本结构：

```swift
struct Counter: AsyncSequence {
    typealias Element = Int

    let limit: Int

    struct AsyncIterator : AsyncIteratorProtocol {
        let limit: Int
        var current = 1
        mutating func next() async -> Int? {
            guard !Task.isCancelled else {
                return nil
            }

            guard current <= limit else {
                return nil
            }

            let result = current
            current += 1
            return result
        }
    }

    func makeAsyncIterator() -> AsyncIterator {
        return AsyncIterator(howHigh: limit)
    }
}
```

如您所见，我们定义了一个实现 `AsyncSequence` 协议的` Counter` 结构体。该协议要求我们返回一个自定义的 `AsyncIterator`，我们使用内部类型解决了这个问题。我们可以决定重写此示例以消除对内部类型的需求：

```swift
struct Counter: AsyncSequence, AsyncIteratorProtocol {
    typealias Element = Int

    let limit: Int
    var current = 1

    mutating func next() async -> Int? {
        guard !Task.isCancelled else {
            return nil
        }

        guard current <= limit else {
            return nil
        }

        let result = current
        current += 1
        return result
    }

    func makeAsyncIterator() -> Counter {
        self
    }
}
```

我们现在可以将`self`作为迭代器返回，并保持所有逻辑的集中。

注意，我们必须通过提供[typealias](https://www.avanderlee.com/swift/typealias-usage-swift/)来帮助编译器遵守`AsyncSequence`协议。



`next()`方法负责对整体数值进行迭代。我们的例子归结为提供尽可能多的计数值，直到我们达到极限。我们通过对`Task.isCancelled`的检查来实现取消支持。[你可以在这里阅读更多关于任务和取消的信息](https://www.avanderlee.com/concurrency/tasks/#handling-cancellation)。



## 异步序列的迭代

现在我们知道了什么是`AsyncSequence`以及它是如何实现的，现在是时候开始迭代这些值了。



以上述例子为例，我们可以使用`Counter`开始迭代：

```swift
for await count in Counter(limit: 5) {
    print(count)
}
print("Counter finished")

// Prints:
// 1
// 2
// 3
// 4
// 5
// Counter finished
```

我们必须使用 `await `关键字，因为我们可能会异步接收数值。一旦不再有预期的值，我们就退出for循环。异步序列的实现者可以通过在`next()`方法中返回`nil`来表示达到极限。在我们的例子中，一旦计数器达到配置的极限，或者迭代取消，我们就会达到这个预期：

```swift
mutating func next() async -> Int? {
    guard !Task.isCancelled else {
        return nil
    }

    guard current <= limit else {
        return nil
    }

    let result = current
    current += 1
    return result
}
```

许多常规的序列操作符也可用于异步序列。其结果是，我们可以以异步的方式执行映射和过滤等操作。



例如，我们可以只对偶数进行过滤：

```swift
for await count in Counter(limit: 5).filter({ $0 % 2 == 0 }) {
    print(count)
}
print("Counter finished")

// Prints: 
// 2
// 4
// Counter finished
```

或者我们可以在迭代之前将计数映射为一个`String`：

```swift
let counterStream = Counter(limit: 5)
    .map { $0 % 2 == 0 ? "Even" : "Odd" }
for await count in counterStream {
    print(count)
}
print("Counter finished")

// Prints:
// Odd
// Even
// Odd
// Even
// Odd
// Counter finished
```



我们甚至可以使用`AsyncSequence`而不使用for循环，通过使用`contains`等方法。

```swift
let contains = await Counter(limit: 5).contains(3)
print(contains) // Prints: true
```

*注意*，上述方法是异步的，意味着它有可能无休止地等待一个值的存在，直到底层的`AsyncSequence`完成。



## 继续你的Swift并发之旅

如果你喜欢你所读到的关于异步序列的内容，你可能也会喜欢其他的并发主题：

- [Sendable and @Sendable closures explained with code examples](https://www.avanderlee.com/swift/sendable-protocol-closures/)
- [AsyncSequence explained with Code Examples](https://www.avanderlee.com/concurrency/asyncsequence/)
- [AsyncThrowingStream and AsyncStream explained with code examples](https://www.avanderlee.com/swift/asyncthrowingstream-asyncstream/)
- [Tasks in Swift explained with code examples](https://www.avanderlee.com/concurrency/tasks/)
- [Async await in Swift explained with code examples](https://www.avanderlee.com/swift/async-await/)
- [Nonisolated and isolated keywords: Understanding Actor isolation](https://www.avanderlee.com/swift/nonisolated-isolated/)
- [Async let explained: call async functions in parallel](https://www.avanderlee.com/swift/async-let-asynchronous-functions-in-parallel/)
- [MainActor usage in Swift explained to dispatch to the main thread](https://www.avanderlee.com/swift/mainactor-dispatch-main-thread/)
- [Actors in Swift: how to use and prevent data races](https://www.avanderlee.com/swift/actors/)



## 结论

`AsyncSequence`是我们在Swift中熟悉的常规`Sequence`的异步替代品。就像你不会经常自己创建一个自定义`Sequence`一样，你也不太可能创建自定义的异步序列。相反，我建议你看一下[AsyncStreams](https://www.avanderlee.com/swift/asyncthrowingstream-asyncstream/)。
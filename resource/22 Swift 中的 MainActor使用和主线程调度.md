[MainActor usage in Swift explained to dispatch to the main thread](https://www.avanderlee.com/swift/mainactor-dispatch-main-thread/)



MainActor 是Swift 5.5中引入的一个新属性，它是一个全局 actor，提供一个在主线程上执行任务的执行器。在构建应用程序时，在主线程上执行UI更新任务是很重要的，在使用几个后台线程时，这有时会很有挑战性。使用`@MainActor`属性将帮助你确保你的UI总是在主线程上更新。



如果您不熟悉 Swift 中的 Actors，我建议您阅读我的文章[Swift中的Actors 使用以如何及防止数据竞争](https://www.avanderlee.com/swift/actors/)，全局Actors的行为类似于Actors，我不会在这篇文章中详细介绍Actors的工作方式。



## 什么是 MainActor?

MainActor 是一个全局唯一的 Actor，他在主线程上执行他的任务。它应该被用于属性、方法、实例和闭包，以在主线程上执行任务。提案SE-0316 全局Actor 引入了 MainActor，作为其全局 Actor 的一个例子，它继承了`GlobalActor`协议。



## 理解全局 Actors

全局 Actor 可以看作是单例：每个只有一个实例。如果你的Xcode不支持，请升级到最新版本或者通过启用实验并发来工作。您可以通过在 Xcode 的构建设置中将以下值添加到“Other Swift Flags”中来实现：

````swift
-Xfrontend -enable-experimental-concurrency
````

我们可以定义我们自己的全局 Actor 如下:

```swift
@globalActor
actor SwiftLeeActor {
    static let shared = SwiftLeeActor()
}
```


共享属性是`GlobalActor`协议的一个要求，它可以确保有一个全球唯一的角色实例。一旦被定义，你就可以在整个项目中使用全局Actor，就像你对其他 Actor 一样：

```swift
@SwiftLeeActor
final class SwiftLeeFetcher {
    // ..
}
```



## 如何在 Swift 中使用 `MainActor`？

全局actor可以与属性、方法、闭包和实例一起使用。例如，我们可以将 `MainActor `属性添加到视图模型中，以使其在主线程上执行所有任务：

```swift
@MainActor
final class HomeViewModel {
    // ..
}
```



使用[nonisolated](https://www.avanderlee.com/swift/actors/#nonisolated-access-within-actors)，我们可以确保没有主线程要求的方法尽可能快地执行。如果一个类没有父类，父类使用相同的全局actor注释，或者父类是NSObject，则只能使用全局actor进行注释。 全局 Actor 注释的类的子类必须与同一个全局 Actor 隔离。



在其他情况下，我们可能希望使用全局Actor定义单个属性：

```swift
final class HomeViewModel {
    
    @MainActor var images: [UIImage] = []

}
```

用`@MainActor`属性标记`images`属性，可以确保它只能从主线程更新:

![The MainActor attribute requirements are enforced by the compiler.](https://www.avanderlee.com/wp-content/uploads/2021/07/mainactor_global_actor-1024x195.png)

编译器执行MainActor的属性要求，可使用如下代码修复错误：

```swift
final class HomeViewModel {
    @MainActor var images: [UIImage] = []
    func updateImages() async {
        await MainActor.run {
            images = []
        }
    }
}
// OR
final class HomeViewModel {
    @MainActor var images: [UIImage] = []
    @MainActor
    func updateImages() {
        images = []
    }
}
```



单独的方法也可以用该属性进行标记：

```swift
@MainActor func updateViews() {
    // Perform UI updates..
}
```

甚至可以将闭包标记为在主线程上执行：

```swift
func updateData(completion: @MainActor @escaping () -> ()) {
    /// Example dispatch to mimic behaviour
    DispatchQueue.global().async {
        async {
            await completion()
        }
    }
}
```



## 直接使用 MainActor

Swift 中的 MainActor 带有一个可以直接使用 Actor 的扩展：

```swift
@available(macOS 12.0, iOS 15.0, watchOS 8.0, tvOS 15.0, *)
extension MainActor {

    /// Execute the given body closure on the main actor.
    public static func run<T>(resultType: T.Type = T.self, body: @MainActor @Sendable () throws -> T) async rethrows -> T
}
```

这允许我们直接在方法中使用 MainActor，即使我们没有使用全局 actor 属性定义它的任何主体：

```swift
async {
    await MainActor.run {
        // Perform UI updates
    }
}
```

换句话说，不再需要使用 `DispatchQueue.main.async`了。



## 我应该在什么时候使用MainActor属性？

在 Swift 5.5 之前，你可能定义了很多调度语句，以确保任务在主线程上运行。一个例子可能是这样的：

```swift
func fetchData(completion: @escaping (Result<[UIImage], Error>) -> Void) {
    URLSession.shared.dataTask(with: URL(string: "..some URL")) { data, response, error in
        // .. Decode data to a result
        
        DispatchQueue.main.async {
            completion(result)
        }
    }
} 
```

在上面的例子中，我们很确定需要一个调度。然而，在其他情况下，调度可能是不必要的，因为我们已经在主线程上。这样做会导致额外的调度被跳过。



无论哪种方式，在这些情况下，将属性、方法、实例或闭包定义为一个主行为体是有意义的，以确保任务在主线程上执行。例如，我们可以把上面的例子改写成如下:

```swift
func fetchData(completion: @MainActor @escaping (Result<[UIImage], Error>) -> Void) {
    URLSession.shared.dataTask(with: URL(string: "..some URL")!) { data, response, error in
        // .. Decode data to a result
        let result: Result<[UIImage], Error> = .success([])
        
        async {
            await completion(result)
        }
    }
}
```

由于我们现在使用的是一个actor定义的闭包，我们需要使用 async-await 技术来调用我们的闭包。在这里使用`@MainActor`属性可以让Swift编译器对我们的代码进行性能优化。



### 选择正确的策略

使用 actors 时选择正确的策略很重要。在上面的例子中，我们决定让闭包成为一个actor，这意味着无论谁使用我们的方法，完成回调都将使用 `MainActor` 执行。在某些情况下，如果数据请求方法也是从一个不需要在主线程上处理完成回调的地方使用，这可能就没有意义了。



在这些情况下，让实现者负责调度到正确的队列可能会更好。

```swift
viewModel.fetchData { result in
    async {
        await MainActor.run {
            // Handle result
        }
    }
}
```



继续你的Swift并发之旅

并发的变化不仅仅是 async-await，还包括许多新的功能，你可以从你的代码中受益。所以，当你在做这件事的时候，为什么不深入研究一下其他的并发功能呢？

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

全局Actor是对Swift中的Actor的一个很好的补充。它允许我们重用常见的Actor，并使UI任务的执行成为可能，因为编译器可以在内部优化我们的代码。全局Actor可以用在属性、方法、实例和闭包上，之后编译器会确保要求在我们的代码中得到保证。

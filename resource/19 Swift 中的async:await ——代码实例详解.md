## 前言

async-await 是在 WWDC 2021 期间的 Swift 5.5 中的结构化并发变化的一部分。Swift 中的并发性意味着允许多段代码同时运行。这是一个非常简化的描述，但它应该让你知道 Swift 中的并发性对你的应用程序的性能是多么重要。有了新的 async 方法和 await 语句，我们可以定义方法来进行异步工作。

你可能读过 Chris Lattner 的 Swift 并发性宣言 [Swift Concurrency Manifesto by Chris Lattner](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782 "Swift Concurrency Manifesto by Chris Lattner")，这是在几年前发布的。Swift社区的许多开发者对未来将出现的定义异步代码的结构化方式感到兴奋。现在它终于来了，我们可以用 async-await 简化我们的代码，使我们的异步代码更容易阅读。

## 什么是 async?

async 是异步的意思，可以看作是一个明确表示一个方法是执行异步工作的一个属性。这样一个方法的例子看起来如下：

```swift
func fetchImages() async throws -> [UIImage] {
    // ..  执行数据请求
}
```

`fetchImages` 方法被定义为异步且可以抛出异常，这意味着它正在执行一个可失败的异步作业。如果一切顺利，该方法将返回一组图像，如果出现问题，则抛出错误。

## async 如何取代完成回调闭包

async 方法取代了经常看到的完成回调。完成回调在 Swift 中很常见，用于从异步任务中返回，通常与一个结果类型的参数相结合。上述方法一般会被写成这样:

```swift
func fetchImages(completion: (Result<[UIImage], Error>) -> Void) {
    // .. 执行数据请求
}
```

在如今的 Swift 版本中，使用完成闭包来定义方法仍然是可行的，但它有一些缺点，async 却刚好可以解决。

- 你必须确保自己在每个可能的退出方法中调用完成闭包。如果不这样做，可能会导致应用程序无休止地等待一个结果。
- 闭包代码比较难阅读。与结构化并发相比，对执行顺序的推理并不那么容易。
- 需要使用弱引用 `weak references` 来避免循环引用。
- 实现者需要对结果进行切换以获得结果。无法从实现层面使用 `try catch` 语句。

这些缺点是基于使用相对较新的 `Result` 枚举的闭包版本。很可能很多项目仍然在使用完成回调，而没有使用这个枚举：

```swift
func fetchImages(completion: ([UIImage]?, Error?) -> Void) {
    // .. 执行数据请求
}
```

像这样定义一个方法使我们很难推理出调用者一方的结果。`value` 和 `error` 都是可选的，这要求我们在任何情况下都要进行解包。对这些可选项解包会导致更多的代码混乱，这对提高可读性没有帮助。

## 什么是 await？

await 是用于调用异步方法的关键字。你可以把它们 (async-await) 看作是 Swift 中最好的朋友，因为一个永远不会离开另一个，你基本上可以这样说：

> "Await 正在等待来自他的伙伴 async 的回调"

尽管这听起来很幼稚，但这并不是骗人的! 我们可以通过调用我们先前定义的异步方法 `fetchImages` 方法来看一个例子：

```swift
do {
    let images = try await fetchImages()
    print("Fetched \(images.count) images.")
} catch {
    print("Fetching images failed with error \(error)")
}
```

也许你很难相信，但上面的代码例子是在执行一个异步任务。使用 `await` 关键字，我们告诉我们的程序等待 `fetchImages` 方法的结果，只有在结果到达后才继续。这可能是一个图像集合，也可能是一个在获取图像时出了什么问题的错误。

## 什么是结构化并发？

使用 async-await 方法调用的结构化并发使得执行顺序的推理更加容易。方法是线性执行的，不用像闭包那样来回走动。

为了更好地解释这一点，我们可以看看在结构化并发到来之前，我们如何调用上述代码示例：

```swift
// 1. 调用这个方法
fetchImages { result in
    // 3. 异步方法内容返回
    switch result {
    case .success(let images):
        print("Fetched \(images.count) images.")
    case .failure(let error):
        print("Fetching images failed with error \(error)")
    }
}
// 2. 调用方法结束
```

正如你所看到的，调用方法在获取图像之前结束。最终，我们收到了一个结果，然后我们回到了完成回调的流程中。这是一个非结构化的执行顺序，可能很难遵循。如果我们在完成回调中执行另一个异步方法，毫无疑问这会增加另一个闭包回调：

```swift
// 1. 调用这个方法
fetchImages { result in
    // 3. 异步方法内容返回
    switch result {
    case .success(let images):
        print("Fetched \(images.count) images.")
        
        // 4. 调用 resize 方法
        resizeImages(images) { result in
            // 6. Resize 方法返回
            switch result {
            case .success(let images):
                print("Decoded \(images.count) images.")
            case .failure(let error):
                print("Decoding images failed with error \(error)")
            }
        }
        // 5. 获图片方法返回
    case .failure(let error):
        print("Fetching images failed with error \(error)")
    }
}
// 2. 调用方法结束
```

每一个闭包都会增加一层缩进，这使得我们更难理解执行的顺序。

通过使用 async-await 重写上述代码示例，最好地解释了结构化并发的作用。

```swift
do {
    // 1. 调用这个方法
    let images = try await fetchImages()
    // 2.获图片方法返回
    
    // 3. 调用 resize 方法
    let resizedImages = try await resizeImages(images)
    // 4.Resize 方法返回
    
    print("Fetched \(images.count) images.")
} catch {
    print("Fetching images failed with error \(error)")
}
// 5. 调用方法结束
```

执行的顺序是线性的，因此，容易理解，容易推理。当我们有时还在执行复杂的异步任务时，理解异步代码会更容易。

## 调用异步方法

在一个不支持并发的函数中调用异步方法

在第一次使用 async-await 时，你可能会遇到这样的错误。

![](https://files.mdnice.com/user/17787/c1dc832f-7da0-4c20-b62d-647e8975efb8.jpg)

当我们试图从一个不支持并发的同步调用环境中调用一个异步方法时，就会出现这个错误。我们可以通过将我们的 `fetchData` 方法也定义为异步来解决这个错误：

```swift
func fetchData() async {
    do {
        try await fetchImages()
    } catch {
        // .. handle error
    }
}
```

然而，这将把错误转移到另一个地方。相反，我们可以使用 `Task.init` 方法，从一个支持并发的新任务中调用异步方法，并将结果分配给我们视图模型中的一个属性：

```swift
final class ContentViewModel: ObservableObject {
    
    @Published var images: [UIImage] = []
    
    func fetchData() {
        Task.init {
            do {
                self.images = try await fetchImages()
            } catch {
                // .. handle error
            }
        }
    }
}
```

使用尾随闭包的异步方法，我们创建了一个环境，在这个环境中我们可以调用异步方法。一旦异步方法被调用，获取数据的方法就会返回，之后所有的异步回调都会在闭包内发生。

## 采用 async-await

在一个现有项目中采用 async-await

当在现有项目中采用 async-await 时，你要注意不要一下子破坏所有的代码。在进行这样的大规模重构时，最好考虑暂时维护旧的实现，这样你就不必在知道新的实现是否足够稳定之前更新所有的代码。这与 SDK 中被许多不同的开发者和项目所使用的废弃方法类似。

显然，你没有义务这样做，但它可以使你更容易在你的项目中尝试使用 async-await。除此之外，Xcode 使重构你的代码变得超级容易，还提供了一个选项来创建一个单独的  async 方法：

![](https://files.mdnice.com/user/17787/22ebbd1a-ea91-44ab-8f58-249273ad5be6.jpg)

每个重构方法都有自己的目的，并导致不同的代码转换。为了更好地理解其工作原理，我们将使用下面的代码作为重构的输入：

```swift
struct ImageFetcher {
    func fetchImages(completion: @escaping (Result<[UIImage], Error>) -> Void) {
        // .. 执行数据请求
    }
}
```

### 将函数转换为异步 (Convert Function to Async)

第一个重构选项将 `fetchImages` 方法转换为异步变量，而不保留非异步变量。如果你不想保留原来的实现，这个选项将很有用。结果代码如下：

```swift
struct ImageFetcher {
    func fetchImages() async throws -> [UIImage] {
        // .. 执行数据请求
    }
}
```

### 添加异步替代方案 (Add Async Alternative)

添加异步替代重构选项确保保留旧的实现，但会添加一个可用（available） 属性：

```swift
struct ImageFetcher {
    @available(*, renamed: "fetchImages()")
    func fetchImages(completion: @escaping (Result<[UIImage], Error>) -> Void) {
        Task {
            do {
                let result = try await fetchImages()
                completion(.success(result))
            } catch {
                completion(.failure(error))
            }
        }
    }


    func fetchImages() async throws -> [UIImage] {
        // .. 执行数据请求
    }
}
```

可用属性对于了解你需要在哪里更新你的代码以适应新的并发变量是非常有用的。虽然，Xcode 提供的默认实现并没有任何警告，因为它没有被标记为废弃的。要做到这一点，你需要调整可用标记，如下所示:

```swift
@available(*, deprecated, renamed: "fetchImages()")
```

使用这种重构选项的好处是，它允许你逐步适应新的结构化并发变化，而不必一次性转换你的整个项目。在这之间进行构建是很有价值的，这样你就可以知道你的代码变化是按预期工作的。利用旧方法的实现将得到如下的警告。

![](https://files.mdnice.com/user/17787/c95cee83-d4bc-4954-b426-d24b1e631af7.jpg)

你可以在整个项目中逐步改变你的实现，并使用Xcode中提供的修复按钮来自动转换你的代码以利用新的实现。

### 添加异步包装器 (Add Async Wrapper)

最后的重构方法将使用最简单的转换，因为它将简单地利用你现有的代码：

```swift
struct ImageFetcher {
    @available(*, renamed: "fetchImages()")
    func fetchImages(completion: @escaping (Result<[UIImage], Error>) -> Void) {
        // .. 执行数据请求
    }

    func fetchImages() async throws -> [UIImage] {
        return try await withCheckedThrowingContinuation { continuation in
            fetchImages() { result in
                continuation.resume(with: result)
            }
        }
    }
}
```

新增加的方法利用了 Swift 中引入的 `withCheckedThrowingContinuation` 方法，可以不费吹灰之力地转换基于闭包的方法。不抛出的方法可以使用 `withCheckedContinuation`，其工作原理与此相同，但不支持抛出错误。

这两个方法会暂停当前任务，直到给定的闭包被调用以触发 async-await 方法的继续。换句话说：你必须确保根据你自己的基于闭包的方法的回调来调用 `continuation` 闭包。在我们的例子中，这归结为用我们从最初的  `fetchImages` 回调返回的结果值来调用继续。

### 为你的项目选择正确的 async-await 重构方法

这三个重构选项应该足以将你现有的代码转换为异步的替代品。根据你的项目规模和你的重构时间，你可能想选择一个不同的重构选项。不过，我强烈建议逐步应用改变，因为它允许你隔离改变的部分，使你更容易测试你的改变是否如预期那样工作。

## 解决错误

解决 "Reference to captured parameter ‘self’ in concurrently-executing code "错误

在使用异步方法时，另一个常见的错误是下面这个:

> “Reference to captured parameter ‘self’ in concurrently-executing code”

这大致意思是说我们正试图引用一个不可变的`self`实例。换句话说，你可能是在引用一个属性或一个不可变的实例，例如，像下面这个例子中的结构体：

![](https://files.mdnice.com/user/17787/f9eed14b-6840-4b26-9c02-d25c72d8e9d8.jpg)

不支持从异步执行的代码中修改不可变的属性或实例。

可以通过使属性可变或将结构体更改为引用类型（如类）来修复此错误。

## 枚举的终点

async-await 将是`Result`枚举的终点吗？

我们已经看到，异步方法取代了利用闭包回调的异步方法。我们可以问自己，这是否会是 Swift 中 [Result 枚举](https://www.avanderlee.com/swift/result-enum-type/ "Result 枚举")的终点。最终我们会发现，我们真的不再需要它们了，因为我们可以利用 try-catch 语句与 async-await 相结合。

`Result` 枚举不会很快消失，因为它仍然在整个 Swift 项目的许多地方被使用。然而，一旦 async-await 的采用率越来越高，我就不会惊讶地看到它被废弃。就我个人而言，除了完成回调，我没有在其他地方使用结果枚举。一旦我完全使用 async-await，我就不会再使用这个枚举了。

## 结论

Swift 中的 async-await 允许结构化并发，这将提高复杂异步代码的可读性。不再需要完成闭包，而在彼此之后调用多个异步方法的可读性也大大增强。一些新的错误类型可能会发生，通过确保异步方法是从支持并发的函数中调用的，同时不改变任何不可变的引用，这些错误将可以得到解决。

> 来自：[Async await in Swift explained with code examples](https://www.avanderlee.com/swift/async-await/)
- [AsyncThrowingStream and AsyncStream explained with code examples](https://www.avanderlee.com/swift/asyncthrowingstream-asyncstream/)



`AsyncThrowingStream` 和 `AsyncStream`是Swift 5.5中由[SE-314](https://github.com/apple/swift-evolution/blob/main/proposals/0314-async-stream.md)引入的并发框架的一部分。异步流允许你替换基于闭包或 Combine 发布器的现有代码。



在深入研究围绕抛出流的细节之前，如果你还没有阅读我的文章，我建议你先阅读我的文章，内容包括async-await。本文解释的大部分代码将使用那里解释的API。



## 什么是 AsyncThrowingStream？

你可以把 `AsyncThrowingStream` 看作是一个有可能导致抛出错误的元素流。他的值随着时间的推移而传递，流可以通过一个结束事件来关闭。一旦发生错误，结束事件既可以是成功，也可以是失败。



## 什么是 AsyncStream?

`AsyncStream` 类似于抛出的变体，但绝不会导致抛出错误。一个非抛出型的异步流会根据明确的完成调用或流的取消而完成。



*在这篇文章中，我们将解释如何使用`AsyncThrowingStream`。除了发生错误处理的部分，代码示例与`AsyncStream`类似。*



## 如何使用 AsyncThrowingStream

`AsyncThrowingStream`可以很好地替代现有的基于闭包的代码，如进度和完成处理程序。为了更好地理解我的意思，我将向你介绍我们在 WeTransfer 应用程序中遇到的一个场景。



在我们的应用程序中，我们有一个基于闭包的现有类，叫做`FileDownloader`：

```swift
struct FileDownloader {
    enum Status {
        case downloading(Float)
        case finished(Data)
    }

    func download(_ url: URL, progressHandler: (Float) -> Void, completion: (Result<Data, Error>) -> Void) throws {
        // .. Download implementation
    }
}
```

文件下载器接受一个URL，报告进度情况，并完成一个包含下载数据的结果或在失败时显示一个错误。



文件下载器在文件下载过程中报告一个数值流。在这种情况下，它报告的是一个状态值流，以报告正在运行的下载的当前状态。`FileDownloader`是一个完美的例子，你可以重写一段代码来使用`AsyncThrowingStream`。然而，重写需要你在实现层面上也重写你的代码，所以让我们定义一个重载方法来代替：

```swift
extension FileDownloader {
    func download(_ url: URL) -> AsyncThrowingStream<Status, Error> {
        return AsyncThrowingStream { continuation in
            do {
                try self.download(url, progressHandler: { progress in
                    continuation.yield(.downloading(progress))
                }, completion: { result in
                    switch result {
                    case .success(let data):
                        continuation.yield(.finished(data))
                        continuation.finish()
                    case .failure(let error):
                        continuation.finish(throwing: error)
                    }
                })
            } catch {
                continuation.finish(throwing: error)
            }
        }
    }
}
```

正如你所看到的，我们把下载方法包裹在一个`AsyncThrowingStream`里面。我们将流的值`Status`的类型描述为一个通用的类型，允许我们用状态更新来延续流。



只要有错误发生，我们就会通过抛出一个错误来完成流。在完成处理程序的情况下，我们要么通过抛出一个错误来完成，要么用一个不抛出的完成回调来跟进数据的产生。

```swift
switch result {
case .success(let data):
    continuation.yield(.finished(data))
    continuation.finish()
case .failure(let error):
    continuation.finish(throwing: error)
}
```

在收到最后的状态更新后，不要忘记`finish()`回调，这一点至关重要。否则，我们将保持流的存活，而实现层面的代码将永远不会继续。



我们可以通过使用另一个`yield`方法来重写上述代码，接受一个`Result`枚举作为参数：

```swift
continuation.yield(with: result.map { .finished($0) })
continuation.finish()
```

重写后的代码简化了我们的代码，并去掉了switch-case 代码。我们必须映射我们的`Reslut`枚举以匹配预期的`Status`值。如果我们产生一个失败的结果，我们的流将在抛出包含的错误后结束。



## AsyncThrowingStream 迭代

一旦你配置好你的异步抛出流，你就可以开始在数值流上进行迭代。在我们的`FileDownloader`例子中，它将看起来如下所示：

```swift
do {
    for try await status in download(url) {
        switch status {
        case .downloading(let progress):
            print("Downloading progress: \(progress)")
        case .finished(let data):
            print("Downloading completed with data: \(data)")
        }
    }
    print("Download finished and stream closed")
} catch {
    print("Download failed with \(error)")
}
```



我们处理任何状态的更新，并且我们可以使用`catch`闭包来处理任何发生的错误。你可以使用基于`AsyncSequence`接口的`for ... in`循环进行迭代，这对`AsyncStream`来说是一样的。



*如果你遇到了类似的编译错误:*

> ‘async’ in a function that does not support concurrency

你可能想读一读我的文章，其中[深入介绍了async-await](https://www.avanderlee.com/swift/async-await/)。



上述代码示例中的打印语句有助于你理解 `AsyncThrowingStream`的生命周期。你可以替换打印语句来处理进度更新和处理数据，为你的用户实现可视化。



## 调试 AsyncStream

如果一个流不能报告数值，我们可以通过放置断点来调试流产生的回调。虽然也可能是上面的*“Download finished and stream closed”* 的打印语句不会调用，这意味着你在实现层的代码永远不会继续。后者可能是一个未完成的流的结果。



为了验证，我们可以利用`onTermination`回调：

```swift
func download(_ url: URL) -> AsyncThrowingStream<Status, Error> {
    return AsyncThrowingStream { continuation in

        ///  配置一个终止回调，以了解你的流的生命周期。
        continuation.onTermination = { @Sendable status in
            print("Stream terminated with status \(status)")
        }

        // ..
    }
}
```

回调在流终止时被调用，它将告诉你你的流是否还活着。我推荐你阅读[Sendable 和 @Sendable 闭包——代码实例详解](https://www.avanderlee.com/swift/sendable-protocol-closures/)来理解`@Sendable`属性。



如果出现了错误，输出结果可能如下:

```swift
Stream terminated with status finished(Optional(FileDownloader.FileDownloadingError.example))
```

上述输出只有在使用`AsyncThrowingStream`时才能实现。如果是一个普通的`AsyncStream`，完成的输出看起来如下：

```swift
Stream terminated with status finished
```

而取消的结果对这两种类型的流来说都是这样的:

```swift
Stream terminated with status cancelled
```

你也可以在流结束后使用这个终止回调进行任何清理。例如，删除任何观察者或在文件下载后清理磁盘空间。



## 取消一个 AsyncStream

一个`AsyncStream`或`AsyncThrowingStream`可以由于一个封闭的任务被取消而取消。一个例子可以如下：

```swift
let task = Task.detached {
    do {
        for try await status in download(url) {
            switch status {
            case .downloading(let progress):
                print("Downloading progress: \(progress)")
            case .finished(let data):
                print("Downloading completed with data: \(data)")
            }
        }
    } catch {
        print("Download failed with \(error)")
    }
}
task.cancel()
```

一个流在超出范围或包围的任务取消时就会取消。如前所述，取消将相应地触发`onTermination`回调。



## 继续你的Swift并发之旅

如果你喜欢你所读到的关于异步流的内容，你可能也会喜欢其他的并发主题:

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

`AsyncThrowingStream`或`AsyncStream`是重写基于闭包的现有代码到支持 async-awai t的替代品的好方法。你可以提供一个连续的值流，并在成功或失败时完成一个流。你可以使用基于`AsyncSequence` APIs的 for 循环在实现层面上迭代值。
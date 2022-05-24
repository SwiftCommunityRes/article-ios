## 前言

Swift 内置并发系统的好处之一是它可以更轻松地并行执行多个异步任务，这反过来又可以使我们显着加快可以分解为单独部分的操作。

在本文中，让我们看一下几种不同的方法，以及这些技术中的每一种何时特别有用。

## 从异步到并发

首先，假设我们正在开发某种形式的购物应用程序来显示各种产品，并且我们已经实现了一个`ProductLoader`允许我们使用一系列异步 API 加载不同产品集合的应用程序，如下所示：

```swift
class ProductLoader {
    ...

    func loadFeatured() async throws -> [Product] {
        ...
    }
    
    func loadFavorites() async throws -> [Product] {
        ...
    }
    
    func loadLatest() async throws -> [Product] {
        ...
    }
}
```

尽管大多数情况下上述每个方法都可能会被单独调用，但假设在我们应用程序的某些部分中，我们还希望形成一个`Recommendations`包含这三个`ProductLoader`方法的所有结果的组合模型：

```swift
extension Product {
    struct Recommendations {
        var featured: [Product]
        var favorites: [Product]
        var latest: [Product]
    }
}
```

一种方法是使用`await`关键字调用每个加载方法，然后使用这些调用的结果来创建我们`Recommendations`模型的实例——如下所示：

```swift
extension ProductLoader {
    func loadRecommendations() async throws -> Product.Recommendations {
        let featured = try await loadFeatured()
let favorites = try await loadFavorites()
let latest = try await loadLatest()
        
        return Product.Recommendations(
            featured: featured,
            favorites: favorites,
            latest: latest
        )
    }
}
```

上面的实现确实有效——然而，即使我们的三个加载操作都是完全异步的，它们目前正在*按顺序*执行，一个接一个。因此，尽管我们的顶级`loadRecommendations`方法相对于我们应用程序的其他代码正在并发执行，但实际上它还没有利用并发来执行其内部操作集。

由于我们的产品加载方法不以任何方式相互依赖，因此实际上没有理由按顺序执行它们，所以让我们看看如何让它们完全同时执行。

关于如何做到这一点的初步想法可能是将上述代码简化为单个表达式，这将使我们能够使用单个`await`关键字来等待我们的每个操作完成：

```swift
extension ProductLoader {
    func loadRecommendations() async throws -> Product.Recommendations {
        try await Product.Recommendations(
            featured: loadFeatured(),
            favorites: loadFavorites(),
            latest: loadLatest()
        )
    }
}
```

然而，即使我们的代码现在*看起来是*并发的，它实际上仍会像以前一样完全按顺序执行。

相反，我们需要利用 Swift 的`async let`绑定来告诉并发系统并行执行我们的每个加载操作。使用该语法使我们能够在后台启动异步操作，而无需我们立即等待它完成。

`await`如果我们在实际*使用*加载的数据时（即形成模型时）将其与单个关键字组合`Recommendations`，那么我们将获得并行执行加载操作的所有好处，而无需担心状态管理或数据竞争之类的事情：

```swift
extension ProductLoader {
    func loadRecommendations() async throws -> Product.Recommendations {
        async let featured = loadFeatured()
async let favorites = loadFavorites()
async let latest = loadLatest()
        
        return try await Product.Recommendations(
            featured: featured,
            favorites: favorites,
            latest: latest
        )
    }
}
```

很整齐！因此`async let`，当我们有一组已知的、有限的任务要执行时，它提供了一种同时运行多个操作的内置方法。但如果不是这样呢？

## 任务组

现在假设我们正在开发一个`ImageLoader`可以让我们通过网络加载图像的工具。要从给定的 加载单个图像`URL`，我们可以使用如下所示的方法：

```swift
class ImageLoader {
    ...

    func loadImage(from url: URL) async throws -> UIImage {
        ...
    }
}
```

为了使一次加载一系列图像变得简单，我们还创建了一个方便的 API，它接受一个 URL 数组并异步返回一个图像字典，该字典由下载图像的 URL 键控：

```swift
extension ImageLoader {
    func loadImages(from urls: [URL]) async throws -> [URL: UIImage] {
        var images = [URL: UIImage]()
        
        for url in urls {
            images[url] = try await loadImage(from: url)
        }
        
        return images
    }
}
```

现在让我们说，就像我们`ProductLoader`之前的工作一样，我们想让上面的`loadImages`方法并发执行，而不是按顺序下载每个图像（目前是这种情况，因为我们`await`在调用时直接使用`loadImage`我们的`for`环形）。

但是，这次我们将无法使用`async let`，因为我们需要执行的任务数量在编译时是未知的。值得庆幸的是，Swift 并发工具箱中还有一个工具可以让我们并行执行动态数量的任务——*任务组*。

要形成一个任务组，我们可以调用`withTaskGroup`或`withThrowingTaskGroup`，这取决于我们是否希望可以选择在我们的任务中抛出错误。在这种情况下，我们将选择后者，因为我们的底层`loadImage`方法是用`throws`关键字标记的。

然后我们将遍历每个 URL，就像以前一样，只是这次我们将每个图像加载任务添加到我们的组中，而不是直接等待它完成。相反，我们将`await`在添加每个任务之后单独分组结果，这将允许我们的图像加载操作完全并发执行：

```swift
extension ImageLoader {
    func loadImages(from urls: [URL]) async throws -> [URL: UIImage] {
        try await withThrowingTaskGroup(of: (URL, UIImage).self) { group in
            for url in urls {
                group.addTask{
    let image = try await self.loadImage(from: url)
    return (url, image)
} 
            }
            
            var images = [URL: UIImage]()
            
            for try await (url, image) in group {
    images[url] = image
}
            
            return images
        }
    }
}
```

要了解有关上述`for try await`语法和一般异步序列的更多信息，请查看[“异步序列、流和组合”](https://www.swiftbysundell.com/articles/async-sequences-streams-and-combine)。

就像使用 时一样`async let`，以我们的操作不会直接改变任何状态的方式编写并发代码的一个巨大好处是，这样做可以让我们完全避免任何类型的数据竞争问题，同时也不需要我们引入任何锁定或序列化代码混合在一起。

`await`因此，在可能的情况下，让我们的每个并发操作返回一个完全独立的结果，然后依次返回这些结果以形成我们的最终数据集，这通常是一种很好的方法。

在以后的文章中，我们将更仔细地研究避免数据竞争的其他方法（例如通过使用 Swift 的新`actor`类型）。

## 结论

重要的是要记住，仅仅因为给定的函数被标记为`async`并不一定意味着它同时执行它的工作。相反，如果这是我们想要做的，我们必须故意让我们的任务并行运行，这只有在执行一组可以独立运行的操作时才有意义。

 
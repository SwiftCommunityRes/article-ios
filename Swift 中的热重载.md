## 前言

我们最新的 MacBook M30X 处理器可以感知到瞬间编译大型 Swift 项目，除此之外，编译代码库只是我们迭代周期的一部分。包括：

- 重新启动它（或将其部署到设备）
- 导航到您在应用程序中的先前位置
- 重新生成您需要的数据。

如果您只需要做一次的话，听起来还不错。但是如果您和我一样，在特别的一天中，对代码库进行 200 - 500 次迭代，该怎么办呢？它增加了。

有一种更好的方法，被其他平台所接受，并且可以在 Swift/iOS 生态系统中实现。我已经用了十多年了。

从今天开始，您想每周节省多达 10 小时的工作时间吗？

## 热重载

热重载是关于摆脱编译整个应用程序并尽可能避免部署/重新启动周期，同时允许您编辑正在运行的应用程序代码并且能立即看到更改。

这种流程改进可以每天为您节省数小时的开发时间。我跟踪我的工作一个多月，对我来说，每天节省了 1-2 小时。

坦白地说，如果每周节省10个小时的开发时间都不能说服您去尝试，那么我认为任何方法都不能说服你。

## 其他平台在做什么？

如果您只使用 Apple 平台，您会惊讶地发现有好多平台几十年前已经采用了热重载。无论您是编写 Node 还是任何其他 JS 框架，都有一个使用热重载的设置。 Go 也提供了热重载（本博客使用了该特性）

另一个例子是谷歌的 Flutter 架构，从一开始就设计用于热重载。如果您与从事 Flutter 工作的工程师交谈，你会发现他们最喜欢 Flutter 开发者体验的一点就是能够实时编写他们的应用程序。当我为《纽约时报》写了一个拼字游戏时，我很喜欢它。

微软最近推出了 [Visual Studio 2022](https://devblogs.microsoft.com/visualstudio/visual-studio-2022-now-available/)，并为 .NET 和 标准 C++ 应用程序提供热重载，在过去的十年中，微软在开发工具和经验方面一直在大杀四方，所以这并不令人惊讶。

## 苹果生态系统怎么样？

早在 2014 年推出时，很多人都对 Swift Playgrounds 感到敬畏，因为它们允许我们快速迭代并查看代码的结果，但它们并不能很好地工作，因为它存在崩溃、挂起等问题。不能支持整个iPad环境。

在它们发布后不久，我启动了一个名为 Objective-C Playgrounds 的开源项目，它比官方 Playgrounds 运行得更快、更可靠。我的想法是设计一个架构/工作流程，利用我已经使用了几年的 [DyCI](https://github.com/DyCI/dyci-main)  代码注入工具，该工具已经由 Paul 制作。

自从 Swift Playgrounds 存在以来，已经过去了八年，而且它们变得更好了，但它们可靠吗？人们是否在使用它们来推动开发？

>*以我的经验：并非如此。Playgrounds 在大型项目中往往不太可靠或适用。*

SwiftUI 出现了，它是一项了不起的技术（尽管仍然存在错误），它引入了与 Playgrounds 非常相似的 Swift Previews 的想法，它们有什么好处吗？

类似的故事，当它工作的时候是很好的，但是在更大的项目中，它的工作是不可靠的，而且往往中断的次数比它们工作的次数多。如果你有任何错误，他们不会为你提供调试代码的能力，因此，采用的情况有限。

## 我们需要等待 Apple 吗？

如果你关注我一段时间，你就已经知道答案了，绝对不要。毕竟，我的职业生涯是构建普通 Apple 解决方案无法解决的问题：从像 [Sourcery](https://github.com/krzysztofzablocki/Sourcery) 这样的语言扩展、像 [Sourcery Pro](http://merowing.info/sourcery-pro/) 这样的 Xcode 改进，再到 [LifetimeTracker](https://github.com/krzysztofzablocki/LifetimeTracker) 以及许多其他开源工具。

我们可以利用我最初在 2014 Playgrounds 中使用的相同方法。我已经使用它十多年了，并且在数十个 Swift 项目中使用它并取得了巨大的成功！

许多年前，我从使用  [DyCI](https://github.com/DyCI/dyci-main "DyCI")  切换到 **InjectionForXcode**，通过利用 LLVM 互操作而不是任何 swizzling ，它的效果更好。它是一个完全免费的开源工具，您可以在菜单栏中运行，它是由多产的工程师 John Holdsworth 创建的。你应该看看他的书 [Swift Secrets](http://books.apple.com/us/book/id1551005489 "Swift Secrets")。

我意识到 [Playgrounds](https://github.com/krzysztofzablocki/Playgrounds) 的方法可能过于笨重，所以今天，我开源了。一个非常专注的名为 [Inject](https://github.com/krzysztofzablocki/Inject) 的微型库，与 [InjectionForXcode](https://github.com/johnno1962/InjectionIII) 搭配使用时，将使您的 Apple 开发更加高效和愉快！

但不要只相信我的话。看看 Alexandra 和 Nate 的反馈，在我将这个工作流程引入  [The Browser Company](https://thebrowser.company/) 设置之前，他们已经非常精通了，这使得它更加令人印象深刻。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8c732bf44986441199f6d24eb84b64a5~tplv-k3u1fbpfcp-zoom-1.image)

## Inject

这个小型库是完全通用的，无论您使用 `UIKit`、 `AppKit` 还是 `SwiftUI`，您都可以使用它。

您无需为生产应用程序添加条件或删除 `Inject` 代码。它变成了无操作内联代码，将在非调试版本中被编译过程剥离。您可以在每个视图中集成一次，并持续使用数年。

请参考 [GitHub repo](https://github.com/krzysztofzablocki/Inject "GitHub repo") 中关于配置项目的说明。现在让我们来看看您有哪些工作流程选项。

## 工作流

### SwiftUI

只需要两行字就可以使任何 SwiftUI 启用实时编程，而当您这样做时，您将拥有比使用 Swift Previews 更快的工作流程，同时能够使用实际的生产数据。

这是我的 [Sourcery Pro](http://merowing.info/sourcery-pro/ "Sourcery Pro") 应用程序的示例，其中加载了我所有的实际数据和逻辑，使我能够即时快速迭代整个应用程序设计，而无需任何重新启动、重新加载或类似的事情。

看看这个开发工作流程有多快吧，告诉我你宁愿在我每次接触代码时等待Xcode的重新构建和重新部署。

### UIKit / AppKit

我们需要一种方法来清理标准命令式UI框架的代码注入阶段之间的状态。

我创建了 `Host` 的概念并且在这种情况下工作的很好。有两个：

```shell
- Inject.ViewHost
- Inject.ViewControllerHost
```

我们如何集成它？我们把我们想迭代的类包装在父级，因此我们不修改要注入的类型，而是改变父级的调用站点。

例如，如果你有一个 SplitViewController ，它创建了 PaneA 和 PaneB ，而你想在PaneA 中迭代布局/逻辑代码，你就修改 SplitViewController 中的调用站点。

```swift
paneA = Inject.ViewHost(
  PaneAView(whatever: arguments, you: want)
)
```

这就是你需要做的所有改变。注入现在允许你更改 PaneAView 中的任何东西，除了它的初始化API。这些变化将立即反映在你的应用程序中。

***

一个更具体的例子？

- 我下载了 [Covid19 App](https://github.com/dkhamsing/covid19.swift)

- 添加 `-Xlinker -interposable` 到 `Other Linker Flags`

- 交换了一行 `Covid19TabController.swift:L63` 行
  
从这句：

```swift
let vc = TwitterViewController(title: Tab.twitter.name, usernames: Twitter.content)
```

替换为：
  
```swift
let vc = Inject.ViewControllerHost(TwitterViewController(title: Tab.twitter.name, usernames: Twitter.content))
```

现在，我可以在不重新启动应用程序的情况下迭代控制器设计。

### 这是如何运作的呢?

Hosts 利用了自动闭包，因此每次您注入代码时，我们都会使用与最初相同的参数创建您类型的新实例，从而允许您迭代任何代码、内存布局和其他所有内容。你唯一不能改变的是你的初始化 API。

> Host 的变化不能完全内联，所以这些类在 Release 构建中被删除。最简单的方法是做一个单独的提交，交换此单行代码，然后在工作流程的最后删除它。

### 逻辑注入如何呢？

像 MVVM / MVC 这样的标准架构可以获得免费的逻辑注入，重新编译你的类，当方法重新执行时，你已经在使用新代码了。

如果像我一样，你喜欢 [PointFree Composable Architecture](https://github.com/pointfreeco/swift-composable-architecture "PointFree Composable Architecture")，你可能想要注入 reducer 代码。 Vanilla TCA 不允许这样做，因为 reducer 代码是一个免费功能，不能直接用注入替换，但我们在 The Browser Company 的分支 支持它。

当我最初开始咨询 TBC 时，我想要的第一件事是将 `Inject` 和 `XcodeInjection` 集成到我们的工作流程中。公司管理层非常支持。

如果您切换到我们的 [TCA 分支](https://github.com/thebrowsercompany/swift-composable-architecture/tree/develop)(我们保持最新)，你可以在 UI 和 TCA 层上使用 `Inject` 。

### 它有多可靠？

没有什么是完美的，但我已经使用它十多年了。它比 Apple 技术（Playgrounds / Previews）可靠得多。

如果您投入时间学习它，它将为您和您的团队节省数千小时！

### Demo 源码

[在 GitHub 上获取项目](https://github.com/krzysztofzablocki/Inject "Demo 源码")
# Sourcery的Swift Package命令行插件

# [什么是Sourcery?](https://www.polpiella.dev/sourcery-swift-package-command-plugin)

**[Sourcery](https://github.com/krzysztofzablocki/Sourcery)** 是当下最流行的Swift代码生成工具之一。其背后使用了 **[SwiftSyntax](https://github.com/apple/swift-syntax)**，旨在通过自动生成样板代码来节省开发人员的时间。Sourcery 通过扫描一组输入文件，然后借助[模板](https://github.com/krzysztofzablocki/Sourcery/blob/master/guides/Writing%20templates.md)的帮助，自动生成模板中定义的Swift代码。

# 示例

考虑一个为摄像机会话服务提供公共 API 的协议：

```swift
protocol Camera {
  func start()
  func stop()
  func capture(_ completion: @escaping (UIImage?) -> Void)
  func rotate()
}
```

当使用此新的**`Camera service`**进行单元测试时，我们希望确保**`AVCaptureSession`**没有被真的创建。我们仅仅希望确认 `camera service` 被测试系统（SUT）正确的调用了，而不是去测试`camera service`本身。

因此，创建一个协议的mock实现，使用空方法和一组变量来帮助我们进行单元测试，并断言（`asset`）进行了正确的调用是有意义的。这是软件开发中非常常见的一个场景，如果你曾维护过一个包含大量单元测试的大型代码库，这么做也可能有点乏味。

好吧~不用担心！Sourcery 会帮助你！⭐️ 它有一个叫做**[AutoMockable](https://github.com/krzysztofzablocki/Sourcery/blob/master/Templates/Templates/AutoMockable.stencil)**的模板，此模板会为任意输入文件中遵守 `AutoMockable` 协议的协议生成mock实现。

> 小侧栏：在本文中，我扩展地使用了术语`Mock`，因为它与 Sourcery 模板使用的术语一致。`Mock`是一个相当重载的术语，但通常，如果我要创建一个[双重测试](https://en.wikipedia.org/wiki/Test_double)，我会根据它的用途进一步指定类型的名称（可能是**`Spy`**、**`Fake`**、**`Stub`**等）。如果您有兴趣了解更多关于双重测试的信息，[**马丁·福勒**](https://twitter.com/martinfowler)（Martin Fowler）有一篇[非常好的文章](https://martinfowler.com/articles/mocksArentStubs.html)，可以解释这些差异。
> 

现在，我们让**`Camera`**遵守**`AutoMockable`**。该接口的唯一目的是充当 Sourcery 的目标，从中查找并生成代码。

```swift
import UIKit

// Protocol to be matched
protocol AutoMockable {}

public protocol Camera: AutoMockable {
  func start()
  func stop()
  func capture(_ completion: @escaping (UIImage?) -> Void)
  func rotate()
}
```

此时，可以在上面的输入文件上运行 Sourcery 命令，指定 **`AutoMockable`**模板的路径：

```bash
sourcery --sources Camera.swift --templates AutoMockable.stencil --output .
```

<aside>
💡 本文通过提供一个**`.sourcery.yml`**文件来配置Sourcery插件。如果提供了配置文件或Sourcery可以找到配置文件，则将忽略与其值冲突的所有命令行参数。如果您想了解有关配置文件的更多信息，Sourcery的[repo中有一节](https://github.com/krzysztofzablocki/Sourcery/blob/master/guides/Usage.md#configuration-file)介绍了该主题。

</aside>

命令执行完毕后，在输出目录下会生成一个`模板名`加**`.generated.swift`**为后缀的文件。在此例是**`./AutoMockable.generated.swift`**：

```swift
// Generated using Sourcery 1.8.2 — https://github.com/krzysztofzablocki/Sourcery
// DO NOT EDIT
// swiftlint:disable line_length
// swiftlint:disable variable_name

import Foundation
#if os(iOS) || os(tvOS) || os(watchOS)
import UIKit
#elseif os(OSX)
import AppKit
#endif

class CameraMock: Camera {

    //MARK: - start

    var startCallsCount = 0
    var startCalled: Bool {
        return startCallsCount > 0
    }
    var startClosure: (() -> Void)?

    func start() {
        startCallsCount += 1
        startClosure?()
    }

    //MARK: - stop

    var stopCallsCount = 0
    var stopCalled: Bool {
        return stopCallsCount > 0
    }
    var stopClosure: (() -> Void)?

    func stop() {
        stopCallsCount += 1
        stopClosure?()
    }

    //MARK: - capture

    var captureCallsCount = 0
    var captureCalled: Bool {
        return captureCallsCount > 0
    }
    var captureReceivedCompletion: ((UIImage?) -> Void)?
    var captureReceivedInvocations: [((UIImage?) -> Void)] = []
    var captureClosure: ((@escaping (UIImage?) -> Void) -> Void)?

    func capture(_ completion: @escaping (UIImage?) -> Void) {
        captureCallsCount += 1
        captureReceivedCompletion = completion
        captureReceivedInvocations.append(completion)
        captureClosure?(completion)
    }

    //MARK: - rotate

    var rotateCallsCount = 0
    var rotateCalled: Bool {
        return rotateCallsCount > 0
    }
    var rotateClosure: (() -> Void)?

    func rotate() {
        rotateCallsCount += 1
        rotateClosure?()
    }

}
```

上面的文件（`AutoMockable.generated.swift`）包含了你对mock的期望：使用空方法实现与目标协议的一致性，以及检查是否调用了这些协议方法的一组变量。最棒的是…Sourcery为您编写了这一切！🎉

# 怎么使用Swift package运行Sourcery？

至此你可能在想如何以及怎样在 Swift package 中运行 Sourcery。你可以手动执行，然后讲文件拖到包中，或者从包目录中的命令运行脚本。但是对于 Swift Package 有两种内置方式运行可执行文件：

1. 通过**命令行插件**，可根据用户输入任意运行
2. 通过**构建工具插件**，该插件作为构建过程的一部分运行。

在本文中，我将介绍Sourcery命令行插件，但我已经在编写第二部分，其中我将创建构建工具插件，这带来了许多有趣的挑战。

# 创建插件包

让我们首先创建一个空包，并去掉测试和其他我们现在不需要的文件夹。然后我们可以创建一个新的插件`Target`并添加 Sourcery 的二进制文件作为其依赖项。

为了让消费者使用这个插件，它还需要被定义为一个产品：

```swift
// swift-tools-version: 5.6
import PackageDescription

let package = Package(
    name: "SourceryPlugins",
    products: [
        .plugin(name: "SourceryCommand", targets: ["SourceryCommand"])
    ],
    targets: [
        // 1
        .plugin(
            name: "SourceryCommand",
            // 2
            capability: .command(
                intent: .custom(verb: "sourcery-code-generation", description: "Generates Swift files from a given set of inputs"),
                // 3
                permissions: [.writeToPackageDirectory(reason: "Need access to the package directory to generate files")]
            ),
            dependencies: ["Sourcery"]
        ),
        // 4
        .binaryTarget(
            name: "Sourcery",
            path: "Sourcery.artifactbundle"
        )
    ]
)
```

让我们一步一步地仔细查看上面的代码：

1. 定义插件目标。
2. 以**`custom`**为意图，定义了**`.command`**功能，因为没有任何默认功能（**`documentationGeneration`** 和 **`sourceCodeFormatting`**）与该命令的用例匹配。给**动词**一个合理的名称很重要，因为这是从命令行调用插件的方式。
3. 插件需要向用户请求写入包目录的权限，因为生成的文件将被转储到该目录。
4. 为插件定义了一个二进制目标文件。这将允许插件通过其上下文访问可执行文件。

<aside>
💡 我知道我并没有详细介绍上面的一些概念，但如果您想了解更多关于命令插件的信息，这里有一篇由**[Tibor Bödecs](https://twitter.com/tiborbodecs)**写的[超级棒的文章](https://theswiftdev.com/beginners-guide-to-swift-package-manager-command-plugins/)⭐。如果你还想了解更多关于 Swift Packages 中二级制的目标（文件），我同样有一篇[文章](https://www.polpiella.dev/binary-targets-in-modern-swift-packages)📦。

</aside>

# 编写插件

现在已经创建了包，是时候编写一些代码了！我们首先在**`Plugins/SourceryCommand`**下创建一个名为**`SourceryCommand.swift`**的文件，然后添加一个**`CommandPlugin`**协议的结构体，这将作为该插件的入口：

```swift
import PackagePlugin
import Foundation

@main
struct SourceryCommand: CommandPlugin {
    func performCommand(context: PluginContext, arguments: [String]) async throws {

    }
}
```

然后我们为命令编写实现：

```swift
func performCommand(context: PluginContext, arguments: [String]) async throws {
    // 1
    let configFilePath = context.package.directory.appending(subpath: ".sourcery.yml").string
    guard FileManager.default.fileExists(atPath: configFilePath) else {
        Diagnostics.error("Could not find config at: \(configFilePath)")
        return
    }

    //2
    let sourceryExecutable = try context.tool(named: "sourcery")
    let sourceryURL = URL(fileURLWithPath: sourceryExecutable.path.string)

    // 3
    let process = Process()
    process.executableURL = sourceryURL

    // 4
    process.arguments = [
        "--disableCache"
    ]

    // 5
    try process.run()
    process.waitUntilExit()

    // 6
    let gracefulExit = process.terminationReason == .exit && process.terminationStatus == 0
    if !gracefulExit {
        Diagnostics.error("🛑 The plugin execution failed")
    }
}
```

让我们仔细看看上面的代码：

1. 首先**`.sourcery.yml`**文件必须在包的根目录，否则将报错。这将使Sourcery神奇的工作，并使包可配置🪄。
2. 可执行文件路径的URL是从命令的上下文中检索的。
3. 创建一个进程，并将Sourcery的可执行文件的URL设置为其可执行文件路径。
4. 这一步有点麻烦。Sourcery 使用缓存来减少后续运行的代码生成时间，但问题是这些缓存是在包文件夹之外读取和写入的文件。插件的沙箱规则不允许这样做，因此**`--disableCache`**标志用于禁用此行为并允许命令运行。
5. 进程同步运行并等待。
6. 最后，检查进程终止状态和代码，以确保进程已正常退出。在任何其他情况下，通过**`Diagnostics`** API 向用户告知错误。

就这样！现在让我们使用它🚀

# 使用（插件）包

考虑一个用户正在使用插件，该插件将依赖项引入了他们的**`Package.swift`**文件：

```swift
// swift-tools-version: 5.6
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "SourceryPluginSample",
    products: [
        // Products define the executables and libraries a package produces, and make them visible to other packages.
        .library(
            name: "SourceryPluginSample",
            targets: ["SourceryPluginSample"]),
    ],
    dependencies: [
        .package(url: "https://github.com/pol-piella/sourcery-plugins.git", branch: "main")
    ],
    targets: [
        .target(
            name: "SourceryPluginSample",
            dependencies: [],
            exclude: ["SourceryTemplates"]
        ),
    ]
)
```

<aside>
💡 注意，与构建工具插件不同，命令插件不需要应用于任何目标，因为它们需要手动运行。

</aside>

用户只使用了上面的**`AutoMockable`**模板（可以在**`Sources/SourceryPluginSample/SourceryTemplates`**下找到），与本文前面显示的示例相匹配：

```swift
protocol AutoMockable {}

protocol Camera: AutoMockable {
    func start()
    func stop()
    func capture(_ completion: @escaping (UIImage?) -> Void)
    func rotate()
}
```

根据插件的要求，用户还提供了一个位于**`SourceryPluginSample`**目录下的**`.sourcery.yml`**配置文件：

```yaml
sources:
  - Sources/SourceryPluginSample
templates:
  - Sources/SourceryPluginSample/SourceryTemplates
output: Sources/SourceryPluginSample
```

# 运行命令

用户已经设置好了，但是他们现在如何运行包？🤔 有两种方法：

## 命令行

运行插件的一种方法是用命令行。可以通过从包目录中运行**`swift package plugin --list`**来检索特定包的可用插件列表。然后可以从列表中选择一个包，并通过运行**`swift package <command's verb>`**来执行，在这个特殊的例子中，运行：**`swift package sourcery-code-generation`**。

注意，由于此包需要特殊权限，因此**`--allow-writing-to-package-directory`**必须与命令一起使用：

[https://www.notion.so](https://www.notion.so)

此时，你可能会想，为什么我要费心编写一个插件，仍然必须从命令行运行，而我可以用一个简单的脚本在几行bash中完成相同的工作？好吧，让我们来看看Xcode 14中会出现什么，你会明白为什么我会提倡编写插件📦。

## Xcode

这是运行命令插件最令人兴奋的方式，但不幸的是，它仅在Xcode 14中可用。因此，如果您需要运行命令，但尚未使用Xcode 14，请参阅命令行部分。

如果你正好在使用Xcode 14，你可以通过在文件资源管理器中右键单击包，从列表中找到要执行的插件，然后单击它来执行包的任何命令。如下：

![]()

# 下一步

这是插件的初始实现。我将研究如何改进它，使它更加健壮。和往常一样，我非常致力于公开构建，并使我的文章中的所有内容都开源，这样任何人都可以提交问题或创建任何具有改进或修复的PRs。这没有什么不同😀, 这是**[公共仓库的链接](https://github.com/pol-piella/sourcery-plugins)**。

此外，如果您喜欢这篇文章，请关注即将到来的第二部分，其中我将制作一个 Sourcery 构建工具插件。我知道这听起来不多，但这不是一项容易的任务！🔨
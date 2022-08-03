## 前言

不久前，我正在工作中开发一项新服务，该服务由 Swift Package 组成，该 Package 公开了一个类似于`Decodable`协议，供我们应用程序的其余部分使用。事实上，该协议是从`Decodable`本身继承下来的，看起来像这样：

>  Fetchable.swit

```swift
protocol Fetchable: Decodable, Equatable {}
```

新的 package 将采用符合`Fetchable`的类型来尝试从远程或缓存的JSON数据块中解码它们。

由于这项服务对应用程序的正确运行至关重要，作为这项工作的一部分，我们希望确保始终存在故障安全（ fail-safe）。因此，我们让该应用程序附带了一个备用的JSON文件，如果远程和缓存的数据解码失败，将使用该文件，来保证程序的正常运行。

**无论如何**，我们需要符合`Fetchable`的新类型从备用数据中正确解码。然而，有一个问题，有时很难发现备用JSON文件或模型本身是否有任何错误，因为解码错误会在**运行时**发生，并且只有在访问某些屏幕/功能时才会发生。

为了让我们对我们要发送的代码更有**信心**，我们添加了一些**单元测试**，试图根据我们附带的备用JSON解码符合`Fetchable`协议的每个模型。这些将使我们在CI上有一个早期指示，表明备用数据或模型中存在错误，如果所有测试都通过，我们将确定，一旦我们发布新服务，它始终具有**故障安全功能**。

我们**手动**编写了这些测试，但我们很快就意识到这个解决方案是**不可扩展的**，因为随着越来越多的符合`Fetchable`协议的类型被添加，我们引入了大量的代码复制，并可能有人最终忘记为特定功能编写这些测试。

我们考虑过自动化该过程，但由于我们的代码库的性质，我们遇到了一些问题，代码库高度模块化，混合了Xcode项目和Swift Package。一些架构决策还意味着我们必须收集大量符号信息，才能获得生成测试的正确类型。

## 是什么让我再次关注到它？

在我忘记了这件事一段时间后，Xcode 14的公告允许在Xcode项目中使用 Swift Package 插件，以及一些架构更改使提取类型信息变得容易得多，这让我有动力再次开始研究这个问题。

> 请注意，Xcode项目的构建工具插件尚未按照发布说明在Xcode 14 Beta 2中提供，但将在Xcode 14的未来版本中提供。
>
> ![图片取自 Xcode Beta 2 版的发布说明](https://www.polpiella.dev/assets/posts/code-generation-using-swift-package-plugins/release-notes.png)
>
> 

在过去的几周里，我一直在研究如何使用软件包插件生成单元测试，在这篇文章中，我将解释我在向哪个方向尝试以及它涉及了什么。

## 实施细节

我开始了一项任务，即创建一个[构建工具插件](https://www.polpiella.dev/an-early-look-at-swift-extensible-build-tools)，与 Xcode 14 引入的[命令插件](https://github.com/apple/swift-evolution/blob/main/proposals/0332-swiftpm-command-plugins.md)不同，该插件可以任意运行并依赖用户输入，作为Swift软件包构建过程的一部分运行。

我知道我需要创建一个可执行文件，因为 Build Tool 插件依赖这些来执行操作。这个脚本将完全用 Swift 编写，因为这是我最熟悉的语言，并承担以下职责：

1. 扫描目标目录并提取所有`.swift`文件。目标将被递归扫描，以确保不会错过子目录。
1. 使用[sourcekit](https://github.com/apple/swift/tree/main/tools/SourceKit)，或者更具体地说，[SourceKitten](https://github.com/jpsim/SourceKitten)，扫描这些`.swift`文件并收集类型信息。这将允许提取符合`Fetchable`协议的所有类型，以便可以针对它们编写测试。
1. 获得这些类型后，生成一个带有`XCTestCase`的`.swift`文件，其中包含每种类型的单元测试。

## 让我们写一些代码吧

与所有 Swift Package 一样，最简单的入门方法是在命令行上运行`swift package init`。

这创建了两个目标，一个是包含`Fetchable`协议定义和符合该定义的类型的实现代码，另一个是应用插件为此类类型生成单元测试的测试目标。

> Package.swit

```swift
// swift-tools-version: 5.6
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "CodeGenSample",
    platforms: [.macOS(.v10_11)],
    products: [
        .library(
            name: "CodeGenSample",
            targets: ["CodeGenSample"]),
    ],
    dependencies: [
    ],
    targets: [
        .target(
            name: "CodeGenSample",
            dependencies: []
        ),
        .testTarget(
            name: "CodeGenSampleTests",
            dependencies: ["CodeGenSample"]
        )
     ]
)
```

### 编写可执行文件

如前所述，所有构建工具插件都需要可执行文件来执行所有必要的操作。

为了帮助开发此命令行，将使用几个依赖项。第一个是[SourceKitten](https://github.com/jpsim/SourceKitten)——特别是其SourceKitten框架库，这是一个Swift包装器，用于帮助使用Swift代码编写[sourcekit](https://github.com/apple/swift/tree/main/tools/SourceKit)请求，第二个是[快速参数解析器](https://github.com/apple/swift-argument-parser)，这是苹果提供的软件包，可以轻松创建命令行工具，并以更快、更安全的方式解析在执行过程中传递的命令行参数。

在创建`executableTarget`并赋予它两个依赖项后，`Package.swift`就是这个样子：

> Package.swift

```swift
// swift-tools-version: 5.6
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "CodeGenSample",
    platforms: [.macOS(.v10_11)],
    products: [
        .library(
            name: "CodeGenSample",
            targets: ["CodeGenSample"]),
    ],
    dependencies: [
        .package(url: "https://github.com/jpsim/SourceKitten.git", exact: "0.32.0"),
        .package(url: "https://github.com/apple/swift-argument-parser", from: "1.0.0")
    ],
    targets: [
        .target(
            name: "CodeGenSample",
            dependencies: []
        ),
        .testTarget(
            name: "CodeGenSampleTests",
            dependencies: ["CodeGenSample"]
        ),
        .executableTarget(
            name: "PluginExecutable",
            dependencies: [
                .product(name: "SourceKittenFramework", package: "SourceKitten"),
                .product(name: "ArgumentParser", package: "swift-argument-parser")
            ]
        )
     ]
)
```

可执行目标需要一个入口点，因此，在`PluginExecutable`目标的源目录下，必须创建一个名为`PluginExecutable.swift`的文件，其中所有可执行逻辑都需要创建。

> 请注意，这个文件可以随心所欲地命名，我倾向于以与我在`Package.swift`中创建的目标相同的方式命名它。

如下所示的脚本导入必要的依赖项，并创建可执行文件的入口点（必须用`@main`装饰），并声明在执行时传递的4个输入。

所有逻辑和方法调用都存在于`run`函数中，该函数是调用可执行文件时运行的方法。这是`ArgumentParser`语法的一部分，如果您想了解更多信息，[Andy Ibañez](https://www.andyibanez.com/posts/writing-commandline-tools-argumentparser-part1/)有[一篇](https://www.andyibanez.com/posts/writing-commandline-tools-argumentparser-part1/)关于该主题[的精彩文章](https://www.andyibanez.com/posts/writing-commandline-tools-argumentparser-part1/)，可能非常有帮助。

> PluginExecutable.swift

```swift
import SourceKittenFramework
import ArgumentParser
import Foundation

@main
struct PluginExecutable: ParsableCommand {
    @Argument(help: "The protocol name to match")
    var protocolName: String

    @Argument(help: "The module's name")
    var moduleName: String

    @Option(help: "Directory containing the swift files")
    var input: String

    @Option(help: "The path where the generated files will be created")
    var output: String

    func run() throws {
		// 1
        let files = try deepSearch(URL(fileURLWithPath: input, isDirectory: true))
        // 2
        setenv("IN_PROCESS_SOURCEKIT", "YES", 1)
        let structures = try files.map { try Structure(file: File(path: $0.path)!) }
        // 3
        var matchedTypes = [String]()
        structures.forEach { walkTree(dictionary: $0.dictionary, acc: &matchedTypes) }
        // 4
        try createOutputFile(withContent: matchedTypes)
    }

    // ...
}
```

现在让我们专注于上面的`run`方法，以了解当插件运行可执行文件时会发生什么：

1. 首先，扫描目标目录以找到其中的所有`.swift`文件。这是递归完成的，这样子目录就不会错过。此目录的路径作为参数传递给可执行文件。
2. 对于上次调用中找到的每个文件，通过[SourceKitten](https://github.com/jpsim/SourceKitten)发出`Structure`请求，以查找文件中Swift代码的类型信息。请注意，环境变量（`IN_PROCESS_SOURCEKIT`）也被设置为true。这需要确保选择源套件的进程中版本，以便它能够遵守插件的沙盒规则。

> Xcode附带两个版本的sourcekit可执行文件，一个版本解析进程中的文件，另一个使用XPC向解析进程外文件的守护进程发送请求。后者是mac上的默认版本，为了能够将sourcekit用作插件进程的一部分，必须选择进程中版本。[这最近在SourceKitten上作为环境变量实现](https://github.com/jpsim/SourceKitten/pull/728)，是运行引擎盖下使用sourcekit的其他可执行文件的关键，例如`SwiftLint`。

3. 浏览上次调用的所有响应，并扫描类型信息以提取符合`Fetchable`协议的任何类型。

4. 在传递给可执行文件的`output`参数指定的位置创建一个输出文件，其中包含每种类型的单元测试。

> 请注意，上面没有重点介绍每个调用的具体细节，但如果你对实现感兴趣，包含所有代码的repo现在已经在Github上公开了! 

### 创建该插件

与可执行文件一样，必须向`Package.swift`添加`.plugin`目标，并且必须创建包含插件实现的`.swift`文件（`Plugins/SourceKitPlugin/SourceKitPlugin.swift`）。

> Package.swift

```swift
// swift-tools-version: 5.6
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "CodeGenSample",
    platforms: [.macOS(.v10_11)],
    products: [
        .library(
            name: "CodeGenSample",
            targets: ["CodeGenSample"]),
    ],
    dependencies: [
        .package(url: "https://github.com/jpsim/SourceKitten.git", exact: "0.32.0"),
        .package(url: "https://github.com/apple/swift-argument-parser", from: "1.0.0")
    ],
    targets: [
        .target(
            name: "CodeGenSample",
            dependencies: []
        ),
        .testTarget(
            name: "CodeGenSampleTests",
            dependencies: [“CodeGenSample"],
plugins: [“SourceKitPlugin”],
        ),
        .executableTarget(
            name: "PluginExecutable",
            dependencies: [
                .product(name: "SourceKittenFramework", package: "SourceKitten"),
                .product(name: "ArgumentParser", package: "swift-argument-parser")
            ]
        ),
        .plugin(
            name: "SourceKitPlugin",
            capability: .buildTool(),
            dependencies: [.target(name: "PluginExecutable")]
        )
     ]
)
```

以下代码显示了插件的初始实现，其`struct`符合`BuildToolPlugin`的协议。这需要实现一个返回具有单个构建命令的数组的`createBuildCommands`方法。

> 此插件使用`buildCommand`而不是`preBuildCommand`，因为它需要作为构建过程的一部分运行，而不是在它之前运行，因此它有机会构建和使用它所依赖的可执行文件。在这种情况下，支持使用`buildCommand`的另一点是，它只会在输入文件更改时运行，而不是每次构建目标时运行。

此命令必须为要运行的可执行文件提供名称和路径，这可以在插件的上下文中找到：

> SourceKitPlugin.swift

```swift
import PackagePlugin

@main
struct SourceKitPlugin: BuildToolPlugin {
    func createBuildCommands(context: PluginContext, target: Target) async throws -> [Command] {
        return [
            .buildCommand(
                displayName: "Protocol Extraction!",
                executable: try context.tool(named: "PluginExecutable").path,
                arguments: [
                    "FindThis",
                    🤷,
                    "--input",
                    🤷,
                    "--output",
                    🤷
                ],
                environment: ["IN_PROCESS_SOURCEKIT": "YES"],
                outputFiles: [🤷]
            )
        ]
    }
}
```

如上面的代码所示，还有一些空白需要填充（🤷）：

1. 提供`outputPath`，用于生成单元测试文件。此文件可以在`pluginWorkDirectory`中生成，也可以在插件的上下文中找到。该目录提供读写权限且其中创建的任何文件都将是软件包构建过程的一部分。

2. 提供输入路径和模块名称。这是最棘手的部分，这些需要指向正在测试的目标的来源，而不是插件正在应用于的目标——单元测试。谢天谢地，插件的目标依赖项是可访问的，我们可以从该数组中获取我们感兴趣的依赖项。此依赖项将是内部的（`target`而不是`product`），它将为可执行文件提供其名称和目录。

> SourceKitPlugin.swift

```swift
import PackagePlugin

@main
struct SourceKitPlugin: BuildToolPlugin {
    func createBuildCommands(context: PluginContext, target: Target) async throws -> [Command] {
        let outputPath = context.pluginWorkDirectory.appending(“GeneratedTests.swift”)

        guard let dependencyTarget = target
            .dependencies
            .compactMap { dependency -> Target? in
                switch dependency {
                case .target(let target): return target
                default: return nil
                }
            }
            .filter { "\($0.name)Tests" == target.name  }
            .first else {
                Diagnostics.error("Could not get a dependency to scan!”)

                return []
        }

        return [
            .buildCommand(
                displayName: "Protocol Extraction!",
                executable: try context.tool(named: "PluginExecutable").path,
                arguments: [
                    "Fetchable",
	                 dependencyTarget.name,
                    "--input",
                    dependencyTarget.directory,
                    "--output",
                    outputPath
                ],
                environment: ["IN_PROCESS_SOURCEKIT": "YES"],
                outputFiles: [outputPath]
            )
        ]
    }
}
```

> 注意上述可选性处理方式。如果在测试目标的依赖项中找不到*合适的*目标，则使用[Diagnostics API](https://github.com/apple/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md#plugin-api)将错误转发回Xcode，并告诉它完成构建过程。

## 让我们看下结果

插件这就完成了！现在让我们在 Xcode 中运行它！为了测试这种方法，将包含以下内容的文件添加到`CodeGenSample`目标中：

> CodeGenSample.swift

```swift
import Foundation

protocol Fetchable: Decodable, Equatable {}

struct FeatureABlock: Fetchable {
    let featureA: FeatureA

    struct FeatureA: Fetchable {
        let url: URL
    }
}

enum Root {
    struct RootBlock: Fetchable {
        let url: URL
        let areAllFeaturesEnabled: Bool
    }
}
```

> 请注意，脚本将在结构中首次出现`Fetchable`协议时停止。这意味着任何嵌套的符合`Fetchable`协议的类型都将被测试，只是外部模型。

给定此输入并在主目标上运行测试，生成并运行`XCTestCase`，其中包含符合`Fetchable`协议的两种类型的测试。

> GeneratedTests.swift

```swift
import XCTest
@testable import CodeGenSample

class GeneratedTests: XCTestCase {
	func testFeatureABlock() {
		assertCanParseFromDefaults(FeatureABlock.self)
	}
	func testRoot_RootBlock() {
		assertCanParseFromDefaults(Root.RootBlock.self)
	}

    private func assertCanParseFromDefaults<T: Fetchable>(_ type: T.Type) {
        // Logic goes here...
    }
}
```

所有测试都通过了😅✅而且，尽管他们目前没有做很多事情，但可以扩展实现，以提供一些示例数据和一个`JSONDecoder`实例来对每个单元测试进行解析.

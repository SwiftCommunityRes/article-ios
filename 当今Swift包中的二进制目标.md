# 当今 Swift 包中的二进制目标

## 文章目录
1. 理解二进制在 Swift 中的演变
2. 命令行工具相关
3. 结论

在 **iOS** 和 **macOS** 开发中， Swift 包现在变得越来越重要。Apple 已经努力推动桥接那些缝隙，并且修复那些阻碍开发者的问题，例如阻碍开发者将他们的库和依赖由其他诸如 **[Carthage](https://github.com/Carthage/Carthage "Carthage")** 或 **[CocoaPods](https://github.com/CocoaPods/CocoaPods "CocoaPods")** 依赖管理工具迁移到 Swift 包依赖管理工具的问题，例如没有能力添加构建步骤的问题。这对任何依赖一些代码生成的库来说都是破坏者，比如，协议和 Swift 生成。

## 理解二进制在 Swift 中的演变
为了充分理解 Apple 的 Swift 团队在二进制目标和他们引入的一些新 API 方面采取的一些步骤，我们需要理解它们从何而来。在后续的部分中，我们将调研 Apple 架构的演变，以及为什么二进制目标的 API 在过去几年中逐渐形成的，特别是自 Apple 发布了自己的硅芯片之后。

### 胖二进制和 Frameworks 框架
如果你曾必须处理二进制依赖，或者你曾创建一个属于你自己的可执行文件，你将会对**胖二进制**这个术语感到熟悉。这些被扩展（或增大）的可执行文件，是包含了为多个不同架构原生构建的切片。这允许库的所有者分发一个运行在所有预期的目标架构上的单独的二进制。

当源码不能被暴露或当处理非常庞大的代码仓库时，预编译库成为可执行文件非常有意义，因为预编译源码以及以二进制文件分发他们，将节省构建程序在他们的应用上的构建时间。
 
**[Pods](https://cocoapods.org/ "Pods")** 是一个非常好的例子，当开发者发现他们自己没必要构建那些非常少改动的依赖。这是一个很共通的问题，它激发了诸如 **[cocoapods-binary](https://github.com/leavez/cocoapods-binary "cocoapods-binary")** 之类的项目，该项目预编译了 pod 依赖项以减少客户端的构建时间。

#### Frameworks 框架
嵌入静态二进制文件可能对应用程序来说已经足够了，但如果需要某些资源（如 assets 或头文件），则需要将这些资源与包含所有切片的**胖二进制文件**捆绑在一起，形成所谓的 **\`frameworks\`** 文件。

这就是诸如 **[Google Cast](https://developers.google.com/cast/docs/ios_sender#manual_setup "Google")** 之类的预编译库在过渡到使用 **\`xcframework\`** 进行分发之前所做的事情 —— 下一节将详细介绍这种过渡的原因。

到目前为止，一切都很好。 如果我们要为分发预编译一个库，那么胖二进制文件听起来很理想，对吧？并且，如果我们需要捆绑一些其他资源，我们可以只使用一个 **\`frameworks\`**。 一个二进制来统治他们所有！

#### XCFrameworks 框架
好吧，不完全是。胖二进制文件有一个大问题，那就是你不能有两个架构相同但命令/指令不同的切片。 这曾经很好，因为设备和模拟器的架构总是不同的，但是随着 Apple Silicon 计算机 (M1) 的推出，模拟器和设备共享相同的架构 (arm64)，但具有不同的加载器命令。 这与面向未来的二进制目标相结合，正是 Apple 引入 **[XCFrameworks](https://developer.apple.com/videos/play/wwdc2019/416/ "XCFrameworks")** 的原因。

> 你可以在 **[Bogo Giertler 撰写的这篇精彩文章](https://twitter.com/giertler)**中详细了解为 iOS 设备构建的 arm64 切片和为 M1 mac 的 iOS 模拟器构建的 arm64 切片之间的区别。

**[XCFrameworks](https://help.apple.com/xcode/mac/11.4/#/dev6f6ac218b "XCFrameworks")** 现在允许将多个二进制文件捆绑在一起，解决了 M1 Mac 引入的设备和模拟器冲突架构问题，因为我们现在可以为每个用例提供包含相关切片的二进制文件。 事实上，如果我们需要，我们可以走得更远，例如，在同一个 xcframework 中捆绑一个包含 iOS 目标的 **\`UIKit\`** 接口的二进制文件和一个包含 macOS 的 **\`AppKit\`** 接口的二进制文件，然后让 Xcode 基于期望的目标架构决定使用哪一个。

在 Swift 包中，那先能够以 **[binaryTarget](https://developer.apple.com/documentation/swift_packages/distributing_binary_frameworks_as_swift_packages "binaryTarget")** 被包含进项目的，能够在包中被引入任意其他目标。这相同的操作同样适用于 `frameworks`。

## 命令行工具相关
由于 Swift 5.6 版本中引入了用于 Swift 包管理器的 **[可扩展构建工具](https://github.com/apple/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md "Extensible Build Tools")** ，因此可以在构建过程中的不同时间执行命令。

这是 iOS 社区长期以来一直强烈要求的事情，例如格式化源代码、代码生成甚至收集公制代码库的指标。 Swift 5.6 中所有这些所谓的 **[插件](https://github.com/apple/swift-evolution/blob/main/proposals/0303-swiftpm-exte""nsible-build-tools.md#plugin-api "Plugins")** 最终都需要调用可执行文件来执行特定任务。 这是二进制文件再次在 Swift 包中参与的地方。

在大多数情况下，对于我们 iOS 开发人员来说，这些工具将来自同时支持 macOS 的不同架构切片 —— Apple Silicon 的 arm64 架构和 Intel Mac 的 x86_64 架构。开发者工具如， **[SwiftLint](https://github.com/realm/SwiftLint "SwiftLint")** 或 **[SwiftGen](https://github.com/SwiftGen/SwiftGen "SwiftGen")** 正是这种案例。 在这种情况下，可以使用包含可执行文件（本地或远程）的 **.zip** 文件的路径创建新的二进制目标。

> 注意可执行文件必须在.zip文件的根目录下，否则找不到。

### Artifact Bundles
到目前为止，命令行工具所采用的方法仅适用于 macOS 架构。但我们不能忘记，Linux 机器也支持 Swift 包。 这意味着如果要同时支持 M1 macs (**\`arm64\`**) 和 Linux **\`arm64\`** 机器，上面的胖二进制方法将不起作用 —— 请记住，二进制不能包含具有相同架构的多个切片。 在这个阶段可能有人会想，我们可以不只使用 **\`xcframeworks\`** 吗？ 不，因为它们在 Linux 操作系统上不受支持！

Apple 已经考虑到这一点，除了引入 **[可扩展构建工具](https://github.com/apple/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md "Extensible Build Tools")** 之外，**[Artifact Bundles](https://github.com/apple/swift-evolution/blob/main/proposals/0305-swiftpm-binary-target-improvements.md "Artifact Bundles")** 和对二进制目标的其他改进也作为 Swift 5.6 的一部分发布。

工件包(Artifact Bundles) 是包含*工件*的目录。 这些工件需要包含支持架构的所有不同二进制文件。 二进制文件和支持的架构的路径是使用清单文件 (**\`info.json\`**) 指定的，该文件位于 Artifact Bundle 目录的根目录中。 你可以将此清单文件视为一个地图或指南，以帮助 Swift 确定哪些可执行文件可用于哪种架构以及可以在哪里找到它们。

#### 以 SwiftLint 为例
**[SwiftLint](https://github.com/realm/SwiftLint "SwiftLint")** 在整个社区中被广泛用作 Swift 代码的静态代码分析工具。 由于很多人都非常渴望让这个插件在他们的 SwiftPM 项目中运行，我认为这将是一个很好的例子来展示我们如何将分发的可执行文件从他们的发布页面变成一个与 macOS 架构和 Linux arm64兼容的工件包。

让我们从下载两个可执行文件（**[macOS](https://github.com/realm/SwiftLint/releases/download/0.47.0/portable_swiftlint.zip macOS)** 和 **[Linux](https://github.com/realm/SwiftLint/releases/download/0.47.0/swiftlint_linux.zip "Linux")**）开始。

至此，bundle的结构就可以创建好了。 为此，创建一个名为 **\`swiftlint.artifactbundle\`** 的目录并在其根目录添加一个空的 **\`info.json\`**：

```shell
mkdir swiftlint.artifactbundle
touch swiftlint.artifactbundle/info.json
```

现在可以使用 **\`schemaVersion\`** 填充清单文件，这可能会在未来版本的工件包和具有两个变体的工件中发生变化，这将很快定义：

```json
{
    "schemaVersion": "1.0",
    "artifacts": {
        "swiftlint": {
            "version": "0.47.0", # The version of SwiftLint being used
            "type": "executable",
            "variants": [
            ]
        },
    }
}
```

需要做的最后一件事是将二进制文件添加到包中，然后将它们作为变体添加到 **\`info.json\`** 文件中。 让我们首先创建目录并将二进制文件放入其中（macOS 的一个在 **\`swiftlint-macos/swiftlint\`**，Linux 的一个在 **\`swiftlint-linux/swiftlint\`**）。

添加这些之后，可以在清单文件中变量：

```json
{
    "schemaVersion": "1.0",
    "artifacts": {
        "swiftlint": {
            "version": "0.47.0", # The version of SwiftLint being used
            "type": "executable",
            "variants": [
			          {
                    "path": "swiftlint-macos/swiftlint",
                    "supportedTriples": ["x86_64-apple-macosx", "arm64-apple-macosx"]
                },
	              {
                    "path": "swiftlint-linux/swiftlint",
                    "supportedTriples": ["x86_64-unknown-linux-gnu"]
                },
            ]
        },
    }
}
```

为此，需要为每个变量指定二进制文件的相对路径（从工件包目录的根目录）和支持的三元组。 如果您不熟悉 **[目标三元组](https://clang.llvm.org/docs/CrossCompilation.html#target-triples "target-triples")**，它们是一种选择构建二进制文件的架构的方法。 请注意，这不是**主机**（构建可执行文件的机器）的体系结构，而是**目标**机器（应该运行所述可执行文件的机器）。

这些三元组具有以下格式： **\`----\`** 并非所有字段都是必需的，如果其中一个字段未知并且要使用默认值，则可以省略或替换为 **\`unknown\`** 关键字。

可执行文件的架构切片可以通过运行 **\`file\`** 找到，这将打印捆绑的任何切片的供应商、系统和架构。 在这种情况下，为这两个命令运行它会显示：

**swiftlint-macos/swiftlint**

```
swiftlint: Mach-O universal binary with 2 architectures: [x86_64:Mach-O 64-bit executable x86_64] [arm64]
swiftlint (for architecture x86_64):	Mach-O 64-bit executable x86_64
swiftlint (for architecture arm64):	Mach-O 64-bit executable arm64
```

**swiftlint-linux/swiftlint**

```
-> file swiftlint
swiftlint: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, with debug_info, not stripped
```
这带来了上面显示的 macOS 支持的两个三元组（**\`x86_64-apple-macosx‌\`**、**\`arm64-apple-macosx\`**）和 Linux 支持的一个三元组（**\`x86_64-unknown-linux-gnu\`**）。

与 **\`XCFrameworks\`** 类似，工件包也可以通过使用 **[binaryTarget](https://developer.apple.com/documentation/swift_packages/distributing_binary_frameworks_as_swift_packages)** 包含在 Swift 包中。

## 结论
简而言之，我们可以总结 2022 年如何在 Swift 包中使用二进制文件的最佳实践，如下所示：

1. 如果你需要为你的 iOS/macOS 项目添加预编译库或可执行文件，您应该使用 **\`XCFramework\`**，并为每个用例（iOS 设备、macOS 设备和 iOS 模拟器）包含单独的二进制文件。
2. 如果你需要创建一个插件并运行一个可执行文件，你应该将其嵌入为一个工件包，其中包含适用于不同支持架构的二进制文件。
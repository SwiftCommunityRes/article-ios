## 前言

我最近了解到，Swift 6 的一些重大变更（如完整的数据隔离和数据竞争安全检查）将成为 Swift 6 语言模式的一部分，该模式将在 Swift 6 编译器中作为可选功能启用。

这意味着，当你更新 Xcode 版本或使用 Swift 6 编译器的 Swift 工具链时，除非你明确启用 Swift 6 语言模式，否则你的代码将使用 Swift 5 语言模式进行编译。

在本文中，我将向你展示如何下载和安装 Swift 6 工具链的开发快照，并在构建 Swift 包时启用 Swift 6 语言模式。

## 下载 Swift 6 工具链

使用 Swift 6 编译器和语言模式构建代码的第一步是下载 Swift 6 开发工具链。

Apple 在 swift.org 网站上提供了从 release/6.0 分支构建的 Swift 编译器版本，适用于多个平台，你可以下载并安装到系统中。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7790360ae094574ae6d944e69d3097d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1758\&h=1488\&s=480181\&e=png\&b=fefefe)

你可以手动执行此操作，但我建议使用像 Swiftenv（用于 macOS）或 Swiftly（用于 Linux）这样的工具来管理你的 Swift 工具链，就像本文中所示的那样。

### Swiftenv - macOS

Swiftenv 是一个受 pyenv 启发的 Swift 版本管理器，它允许你轻松安装和管理多个版本的 Swift。

使用 Swiftenv，安装最新的 Swift 6 开发快照只需运行以下命令：

```bash
# 安装最新的 Swift 6 开发工具链
swiftenv install 6.0-DEVELOPMENT-SNAPSHOT-2024-04-30-a

# 进入你的 Swift 包目录
cd your-swift-package

# 将 Swift 6 工具链设置为此目录的默认工具链
swiftenv local 6.0-DEVELOPMENT-SNAPSHOT-2024-04-30-a
```

### Swiftly - Linux

如果你在 Linux 机器上构建代码，可以使用 Swift Server Workgroup 的 Swiftly 命令行工具来安装和管理 Swift 工具链，运行以下命令：

```bash
# 安装最新的 Swift 6 开发工具链
swiftly install 6.0-DEVELOPMENT-SNAPSHOT-2024-04-30-a

# 将 Swift 6 工具链设置为活动工具链
swiftly use 6.0-DEVELOPMENT-SNAPSHOT-2024-04-30-a
```

## 在 SPM 中启用语言模式

让我们考虑一个 Swift 包目标，其代码在使用 Swift 6 编译器和 Swift 6 语言模式编译时会产生错误：

```swift
class NonIsolated {
    func callee() async {}
}

actor Isolated {
    let isolated = NonIsolated()
    
    func callee() async {
        await isolated.callee()
    }
}
```

让我们使用我们之前下载的 Swift 6 工具链并启用 StrictConcurrency 实验功能进行构建：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05b9001a6b0446f58278184d10ae80c8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=600\&h=407\&s=2089769\&e=gif\&f=138\&b=262835)

如你所见，构建结果是警告而不是错误。这是因为默认情况下，Swift 6 编译器使用的是 Swift 5 语言模式，而 Swift 6 语言模式是可选的。

有两种方法可以启用 Swift 6 语言模式：直接从命令行通过将 `-swift-version` 标志传递给 swift 编译器，或者在包清单文件中指定它。

### 命令行

要启用 Swift 6 语言模式编译代码，可以使用以下命令：

```bash
swift build -Xswiftc -swift-version -Xswiftc 6
```

### 包清单文件

你可以通过更新 tools-version 到 6.0 并在包清单文件中添加 `swiftLanguageVersions` 键来为你的 Swift 包启用 Swift 6 语言模式：

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "Swift6Examples",
    platforms: [.macOS(.v10_15), .iOS(.v13)],
    products: [
        .library(
            name: "Swift6Examples",
            targets: ["Swift6Examples"]
        )
    ],
    targets: [
        .target(name: "Swift6Examples")
    ],
    swiftLanguageVersions: [.version("6")]
)
```

## 输出

正如你所见，当启用了 Swift 6 语言模式后，编译器报告了与数据隔离相关的错误。这些错误表明我们在代码中存在需要修复的并发问题。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52dfa20affad46808de2281bcc02d5c3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=600\&h=407\&s=1666523\&e=gif\&f=75\&b=262835)

### 结论

Swift 6 带来了许多重要的新特性，如数据隔离和数据竞争安全检查，这些特性有助于编写更安全、更高效的代码。然而，这些新特性并不会自动启用，需要通过 Swift 6 语言模式显式开启。通过下载和安装 Swift 6 工具链，并在命令行或包清单文件中启用 Swift 6 语言模式，我们可以提前体验和适应这些变化。尽管新特性带来了一些学习和调整成本，但它们最终会使我们的代码更加健壮。

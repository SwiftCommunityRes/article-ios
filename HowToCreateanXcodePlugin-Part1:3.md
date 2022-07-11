# 如何创建一个 Xcode 插件：1/3

在这三部分系列教程的第一篇中，你将会与探索 App 内部结构一起，开始学习有关开发 Xcode 插件和一些 LLDB 提示。

![](https://koenig-media.raywenderlich.com/uploads/2015/05/rayroll-square.png "Customize your Xcode just the way you like it!")

**更新提示** 该教程已经使用 **Xcode 6.3.2** 进行测试，如果你正使用其他版本的 Xcode 开发，你对本教程的的体验可能不那么完美相匹配。

“一种尺寸适配所有”的范例是 Apple 为他的产品延伸的特性，很可能是一个难以下咽的药片。尽管 Apple 已将其工作流程强制内置提供给 iOS/OS X 开发人员，但仍然可以通过插件将 Xcode 曲线实现开发者的意愿。

Apple 官方没有任何关于如何创建 Xcode 插件的文档，但是开发社区已经投入了相当大的工作量在一些非常有用的工具中，从而帮助开发人员。

从 [自动提示图片](https://github.com/ksuther/KSImageNamed-Xcode "autocompletion for images")，到清除 [DerivedData](https://github.com/kattrali/deriveddata-exterminator "deriveddata exterminator")，再到 [vim 编辑器](https://github.com/XVimProject/XVim "vim editor")，Xcode 插件社区已经突破了最初被认为可能的边界。

在这个漫长的三部分教程中，你讲创建一个 Xcode 插件，去捉弄你的同事，最佳整蛊对象非 [Ray](https://www.raywenderlich.com/u/rwenderlich) 他自己莫属。并且尽管在自然情况下 Xcode 插件是轻松愉快的，但你仍要学习很多关于追踪 Xcode，怎样找到你想要修改的元素，以及怎样调换你自己的函数。

当探索私有接口和用 [method swizzling](http://nshipster.com/method-swizzling/ "method swizzling") 注入代码时，你将会用到 [x86 汇编知识](https://www.mikeash.com/pyblog/friday-qa-2011-12-16-disassembling-the-assembly-part-1.html "assembly")审查没有文档说明的框架，[代码定位能力](http://www.raywenderlich.com/79600/navigating-a-new-codebase)，以及 [LLDB 技能](http://www.raywenderlich.com/?s=lldb "lldb skills")。由于要涵盖的内容很多，因此本教程进行非常快速。在开始前，请确保你熟悉 iOS 或 macOS 开发。

使用 Swift 开发 Xcode 插件，使一个已经非常棘手的话题边等更加复杂，并且 Swift 的调试工具只是还没有达到 Objective-C 的水平。对于现在而言，这就意味着对于插件开发最佳的选择是 Objective-C。

## 开始
为了庆祝**整蛊你的同事节**，你的 Xcode 插件将 **Rayroll** 你的受害者。等等，Rayrolling 是什么？ 这是 Rickrolling 的版权和免版税版本，你可以在其中使用与预期不同的内容诱饵和转换受害者。当你完成这个系列，你的插件将修改 Xcode，从而它将：
1. 在 Xcode 的警告中，替换 Ray 的脸（比如，构建成功/构建失败）。

![](https://koenig-media.raywenderlich.com/uploads/2015/04/Xcode_Swizzle_DispalAlert.png)
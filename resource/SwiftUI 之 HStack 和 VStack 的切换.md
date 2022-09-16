# SwiftUI 之 HStack 和 VStack 的切换

## 前言

`SwiftUI` 的各种堆栈是许多框架中最基本的布局工具，能够让我们定义组视图，这些组视图可以按照水平、垂直或覆盖视图对齐。

当涉及到水平和垂直的变体时( `HStack` 和 `VStack` )，我们需要在这两者之间动态的切换。举个例子，假如我们正在构建一个 `app` 其中包含 `LoginActionsView` ，一个让用户登录时在列表中选择操作的类：

```swift
struct LoginActionsView: View {
    ...

    var body: some View {
        VStack {
            Button("Login") { ... }
            Button("Reset password") { ... }
            Button("Create account") { ... }
        }
        .buttonStyle(ActionButtonStyle())
    }
}

struct ActionButtonStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .fixedSize()
            .frame(maxWidth: .infinity)
            .padding()
            .foregroundColor(.white)
            .background(Color.blue)
            .cornerRadius(10)
    }
}

```

>以上代码中，我们用到了 `fixedSize` 防止按钮文本被截断，这仅是在我们确信给定的内容视图不会比视图本身更大的情况。想了解更多信息，可以查看我的文章 -  [SwiftUI 布局系统第三章](https://www.swiftbysundell.com/articles/swiftui-layout-system-guide-part-3/#fixed-dimensions)

目前，我们的按钮是垂直排列的，并且填满了水平线上的可用空间（你可以用以上示例代码预览按钮的样子），虽然这在竖向的 iPhone 上看起来很好，但假设我们现在想要在横向模式下让 `UI` 横向排列。

## GeometryReader 能实现吗？

一种方式是用 `GeometryReader` 测量当前可用空间，并根据宽度是否大于其高度，可以选择使用 `HStack` 或  `VStack` 来渲染内容。

虽然可以在 `LoginActionsView` 中放入该逻辑，但我们希望以后能复用代码，因此需要重新创建一个专门的视图，作为一个独立的组件来实现动态堆栈的切换逻辑。

为了使代码可用性更高，我们不会硬编码让两个堆栈变体使用对齐或间距什么的。相反，让我们像 `SwiftUI` 一样，对这些属性参数化，同时设定框架所使用的默认值 — 就像这样：

```swift
struct DynamicStack<Content: View>: View {
    var horizontalAlignment = HorizontalAlignment.center
var verticalAlignment = VerticalAlignment.center
var spacing: CGFloat?
    @ViewBuilder var content: () -> Content

    var body: some View {
        GeometryReader { proxy in
            Group {
                if proxy.size.width > proxy.size.height {
                    HStack(
                        alignment: verticalAlignment,
                        spacing: spacing,
                        content: content
                    )
                } else {
                    VStack(
                        alignment: horizontalAlignment,
                        spacing: spacing,
                        content: content
                    )
                }
            }
        }
    }
}
```

由于我们使新的 `DynamicStack` 使用了与 `HStack`  和 `VStack` 相同的 `API` ，现在可以在 `LoginActionsView` 中直接将以前的 `VStack` 换成新的自定义的实例：

```swift
struct LoginActionsView: View {
    ...

    var body: some View {
        DynamicStack {
            Button("Login") { ... }
            Button("Reset password") { ... }
            Button("Create account") { ... }
        }
        .buttonStyle(ActionButtonStyle())
    }
}
```

优秀！然而，就像上面的代码展示的那样，使用 `GeometeryReader` 来展示动态切换有一个相当明显的缺点，在几何图形阅读器中总是会填充水平和垂直方向的所有可用空间（以便测量实际空间）。在我们的例子中，`LoginActionsView` 不再只是水平方向的排列，它现在也能移动到屏幕的顶部。

虽然我们也有很多方法能解决这些问题（例如使用类似在[这篇 Q&A ](https://swiftbysundell.com/questions/syncing-the-width-or-height-of-two-swiftui-views/)中用来使多个视图具有相同宽度和高度的技术），但真正的问题是当我们要动态的确定方向时，测量可用空间是否是一个好的方法。

## 一个使用尺寸类的例子

相反，让我们使用 `Apple` 的尺寸类系统来决定 `DynamicStack` 应该在底层使用 `HStack` 还是 `VStack` 。这样做的好处不仅仅是在引入 `GeometeryReader` 之前保留同样紧凑的布局，并且会使 `DynamicStack` 在开始的时候以一种和系统组件类似的方式在所有设备和方向上构建。

为了观察当前水平方向的尺寸，我们需要用到 [SwiftUI 环境系统](https://swiftbysundell.com/articles/swiftui-state-management-guide/#observing-and-modifying-the-environment)  —  通过在 `DynamicStack` 中声明 `@Environment` - 标记属性（带有  `horizontalSizeClass` [关键路径](https://swiftbysundell.com/articles/the-power-of-key-paths-in-swift/)），将会使我们在视图内容中切换到当前 `sizeClass` 的值：

```swift
struct DynamicStack<Content: View>: View {
    ...
    @Environment(\.horizontalSizeClass) private var sizeClass

    var body: some View {
        switch sizeClass {
        case .regular:
            hStack
        case .compact, .none:
            vStack
        @unknown default:
            vStack
        }
    }
}

private extension DynamicStack {
    var hStack: some View {
        HStack(
            alignment: verticalAlignment,
            spacing: spacing,
            content: content
        )
    }

    var vStack: some View {
        VStack(
            alignment: horizontalAlignment,
            spacing: spacing,
            content: content
        )
    }
}
```

经过以上操作，`LoginActionsView` 将可以在常规的尺寸渲染时动态切换成水平布局（例如在大尺寸的 `iPhone` 使用横屏，或者全屏 `iPad` 上的任一方向），而其它所有尺寸的配置使用垂直布局。所有这些仍然使用紧凑垂直布局，它使用的空间不超过渲染其内容所需的空间。

## 使用布局协议

虽然我们最后已经用了非常棒的解决方案，可以在所有支持 `SwiftUI `  的 `iOS` 版本中使用，但也让我们来探索一下在 `iOS 16` 中引入的一些新的布局工具（在写这篇文章时，它作为 `Xcode 14` 的一部分仍在测试阶段） 

其中一个工具是新的 `Layout` 协议，它既能让我们创建完整的自定义布局，直接集成到 `SwiftUI `  的布局系统中，同时也提供给我们一种更丝滑更动画的方式在各种布局之间动态切换 。

这都是因为事实证明 `Layout` 不仅仅是我们第三方开发者的 `API` ，`Apple` 也让 `SwiftUI ` 自己的布局容器使用这个新协议 。所以，与其直接使用 `HStack ` 和 `VStack ` 作为容器视图，不如将它们作为符合 `Layout` 的实例，使用 `AnyLayout` 类型进行包装 — 就像这样：

```swift
private extension DynamicStack {
    var currentLayout: AnyLayout {
        switch sizeClass {
        case .regular, .none:
            return horizontalLayout
        case .compact:
            return verticalLayout
        @unknown default:
            return verticalLayout
        }
    }

    var horizontalLayout: AnyLayout {
        AnyLayout(HStack(
            alignment: verticalAlignment,
            spacing: spacing
        ))
    }

    var verticalLayout: AnyLayout {
        AnyLayout(VStack(
            alignment: horizontalAlignment,
            spacing: spacing
        ))
    }
}
```

以上的操作是可行的，因为当 `HStack` 和 `VStack` 的内容类型是 `EmptyView` 时，它们都符合新的 `Layout` 协议（当内容为空时就是这种情况），让我们来看一下`SwiftUI `  的 公共接口

```swift
struct DynamicStack<Content: View>: View {
    ...

    var body: some View {
        currentLayout(content)
    }
}
```

>   注意：由于回归， `Xcode 14 beta 3` 中省略了以上条件的一致性，根据 `SwiftUI ` 团队的 [Matt Ricketson 的说法](https://twitter.com/ricketson_/status/1544784314453282817)，可以直接使用底层的 `_HStackLayout` 和 `_VStackLayout` 类型作为临时的解决方法。并希望能在未来测试版本中修复。    

现在我们能通过使用新的 `currentLayout` 解决使用什么布局，现在我们来更新 `body` 的实现，简单调用从该属性返回的 `AnyLayout` ，就像函数一样 — 像这样：

```swift
struct DynamicStack<Content: View>: View {
    ...

    var body: some View {
        currentLayout(content)
    }
}
```

>   我们之所以能像一个函数一样调用布局方法（尽管它实际上是一个结构）是因为 `Layout` 协议使用了 `Swift` [”像函数一样调用“ 的特性](https://swiftbysundell.com/articles/exploring-swift-5-2s-new-functional-features/#calling-types-as-functions)

那么我们之前的方案和上面基于布局的方案有什么区别呢？关键的区别在于（除了后者需要 `iOS 16` ）切换布局可以保留正在渲染的底层视图的标识，而在 `HStack` 和 `VStack` 之间切换就不会这样。这样做会令动画更流畅，例如在切换设备方向时，我们也有可能在执行此类更改时获得小幅的性能提升（因为 `SwiftUI` 总是在其视图层次结构为静态时尽可能表现最佳）

## 选择合适的视图

但我们还没有结束，因为 `iOS 16` 也给了我们其他有趣的新的布局工具，它有可能也能用于实现 `DynamicStack`  — 一种全新的视图类型，名字叫做 `ViewThatFits` 。就像字面意思一样，这种新的容器将会在我们初始化时传递的候选列表中，基于当前上下文挑选出最优视图。

在我们的例子中，这意味着我们能同时把 `HStack` 和 `VStack` 传递给它，并且代表我们在它们中间自动切换。

```swift
struct DynamicStack<Content: View>: View {
    ...

    var body: some View {
        ViewThatFits {
            HStack(
                alignment: verticalAlignment,
                spacing: spacing,
                content: content
            )

            VStack(
                alignment: horizontalAlignment,
                spacing: spacing,
                content: content
            )
        }
    }
}
```

注意：在这种情况下，我们首先放置 `HStack` 是很重要的，因为 `VStack` 可能总是合适的，即使在我们希望布局是横向的情况下（例如 `iPad` 的全屏模式）。同样重要的是要指出，上述基于 `ViewThatFits` 的技术将会始终尝试 `HStack` ，即使在用紧凑尺寸渲染布局时也是如此，只有在 `HStack` 不适合时才会选择基于`VStack` 的布局。  

## 结语

以上就是通过四种不同的方式实现 `DynamicStack` 视图，它可以根据当前内容在 `HStack` 和 `VStack` 之间动态切换。

>译自：https://www.swiftbysundell.com/articles/switching-between-swiftui-hstack-vstack/
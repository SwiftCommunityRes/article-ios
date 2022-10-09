# SwiftUI 锁屏小组件

> [原文链接](https://swiftwithmajid.com/2022/08/30/lock-screen-widgets-in-swiftui/)
> 

iOS呼声最高的功能之一是可定制的锁屏。终于，在最新发布的iOS 16得以实现。我们可以用可浏览的小组件填充锁屏。实现锁屏小组件很简单，因为它的API与主屏小组件共享相同的代码。本周我们将学习如何为我们的pp实现锁屏小组件。

> 查看我专门发布的[“构建SwiftUI小组件”](https://swiftwithmajid.com/2020/09/09/building-widgets-in-swiftui/)，了解有关主屏小组件的更多信息。
> 

让我们从你可能早就有的App主屏小组件代码开始。

```swift
struct WidgetView: View {
    let entry: Entry
    
    @Environment(\.widgetFamily) private var family
    
    var body: some View {
        switch family {
        case .systemSmall:
            SmallWidgetView(entry: entry)
        case .systemMedium:
            MediumWidgetView(entry: entry)
        case .systemLarge, .systemExtraLarge:
            LargeWidgetView(entry: entry)
        default:
            EmptyView()
        }
    }
}
```

在上面的示例中，我们有一个定义小组件的典型视图。我们使用`Environment`来知道`widget family`并显示适当的大小。我们需要做的就是删除默认语句，并实现定义锁屏小组件的所有新用例。

```swift
struct WidgetView: View {
    let entry: Entry
    
    @Environment(\.widgetFamily) private var family
    
    var body: some View {
        switch family {
        case .systemSmall:
            SmallWidgetView(entry: entry)
        case .systemMedium:
            MediumWidgetView(entry: entry)
        case .systemLarge, .systemExtraLarge:
            LargeWidgetView(entry: entry)
        case .accessoryCircular:
            Gauge(value: entry.goal) {
                Text(verbatim: entry.label)
            }
            .gaugeStyle(.accessoryCircularCapacity)
        case .accessoryInline:
            Text(verbatim: entry.label)
        case .accessoryRectangular:
            VStack(alignment: .leading) {
                Text(verbatim: entry.label)
                Text(entry.date, format: .dateTime)
            }
        default:
            EmptyView()
        }
    }
}
```

最好记住，系统对锁屏和主屏小组件使用不同的渲染模式。系统为我们提供了三种不同的渲染模式。

1. 主屏小组件和 Watch OS支持颜色的[全色模式](https://www.interaction-design.org/literature/topics/color-modes)。是的，从 watchOS 9 开始，你还可以用 WidgetKit 去实现 watchOS 的复杂性。
2. 振动模式(vibrant mode)是指系统将文本、图像和仪表还原为单色，并为锁屏背景正确着色。
3. 重音模式(accented mode)仅在 watchOS 上使用，系统将小部件分为两组，默认和重音。 系统使用用户在表盘设置中选择的色调颜色为小部件的重音部分着色。

渲染模式可通过SwiftUI `Environment`变量使用，因此你可以始终检查哪个渲染模式处于活动状态，并将其反映在设计中。例如，可以使用具有不同渲染模式的不同图片。

```swift
struct WidgetView: View {
    let entry: Entry
    
    @Environment(\.widgetRenderingMode) private var renderingMode
    
    var body: some View {
        switch renderingMode {
        case .accented:
            AccentedWidgetView(entry: entry)
        case .fullColor:
            FullColorWidgetView(entry: entry)
        case .vibrant:
            VibrantWidgetView(entry: entry)
        default:
            EmptyView()
        }
    }
}
```

如上所示，我们使用`widgetRenderingMode`环境值来获得实际的渲染模式，并表现出不同的行为。像之前讲到的，在重音模式(accented mode)下，系统将小部件分为两部分，并对它们进行特殊着色。可以使用`widgetAccentable`视图修改器标记视图层次的一部分。在这种情况下，系统将知道哪些视图应用着色颜色。

```swift
struct AccentedWidgetView: View {
    let entry: Entry
    var body: some View {
        HStack {
            Image(systemName: "moon")
                .widgetAccentable()
            Text(verbatim: entry.label)
        }
    }
}
```

最后，我们需要为小组件配置支持类型。

```swift
@main
struct MyAppWidget: Widget {
    let kind: String = "Widget"
    
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: Provider()) { entry in
            WidgetView(entry: entry)
        }
        .configurationDisplayName("My app widget")
        .supportedFamilies(
            [
                .systemSmall,
                .systemMedium,
                .systemLarge,
                .systemExtraLarge,
                .accessoryInline,
                .accessoryCircular,
                .accessoryRectangular
            ]
        )
    }
}
```

如果你仍然支持iOS 15，可以检查新锁屏小组件的可用性。

```swift
@main
struct MyAppWidget: Widget {
    let kind: String = "Widget"
    
    private var supportedFamilies: [WidgetFamily] {
        if #available(iOSApplicationExtension 16.0, *) {
            return [
                .systemSmall,
                .systemMedium,
                .systemLarge,
                .accessoryCircular,
                .accessoryRectangular,
                .accessoryInline
            ]
        } else {
            return [
                .systemSmall,
                .systemMedium,
                .systemLarge
            ]
        }
    }
    
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: Provider()) { entry in
            WidgetView(entry: entry)
        }
        .configurationDisplayName("My app widget")
        .supportedFamilies(supportedFamilies)
    }
}
```
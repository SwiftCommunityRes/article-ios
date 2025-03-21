## 前言

SwiftUI 为我们提供了一系列丰富的视图修饰符，用于操作视图的可访问性树。我已经介绍了其中许多，你可以在博客中找到它们。本文我们将讨论 `accessibilityChildren` 视图修饰符以及我们如何从中受益。

`accessibilityChildren` 视图修饰符允许我们为视图创建一个可访问性容器，并使用 `ViewBuilder` 闭包提供的视图元素进行填充。

## 示例

让我们来看一个简单的示例。

```swift
struct BarChartShape: Shape {
    let dataPoints: [DataPoint]
    
    func path(in rect: CGRect) -> Path {
        Path { p in
            let spacing: CGFloat = 10  // 间距为 10 像素
            let totalSpacing = CGFloat(dataPoints.count - 1) * spacing
            let availableWidth = rect.size.width - totalSpacing
            let width: CGFloat = availableWidth / CGFloat(dataPoints.count)
            var x: CGFloat = 0
            
            for point in dataPoints {
                let pointRect = CGRect(
                    x: x,
                    y: rect.size.height - point.value,
                    width: width,
                    height: point.value
                )
                let pointPath = RoundedRectangle(cornerRadius: 8).path(in: pointRect)
                p.addPath(pointPath)
                x += width + spacing  // 添加间距
            }
        }
    }
}
```

如你在上面的示例中所见，我们有一个绘制数据点的形状类型。我们无法为每个数据点提供可访问性值，因为在描边或填充形状后，该形状将成为一个单一视图。

## accessibilityChildren 使用

不过，SwiftUI 为这种情况专门提供了 `accessibilityChildren` 视图修饰符。

```swift
struct ContentView: View {
    @State private var dataPoints: [DataPoint] = [
        .init(value: 200),
        .init(value: 300),
        .init(value: 50),
        .init(value: 600),
        .init(value: 500)
    ]
    
    var body: some View {
        BarChartShape(dataPoints: dataPoints)
            .fill(.red)
            .accessibilityLabel("Chart")
            .accessibilityChildren {
                HStack(alignment: .bottom, spacing: 0) {
                    ForEach(dataPoints) { point in
                        RoundedRectangle(cornerRadius: 8)
                            .accessibilityValue(Text(point.value.formatted()))
                    }
                }
            }
    }
}
```

通过应用 `accessibilityChildren` 视图修饰符，我们创建了一个可访问性容器，并使用 ViewBuilder 闭包中提供的视图元素进行填充。SwiftUI 不会渲染我们通过 ViewBuilder 闭包传递的视图，它仅用于填充可访问性树的子元素。

`accessibilityChildren` 和 `accessibilityRepresentation` 视图修饰符之间的主要区别在于前者不会影响视图本身。它仅为子元素创建一个可访问性容器，而 `accessibilityRepresentation` 视图修饰符会完全替换当前视图的可访问性树。

## 完整代码

首先，你需要定义 `DataPoint` 结构体，然后可以在 `ContentView` 中初始化 `dataPoints` 数组。以下是完善后的代码：

```swift
import SwiftUI

struct DataPoint: Identifiable {
    var id = UUID()
    var value: CGFloat
}

struct BarChartShape: Shape {
    let dataPoints: [DataPoint]
    
    func path(in rect: CGRect) -> Path {
        Path { p in
            let width: CGFloat = rect.size.width / CGFloat(dataPoints.count)
            var x: CGFloat = 0
            
            for point in dataPoints {
                let pointRect = CGRect(
                    x: x,
                    y: rect.size.height - point.value,
                    width: width,
                    height: point.value
                )
                let pointPath = RoundedRectangle(cornerRadius: 8).path(in: pointRect)
                p.addPath(pointPath)
                x += width
            }
        }
    }
}

struct ContentView: View {
    @State private var dataPoints: [DataPoint] = [
        .init(value: 20),
        .init(value: 30),
        .init(value: 5),
        .init(value: 100),
        .init(value: 80)
    ]
    
    var body: some View {
        BarChartShape(dataPoints: dataPoints)
            .fill(Color.red)
            .accessibilityLabel("Chart")
            .accessibilityChildren {
                HStack(alignment: .bottom, spacing: 0) {
                    ForEach(dataPoints) { point in
                        RoundedRectangle(cornerRadius: 8)
                            .fill(Color.blue)  // You can change the color here
                            .frame(width: 20, height: point.value)
                            .accessibilityValue(Text(String(Int(point.value)))
                        }
                    }
                }
            }
        }
    }
```

上述代码已添加了所需的 `DataPoint` 结构体和 `ContentView`，并通过 `BarChartShape` 创建了柱状图。此代码将以红色柱状图的形式显示数据点，每个数据点的值决定柱状的高度，同时也包括辅助功能信息以提供无障碍体验。

请注意，柱状图的颜色可以通过 `.fill(Color.red)` 进行自定义。在上述代码中，将柱状图填充颜色设为红色。您可以根据需要自行更改填充颜色。

运行截图：

![](https://files.mdnice.com/user/17787/c7bc4e90-32b0-4cc4-969c-dec3e2a8fa13.png)

## 总结

今天，我们了解了 SwiftUI 为我们提供的又一个强大的可访问性视图修饰符。SwiftUI 凭借提供如此多友好的 API，简化了我们为了使我们的应用对每个人都具有可访问性而必须做的工作，做得非常出色。
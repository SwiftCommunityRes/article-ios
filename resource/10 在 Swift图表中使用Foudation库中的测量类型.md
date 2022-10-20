# 在 Swift 图表中使用 Foudation 库中的测量类型


## 前言

在这篇文章中，我们将建立一个条形图，比较基督城地区自然散步的持续时间。我们将使用今年推出的新的Swift `Charts` 框架，并将看到如何绘制默认不符合 `Plottable` 协议的类型的数据，如 `Measurement<UnitDuration>`。

## 定义图表的数据

让我们先定义一下要在图表中展现的数据。

我们声明了一个包含标题和步行时间（小时）的 `Walk` 结构体。我们使用 `Foundation` 框架中的测量类型[Measurement](https://developer.apple.com/documentation/foundation/measurement "Measurement")和单位类型[UnitDuration](https://developer.apple.com/documentation/foundation/unitduration "UnitDuration")来表示每次步行的时间。

```swift
struct Walk {
    let title: String
    let duration: Measurement<UnitDuration>
}
```

我们在数组 `works` 中存储要在图表中显示的数据。

```swift
let walks = [
    Walk(
        title: "Taylors Mistake to Sumner Beach Coastal Walk",
        duration: Measurement(value: 3.1, unit: .hours)
    ),
    Walk(
        title: "Bottle Lake Forest",
        duration: Measurement(value: 2, unit: .hours)
    ),
    Walk(
        title: "Old Halswell Quarry Loop",
        duration: Measurement(value: 0.5, unit: .hours)
    ),
    ...
]
```

## 在图表中使用测量值

尝试直接在图表中使用测量值

让我们定义一个 `Chart`，并将 `walks` 数组作为数据参数传递给它。因为我们知道我们的`walk` 标题是唯一的，所以我们可以直接使用它们作为 `id`，但你也可以将你的数据模型改为 `Identifiable`。

```swift
Chart(walks, id: \.title) { walk in
    BarMark(
        x: .value("Duration", walk.duration),
        y: .value("Walk", walk.title)
    )
}
```

注意，因为 `Measurement<UnitDuration>` 没有遵守 `Plottable` 协议，我们会得到一个错误：「Initializer 'init(x:y:width:height:stacking:)' requires that 'Measurement<UnitDuration>' conform to 'Plottable'」

`BarkMark` 的初始化器期望收到一个用于 x 和 y 的 `PlottableValue` 参数。而且 `PlottableValue` 的值类型必须符合 `Plottable` 协议。

我们有几个选择来解决这个错误。我们可以提取测量值的 `value`，它是一个 `Double` 类型，它是默认符合 `Plottable` 的，我们可以扩展具有 `Plottable` 一致性的 `Measurement<UnitDuration>`，或者我们可以定义一个包装了测量的类型并使其符合 `Plottable` 协议。

如果我们简单地从测量值中提取，我们就会失去上下文，不知道用什么单位来创建测量值。这意味着，我们将无法正确格式化图表的标签来向用户表示单位。虽然我们可以记住我们在创建测量时使用了小时 `hours`，但这并不理想。例如，我们可以决定以后改变数据模型，以分钟为单位存储持续时间，或者数据可能来自其他地方，所以手动重构单位并不是一个完美的解决方案。

用 `Plottable` 的一致性来扩展 `Measurement<UnitDuration>` 是可行的，但根据 Swift 中关于外部类型的追溯一致性的警告 ([Warning for Retroactive Conformances of External Types](https://github.com/apple/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md "Warning for Retroactive Conformances of External Types"))，如果 Swift Charts 在未来添加了这种一致性，它可能会被破坏。



我们将研究如何定义我们自己的类型来包装 `measurement`，并为我们的自定义类型添加 `Plottable` 的一致性。

## 设计一个包装器类型

设计一个符合 Plottable 标准的包装器类型
  
我们将定义一个自定义的 `PlottableMeasurement` 类型，并使其成为通用的，所以它可以容纳任何类型的单位的测量类型。

```swift
struct PlottableMeasurement<UnitType: Unit> {
    var measurement: Measurement<UnitType>
}
```

然后，我们将为 `PlottableMeasurement` 添加 `Plottable` 的一致性，其单位为 `UnitDuration` 类型。我们可以在将来添加对其他单位的支持。

```swift
extension PlottableMeasurement: Plottable where UnitType == UnitDuration {
    var primitivePlottable: Double {
        self.measurement.converted(to: .minutes).value
    }
    
    init?(primitivePlottable: Double) {
        self.init(
            measurement: Measurement(
                value: primitivePlottable,
                unit: .minutes
            )
        )
    }
}
```

`Plottable` 协议有两个要求： `primitivePlottable` 属性必须返回原始类型之一，如 `Double`、`String` 或 `Date`，以及一个可失败的初始化器，从原始 `plottable` 类型创建一个值。

我决定将测量值转换为分钟，但你可以选择适合你需要的任何其他单位。只是在与原始值转换时要使用相同的单位，这一点很重要。



我们现在可以更新我们的图表，以使用我们的自定义 `Plottable` 类型。

```swift
Chart(walks, id: \.title) { walk in
    BarMark(
        x: .value(
            "Duration",
            PlottableMeasurement(measurement: walk.duration)
        ),
        y: .value("Walk", walk.title)
    )
}
```

它可以工作，但X轴上的标签没有格式化，没有向用户显示测量单位。我们接下来要解决这个问题。

![](https://nilcoalescing.com/static/blog/UsingMeasurementsFromFoundationAsValuesInSwiftCharts/Chart-without-formatting.GXK9NAqspTDEYy9wm4szJRGI8sasR0fo_xmhJSdYgQA.png)

## 显示格式化标签

显示带有测量单位的格式化标签

为了定制X轴上的标签，我们将使用`chartXAxis(content:)`修改器，并用传递给我们的值重构x轴的标记。

```swift
Chart(walks, id: \.title) { ... }
    .chartXAxis {
        AxisMarks { value in
            AxisGridLine()
            AxisValueLabel("""
            \(value.as(PlottableMeasurement.self)!
                .measurement
                .converted(to: .hours),
            format: .measurement(
                width: .narrow,
                numberFormatStyle: .number.precision(
                    .fractionLength(0))
                )
            )
            """)
        }
    }
```

我们首先添加网格线，然后重构给定值的标签。

`AxisValueLabel`在初始化器中接受一个`LocalizedStringKey`，它可以通过插值测量和指定其格式风格来构建。

我们收到的值是使用我们在 `Plottable` 一致性中定义的初始化器创建的，所以在我们的案例中，测量值是以分钟为单位提供的。但我相信对于这个特定的图表，使用小时会更好。我们可以很容易地将测量值转换为插值内部所需的单位。在这里，我们确定该值是 `PlottableMeasurement` 类型的，所以我们可以强制解包类型转换。

我选择了缩小的格式和小数点后零位数作为数字样式，但你可以根据你的具体图表调整这些设置。

最后的结果是在X轴上显示以小时为单位的格式化持续时间。

![](https://nilcoalescing.com/static/blog/UsingMeasurementsFromFoundationAsValuesInSwiftCharts/Chart-with-formatting.gOyj3M-zKoErXg8eGC4TcKBVM43CpFVE2lNU3k8zF9E.png)

你可以从我们的 GitHub repo 中获得这篇文章中使用的项目的完整 [示例代码](https://github.com/SwiftCommunityRes/SwiftUI-Code-Examples/blob/main/Using-Measurements-from-Foundation-as-values-in-Swift-Charts/Using-Measurements-from-Foundation-as-values-in-Swift-Charts.swift)。

> 来源：[Using Measurements from Foundation for values in Swift Charts](https://nilcoalescing.com/blog/UsingMeasurementsFromFoundationAsValuesInSwiftCharts/)

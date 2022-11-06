![Create a line chart with SwiftUI Charts in iOS 16](https://swdevnotes.com/images/swift/2022/0807/create-a-line-chart-with-swiftui-charts-in-ios-16-thumb.png)

苹果在 [WWWDC 2022](https://developer.apple.com/wwdc22/) 上推出了 SwiftUI 图表，这使得在 SwiftUI 视图中创建图表变得异常简单。图表是以丰富的格式呈现可视化数据的一种很好的方式，而且易于理解。本文展示了如何用比以前从头开始创建同样的折线图少得多的代码轻松创建折线图。此外，自定义图表的外观和感觉以及使图表中的信息易于访问也是非常容易的。

如以前的文章所示，不使用 SwiftUI Charts 也可以创建一个折线图。然而，使用 [Charts](https://developer.apple.com/documentation/charts) 框架可以提供大量的图表来探索对应用程序中的数据最有效的方法，从而使它变得更加容易。

下面是以前关于在 SwiftUI 中从头开始创建条形图和线形图的文章。

- [在SwiftUI中创建折线图](https://swdevnotes.com/swift/2021/create-a-line-chart-in-swiftui/)
- [How to create a Bar Chart in SwiftUI](https://swdevnotes.com/swift/2021/how-to-create-bar-chart-swiftui/)

## 简单折线图

从包含一周的步数的数据开始，类似于[在SwiftUI中创建折线图](https://swdevnotes.com/swift/2021/create-a-line-chart-in-swiftui/)中使用的数据。定义一个结构来保存日期和该日的步数，并为当前周创建一个数组。

```swift
struct StepCount: Identifiable {
    let id = UUID()
    let weekday: Date
    let steps: Int
    
    init(day: String, steps: Int) {
        let formatter = DateFormatter()
        formatter.dateFormat = "yyyyMMdd"
        
        self.weekday = formatter.date(from: day) ?? Date.distantPast
        self.steps = steps
    }
}


let currentWeek: [StepCount] = [
    StepCount(day: "20220717", steps: 4200),
    StepCount(day: "20220718", steps: 15000),
    StepCount(day: "20220719", steps: 2800),
    StepCount(day: "20220720", steps: 10800),
    StepCount(day: "20220721", steps: 5300),
    StepCount(day: "20220722", steps: 10400),
    StepCount(day: "20220723", steps: 4000)
]
```

要创建一个折线图，为步数数据中的每个元素创建一个带有[LineMark](https://developer.apple.com/documentation/charts/linemark)的图表。在`LineMark`的 X 值中指定工作日，在 Y 值中指定步数。注意，还需要导入`Charts`框架。

这就为步数数据创建了一个线形图。由于只有一个系列的数据，`ForEach` 可以省略，数据可以直接传递给 `Chart` 初始化器。两个部分都产生相同的折线图。

```swift
import SwiftUI
import Charts

struct LineChart1: View {
    var body: some View {
        VStack {
            GroupBox ( "Line Chart - Step Count") {
                Chart {
                    ForEach(currentWeek) {
                        LineMark(
                            x: .value("Week Day", $0.weekday, unit: .day),
                            y: .value("Step Count", $0.steps)
                        )
                    }
                }
            }
            
            GroupBox ( "Line Chart - Step Count") {
                Chart(currentWeek) {
                    LineMark(
                        x: .value("Week Day", $0.weekday, unit: .day),
                        y: .value("Step Count", $0.steps)
                    )
                    
                }
            }
        }
    }
}
```

![使用 SwiftUI Charts 创建的折线图显示每日步数](https://swdevnotes.com/images/swift/2022/0807/simple-line-chart.png)

使用 SwiftUI Charts 创建的折线图显示每日步数

## 其他图表

SwiftUI Charts 有许多可用的图表选项。这些可以通过将图表标记从`LineMark`改为其他类型的标记（如`BarMark`）来生成条形图。

```swift
struct OtherCharts: View {
    var body: some View {
        VStack {
            GroupBox ( "Line Chart - Step count") {
                Chart(currentWeek) {
                    LineMark(
                        x: .value("Week Day", $0.weekday, unit: .day),
                        y: .value("Step Count", $0.steps)
                    )
                }
            }
            
            GroupBox ( "Bar Chart - Step count") {
                Chart(currentWeek) {
                    BarMark(
                        x: .value("Week Day", $0.weekday, unit: .day),
                        y: .value("Step Count", $0.steps)
                    )
                }
            }
            
            GroupBox ( "Point Chart - Step count") {
                Chart(currentWeek) {
                    PointMark(
                        x: .value("Week Day", $0.weekday, unit: .day),
                        y: .value("Step Count", $0.steps)
                    )
                }
            }
            
            GroupBox ( "Rectangle Chart - Step count") {
                Chart(currentWeek) {
                    RectangleMark(
                        x: .value("Week Day", $0.weekday, unit: .day),
                        y: .value("Step Count", $0.steps)
                    )
                }
            }
            
            GroupBox ( "Area Chart - Step count") {
                Chart(currentWeek) {
                    AreaMark(
                        x: .value("Week Day", $0.weekday, unit: .day),
                        y: .value("Step Count", $0.steps)
                    )
                }
            }
        }
    }
}
```

![使用 SwiftUI 图表创建的其他图表类型，显示每日步数](https://swdevnotes.com/images/swift/2022/0807/other-charts.png)

使用 SwiftUI 图表创建的其他图表类型，显示每日步数

## 让折线图增加可访问性

将图表植入 SwiftUI 的一个好处是，可以很容易地使用[可访问性修饰符](https://developer.apple.com/documentation/swiftui/view-accessibility)使图表变得可访问。为 StepCount 添加一个计算属性，将数据返回为一个字符串，可由 `accessibilityLabel` 使用。然后为图表中的每个标记添加可访问性标签和值。

```swift
struct StepCount: Identifiable {
    let id = UUID()
    let weekday: Date
    let steps: Int
    
    init(day: String, steps: Int) {
        let formatter = DateFormatter()
        formatter.dateFormat = "yyyyMMdd"
        
        self.weekday = formatter.date(from: day) ?? Date.distantPast
        self.steps = steps
    }
    
    var weekdayString: String {
        let dateFormatter = DateFormatter()
        dateFormatter.dateFormat = "yyyyMMdd"
        dateFormatter.dateStyle = .long
        dateFormatter.timeStyle = .none
        dateFormatter.locale = Locale(identifier: "en_US")
        return  dateFormatter.string(from: weekday)
    }
}
```

```swift
    GroupBox ( "Line Chart - Daily Step Count") {
        Chart(currentWeek) {
            LineMark(
                x: .value("Week Day", $0.weekday, unit: .day),
                y: .value("Step Count", $0.steps)
            )
            .accessibilityLabel($0.weekdayString)
            .accessibilityValue("\($0.steps) Steps")
        }
    }
```

![](https://swdevnotes.com/images/swift/2022/0807/accessibile-line-chart.png)

在 SwiftUI 图表中使折线图可访问性

## 为折线图添加多个数据序列

折线图是比较两个不同系列数据的好方法。创建第二个系列，即前一周的步数，并将这两个系列添加到折线图中。

```swift
let previousWeek: [StepCount] = [
    StepCount(day: "20220710", steps: 15800),
    StepCount(day: "20220711", steps: 7300),
    StepCount(day: "20220712", steps: 8200),
    StepCount(day: "20220713", steps: 25600),
    StepCount(day: "20220714", steps: 16100),
    StepCount(day: "20220715", steps: 16500),
    StepCount(day: "20220716", steps: 3200)
]

let currentWeek: [StepCount] = [
    StepCount(day: "20220717", steps: 4200),
    StepCount(day: "20220718", steps: 15000),
    StepCount(day: "20220719", steps: 2800),
    StepCount(day: "20220720", steps: 10800),
    StepCount(day: "20220721", steps: 5300),
    StepCount(day: "20220722", steps: 10400),
    StepCount(day: "20220723", steps: 4000)
]

let stepData = [
    (period: "Current Week", data: currentWeek),
    (period: "Previous Week", data: previousWeek)
]
```

第一次尝试添加这两个系列的数据没有按预期显示。

```swift
struct LineChart2: View {
    var body: some View {
        GroupBox ( "Line Chart - Daily Step Count") {
            Chart {
                ForEach(stepData, id: \.period) {
                    ForEach($0.data) {
                        LineMark(
                            x: .value("Week Day", $0.weekday, unit: .day),
                            y: .value("Step Count", $0.steps)
                        )
                        .accessibilityLabel($0.weekdayString)
                        .accessibilityValue("\($0.steps) Steps")
                    }
                }
            }
        }
    }
}
```

![第一次尝试在 SwiftUI Charts 中创建一个包含两个系列步数数据的折线图](https://swdevnotes.com/images/swift/2022/0807/incorrect-line-chart.png)

第一次尝试在 SwiftUI Charts 中创建一个包含两个系列步数数据的折线图

## 显示步数系列

在折线图中显示多个基于工作日的步数系列

最初尝试在折线图中显示多组数据的问题是X轴使用了日期。当前的周数紧接着上一周，所以每一个点都是沿着X轴线性递增绘制的。

有必要只用工作日作为X轴的数值，这样所有的周日都在同一个X坐标上绘制。

在`StepCount`中添加另一个计算属性，以便以字符串格式返回工作日的短日。

```swift
struct StepCount: Identifiable {

    . . .
    
        
    var shortDay: String {
        let dateFormatter = DateFormatter()
        dateFormatter.dateFormat = "EEE"
        return  dateFormatter.string(from: weekday)
    }
}
```

此 `shortDay` 用于图表中 `LineMarks` 的 x 值。另外，前景的样式设置为基于`stepCount`数组的周期。折线图使用 x 轴的工作日来显示两周的步数，以便在周之间进行比较。

```swift
struct LineChart3: View {
    var body: some View {
        VStack {
            GroupBox ( "Line Chart - Daily Step Count") {
                Chart {
                    ForEach(stepData, id: \.period) { steps in
                        ForEach(steps.data) {
                            LineMark(
                                x: .value("Week Day", $0.shortDay),
                                y: .value("Step Count", $0.steps)
                            )
                            .foregroundStyle(by: .value("Week", steps.period))
                            .accessibilityLabel($0.weekdayString)
                            .accessibilityValue("\($0.steps) Steps")
                        }
                    }
                }
                .frame(height:400)
            }
            .padding()

            Spacer()
        }
    }
}
```

![SwiftUI图表中带有两个系列的步数数据的折线图](https://swdevnotes.com/images/swift/2022/0807/line-chart-multiple-series.png)

SwiftUI 图表中带有两个系列的步数数据的折线图

## 结论

在 SwiftUI [Charts](https://developer.apple.com/documentation/charts) 中还有很多东西可以探索。使用这个框架显然比从头开始建立你自己的图表要好。


>  来自：[Create a line chart with SwiftUI Charts in iOS 16](https://swdevnotes.com/swift/2022/create-a-line-chart-with-swiftui-charts-in-ios-16/)

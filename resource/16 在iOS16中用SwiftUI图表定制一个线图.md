![Customise a line chart with SwiftUI Charts in iOS 16](https://swdevnotes.com/images/swift/2022/0814/customise-a-line-chart-with-swiftui-charts-in-ios-16-feature.png)

在iOS 16中引入的SwiftUI图表，可以以直观的视觉格式呈现数据，并且可以使用SwiftUI图表快速创建。本文演示了几种定制折线图并与区域图结合来展示数据的方法。



## 默认折线图

从[在iOS 16中用SwiftUI Charts创建一个折线图](https://swdevnotes.com/swift/2022/create-a-line-chart-with-swiftui-charts-in-ios-16/)中使用SwiftUI [Charts](https://developer.apple.com/documentation/charts)创建默认折线图开始。这显示了两个不同星期的步数数据，比较了每个工作日的步数。

```swift
struct ChartView1: View {
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
                            .accessibilityLabel("\($0.weekdayString)")
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

![使用SwiftUI图表创建的默认折线图](https://swdevnotes.com/images/swift/2022/0814/default-line-chart.png)

使用SwiftUI图表创建的默认折线图



## 改变图表背后的背景

技术上讲，这与图表无关，但 `GroupBox` 的背景可以用颜色或[`GroupBoxStyle`](https://developer.apple.com/documentation/swiftui/groupboxstyle)来设置。

```swift
struct YellowGroupBoxStyle: GroupBoxStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.content
            .padding(.top, 30)
            .padding(20)
            .background(Color(hue: 0.10, saturation: 0.10, brightness: 0.98))
            .cornerRadius(20)
            .overlay(
                configuration.label.padding(10),
                alignment: .topLeading
            )
    }
}
```

```swift
struct ChartView2: View {
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
                            .accessibilityLabel("\($0.weekdayString)")
                            .accessibilityValue("\($0.steps) Steps")
                        }
                    }
                }
                .frame(height:400)
            }
            // Add a style to the GroupBox
            .groupBoxStyle(YellowGroupBoxStyle())
            .padding()
            
            Spacer()
        }
    }
}
```

![为 GroupBox 背景设置样式](https://swdevnotes.com/images/swift/2022/0814/line-chart-background.png)

为 GroupBox 背景设置样式



## 设置绘图或图表的背景

可以使用[`chartPlotStyle`](https://developer.apple.com/documentation/swiftui/view/chartplotstyle(content:))为图表绘图区域设置背景，或者使用[`chartBackground`](https://developer.apple.com/documentation/swiftui/view/chartplotstyle(content:))为整个图表设置一个背景。



### 设置绘图区域背景

```swift
GroupBox ( "Line Chart - Plot Background") {
    Chart {
        ForEach(stepData, id: \.period) { steps in
            ForEach(steps.data) {
                LineMark(
                    x: .value("Week Day", $0.shortDay),
                    y: .value("Step Count", $0.steps)
                )
                .foregroundStyle(by: .value("Week", steps.period))
                .accessibilityLabel("\($0.weekdayString)")
                .accessibilityValue("\($0.steps) Steps")
            }
        }
    }
    .chartPlotStyle { plotArea in
        plotArea
            .background(.orange.opacity(0.1))
            .border(.orange, width: 2)
    }
    .frame(height:200)
}
.groupBoxStyle(YellowGroupBoxStyle())
```

### 设置图表背景

```swift
GroupBox ( "Line Chart - Chart Background") {
    Chart {
        ForEach(stepData, id: \.period) { steps in
            ForEach(steps.data) {
                LineMark(
                    x: .value("Week Day", $0.shortDay),
                    y: .value("Step Count", $0.steps)
                )
                .foregroundStyle(by: .value("Week", steps.period))
                .accessibilityLabel("\($0.weekdayString)")
                .accessibilityValue("\($0.steps) Steps")
            }
        }
    }
    .chartBackground { chartProxy in
        Color.red.opacity(0.1)
    }
    .frame(height:200)
}
.groupBoxStyle(YellowGroupBoxStyle())            
```

### 设置绘图区域和图表的背景

```swift
GroupBox ( "Line Chart - Plot & Chart Backgroundt") {
    Chart {
        ForEach(stepData, id: \.period) { steps in
            ForEach(steps.data) {
                LineMark(
                    x: .value("Week Day", $0.shortDay),
                    y: .value("Step Count", $0.steps)
                )
                .foregroundStyle(by: .value("Week", steps.period))
                .accessibilityLabel("\($0.weekdayString)")
                .accessibilityValue("\($0.steps) Steps")
            }
        }
    }
    .chartBackground { chartProxy in
        Color.red.opacity(0.1)
    }
    .chartPlotStyle { plotArea in
        plotArea
            .background(.orange.opacity(0.1))
            .border(.orange, width: 2)
    }
    
    .frame(height:200)
}
.groupBoxStyle(YellowGroupBoxStyle())
```

![使用SwiftUI Charts在图表情节和全图表上设置背景](https://swdevnotes.com/images/swift/2022/0814/line-chart-plot-chart-background.png)

使用SwiftUI Charts在绘图区域和全图表上设置背景



## 将Y轴移至左侧边缘（leading）

可以隐藏坐标轴或调整坐标轴的位置，比如将Y轴放在图表的左侧(leading)。y轴默认显示在图表的右方(trailing)。可以使用`chartYAxis`的[`AxisMarks`](https://developer.apple.com/documentation/swiftui/view/chartyaxis(content:))将其放置在左侧。也可以通过设置可见性属性为隐藏来完全隐藏轴。

```swift
struct ChartView4: View {
    var body: some View {
        VStack {
            GroupBox ( "Line Chart - Y-axis on leading edge") {
                Chart {
                    ForEach(stepData, id: \.period) { steps in
                        ForEach(steps.data) {
                            LineMark(
                                x: .value("Week Day", $0.shortDay),
                                y: .value("Step Count", $0.steps)
                            )
                            .foregroundStyle(by: .value("Week", steps.period))
                            .accessibilityLabel("\($0.weekdayString)")
                            .accessibilityValue("\($0.steps) Steps")
                        }
                    }
                }
                .chartPlotStyle { plotArea in
                    plotArea
                        .background(Color(hue: 0.12, saturation: 0.10, brightness: 0.92))
                }
                // Place the y-axis on the leading side of the chart
                .chartYAxis {
                   AxisMarks(position: .leading)
                }
                .frame(height:400)
            }
            .groupBoxStyle(YellowGroupBoxStyle())
            .padding()
            
            Spacer()
        }
    }
}
```

![使用SwiftUI图表将Y轴置于图表的左侧](https://swdevnotes.com/images/swift/2022/0814/line-chart-y-axis-leading.png)

使用SwiftUI图表将Y轴置于图表的左侧 



## 移动图表的图例

图表图例默认显示在图表的底部。图例可以放在图表的任何一面，也可以放在图表的多个位置上。

```swift
GroupBox ( "Line Chart - legend overlay on top center") {
    Chart {
        ForEach(stepData, id: \.period) { steps in
            ForEach(steps.data) {
                LineMark(
                    x: .value("Week Day", $0.shortDay),
                    y: .value("Step Count", $0.steps)
                )
                .foregroundStyle(by: .value("Week", steps.period))
                .accessibilityLabel("\($0.weekdayString)")
                .accessibilityValue("\($0.steps) Steps")
            }
        }
    }
    // Position the Legend
    .chartLegend(position: .overlay, alignment: .top)
    .chartPlotStyle { plotArea in
        plotArea
            .background(Color(hue: 0.12, saturation: 0.10, brightness: 0.92))
    }
    .chartYAxis() {
        AxisMarks(position: .leading)
    }
    .frame(height:200)
}
.groupBoxStyle(YellowGroupBoxStyle())
```

```swift
GroupBox ( "Line Chart - legend trailing center") {
    Chart {
        ForEach(stepData, id: \.period) { steps in
            ForEach(steps.data) {
                LineMark(
                    x: .value("Week Day", $0.shortDay),
                    y: .value("Step Count", $0.steps)
                )
                .foregroundStyle(by: .value("Week", steps.period))
                .accessibilityLabel("\($0.weekdayString)")
                .accessibilityValue("\($0.steps) Steps")
            }
        }
    }
    // Position the Legend
    .chartLegend(position: .trailing, alignment: .center, spacing: 10)
    .chartPlotStyle { plotArea in
        plotArea
            .background(Color(hue: 0.12, saturation: 0.10, brightness: 0.92))
    }
    .chartYAxis() {
        AxisMarks(position: .leading)
    }
    .frame(height:200)
}
.groupBoxStyle(YellowGroupBoxStyle())
```

![使用 SwiftUI 图表定位图例，例如在图表的顶部或左侧](https://swdevnotes.com/images/swift/2022/0814/line-chart-legent-top.png)



### 改变折线线型

折线图用一条直线将图表上的数据点连接起来。插值方法（[interpolationMethod](https://developer.apple.com/documentation/charts/chartcontent/interpolationmethod(_:))）函数可以用各种方式将数据点通过曲线连接。

```swift
struct ChartView6: View {
    var body: some View {
        VStack(spacing:30) {
            GroupBox ( "Line Chart - Curved line connector") {
                Chart {
                    ForEach(stepData, id: \.period) { steps in
                        ForEach(steps.data) {
                            LineMark(
                                x: .value("Week Day", $0.shortDay),
                                y: .value("Step Count", $0.steps)
                            )
                            .foregroundStyle(by: .value("Week", steps.period))
                            // Use curved line to join points
                            .interpolationMethod(.catmullRom)
                            .accessibilityLabel("\($0.weekdayString)")
                            .accessibilityValue("\($0.steps) Steps")
                        }
                    }
                }
                .chartLegend(position: .overlay, alignment: .top)
                .chartPlotStyle { plotArea in
                    plotArea
                        .background(Color(hue: 0.12, saturation: 0.10, brightness: 0.92))
                }
                .chartYAxis() {
                    AxisMarks(position: .leading)
                }
                .frame(height:300)
            }
            .groupBoxStyle(YellowGroupBoxStyle())
            
            GroupBox ( "Line Chart - Step line connector") {
                Chart {
                    ForEach(stepData, id: \.period) { steps in
                        ForEach(steps.data) {
                            LineMark(
                                x: .value("Week Day", $0.shortDay),
                                y: .value("Step Count", $0.steps)
                            )
                            .foregroundStyle(by: .value("Week", steps.period))
                            // Use step line to join points
                            .interpolationMethod(.stepCenter)
                            .accessibilityLabel("\($0.weekdayString)")
                            .accessibilityValue("\($0.steps) Steps")
                        }
                    }
                }
                .chartLegend(position: .overlay, alignment: .top)
                .chartPlotStyle { plotArea in
                    plotArea
                        .background(Color(hue: 0.12, saturation: 0.10, brightness: 0.92))
                }
                .chartYAxis() {
                    AxisMarks(position: .leading)
                }
                .frame(height:300)
            }
            .groupBoxStyle(YellowGroupBoxStyle())
            

            Spacer()
        }
        .padding()
    }
}
```

![在 SwiftUI 图表中更改将数据点连接线型](https://swdevnotes.com/images/swift/2022/0814/line-chart-curve.png)

在 SwiftUI 图表中更改将数据点连接线型



## 改变折线的颜色

可以使用[chartForegroundStyleScale](https://developer.apple.com/documentation/swiftui/view/chartforegroundstylescale(_:)?changes=_10)来设置线形图中线条的默认颜色。

```swift
struct ChartView7: View {
    var body: some View {
        VStack() {
            GroupBox ( "Line Chart - Custom line colors") {
                Chart {
                    ForEach(stepData, id: \.period) { steps in
                        ForEach(steps.data) {
                            LineMark(
                                x: .value("Week Day", $0.shortDay),
                                y: .value("Step Count", $0.steps)
                            )
                            .foregroundStyle(by: .value("Week", steps.period))
                            .interpolationMethod(.catmullRom)
                            .symbol(by: .value("Week", steps.period))
                            .symbolSize(30)
                            .accessibilityLabel("\($0.weekdayString)")
                            .accessibilityValue("\($0.steps) Steps")
                        }
                    }
                }
                // Set color for each data in the chart
                .chartForegroundStyleScale([
                    "Current Week" : Color(hue: 0.33, saturation: 0.81, brightness: 0.76),
                    "Previous Week": Color(hue: 0.69, saturation: 0.19, brightness: 0.79)
                ])
                .chartLegend(position: .overlay, alignment: .top)
                .chartPlotStyle { plotArea in
                    plotArea
                        .background(Color(hue: 0.12, saturation: 0.10, brightness: 0.92))
                }
                .chartYAxis() {
                    AxisMarks(position: .leading)
                }
                .frame(height:400)
            }
            .groupBoxStyle(YellowGroupBoxStyle())
           
            Spacer()
        }
        .padding()
    }
}
```

![为SwiftUI图表中的线条设置自定义颜色](https://swdevnotes.com/images/swift/2022/0814/line-chart-line-colors.png)

为SwiftUI图表中的线条设置自定义颜色



## 改变折线风格

线形图上的线条可以通过使用[StrokeStyle](https://developer.apple.com/documentation/swiftui/strokestyle)设置`lineStyle`来修改。在步骤数据中使用了两种不同的风格，以区分前一周的数据和当前的数据。此外，还为图表上的数据点设置了一个自定义符号。

```swift
struct ChartView8: View {
    
    let prevColor = Color(hue: 0.69, saturation: 0.19, brightness: 0.79)
    let curColor = Color(hue: 0.33, saturation: 0.81, brightness: 0.76)
    
    var body: some View {
        VStack() {
            GroupBox ( "Line Chart - Line color and format") {
                Chart {
                    ForEach(previousWeek) {
                        LineMark(
                            x: .value("Week Day", $0.shortDay),
                            y: .value("Step Count", $0.steps)
                        )
                        .interpolationMethod(.catmullRom)
                        .foregroundStyle(prevColor)
                        .foregroundStyle(by: .value("Week", "Previous Week"))
                        .lineStyle(StrokeStyle(lineWidth: 3, dash: [5, 10]))
                        .symbol() {
                            Rectangle()
                                .fill(prevColor)
                                .frame(width: 8, height: 8)
                        }
                        .symbolSize(30)
                        .accessibilityLabel("\($0.weekdayString)")
                        .accessibilityValue("\($0.steps) Steps")
                    }
                    
                    ForEach(currentWeek) {
                        LineMark(
                            x: .value("Week Day", $0.shortDay),
                            y: .value("Step Count", $0.steps)
                        )
                        .interpolationMethod(.catmullRom)
                        .foregroundStyle(curColor)
                        .foregroundStyle(by: .value("Week", "Current Week"))
                        .lineStyle(StrokeStyle(lineWidth: 3))
                        .symbol() {
                            Circle()
                                .fill(curColor)
                                .frame(width: 10)
                        }
                        .symbolSize(30)
                        .accessibilityLabel("\($0.weekdayString)")
                        .accessibilityValue("\($0.steps) Steps")
                    }
                }
                // Set the Y axis scale
                .chartYScale(domain: 0...30000)
                .chartForegroundStyleScale([
                    "Current Week" : curColor,
                    "Previous Week": prevColor
                ])
                .chartLegend(position: .overlay, alignment: .top)
                .chartPlotStyle { plotArea in
                    plotArea
                        .background(Color(hue: 0.12, saturation: 0.10, brightness: 0.92))
                }
                .chartYAxis() {
                    AxisMarks(position: .leading)
                }
                .frame(height:400)
            }
            .groupBoxStyle(YellowGroupBoxStyle())
            Spacer()
        }
        .padding()
    }
}
```

![为 SwiftUI 图表中的一个数据集设置自定义线型](https://swdevnotes.com/images/swift/2022/0814/line-chart-line-style.png)

为 SwiftUI 图表中的一个数据集设置自定义线型



##  结合面积图和折线图

最后，将折线图与面积图结合起来，帮助区分一个数据集与另一个数据集。区域图只为当前一周的数据添加，并且区域的颜色被设置为渐变的线下。

```swift
struct ChartView9: View {
        
    var body: some View {
    
        let prevColor = Color(hue: 0.69, saturation: 0.19, brightness: 0.79)
        let curColor = Color(hue: 0.33, saturation: 0.81, brightness: 0.76)
        let curGradient = LinearGradient(
            gradient: Gradient (
                colors: [
                    curColor.opacity(0.5),
                    curColor.opacity(0.2),
                    curColor.opacity(0.05),
                ]
            ),
            startPoint: .top,
            endPoint: .bottom
        )

        VStack() {
            GroupBox ( "Line Chart - Combine LIne and Area chart") {
                Chart {
                    ForEach(previousWeek) {
                        LineMark(
                            x: .value("Week Day", $0.shortDay),
                            y: .value("Step Count", $0.steps)
                        )
                        .interpolationMethod(.catmullRom)
                        .foregroundStyle(prevColor)
                        .foregroundStyle(by: .value("Week", "Previous Week"))
                        .lineStyle(StrokeStyle(lineWidth: 3, dash: [5, 10]))
                        .symbol() {
                            Rectangle()
                                .fill(prevColor)
                                .frame(width: 8, height: 8)
                        }
                        .symbolSize(30)
                        .accessibilityLabel("\($0.weekdayString)")
                        .accessibilityValue("\($0.steps) Steps")
                    }
                    
                    ForEach(currentWeek) {
                        LineMark(
                            x: .value("Week Day", $0.shortDay),
                            y: .value("Step Count", $0.steps)
                        )
                        .interpolationMethod(.catmullRom)
                        .foregroundStyle(curColor)
                        .foregroundStyle(by: .value("Week", "Current Week"))
                        .lineStyle(StrokeStyle(lineWidth: 3))
                        .symbol() {
                            Circle()
                                .fill(curColor)
                                .frame(width: 10)
                        }
                        .symbolSize(30)
                        .accessibilityLabel("\($0.weekdayString)")
                        .accessibilityValue("\($0.steps) Steps")

                        AreaMark(
                            x: .value("Week Day", $0.shortDay),
                            y: .value("Step Count", $0.steps)
                        )
                        .interpolationMethod(.catmullRom)
                        .foregroundStyle(curGradient)
                        .foregroundStyle(by: .value("Week", "Current Week"))
                        .accessibilityLabel("\($0.weekdayString)")
                        .accessibilityValue("\($0.steps) Steps")
                        
                    }
                }
                // Set the Y axis scale
                .chartYScale(domain: 0...30000)
                
                .chartForegroundStyleScale([
                    "Current Week" : curColor,
                    "Previous Week": prevColor
                ])
                .chartLegend(position: .overlay, alignment: .top)
                .chartPlotStyle { plotArea in
                    plotArea
                        .background(Color(hue: 0.12, saturation: 0.10, brightness: 0.92))
                }
                .chartYAxis() {
                    AxisMarks(position: .leading)
                }
                .frame(height:400)
            }
            .groupBoxStyle(YellowGroupBoxStyle())
            Spacer()
        }
        .padding()
    }
}
```

![在SwiftUI图表中使用自定义颜色将折线图与面积图结合起来](https://swdevnotes.com/images/swift/2022/0814/line-chart-area-and-line.png)

在SwiftUI图表中使用自定义颜色将折线图与面积图结合起来



## 结论

SwiftUI Charts目前处于测试阶段，在Xcode性能和编译一些图表选项方面可能会有一些问题，但它很容易就能开始使用图表。帮助文档是可用的，而且很好，但我希望看到更多的代码示例。它是有很大的潜力来定制图表然后以直观的方式向应用程序的用户展示数据。

> 来自：[Customise a line chart with SwiftUI Charts in iOS 16](https://swdevnotes.com/swift/2022/customise-a-line-chart-with-swiftui-charts-in-ios-16/)  

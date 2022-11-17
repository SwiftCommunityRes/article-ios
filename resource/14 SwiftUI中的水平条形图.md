![](https://swdevnotes.com/images/swift/2021/0829/horizontal-bar-chart.gif)

水平条形图以矩形条的形式呈现数据类别，其宽度与它们所代表的数值成正比。本文展示了如何在垂直条形图的基础上创建一个水平柱状图。

水平条形图不是简单的垂直条形图的旋转。在 ` Numbers` 等应用程序中，水平条形图被定义为独立的图表类型，而不是垂直条形图。除了条形差异外，x 轴和 y 轴的格式也需要不同。

**系列文章**

1.   如何在 SwiftUI 中创建条形图
2.   SwiftUI 中的水平条形图
3.   在 iOS 16 中用 SwiftUI Charts 创建一个折线图
4.   在 iOS16 中用 SwiftUI 图表定制一个线图
5.   在 Swift 图表中使用 Foudation 库中的测量类型

## 将条形图转换为水平

水平条形图不仅仅是在垂直条形图上的配置，有一些元素是可以重复使用的。对于垂直条形图组件和水平条形图组件来说，重复使用一些结构和SwiftUI视图并不简单。标题和关键区域可以原样重用。创建 `BarChartView` 的副本，并将其名称改为 `BarChartHView`。它控制了图表的布局，其中的三个视图被改为 `YaxisHView`、`ChartAreaHView` 和 `XaxisHView`，它们最初只是垂直条形图中使用的视图的副本。`maxTickHeight` 被改为 `maxTickWidth`，因为它现在取决于可用的水平空间。

```swift
struct BarChartHView: View {
    var title: String
    var chartData: BarChartData
    var isShowingYAxis = true
    var isShowingXAxis = true
    var isShowingHeading = true
    var isShowingKey = true

    var body: some View {
        let data = chartData.data
        GeometryReader { gr in
            let axisWidth = gr.size.width * (isShowingYAxis ? 0.15 : 0.0)
            let axisHeight = gr.size.height * (isShowingXAxis ? 0.1 : 0.0)
            let keyHeight = gr.size.height * (isShowingKey ? 0.1 : 0.0)
            let headHeight = gr.size.height * (isShowingHeading ? 0.14 : 0.0)
            let fullChartHeight = gr.size.height - axisHeight - headHeight - keyHeight
            let fullChartWidth = gr.size.width - axisWidth
            
            let maxValue = data.flatMap { $0.values }.max()!
            let tickMarks = AxisParameters.getTicks(top: Int(maxValue))
            let maxTickWidth = fullChartWidth * 0.95
            let scaleFactor = maxTickWidth / CGFloat(tickMarks[tickMarks.count-1])
            
            VStack(spacing:0) {
                if isShowingHeading {
                    ChartHeaderView(title: title)
                        .frame(height: headHeight)
                }
                ZStack {
                    Rectangle()
                        .fill(Color(#colorLiteral(red: 0.8906477705, green: 0.9005050659, blue: 0.8208766097, alpha: 1)))
                    
                    VStack(spacing:0) {
                        if isShowingKey {
                            KeyView(keys: chartData.keys)
                                .frame(height: keyHeight)
                        }
                        
                        HStack(spacing:0) {
                            if isShowingYAxis {
                                YaxisHView(ticks: tickMarks, scaleFactor: Double(scaleFactor))
                                    .frame(width:axisWidth, height: fullChartHeight)
                            }
                            ChartAreaHView(data: data, scaleFactor: Double(scaleFactor))
                                .frame(height: fullChartHeight)
                        }
                        HStack(spacing:0) {
                            Rectangle()
                                .fill(Color.clear)
                                .frame(width:axisWidth, height:axisHeight)
                            if isShowingXAxis {
                                XaxisHView(data: data)
                                    .frame(height:axisHeight)
                            }
                        }
                    }
                }
            }
        }
    }
}
```

`ChartAreaHView` 与 `ChartAreaView` 几乎相同，只是Bars被放置在一个垂直的堆栈中，而不是水平的堆栈。

```swift
struct ChartAreaHView: View {
    var data: [DataItem]
    var scaleFactor: Double
    
    var body: some View {
        ZStack {
            RoundedRectangle(cornerRadius: 5.0)
                .fill(Color(#colorLiteral(red: 0.8906477705, green: 0.9005050659, blue: 0.8208766097, alpha: 1)))
            
            VStack {
                VStack(spacing:0) {
                    ForEach(data) { item in
                        BarHView(
                            name: item.name,
                            values: item.values,
                            scaleFactor: scaleFactor)
                    }
                }
            }
        }
    }
}
```
 
`BarHView` 是由原来的 `BarView` 改变的，以水平方式布局条形。矩形条的宽度与数据的值成正比。

```swift
struct BarHView: View {
    var name: String
    var values: [Double]
    var scaleFactor: Double
    
    var body: some View {
        GeometryReader { gr in
            let padHeight = gr.size.height * 0.07
            VStack(spacing:0) {
                Spacer()
                    .frame(height:padHeight)
                
                ForEach(values.indices) { i in
                    let barSize = values[i] * scaleFactor
                    
                    ZStack {
                        HStack(spacing:0) {
                            Rectangle()
                                .fill(ChartColors.BarColor(i))
                                .frame(width: min(5.0, CGFloat(barSize)), alignment: .trailing)
                            Spacer()
                        }
                        
                        HStack(spacing:0) {
                            RoundedRectangle(cornerRadius:5.0)
                                .fill(ChartColors.BarColor(i))
                                .frame(width: CGFloat(barSize), alignment: .trailing)
                                .overlay(
                                    Text("\(values[i], specifier: "%.0F")")
                                        .font(.footnote)
                                        .foregroundColor(.white)
                                        .fontWeight(.bold)
                                        .offset(x:-10, y:0)
                                    ,
                                    alignment: .trailing
                                )
                            Spacer()
                        }
                    }
                }

                Spacer()
                    .frame(height:padHeight)
            }
        }
    }
}
```

![](https://swdevnotes.com/images/swift/2021/0829/horizontal-bars.png)

## 更新 Y 轴

我们创建了一个 `YaxisHView` 视图，用于在水平条形图上显示 Y 轴和条形图中的数据类别。Y轴标签的 Swift 代码与垂直条形图的X轴代码相似，宽度设置与高度设置互换。两种图表类型的y轴线的代码都是一样的。

```swift
struct YaxisHView: View {
    var data: [DataItem]
    
    var body: some View {
        GeometryReader { gr in
            let labelHeight = (gr.size.height * 0.9) / CGFloat(data.count)
            let padHeight = gr.size.height * 0.05 / CGFloat(data.count)
            
            ZStack {
                Rectangle()
                    .fill(Color(#colorLiteral(red: 0.8906477705, green: 0.9005050659, blue: 0.8208766097, alpha: 1)))
                
                // y-axis line
                Rectangle()
                    .fill(Color.black)
                    .frame(width:1.5)
                    .offset(x: (gr.size.width/2.0)-1, y: 1)
                
                VStack(spacing:0) {
                    ForEach(data) { item in
                        Text(item.name)
                            .font(.footnote)
                            .frame(height: labelHeight)
                    }
                    .padding(.vertical, padHeight)
                }
            }
        }
    }
}
```

![](https://swdevnotes.com/images/swift/2021/0829/horizontal-bar-chart-y-axis.png)

## 更新X轴

同样，创建了一个 `XaxisHView` 视图来显示水平条形图的X轴，并使用与垂直条形图的Y轴类似的代码来布置刻度线和刻度值。

```swift
struct XaxisHView: View {
    var ticks: [Int]
    var scaleFactor: Double
    
    var body: some View {
        GeometryReader { gr in
            let fullChartWidth = gr.size.width
            ZStack {
                // x-axis line
                Rectangle()
                    .fill(Color.black)
                    .frame(height: 1.5)
                    .offset(x: 0, y: -(gr.size.height/2.0))
                
                // Tick marks
                ForEach(ticks, id:\.self) { t in
                    VStack {
                        Rectangle()
                            .frame(width: 1, height: 10)
                        Text("\(t)")
                            .font(.footnote)
                            .rotationEffect(Angle(degrees: -45))
                        Spacer()
                    }
                    .offset(x:  (CGFloat(t) * CGFloat(scaleFactor)) - (fullChartWidth/2.0) - 1)
                }
            }
        }
    }
}
```

![](https://swdevnotes.com/images/swift/2021/0829/horizontal-bar-chart-x-axis.png)



## 水平和垂直条形图

一个 iPad 模拟器被用来比较垂直和水平条形图的使用，以显示2018年五岁以下儿童死亡率最高的国家。柱状图的多数据功能被用来比较男孩和女孩的死亡率。

![](https://swdevnotes.com/images/swift/2021/0829/vertical-and-horizontal-bar-charts.png)

2018年最高的5岁以下儿童死亡率显示在垂直和水平条形图中。

水平条形图重用了垂直条形图的很多代码，所以显示或隐藏标题、键和轴的效果是有效的。在水平条形图中，显示条形图上的数值并隐藏X轴可以使图表更简洁。



![](https://swdevnotes.com/images/swift/2021/0829/horizontal-bar-chart.gif)

显示和隐藏水平条形图上的元素

## 结论

创建水平条形图的 SwiftUI 代码与创建垂直条形图的代码不同。在创建垂直条形图时学到的技术可以重复使用，但最好将水平条形图视为与垂直条形图不同的图表。当我们深入到轴等组件时，可以看到两个图表中的轴线都是一样的，但是它们的标签和定位在 x 和 y 之间是换位的。这可能是将这些组件分解成更小的 SwiftUI 视图并通过组合来重用的原因。

> 来自：[Horizontal Bar Chart in SwiftUI](https://swdevnotes.com/swift/2021/horizontal-bar-chart-in-swiftui/)
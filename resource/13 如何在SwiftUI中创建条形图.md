### 如何在 SwiftUI 中创建条形图

![如何在 SwiftUI 中创建条形图](https://swdevnotes.com/images/swift/2021/0725/how-to-create-bar-chart-swiftui-thumb.png)

条形图以矩形条的形式呈现数据的类别，其宽度和高度与它们表示的值成比例。本文将展示如何创建一个垂直条形图，其中矩形的高度将代表每个类别的值。

**系列文章**

1.   如何在 SwiftUI 中创建条形图
2.   SwiftUI 中的水平条形图
3.   在 iOS 16 中用 SwiftUI Charts 创建一个折线图
4.   在 iOS16 中用 SwiftUI 图表定制一个线图
5.   在 Swift 图表中使用 Foudation 库中的测量类型

## 开始图表布局

SwiftUI 对探索不同布局和预览实时视图结果是很友好的。很容易将部分内容提取到子视图中，以便每个部分都很小且易于维护。从将包含 `BarChartView` 以及可能的其他文本或数据的视图开始。这个 `BarChartView` 包含一个标题和一个图表区，它们由文本和圆角矩形表示。

```SWIFT
struct ChartView1: View {
    var body: some View {
        VStack {
            Text("Sample Bar Chart")
                .font(.title)

            BarChartView(
                title: "the chart title")
                .frame(width: 300, height: 300, alignment: .center)

            Spacer()
        }
    }
}
```

```SWIFT
struct BarChartView: View {
    var title: String

    var body: some View {
        GeometryReader { gr in
            let headHeight = gr.size.height * 0.10
            VStack {
                ChartHeaderView(title: title, height: headHeight)
                ChartAreaView()
            }
        }
    }
}
```

```SWIFT
struct ChartHeaderView: View {
    var title: String
    var height: CGFloat

    var body: some View {
        Text(title)
            .frame(height: height)
    }
}
```

```SWIFT
struct ChartAreaView: View {
    var body: some View {
        ZStack {
            RoundedRectangle(cornerRadius: 5.0)
                .fill(Color(#colorLiteral(red: 0.8906477705, green: 0.9005050659, blue: 0.8208766097, alpha: 1)))
        }
    }
}

```

![](https://swdevnotes.com/images/swift/2021/0725/Layout-chart.png)

## 图表区添加条形图

定义一些简单的数据类别，例如一周内每天的步数。以下列表数据被作为主视图的项目数据，每一条数据包含一个对（名称，值）。在真正的 app 里，这里的数据应该通过 ViewModel 从 model 里取数据。

每日步数数据

| Day  | Steps |
| ---- | ----- |
| Mon  | 898   |
| Tue  | 670   |
| Wed  | 725   |
| Thu  | 439   |
| Fri  | 1232  |
| Sat  | 771   |
| Sun  | 365   |

```SWIFT
struct DataItem: Identifiable {
    let name: String
    let value: Double
    let id = UUID()
}

struct ChartView2: View {

    let chartData: [DataItem] = [
        DataItem(name: "Mon", value: 898),
        DataItem(name: "Tue", value: 670),
        DataItem(name: "Wed", value: 725),
        DataItem(name: "Thu", value: 439),
        DataItem(name: "Fri", value: 1232),
        DataItem(name: "Sat", value: 771),
        DataItem(name: "Sun", value: 365)
    ]

    var body: some View {
        VStack {
            Text("Sample Bar Chart")
                .font(.title)

            BarChartView(
                title: "Daily step count", data: chartData)
                .frame(width: 350, height: 500, alignment: .center)

            Spacer()
        }
    }
}

```

更新 `BarChartView` 使数据可以作为参数传递到 `ChartAreaView`

```SWIFT
struct BarChartView: View {
    var title: String
    var data: [DataItem]

    var body: some View {
        GeometryReader { gr in
            let headHeight = gr.size.height * 0.10
            VStack {
                ChartHeaderView(title: title, height: headHeight)
                ChartAreaView(data: data)
            }
        }
    }
}

```

更新后的 `BarChartView` 需要一个 `DataItem` 的列表。[GeometryReader](https://developer.apple.com/documentation/swiftui/geometryreader) 被用来确定条形图的可用高度。数据中的最大值得到后并传递给每个 `BarView`。主图表区域保持原来的圆角矩形，并以水平堆叠的方式叠加一系列条形，每个 `DataItem` 一个。

```SWIFT
struct ChartAreaView: View {
    var data: [DataItem]

    var body: some View {
        GeometryReader { gr in
            let fullBarHeight = gr.size.height * 0.90
            let maxValue = data.map { $0.value }.max()!

            ZStack {
                RoundedRectangle(cornerRadius: 5.0)
                    .fill(Color(#colorLiteral(red: 0.8906477705, green: 0.9005050659, blue: 0.8208766097, alpha: 1)))

                VStack {
                    HStack(spacing:0) {
                        ForEach(data) { item in
                            BarView(
                                name: item.name,
                                value: item.value,
                                maxValue: maxValue,
                                fullBarHeight: Double(fullBarHeight))
                        }
                    }
                    .padding(4)
                }

            }
        }
    }
}

```

为  `BarView` 创建一个新的试图，该视图为每条数据创建一个条形图。它需要每一条数据的名称和值以及最大值和可用的条形高度。每个条形图都表示为圆角矩形，条形高度相对于最大条形高度设置。条形的颜色设置为纯蓝色。

```swift
struct BarView: View {
    var name: String
    var value: Double
    var maxValue: Double
    var fullBarHeight: Double

    var body: some View {
        let barHeight = (Double(fullBarHeight) / maxValue) * value
        VStack {
            Spacer()
            ZStack {
                VStack {
                    Spacer()
                    RoundedRectangle(cornerRadius:5.0)
                        .fill(Color.blue)
                        .frame(height: CGFloat(barHeight), alignment: .trailing)
                }

                VStack {
                    Spacer()
                    Text("\(value, specifier: "%.0F")")
                        .font(.footnote)
                        .foregroundColor(.white)
                        .fontWeight(.bold)
                }
            }
            Text(name)
        }
        .padding(.horizontal, 4)
    }
}

```

![](https://swdevnotes.com/images/swift/2021/0725/add-bars-to-chart.png)

## 屏幕旋转

条形图在使用样本数据时看起来不错。图表会调整到适合它所处的容器视图之中。同样的图表可以放到任何没有其他视图的新试图上，当设备旋转时，图标将会充满空间并调整大小。

```SWIFT
struct ChartView3: View {
    var body: some View {
        VStack() {

            BarChartView(
                title: "Daily step count", data: chartData)

            Spacer()
        }
        .padding()
    }
}

```

![](https://swdevnotes.com/images/swift/2021/0725/screen-rotation.png)

***手机旋转时显示的图表***

## 真实数据的条形图

给条形图使用真实世界的数据。[联合国儿童基金会数据集](https://data.unicef.org/resources/resource-type/datasets/)中五岁以下儿童死亡率最高的十个国家。

>   五岁以下儿童死亡率：
>
>   指从出生到五岁之间死亡的概率，每1000名活产婴儿

***2019年特定国家五岁以下儿童死亡率估计数***

| ISO Code | Country Name             | 2019  |
| -------- | ------------------------ | ----- |
| NGA      | Nigeria                  | 117.2 |
| SOM      | Somalia                  | 116.9 |
| TCD      | Chad                     | 113.7 |
| CAF      | Central African Republic | 110.0 |
| SLE      | Sierra Leone             | 109.2 |
| GIN      | Guinea                   | 98.8  |
| SSD      | South Sudan              | 96.2  |
| MLI      | Mali                     | 94.0  |
| BEN      | Benin                    | 90.2  |
| BFA      | Burkina Faso             | 87.5  |
| LSO      | Lesotho                  | 86.4  |

可以看出，国家名称比示例数据中一周中的几天使用多个数据名称要长的多。数据使用国家名称在条形图中绘制。

```SWIFT
struct ChartView4: View {
    let chartData: [DataItem] = [
        DataItem(name: "Nigeria", value: 117.2),
        DataItem(name: "Somalia", value: 116.9),
        DataItem(name: "Chad", value: 113.7),
        DataItem(name: "Central African Republic", value: 110.0),
        DataItem(name: "Sierra Leone", value: 109.2),
        DataItem(name: "Guinea", value:  98.8),
        DataItem(name: "South Sudan", value:  96.2),
        DataItem(name: "Mali", value:  94.0),
        DataItem(name: "Benin", value:  90.2),
        DataItem(name: "Burkina Faso", value:  87.5)
    ]

    var body: some View {
        VStack() {

            BarChartView(
                title: "Under Five Mortality Rates in 2019", data: chartData)
                .frame(width: 350, height: 500, alignment: .center)

            Text("Under-five mortality rate:")
            Text("is the probability of dying between birth and exactly 5 years of age, expressed per 1,000 live births.")

            Spacer()
        }
        .padding()
    }
}
```

这里对 `BarView` 做出了一些改动。条形图上的值使用叠加视图修改移到了条形图的顶部。这个值是偏移的，所以文本不会离条形图的顶部太近。数据名称的字体大小和字重也可以被设置。向国家名称那样较长的文本，显示出条形图下面的文本将条形图推到了线外。文本视图的宽度被限制在条形图宽度的范围内，而且条形图的标签文本会被截断，条形图的文本视图也被限制在条形宽度的范围内，并且文本可以被隐藏起来。

```SWIFT
struct BarView: View {
    var name: String
    var value: Double
    var maxValue: Double
    var fullBarHeight: Double

    var body: some View {
        GeometryReader { gr in
            let barHeight = (Double(fullBarHeight) / maxValue) * value
            let textWidth = gr.size.width * 0.80
            VStack {
                Spacer()
                RoundedRectangle(cornerRadius:5.0)
                    .fill(Color.blue)
                    .frame(height: CGFloat(barHeight), alignment: .trailing)
                    .overlay(
                        Text("\(value, specifier: "%.0F")")
                            .font(.footnote)
                            .foregroundColor(.white)
                            .fontWeight(.bold)
                            .frame(width: textWidth)
                            .offset(y:10)
                        ,
                        alignment: .top
                    )

                Text(name)
                    .font(.system(size: 11))
                    .fontWeight(.semibold)
                    .lineLimit(1)
                    .frame(width: textWidth)
            }
            .padding(.horizontal, 4)
        }
    }
}

```

![](https://swdevnotes.com/images/swift/2021/0725/highest-u5mr-country-names.png)

所有的国家名称都被截断了，所以将数据更代为使用国家码而不是国家名称。图标被设置为固定大小，视图被嵌入到 `ScrollView ` 中，以便在设备旋转时滚动。

```SWIFT
struct ChartView5: View {
    let chartData: [DataItem] = [
        DataItem(name: "NGA", value: 117.2),
        DataItem(name: "SOM", value: 116.9),
        DataItem(name: "TCD", value: 113.7),
        DataItem(name: "CAF", value: 110.0),
        DataItem(name: "SLE", value: 109.2),
        DataItem(name: "GIN", value:  98.8),
        DataItem(name: "SSD", value:  96.2),
        DataItem(name: "MLI", value:  94.0),
        DataItem(name: "BEN", value:  90.2),
        DataItem(name: "BFA", value:  87.5)
    ]

    var body: some View {
        ScrollView {
            VStack() {

                BarChartView(
                    title: "Countries with the highest Under Five Mortality Rates in 2019", data: chartData)
                    .frame(width: 350, height: 500, alignment: .center)

                Spacer().frame(height:20)

                VStack() {
                    Text("Under-five mortality rate:")
                        .font(.system(.title2, design:.rounded))
                        .fontWeight(.bold)
                    Text("is the probability of dying between birth and exactly 5 years of age, expressed per 1,000 live births.")
                        .font(.body)
                }
                .frame(width: 300, height: 130)
                .background(Color(#colorLiteral(red: 0.8906477705, green: 0.9005050659, blue: 0.8208766097, alpha: 1)))
                .cornerRadius(10)

                Spacer()
            }
            .padding()
        }
    }
}

```

![](https://swdevnotes.com/images/swift/2021/0725/highest-under-five-mortality-rates.png)

## 结语

在 SwiftUI 中组合矩形来创建条形图是比较容易的。SwiftUI 是一个很好的平台，用于创建视图和快速重构独立的子视图。在 SwiftUI 中构建条形图需要做一些工作，随着使用数据来试用条形图，可以确定更多的定制化。使用 `GeometryReader ` 可以创建适应更多可用环境的条形图。在这篇文章中，我们创建了一个简单的条形图，有数值，下面有标签，还有图表的标题，下一步就是分离出  x 轴和 y 轴。

> 来源：[How to create a Bar Chart in SwiftUI](https://swdevnotes.com/swift/2021/how-to-create-bar-chart-swiftui/)
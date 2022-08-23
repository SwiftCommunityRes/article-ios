## 介绍

早在 2020 年，我们就拥有了在 SwiftUI（LazyVGrid 和 LazyHGrid）中绘制网格的新视图控件。两年后，我们又获得了另一种在网格（Grid）中显示视图的视图控件。但是，这些新增功能非常不同，不仅在您使用它的方式上，而且在它内部的行为方式上。 2020 年的观点很懒惰。这些新人很热心。

`lazy grids`不会渲染甚至实例化屏幕外的视图。单元格视图仅在它们被滚动时创建，并且在它们滚动时停止计算。

这篇文章的主题 Eager Grids 正好相反。 SwiftUI 不在乎它们是在屏幕上还是在屏幕外。所有视图都被同等对待。这可能会出现大量单元的性能问题。然而，多少是一个很大的数字是一个不可能回答的问题。这将取决于您的单元格视图的复杂性。

所以如果`lazy grids`表现更好，这就引出了一个问题，我为什么要使用`Eager Grids`？事实是，`Eager Grids`比`lazy grids`更有优势，反之亦然。例如，`Eager Grids`支持列跨越，而`lazy grids`不支持。归根结底，性能并不是唯一需要考虑的因素。在本文中，我们将探索这些新网格，以便您在选择其中一个时做出明智的决定。

## 关于容器视图的一句话

在我们开始探索 Grid 视图之前，让我先谈谈容器视图。也就是说，接收视图构建器并以特定方式呈现其内容的视图（HStack、VStack、ZStack、Lazy*Grid、Group、List、ForEach 等）。请耐心等待，这将在以后有所帮助。

有两种类型的容器视图。我认为这些类型没有正式名称。我只会称它们为“有布局的容器”和“没有布局的容器”。用几个例子可以更好地解释这一点：

![eagergrids-group](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrids-group.png)

```swift
struct ContentView: View {
    var body: some View {
        
        HStack {
            Group {
                Text("Hello")
                Text("World")
                Image(systemName: "network")
            }
            .padding(10)
            .border(.red)
        }
    }
}
```

同样可以这么写:

```swift
struct ContentView: View {
    var body: some View {
        
        HStack {
            Text("Hello")
                .padding(10)
                .border(.red)
            
            Text("World")
                .padding(10)
                .border(.red)
            
            Image(systemName: "network")
                .padding(10)
                .border(.red)
        }
    }
}
```

从示例中可以看出，Group 修饰符分别应用于每个包含的视图。此外，Group 视图本身没有提供任何布局，也没有任何自己的几何图形。所有布局都由其父级执行：HStack。

但是，具有布局的容器（例如 HStack）上的修饰符应用于容器，该容器确实具有自己的几何形状：

![eagergrids-hstack](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrids-hstack.png)

```swift
struct ContentView: View {
    var body: some View {
        
        HStack {
            Text("Hello")
            Text("World")
            Image(systemName: "network")
        }
        .padding(10)
        .border(.red)
    }
}
```

您可能会问，当 Group 没有父级时会发生什么。这不是问题。当没有布局容器存在时，SwiftUI 会隐式使用 VStack。这就是为什么这也有效：

![eagergrids-nocontainer](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrids-nocontainer.png)

```swift
struct ContentView: View {
    var body: some View {        
        Text("Hello")
        Text("World")
        Image(systemName: "network")
    }
}
```

另一个没有布局的容器示例是 `ForEach`：

![eagergrids-foreach](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrids-foreach.png)

```swift
struct ContentView: View {
    var body: some View {
        
        HStack {
            ForEach(0..<5) { idx in
                Text("\(idx)")
            }
            .padding(10)
            .border(.blue)
        }
    }
}
```

这与网格有什么关系？我们将在下一节中找到答案。

## 我们的第一个网格

让我们建立我们的第一个网格。语法非常简单。您使用 Grid 容器视图，然后通过对 GridRow 容器内的单元格视图进行分组来定义其行。

![eagergrids-first-grid](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrids-first-grid.png)

```swift
struct ContentView: View {
    var body: some View {
        Grid {
            GridRow {
                Text("Cell #1")
                    .padding(20)
                    .border(.red)

                Text("Cell #2")
                    .padding(20)
                    .border(.red)
            }

            GridRow {
                Text("Cell #3")
                    .padding(20)
                    .border(.green)

                Text("Cell #4")
                    .padding(20)
                    .border(.green)
            }
        }
        .padding(10)
        .border(.blue)
    }
}
```

这就是我们谈论容器的地方。如果我告诉你 Grid 是一个带有布局的容器，但 GridRow 不是。这意味着我们可以重写我们的代码并获得相同的结果：

```swift
struct ContentView: View {
    var body: some View {
        Grid {
            GridRow {
                Text("Cell #1")

                Text("Cell #2")
            }
            .padding(20)
            .border(.red)

            GridRow {
                Text("Cell #3")

                Text("Cell #4")
            }
            .padding(20)
            .border(.green)
        }
        .padding(10)
        .border(.blue)
    }
}
```

请注意，并非所有行都具有相同数量的单元格。尽管这里的大多数示例都可以，但每一行可以包含任意数量的单元格。

## 探索网格选项

在以下部分中，我们将探讨不同的网格大小、对齐和跨越选项。但为了让事情变得更容易，我创建了一个名为 Grid Trainer 的小应用程序。该应用程序可让您以交互方式使用所有这些网格参数。当您更改网格时，该应用程序还将向您显示生成您创建的网格的代码。

整个应用程序位于一个 swift 文件中，因此只需几秒钟即可完成设置。只需创建一个新的 Xcode 项目，将 ContentView.swift 文件替换为此 gist 文件中的文件，就可以开始了。请注意，虽然我在设计应用程序时主要考虑了 macOS，但该应用程序在 iPad 上也能流畅运行。无需更改。

当您阅读以下部分时，最好运行 Grid Trainer 应用程序并测试您对网格的理解。试着看看你是否可以预测当你改变参数时网格会做什么。每次你得到你所期望的不同结果时，你都会学到一些关于网格的新东西。如果你得到你所期望的，你会重申你已经知道的。

### 空间

与 HStack 和 VStack 类似，Grid 容器具有用于间距的垂直和水平参数。如果未指定，则将使用系统默认值。

![eagergrids-spacing](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrids-spacing.png)

```swift
Grid(horizontalSpacing: 5.0, verticalSpacing: 15.0) {
    GridRow {
        Rectangle().fill(Color(white: 0.20).gradient)

        Rectangle().fill(Color(white: 0.40).gradient)

        Rectangle().fill(Color(white: 0.60).gradient)

        Rectangle().fill(Color(white: 0.80).gradient)

    }
    .frame(width: 50.0, height: 50.0)
    
    GridRow {
        Rectangle().fill(Color(white: 0.80).gradient)

        Rectangle().fill(Color(white: 0.60).gradient)

        Rectangle().fill(Color(white: 0.40).gradient)

        Rectangle().fill(Color(white: 0.20).gradient)
    }
    .frame(width: 50.0, height: 50.0)
}
```

### 列宽，行高

网格中的单元格是视图，视图会适应父级提供的大小。在这种情况下，父级是网格。通常，列与其中最宽的单元格一样宽。在下面的示例中，橙色列的宽度由第二行中最宽的单元格决定。身高也是如此。在示例中，第二行与行中最高的紫色单元格一样高。

![eagergrids-widths](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrids-widths.png)

### 未定义大小的单元

默认情况下，网格将为单元格提供尽可能多的空间。那么如果一个网格是由一个 Rectangle() 视图组成的，会发生什么呢？如您所知，没有框架修饰符的形状喜欢增长以填充父级提供的所有空间。在这种情况下，网格将增长以填充其父级提供的所有空间。

在下面的示例中，绿色单元格在其水平维度上不受限制，因此它使用了所有可用空间。网格尽可能地增长，绿色单元格填充空间。然而，蓝色单元格被框架修改器限制为 50.0 pt 宽度。虚线表示网格边界。

![eagergrids-parent](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrids-parent.png)

```swift
struct ContentView: View {
    let dash = StrokeStyle(lineWidth: 1.0, lineCap: .round, lineJoin: .miter, dash: [5, 5], dashPhase: 0)

    var body: some View {

        HStack(spacing: 0) {
            Circle().fill(.yellow).frame(width: 30, height: 30)
            
            Grid(horizontalSpacing: 0) {
                GridRow {
                    RoundedRectangle(cornerRadius: 15.0)
                        .fill(.green.gradient)
                        .frame(height: 50)
                    
                    RoundedRectangle(cornerRadius: 15.0)
                        .fill(.blue.gradient)
                        .frame(width: 50, height: 50)
                }
            }
            .overlay { Rectangle().stroke(style: dash) }

            Circle().fill(.yellow).frame(width: 30, height: 30)
        }
    }
}
```

到目前为止，没有什么太令人惊讶的。这与我们从使用 HStack 容器的第一天起就看到的行为相同。但是，Grids 在这里为我们提供了一个选择。我们可以让单元格避免让网格增长以获得额外的空间。例如，对于水平维度，单元格只会增长到与其列中最宽的单元格一样多的空间。这样的单元格在确定列宽方面没有任何作用。这是通过应用于相关单元格的 gridCellUnsizedAxes() 修饰符来完成的。它接收一个 Axis.Set 值。它可以是 .horizontal、.vertical 或两者的组合：[.horizontal, .vertical]。这告诉网格给定单元格选择不要求额外空间的维度。

如果您还没有，现在是开始使用 Grid Trainer 应用程序并挑战您迄今为止的知识的好时机。

在下面的示例中，红色单元格在水平轴上未调整大小，使其仅与绿色单元格一样大。即使父母提供更多，红细胞也不会接受。

![eagergrids-unsized](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrids-unsized.png)

```swift
Grid {
  GridRow {
    RoundedRectangle(cornerRadius: 5.0)
      .fill(.green.gradient)
      .frame(width: 160.0, height: 80.0)

    RoundedRectangle(cornerRadius: 5.0)
      .fill(.blue.gradient)
      .frame(width: 80.0, height: 80.0)
  }

  GridRow {
    RoundedRectangle(cornerRadius: 5.0)
      .fill(.red.gradient)
      .frame(height: 80.0)
      .gridCellUnsizedAxes(.horizontal)

    RoundedRectangle(cornerRadius: 5.0)
      .fill(.yellow.gradient)
      .frame(width: 80.0, height: 80.0)
  }
}
```

### 对齐路线

#### 网格对齐

当单元格的视图小于可用空间时，对齐方式将取决于几个参数。第一个要考虑的参数是 Grid(alignment: Alignment)。它影响网格中的所有单元格，除非被下一个参数之一覆盖。如果未指定，则默认为 .center。

![eagergrid-grid-alignment](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrid-grid-alignment.gif)

```swift
Grid(alignment: .topLeading) {
    GridRow {
        Rectangle().fill(.yellow.gradient)
            .frame(width: 50.0, height: 50.0)
        
        Rectangle().fill(.green.gradient)
            .frame(width: 100.0, height: 100.0)

    }
    
    GridRow {
        Rectangle().fill(.orange.gradient)
            .frame(width: 100.0, height: 100.0)

        Rectangle().fill(.red.gradient)
            .frame(width: 50.0, height: 50.0)
    }
}
```

#### 行垂直对齐

您还可以使用 GridRow(alignment: VerticalAlignment) 指定行对齐方式。请注意，在这种情况下，对齐方式只是垂直的。此行中的单元格将结合 Grid 参数和 GridRow 参数。行的垂直对齐将优先于对齐的网格垂直组件。在下面的示例中，具有 .topTrailing 值的网格与 .bottom 垂直行值相结合，会导致第二行中的单元格以 .bottomTrailing 对齐。其他行将使用网格对齐方式（即 .topTrailing）。



![eagergrid-row-alignment-1](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrid-row-alignment-1.png)

```swift
Grid(alignment: .topTrailing) {
    GridRow {
        Rectangle().fill(Color(white: 0.25).gradient)
            .frame(width: 120.0, height: 100.0)

        Rectangle().fill(Color(white: 0.50).gradient)
            .frame(width: 50.0, height: 50.0)

        Rectangle().fill(Color(white: 0.50).gradient)
            .frame(width: 120.0, height: 100.0)
    }
    
    GridRow(alignment: .bottom) {
        Rectangle().fill(Color(white: 0.25).gradient)
            .frame(width: 120.0, height: 100.0)

        Rectangle().fill(Color(white: 0.50).gradient)
            .frame(width: 50.0, height: 50.0)

        Rectangle().fill(Color(white: 0.50).gradient)
            .frame(width: 50.0, height: 50.0)
    }
    
    GridRow {
        Rectangle().fill(Color(white: 0.25).gradient)
            .frame(width: 120.0, height: 100.0)

        Rectangle().fill(Color(white: 0.50).gradient)
            .frame(width: 120.0, height: 100.0)

        Rectangle().fill(Color(white: 0.50).gradient)
            .frame(width: 50.0, height: 50.0)
    }
}
```

#### 列水平对齐

除了指定垂直行对齐方式外，您还可以指定列水平对齐方式。与行对齐的情况一样，该值将与行垂直值和网格的对齐值合并。您使用修饰符 gridColumnAlignment() 指示列的对齐方式

注意：文档非常清楚。 gridColumnAlignment 只能在每列一个单元格中使用。否则行为未定义。

在以下示例中，您可以看到所有对齐组合：

单元格 (1,1)：对齐顶部前导。 （网格对齐）
单元格 (1, 2)：对齐的 topTrailing。 （网格对齐+列对齐）
单元格（2,1）：对齐的底部前导（网格对齐+行对齐）
单元格 (2,2)：对齐的底部尾随（网格对齐 + 行对齐 + 列对齐）

![eagergrids-column-alignment](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrids-column-alignment.png)

```swift
struct ContentView: View {
    var body: some View {
        Grid(alignment: .topLeading, horizontalSpacing: 5.0, verticalSpacing: 5.0) {
            GridRow {
                CellView(color: .green, width: 80, height: 80)

                CellView(color: .yellow, width: 80, height: 80)
                    .gridColumnAlignment(.trailing)

                CellView(color: .orange, width: 80, height: 120)
            }
            
            GridRow(alignment: .bottom) {
                CellView(color: .green, width: 80, height: 80)

                CellView(color: .yellow, width: 80, height: 80)

                CellView(color: .orange, width: 80, height: 120)
            }
            
            GridRow {
                CellView(color: .green, width: 120, height: 80)

                CellView(color: .yellow, width: 120, height: 80)

                CellView(color: .orange, width: 80, height: 80)
            }
        }
    }
    
    struct CellView: View {
        let color: Color
        let width: CGFloat
        let height: CGFloat
        
        var body: some View {
            RoundedRectangle(cornerRadius: 5.0)
                .fill(color.gradient)
                .frame(width: width, height: height)
        }
    }

}
```

#### 单元格对齐

最后，您还可以使用 .gridCellAnchor(_: anchor: UnitPoint) 修饰符为单元格指定单独的对齐方式。此对齐方式将覆盖给定单元格的任何网格、列和行对齐方式。注意参数类型不是Alignment，而是UnitPoint。这意味着除了使用预定义的点 .topLeading、.center 等之外，您还可以创建任意点，例如 UnitPoint(x: 0.25, y: 0.75)：

![eagergrid-cell-alignment](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrid-cell-alignment.png)

```swift
Grid(alignment: .topTrailing) {
    GridRow {
        Rectangle().fill(.green.gradient)
            .frame(width: 120.0, height: 100.0)

        Rectangle().fill(.blue.gradient)
            .frame(width: 50.0, height: 50.0)
            .gridCellAnchor(UnitPoint(x: 0.25, y: 0.75))
    }
    
    GridRow {
        Rectangle().fill(.blue.gradient)
            .frame(width: 50.0, height: 50.0)

        Rectangle().fill(.green.gradient)
            .frame(width: 120.0, height: 100.0)

    }
}
```

#### 文本基线对齐

除了常见的对齐方式，请记住您还可以使用文本基线对齐方式。对于 Grid 和 GridRow：

<img src="https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrid-text-alignment.png" alt="eagergrid-text-alignment" />

```swift
Grid(alignment: .centerFirstTextBaseline) {
    GridRow {
        Text("Align")
        
        Rectangle()
            .fill(.green.gradient.opacity(0.7))
            .frame(width: 50, height: 50)
    }
}
.font(.system(size: 36))
```

#### 没有 GridRow 的行

如果 Grid 在 GridRow 容器之外有一个视图，则它被用作跨越所有列的单个单元格行。这种类型的单元格的常见用途是创建分隔符。例如，您可以使用 Divider() 视图，或者更复杂的视图，如下例所示。请注意，我们通常不希望分隔线使网格增长到最大值，因此我们使视图在水平轴上未调整大小。这将使分隔线与最宽的行一样宽，但不会更宽。

![eagergrid-divider](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrid-divider.png)

```swift
Grid(horizontalSpacing: 5.0, verticalSpacing: 5.0) {
    GridRow {
        RoundedRectangle(cornerRadius: 5.0).fill(.green.gradient)
        
        RoundedRectangle(cornerRadius: 5.0).fill(.purple.gradient)
        
        RoundedRectangle(cornerRadius: 5.0).fill(.blue.gradient)
    }
    .frame(width: 50.0, height: 50.0)

    Rectangle()
        .fill(LinearGradient(colors: [.gray, .clear, .gray], startPoint: .leading, endPoint: .trailing))
        .frame(height: 2.0)
        .gridCellUnsizedAxes(.horizontal)
    
    
    GridRow {
        RoundedRectangle(cornerRadius: 5.0).fill(.green.gradient)
        
        RoundedRectangle(cornerRadius: 5.0).fill(.purple.gradient)
        
        RoundedRectangle(cornerRadius: 5.0).fill(.blue.gradient)
    }
    .frame(width: 50.0, height: 50.0)
}
```

#### 列跨越

`Eager Grids`优于`Lazy Grids`的优点之一是所有单元几何形状始终是已知的。这使得有一个跨越多列的单元格成为可能。要将单元格配置为跨越，请使用 .gridCellColumns(_ count: Int)

![eagergrid-spanning](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrid-spanning.png)

```swift
Grid {
    GridRow {
        RoundedRectangle(cornerRadius: 5.0).fill(.green.gradient)
            .frame(width: 50.0, height: 50.0)
        
        RoundedRectangle(cornerRadius: 5.0).fill(.yellow.gradient)
            .frame(height: 50.0)
            .gridCellColumns(3)
            .gridCellUnsizedAxes(.horizontal)
        
        RoundedRectangle(cornerRadius: 5.0).fill(.purple.gradient)
            .frame(width: 50.0, height: 50.0)
    }
    
    GridRow {
        RoundedRectangle(cornerRadius: 5.0).fill(.green.gradient)
            .frame(width: 50.0, height: 50.0)
        
        RoundedRectangle(cornerRadius: 5.0).fill(.yellow.gradient)
            .frame(width: 50.0, height: 50.0)
        
        RoundedRectangle(cornerRadius: 5.0).fill(.orange.gradient)
            .frame(width: 50.0, height: 50.0)
        
        RoundedRectangle(cornerRadius: 5.0).fill(.red.gradient)
            .frame(width: 50.0, height: 50.0)
        
        RoundedRectangle(cornerRadius: 5.0).fill(.purple.gradient)
            .frame(width: 50.0, height: 50.0)
    }
}
```

#### 注意歧义

考虑以下示例。我们每行有 4 个单元格。除了第一行的第二个单元格和第二行的第三个单元格之外，每个单元格都是 50.0 pt 宽。这些将尽可能地增长（不扩大网格）。这两个单元格也分别跨越两列。

```swift
struct ContentView: View {
    var body: some View {
        Grid(horizontalSpacing: 20.0, verticalSpacing: 20.0) {
            GridRow {
                CellView(width: 50.0, color: .green)

                CellView(color: .purple)
                    .gridCellColumns(2)
                
                CellView(width: 50.0, color: .blue)
                
                CellView(width: 50.0, color: .yellow)
            }
            .gridCellUnsizedAxes([.horizontal, .vertical])

            GridRow {
                CellView(width: 50.0, color: .green)

                CellView(width: 50.0, color: .purple)
                
                CellView(color: .blue)
                    .gridCellColumns(2)

                CellView(width: 50.0, color: .yellow)
            }
            .gridCellUnsizedAxes([.horizontal, .vertical])
        }
    }
    
    struct CellView: View {
        var width: CGFloat? = nil
        let color: Color
        
        var body: some View {
            RoundedRectangle(cornerRadius: 5.0)
                .fill(color.gradient)
                .frame(width: width, height: 50.0)
        }
    }
}
```

你认为应该发生什么？如果仔细看，这是“先有鸡还是先有蛋的问题”。如果您查看第一行中的第二个单元格，它应该跨越到以下列。但是第二行中的以下列应该扩展到第三列。那是什么？我们可以满足一个条件或另一个条件，但不能同时满足这两个条件。这是因为第一行查看第二行以确定下一列，而第二行查看第一行以执行相同操作。 SwiftUI 需要以某种方式解决这个问题，如果你运行代码，你会得到以下结果：

![eagergrid-ambiguity1](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrid-ambiguity1.png)

为了打破平局，一个简单的解决方案是添加第三行：

```swift
GridRow {
    CellView(width: 50, color: .green)

    CellView(width: 50, color: .purple)

    CellView(width: 50, color: .blue)

    CellView(width: 50, color: .yellow)
}
```

第三排打破平局，这就是它的样子：

![eagergrid-ambiguity2](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrid-ambiguity2.png)

如果您不需要第三行，则无论如何都可以添加一个，但高度为零。不过，您可能仍需要处理间距。幸运的是，这并不常见，但我会提到以防您遇到这种情况。

## 蜂窝再访

在文章 [Impossible Grids](https://swiftui-lab.com/impossible-grids/) 中，我们是否探索了`Lazy Grid`，我写了一个示例，说明如何使用这些网格来呈现蜂窝中的单元格。创建这样的网格是测试网格可能的极限的好方法，所以我想我会重复这个练习，但这次使用`Eager Grids`。

此[gist file](https://gist.github.com/swiftui-lab/d38440a2281b2e069f81a94baa741073)中提供了完整的工作网格。如果需要图片来测试代码，可以访问 https://this-person-does-not-exist.com。您可以下载带有随机面孔的不存在的人的方形图片！它们是人工智能生成的。 😲 视频中使用的图片来自该网站。

### 从方形到六边形的步骤

我们必须从某个地方开始，所以我们将创建一个方形图像网格，然后逐渐添加代码将我们的简单网格转换为蜂窝。

到现在为止，您应该具备实现转换所需的所有知识。我将为您提供一个起点和您需要执行的一系列步骤，以便成功实现转换。但是，如果您没有时间，或者遇到困难，您可以检查上述 gist 文件中的代码。该代码有注释，指示它执行的每个步骤的位置。

请注意，单元格的翻转并不是练习的一部分，但我也将其包含在要点中。

以下视频显示了起点以及它如何变成蜂窝：

步骤#1：我们从方形图片网格开始。
步骤#2：六边形没有 1:1 的尺寸比。它的高度等于宽度 * cos(.pi/6)。如果您想知道原因，请查看 Impossible Grids，我在其中解释了原因。
步骤#3：用提供的六边形剪裁图像。
步骤#4：将偶数行和奇数行移动到相对的两侧。偏移量是六边形宽度的一半 + 网格水平间距。
第 5 步：行需要重叠，因此您需要将行高减少到四分之三 (3/4)。为什么是 3/4？，再次检查 Impossible Grids，我解释了原因。
第 6 步：要删除空白区域，请剪裁网格边框（或将其放在 ScrollView 中，它会为您进行剪裁）。
步骤#7：如果使垂直间距等于水平间距，则单元格将均匀分布。

### 初始点

为了让你开始，这里有一些代码。首先，我们需要一些数据：

```swift
struct Person {
    let name: String
    let image: String
    var color: Color = .accentColor
    var flipped: Bool = false
}

class DataModel: ObservableObject {
    static let people: [Person] = [
        Person(name: "Peter", image: "image-1"),
        Person(name: "Carlos", image: "image-2"),
        Person(name: "Jennifer", image: "image-3"),
        Person(name: "Paul", image: "image-4"),
        Person(name: "Charlotte", image: "image-5"),
        Person(name: "Thomas", image: "image-6"),
        Person(name: "Sophia", image: "image-7"),
        Person(name: "Isabella", image: "image-8"),
        Person(name: "Ivan", image: "image-9"),
        Person(name: "Laura", image: "image-10"),
        Person(name: "Scott", image: "image-11"),
        Person(name: "Henry", image: "image-12"),
        Person(name: "Laura", image: "image-13"),
        Person(name: "Abigail", image: "image-14"),
        Person(name: "James", image: "image-15"),
        Person(name: "Amelia", image: "image-16"),
    ]
    
    static let colors: [Color] = [.yellow, .orange, .red, .purple, .blue, .pink, .green, .indigo]

    @Published var rows: [[Person]] = DataModel.buildDemoCells()
    
    var columns: Int { rows.first?.count ?? 0 }    
    var colCount: CGFloat { CGFloat(columns) }
    var rowCount: CGFloat { CGFloat(rows.count) }

    static func buildDemoCells() -> [[Person]] {
        var array = [[Person]]()

        // Add 7 rows
        for r in 0..<7 {
            var a = [Person]()

            // Add 6 cells per row
            for c in 0..<6 {
                let idx = (r*6 + c)
                var person = people[idx % people.count]
                person.color = colors[idx % colors.count]
                a.append(person)
            }

            array.append(a)
        }

        return array
    }
}
```

### 您还需要一个六边形：

![eagergrids-hexagon](https://swiftui-lab.com/wp-content/uploads/2022/07/eagergrids-hexagon.png)

```swift
struct HexagonShape: Shape {
    func path(in rect: CGRect) -> Path {
        Path { path in
        
            let height = rect.height
            let width = rect.height * cos(.pi/6)
            
            let h = height / 4
            let w = width / 2
            
            let pt1 = CGPoint(x: rect.midX, y: rect.minY)
            let pt2 = CGPoint(x: rect.midX + w, y: h + rect.minY)
            let pt3 = CGPoint(x: rect.midX + w, y: h * 3 + rect.minY)
            let pt4 = CGPoint(x: rect.midX, y: rect.maxY)
            let pt5 = CGPoint(x: rect.midX - w, y: h * 3 + rect.minY)
            let pt6 = CGPoint(x: rect.midX - w, y: h + rect.minY)
            
            path.addLines([pt1, pt2, pt3, pt4, pt5, pt6])
            
            path.closeSubpath()
        }
    }
}
```

最后，你开始设置网格:

```swift
struct ContentView: View {
    @StateObject private var model = DataModel()
    
    private let cellWidth: CGFloat = 100
    private let cellHeight: CGFloat = 100
    
    var body: some View {
        VStack {
            Grid(alignment: .center, horizontalSpacing: 2, verticalSpacing: 2) {
                ForEach(model.rows.indices, id: \.self) { rowIdx in
                    GridRow {
                        ForEach(model.rows[rowIdx].indices, id: \.self) { personIdx in
                            
                            let person = model.rows[rowIdx][personIdx]
                            
                            Image(person.image)
                                .resizable()
                                .frame(width: cellWidth, height: cellHeight)
                        }
                    }
                }
            }
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(Color.white)
    }
}
```

## 概括

今年添加的 Grid 视图使用起来非常简单，并且添加到我们已经拥有的现有布局容器视图中。然而，今年还引入了一个新的布局协议，在将我们的视图放置在屏幕上时，它提供了更多的选择。我们将在以后的文章中对此进行探讨。同时，我希望您喜欢这篇文章和 Grid 教练应用程序。

> 译自：[Eager Grids with SwiftUI](https://swiftui-lab.com/eager-grids/)
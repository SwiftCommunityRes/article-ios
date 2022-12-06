## 简介

今年 `SwiftUI` 新增最好的功能之一必须是布局协议。它不但让我们参与到布局过程中，而且也给了我们一个很好的机会去更好的理解布局在 `SwiftUI` 中的作用。

早在2019年，我写了一篇文章[SwiftUI 中 frame 的表现](https://swiftui-lab.com/frame-behaviors/ "SwiftUI中frame的表现")，其中，我阐述了父视图和子视图如何协调形成最终视图效果。那里描述的许多情况需要通过观察不同测试的结果去猜测。整个过程就像是发现外星行星，天文学家发现太阳亮度微小的减少，然后推断出这一定是行星过境（[了解行星过境](https://exoplanets.nasa.gov/faq/31/whats-a-transit/ "了解行星过境")）。

现在，有了布局协议，就像用自己的眼睛在遥远的太阳系漫游，令人振奋。

创建一个基础布局并不难，只需要实现两个方法。尽管如此，我们仍然有很多选择去实现一个复杂的容器。我们将会探索常规布局案例之外的内容。有许多有趣的话题到目前为止我还没有在任何地方看到过解释，所以我将在这里介绍它们。然而，在深入这些领域之前，我们需要先打下扎实的基础。

由于涉及到许多内容，我将分成两个部分：

**Part 1 - 基础：**

* 什么是布局协议
* 视图层次结构的族动态
* 我们的第一个布局实现
* 容器对齐
* 自定义值：LayoutValueKey
* 默认间距
* 布局属性和 Spacer()
* 布局缓存
* 高明的伪装者
* 使用AnyLayout切换布局
* 结语

**Part 2 - 高级布局：**

* 开启有趣的旅程
* 自定义动画
* 双向自定义值
* 避免布局循环和崩溃
* 递归布局
* 布局组合
* 另一个组合案例：插入两个布局
* 使用绑定参数
* 一个有用的调试工具
* 最后的思考

如果你已经熟悉布局协议，你可能想直接跳到第二部分。这是可以的，尽管我仍然推荐你浏览第一部分，至少浅读一下。这将确保我们在开始探索第二部分中描述的更多高级特性时，我们在同一进度。

如果在阅读本文的任何时候，你认为布局协议不适合你（至少目前来说），我仍然建议你查看 Part2 的这一小节—一个有用的调试工具，这个工具可以帮助你使用 SwiftUI ，且不需要理解布局协议就可以使用。我将它放在第二部分结尾是有原因的，这个工具是使用本文的知识构建的。不过，你可以直接复制代码使用它。

## 什么是布局协议

采用布局协议类型的任务，是告诉 SwiftUI 如何放置一组视图，需要多少空间。这类型常常被作为视图容器，虽然布局协议是今年新推出的（至少公开来说），但是我们在第一天使用 SwiftUI 的时候就在使用了，当每次使用 `HStack` 或者 `VStack` 放置视图时都是如此。

请注意至少到现在，布局协议不能创建懒加载容器，比如 `LazyHStack` 或 `LazyVStack`。懒加载容器是指那些只在滚入屏幕时渲染，滚出到屏幕外就停止渲染的视图。

一个重要的知识点，`Layout` 类型不是视图 。例如，它们没有视图拥有的 body 属性。但是不用担心，目前为止你可以认为它们就是视图并且像视图一样使用它们。这个框架使用了漂亮的 `Swift` 语言技巧使你的布局代码在向 `SwiftUI` 中插入时产生一个透明视图 。我将在后面-高明的伪装者部分说明。

## 视图层次结构的族动态

在我们开始布局代码之前，让我们重新审视一下 `SwiftUI` 框架的核心。就像我在以前的文章 **SwiftUI 中 frame 的表现** 所描述的的那样，在布局过程中，父视图给子视图提供一个尺寸，但最终还是由子视图决定如何绘制自己。然后，它将此传达给父视图，以便采取相应的动作。有三个可能的情况，我们将专注讨论于横轴（宽度），但纵轴（高度）同理：

情况一：如果子视图需求小于提供的视图

在这个例子中考虑文本视图，提供了比需要绘制文字更多的空间

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-1.png)

```swift
struct ContentView: View {
    var body: some View {
        HStack(spacing: 0) {

            Rectangle().fill(.green)

            Text("Hello World!")

            Rectangle().fill(.green)

        }
        .padding(20)
    }
}
```

在这个例子中，屏幕宽度是 400pt。因此，文本提供 `HStack` 宽度的三分之一 ((400 – 40) / 3 = 120)。在这 120pt 中，文本只需要 74，并传达给父视图，父视图现在可以拿走多余的 46pt 给其他的子视图用。因为其他子视图是图形，所以它们可以接收给它们的一切东西。在这种情况下，120+46/2=143。

情况二：如果子视图完全接收提供的视图

图形就是视图中的一个例子，不管你提供了什么他都能接收。在上一个例子中，绿色矩形占据了提供的所有空间，但没有一个多余的像素。

情况三：如果子视图需求超出提供的视图

思考下面这个例子，图片视图特别严格（除非他们修改了 `resizable` 方法），它们需要多少空间就要占用多少空间，在下面这个例子中，图片是 300×300，这也是它们需要绘制自己需要的空间，然而，通过调用 `frame(width:100)` 子视图只得到了 100pt，父视图就没有办法只能听从子视图的做法吗？并非如此，子视图仍然会使用 300pt 绘制，但是父视图将会布局其他视图，就好像子视图只有 100pt 宽度一样。结果呢，我们将会有一个超出边界的子视图，但是周围的视图不会被图片额外使用的空间影响。在下面这个例子中，黑色边框展示的空间是提供给图片的。

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-2.png)

```swift
struct ContentView: View {
    var body: some View {
        HStack(spacing: 0) {

            Rectangle().fill(.yellow)

            Image("peach")
                .frame(width: 100)
                .border(.black, width: 3)
                .zIndex(1)

            Rectangle().fill(.yellow)

        }
        .padding(20)
    }
}
```

视图的行为方式有很多差异。例如，我们看见文本获取需求空间后如何处置多余的不需要的空间，然而，如果需求的空间大于提供，就可能会发生一些事情，具体取决于你如何配置你的视图。例如，可能会根据提供的尺寸截取文本，或者在提供的宽度内垂直的展示文本，如果你使用 `fixedSize` 修改甚至可能超出屏幕就像例子中的图片一样。请记住， `fixedSize` 告诉视图使用其理想尺寸，无论提供的是多少。

如果你想了解更多这些行为以及如何改变它们，请查看我以前的文章 **SwiftUI 中 frame 的表现**

## 我们的第一个布局实现

创建一个布局类型需要我们实现至少两个方法， `sizeThatFits` 和 `placeSubviews` 。这些方法接收一些新类型作为参数： `ProposedViewSize` 和 `LayoutSubview` 。在我们开始写方法之前，先看看这些参数长什么样：

### ProposedViewSize

 `ProposedViewSize` 被父视图用来告知子视图如何计算自己的尺寸。这是一个简单的类型，但很强大。它只是一对可选的 `CGFloat` ，用于建议宽度和高度。然而，正是我们如何解释这些值才使它们变得有趣。

这些属性可以有具体的值（例如35，74等），但当它们等于0.0 ，nil 或者 .infinity 时是有特殊的含义。

* 对于一个具体的宽度，例如 45，父视图提供的也是 45pt，这个视图应该由提供的宽度来决定自身的尺寸
* 对于宽度为 0.0，子视图应该响应为最小尺寸
* 对于宽度为  .infinity ,子视图应该响应为最大尺寸
* 对于 nil，父视图应该响应为理想尺寸

`ProposedViewSize` 也可以有一些预定义值：

```swift
ProposedViewSize.zero = ProposedViewSize(width: 0, height: 0)
ProposedViewSize.infinity = ProposedViewSize(width: .infinity, height: .infinity)
ProposedViewSize.unspecified = ProposedViewSize(width: nil, height: nil)
```

### LayoutSubview

`sizeTheFits` 和 `placeSubviews` 方法也接收一个 `Layout.Subviews` 参数，它是一个 `LayoutSubview` 元素的合集。每个视图都有一个，作为父视图的直接后代。尽管有这个名称，但它的类型不是视图，而是一个代理。我们可以查询这些代理去了解我们正在布局的各个视图的布局信息。例如，自 `SwiftUI` 推出以来，我们第一次可以直接查询到视图最小，理想和最大的尺寸，或者我们可以获得每个视图的布局优先级以及其他有趣的值。

### `sizeThatFits` 方法

```swift
func sizeThatFits(proposal: ProposedViewSize, subviews: Self.Subviews, cache: inout Self.Cache) -> CGSize
```

`SwiftUI` 将会调用  `sizeThatFits` 方法决定我们布局容器的尺寸，当我们写这个方法我们应该认为我们既是父视图又是子视图：当作为父视图时需要询问子视图的尺寸，当我们是子视图时，要基于我们子视图的回复告诉父视图需要的尺寸，

这个方法将会收到建议尺寸，一个子视图代理的合集和一个缓存。最后一个参数可能用以提高我们的布局和一些其他高级应用的性能，但现在我们不会使用它，我们会在后面一点再去看它。

当 `sizeThatFits` 方法在给定维度中（即宽度或高度）收到的建议尺寸为 nil 时，我们应该返回容器的理想尺寸。当收到的建议尺寸为0.0时，我们应该返回容器的最小尺寸。当收到的建议尺寸为 .infinity 时，我们应该返回容器的最大尺寸。

注意 `sizeThatFits` 可能通过不同提案多次调用来测试容器的灵活性，提案可以是上述每个维度案例的任意组合。例如，你可能会得到一个带有 `ProposedViewSize(width: 0.0, height: .infinity)`的调用。

在我们掌握了这些信息后，让我们开始第一个布局。我们通过创建一个基础的 `HStack` 开始。我们把它命名为 `SimpleHStack` 。为了比较两者，我们创建一个标准的 `HStack` （蓝色）视图放置在`SimpleHStack` （绿色）上方。在我们的第一次尝试中，我们将会实现 `sizeThatFits` ，但是同时我们将会使其他需要的方法（`placeSunviews`）为空。

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-3.png)

```swift
struct ContentView: View {
    var body: some View {
        VStack(spacing: 20) {
            
            HStack(spacing: 5)  { 
                contents() 
            }
            .border(.blue)
            
            SimpleHStack(spacing: 5) {
                contents() 
            }
            .border(.blue)

        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(.white)
    }
    
    @ViewBuilder func contents() -> some View {
        Image(systemName: "globe.americas.fill")
        
        Text("Hello, World!")

        Image(systemName: "globe.europe.africa.fill")
    }

}

struct SimpleHStack: Layout {
    let spacing: CGFloat
    
    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        let idealViewSizes = subviews.map { $0.sizeThatFits(.unspecified) }
        
        let spacing = spacing * CGFloat(subviews.count - 1)
        let width = spacing + idealViewSizes.reduce(0) { $0 + $1.width }
        let height = idealViewSizes.reduce(0) { max($0, $1.height) }
        
        return CGSize(width: width, height: height)
    }
    
    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) 
    {
        // ...
    }
}
```

你可以观察到，这两个图形的尺寸是一样的。然而，这是因为我们没有在 `placeSubviews` 方法中编写任何代码，所有的视图都放置在容器中间。如果你没有明确的放置位置，这就是容器的默认视图。

在我们的 `sizeThatFits` 方法中，我们首先要计算每个视图的所有理想尺寸。我们可以很容易的实现，因为子视图代理中有返回建议尺寸的方法。

一旦我们计算好所有理想尺寸，我们可以通过添加子视图宽度和视图间距来计算容器尺寸。从高度上来说，我们的视图将会和最高子视图一样高。

你或许已经察觉到了我们完全忽视了提供的尺寸，我们马上回到这里，现在，让我们实现 `placeSubviews` 。

### `placeSubviews` 方法

```swift
func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Self.Subviews, cache: inout Self.Cache)
```

在 `SwiftUI` 通过不同的提案值反复调用 `sizeThatFits` 来测试过容器视图后，终于可以调用 `placeSubviews` 。在这里我们的目标是遍历子视图，确定它们的位置并放置。

除了 `sizeThatFits` 收到同样的参数外，`placeSubviews` 还得到一个 `CGRect` 参数 。bounds rect 具有我们在 `sizeThatFits` 方法中要求的尺寸。通常，矩形的原点是（0，0），但是你不应该这样假设，如果我们正在组合布局，这个原点可能会有不同的值，我们将在后面看到。

放置视图很简单，这多亏了拥有放置方法的子视图代理。我们必须提供视图的坐标，锚点（默认为中心）和建议尺寸，以便子视图可以相应地绘制自己。

```swift
struct SimpleHStack: Layout {
    
    // ...
    
    
    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) 
    {
        var pt = CGPoint(x: bounds.minX, y: bounds.minY)
        
        for v in subviews {
            v.place(at: pt, anchor: .topLeading, proposal: .unspecified)
            
            pt.x += v.sizeThatFits(.unspecified).width + spacing
        }
    }
}
```

现在，还记得我之前提到的我们忽略了从父容器收到的建议了吗？这意味着 `SimpleHStack` 容器将会一直拥有一样的大小。不管提供什么，容器都会使用 `.unspecified` 计算尺寸和放置，意味着容器始终拥有理想的尺寸。在这个例子中容器的理想尺寸就是允许它以自己的理想尺寸放置所有子视图的尺寸。如果我们改变提供尺寸看看会发生什么，在这个动画中红框代表提供的宽度。

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-4.gif)

观察 `SimpleHStack` 是如何忽视提供的尺寸并且总是以理想尺寸绘制自己，该尺寸适合所有子视图的理想尺寸。

## 容器对齐

布局协议让我们也为容器定义对齐指南。注意，这表明容器是作为一个整体如何与其余视图对齐的。它对容器内的视图没有任何影响。

在下面这个例子中，我们让 `SimpleHStack` 对齐第二个视图，但前提是容器与头部对齐（如果把 `VStack` 的对齐方式改为尾部对齐，你将不会看到任何特殊的对齐方式）。

有红色边框的视图是 `SimpleHStack` ，黑色边框的视图是标准的 `HStack` 容器，绿色边框的表示封闭的 `VStack` 。

![](https://swiftui-lab.com/wp-content/uploads/2022/09/layout-alignment.png)

```swift
struct ContentView: View {
    var body: some View {
        VStack(alignment: .leading, spacing: 5) {
            HStack(spacing: 5) {
                contents()
            }
            .border(.black)

            SimpleHStack(spacing: 5) {
                contents()
            }
            .border(.red)
            
            HStack(spacing: 5) {
                contents()
            }
            .border(.black)
            
        }
        .background { Rectangle().stroke(.green) }
        .padding()
        .font(.largeTitle)
            
    }
    
    @ViewBuilder func contents() -> some View {
        Image(systemName: "globe")
            .imageScale(.large)
            .foregroundColor(.accentColor)
        Text("Hello, world!")
    }
}
```

```swift
struct SimpleHStack: Layout {
    
    // ...

    func explicitAlignment(of guide: HorizontalAlignment, in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGFloat? {
        if guide == .leading {
            return subviews[0].sizeThatFits(proposal).width + spacing
        } else {
            return nil
        }
    }
}
```

## 优先布局

当我们使用 `HStack` 时，我们知道所有视图都在平等的竞争宽度，除非它们有不同的布局优先级。所有的视图默认优先级都是0.0，但是，你可以通过调用 `layoutPriority()` 来修改布局优先级。

执行布局优先级是容器布局的责任，所以如果我们创建一个新布局，如果相关的话，我们需要添加一些逻辑去考虑布局优先级。我们如何做到这一点，这取决于我们自己。尽管有更好的方法（我们将在一分钟内解决它们），但你可以使用视图布局优先级的值赋予它们任何意义。例如，在上一个例子中，我们将会根据视图优先级的值从左往右放置视图。

为了实现效果，无需对子视图集合进行迭代，只需要简单的通过优先级排序。

```swift
truct SimpleHStack: Layout {
    
    // ...

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) 
    {
        var pt = CGPoint(x: bounds.minX, y: bounds.minY)
        
        for v in subviews.sorted(by: { $0.priority > $1.priority }) {
            v.place(at: pt, anchor: .topLeading, proposal: .unspecified)
            
            pt.x += v.sizeThatFits(.unspecified).width + spacing
        }
    }
}
```

在下面这个例子中，蓝色圆圈将会首先出现，因为它比起其他视图拥有较高的优先级

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-5.png)

```swift
SimpleHStack(spacing: 5) {
    Circle().fill(.yellow)
         .frame(width: 30, height: 30)
                        
    Circle().fill(.green)
        .frame(width: 30, height: 30)

    Circle().fill(.blue)
        .frame(width: 30, height: 30)
        .layoutPriority(1)
}
```

## LayoutValueKey

**自定义值：LayoutValueKey**

不建议将布局优先级用于优先级以外的内容，这可能使其他的用户不理解你的容器，甚至将来的你也不理解。幸运的是，我们有别的方法在视图中添加新值。这个值并不限制于 `CGFloat` ，它们可以拥有任何类型（后面我们将在别的例子中看到）。

我们将重写前面的例子，使用一个新值，我们把它称为 `PreferredPosition`。第一件事就是创建一个符合`LayoutValueKey` 的类型，我们只需要一个带有静态默认值的结构体。这个默认值用于没有指明具体值的时候。

```swift
struct PreferredPosition: LayoutValueKey {
    static let defaultValue: CGFloat = 0.0
}
```

这样，我们的视图就拥有了新的属性。为了设置这个值，我们需要用到 `layoutValue()` ，为了读取这个值，我们使用 `LayoutValueKey` 类型作为视图代理的下标：

```swift
SimpleHStack(spacing: 5) {
    Circle().fill(.yellow)
         .frame(width: 30, height: 30)
                        
    Circle().fill(.green)
        .frame(width: 30, height: 30)

    Circle().fill(.blue)
        .frame(width: 30, height: 30)
        .layoutValue(key: PreferredPosition.self, value: 1.0)
}
```

```swift
struct SimpleHStack: Layout {
    // ...

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) 
    {
        var pt = CGPoint(x: bounds.minX, y: bounds.minY)
        
        let sortedViews = subviews.sorted { v1, v2 in
            v1[PreferredPosition.self] > v2[PreferredPosition.self]
        }
        
        for v in sortedViews {
            v.place(at: pt, anchor: .topLeading, proposal: .unspecified)
            
            pt.x += v.sizeThatFits(.unspecified).width + spacing
        }
    }
}
```

这段代码不像第一段我们写的 `layoutPriority` 那样整洁，但是用这两个扩展很容易解决：

``` swift
extension View {
    func preferredPosition(_ order: CGFloat) -> some View {
        self.layoutValue(key: PreferredPosition.self, value: order)
    }
}

extension LayoutSubview {
    var preferredPosition: CGFloat {
        self[PreferredPosition.self]
    }
}
```

现在我们可以像这样重写：

```swift
SimpleHStack(spacing: 5) {
    Circle().fill(.yellow)
         .frame(width: 30, height: 30)
                        
    Circle().fill(.green)
        .frame(width: 30, height: 30)

    Circle().fill(.blue)
        .frame(width: 30, height: 30)
        .preferredPosition(1)
}
```

```swift
struct SimpleHStack: Layout {
    // ...

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) 
    {
        var pt = CGPoint(x: bounds.minX, y: bounds.minY)
        
        for v in subviews.sorted(by: { $0.preferredPosition > $1.preferredPosition }) {
            v.place(at: pt, anchor: .topLeading, proposal: .unspecified)
            
            pt.x += v.sizeThatFits(.unspecified).width + spacing
        }
    }

}
```

## 默认间距

到目前为止，我们在初始化布局的时候 `SimpleHStack` 使用的都是我们提供的间距值，然而，在你使用了 `HStack` 一阵子，你就会知道如果没有指明间距，视图将会根据不同的平台和内容提供默认的间距。一个视图可以拥有不同间距，如果旁边是文本视图和旁边是图像间距是不一样的。除此之外，每个边缘都会有自己的偏好。

所以我们应该如何用 `SimpleHStack` 让它们行为一致？我曾提到过子视图代理是布局知识的宝藏，而且它们不会让人失望。它们有可以查询它们空间偏好的方法。

```swift
struct SimpleHStack: Layout {
    
    var spacing: CGFloat? = nil
    
    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        let idealViewSizes = subviews.map { $0.sizeThatFits(.unspecified) }
        let accumulatedWidths = idealViewSizes.reduce(0) { $0 + $1.width }
        let maxHeight = idealViewSizes.reduce(0) { max($0, $1.height) }

        let spaces = computeSpaces(subviews: subviews)
        let accumulatedSpaces = spaces.reduce(0) { $0 + $1 }
        
        return CGSize(width: accumulatedSpaces + accumulatedWidths,
                      height: maxHeight)
    }
    
    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) 
    {
        var pt = CGPoint(x: bounds.minX, y: bounds.minY)
        let spaces = computeSpaces(subviews: subviews)

        for idx in subviews.indices {
            subviews[idx].place(at: pt, anchor: .topLeading, proposal: .unspecified)
            
            if idx < subviews.count - 1 {
                pt.x += subviews[idx].sizeThatFits(.unspecified).width + spaces[idx]
            }
        }
    }
    
    func computeSpaces(subviews: LayoutSubviews) -> [CGFloat] {
        if let spacing {
            return Array<CGFloat>(repeating: spacing, count: subviews.count - 1)
        } else {
            return subviews.indices.map { idx in
                guard idx < subviews.count - 1 else { return CGFloat(0) }
                
                return subviews[idx].spacing.distance(to: subviews[idx+1].spacing, along: .horizontal)
            }
        }

    }
}
```

请注意，除了使用空间偏好外，你还可以告诉系统容器视图的空间偏好。这样， `SwiftUI` 就会知道如何将其与周围的视图分开，为此，你需要实现布局方法 [spacing(subviews:cache:)](https://developer.apple.com/documentation/swiftui/layout/spacing(subviews:cache:)-86z2e)。

## 布局属性和 Spacer()

布局协议有一个你可以实现的名为 `layoutProperties` 的静态属性。根据文档， `LayoutProperties` 包含布局容器的特定布局属性。在写这篇文章时，只定义了一个属性：`stackOrientation`。

```swift
struct MyLayout: Layout {
    
    static var layoutProperties: LayoutProperties {
        var properties = LayoutProperties()
        
        properties.stackOrientation = .vertical
        
        return properties
    }

    // ...
}
```

`stackOrientation` 告诉是像 `Spacer` 这样的视图是否应该在横轴或纵轴上展开。例如，如果你检查 `Spacer` 视图代理的最小，理想和最大尺寸，这就是它在不同容器返回的结果，每个容器都有不同的`stackOrientation` ：

| stackOrientation |  minimum  |   ideal   |        maximum        |
| :--------------: | :-------: | :-------: | :-------------------: |
|   .horizontal    | 8.0 × 0.0 | 8.0 × 0.0 |    .infinity × 0.0    |
|    .vertical     | 0.0 × 8.0 | 0.0 × 8.0 |    0.0 × .infinity    |
|   .none or nil   | 8.0 × 8.0 | 8.0 × 8.0 | .infinity × .infinity |

## 布局缓存

布局缓存是常被用来提高我们布局性能的一种方式。然而，它还有别的用途。只需要把它看作是一个存储数据的地方，我们需要在 `sizeThatFits` 和  `placeSubviews` 调用中持久保存。首先想到的是提高性能，但是，它对于和其他子视图布局共享信息也是非常有用的。当我们讲到组合布局的例子时，我们将对此进行探讨，但让我们从了解如何使用缓存提高性能开始。

在 `SwiftUI` 的布局过程中会多次调用 `sizeThatFits` 和 `placeSubviews` 方法。这个框架测试我们的容器的灵活性，以确定整体视图层级结构的最终布局。为了提高布局容器性能， `SwiftUI` 让我们实现了一个缓存， 只有当容器内的至少一个视图改变时才更新缓存。因为 `sizeThatFits` 和 `placeSubviews` 都可以为单个视图更改时多次调用，所以保留不需要为每次调用而重新计算的数据缓存是有意义的。

使用缓存不是必须的。事实上，很多时候你不需要。无论如何，在没有缓存的情况下编写我们的布局更简单一点，当我们以后需要时再添加。 `SwiftUI` 已经做了一些缓存。例如，从子视图代理获得的值会自动存储在缓存中。相同的参数的反复调用将会使用缓存结果。在 [makeCache(subviews:)](https://developer.apple.com/documentation/swiftui/layout/makecache(subviews:)-23agy "makeCache(subviews:)") 文档页面，有一个很好的讨论关于你可能想要实现自己的缓存的原因。

同时也要注意， `sizeThatFits` 和 `placeSubviews` 中的缓存参数有一个是 `inout` 参数，这意味着你也可以用这个函数更新缓存存储，我们将会看到它在 `RecursiveWheel` 例子中特别有帮助。

例如，这里是使用更新缓存的 `SimpleHStack` 。下面是我们需要做的：

* 创建一个将包含缓存数据的类型。在本例中，我把它叫做 `CacheData` ，它将会计算视图间的最大高度和空间。
* 实现 makeCache(subviews:) 创建缓存。
* 可选的实现 [updateCache(subviews:)](https://developer.apple.com/documentation/swiftui/layout/updatecache(_:subviews:)-9hkj9 "updateCache(subviews:)")，这个方法会在检测到更改时调用。它提供了默认实现，基本上通过调用 `makeCache` 重新创建缓存。 
* 记住要更新 `sizeThatFits` 和 `placeSubviews`中的缓存参数类型。

```swift
struct SimpleHStack: Layout {
    struct CacheData {
        var maxHeight: CGFloat
        var spaces: [CGFloat]
    }
    
    var spacing: CGFloat? = nil
    
    func makeCache(subviews: Subviews) -> CacheData {
        return CacheData(maxHeight: computeMaxHeight(subviews: subviews),
                         spaces: computeSpaces(subviews: subviews))
    }
    
    func updateCache(_ cache: inout CacheData, subviews: Subviews) {
        cache.maxHeight = computeMaxHeight(subviews: subviews)
        cache.spaces = computeSpaces(subviews: subviews)
    }
    
    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout CacheData) -> CGSize {
        let idealViewSizes = subviews.map { $0.sizeThatFits(.unspecified) }
        let accumulatedWidths = idealViewSizes.reduce(0) { $0 + $1.width }
        let accumulatedSpaces = cache.spaces.reduce(0) { $0 + $1 }
        
        return CGSize(width: accumulatedSpaces + accumulatedWidths,
                      height: cache.maxHeight)
    }
    
    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout CacheData) {
        var pt = CGPoint(x: bounds.minX, y: bounds.minY)

        for idx in subviews.indices {
            subviews[idx].place(at: pt, anchor: .topLeading, proposal: .unspecified)
            
            if idx < subviews.count - 1 {
                pt.x += subviews[idx].sizeThatFits(.unspecified).width + cache.spaces[idx]
            }
        }
    }
    
    func computeSpaces(subviews: LayoutSubviews) -> [CGFloat] {
        if let spacing {
            return Array<CGFloat>(repeating: spacing, count: subviews.count - 1)
        } else {
            return subviews.indices.map { idx in
                guard idx < subviews.count - 1 else { return CGFloat(0) }
                
                return subviews[idx].spacing.distance(to: subviews[idx+1].spacing, along: .horizontal)
            }
        }
    }
    
    func computeMaxHeight(subviews: LayoutSubviews) -> CGFloat {
        return subviews.map { $0.sizeThatFits(.unspecified) }.reduce(0) { max($0, $1.height) }
    }
}
```

如果我们每次调用其中一个布局函数都打印出一条信息，我们将会获得的下面的结果。如你所见，缓存将会计算两次，但是其他方法将会被调用25次！

```
makeCache called <<<<<<<<
sizeThatFits called
sizeThatFits called
sizeThatFits called
sizeThatFits called
placeSubiews called
placeSubiews called
updateCache called <<<<<<<<
sizeThatFits called
sizeThatFits called
sizeThatFits called
sizeThatFits called
placeSubiews called
placeSubiews called
sizeThatFits called
sizeThatFits called
placeSubiews called
sizeThatFits called
placeSubiews called
placeSubiews called
sizeThatFits called
placeSubiews called
placeSubiews called
sizeThatFits called
sizeThatFits called
sizeThatFits called
placeSubiews called
```

注意除了使用缓存参数提高性能，它们也有其他用途。我们将会在第二部分的 `RecursiveWheel`例子中再谈论。

## 高明的伪装者

正如我已经提到的，布局协议没有采用视图协议。那么我们为什么一直在 `ViewBuilder`中使用布局容器，就好像它们是视图一样？事实证明，当你用代码放置你的布局时，会有一个系统函数调用来产生视图。那这个函数叫什么呢？你可能已经猜到了：

```swift
func callAsFunction<V>(@ViewBuilder _ content: () -> V) -> some View where V : View
```

由于语言的增加（在 [SE-0253](https://github.com/apple/swift-evolution/blob/master/proposals/0253-callable.md "SE-0253")中有描述和解释），被命名为 `callAsFunction` 的方法是特殊的。当我们使用一个类型实例时，这些方法会像一个函数一样被调用。在这种情况下，我们可能会感到困惑，因为我们似乎只是在初始化类型，而实际上，我们做的更多。我们初始化类型然后调用 `callAsFunction`，因为 `callAsFunction`的返回值是一个视图，所以我们可以把它放到我们的 `SwiftUI` 代码中。

```swift
SimpleHStack(spacing: 10).callAsFunction({
    Text("Hello World!")
})

// Thanks to SE-0253 we can abbreviate it by removing the .callAsFunction
SimpleHStack(spacing: 10)({
    Text("Hello World!")
})

// And thanks to trailing closures, we end up with:
SimpleHStack(spacing: 10) {
    Text("Hello World!")
}
```

如果布局没有初始化参数，代码甚至可以更简单：

```swift
SimpleHStack().callAsFunction({
    Text("Hello World!")
})

// Thanks to SE-0253 we can abbreviate it by removing the .callAsFunction
SimpleHStack()({
    Text("Hello World!")
})

// And thanks to single trailing closures, we end up with:
SimpleHStack {
    Text("Hello World!")
}
```

所以你明白了，布局类型并不是视图，但是当你在 `SwiftUI` 中使用它们的时候它们就会产生一个视图。这个技巧（`callAsFunction`）还可以切换到不同布局，同时保持视图的标识，就像接下来的部分描述的那样。

## 使用 AnyLayout 切换布局

布局容器的另一个有趣的地方，我们可以修改容器的布局， `SwiftUI` 会友好地用动画处理两者的切换。不需要额外的代码！那是因为视图会识别标识并且维护， `SwiftUI` 将这个行为认为是视图的改变，而不是两个单独的视图。

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-6.gif)

```swift
struct ContentView: View {
    @State var isVertical = false
    
    var body: some View {
        let layout = isVertical ? AnyLayout(VStackLayout(spacing: 5)) : AnyLayout(HStackLayout(spacing: 10))
        
        layout {
            Group {
                Image(systemName: "globe")
                
                Text("Hello World!")
            }
            .font(.largeTitle)
        }
        
        Button("Toggle Stack") {
            withAnimation(.easeInOut(duration: 1.0)) {                
                isVertical.toggle()
            }
        }
    }
}
```

三元运算符（条件？结果1:结果2）要求两个表达式返回同一类型。`AnyLayout` 在这里发挥了作用。

注意：如果你观看过 [2022 WWDC Layout session](https://developer.apple.com/videos/play/wwdc2022/10056/ "2022 WWDC Layout session")，你或许看见过苹果工程师使用的例子，但使用的是 `VStack` 代替 `VStackLayout` 和 `HStack` 代替 `HStackLayout` 。那已经过时了。在 beta3 过后， `HStack` 和 `VStack` 不再采用布局协议，并且他们添加了 `VStackLayout` 和 `HStackLayout` 布局（分别由`HStack` 和 `VStack` 使用），他们还添加了 `ZStackLayout` 和 `GridLayout`。

## 结语

如果我们停下来考虑每一种可能的情况，编写布局容器可能会让我们举步维艰。有的视图使用尽可能多的空间，有的视图会尽量适应，还有的将会使用的更少，等等。当然还有布局优先级，当多个视图需要竞争同一个空间会变得更加艰难。然而，这项任务可能并不像看起来艰巨。我们可能会使用自己的布局，并且可能会提前知道我们的容器会有什么类型的视图。例如，如果你打算只用方形图片或者文本视图来使用自己的容器，或者你知道你的容器会有具体尺寸，或者你确定你所有的视图都拥有一样的优先级，等等。这些信息都可以大大的简化任务。即使你不能有这种奢望来做这种假设，它也可能是开始编码的好地方，让你的布局在一些情况下工作，然后开始为更复杂的情况添加代码。

在本文的第二部分，我们将开始探索一些有趣的话题，比如自定义动画，双向自定义值，递归布局或布局组合。我还会介绍一个非常有用的调试工具，即使你没有创建自己的布局也可以使用。

> 来自：[The SwiftUI Layout Protocol – Part 1](https://swiftui-lab.com/layout-protocol-part-1/)
# SwiftUI布局协议-Part2

## 前言

在 Part 1 我们探索了布局协议的基础知识，为理解布局是如何工作的打下了坚实的基础。现在，是时候深入研究那些更少提及的功能了，以及如何使用它们来为我们带来便利。

**Part 1 - 基础：**

  * 什么是布局协议
  * 视图层次结构的族动态
  * 我们的第一个布局实现
  * 容器对齐
  * 自定义值：`LayoutValueKey`
  * 默认间距
  * 布局属性和 `Spacer()`
  * 布局缓存
  * 高明的伪装者
  * 使用 `AnyLayout` 切换布局
  * 结语

**Part 2 - 高级布局：**

  * 前言
  * 自定义动画
  * 双向自定义值
  * 避免布局循环和崩溃
  * 递归布局
  * 布局组合
  * 插入两个布局
  * 使用绑定参数
  * 一个有用的调试工具
  * 最后的思考

## 自定义动画

让我们从写一个圆形布局的视图容器开始吧。我们将它叫做 `WheelLayout`:

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-2-1.png)

```swift
struct ContentView: View {
    let colors: [Color] = [.yellow, .orange, .red, .pink, .purple, .blue, .cyan, .green]
    
    var body: some View {
        WheelLayout(radius: 130.0, rotation: .zero) {
            ForEach(0..<8) { idx in
                RoundedRectangle(cornerRadius: 8)
                    .fill(colors[idx%colors.count].opacity(0.7))
                    .frame(width: 70, height: 70)
                    .overlay { Text("\(idx+1)") }
            }
        }
    }
}

struct WheelLayout: Layout {
    var radius: CGFloat
    var rotation: Angle
    
    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        
        let maxSize = subviews.map { $0.sizeThatFits(proposal) }.reduce(CGSize.zero) {
            
            return CGSize(width: max($0.width, $1.width), height: max($0.height, $1.height))
            
        }
        
        return CGSize(width: (maxSize.width / 2 + radius) * 2,
                      height: (maxSize.height / 2 + radius) * 2)
    }
    
    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) 
    {
        let angleStep = (Angle.degrees(360).radians / Double(subviews.count))

        for (index, subview) in subviews.enumerated() {
            let angle = angleStep * CGFloat(index) + rotation.radians
            
            // Find a vector with an appropriate size and rotation.
            var point = CGPoint(x: 0, y: -radius).applying(CGAffineTransform(rotationAngle: angle))
            
            // Shift the vector to the middle of the region.
            point.x += bounds.midX
            point.y += bounds.midY
            
            // Place the subview.
            subview.place(at: point, anchor: .center, proposal: .unspecified)
        }
    }
}
```

当布局发生改变时，`SwfitUI` 提供了内置动画支持。所以如果我们将轮子的旋转值更改为90度，我们将会看见它是如何逐渐的移动到新的位置上：

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-2-wheel-1.gif)

```swift
WheelLayout(radius: radius, rotation: angle) {
// ...
}

Button("Rotate") {
    withAnimation(.easeInOut(duration: 2.0)) {
        angle = (angle == .zero ? .degrees(90) : .zero)
    }
}
```

这非常好我本可以在此结束动画部分。但是，你已经知道我们的博客不满足于肤浅的表面，所以让我们深层次的看看到底发生了什么。

当我们改变角度时，`SwiftUI` 会计算好每个视图最初和最终的位置，然后在动画期间内修改它们的位置，从A点到B点成一条直线。 起初它似乎没有这样做，但是检查下面这个动画，集中注意观察单个视图，看看它们是如何都跟随直虚线移动的？

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-2-wheel-2.gif)

你有想过如果动画的角度是从0到360会发生什么吗？给你一分钟... 对！...什么都不会发生。开始的位置和结束的位置是一样的，因此就 `SwiftUI` 而言，没有动画。

如果这就是你要找的东西，那就太好了，但由于我们将视图围绕一个圆圈放置，如果视图沿着那个假想的圆圈移动不是更有意义吗？好吧，事实证明，这样做非常容易！

我们问题的答案是很幸运的，这个布局协议采用 `Animatable` 协议！如果你不知道或者忘记这是什么，我建议你查看 [SwiftUI 布局协议 - Part 1](https://mp.weixin.qq.com/s/SfqdGs8TbEjxISgrsy_yIg) 的 `Animating Shape Paths` 部分 。

简单的说，通过添加 `animatableData` 属性到我们的布局，我们要求 `SwiftUI` 动画的每一帧重新计算布局。但是，在每个布局传递中，角度都会收到一个内插值。现在 `SwiftUI` 不会为我们插入位置。相反，它会插入角度值。我们的布局代码将会完成剩下的工作。

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-2-wheel-3.gif)

```swift
struct Wheel: Layout {
    // ...

    var animatableData: CGFloat {
        get { rotation.radians }
        set { rotation = .radians(newValue) }
    }

    // ...
}
```

添加一个 `animatableData` 属性足够让我们的视图正确的跟随圆圈。但是，既然我们已经到了这步。。。为什么我们不让半径也可以动画呢？

```swift
var animatableData: AnimatablePair<CGFloat, CGFloat> {
    get { AnimatablePair(rotation.radians, radius) }
    set {
        rotation = Angle.radians(newValue.first)
        radius = newValue.second
    }
}
```

## 双向自定义值

在文章的第一部分我们了解到如何使用 `LayoutValues` 将信息附加到视图，以便它们的代理可以在 `placeSubviews` 和 `sizeThatFits` 方法中暴露这些信息。我们的想法是信息从视图流向布局，一会儿将看见这一点是如何被逆转。

本节所解释的想法应谨慎使用，以避免布局循环和  `CPU` 峰值。在下一部分我将会解释原因和如何避免它。但是不用担心，这并不复杂，你只需要遵循一些准则。

让我们回到轮子的这个例子，假设我们想要视图旋转起来，让它们指向中心。

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-2-2.png)

布局协议只能决定视图位置和它们的建议尺寸，但是不能应用样式、旋转或者其他的效果。如果我们想要这些效果，那么布局应该有一种传达回视图的方式。这时候布局值就变得重要起来，到目前为止，我们已经使用它们传递信息给布局，但只要加上一点创意，我们就可以反向使用它们。

我之前提到过的 `LayoutValues` 并不局限于传递 `CGFloats` ，你可以将它用于任何事情，包括`Binding`，在这个例子中，我们将使用 `Binding<Angle>`:

```swift
struct Rotation: LayoutValueKey {
    static let defaultValue: Binding<Angle>? = nil
}
```

注意：我称它为双向自定义值，因为信息是可以双向流动的，但是，这不是 `SwiftUI` 的官方术语，只是为了更清晰的解释这个想法的术语。

在布局的 `placeSubview` 方法中，我们设置每个子视图的角度：

```swift
struct WheelLayout: Layout {

    // ...

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) 
    {
        let angleStep = (Angle.degrees(360).radians / Double(subviews.count))

        for (index, subview) in subviews.enumerated() {
            let angle = angleStep * CGFloat(index) + rotation.radians
            
            // ...
            
            DispatchQueue.main.async {
                subview[Rotation.self]?.wrappedValue = .radians(angle)
            }
        }
    }
}
```

回到我们的视图，我们可以读取值，并用它来旋转视图：

```swift
struct ContentView: View {

    // ...

    @State var rotations: [Angle] = Array<Angle>(repeating: .zero, count: 16)
    
    var body: some View {
        
        WheelLayout(radius: radius, rotation: angle) {
            ForEach(0..<16) { idx in
                RoundedRectangle(cornerRadius: 8)
                    .fill(colors[idx%colors.count].opacity(0.7))
                    .frame(width: 70, height: 70)
                    .overlay { Text("\(idx+1)") }
                    .rotationEffect(rotations[idx])
                    .layoutValue(key: Rotation.self, value: $rotations[idx])
            }
        }
 
        // ...
}
```

这段代码将确保所有的视图都指向圆心，但是我们可以更优雅一点。我提供的解决方案需要设置一个旋转数组，将它们作为布局值然后使用这些值旋转视图。如果我们可以向布局用户隐藏这种复杂性那不是很好吗？这里就是重写之后的。

首先我们创建一个封装视图 `WheelComponent`:

```swift
struct WheelComponent<V: View>: View {
    @ViewBuilder let content: () -> V
    @State private var rotation: Angle = .zero
    
    var body: some View {
        content()
            .rotationEffect(rotation)
            .layoutValue(key: Rotation.self, value: $rotation)
    }
}
```

然后我们摆脱旋转数组（我们不再需要了！）并将每个视图包装在 `WheelComponent`视图中。

```swift
WheelLayout(radius: radius, rotation: angle) {
    ForEach(0..<16) { idx in

        WheelComponent {
            RoundedRectangle(cornerRadius: 8)
                .fill(colors[idx%colors.count].opacity(0.7))
                .frame(width: 70, height: 70)
                .overlay { Text("\(idx+1)") }
        }

    }
}
```

就是这样。用户使用容器只需要记住将视图封装在 `WheelComponent`里面。他们不需要担心布局值，绑定，角度等等。当然，不在封装里的视图不会受到任何影响，视图不会旋转指向中心。

我们还可以添加一个改进，那就是视图旋转的动画。仔细观察并比较下面三个轮子：一个不旋转。另外两个旋转指向中心，但是一个不使用动画而另一个使用。

<video autoplay="" controls="" loop="" src="https://swiftui-lab.com/wp-content/uploads/2022/08/wheel-comparison.mp4" style="display: inline-block; vertical-align: baseline; width: 965px;"></video>

## 避免布局循环和崩溃

众所周知我们在布局期间不能更新视图状态。这会导致不可预测的结果，很可能会使 `CPU` 达到峰值。在此之前我们看到过这种情况，即闭包在布局期间运行时，也许当时不是太明显。但是现在，这是毫无疑问的。 `sizeThatFits` 和 `placeSubviews` 是布局过程中的一部分。因此当我们使用上一部分中描述的"欺骗"的技巧，我们必须使用 `DispatchQueue` 用队列更新。就像上面的例子一样：

```swift
DispatchQueue.main.async {
    subview[Rotation.self]?.wrappedValue = .radians(angle)
}
```

使用双向自定义值还有另一个潜在的问题，那就是你的视图必须在不影响别的布局的前提下使用该值，否则的话你将会陷入布局循环。

例如，如果用 `placeSubviews` 设置去更改视图颜色，那就是安全的。在案例中，可能看起来旋转会影响布局，但其实不是这样的，当你旋转视图，它的周围从来没产生影响，边界仍然保持不变。如果你设置了偏移，或者其他的变换矩阵，也会发生同样的事情。但无论如何，我建议你监测 `CPU` 来发现布局中其他潜在的问题。如果 `CPU` 开始飙升，或许可以在 `placeSubviews` 中添加一条打印语句查看它是否无休止的调用。注意动画也会使 `CPU` 增长。如果你想测试你的容器是否循环，不要在动画时查看 `CPU` 。

注意这不是新问题。过去我们在使用 `GeometryReader` 获取视图尺寸并传递值到父视图的时候遇到过这个问题，然后父视图使用该信息去改变视图，即使用 `GeometryReader` 去再一次改变，然后我们就陷入了布局循环。这是个老问题，我在 `SwiftUI` 刚发布的时候就写过此类问题，在 [Safely Updating The View State ](https://swiftui-lab.com/state-changes/ "Safely Updating The View State ") 一文中可以查看更多信息。

我还想再提一下潜在的崩溃。这与双向自定义值无关。这是你在写任何布局都必须要考虑的。我们提到 `SwiftUI` 可能会多次调用 `sizeThatFits` 去测试视图的灵活性。在这些调用中，你返回的值应该是合理的。例如，下面的代码会崩溃：

```swift
struct ContentView: View {
    var body: some View {
        CrashLayout {
            Text("Hello, World!")
        }
    }
}

struct CrashLayout: Layout {
    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        if proposal.width == 0 {
            return CGSize(width: CGFloat.infinity, height: CGFloat.infinity)
        } else if proposal.width == .infinity {
            return CGSize(width: 0, height: 0)
        }
        
        return CGSize(width: 0, height: 0)
    }
    
    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) 
    {
    }
}
```

这里 `sizeThatFits` 返回了 .infinity 作为最小尺寸， .zero 作为最大尺寸。这是不合理的，最小尺寸不可能大于最大尺寸！

## 递归布局

在下面这个例子中我们将探索递归布局。我们将会把之前的 `WheelLayout` 视图转变为 `RecursiveWheel`.我们的新布局将会在圆圈里放置12个视图。里面的12个视图将会按比例缩小到内圈中，直到它们不会再有别的视图。视图的缩放和旋转要再一次使用双向自定义值实现。

在这个例子中在容器中一共有44个视图，所以我们的新容器将会分别以12，12，12和8为一圈。

注意本案例中如何使用缓存与子视图通信。这是可以实现的，因为缓存是一个 `inout` 参数，我们可以在 `placeSubviews` 中更新。

<video autoplay="" controls="" loop="" src="https://swiftui-lab.com/wp-content/uploads/2022/08/recursive-wheel.mp4"></video>

`placeSubviews` 方法遍历并放置12个子视图：

```swift
for (index, subview) in subviews[0..<12].enumerated() {
  // ...
}
```

然后递归调用`placeSubviews` 但仅限于剩下的视图，如此直到没有别的视图。

```swift
placeSubviews(in: bounds,
              proposal: proposal,
              subviews: subviews[12..<subviews.count],
              cache: &cache)
```

## 布局组合

在上一个例子中我们使用了相同的布局递归。但是，我们也可以组合一些不同布局到容器中。在下一个例子中我们将会把前三个视图水平的放置在视图顶部，后三个水平的放置在底部。剩下的视图将会在中间，垂直排列。

![](https://swiftui-lab.com/wp-content/uploads/2022/08/layout-composed.gif)

我们不需要编写垂直或者水平的间距逻辑代码，因为 `SwiftUI` 已经有这样的布局了：`HStackLayout` 和 `VStackLayout`。

只是有点小问题，很轻易就可以修复。由于某些原因，系统布局私下实现了 `sizeThatFits` 和 `placeSubviews` 。这意味着我们无法调用它们。但是，类型消除布局确实暴露它的所有方法，所以不要这样做：

```swift
HStackLayout(spacing: 0).sizeThatFits(...) // not possible
```

我们可以：

```swift
AnyLayout(HStackLayout(spacing: 0)).sizeThatFits(...) // it is possible!
```

此外，在与其他视图布局工作的时候，我们就相当于 `SwiftUI` 的角色。子布局的任何缓存创建和更新都属于我们的责任，幸运的是，这都很容易处理。我们只需要添加子布局缓存到我们自己的缓存里。

```swift
struct ComposedLayout: Layout {
    private let hStack = AnyLayout(HStackLayout(spacing: 0))
    private let vStack = AnyLayout(VStackLayout(spacing: 0))

    struct Caches {
        var topCache: AnyLayout.Cache
        var centerCache: AnyLayout.Cache
        var bottomCache: AnyLayout.Cache
    }

    func makeCache(subviews: Subviews) -> Caches {
        Caches(topCache: hStack.makeCache(subviews: topViews(subviews: subviews)),
               centerCache: vStack.makeCache(subviews: centerViews(subviews: subviews)),
               bottomCache: hStack.makeCache(subviews: bottomViews(subviews: subviews)))
    }

    // ...
}
```

## 插入两个布局

另一个组合案例：插入两个布局

下一个例子将会创建一个以轮子，或者波浪形式显示视图的布局。当然它还提供了一个从 0.0 到1.0 的 `pct` 参数，当 pct == 0.0，视图将会展示轮子，当pct == 1.0，视图将会展示 `sin`波动。中间的数值将会穿插在两者的位置之中。

在我们创建组合布局之前，让我先来介绍一下 `WaveLayout`，这个布局有好几个参数来让你改变正弦波动的幅度，频率和角度。

<video autoplay="" controls="" loop="" src="https://swiftui-lab.com/wp-content/uploads/2022/09/wave-layout.mp4" style="display: inline-block; vertical-align: baseline; width: 965px; border-radius: 9px;"></video>

`InterpolatedLayout` 将会计算两个布局（波动和轮子）的尺寸和位置然后它将插入这些值以进行最终定位。注意。在 `placeSubviews` 方法中，如果子视图被多次定位，最后一次调用`place()`  才会产生影响。

使用以下公式计算插值：

```swift
(wheelValue * pct) + (waveValue * (1-pct))
```

我们需要一种方法让  `WaveLayout` 和 `WheelLayout` 将每一个视图位置和旋转返回给 `InterpolatedLayout` 。我们通过缓存做到了这一点。我们再一次看到了缓存不是性能提升的唯一用途。

我们也需要  `WaveLayout` 和 `WheelLayout` 去检测它们是否被 `InterpolatedLayout` 使用，以便它们可以相应的更新缓存。这些视图可以轻易的检测到这种情况，这要归功于独立的缓存值，如果缓存是由 `InterpolatedLayout`  创建的，则该值仅为 `false`。

## 使用绑定参数

今年 `SwfitUI Lounges` 出现了一个有趣的问题，询问是否可能使用新的布局协议去创建一个层次树，用线连接。挑战的不是视图树结构，而是我们如何画连接线。

![](https://swiftui-lab.com/wp-content/uploads/2022/09/tree-layout-1024x397.png)

还有其它方法可以实现它，例如，使用 [Canvas](https://swiftui-lab.com/swiftui-animations-part5/ "Canvas") ，但是我们这里都是关于布局协议的，让我们来看看可以如何解决连接线的问题。

我们现在都知道，这根线不可能被布局绘制出来。那我们需要的是一种让布局告诉视图如何绘制线条的方法。初步想法可以（在这个问题上苹果的工程师是[这么建议的](http://swiftui-lab.com/digital-lounges-2022#layout-9 "建议")) 使用布局值。这正是我们在上一个例子中做的事情，双向自定义值。但是，仔细思考之后，还有一种更简单的方式。

相比于使用布局值去分别通知树的每个节点的最终位置，使用布局代码创建整个路径来的更简单一点。然后，我们只需要将路径返回给负责展示的视图。通过添加绑定布局参数很容易完成。

```swift
struct TreeLayout {

    @Binding var linesPath: Path

    // ...
}
```

在完成放置视图以后，我们知道了位置并使用它们的坐标去创建路径。再次注意，我们必须非常小心的避免布局循环。我发现更新路径会产生一个循环，即使该路径被绘制为不影响布局的背景视图也是如此，所以为了避免这种循环，我们要确保路径发生改变，然后才更新绑定，这样就可以成功的打破循环。

```swift
let newPath = ...

if newPath.description != linesPath.description {

    DispatchQueue.main.async {
        linesPath = newPath
    }

}
```

这个挑战的另一个有趣的部分是告诉布局这些视图如何分层连接。在本例中，我创建了两个 `UUID` 布局值，一个标识视图，另一个作为父视图的 `ID`。

```swift
struct ContentView: View {
    @State var path: Path = Path()
    
    var body: some View {
        let dash = StrokeStyle(lineWidth: 2, dash: [3, 3], dashPhase: 0)

        TreeLayout(linesPath: $path) {
            ForEach(tree.flattenNodes) { node in
                Text(node.name)
                    .padding(20)
                    .background {
                        RoundedRectangle(cornerRadius: 15)
                            .fill(node.color.gradient)
                            .shadow(radius: 3.0)
                    }
                    .node(node.id, parentId: node.parentId)
            }
        }
        .background {
            // Connecting lines
            path.stroke(.gray, style: dash)
        }
    }
}

extension View {
    func node(_ id: UUID, parentId: UUID?) -> some View {
        self
            .layoutValue(key: NodeId.self, value: id)
            .layoutValue(key: ParentNodeId.self, value: parentId)
    }
}
```

当我们使用这些代码的时候需要注意一些事项。这里应该只有一个父节点是 nil 的节点（根结点），你应该小心的避免循环引用（例如：两个节点互为父节点）。

同时也要注意，这里有一个好的选择，即放置到具有垂直和水平的滚动 `ScrollView` 中。

注意这是基本实现，仅用于说明如何实现。还有许多潜在的优化，但制作树布局所需的关键元素都在这里。

## 一个有用的调试工具

回到当 `SwiftUI` 刚发布的时候，我尽力搞清楚布局是如何工作的，我希望我有一个像我今天要介绍的这种工具 。直到现在，它都是最好的工具，用来添加围绕视图的边框观察视图边缘。那是我们最好的盟友。

使用边框依然是很好的调试工具，但我们可以添加一个新的工具。感谢新的布局协议，我创建了一个修饰器，它在尝试理解为什么视图不像您认为的那样工作的时候非常有用，修饰器在这儿：

```swift
func showSizes(_ proposals: [MeasureLayout.SizeRequest] = [.minimum, .ideal, .maximum]) -> some View
```

你可以在任何视图使用它，一个覆盖层将会浮动在视图的头部角落，显示一组给定的建议尺寸。如果你未制定建议，最小，理想和最大尺寸都将被覆盖。

```swift
MyView()
    .showSizes()
```

一些使用示例：

```swift
showSizes() // minimum, ideal and maximum

showSizes([.current, .ideal]) // the current size of the view and the ideal size

showSizes([.minimum, .maximum]) // the minimum and maximum

showSizes([.proposal(size: ProposedViewSize(width: 30, height: .infinity))]) // a specific proposal
```

更多的例子：

![](https://swiftui-lab.com/wp-content/uploads/2022/09/layout-debugging.png)

```swift
ScrollView {
    Text("Hello world!")
}
.showSizes([.current, .maximum])

Rectangle()
    .fill(.yellow)
    .showSizes()

Text("Hello world")
    .showSizes()

Image("clouds")
    .showSizes()

Image("clouds")
    .resizable()
    .aspectRatio(contentMode: .fit)
    .showSizes([.minimum, .ideal, .maximum, .current])
```

突然间一切都变得有意义了。例如：检查一下使用和不使用 `resizable()`的图像尺寸。终于能看到数字是不是有一种奇怪的满足感？

## 总结

即使你不打算写你自己的布局容器，明白它是如何工作也会帮助你理解布局在 `SwiftUI` 的一般工作方式。

就个人而言，深入布局协议让我对 `HStack` 或  `VStack` 等容器编写代码的团队有了新的认识。我经常认为这些视图是理所当然的，并将它们视为简单而不复杂的容器，好吧，尝试编写自己的版本，在各种情况下复制一个 `HStack` ，多种类型的视图和布局优先级竞争同一个空间。。。这是一个不错的挑战！

> 来自：[The SwiftUI Layout Protocol – Part 2](https://swiftui-lab.com/layout-protocol-part-2/)
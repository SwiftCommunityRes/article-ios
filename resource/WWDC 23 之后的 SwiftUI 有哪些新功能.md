## 前言

WWDC 23 已经到来，SwiftUI 框架中有很多改变和新增的功能。在本文中将主要介绍 SwiftUI 中**数据流**、**动画**、**ScrollView**、**搜索**、**新手势**等功能的新变化。

## 数据流

Swift 5.9 引入了**宏功能**，成为 SwiftUI 数据流的核心。SwiftUI 不再使用 `Combine`，而是使用新的 `Observation` 框架。Observation 框架为我们提供了 `Observable` 协议，必须使用它来允许 SwiftUI 订阅更改并更新视图。

```Swift
@Observable
final class Store {
    var products: [String] = []
    var favorites: [String] = []
    
    func fetch() async {
        try? await Task.sleep(nanoseconds: 1_000_000_000)
        // load products
        products = [
            "Product 1",
            "Product 2"
        ]
    }
}
```

不需要在代码中遵循 Observable 协议。相反，可以使用 `@Observable` 宏来标记你的类型，它会自动为符合 Observable 协议。也不再需要 `@Published` 属性包装器，因为 SwiftUI 视图会自动跟踪任何可观察类型的可用属性的更改。

```Swift
struct ProductsView: View {
    @State private var store = Store()
    
    var body: some View {
        List(store.products, id: \.self) { product in
            Text(verbatim: product)
        }
        .task {
            if store.products.isEmpty {
                await store.fetch()
            }
        }
    }
}
```

以前，有一系列的属性包装器，如 `State`、`StateObject`、`ObservedObject` 和 `EnvironmentObject`，你应该了解何时以及为何使用它们。

现在，状态管理变得更加简单。对于值类型（如字符串和整数）和符合 Observable 协议的引用类型，只需使用 State 属性包装器。

```Swift
struct FavoriteProductsView: View {
    let store: Store
    
    var body: some View {
        List(store.favorites, id: \.self) { product in
            Text(verbatim: product)
        }
    }
}
```

在上面的示例中，有一个接受 Store 类型的视图。在之前的 SwiftUI 框架版本中，应该使用 `@ObservedObject` 属性包装器来订阅更改。现在不需要了，因为 SwiftUI 视图会自动跟踪符合 Observable 协议的类型的更改。

```Swift
struct EnvironmentViewExample: View {
    @Environment(Store.self) private var store
    
    var body: some View {
        Button("Fetch") {
            Task {
                await store.fetch()
            }
        }
    }
}

struct ProductsView: View {
    @State private var store = Store()
    
    var body: some View {
        List(store.products, id: \.self) { product in
            Text(verbatim: product)
        }
        .task {
            if store.products.isEmpty {
                await store.fetch()
            }
        }
        .toolbar {
            NavigationLink {
                EnvironmentViewExample()
            } label: {
                Text(verbatim: "Environment")
            }
        }
        .environment(store)
    }
}
```

还可以使用 Environment 属性包装器与 environment 视图修饰符配对，将可观察类型放入 SwiftUI 环境中。不需要使用 `@EnvironmentObject` 属性包装器或 environmentObject 视图修饰符。同样的 Environment 属性包装器现在适用于可观察类型。

```Swift
struct BindanbleViewExample: View {
    @Bindable var store: Store
    
    var body: some View {
        List($store.products, id: \.self) { $product in
            TextField(text: $product) {
                Text(verbatim: product)
            }
        }
    }
}
```

每当需要从可观察类型中提取绑定时，可以使用新的 Bindable 属性包装器。

## 动画

动画始终是 SwiftUI 框架中最重要的部分。在 SwiftUI 中轻松实现任何动画，但之前的框架版本缺少一些现在具有的功能。

```Swift
struct AnimationExample: View {
    @State private var value = false
    
    var body: some View {
        Text(verbatim: "Hello")
            .scaleEffect(value ? 2 : 1)
            .onTapGesture {
                withAnimation {
                    value.toggle()
                } completion: {
                    print("Animation have finished")
                }
            }
    }
}
```

如上例所示，我们有了新版本的 `withAnimation` 函数，允许提供动画完成处理程序。这是一个很好的补充，现在您可以构建阶段性动画。

```Swift
enum Phase: CaseIterable {
    case start
    case loading
    case finish
    
    var offset: CGFloat {
        // Calculate offset for the particular phase
        switch self {
        case start: 100.0
        case loading: 0.0
        case finish: 50.0
        }
    }
}

struct PhasedAnimationExample: View {
    @State private var value = false
    
    var body: some View {
        PhaseAnimator(Phase.allCases, trigger: value) { phase in
            LoadingView()
                .offset(x: phase.offset)
        } animation: { phase in
            switch phase {
            case .start: .easeIn(duration: 0.3)
            case .loading: .easeInOut(duration: 0.5)
            case .finish: .easeOut(duration: 0.1)
            }
        }
    }
}
```

SwiftUI 框架引入了新的 `PhaseAnimator` 视图，它遍历阶段序列，允许为每个阶段提供不同的动画，并在阶段更改时更新内容。还有 `KeyframeAnimator` 视图，可以使用关键帧来实现动画。

## ScrollView

今年 ScrollView 有了很多优秀的新增功能。首先，可以使用 `scrollPosition` 视图修饰符来观察内容偏移量。

```Swift
struct ContentView: View {
    @State private var scrollPosition: Int? = 0
    
    var body: some View {
        ScrollView {
            Button("Scroll") {
                scrollPosition = 80
            }
            
            ForEach(1..<100, id: \.self) { number in
                Text(verbatim: number.formatted())
            }
            .scrollTargetLayout()
        }
        .scrollPosition(id: $scrollPosition)
    }
}
```

如上例所示，使用 `scrollPosition` 视图修饰符将内容偏移量绑定到一个状态属性上。每当用户滚动视图时，它会通过设置第一个可见视图的标识来更新绑定。还可以通过编程方式滚动到任何视图，但是，应该使用 `scrollTargetLayout` 视图修饰符来告诉 SwiftUI 框架在哪里查找标识以更新绑定。

```Swift
struct ContentView: View {
    var body: some View {
        ScrollView {
            ForEach(1..<100, id: \.self) { number in
                Text(verbatim: number.formatted())
            }
            .scrollTargetLayout()
        }
        .scrollTargetBehavior(.paging)
    }
}
```

可以通过使用 `scrollTargetBehavior` 视图修饰符来更改滚动行为。它允许在滚动视图中启用分页。

## 搜索

与搜索相关的视图修饰符也有一些很好的新增功能。例如，可以通过编程方式聚焦到搜索字段。

```Swift
struct ProductsView: View {
    @State private var store = Store()
    @State private var query = ""
    @State private var scope: Scope = .default
    
    var body: some View {
        List(store.products, id: \.self) { product in
            Text(verbatim: product)
        }
        .task {
            if store.products.isEmpty {
                await store.fetch()
            }
        }
        .searchable(text: $query, isPresented: .constant(true), prompt: "Query")
        .searchScopes($scope, activation: .onTextEntry) {
            Text(verbatim: scope.rawValue)
        }
    }
}
```

如上例所示，可以使用可搜索视图修饰符的 `isPresented` 参数来显示/隐藏搜索字段。还可以使用 `searchScopes` 视图修饰符的 `activation` 参数来定义范围的可见性逻辑。

## 新手势

新增的 `RotateGesture` 和 `MagnifyGesture` 使我们能够跟踪视图的旋转和放大。

```Swift
struct RotateGestureView: View {
    @State private var angle = Angle(degrees: 0.0)

    var rotation: some Gesture {
        RotateGesture()
            .onChanged { value in
                angle = value.rotation
            }
    }

    var body: some View {
        Rectangle()
            .frame(width: 200, height: 200, alignment: .center)
            .rotationEffect(angle)
            .gesture(rotation)
    }
}
```

## 新增的小功能

增加了全新的 `ContentUnavailableView` 类型，当需要显示空视图时可以使用它。示例如下：

```Swift
struct ProductsView: View {
    @State private var store = Store()
    
    var body: some View {
        List(store.products, id: \.self) { product in
            Text(verbatim: product)
        }
        .background {
            if store.products.isEmpty {
                ContentUnavailableView("Products list is empty", systemImage: "list.dash")
            }
        }
        .task {
            if store.products.isEmpty {
                await store.fetch()
            }
        }
    }
}
```

还有新增了新的视图修饰符，允许调整列表中的间距。可以使用 `listRowSpacing` 和 `listSectionSpacing` 视图修饰符来设置列表中所需的间距。`EnvironmentValues` 结构体包含了一系列与最新平台更新相关的新属性，例如 `isActivityFullscreen` 和 `showsWidgetContainerBackground`。Swift Charts 也具有可滚动和可动画的功能。

```Swift
#Preview {
    ContentView()
}
```

还有一个新的 Preview 宏，可以让我们轻松地为 UIKit 和 SwiftUI 构建预览，只需几行代码。

## 总结

SwiftUI 框架中有许多小的新增功能，我们将会继续分享。希望能帮到你。

特别感谢 Swift社区 编辑部的每一位编辑，感谢大家的辛苦付出，为 Swift社区 提供优质内容，为 Swift 语言的发展贡献自己的力量。
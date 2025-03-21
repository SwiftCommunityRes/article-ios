## 前言

WWDC 24 已经到来，我们有很多内容要讨论。每年，SwiftUI 都会通过引入更多功能来赶上 UIKit。今年也不例外。让我们深入了解 SwiftUI 框架引入的新功能。

我首先要提到的主要变化是 App、Scene 和 View 协议的 `@MainActor` 隔离。这可能会破坏你的代码，所以请记住这一点。

## 视图集合

SwiftUI 为 Group 和 ForEach 视图引入了新的重载，允许我们创建自定义容器，如 List 或 TabView。

```swift
struct AppStoreView<Content: View>: View {
    @ViewBuilder var content: Content
    
    var body: some View {
        VStack {
            Group(subviewsOf: content) { subviews in
                HStack {
                    if !subviews.isEmpty {
                        subviews[0]
                    }
                    
                    if subviews.count > 1 {
                        subviews[1]
                    }
                }
                
                if subviews.count > 2 {
                    VStack {
                        subviews[2...]
                    }
                }
            }
        }
    }
}
```

如上例所示，我们使用带有新初始化器的 Group 视图，允许我们访问通过 `@ViewBuilder` 闭包传递的内容视图的子视图。SwiftUI 引入了新的 `Subview` 和 `SubviewsCollection` 类型，提供了对真实视图的代理访问。

## 新的标签栏体验

使用新的 Tab 类型，SwiftUI 提供了新的可定制标签栏体验，带有流畅过渡到侧边栏。

```swift
enum Destination: Hashable {
    case home
    case search
    case settings
    case trends
}

struct RootView: View {
    @State private var selection: Destination = .home
    
    var body: some View {
        TabView {
            Tab("home", systemImage: "home", value: .home) {
                HomeView()
            }
            
            Tab("search", systemImage: "search", value: .search) {
                SearchView()
            }
            
            TabSection("Other") {
                Tab("trends", systemImage: "trends", value: .trends) {
                    TrendsView()
                }
                Tab("settings", systemImage: "settings", value: .settings) {
                    SettingsView()
                }
            }
            .tabViewStyle(.sidebarAdaptable)
        }
    }
}
```

如上例所示，我们使用新的 Tab 类型来定义标签。我们还在 `TabSection` 实例上使用 `tabViewStyle` 视图修饰符，将特定的标签部分分组并移动到侧边栏。

## 英雄动画

SwiftUI 引入了 `matchedTransitionSource` 和 `navigationTransition`，我们可以在任何 `NavigationLink` 实例中配对使用。

```swift
struct HeroAnimationView: View {
    @Namespace var hero
    
    var body: some View {
        NavigationStack {
            NavigationLink {
                DetailView()
                    .navigationTransition(.zoom(sourceID: "myId", in: hero))
            } label: {
                ThumbnailView()
            }
            .matchedTransitionSource(id: "myId", in: hero)
        }
    }
}
```

这使我们能够在 NavigationStack 内从一个视图导航到另一个视图时，使用相同的标识符和命名空间创建平滑的过渡。

## 滚动位置

新的 ScrollPosition 类型与 scrollPosition 视图修饰符配对，允许我们读取 ScrollView 实例的精确位置。我们还可以使用它编程地滚动到滚动内容的特定点。

```swift
struct ScrollPositionExample: View {
    @State private var position: ScrollPosition = .init(point: .zero)
    
    var body: some View {
        ScrollView {
            ForEach(1..<1000) { item in
                Text(item.formatted())
            }
            
            Button("jump to top") {
                position = ScrollPosition(point: .zero)
            }
        }
        .scrollPosition($position)
    }
}
```

## Entry 宏

新的 Entry 宏允许我们快速引入环境值、聚焦值、容器值等，无需样板代码。让我们看看在 Entry 宏之前我们如何定义环境值。

```swift
struct ItemsPerPageKey: EnvironmentKey {
    static var defaultValue: Int = 10
}

extension EnvironmentValues {
    var itemsPerPage: Int {
        get { self[ItemsPerPageKey.self] }
        set { self[ItemsPerPageKey.self] = newValue }
    }
}
```

现在，我们可以通过使用 Entry 宏来简化代码。

```swift
extension EnvironmentValues {
    @Entry var itemsPerPage: Int = 10
}
```

## 预览

新的 Previewable 宏允许我们在预览中引入状态，而无需将其包装到额外的包装视图中。

```swift
#Preview("toggle") {
    @Previewable @State var toggled = true
    return Toggle("Loud Noises", isOn: $toggled)
}
```

## 其他

SwiftUI 框架的下一版本包括许多新 API，如窗口推送、TextField 和 TextEditor 视图中的文本选择观察、搜索焦点监控、自定义文本渲染、新的 MeshGradient 类型等等，我无法在一篇文章中涵盖所有内容。

## 总结

在 WWDC 24 上，SwiftUI 再次通过引入更多新功能来提升其成熟度，以赶上 UIKit。今年的主要变化包括 @MainActor 隔离、视图集合的新重载、新的可定制标签栏体验、英雄动画、滚动位置的新功能以及新的 Entry 和 Previewable 宏。这些改进使开发者能够创建更灵活和高效的用户界面。SwiftUI还引入了许多新的API，如窗口推送、文本选择观察、搜索焦点监控等，使开发更加便捷和强大。
# SwiftUI 中的自定义导航

默认情况下，SwiftUI提供的各种导航API在很大程度上是以用户直接输入为中心的——也就是说，导航是在系统响应例如按钮的点击和标签切换等事件时由系统本身处理的。

然而，有时我们可能想更直接地控制应用程序的导航执行方式，尽管SwiftUI在这方面仍然不如UIKit或AppKit灵活，但它确实提供了相当多的方法，让我们在构建的视图中执行完全自定义的导航。

### 切换标签（tabs）

让我们先来看看我们如何能控制当前在`TabView`中显示的标签。通常情况下，当用户手动点击每个标签栏中的一个项目时，标签就会被切换，但是通过在一个给定的`TabView`中注入一个选择(`selection`)绑定，我们可以观察并控制当前显示的标签。在这里，我们要做的就是在两个标签之间切换，这两个标签是用整数`0`和`1`标记的:

```swift
struct RootView: View {
    @State private var activeTabIndex = 0

    var body: some View {
        TabView(selection: $activeTabIndex) {
            Button("Switch to tab B") {
                activeTabIndex = 1
            }
            .tag(0)
            .tabItem { Label("Tab A", systemImage: "a.circle") }

            Button("Switch to tab A") {
                activeTabIndex = 0
            }
            .tag(1)
            .tabItem { Label("Tab B", systemImage: "b.circle") }
        }
    }
}
```

但真正好的地方是，在识别和切换标签时，我们并不仅仅局限于使用整数。相反，我们可以自由地使用任何`Hashable`值来表示每个标签——例如通过使用一个枚举，其中包含我们想要显示的每个标签的情况。然后我们可以将这部分状态封装在一个`ObservableObject`中，这样我们就可以很容易地注入到我们的视图层次环境中:

```swift
enum Tab {
    case home
    case search
    case settings
}

class TabController: ObservableObject {
    @Published var activeTab = Tab.home

    func open(_ tab: Tab) {
        activeTab = tab
    }
}
```

有了上述内容，我们现在可以用新的`Tab`类型来标记`TabView`中的每个视图，如果我们再把`TabController`注入到视图层次结构的环境中，那么其中的任何视图都可以随时切换显示的Tab。

```swift
struct RootView: View {
    @StateObject private var tabController = TabController()

    var body: some View {
        TabView(selection: $tabController.activeTab) {
            HomeView()
                .tag(Tab.home)
                .tabItem { Label("Home", systemImage: "house") }

            SearchView()
                .tag(Tab.search)
                .tabItem { Label("Search", systemImage: "magnifyingglass") }

            SettingsView()
                .tag(Tab.settings)
                .tabItem { Label("Settings", systemImage: "gearshape") }
        }
        .environmentObject(tabController)
    }
}
```

例如，现在我们的`HomeView`可以使用一个完全自定义的按钮切换到设置标签——它只需要从环境中获取我们的`TabController`，然后它可以调用`open`方法来执行标签切换，像这样：

```swift
struct HomeView: View {
    @EnvironmentObject private var tabController: TabController

    var body: some View {
        ScrollView {
            ...
            Button("Open settings") {
                tabController.open(.settings)
            }
        }
    }
}
```

很好! 另外，由于`TabController`是一个完全由我们控制的对象，我们也可以用它来切换主视图层次结构以外的标签。例如，我们可能想根据推送通知或其他类型的服务器事件来切换标签，现在可以通过调用上述视图代码中的相同的`open`方法来完成。

> 要了解更多关于环境对象以及SwiftUI状态管理系统的其余部分，[请查看本指南](https://www.swiftbysundell.com/articles/swiftui-state-management-guide)。

### 控制导航堆栈

就像标签视图一样，SwiftUI的`NavigationView`也可以被编程自定义控制。例如，假设我们正在开发一个应用程序，在其主导航堆栈中显示一个日历视图作为根视图，然后用户可以通过点击位于该应用程序导航栏中的编辑按钮来打开一个日历编辑视图。为了连接这两个视图，我们使用了一个`NavigationLink`，每当点击一个给定的视图时，它就会自动将其压入到导航栈中：

```swift
struct RootView: View {
    @ObservedObject var calendarController: CalendarController

    var body: some View {
        NavigationView {
            CalendarView(
                calendar: calendarController.calendar
            )
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    NavigationLink("Edit") {
   										 CalendarEditView(
       										 calendar: $calendarController.calendar
   										 )
   										 .navigationTitle("Edit your calendar")
										}
                }
            }
            .navigationTitle("Your calendar")
        }
        .navigationViewStyle(.stack)
    }
}
```

> 在这种情况下，我们在所有设备上使用堆栈式导航风格，甚至是iPad，而不是让系统选择使用哪种导航风格。

现在我们假设，我们想让我们的`CalendarView`以自定义方式显示其编辑视图，而不需要构建一个单独的实例。要做到这一点，我们可以在编辑按钮的`NavigationLink`中注入一个`isActive`绑定，然后将其传递给我们的`CalendarView`:

```swift
struct RootView: View {
    @ObservedObject var calendarController: CalendarController
    @State private var isEditViewShown = false

    var body: some View {
        NavigationView {
            CalendarView(
                calendar: calendarController.calendar,
                isEditViewShown: $isEditViewShown
            )
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    NavigationLink("Edit", isActive: $isEditViewShown) {
                        CalendarEditView(
                            calendar: $calendarController.calendar
                        )
                        .navigationTitle("Edit your calendar")
                    }
                }
            }
            .navigationTitle("Your calendar")
        }
        .navigationViewStyle(.stack)
    }
}
```

如果我们现在也更新`CalendarView`，使其使用`@Binding`绑定属性接受上述值，那么现在只要我们想显示我们的编辑视图，就可以简单地将该属性设置为`true`，我们的根视图的`NavigationLink`将自动被触发:

```swift
struct CalendarView: View {
    var calendar: Calendar
    @Binding var isEditViewShown: Bool

    var body: some View {
        ScrollView {
            ...
            Button("Edit calendar settings") {
                isEditViewShown = true
            }
        }
    }
}
```

> 当然，我们也可以选择将`isEditViewShown`属性封装在某种形式的`ObservableObject`中，例如`NavigationController`，就像我们之前处理`TabView`时那样。

这就是我们如何以自定义编程方式触发显示在我们的用户界面中的`NavigationLink`——但如果我们想在不给用户任何直接控制的情况下执行这种导航呢？

例如，我们现在假设我们正在开发一个包括导出功能的视频编辑应用程序。当用户进入导出流程时，一个`VideoExportView`被显示为模态，一旦导出操作完成，我们想把`VideoExportFinishedView`推送到该模态的导航栈中。

最初，这可能看起来非常棘手，因为（由于SwiftUI是一个声明式的UI框架）没有`push`方法，当我们想在导航栈中添加一个新视图时，我们可以调用该方法。事实上，在`NavigationView`中显示一个新视图的唯一内置方法是使用`NavigationLink`，它需要成为我们视图层次结构本身的一部分。

也就是说，这些`NavigationLink`实际上不一定是可见的——所以在这种情况下，实现我们目标的一个方法是在我们的视图中添加一个隐藏的导航链接，然后我们可以在视频导出操作完成后以编程方式触发该链接。如果我们也在我们的目标视图中隐藏系统提供的返回按钮，那么我们就可以完全锁定用户能够在这两个视图之间手动导航:

```swift
struct VideoExportView: View {
    @ObservedObject var exporter: VideoExporter
    @State private var didFinish = false
    @Environment(\.presentationMode) private var presentationMode

    var body: some View {
        NavigationView {
            VStack {
                ...
                Button("Export") {
                    exporter.export {
    didFinish = true
}
                }
                .disabled(exporter.isExporting)

                NavigationLink("Hidden finish link", isActive: $didFinish) {
                    VideoExportFinishedView(doneAction: {
                        presentationMode.wrappedValue.dismiss()
                    })
                    .navigationTitle("Export completed")
                    .navigationBarBackButtonHidden(true)
                }
                .hidden()
            }
            .navigationTitle("Export this video")
        }
        .navigationViewStyle(.stack)
    }
}

struct VideoExportFinishedView: View {
    var doneAction: () -> Void

    var body: some View {
        VStack {
            Label("Your video was exported", systemImage: "checkmark.circle")
            ...
            Button("Done", action: doneAction)
        }
    }
}
```

> 我们在`VideoExportFinishedView`中注入一个`doedAction`闭包，而不是让它检索当前的`presentationMode`本身，是因为我们希望解耦整个模态流程，而不仅仅是那个特定的视图。要了解更多信息，请查看 "[解耦SwiftUI模态或详细视图](https://www.swiftbysundell.com/articles/dismissing-swiftui-modal-and-detail-views)"。

使用这样一个隐藏的`NavigationLink`绝对可以被认为是一个有点 "黑 "的解决方案，但它的效果非常好，如果我们把一个导航链接看成是导航堆栈中两个视图之间的连接（而不仅仅是一个按钮），那么上述设置可以说是有意义的。

### 小结

尽管SwiftUI的导航系统仍然不如UIKit和AppKit提供的系统灵活，但它已经足够强大，可以满足很多不同的使用情——-特别是当与SwiftUI非常全面的状态管理系统相结合时。

当然，我们也可以选择将我们的SwiftUI视图层次包裹在[托管控制器](https://www.swiftbysundell.com/articles/swiftui-and-uikit-interoperability-part-2)中，只使用UIKit/AppKit来实现我们的导航代码。哪种解决方案是最合适的，可能取决于我们在每个项目中实际想要执行多少自定义和程序化的导航。

感谢您的阅读!

> https://www.swiftbysundell.com/articles/swiftui-programmatic-navigation/

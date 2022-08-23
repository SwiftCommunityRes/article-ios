# SwiftUI 状态管理系统指南

SwiftUI与苹果之前的UI框架的区别不仅仅在于如何定义视图和其他UI组件，还在于如何在整个使用它的应用程序中管理视图层级的状态。

SwiftUI没有使用委托、数据源或任何其他在UIKit和AppKit等命令式框架中常见的状态管理模式，而是配备了一些[属性包装器](https://www.swiftbysundell.com/articles/property-wrappers-in-swift)，使我们能够准确地声明我们的数据如何被我们的视图观察、渲染和改变。

本周，让我们仔细看看这些属性包装器中的每一个，它们之间的关系，以及它们如何构成SwiftUI整体状态管理系统的不同部分。

### 属性状态

由于SwiftUI主要是一个UI框架（尽管它也开始获得用于定义更高层次结构（如应用程序和场景）的API），其声明式设计不一定需要影响应用程序的整个模型和数据层——而只是直接绑定到我们各种视图的状态。

例如，假设我们正在开发一个`SignupView`，使用户能够通过输入用户名和电子邮件地址在应用程序中注册一个新账户。我们将使用这两个值形成一个用户模型，并将其传递给一个闭包：

```swift
struct SignupView: View {
    var handler: (User) -> Void
    var username = ""
    var email = ""

    var body: some View {
        ...
    }
}
```

由于这三个属性中只有两个——`username`和`email`——实际上会被我们的视图修改，而且这两个状态可以保持私有，我们将使用SwiftUI的`State`属性包装器来标记它们——像这样：

```swift
struct SignupView: View {
    var handler: (User) -> Void
    
    @State private var username = ""
    @State private var email = ""

    var body: some View {
        ...
    }
}
```

这样做将自动在这两个值和我们的视图本身之间建立一个连接——这意味着我们的视图将在每次改变这两个值的时候被重新渲染。在我们的主体中，我们将把这两个属性分别绑定到一个相应的`TextField`上，以使它们可以被用户编辑：

```swift
struct SignupView: View {
    var handler: (User) -> Void

    @State private var username = ""
    @State private var email = ""

    var body: some View {
        VStack {
            TextField("Username", text: $username)
            TextField("Email", text: $email)
            Button(
                action: {
                    self.handler(User(
                        username: self.username,
                        email: self.email
                    ))
                },
                label: { Text("Sign up") }
            )
        }
        .padding()
    }
}
```

因此，`State`被用来表示SwiftUI视图的内部状态，并在该状态被改变时自动使视图更新。因此，最常见的做法是将`State`属性包装器保持为私有，这可以确保它们只在该视图的主体内被改变（试图在其他地方改变它们实际上会导致运行时崩溃）。

### 双向绑定

看一下上面的代码样本，我们将每个属性传入其`TextField`的方式是在这些属性名称前加上`$`。这是因为我们不只是将普通的`String`值传入这些文本字段，而是与我们的`State`包装的属性本身绑定。

为了更详细地探讨这意味着什么，让我们现在假设我们想创建一个视图，让我们的用户编辑他们最初在注册时输入的个人资料信息。由于我们现在要修改外部状态值，而不仅仅是私人状态值，所以这次我们将`username`和`email`属性标记为`Bingding`:

```swift
struct ProfileEditingView: View {
    @Binding var username: String
    @Binding var email: String

    var body: some View {
        VStack {
            TextField("Username", text: $username)
            TextField("Email", text: $email)
        }
        .padding()
    }
}
```

最酷的是，绑定不仅仅局限于单一的内置值，比如字符串或整数，而是可以用来将任何Swift值绑定到我们的一个视图中。例如，我们可以将用户模型本身传递给`ProfileEditingView`，而不是传递两个单独的`username`和`email`:

```swift
struct ProfileEditingView: View {
    @Binding var user: User

    var body: some View {
        VStack {
            TextField("Username", text: $user.username)
            TextField("Email", text: $user.email)
        }
        .padding()
    }
}
```

就像我们在将`State`和`Binding`包装的属性传入各种`TextField`实例时用`$`作为前缀一样，我们在将任何`State`值连接到我们自己定义的`Binding`属性时也可以做同样的事情。

例如，这里有一个`ProfileView`的实现，它使用一个`Stage`包装属性来跟踪一个用户模型，然后在将上述`ProfileEditingView`的实例作为工作表呈现时，将该模型传递一个绑定——这将自动同步用户对该原始`State`属性值的任何改变:

```swift
struct ProfileView: View {
    @State private var user = User.load()
    @State private var isEditingViewShown = false

    var body: some View {
        VStack(alignment: .leading, spacing: 10) {
            Text("Username: ")
                .foregroundColor(.secondary)
                + Text(user.username)
            Text("Email: ")
                .foregroundColor(.secondary)
                + Text(user.email)
            Button(
                action: { self.isEditingViewShown = true },
                label: { Text("Edit") }
            )
        }
        .padding()
        .sheet(isPresented: $isEditingViewShown) {
            VStack {
                ProfileEditingView(user: self.$user)
                Button(
                    action: { self.isEditingViewShown = false },
                    label: { Text("Done") }
                )
            }
        }
    }
}
```

> 请注意，我们也可以通过给一个`State`包装的属性分配一个新的值来改变它——比如我们在 "Done "按钮的动作处理程序中把`isEditingViewShown`设置为`false`。

因此，一个`Binding`标记的属性在给定的视图和定义在该视图之外的状态属性之间提供了一个双向的连接，而`Statr`和`Binding`包装的属性都可以通过在其属性名前加上`$`来作为绑定物传递。

### 观察对象

`State`和`Bingding`的共同点是，它们处理的是在SwiftUI视图层次结构本身中管理的值。然而，虽然建立一个将所有的状态都保存在其各种视图中的应用程序是肯定可行的，但从架构和关注点分离的角度来看，这通常不是一个好主意，而且很容易导致我们的视图变得相当[庞大和复杂](https://www.swiftbysundell.com/articles/avoiding-massive-swiftui-views/)。

值得庆幸的是，SwiftUI还提供了一些机制，使我们能够将外部模型对象连接到我们的各种视图。其中一个机制是`ObservableObject`协议，当它与`ObservedObject`属性包装器结合时，我们可以设置与我们视图层之外管理的引用类型的绑定。

作为一个例子，让我们更新上面定义的`ProfileView`——通过将管理`User`模型的责任从视图本身转移到一个新的、专门的对象中。现在，我们可以用许多不同的方式来描述这样一个对象，但由于我们正在寻找创建一个类型来控制我们的一个模型的实例——让我们把它变成一个符合SwiftUI的`ObservableObject`协议的[模型控制器](https://www.swiftbysundell.com/articles/model-controllers-in-swift/):

```swift
class UserModelController: ObservableObject {
    @Published var user: User
    ...
}
```

> `Published`属性包装器用于定义对象的哪些属性在被修改时应让观察通知被触发。

有了上面的类型，现在让我们回到`ProfileView`，让它观察新的`UserModelController`的实例，作为一个`ObservedObject`，而不是用一个`State`属性包装器来跟踪我们的用户模型。最重要的是，我们仍然可以很容易地将这个模型绑定到我们的`ProfileEditingView`上，就像以前一样，因为`ObservedObject`属性包装器也可以转换为绑定：

```swift
struct ProfileView: View {
    @ObservedObject var userController: UserModelController
    @State private var isEditingViewShown = false

    var body: some View {
        VStack(alignment: .leading, spacing: 10) {
            Text("Username: ")
                .foregroundColor(.secondary)
                + Text(userController.user.username)
            Text("Email: ")
                .foregroundColor(.secondary)
                + Text(userController.user.email)
            Button(
                action: { self.isEditingViewShown = true },
                label: { Text("Edit") }
            )
        }
        .padding()
        .sheet(isPresented: $isEditingViewShown) {
            VStack {
                ProfileEditingView(user: self.$userController.user)
                Button(
                    action: { self.isEditingViewShown = false },
                    label: { Text("Done") }
                )
            }
        }
    }
}
```

然而，我们的新实现与之前使用的基于状态的实现之间的一个重要区别是，我们的`UserModelController`现在需要作为初始化器的一部分被[注入](https://www.swiftbysundell.com/articles/different-flavors-of-dependency-injection-in-swift/)到`ProfileView`中。

除了 "迫使 "我们在代码库中建立一个更明确的依赖关系图之外，原因是一个标有`ObservedObject`的属性并不意味着对这个属性所指向的对象有任何形式的所有权。

因此，虽然下面的内容在技术上可能会被编译，但最终会导致运行时的问题——因为当我们的视图在更新时被重新创建，`UserModelController`实例可能会被删除（因为我们的视图现在是它的主要所有者）:

```swift
struct ProfileView: View {
    @ObservedObject var userController = UserModelController.load()
    ...
}
```

> 重要的是要记住: SwiftUI视图不是对正在屏幕上渲染的实际UI组件的引用，而是描述我们的UI的轻量级值——因此它们没有像`UIView`实例那样的生命周期。

为了解决上述问题，苹果在iOS 14和macOS Big Sur中引入了一个新的属性包装器，名为`StateObject`。标记为`StateObject`的属性与`ObservedObject`的行为完全相同——此外，SwiftUI将确保存储在此类属性中的任何对象不会因为框架在重新渲染视图时重新创建新实例而被意外释放：

```swift
struct ProfileView: View {
    @StateObject var userController = UserModelController.load()
    ...
}
```

尽管从技术上来说，从现在开始可以只使用`StateObject`——我仍然建议在观察外部对象时使用`ObservedObject`，而在处理视图本身拥有的对象时只使用`StateObject`。把`StateObject`和`ObservedObject`看作是`State`和`Binding`的参考类型，或者SwiftUI版本的强和弱属性。

### 观察和修改环境变量

最后，让我们来看看SwiftUI的环境系统如何被用来在两个互不直接连接的视图之间传递各种状态。尽管在一个父视图和它的一个子视图之间创建绑定通常很容易，但在整个视图层次结构中传递某个对象或值可能相当麻烦——而这正是环境变量旨在解决的问题类型。

有两种主要的方法来使用SwiftUI的环境。一种是首先在想要检索给定对象的视图中定义一个`EnvironmentObject`包装的属性——例如像这个`ArticleView`如何检索一个包含颜色信息的`Theme`对象:

```swift
struct ArticleView: View {
    @EnvironmentObject var theme: Theme
    var article: Article

    var body: some View {
        VStack(alignment: .leading) {
            Text(article.title)
                .foregroundColor(theme.titleTextColor)
            Text(article.body)
                .foregroundColor(theme.bodyTextColor)
        }
    }
}
```

然后，我们必须确保在我们的视图的某一个父类中提供我们的环境对象（在这种情况下是一个`Theme`实例），然后SwiftUI会处理其余的事情。这是通过使用`environmentalObject`修饰符完成的，例如，像这样：

```swift
struct RootView: View {
    @ObservedObject var theme: Theme
    @ObservedObject var articleLibrary: ArticleLibrary

    var body: some View {
        ArticleListView(articles: articleLibrary.articles)
            .environmentObject(theme)
    }
}
```

> 请注意，我们不需要将上述修改器应用于将使用我们的环境对象的确切视图——我们可以将其应用于我们的层次结构中任何在其之上的视图。

使用 SwiftUI 环境系统的第二种方式是定义一个自定义的`EnvironmentKey` ——然后它可以被用来向内置的` EnvironmentValues` 类型分配和检索值:

```swift
struct ThemeEnvironmentKey: EnvironmentKey {
    static var defaultValue = Theme.default
}

extension EnvironmentValues {
    var theme: Theme {
        get { self[ThemeEnvironmentKey.self] }
        set { self[ThemeEnvironmentKey.self] = newValue }
    }
}
```

有了上述内容，我们现在可以使用`Enviroment`属性包装器（而不是`EnvironmentObject`）来标记我们视图的`theme`属性，并传入我们希望检索的环境键的[键值路径](https://www.swiftbysundell.com/articles/the-power-of-key-paths-in-swift):

```swift
struct ArticleView: View {
    @Environment(\.theme) var theme: Theme
    var article: Article

    var body: some View {
        VStack(alignment: .leading) {
            Text(article.title)
                .foregroundColor(theme.titleTextColor)
            Text(article.body)
                .foregroundColor(theme.bodyTextColor)
        }
    }
}
```

上述两种方法的一个明显区别是，基于键的方法要求我们在编译时定义一个默认值，而基于环境对象`EnvironmentObject`的方法则假设在运行时提供这样一个值（如果不这样做将导致崩溃）。

### 小结

SwiftUI管理状态的方式绝对是该框架最有趣的方面之一，它可能需要我们稍微重新思考数据在应用中的传递方式——至少在涉及到将被我们的UI直接消费和修改的数据时是这样。

我希望这篇指南能成为一个很好的方式来概述SwiftUI的各种状态处理机制，尽管一些更具体的API被遗漏了，这篇文章中强调的概念应该涵盖了所有基于SwiftUI的状态处理的绝大多数用例。

感谢你的阅读! 



> https://www.swiftbysundell.com/articles/swiftui-state-management-guide/
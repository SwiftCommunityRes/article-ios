## 前言

`Sendable` 和 `@Sendable` 是 Swift 5.5 中的并发修改的一部分，解决了结构化的并发结构体和执行者消息之间传递的类型检查的挑战性问题。

在深入探讨`Sendable`的话题之前，我鼓励你阅读我围绕 [async/await](https://www.avanderlee.com/swift/async-await/)、[actors](https://www.avanderlee.com/swift/actors/) 和 [actor isolation](https://www.avanderlee.com/swift/nonisolated-isolated/) 的文章。这些文章涵盖了新的并发性变化的基础知识，它们与本文所解释的技术直接相关。

## 使用 Sendable

应该在什么时候使用 `Sendable`？

`Sendable`协议和闭包表明那些传递的值的公共API是否线程安全的向编译器传递了值。当没有公共修改器、有内部锁定系统或修改器实现了与值类型一样的复制写入时，公共API可以安全地跨并发域使用。

标准库中的许多类型已经支持了`Sendable`协议，消除了对许多类型添加一致性的要求。由于标准库的支持，编译器可以为你的自定义类型创建隐式一致性。

例如，整型支持该协议：

```swift
extension Int: Sendable {}
```

一旦我们创建了一个具有 `Int` 类型的单一属性的值类型结构体，我们就隐式地得到了对 `Sendable` 协议的支持。

```swift
// 隐式地遵守了 Sendable 协议
struct Article {
    var views: Int
}
```

与此同时，同样的 `Article` 内容的类，将不会有隐式遵守该协议：

```swift
// 不会隐式的遵守 Sendable 协议
class Article {
    var views: Int
}
```

类不符合要求，因为它是一个引用类型，因此可以从其他并发域变异。换句话说，该类文章(`Article`)的传递不是线程安全的，所以编译器不能隐式地将其标记为遵守`Sendable`协议。

### 使用泛型和枚举时的隐式一致性

很好理解的是，如果泛型不符合`Sendable`协议，编译器就不会为泛型添加隐式的一致性。

```swift
// 因为 Value 没有遵守 Sendable 协议，所以 Container 也不会自动的隐式遵守该协议
struct Container<Value> {
    var child: Value
}
```

然而，如果我们将协议要求添加到我们的泛型中，我们将得到隐式支持:

```swift
// Container 隐式地符合 Sendable，因为它的所有公共属性也是如此。
struct Container<Value: Sendable> {
    var child: Value
}
```

对于有关联值的枚举也是如此：

![如果枚举值们不符合 Sendable 协议，隐式的Sendable协议一致性就不会起作用。](https://www.avanderlee.com/wp-content/uploads/2021/10/sendable_protocol_swift-1024x183.png)

你可以看到，我们自动从编译器中得到一个错误：

> *Associated value ‘loggedIn(name:)’ of ‘Sendable’-conforming enum ‘State’ has non-sendable type ‘(name: NSAttributedString)’*

我们可以通过使用一个值类型`String`来解决这个错误，因为它已经符合`Sendable`。

```swift
enum State: Sendable {
    case loggedOut
    case loggedIn(name: String)
}
```

### 从线程安全的实例中抛出错误

同样的规则适用于想要符合`Sendable`的错误类型。

```swift
struct ArticleSavingError: Error {
    var author: NonFinalAuthor
}

extension ArticleSavingError: Sendable { }
```

由于作者不是不变的（non-final），而且不是线程安全的（后面会详细介绍），我们会遇到以下错误：

> *Stored property ‘author’ of ‘Sendable’-conforming struct ‘ArticleSavingError’ has non-sendable type ‘NonFinalAuthor’*

你可以通过确保`ArticleSavingError`的所有成员都符合`Sendable`协议来解决这个错误。

## 如何使用Sendable协议

隐式一致性消除了很多我们需要自己为`Sendable`协议添加一致性的情况。然而，在有些情况下，我们知道我们的类型是线程安全的，但是编译器并没有为我们添加隐式一致性。

常见的例子是被标记为不可变和内部具有锁定机制的类：

```swift
/// User 是不可改变的，因此是线程安全的，所以可以遵守 Sendable 协议
final class User: Sendable {
    let name: String

    init(name: String) { self.name = name }
}
```

你需要用`@unchecked`属性来标记可变类，以表明我们的类由于内部锁定机制所以是线程安全的：

```swift
extension DispatchQueue {
    static let userMutatingLock = DispatchQueue(label: "person.lock.queue")
}

final class MutableUser: @unchecked Sendable {
    private var name: String = ""

    func updateName(_ name: String) {
        DispatchQueue.userMutatingLock.sync {
            self.name = name
        }
    }
}
```

## 要在同一源文件中遵守 `Sendable`的限制

`Sendable`协议的一致性必须发生在同一个源文件中，以确保编译器检查所有可见成员的线程安全。

例如，你可以在例如 Swift package这样的模块中定义以下类型： 

```swfit
public struct Article {
    internal var title: String
}
```

`Article` 是公开的，而标题`title`是内部的，在模块外不可见。因此，编译器不能在源文件之外应用`Sendable`一致性，因为它对标题属性不可见，即使标题使用的是遵守`Sendable`协议的`String`类型。

同样的问题发生在我们想要使一个可变的非最终类遵守`Sendable`协议时：

![可变的非最终类无法遵守 Sendable 协议](https://www.avanderlee.com/wp-content/uploads/2021/10/non_final_sendable-1024x185.png)

由于该类是非最终的，我们无法符合`Sendable`协议的要求，因为我们不确定其他类是否会继承`User`的非`Sendable`成员。因此，我们会遇到以下错误:

> *Non-final class ‘User’ cannot conform to `Sendable`; use `@unchecked Sendable`*

正如你所看到的，编译器建议使用`@unchecked Sendable`。我们可以把这个属性添加到我们的`User`类中，并摆脱这个错误:

```swift
class User: @unchecked Sendable {
    let name: String

    init(name: String) { self.name = name }
}
```

然而，这确实要求我们无论何时从`User`继承，都要确保它是线程安全的。由于我们给自己和同事增加了额外的责任，我不鼓励使用这个属性，建议使用组合、最终类或值类型来实现我们的目的。

## 如何使用 `@Sendabele`

函数可以跨并发域传递，因此也需要可发送的一致性。然而，函数不能符合协议，所以Swift引入了`@Sendable`属性。你可以传递的函数的例子是全局函数声明、闭包和访问器，如`getters`和`setters`。

[SE-302](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)的部分动机是执行尽可能少的同步

> 我们希望这样一个系统中的绝大多数代码都是无同步的。

使用`@Sendable`属性，我们将告诉编译器，他不需要额外的同步，因为闭包中所有捕获的值都是线程安全的。一个典型的例子是在[Actor isolation](https://www.avanderlee.com/swift/nonisolated-isolated/)中使用闭包。

```swift
actor ArticlesList {
    func filteredArticles(_ isIncluded: @Sendable (Article) -> Bool) async -> [Article] {
        // ...
    }
}
```

如果你用非 `Sendabel` 类型的闭包，我们会遇到一个错误:

```swift
let listOfArticles = ArticlesList()
var searchKeyword: NSAttributedString? = NSAttributedString(string: "keyword")
let filteredArticles = await listOfArticles.filteredArticles { article in
 
    // Error: Reference to captured var 'searchKeyword' in concurrently-executing code
    guard let searchKeyword = searchKeyword else { return false }
    return article.title == searchKeyword.string
}
```

当然，我们可以通过使用一个普通的`String`来快速解决这种情况，但它展示了编译器如何帮助我们执行线程安全。

## Swift 6: 为你的代码启用严格的并发性检查

Xcode 14 允许您通过 `SWIFT_STRICT_CONCURRENCY` 构建设置启用严格的并发性检查。

![启用严格的并发性检查，以修复 Sendable 的符合性](https://www.avanderlee.com/wp-content/uploads/2021/10/swift_6_strict_concurrency_checking_sendable-1024x348.jpg)

这个构建设置控制编译器对`Sendable`和`actor-isolation`检查的执行水平:

- *Minimal* : 编译器将只诊断明确标有`Sendable`一致性的实例，并等同于Swift 5.5和5.6的行为。不会有任何警告或错误。
- *Targeted*:  强制执行`Sendable`约束，并对你所有采用`async/await`等并发的代码进行`actor-isolation`检查。编译器还将检查明确采用`Sendable`的实例。这种模式试图在与现有代码的兼容性和捕捉潜在的数据竞赛之间取得平衡。
- *Complete*:  匹配预期的 Swift 6语义，以检查和消除数据竞赛。这种模式检查其他两种模式所做的一切，并对你项目中的所有代码进行这些检查。

严格的并发检查构建设置有助于 Swift 向数据竞赛安全迈进。与此构建设置相关的每一个触发的警告都可能表明你的代码中存在潜在的数据竞赛。因此，必须考虑启用严格并发检查来验证你的代码。

### Enabling strict concurrency in Xcode 14

你会得到的警告数量取决于你在项目中使用并发的频率。对于[Stock Analyzer](https://stock-analyzer.app/)，我有大约17个警告需要解决：

![并发相关的警告，表明潜在的数据竞赛.](https://www.avanderlee.com/wp-content/uploads/2021/10/strict_concurrency_checking_warnings-1024x649.jpg)

这些警告可能让人望而生畏，但利用本文的知识，你应该能够摆脱大部分警告，防止数据竞赛的发生。然而，有些警告是你无法控制的，因为是外部模块触发了它们。在我的例子中，我有一个与`SWHighlight`有关的警告，它不符合`Sendable`，而苹果在他们的`SharedWithYou`框架中定义了它。

在上述`SharedWithYou`框架的例子中，最好是等待库的所有者添加`Sendable`支持。在这种情况下，这就意味着要等待苹果公司为`SWHighlight`实例指明`Sendable`的一致性。对于这些库，你可以通过使用`@preconcurrency`属性来暂时禁用`Sendable`警告:

```swift
@preconcurrency import SharedWithYou
```

重要的是要明白，我们并没有解决这些警告，而只是禁用了它们。来自这些库的代码仍然有可能发生数据竞赛。如果你正在使用这些框架的实例，你需要考虑实例是否真的是线程安全的。一旦你使用的框架被更新为`Sendable`的一致性，你可以删除`@preconcurrency`属性，并修复可能触发的警告。

> 来源：[Sendable and @Sendable closures explained with code examples](https://www.avanderlee.com/swift/sendable-protocol-closures/)

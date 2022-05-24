Swift 的类型推断能力从一开始就是语言的核心部分，它极大地减少了我们在声明有默认值的变量和属性时手动指定类型的工作。例如，表达式`var number = 7`不需要包含任何类型注释，因为编译器能够推断出值`7`是一个`Int`，我们的`number`变量应该被相应的类型化。

作为 Xcode 13.3 的一部分而一起发布的 Swift 5.6，通过引入 "类型占位符（type placeholders） "的概念，继续扩展这些类型推理能力，这在处理集合和其他通用类型时非常有用。

例如，假设我们想创建一个`Combine`里面具有默认整数值的 `CurrentValueSubject`的实例。关于如何做到这一点的初步想法可能是简单地将我们的默认值传递给该主体的初始化器，然后将结果存储在本地的一个`let`声明的属性中（就像创建一个普通的`Int`值时一样）。然而，这样做会给我们带来以下编译器错误：

```swift
// Error: "Generic parameter 'Failure' could not be inferred"
// Error: “无法被推断出泛型的`Failure`参数 ”
let counterSubject = CurrentValueSubject(0)
```

这是因为`CurrentValueSubject`是一个泛型类型，实例化时不仅需要`Output`类型，还需要`Failure`类型——这是该主体能够抛出的错误类型。

因为我们不希望我们的主体在这种情况下抛出任何错误，所以我们会给它一个`Failure`类型的值`Never`（这是在 Swift 中使用 `Combine` 的一个常见惯例）。但为了做到这一点，在 Swift 5.6 之前，我们需要明确地指定我们的`Int`输出类型——像这样：

```swift
let counterSubject = CurrentValueSubject<Int, Never>(0)
```

不过从 Swift 5.6 开始，这种情况就不存在了——因为我们现在可以使用一个类型占位符来表示我们主体的`Output`类型，这让我们再次利用编译器为我们自动推断出该类型，就像在声明一个普通的`Int`值一样：

```swift
let counterSubject = CurrentValueSubject<_, Never>(0)
```

这很好，但可以说这并不是 swift 里面很大的改进。毕竟，我们用`_`代替`Int`只是节省了两个字符，而且手动指定像`Int`这样的简单类型也不是一开始就有问题的。

**但现在让我们看看这个功能如何扩展到更复杂的类型，这是它真正开始发光的地方。**例如，假设我们的项目包含以下函数，让我们加载一个用户注解的PDF文件:

```swift
func loadAnnotatedPDF(named: String) -> Resource<PDF<UserAnnotations>> {
    ...
}
```

上面的函数使用了一个相当复杂的泛型作为它的返回类型，这可能是因为我们需要在多个地方中重复使用我们的`Resource`类型，也因为我们选择了使用*[幻象类型](https://www.swiftbysundell.com/articles/phantom-types-in-swift)*来指定我们当前处理的是哪种PDF。

现在让我们看看，如果我们在创建主体时调用上述函数，而不是仅仅使用一个简单的整数，那么我们之前基于`CurrentValueSubject`的代码会是什么样子:

```swift
// Before Swift 5.6:
let pdfSubject = CurrentValueSubject<Resource<PDF<UserAnnotations>>, Never>(
    loadAnnotatedPDF(named: name)
)

// Swift 5.6:
let pdfSubject = CurrentValueSubject<_, Never>(
    loadAnnotatedPDF(named: name)
)
```

这是一个相当大的改进啊 基于 Swift 5.6 的版本不仅为我们节省了一些输入，而且由于 `pdfSubject` 的类型现在完全来自 `loadAnnotatedPDF` 函数，这可能会使该函数（及其相关代码）的迭代更加容易——因为如果我们改变该函数的返回类型，需要更新的手动类型注释将减少。

不过，值得指出的是，在上述情况下，还有另一种方法可以利用Swift的类型推理能力——那就是使用**类型别名**，而不是**类型占位符**。例如，我们可以在这里定义一个`UnfailingValueSubject`类型别名，我们可以用它来轻松地创建不会产生任何错误的主体：

```swift
typealias UnfailingValueSubject<T> = CurrentValueSubject<T, Never>
```

有了上述内容，我们现在就可以在没有任何泛型注解的情况下创建我们的`pdfSubject`了——因为编译器能够推断出`T`指的是什么类型，而且失败类型`Never`已经被硬编码到我们的新类型别名中:

```swift
let pdfSubject = UnfailingValueSubject(loadAnnotatedPDF(named: name))
```

但这并不意味着类型别名在通常情况下都比类型占位符好，因为如果我们要为每种特定情况定义新的类型别名，那么这也会使我们的代码库变得更加复杂。有时，在内联中指定所有的东西（比如使用类型占位符时）绝对是个好办法，因为这可以让我们定义完全独立的表达式。

在我们总结之前，让我们也来看看类型占位符是如何与集合字面量(literals)一起使用的——例如在创建一个字典时。在这里，我们选择手动指定我们的字典的 `Key` 类型（为了能够使用点语法来指代枚举的各种情况），同时为该字典的值使用一个类型占位符：

```swift
enum UserRole {
    case local
    case remote
}

let latestMessages: [UserRole: _] = [
    .local: "",
    .remote: ""
]
```

这就是类型占位符——Swift 5.6 中引入的一个新功能，在处理稍微复杂的通用类型时，它可能真的很有用。但值得指出的是，这些占位符只能在调用站点使用，而不是在指定函数或计算属性的返回类型时使用。
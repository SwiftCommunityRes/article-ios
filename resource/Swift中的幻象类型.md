## 前言

模糊的数据可以说是一般应用程序中最常见的错误和问题的来源之一。虽然 Swift 通过其强大的类型系统和完善的编译器帮助我们避免了许多含糊不清的来源——但只要我们无法在编译时保证某个数据总是符合我们的要求，就总是有风险，我们最终会处于含糊不清或不可预测的状态。

本周，让我们来看看一种技术，它可以让我们利用 Swift 的类型系统在编译时执行更多种类的数据验证——消除更多潜在的歧义来源，并帮助我们在整个代码库中保持类型安全——通过使用**幻象类型**(*phantom types*)。

## 定义良好，但仍然含糊不清

举个例子，假设我们正在开发一个文本编辑器，虽然它最初只支持纯文本文件——随着时间的推移，我们还增加了对编辑HTML文档的支持，以及PDF预览。

为了能够尽可能多地重复使用我们原来的文档处理代码，我们继续使用与开始时相同的`Document`模型——只是现在它获得了一个`Format`属性，告诉我们正在处理什么样的文档：

```swift
struct Document {
    enum Format {
        case text
        case html
        case pdf
    }

    var format: Format
    var data: Data
    var modificationDate: Date
    var author: Author
}
```

能够避免代码重复当然是件好事，而且枚举是当我们在处理一个模型的不同格式或变体时[一般情况下建模](https://www.swiftbysundell.com/articles/modelling-state-in-swift) 的好方法，但是上述那种设置实际上最终会造成相当多的模糊性。

例如，我们可能有一些API，只有在调用给定格式的文档时才有意义——比如这个打开文本编辑器的函数，它假定任何传入它的`Document`都是文本文档：

```swift
func openTextEditor(for document: Document) {
    let text = String(decoding: document.data, as: UTF8.self)
    let editor = TextEditor(text: text)
    ...
}
```

虽然如果我们不小心将一个HTML文档传递给上述函数并不是世界末日（HTML毕竟只是文本），但试图以这种方式打开一个PDF，很可能会导致呈现出完全无法理解的东西，我们的文本编辑功能将无法工作，我们的应用程序甚至可能最终崩溃。

我们在编写任何其他特定格式的代码时都会不断遇到同样的问题，例如，如果我们想通过实现一个解析器和一个专门的编辑器来改善编辑HTML文档的用户体验：

```swift
func openHTMLEditor(for document: Document) {
    // 就像我们上面用于文本编辑的函数一样，
    // 这个函数假设它总是被传递给HTML文档。
    let parser = HTMLParser()
    let html = parser.parse(document.data)
    let editor = HTMLEditor(html: html)
    ...
}
```

一个关于如何解决上述问题的初步想法可能是编写一个包装函数，切换到所传递文档的格式，然后为每种情况打开正确的编辑器。然而，虽然这对文本和HTML文档很有效，但由于PDF文档在我们的应用程序中是不可编辑的——当遇到PDF时，我们将被迫抛出一个错误，触发一个断言，或以[其他方式失败](https://www.swiftbysundell.com/articles/picking-the-right-way-of-failing-in-swift)：

```swift
func openEditor(for document: Document) {
    switch document.format {
    case .text:
        openTextEditor(for: document)
    case .html:
        openHTMLEditor(for: document)
    case .pdf:
        assertionFailure("Cannot edit PDF documents")
    }
}
```



上述情况不是很好，因为它要求我们作为开发者始终跟踪我们在任何给定的代码路径中所处理的文件类型，而我们可能犯的任何错误只能在运行时被发现——编译器根本没有足够的信息可以在编译时进行这种检查。

因此，尽管我们的 "Document "模型乍一看可能非常优雅和完善，但事实证明，它并不完全是手头情况的正确解决方案。

## 看起来我们需要一个协议!

解决上述问题的一个方法是把`Document`变成一个协议，而不是作为一个具体的类型，把它的所有属性（除了`format`）都作为要求：

```swift
protocol Document {
    var data: Data { get }
    var modificationDate: Date { get }
    var author: Author { get }
}
```

有了上述变化，我们现在可以为我们的三种文档格式中的每一种实现专门的类型，并让这些类型都符合我们新的文档协议——比如这样：

```swift
struct TextDocument: Document {
    var data: Data
    var modificationDate: Date
    var author: Author
}
```

上述方法的好处是，它使我们既能实现可以对任何`Document`进行操作的通用功能，又能实现只接受某种具体类型的特定API：

```swift
// 这个函数可以保存任何文件，
// 所以它接受任何符合我们的新文档协议。
func save(_ document: Document) {
    ...
}

// 我们现在只能向我们的函数传递文本文件，
// 即打开一个文本编辑器。
func openTextEditor(for document: TextDocument) {
    ...
}
```

我们在上面所做的基本上是将以前在运行时进行的检查转为在编译时进行验证——因为编译器现在能够检查我们是否总是向我们的每个API传递正确格式的文件，这是一个很大的进步。

然而，通过执行上述改变，我们也**失去了我们最初实现的优点——代码重用**。由于我们现在使用一个协议来表示所有的文档格式，我们将需要为我们的三种文档类型中的每一种编写完全重复的模型实现，以及为我们将来可能增加的任何其他格式提供支持。

## 引入幻象类型

如果我们能找到一种方法，既能为所有格式重用相同的`Document`模型，又能在编译时验证我们特定格式的代码，岂不妙哉？事实证明，我们之前的一行代码实际上可以给我们一个实现这一目标的提示：

```swift
let text = String(decoding: document.data, as: UTF8.self)
```

当把`Data`转换为`String`时，就像我们上面做的那样，我们通过传递对该类型本身的引用来传递我们希望字符串被解码的编码——在本例中是UTF8。这真的很有趣。如果我们再深入一点，就会发现 Swift 标准库将我们上面提到的UTF8类型定义为另一个类似命名空间的枚举中的一个无大小写枚举，称为`Unicode`。

```swift
enum Unicode {
    enum UTF8 {}
    ...
}

typealias UTF8 = Unicode.UTF8
```

> 请注意，如果你看一下`UTF8`类型的实际实现，它确实包含一个私有case，只是为了向后兼容 Swift 3 而存在。

我们在这里看到的是一种被称为**幻象类型的技术——当类型被用作标记，而不是被实例化来表示值或对象时**。事实上，由于上述枚举都没有任何公开的情况，它们甚至不能被实例化！

让我们看看是否可以用同样的技术来解决我们的`Document`困境。我们首先将`Document`还原成一个结构体，只是这次我们将删除它的`format`属性（以及相关的枚举），而将它变成一个覆盖任何`Format`类型的泛型——比如这样:

```swift
struct Document<Format> {
    var data: Data
    var modificationDate: Date
    var author: Author
}
```

受标准库的`Unicode`枚举及其各种编码的启发，我们将定义一个类似的枚举——`DocumentFormat`——作为三个无大小写的枚举的命名空间，每种格式都有一个：

```swift
enum DocumentFormat {
    enum Text {}
    enum HTML {}
    enum PDF {}
}
```

请注意，这里不涉及任何协议——任何类型都可以被用作格式，因为就像`String`和它的各种编码一样，我们将只使用文档的`Format`类型作为编译时的标记。这将使我们能够像这样写出我们特定格式的API：

```swift
func openTextEditor(for document: Document<DocumentFormat.Text>) {
    ...
}

func openHTMLEditor(for document: Document<DocumentFormat.HTML>) {
    ...
}

func openPreview(for document: Document<DocumentFormat.PDF>) {
    ...
}
```

当然，我们仍然可以编写不需要任何特定格式的通用代码。例如，这里我们可以把之前的`save`API变成一个完全通用的函数：

```swift
func save<F>(_ document: Document<F>) {
    ...
}
```

然而，总是输入`Document<DocumentFormat.Text>`来引用一个文本文档是相当乏味的，所以让我们也使用类型别名为每种格式定义速记。这将给我们提供漂亮的、有语义的名字，而不需要任何重复的代码：

```swift
typealias TextDocument = Document<DocumentFormat.Text>
typealias HTMLDocument = Document<DocumentFormat.HTML>
typealias PDFDocument = Document<DocumentFormat.PDF>
```

在涉及到特定格式的扩展时，幻象类型也确实大放异彩，现在可以直接使用 Swift 强大的泛型系统和[泛型型约束](https://www.swiftbysundell.com/articles/using-generic-type-constraints-in-swift-4/)来实现。例如，我们可以用一个生成`NSAttributedString`的方法来扩展所有文本文档:

```swift
extension Document where Format == DocumentFormat.Text {
    func makeAttributedString(withFont font: UIFont) -> NSAttributedString {
        let string = String(decoding: data, as: UTF8.self)

        return NSAttributedString(string: string, attributes: [
            .font: font
        ])
    }
}
```

由于我们的幻象类型在最后只是普通的类型——我们也可以让它们遵守协议，并使用这些协议作为泛型约束。例如，我们可以让我们的一些`DocumentFormat`类型遵守`Printable`协议，然后我们可以在打印代码中使用这些协议作为约束条件。这里有大量的可能性。

## 一个标准的模式

起初，幻象类型在 Swift 中可能看起来有点 "格格不入"。然而，虽然 Swift 并没有像更多的纯函数式语言（如Haskell）那样为幻象类型提供一流的支持，但在标准库和苹果平台SDK的许多不同地方都可以找到这种模式。

例如，`Foundation`的`Measurement` API使用幻象类型来确保在传递各种测量值时的类型安全——例如度数、长度和重量：

```swift
let meters = Measurement<UnitLength>(value: 5, unit: .meters)
let degrees = Measurement<UnitAngle>(value: 90, unit: .degrees)
```

通过使用幻影类型，上述两个测量值不能被混合，因为每个值是哪种单位，都被编码到该值的类型中。这可以防止我们不小心将一个长度传递给一个接受角度的函数，反之亦然——就像我们之前防止文档格式被混淆一样。

## 结论

使用幻象类型是一种非常强大的技术，它可以让我们利用类型系统来验证一个特定值的不同变体。虽然使用幻象类型通常会使API更加冗长，而且确实伴随着泛型的复杂性——当处理不同的格式和变体时，它可以让我们减少对运行时检查的依赖，而让编译器来执行这些检查。

就像一般的泛型一样，我认为在部署幻象类型之前，首先要仔细评估当前的情况，这很重要。就像我们最初的`Document`模型并不是手头任务的正确选择，尽管它的结构很好，但如果部署在错误的情况下，幻象类型会使简单的设置变得更加复杂。像往常一样，它归结为为工作选择正确的工具。

谢谢你的阅读! 🚀
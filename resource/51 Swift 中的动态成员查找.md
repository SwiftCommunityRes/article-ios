## 前言

我最喜欢 Swift 语言的一个特性是动态成员查找（dynamic member lookup）。虽然我们并不经常使用它，但它通过改进我们访问特定类型数据的方式，显著改善了所提供类型的 API。

> Glassfy：简化构建、管理和推广应用内购买。从订阅管理 SDK 到付费墙等完整的货币化工具。立即免费构建。

## 基础知识

假设我们正在开发一个提供缓存功能的类型，并将其建模为名为 Cache 的结构体。

```Swift
struct Cache {
    var storage: [String: Data] = [:]
}
```

为了访问缓存的数据，我们调用存储属性的下标，该存储属性是 Dictionary 类型提供的。

```Swift
var cache = Cache()
let profile = cache.storage["profile"]
```

在这里没有什么特别之处。我们像以前一样通过 Dictionary 类型的下标访问字典。让我们看看如何使用 `@dynamicMemberLookup` 属性改进 Cache 类型的 API。

```Swift
@dynamicMemberLookup
struct Cache {
    private var storage: [String: Data] = [:]

    subscript(dynamicMember key: String) -> Data? {
        storage[key]
    }
}
```

如上例所示，我们使用 `@dynamicMemberLookup` 属性标记了 Cache 类型。我们必须实现具有 `dynamicMember` 参数并返回我们需要的任何内容的下标。

```Swift
var cache = Cache()
let profile = cache.profile
```

现在，我们可以更方便地访问 Cache 类型的配置文件数据。我们的 API 的使用者可能会认为配置文件是 Cache 类型的属性，但事实并非如此。

此特性完全在运行时工作，并利用了在点符号后键入的任何属性名称来访问 Cache 类型的下标，该下标具有 dynamicMember 参数。

整个逻辑在运行时运行，编译期间的结果是不确定的。在运行时，您完全可以决定应该从下标返回哪些数据以及如何处理 dynamicMember 参数。

## 使用 KeyPath 的编译时安全性

我们唯一能找到的缺点是缺乏编译时安全性。我们可以将 Cache 类型视为代码中键入的任何属性名称。幸运的是，`@dynamicMemberLookup` 下标的参数不仅可以是 String 类型，还可以是 KeyPath 类型。

```Swift
@dynamicMemberLookup
final class Store\<State, Action>: ObservableObject {
typealias ReduceFunction = (State, Action) -> State

    @Published private var state: State
    private let reduce: ReduceFunction

    init(
        initialState state: State,
        reduce: @escaping ReduceFunction
    ) {
        self.state = state
        self.reduce = reduce
    }

    subscript<T>(dynamicMember keyPath: KeyPath<State, T>) -> T {
        state[keyPath: keyPath]
    }

    func send(_ action: Action) {
        state = reduce(state, action)
    }

}
```

如上例所示，我们定义了接受强类型 KeyPath 实例的 dynamicMember 参数下标。在这种情况下，我们允许 State 类型的 KeyPath，这有助于我们获得编译时安全性。因为每当我们传递与 State 类型无关的错误 KeyPath 时，编译器都会显示错误。

```Swift
struct State {
var products: \[String] = \[]
var isLoading = false
}

enum Action {
case fetch
}

let store: Store\<State, Action> = .init(initialState: .init()) { state, action in
var state = state
switch action {
case .fetch:
state.isLoading = true
}
return state
}

print(store.isLoading)
print(store.products)
print(store.favorites) // Compiler error
```

在上例中，我们通过接受 KeyPath 的下标访问 Store 的私有 state 属性。这看起来与前面的例子类似，但在这种情况下，只要您尝试访问 State 类型的不可用属性，编译器就会显示错误。

## 总结

今天我们学习了如何使用 `@dynamicMemberLookup` 属性改进特定类型的 API。虽然并不是每个类型都需要它，但您可以谨慎使用它来改善 API。

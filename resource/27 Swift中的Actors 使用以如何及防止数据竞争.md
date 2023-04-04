
## 前言


Swift Actors 是Swift 5.5中的新内容，也是WWDC 2021上并发重大变化的一部分。在有 actors 之前，数据竞争是一个常见的意外情况。因此，在我们深入研究具有隔离和非隔离访问的行为体之前，最好先了解什么是[数据竞争](https://www.avanderlee.com/swift/thread-sanitizer-data-races/#what-are-data-races)，并了解当前你如何[解决这些问题](https://www.avanderlee.com/swift/thread-sanitizer-data-races/#using-the-thread-sanitizer-to-detect-data-races)。

Swift 中的 Actors 旨在完全解决数据竞争问题，但重要的是要明白，很可能还是会遇到数据竞争。本文将介绍 Actors 是如何工作的，以及你如何在你的项目中使用它们。

## 什么是 Actors?

Swift 中的 Actor 并不新鲜：它们受到 [Actor Model](https://en.wikipedia.org/wiki/Actor_model) 的启发，该模型将行为视为并发计算的通用基元。然后，[SE-0306](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)提案引入了 Actor，并解释了它们解决了哪些问题：数据竞争。


当多个线程在没有同步的情况下访问同一内存，并且至少有一个访问是写的时候，就会发生数据竞争。数据竞争会导致不可预测的行为、内存损坏、不稳定的测试和奇怪的崩溃。你可能会遇到无法解决的崩溃，因为你不知道它们何时发生，如何重现它们，或者如何根据理论来修复它们。我的文章[Thread Sanitizer explained: Data Races in Swift](https://www.avanderlee.com/swift/thread-sanitizer-data-races/)深入解释了如何解决、发现和修复数据竞争。


Swift 中的 Actors 可以保护他们的状态免受数据竞争的影响，并且使用它们可以让编译器在编写应用程序时为我们提供有用的反馈。此外，Swift 编译器可以静态地强制执行 Actors 附带的限制，并防止对可变数据的并发访问。

您可以使用 `actor` 关键字定义一个 Actor，就像您使用类或结构体一样：

```swift
actor ChickenFeeder {
    let food = "worms"
    var numberOfEatingChickens: Int = 0
}
```

Actor 和其他 Swift 类型一样，它们也可以有初始化器、方法、属性和子标号，同时你也可以用协议和泛型来使用它们。此外，与结构体不同的是：当你定义的属性需要手动定义时，`actor` 需要自定义初始化器。最后，重要的是要认识到 `actor` 是引用类型。

### Actor 是引用类型，但与类相比仍然有所不同

Actor 是引用类型，简而言之，这意味着副本引用的是同一块数据。因此，修改副本也会修改原始实例，因为它们指向同一个共享实例。你可以在我的文章[Swift中的Struct与class的区别](https://www.avanderlee.com/swift/struct-class-differences/)中了解更多这方面的信息。

**然而，与类相比，Actor 有一个重要的区别：他们不支持继承。**

![Swift中的Actor几乎和类一样，但不支持继承。](https://www.avanderlee.com/wp-content/uploads/2021/06/swift_actors_data_races.png)

Swift中的Actor几乎和类一样，但不支持继承。

不支持继承意味着不需要像便利初始化器和必要初始化器、重写、类成员或`open`和`final`语句等功能。

**然而，最大的区别是由 Actor 的主要职责决定的，即隔离对数据的访问。**

## Actors 如何通过同步来防止数据竞争

Actor 通过创建对其隔离数据的同步访问来防止数据竞争。在Actors之前，我们会使用各种锁来创建相同的结果。这种锁的一个例子是并发调度队列与处理写访问的屏障相结合。受我在[Concurrent vs. Serial DispatchQueue: Concurrency in Swift explained](https://www.avanderlee.com/swift/concurrent-serial-dispatchqueue/)一文中解释的技术的启发。我将向你展示使用 Actor 的前后对比。

在 Actor 之前，我们会创建一个线程安全的小鸡喂食器，如下所示：

```swift
final class ChickenFeederWithQueue {
    let food = "worms"
    
    /// 私有支持属性和计算属性的组合允许同步访问。
    private var _numberOfEatingChickens: Int = 0
    var numberOfEatingChickens: Int {
        queue.sync {
            _numberOfEatingChickens
        }
    }
    
    /// 一个并发的队列，允许同时进行多次读取。
    private var queue = DispatchQueue(label: "chicken.feeder.queue", attributes: .concurrent)
    
    func chickenStartsEating() {
        /// 使用栅栏阻止写入时的读取
        queue.sync(flags: .barrier) {
            _numberOfEatingChickens += 1
        }
    }
    
    func chickenStopsEating() {
        /// 使用栅栏阻止写入时的读取
        queue.sync(flags: .barrier) {
            _numberOfEatingChickens -= 1
        }
    }
}
```

正如你所看到的，这里有相当多的代码需要维护。在访问非线程安全的数据时，我们必须仔细考虑自己使用队列的问题。需要一个栅栏标志来停止读取并允许写入。再一次，我们需要自己来处理这个问题，因为编译器并不强制执行它。最后，我们在这里使用了一个`DispatchQueue`，但是经常有围绕着哪个锁是最好的争论。

为了看清这一点，我们可以使用我们先前定义的 Actor 小鸡喂食器来实现上述例子：

```swift
actor ChickenFeeder {
    let food = "worms"
    var numberOfEatingChickens: Int = 0
    
    func chickenStartsEating() {
        numberOfEatingChickens += 1
    }
    
    func chickenStopsEating() {
        numberOfEatingChickens -= 1
    }
}
```

你会注意到的第一件事是，这个实例更简单，更容易阅读。所有与同步访问有关的逻辑都被隐藏在Swift标准库中的实现细节里。然而，最有趣的部分发生在我们试图使用或读取任何可变属性和方法的时候:

![Methods in Actors are isolated for synchronized access.](https://www.avanderlee.com/wp-content/uploads/2021/06/actor_isolated_methods.png)

Actors中的方法是隔离的，以便同步访问。

在访问可变属性 `numberOfEatingChickens`时，也会发生同样的情况：

![Mutable properties can only be accessed from within the Actor.](https://www.avanderlee.com/wp-content/uploads/2021/06/actor_isolated_mutable_property.png)

可变的属性只能从Actor内部访问。



然而，我们被允许编写以下代码：

```swift
let feeder = ChickenFeeder()
print(feeder.food) 
```

我们的喂食器上的`food`属性是不可变的，因此是线程安全的。没有数据竞争的风险，因为在读取过程中，它的值不能从另一个线程中改变。

然而，我们的其他方法和属性会改变一个引用类型的可变状态。为了防止数据竞争，需要同步访问，允许按顺序访问。

## 使用async/await从 Actors 访问数据

在 Swift 中，我们可以通过使用 `await `关键字来创建异步访问：

````swift
let feeder = ChickenFeeder()
await feeder.chickenStartsEating()
print(await feeder.numberOfEatingChickens) // Prints: 1 
````

## 防止不必要的暂停

在上面的例子中，我们正在访问我们 Actor 的两个不同部分。首先，我们更新吃食的鸡的数量，然后我们执行另一个异步任务，打印出吃食的鸡的数量。每个`await`都会导致你的代码暂停，以等待访问。在这种情况下，有两个暂停是有意义的，因为两部分其实没有什么共同点。然而，你需要考虑到可能有另一个线程在等待调用`chickenStartsEating`，这可能会导致在我们打印出结果的时候有两只吃食的鸡。


为了更好地理解这个概念，让我们来看看这样的情况：你想把操作合并到一个方法中，以防止额外的暂停。例如，设想在我们的 `actor` 中有一个通知方法，通知观察者有一只新的鸡开始吃东西：

```swift
extension ChickenFeeder {
    func notifyObservers() {
        NotificationCenter.default.post(name: NSNotification.Name("chicken.started.eating"), object: numberOfEatingChickens)
    }
} 
```

我们可以通过使用 `await` 两次来使用此代码：

```swift
let feeder = ChickenFeeder()
await feeder.chickenStartsEating()
await feeder.notifyObservers() 
```

然而，这可能会导致两个暂停点，每个`await`都有一个。相反，我们可以通过从`chickenStartsEating`中调用`notifyObservers`方法来优化这段代码：

```swift
func chickenStartsEating() {
    numberOfEatingChickens += 1
    notifyObservers()
} 
```

由于我们已经在Actor内有了同步的访问，我们不需要另一个等待。这些都是需要考虑的重要改进，因为它们可能会对性能产生影响。

## Actor 内的非隔离(nonisolated)访问

了解 Actor 内部的隔离概念很重要。上面的例子已经展示了如何通过要求使用 `await` 从外部参与者实例同步访问。但是，如果您仔细观察，您可能已经注意到我们的 `notifyObservers` 方法不需要使用 `await` 来访问我们的可变属性 `numberOfEatingChickens`。

当访问 Actor 中的隔离方法时，你基本上可以访问任何其他需要同步访问的属性或方法。因此，你基本上是在重复使用你给定的访问，以获得最大的收益。

然而，在有些情况下，你知道不需要有隔离的访问。actor 中的方法默认是隔离的。下面的方法只访问我们的不可变的属性`food`，但仍然需要`await`访问它:

```swift
let feeder = ChickenFeeder()
await feeder.printWhatChickensAreEating() 
```

这很奇怪，因为我们知道，我们不访问任何需要同步访问的东西。[SE-0313](https://github.com/apple/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md)的引入正是为了解决这个问题。我们可以用`nonisolated`关键字标记我们的方法，告诉 Swift编 译器我们的方法没有访问任何隔离数据：

```swift
extension ChickenFeeder {
    nonisolated func printWhatChickensAreEating() {
        print("Chickens are eating \(food)")
    }
}

let feeder = ChickenFeeder()
feeder.printWhatChickensAreEating() 
```

注意，你也可以对计算的属性使用`nonisolated`的关键字，这对实现`CustomStringConvertible`等协议很有帮助：

```swift
extension ChickenFeeder: CustomStringConvertible {   
    nonisolated var description: String {     
        "A chicken feeder feeding \(food)"   
    } 
}
```

然而，在不可变的属性上定义它们是不需要的，因为编译器会告诉你：

![Marking immutable properties nonisolated is redundant.](https://www.avanderlee.com/wp-content/uploads/2021/06/nonisolated_properties.png)

将不可变的属性标记为 nonisolated 是多余的。

## 为什么在使用 Actors 时仍会出现数据竞争？

当在你的代码中持续使用 Actors 时，你肯定会降低遇到数据竞争的风险。创建同步访问可以防止与数据竞争有关的奇怪崩溃。然而，你显然需要持续地使用它们来防止你的应用程序中出现数据竞争。

在你的代码中仍然可能出现竞争条件，但可能不再导致异常。认识到这一点很重要，因为Actors 毕竟被宣扬为可以解决一切问题的工具。例如，想象一下两个线程使用 `await`正确地访问我们的 Actor 的数据：

```swift
queueOne.async {
    await feeder.chickenStartsEating()
}
queueTwo.async {
    print(await feeder.numberOfEatingChickens)
} 
```

这里的竞争条件定义为：“哪个线程将首先开始隔离访问？”。所以基本上有两种结果：

- 队列一在先，增加吃食的鸡的数量。队列二将打印：1
- 队列二在先，打印出吃食的鸡的数量，该数量仍为：0


这里的不同之处在于我们在修改数据时不再访问数据。如果没有同步访问，在某些情况下这可能会导致无法预料的行为。

## 继续你的Swift并发之旅

并发更改不仅仅是 async-await，还包括许多您可以在代码中受益的新功能。所以当你在使用它的时候，为什么不深入研究其他并发特性呢？

- [Sendable 和 @Sendable 闭包代码实例详解](https://mp.weixin.qq.com/s/IA9CgMjZf63_RFwNBB9QqQ)


- [Swift AsyncSequence — 代码实例详解](https://mp.weixin.qq.com/s/7HuYcMFCjqEhRHWlPc3ydA)


- [Swift AsyncThrowingStream 和 AsyncStream 代码实例详解](https://mp.weixin.qq.com/s/j42yzmsOMNAsq3bRmUzUEA)


- [Swift 中的 async/await ——代码实例详解](https://mp.weixin.qq.com/s/TL1LPvVdhSFxrYCFWQ2LfQ)


## 结论

Swift Actors 解决了用 Swift 编写的应用程序中常见的数据竞争问题。可变数据是同步访问的，这确保了它是安全的。我们还没有介绍 `MainActor` 实例，它本身就是一个主题。我将确保在以后的文章中介绍这一点。希望您能够跟随并知道如何在您的应用程序中使用 Actor。

> 来自：[Actors in Swift: how to use and prevent data races](https://www.avanderlee.com/swift/actors/)
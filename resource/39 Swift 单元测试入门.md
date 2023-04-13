编程语言中的单元测试是为了确保编写的代码按预期工作。给定一个特定的输入，您希望代码带有一个特定的输出。通过测试您的代码，能够给您当前的重构和发布建立信心，因为您将能够确保代码在成功运行您的测试套件后按预期工作。



许多开发人员不编写单元测试，因为他们认为这会花费太多时间，有可能错过最后期限。在我看来，单元测试会让你在最后期限前完成更多工作，因为你会花更少的时间解决错误或为关键问题打补丁。

**这篇文章内不会涵盖 内存泄漏测试 或 为共享扩展编写 UI 测试，而是主要关注编写更好的单元测试。我还将分享帮助我开发更好、更稳定的应用程序的最佳实践。**



## 什么是单元测试

单元测试是运行和验证一段代码（称为“单元”）以确保其按预期运行并符合其设计的自动化测试。



单元测试在 Xcode 中有它们的 target，并使用 [XCTest 框架](https://developer.apple.com/documentation/xctest)编写。 `XCTestCase` 的子类包含要运行的测试方法，其中只有以 "test" 开头的方法才会被 Xcode 解析并允许运行。



例如，假设有一个字符串扩展方法将第一个字母大写：

```swift
extension String {
    func uppercasedFirst() -> String {
        let firstCharacter = prefix(1).capitalized
        let remainingCharacters = dropFirst().lowercased()
        return firstCharacter + remainingCharacters
    }
}
```

我们要确保 `uppercasedFirst() `方法按预期工作。如果我们给它一个输入 `antoine`，我们期望它输出 `Antoine`。我们可以使用` XCTAssertEqual` 方法为此方法编写单元测试：

```swift
final class StringExtensionsTests: XCTestCase {
    func testUppercaseFirst() {
        let input = "antoine"
        let expectedOutput = "Antoine"
        XCTAssertEqual(input.uppercasedFirst(), expectedOutput, "The String is not correctly capitalized.")
    }
}
```



如果我们的方法不再按预期工作(比如上面的扩展代码不小心被修改了)，Xcode 将使用我们提供的描述显示失败：

![单元测试失败，因为输入与预期输出不匹配。](https://upload-images.jianshu.io/upload_images/2955252-e2ea09af17709336.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 在 Swift 中编写单元测试

有多种方法可以测试相同的结果，但是当测试失败时它并不总是给出相同的反馈。以下提示可帮助您编写测试，通过从详细的失败消息中获益，帮助您更快地解决失败的测试。



### 命名测试用例和方法

描述你的单元测试是很重要的，这样你就会明白测试试图验证什么。如果你不能想出一个简短的名字，那你可能测试了太多东西。一个好名字还可以帮助您更快地解决失败的测试。



要快速找到特定类的测试用例，建议使用相同的命名并结合 “test”。就像上面的例子一样，我们根据我们正在测试一组字符串扩展的事实命名了 `StringExtensionTests`。如果您正在测试`ContentViewModel` 实例，另一个示例可能是 `ContentViewModelTests`。



### 不要所有测试都使用 XCTAssert

许多场景都可以使用 `XCTAssert`，但当测试失败时会导致不同的结果。以下代码行都测试了完全相同的结果：

```swift
func testEmptyListOfUsers() {
    let viewModel = UsersViewModel(users: ["Ed", "Edd", "Eddy"])
    XCTAssert(viewModel.users.count == 0)
    XCTAssertTrue(viewModel.users.count == 0)
    XCTAssertEqual(viewModel.users.count, 0)
}
```

正如你所看到的，该方法使用了一个描述性的名字，告诉人们要测试一个空的用户列表。然而，我们定义的视图模型不是空的，因此，所有的断言都失败了。


![使用正确的断言可以帮助您更快地解决故障。](https://upload-images.jianshu.io/upload_images/2955252-99021cc17e03493c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


结果显示了为什么必须对验证类型使用正确的断言。 `XCTAssertEqual` 方法为我们提供了有关断言失败原因的更多上下文。这显示在红色错误和控制台日志中，可帮助您快速识别失败的测试。



### Setup and Teardown

多个测试方法中使用的参数可以定义为测试用例类中的属性。您可以使用 `setUp()` 方法为每个测试方法设置初始状态，并使用 `tearDown()` 方法进行清理。有多种设置和拆卸方法的变体供您选择，例如支持并发的变体或抛出变体，如果设置失败，您可以在其中提前使测试失败。



一个可以生成用户默认实例以用于单元测试的示例：

```swift
struct SearchQueryCache {
    var userDefaults: UserDefaults = .standard

    func storeQuery(_ query: String) {
        /// ...
    }
}

final class SearchQueryCacheTests: XCTestCase {

    private var userDefaults: UserDefaults!
    private var userDefaultsSuiteName: String!

    override func setUpWithError() throws {
        try super.setUpWithError()
        userDefaultsSuiteName = UUID().uuidString
        userDefaults = UserDefaults(suiteName: userDefaultsSuiteName)
    }

    override func tearDownWithError() throws {
        try super.tearDownWithError()
        userDefaults.removeSuite(named: userDefaultsSuiteName)
        userDefaults = nil
    }

    func testSearchQueryStoring() {
        /// 使用生成的用户默认值作为输入。
        let cache = SearchQueryCache(userDefaults: userDefaults)

        /// ... write the test
    }
}
```

这样做可以确保您不会操纵在模拟器上测试期间使用的标准用户默认值。其次，您将确保在测试开始时处于干净状态。我们使用了拆卸方法来删除用户默认套件并进行相应的清理。



### 抛出方法

和编写应用程序代码时一样，您也可以定义一个可抛出测试的方法。这允许您在测试中的方法抛出错误时使测试失败。例如，在测试 JSON 响应的解码时：

```swift
func testDecoding() throws {
    /// 当数据初始值设定项抛出错误时，测试将失败。
    let jsonData = try Data(contentsOf: URL(string: "user.json")!)

    /// `XCTAssertNoThrow` 可用于获取有关抛出的额外上下文
    XCTAssertNoThrow(try JSONDecoder().decode(User.self, from: jsonData))
}
```

当在任何进一步的测试执行中不需要 throwing 方法的结果时，可以使用 `XCTAssertNoThrow` 方法。您应该使用 `XCTAssertThrowsError` 方法来匹配预期的错误类型。例如，您可以为证书密钥验证程序编写测试：

```swift
struct LicenseValidator {
    enum Error: Swift.Error {
        case emptyLicenseKey
    }

    func validate(licenseKey: String) throws {
        guard !licenseKey.isEmpty else {
            throw Error.emptyLicenseKey
        }
    }
}

class LicenseValidatorTests: XCTestCase {
    let validator = LicenseValidator()

    func testThrowingEmptyLicenseKeyError() {
        XCTAssertThrowsError(try validator.validate(licenseKey: ""), "An empty license key error should be thrown") { error in
            /// 我们确保预期的错误被抛出。
            XCTAssertEqual(error as? LicenseValidator.Error, .emptyLicenseKey)
        }
    }

    func testNotThrowingLicenseErrorForNonEmptyKey() {
        XCTAssertNoThrow(try validator.validate(licenseKey: "XXXX-XXXX-XXXX-XXXX"), "Non-empty license key should pass")
    }
}
```



### 可选值解包

`XCTUnwrap` 方法最适合用于抛出测试，因为它是一个抛出断言：

```swift
func testFirstNameNotEmpty() throws {
    let viewModel = UsersViewModel(users: ["Antoine", "Maaike", "Jaap"])

    let firstName =  try XCTUnwrap(viewModel.users.first)
    XCTAssertFalse(firstName.isEmpty)
}
```

`XCTUnwrap` 断言可选变量的值不为 `nil`，如果断言成功则返回它的值。它会阻止您编写 `XCTAssertNotNil` 并结合解包或处理其余测试代码的条件链接。我鼓励您阅读我的文章 [《如何使用 XCTest 在 Swift 中测试可选值》](https://www.avanderlee.com/swift/test-optionals-xctest/)以了解更多详细信息。



## 在 Xcode 中运行单元测试

编写测试后，就该运行它们了。通过以下提示，这将变得更有效率。



### 使用测试三角形

您可以使用前导三角形运行单个测试或一组测试：


![前导三角形可用于运行单个或一组测试。](https://upload-images.jianshu.io/upload_images/2955252-8c88cdde322fe065.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


根据最新的测试运行结果，同一方块显示红色或绿色。



### 重新运行最新的测试

使用以下命令重新运行上次运行测试：

`⌃ Control + ⌥ Option + ⌘ Command + G`.

上面的快捷方式可能是我最常用的快捷方式之一，因为它可以帮助我在对失败测试实施修复后快速重新运行测试。



### 运行测试组合

使用 CTRL 或 SHIFT 选择要运行的测试，右键单击并选择“Run X Test Methods”。

![运行测试组合](https://upload-images.jianshu.io/upload_images/2955252-a5e5035bc45e90d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 在测试导航器中应用过滤器

测试导航器底部的过滤栏允许您缩小测试概览范围。

![测试导航器过滤栏](https://upload-images.jianshu.io/upload_images/2955252-dae6d592b08d509b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 使用搜索字段根据名称搜索特定测试
- 仅显示当前所选方案的测试。如果您有多个测试方案，这将很有用。
- 只显示失败的测试。这将帮助您快速找到失败的测试。

### 在侧边栏中启用覆盖

![在编辑器中启用代码覆盖](https://upload-images.jianshu.io/upload_images/2955252-50a9a28c44816dd4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

测试迭代计数向您显示在上次运行测试期间是否命中了特定代码段。


![命中提示](https://upload-images.jianshu.io/upload_images/2955252-41e0dc1c79fc0fc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


它显示了迭代次数（在上面的示例中为 3），一段代码在到达时变为绿色。当一段代码是红色时，这意味着它在上次运行的测试中没有被覆盖。



## 编写单元测试时的心态

你的心态是编写高质量单元测试的一个很好的起点。通过一些基本原则，您可以确保工作效率、保持专注并编写您的应用程序最需要的测试。



### 您的测试代码与您的应用程序代码一样重要

在深入探讨实用技巧之后，我想介绍一种必要的心态。就像编写应用程序代码一样，您应该尽最大努力编写高质量的测试代码。



考虑重用代码、使用协议、在多个测试中使用时定义属性，并确保您的测试清理所有创建的数据。这将使您的单元测试更易于维护，并防止不稳定和奇怪的测试失败。如果您不熟悉片状的测试，我鼓励您阅读我的文章 [Flaky tests resolving using Test Repetitions in Xcode](https://www.avanderlee.com/debugging/flaky-tests-test-repetitions/)。



### 100% 的代码覆盖率不应该是你的目标

尽管它是很多人的目标，但 100% 的覆盖率不应该是您编写测试时的主要目标。一个很好的开始是确保至少测试您最关键的业务逻辑。覆盖率达到 100% 可能会很耗时，而收益并不总是那么显著。并且达到100%，也意味着可能需要付出很大的努力。



最重要的是，100% 的覆盖率可能会产生误导。上面的单元测试示例覆盖了所有方法，覆盖率为 100%。但是，它并没有测试所有场景，因为它只测试了一个非空数组。同时，也可能存在空数组的情况，其中 `hasUsers` 属性应该返回 false。

![可以通过编辑 Scheme 来启用单元测试代码覆盖率](https://upload-images.jianshu.io/upload_images/2955252-63940dc7afd0e48d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


您可以从 Scheme 设置窗口启用测试覆盖率。这个窗口可以通过`Product ➞ Scheme ➞ Edit Scheme`打开。



### 在修复错误之前编写测试

跳到一个错误上并尽快修复它是很诱人的。虽然这很好，但如果您可以防止将来再次出现相同的错误，那就更好了。通过在修复 bug 之前编写单元测试，可以确保相同的 bug 不会再次发生。将其视为“测试驱动的错误修复”，从现在开始也称为 TDBF 。



其次，您可以开始编写修复程序并运行新的单元测试来验证修复程序是否有效。此技术比运行模拟器来验证您的修复是否有效要快。



## 结论

编写定性的单元测试是开发人员的基本技能。将能够对您的代码库建立信心，确保您在新版本发布之前没有破坏任何东西。使用正确的断言，您可以更快地解决失败的测试。确保至少测试关键业务代码并避免达到 100% 的代码覆盖率。

> 来自：[Getting started with Unit Tests in Swift](https://www.avanderlee.com/swift/unit-tests-best-practices/)

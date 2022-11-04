# Flutter 多引擎渲染，在稿定 App 的实践（三）：躺坑篇

这一篇会把笔者总结踩过的坑放出来给大家参考，能给要走 Flutter 多引擎之坑的同学一些帮助，不要轻易放弃，总是能走出一条路来。

## FAQ

### A. Flutter 为什么需要升级到 ~~2.5.3 2.10.5~~ 3.0.5

先是在“稿定设计 APP”中接入 FlutterEngineGroup 发现，编译没有问题，但就是死活无法正常显示 FlutterView，翻查了大量资料（也没什么有用的资料），跟 Demo 工程对比等方式，耗时2天，最后只能锁定在 flutter 版本或者 flutter_boost 的问题上，死马当作活马医，直接硬干升级 flutter 到当时最新版（2.10.2）及相关组件，发现正常 ...

再就是在打包 flutter Android 时又发现， flutter_boost 报错，从 github issues 了解到，flutter_boost 并没去支持 flutter 2.10.x，且还有闪白屏问题。根据 issues 建议，2.8+版本上存在 Release 包不可用的问题，推荐降低到 2.5.3，这才总算是从 FlutterEngineGroup 初步落地的可行性坑中爬了出来。

===========

最新，因为 2.5.3 同时布局多个 Engine，导致会发生 ANR 的现象，在寻找解决方案无果的情况下，尝试升级到最新版本 Flutter， 2.10.5 ，结果正常

===========

Flutter 版本 2.5.3+ ～ 3.0.5- 在 iOS 上会有压缩指针释放导致的崩溃问题，所以建议还是升级到 3.0.5 及其以上

### B. 官方 Demo 的坑

官方 Samples 地址： <https://github.com/flutter/samples>

FluterEngineGroup: <https://github.com/flutter/samples/tree/master/add_to_app/multiple_flutters>

Pigeon: <https://github.com/flutter/samples/tree/master/add_to_app/books>

官方 Demo 最大的坑就是 Demo 都是可用的 ...

Debug 包可用，Release 包会报 engine 配置找不到的白黑屏问题

```objc
- (id)makeEngineWithEntrypoint:(nullable NSString *)entrypoint libraryURI:(nullable NSString *)libraryURI;
```

原因是 libraryURI 参数为 nil，在 Release 下无法索引到 entrypoint .

libraryURI 是传你当前入口的包名 + dart，以上一篇的 switch 为例:

```objc
entrypoint:@"componentSwitch"
libraryURI:@"package:fgui/ui_components.dart"];
```

### C. 如何 Flutter 内部控制 Size

外部约束必须提供宽高才可正确显示 FlutterView。如果想要做 FlutterView 基于内部自适应，就需要通过 Flutter 传给 Native 宽高后再确定外部约束。

但又引发一个问题，外部如果约束没有宽高，则不会渲染 FlutterView。 这就巧妙的用了 0.1 这个默认约束条件，当然已经内置在 ComponentAPI 中，外部调用无需关心。

### D. Android 可行性验证上走过的坑

- top-level 找不到，渲染白屏，问题最后排查到 debug 包正常，release 包不正常。最后找到该 issue（<https://github.com/flutter/flutter/issues/91841>），这是 flutter/dart 的 bug，在 2.5.3 上可以通过*指定入口所在文件*解决，在 2.8.0 以上版本建议退回 2.5.3 (手动狗头)。
- 在使用 flutter debug 包情况下，每个引擎会多占用 100 M 内存，且在同时渲染 10 个引擎的情况下，会导致页面卡死。（通过 <https://flutter.cn/docs/development/add-to-app/multiple-flutters> 官网说明，JIT 模式下会有内存泄漏问题，推荐使用 AOT release 包）。
- 在 release 包情况下，for 循环同时增加 10 个 FlutterView，直接就 OOM 崩溃 ... 最后排查结果，如果 for 中加一个 delay(1)，就显示正常且内存占用也正常，怀疑是 Flutter 本身的 Bug，从 issues 中了解到可能是 dart 的 observe 有问题。这个问题在 Flutter 后续版本修复了，具体没有细追究，大概是 2.8+ 或者 2.10+。

### E. 打包以及依赖

由于 Flutter 只有一个 main() 入口，所以做不到页面和组件化分开打包引用，这就导致出现了一个依赖问题，我们的 Flutter 包是按项目打包的，那去使用组件的模块很多都是通用模块，不能去依赖 Flutter 包。

最终的处理方案是反射解耦，双端生成的调用类不再依赖 Pigeon 生成的 API 类，而是通过反射的形式去调用，外部调用者只需引用 FGUIComponentAPI 模块，即可使用 Flutter UI。减少了直接依赖也就减少了构建时长。同时，FGUIComponentAPI 是自动生成的，所以不会存在维护上的问题。

### F. 文字国际化

由于 FlutterEngineGroup 不是传统的 main() 入口，也不能继承 MaterialApp 或者 WidgetApp ，所以 Flutter 本身的国际化方案并不适用。

最终是做了国际化内置的形式，由源生宿主在创建 FlutterView 时通过 MessageChannel 通知 Flutter 当前是什么语言环境，然后在有限复用现有的 intl 生成国际化方式，解决国际化问题。

### G. Flutter-Debug <> Flutter-Release

被摧残过才明白，这俩就是不同的物种，生殖隔离的那种

除非是非要  `attach to Flutter Progress` ，开发调试上只建议使用 Flutter-Release

#### a. Flutter-Debug 内存泄漏

以 iOS 为例：

> 真机 + Flutter-Release 模式 = 没有问题，个人观测基本 1 M / Engine (官方说 180K / Engine，民间测试 1.33M / Engine)
>
> 真机 + Flutter-Debug 模式 = 内存 100 M / Engine

内存问题在 Flutter Debug 模式下是无解的，这个是因为 Flutter 调试功能会导致内存泄漏和增大问题，是 dart 本身的问题且社区上看暂时没有解决方案。

#### b. Flutter-Release 存在调用陷阱

背景：

同时布局多个 FlutterView

在 Flutter-Debug 下除了内存加载问题，展示及操作都正常

在 Flutter-Release 下发现会产生主线程 pThread 锁死等待，界面卡死现象

分析：

第一步，经大量测试发现，先去单独加载一个 FlutterView，然后再同时布局多个 FlutterView，结果正常。（比如先进入下设置页面，FlutterEngineGroup 创建的还是 flutter_boost 创建的都可以）

> 初步怀疑是 Flutter 机制的问题，在复用 isolate 时，如果还未创建 isolate，会去走创建流程，但如果外部是循环加载，而创建 isolate 的过程不是线程安全的（调用了还未创建完成的方法），导致某一段代码出现了死锁。

第二步，想到另一个页面也是同时布局多个 FlutterView，但在未先单独加载一个 FlutterView 也可以正常使用，对比代码发现：

是因为布局时机上不同：

```objc
- (void)init ... {
    super
    ...
    [self setupOneFlutterView];
}

// 引发问题的代码，在布局时再去另一个 FlutterView
- (void)layoutSubviews {
    [super layoutSubviews];
    [self setupTwoFlutterView];
}

// 但如果只有一个 FlutterView 在 layoutSubviews 上布局，又是可以的
```

结论：

根本原因是 Flutter 自身 C++ 代码的问题，但真的是因为用的人太少，大部分卡在 Demo 都没玩过去，所以也没人提到这个。

类似的，Android 也有这问题，多个同时布局会导致 FlutterJNI 死锁，界面无响应。这个可能是 Flutter 2.5.3 的 Bug，反正官方 issues 就一句话，升级最新版 → issue closed。

### I. FlutterView 阴影

需要注意是，如果开发的 Flutter 组件需要显示阴影，Native 上的宽高约束需要包括阴影的宽高，超过 FlutterView 的 Size 就会被 Native 截掉，会导致样式上问题。

### J. Flutter 手势失效

在 iOS 上，由于 Flutter 是使用更底层的 touch 事件，响应优先级比手势低，如果布局上存在 Native 手势，就会被手势拦截，产生 FlutterView 无响应的问题。

临时解决方式，iOS 可以在外部源生手势上增加 cancelsTouchesInView = NO (default = YES)，让 touch 事件生效。

最终解决方式，FGUIComponentAPI 提供了点击、滑动手势竞争者，来保证 FlutterView 作为子视图能优先响应而不被父视图拦截。

### K. FlutterView 透明部分无法传递事件的问题

在 iOS 上，FlutterView 透明部分想要让底层接收到事件

控制 userInteractionEnabled=NO 可以暂时解决

但并不是一个最佳的实现方案吧，确实在 FlutterView 和 NativeView 叠加的场景下，事件响应是一个比较麻烦的问题。

### M. Flutter 开发需要注意的 Root 不是一个 MaterialApp 会产生的问题

由于 Root 不是一个 MaterialApp，所以诸如 MediaQuery 等 API 都不可用。

当然，由于 ListView 有个要求，父类需要有 Directionality（这个只有在使用时才会报错，编译时不会报错）， MaterialApp 是有封装掉的。

解决方式，这个生成模版时，根节点默认已为 Directionality。

可能还有更多类似的问题，需要注意。

场景持续更新：

1. [flutter_easyrefresh](https://pub.flutter-io.cn/packages/flutter_easyrefresh)（Third Party） 不可用，因为它的 footer 使用了 MediaQuery 来做 safeArea 判断。可选 [pull_to_refresh](https://pub.flutter-io.cn/packages/pull_to_refresh) 「但它没再更新了，不支持 Flutter 3.0+ 语法，可以换 pull_to_refresh_flutter3 [手动狗头]」。

### N. Flutter 第三方库选择需谨慎

由 M 问题拓展出一个新的问题：如果第三方库是一个源生混合型插件，通过 plugin 跟 Native 交互的，也不适合在多引擎场景下使用。

1. 因没有去注册 plugin，所以第三方库无法获取到 Native 结果，导致异常。这已持 plugin 注册，但要小心不要滥用。因为以前使用方式下，plugin 不释放也没什么问题，毕竟只有一个 FlutterEngine。但现在多引擎下，注册的 plugin 必须是内存安全可释放的，着重注意出现循环引用。
2. 但也会存在多引擎间消息不可控的问题 <https://developer.aliyun.com/ask/388217>。

### Q. 慎用 Timer

Flutter Timer 在 iOS 会通过 dart:io EventHandler 线程来 IO 通信，如果频繁的 Timer 或者存在多个 Timer 会导致频繁 IO 结果就是 CPU 占用过高。如果非要使用，那尽量不要使用周期性任务。

有兴趣的同学可以去搜一下 Flutter Timer 在各端上的实现原理。

### I. flutter_cache_manager 的使用误区

包括好评 100% 的 [cached_network_image](https://pub.dev/packages/cached_network_image) 都是基于 flutter_cache_manager 来做资源缓存。它的设计跟 SDWebImage 相同，也分为硬盘缓存（sqlite 做索引）、内存缓存。但问题就是因为 Flutter 自身不具备 sqlite、文件存储的能力，其实都是通过 Bridge 来跟 Native 交互的，这就导致从硬盘加载资源的效率（sqlite 查询地址 → 地址加载资源）比不上源生。

所以对于需要常驻的资源最好由 dart 持有，一旦被释放，内存持有释放的也特别快（据测试 20 多秒就被回收了）。

再从硬盘重新加载就会有短暂延迟，不符合 UI 交互效果。

### S. sqlite 使用需谨慎

背景是上线前测试发现，部分 Android 设备在第一次安装后出现图片展示失败的问题，但重开后就又正常的。排查上，也并没触发图片加载失败的日志。

最后，查到可疑点

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/656eb1b5757742d98843fe31d59a3b9e~tplv-k3u1fbpfcp-watermark.image?)

锁定问题，是在多引擎模式下使用 [cached_network_image](https://pub.dev/packages/cached_network_image) 导致。

细究原因，

[cached_network_image](https://pub.dev/packages/cached_network_image) ← flutter_cache_manager ← sqflite ，在 iOS / Android 上缓存的图片路径是用的 sqlite 实现的，而 sqlite 在多引擎模式下被多次同时访问导致出现 lock 的情况。

这也说明当下 pub 库中的插件大都是在单引擎模式下设计出来的，在多引擎下确实存在多种陷阱。

但问题还是很好处理，flutter_cache_manager 提供了 cachekey 字段，对于需同时做缓存的多引擎资源，使用不同的 cachekey 来区分成多个 DB 索引库。

也思考下 iOS 为什么不会出现这个问题，因为 iOS FlutterEngineGroup 设计上，一个 Group 中多个引擎都只使用同一个 iO 线程、raster 线程，所以对 sqlite 来说没有产生并发问题。

## 后续

FGUIComponentAPI 可能也有同学感兴趣是个什么，并没有什么高大上的原理，其实质是一个模版代码处理，语言的话，笔者用的 ruby，也可以换 python，这些脚本语言执行速度还是很可靠的，至少比 dart 做脚本好很多。

放一下目录结构吧，可以看到整个 fgui_component_api 就是 ruby 做的脚本执行文件

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9e6c2ef8fa54b4b9446b8a6671283dd~tplv-k3u1fbpfcp-watermark.image?)

有时间会整理下代码，放出来给大家参考。

> 作者投稿
> 
> 来源：[Flutter 多引擎渲染，在稿定 App 的实践](https://juejin.cn/post/7130811413840429093)
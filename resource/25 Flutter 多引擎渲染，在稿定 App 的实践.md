# Flutter 多引擎渲染，在稿定 App 的实践

发这篇文章的原因主要是关于 [multiple-flutters](<https://flutter.cn/docs/development/add-to-app/multiple-flutters>) Flutter 多引擎的介绍也好，实践也好，可参考的资源实在太少，包括官方的 issues 也没很多有价值的信息，前几个月确实在坑的泥潭里死去活来。但好在已经走出了一条羊肠小道，可供大家参考。

对于 Flutter 多引擎的优劣，笔者在这里不多做介绍，只说最重要的一点：如果有 Native + Flutter 同一页面混合布局的需求（UI 一致性 / 降本增效），但又不能整个 App 或者整个页面替换成 Flutter 的，可以考虑使用 FlutterEngineGroup（multiple-flutters）。

闲话少说，先看效果。

## APP 展示

![1660267286030.jpg](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b783d937daef45fa8bdafbca059b96d2~tplv-k3u1fbpfcp-watermark.image?)

如上图红框处，即为4个不同引擎的 FlutterView，绘制在同一个 Native 布局中。

篇幅有限，就不发视频了，有兴趣的同学可以下载 “稿定设计” 来看下效果（不过还在 AB 放量阶段，不一定能看到新版模版页哈～）。

## 为什么市面多引擎用的人那么少？

> multiple-flutters 绝对是 Flutter 的坑中之王

首先，Flutter 版本至少升级到 2.10+，才能有初步的 iOS / Android 多引擎同时布局的可用性。
但建议升级到 Flutter 3+ ，2.5.3 ～ 2.10.5 版本中，iOS 有内存崩溃风险，详细原因可以见同事发的这篇 [解决 Flutter 引起的 iOS 内存崩溃问题](https://juejin.cn/post/7123765829929762847)。

> 第一次渡劫历程：
>  
> 先是接入 FlutterEngineGroup 时发现，编译没有问题，但就是死活无法正常显示 FlutterView，翻查了大量资料（也没什么有用的资料），跟 Flutter 官方 Demo 对比等方式，耗时2天，最后只能锁定在 Flutter 版本或者 flutter_boost 的问题上，死马当作活马医，直接硬干升级 Flutter 到最新版（2.10.2）及相关依赖升级，发现 Debug 正常了 ...
>  
> 再就是在打包 flutter Android 时又发现， flutter_boost 报错，从 github issues 了解到，flutter_boost 并没去支持 Flutter 2.10.x，且还有闪白屏问题。根据 issues 建议，2.8+版本上存在 Release 包不可用的问题，推荐降低到 2.5.3，这才总算是从 FlutterEngineGroup 初步落地的可行性坑中爬了出来。
>  
> 因为 2.5.3 同时布局多个 FlutterEngine （3 ～ 10 个不等），导致会发生 ANR 的现象，在寻找解决方案无果的情况下，尝试升级到当时最新版本 Flutter 2.10.5 ，结果正常了

这在升级过程中还遇到另一个问题，笔者公司项目里还有很多 flutter_boost 的实现，而 flutter_boost 由于某些原因（可以见他们的 issues） 不支持 Flutter 2.5.3 以上版本。那就还需 Fork 下 flutter_boost 进行修改才可正常使用。

### FlutterEngineGroup 离实用太远

- 缺乏内部固定布局方式，只能通过外部布局位置大小来让 FlutterView 自适应。
- 通信层极其繁琐，从有限的 Demo 中看出需三端各自实现 Bridge Channel。桥方法通过“字符串”作为对应类型，导致个性化开发维护成本非常高。
- 应用场景狭窄，多 FlutterEngine 间只能通过 Native 交互通信。
- Flutter Debug 模式下多引擎 = 内存炸裂，要用 Flutter Release 才可以稳定正常到官方描述的 180K / Engine 的内存占用效果

## 我们是怎么做的

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c8f71ba78fa74b718e3b5a50f5d2d877~tplv-k3u1fbpfcp-watermark.image?)

利用脚本开发了一套 FGUIComponentAPI 工具链来链接 Native 与 Flutter UI 的关系。

- 保证 Flutter 开发无感，对于 Flutter 来说，和通常一样开发 UI，并可以在独立调试中直接验证效果。
- 保证 Native 开发无感，对于 Native 来说，只是直接引用使用源生类，无需关心其中实现，开箱即用。
- 额外的带来的好处就是天然的 UI 单元测试，并且只要 Flutter 一端验证即可。

> 这里特别说一下，内置了官方推荐的 pigeon 插件来处理 model 传输的问题，但 pigeon 插件执行起来效率不高，越多的组件执行起来就越慢，所以后面又增加了文件对比跳过的功能来加速。后续考虑替换掉 pigeon，不用 dart 来实现，改用 python 就能解决效率问题。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a338910bc1141f293951aec79287ea9~tplv-k3u1fbpfcp-watermark.image?)

在开发过程上，笔者使用 YAML 来定义 UI 组件，通过 FGUIComponentAPI 多向生成各类代码及服务。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80b309b818144ae19eabc83add4f51af~tplv-k3u1fbpfcp-watermark.image?)

上图即为自动生成的开发文档，可以看到 Native 调用上是完全无感知的，右侧的预览页面也是天然使用 Flutter 跨端 Web 的能力，直接把 Flutter Example 输出在文档上。

### 还有多少坑

笔者也还在一步一踩。

比如市面上常见的 pub 也要慎用，特别是有跟 Native 交互的插件，基本上都没有考虑多引擎实现的。
举个例子，常用的 flutter_cache_manager，它因为使用了 sqlite 数据库做存储，在多引擎同时布局的情况下，Android 设备可能会出现数据库等待导致图片缓存写入/读取失败的问题（当然可以通过定义 cachedKey 来指定使用不同的 db 来解决）。这其实也不是第三方库的问题，而是多引擎市面真实使用的人太少的缘故，没有需求就没有市场。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/128f5575082d48e6a4dc46ba2be63567~tplv-k3u1fbpfcp-watermark.image?)

可以看到笔者已经快踩完整个字母表了 ... 手动狗头

篇幅有限，这里不展开说明了，如果有需求的同学可以下方评论，人数多的话单独开一篇来介绍如何优雅的避开其中的坑坑坑坑坑炕钪锟烫烫烫

## 后续

感谢大家厚爱，会逐步推出后续更详细的内容

> 作者投稿
> 
> 来源：[Flutter 多引擎渲染，在稿定 App 的实践](https://juejin.cn/post/7130811413840429093)
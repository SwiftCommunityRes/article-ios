# Flutter 多引擎渲染，在稿定 App 的实践（二）：原理篇

在这么偏僻的技术路线上，还是有蛮多读者和社区认可的。所以笔者会把这方案的原理详细介绍给大家，让大家能少走一些弯路。  

## 前言

先讲下对于 Flutter 开发架构的理解，大概存在这3种：

1. Flutter 为主开发的 APP。
2. Flutter 与 Native 容器混合型，页面可以是 Flutter，也可以是 Native，代表比如 flutter_boost。
3. FlutterEngineGroup 多引擎渲染，容器是 Native 提供，Flutter 只关心 View 部分即可。

这里不是比较各自的优劣，选型上只选择最适合的方式。

像笔者公司前期是用 flutter_boost 做页面容器混合型，但现在架构上的变化，会逐渐减少 Native 的实现，变为跨端架构，而纯 Flutter 并不满足于我们的开发，且从代码量上也不可能改为 Flutter 为主的 APP 架构。（dart 说实话也不是一个好的开发语言 ...）。

基于这个前提能选择的很少，Flutter 多引擎是实现跨端 UI 现在是最现实的方案而已。毕竟官方也是只有 Demo，甚至官方推荐的 [pigeon Demo](https://github.com/flutter/samples/tree/master/add_to_app/books) 也没和 [multiple_flutters Demo](https://github.com/flutter/samples/tree/master/add_to_app/multiple_flutters) 联系起来。
> 至于为什么不继续使用容器混合型开发？大家有没有感觉到 add_to_app 的方式开发调试起来也是蛮痛苦的，单元测试也不好做。而且要保持业务层不动的情况下，开发很多额外的 plugins 来支撑 UI，这个成本还是很高的。

## 实现及原理

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b84acb558a304f17ae02e07b77c82142~tplv-k3u1fbpfcp-watermark.image?)

整套方案实现下即为跨端 UI 组件化，如上图所示。

> 跨端 UI 组件化优势：
>
> 1. APP 双端 UI 一致性实现，并且可以部署为独立的 Web Demo，提前进行 UI 走查。
> 2. Flutter UI 组件独立开发调试，且只关心 API 定义，不关心具体实现。
> 3. 解决开发使用痛点，减少开发难度曲线，自动生成调用 ComponentAPI 给 Native 侧无感调用抹平开发使用成本。

下面会从开发流程的角度，逐步分析整套方案的实现关键点。

### 定义 UI 组件

组件定义采用 YAML 标准化语言定义

#### RULE 定义

| 定义         | 说明                                                          |
| ---------- | ----------------------------------------------------------- |
| name       | 组件名称                                                        |
| init       | 初始化数据 →   List<{ name(名称)、type(类型)、note(注释)、default(默认值) }> |
| options    | 额外配置 → { note(组件注释)、autolayout(是否是自动布局) }                   |
| properties | 组件属性 →  List<{ name(名称)、type(类型)、note(注释)、default(默认值) }>   |
| methods    | 提供对外方法 → List<{ name(名称)、note(注释)、inputs(入参 List) }>        |
| cb_methods | 提供回调方法 → 同上                                                 |
| classes    | 定义 Class → List< name(名称)、note(注释)、properties(属性 List)>     |

#### TYPE 支持

| Flutter（定义）     | iOS                        | Android         |
| --------------- | -------------------------- | --------------- |
| String          | NSString                   | String          |
| int/long/double | int/long/double            | Int/Long/Double |
| bool            | BOOL                       | Boolean         |
| Map             | NSDictionary               | Map             |
| List            | NSArray                    | List<\Object\>    |
| List\<Class\>     | NSArray\<id\<ClassProtocol\>\> | List\<Class\> |
| Image           | UIImage                    | Bitmap          |

* 后续有需要会继续补充：比如 Color、Class extends Class、Class use Class*

#### 示例

\* ui_components.yaml \*

> 以最开始开发的 Switch 组件为例（后续上都以它为例），定义如下：

```YAML
- name: Switch
  init:
    - { name: title, type: String, note: 标题, default: -- }
    - { name: textColor, type: String, note: 默认（关闭）文字颜色, default: "255,255,255,0.4" }
    - { name: textColorAtOn, type: String, note: 在开启时文字颜色, default: "255,255,255,1" }
  options:
    note: GUI 切换按钮组件
    autolayout: false
  properties:
    - { name: "on", type: bool, note: 是否开启, default: false }
```

### FGUIComponentAPI 生成 Flutter 开发套件

生成的调用类分为多个部分，.gitigore 即为自动生成的文件

文件结构如下：

```markdown
    FlutterProject/                         # Flutter 项目目录
      ↓ fgui/                           # Flutter GUI Kit 组件库
          ↓ ui_components.yaml          # 定义组件
          ↓ ui_components.dart          # 调用入口层（.gitigore）
          ↓ .api/                       # API 索引，自动生成，被用于 pigeon（.gitigore）
          ↓ lib/                        # lib
              ↓ .caches/                # 组件基类及 API 实例，自动生成（.gitigore）
                    ↓ switch.api.dart   # pigeon 生成类
                    ↓ switch.base.dart  # 组件基类，用于封装 api.dart 
              ↓ {switch}/{switch.dart}  # **进行组件开发**
```

#### 入口层（ui_components.dart）

```dart
@pragma('vm:entry-point')
void componentSwitch() => runApp(const fgui_switch.Switch());
```

多引擎的入口必须是 root 节点的方法，且必须实现 runApp。

#### API 索引（.api/）

```dart
// AUTO GENERATE API
//
// NOTES: 自动生成，无需手动修改. 

import 'package:pigeon/pigeon.dart';

class SwitchConfig {
  /// 标题
  String? title;

  /// 默认（关闭）文字颜色
  String? textColor;

  /// 在开启时文字颜色
  String? textColorAtOn;

  /// 「通用」当前环境语言<lang:, country:>
  Map? currentLocale;

  /// 「通用」屏幕宽度(pt)
  double? screenWidth;

  /// 「通用」屏幕高度(pt)
  double? screenHeight;

}

@HostApi()
abstract class SwitchHostAPI {
  /// 触发埋点
  void windTrack(String eventName, int eventID, Map detailInfo);

  /// 布局视图大小
  void contentSize(double width, double height);

  /// 更新-是否开启
  void fUpdateOn(bool on);
}

@FlutterApi()
abstract class SwitchFlutterAPI {
  /// 初始化配置
  void config(SwitchConfig maker);

  /// 是否开启
  void on(bool? on);

  /// 「通用」更新埋点补充信息
  void updateWindSupplementaryInfo(Map<String?, Object?> windSupplementaryInfo);
}
```

以上代码是根据组件 YAML 定义，通过 FGUIComponentAPI 生成的，主要作用是提供给 pigeon 组件进行 xx.api.dart 代码生成。

#### 开发基类（xx.base.dart）

pigeon 的作用只是多端的 messageChannel 封装，离我们想要的组件基类其实有很大的距离，这个大家可以去体验下就知道了。

所以调用基类的作用是进一步封装 pigeon 的 api.dart，让开发者无感知是一个对 App 的组件，只要调用/实现 base.dart 的方法，就可以做到独立调用以及给 add_to_app 调用。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4fb9d006e42e4ceba8e4fba1f24b32c6~tplv-k3u1fbpfcp-watermark.image?)

如上图所示，

基类对 `on` 属性的 set / get 重写，在设置上，如果是独立使用，那会走 widget.fUpdateOn(on) 方法，如果是 add_to_app 的方式，那就会调用 api.dart 中的 host.fUpdateOn(on) 通知给 Native，Native 就会通过 messageChannerl 收到消息。

@protected 的方法就是组件开发需要实现的方法，比如这边 Native 需要跨端组件的宽度进行布局。

#### 开发侧（xx.dart）

```dart
/// GUI 切换按钮组件
///
/// FIXED LAYOUT
class Switch extends SwitchBase {
  const Switch({Key? key, EventBus? eventBus}) : super(key: key, eventBus: eventBus);

  @override
  _SwitchState createState() {
    return _SwitchState();
  }
}

class _SwitchState extends SwitchStateBase {
  @override
  Widget build(BuildContext context) {
    return Directionality(
      textDirection: TextDirection.ltr,
      child: Container(), // Replace it!
    );
  }

  @override
  void updateCurrentLocale(Locale locale) {
    setState(() {});
  }
} 
```

以上也是 FGUIComponentAPI 生成的初始代码，也是为了防止一些坑。

比如最外层用 Directionality 包裹，是因为 multiple_flutters 不能是以 MaterialApp 作为根，而如果忽略了 Directionality，那在 add_to_app 有些实现会报错，比如 ListView，因为它需要文字排序方式，这个很多人都会忽略掉，因为 main.dart 都基本是以 MaterialApp 作为根的，它内置了 Directionality 实现。

还有一点比较有趣的设计，因为 Flutter 设计上是状态驱动，而不是方法驱动，所以生成上也加入了最简单的 EventBus 方式，让独立运行以及 add_to_app 的实现都统一起来。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08de4de7d931433e97bd883fd85b0847~tplv-k3u1fbpfcp-watermark.image?)

比如在测试 Demo 中，通过 UpdateBannersEvent 来直接修改组件数据，跟 App 调用 updateBanners 方法保持一致。

当然，测试工程也是自动生成的，只要填补关键代码即可。

### FGUIComponentAPI 生成双端调用类

#### iOS 端

从 [官方示例](https://github.com/flutter/samples/tree/master/add_to_app/multiple_flutters/multiple_flutters_ios/MultipleFluttersIos) 我们可以得知：

一个 FlutterEngineGroup 包括多个 FlutterEngine 实例

FlutterEngine 实例创建上需要指定 Entrypoint，这个就是我们上面入口层声明的 `componentSwitch`

每个 FlutterEngine 必须是 FlutterViewController 来承载

那我们需要对外封装成一个 View 来让 iOS 调用层使用。

```objc
//
// FGUISwitch.h
// AUTO GENERATE API
//
// NOTES: 自动生成，无需手动修改. 
// 

#import "FGUIComponent.h"
    
NS_ASSUME_NONNULL_BEGIN

@interface FGUISwitchInitConfig : NSObject

/// 标题
@property(nonatomic, nullable, copy) NSString *title;

/// 默认（关闭）文字颜色
@property(nonatomic, nullable, copy) NSString *textColor;

/// 在开启时文字颜色
@property(nonatomic, nullable, copy) NSString *textColorAtOn;

@end

/// [Flutter]: GUI 切换按钮组件
@interface FGUISwitch : FGUIComponent

- (instancetype)initWithMaker:(void(^)(FGUISwitchInitConfig *make))block hostVC:(UIViewController *)hostVC contentSizeDidUpdateBlock:(void(^)(CGSize contentSize))contentSizeDidUpdateBlock;

// MARK: - ContentSize
- (CGSize)intrinsicContentSize;

// MARK: - Properties

/// 是否开启
@property (nonatomic, assign) BOOL on;
@property (nonatomic, copy) void(^fUpdateOnBlock)(BOOL on);

// MARK: - Customer Blocks

// MARK: - Public Methods

// MARK: - Creators

@end

NS_ASSUME_NONNULL_END

```

以上就是示例自动生成的调用 h 文件
> m 文件过长，这里忽略展示，里面为了减少依赖以及多项目使用，是通过反射的形式生成调用代码。

关键点是需要外部传入一个 hostVC，内部通过 addChild 的形式将 FlutterViewController 加入到 hostVC 上。

#### Android 端

按[官方示例](https://github.com/flutter/samples/tree/master/add_to_app/multiple_flutters/multiple_flutters_android/app/src/main/java/dev/flutter/multipleflutters)是代码布局的形式，但按照 Android 小伙伴们的习惯，我们改成了支持 xml 布局的形式。

```kotlin
/**
 * AUTO GENERATE API
 * NOTES: 自动生成，无需手动修改. 
 */
...
/**
 * GUI 切换按钮组件
 */
class FGUISwitch : FrameLayout {
    private var mFragmentManager: FragmentManager? = null

    private var mEngineBinding: FGUISwitchBinding? = null

    private val entryPoint = "componentSwitch"
    
    private var _on: Boolean = false

    /**
     * 初始化
     * @param title 标题
     * @param textColor 默认（关闭）文字颜色
     * @param textColorAtOn 在开启时文字颜色
     */
    fun init(
        fragmentManager: FragmentManager,
        title: String = "--",
        textColor: String = "255,255,255,0.4",
        textColorAtOn: String = "255,255,255,1",
    ) {
      ...
    }
    
    /**
     * 设置 是否开启
     */
    fun setOn(on: Boolean) {
        _on = on
        mEngineBinding?.on(on)
    }
    
    /**
     * 是否开启
     */
    fun getOn(): Boolean {
        return _on
    }
    
    ...
}
```

这里也简单的把生成的调用部分放出来供大家参考。

> 特别说一下，因为 Android 不能用 Interface 的形式模拟 Class（这点 OC 真的是太好反射了）所以只能是直接依赖的 Flutter 的包，不过好处是，Android 里 Flutter 的包是根据 FlutterPlugin 拆包的，所以问题也不大。

### 示例效果

讲了半天干货，没有放实际示例效果给大家看下

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d2f883e1b934b70a742642a559d2e8f~tplv-k3u1fbpfcp-watermark.image?)

可以看到笔者开发调试都是在 Web 上，开发起来简单、轻松、明了。

因为也生成了 VO（ViewModel）代码，所以也天然的 VO / BO 代码分离。

也补充下线上真实效果

![IMG_4873.JPG](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0247bc8a90f94e25a21eb6f0346a5c07~tplv-k3u1fbpfcp-watermark.image?)

## 后续

这里面细节倒是有很多，篇（jing）幅（li）有限，先就这样，感谢阅读。

> 作者投稿
> 
> 来源：[Flutter 多引擎渲染，在稿定 App 的实践](https://juejin.cn/post/7130811413840429093)
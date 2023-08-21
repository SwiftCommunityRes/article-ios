## 前言

了解 iOS 17 中的 MapKit 后，我们会发现 Apple 引入了更适合 SwiftUI 的 API。

## MapKit 弃用项

一旦将你的 App 目标更新到 iOS 17，Xcode 会将任何使用旧的 Map 初始化器的用法标记为已弃用：

![](https://files.mdnice.com/user/17787/e99b3a8b-50af-474d-9cc5-5b5b2bbdfb46.png)

会有警告提示：init coordinate region 已在 iOS 17 中弃用。请改用带有 MapContentBuilder 参数的地图初始化器。

在 iOS 17 中，MapKit 为 SwiftUI 引入了需要 `MapContentBuilder` 参数的地图初始化器。下面为大家介绍一下MapKit 相关的基础知识。

## MapContentBuilder（iOS 17）

在 iOS 17 中，用于地图视图的各种初始化器都需要一个名为 `MapContentBuilder` 的 content 参数。MapContentBuilder 是一个结果构建器，允许在闭包中添加地图内容，例如标记、注释和自定义内容。

下面让我们看看是如何使用的，这里是一些伦敦地标的坐标：

```swift
extension CLLocationCoordinate2D {
  static let towerBridge = CLLocationCoordinate2D(latitude: 51.5055, longitude: -0.075406)
  static let boe = CLLocationCoordinate2D(latitude: 51.5142, longitude: -0.0885)
  static let hydepark = CLLocationCoordinate2D(latitude: 51.508611, longitude: -0.163611)
  static let kingsCross = CLLocationCoordinate2D(latitude: 51.5309, longitude: -0.1233)
}
```

要创建一个带有标记和注释的地图视图，详细代码如下：

```swift
struct ContentView: View {
  var body: some View {
    Map {
      Marker("Tower Bridge", coordinate: .towerBridge)
      Marker("Hyde Park", coordinate: .hydepark)
      Marker("Bank of England", 
        systemImage: "sterlingsign", coordinate: .boe)
        .tint(.green)
    
      Annotation("Kings Cross", 
        coordinate: .kingsCross, anchor: .bottom) {
          VStack {
              Text("在此搭乘火车！")
              Image(systemName: "train.side.front.car")
          }
          .foregroundColor(.blue)
          .padding()
          .background(in: .capsule)
      }
    }
  }
}
```

在没有其他选项的情况下，地图视图的边界将包围地图内容。

## 地图交互

为了控制用户与地图的交互方式，可以传递一组允许的模式。默认情况下允许所有模式（平移、缩放、倾斜、旋转），代码如下：

```swift
Map(interactionModes: [.pan,.pitch]) { ... }
```

## 地图样式

使用 Map Style 视图修饰符可以在标准、卫星或混合样式之间切换，控制高度、显示兴趣点和显示交通情况，代码如下：

```swift
Map { ...
}
.mapStyle(.hybrid(elevation: .realistic,
  pointsOfInterest: .including([.publicTransport]), 
  showsTraffic: true))
```

## 地图控件

标准的地图控件，如指南针、用户位置、倾斜、比例尺和缩放控件都实现为 SwiftUI 视图。这意味着可以将它们放置在视图的任何位置，不过需要定义一个地图范围命名空间，以将它们与它们控制的地图关联起来，代码如下：

```swift
struct ContentView: View {
  @Namespace var mapScope

  var body: some View {
    VStack {
      Map(scope: mapScope) { ... }
      MapCompass(scope: mapScope)
    }
    .mapScope(mapScope)
  }
}
```

要将它们放置在标准位置，使用地图控件视图修饰符，代码如下：

```swift
Map { ...
}
.mapControls {
  MapPitchToggle()
  MapUserLocationButton()
  MapCompass()
}
```

## 地图相机位置

地图相机位置定义了从地图表面上方查看地图的虚拟位置。可以使用现有的地图项、地图边界、区域或用户位置来创建地图相机位置并设置初始地图位置，代码如下：

```swift
Map(initialPosition: position)
```

将 `MapCameraPosition` 的绑定传递给地图，使其在用户在地图上移动时跟踪相机位置，代码如下：

```swift
struct ContentView: View {
  @State private var position: MapCameraPosition = .region(.uk)

  var body: some View {
    Map(position: $position) {
      Marker("Tower Bridge", coordinate: .towerBridge)
    }
  }
}
```

设置位置会导致地图更改其相机位置以匹配。例如，在用户移动位置后，要在 toolbar 中添加一个按钮，以将地图重置为初始位置，代码如下：

```swift
Map(position: $position) { ...
}
.toolbar {
  ToolbarItem {
    Button("重置") {
      position = .region(.uk)
    }
  }
}
```

将位置设置为 `.automatic` 可以使地图框架内容。

## 总结

这就是在 iOS 17 中使用 SwiftUI 中的 MapKit 所需要了解的内容。通过引入 MapContentBuilder 和其他新的初始化器，可以更方便地创建交互式地图视图，添加标记、注释和自定义内容，并在用户移动地图相机时自动更新位置。

此外，还可以使用 Map Style 修饰符和 Map 控件来自定义地图的样式和控件。这些改进使得在 SwiftUI 中使用 MapKit 变得更加强大和灵活。
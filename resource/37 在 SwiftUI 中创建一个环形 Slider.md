![环形Slider](https://upload-images.jianshu.io/upload_images/2955252-4d65dd29a92b221a.gif?imageMogr2/auto-orient/strip)


Slider 控件是一种允许用户从一系列值中选择一个值的 UI 控件。在 SwiftUI 中，它通常呈现为直线上的拇指选择器。有时将这种类型的选择器呈现为一个圆圈，拇指绕着圆周移动可能会更好。本文介绍如何在 SwiftUI 中定义一个环形的 Slider。

## 初始化环形轮廓

从`ZStack`中的三个圆环开始。一个灰色的圆环代表滑块的路径轮廓，一个淡红色的圆弧代表沿着圆环的进度，一个圆圈代表当前光标或拇指的位置。将滑块的范围设置为0.0到1.0，并硬编码一个直径和一个的当前位置进度 - 0.33。

```swift
struct CircularSliderView1: View {
    let progress = 0.33
    let ringDiameter = 300.0
    
    private var rotationAngle: Angle {
        return Angle(degrees: (360.0 * progress))
    }
    
    var body: some View {
        VStack {
            ZStack {
                Circle()
                    .stroke(Color(hue: 0.0, saturation: 0.0, brightness: 0.9), lineWidth: 20.0)
                Circle()
                    .trim(from: 0, to: progress)
                    .stroke(Color(hue: 0.0, saturation: 0.5, brightness: 0.9),
                            style: StrokeStyle(lineWidth: 20.0, lineCap: .round)
                    )
                    .rotationEffect(Angle(degrees: -90))
                Circle()
                    .fill(Color.white)
                    .frame(width: 21, height: 21)
                    .offset(y: -ringDiameter / 2.0)
                    .rotationEffect(rotationAngle)
            }
            .frame(width: ringDiameter, height: ringDiameter)

            Spacer()
        }
        .padding(80)
    }
}

```


![初始化环形轮廓](https://upload-images.jianshu.io/upload_images/2955252-16aecfb75df139e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 将进度值和拇指位置绑定

将进度变量更改为[状态](https://developer.apple.com/documentation/swiftui/state)变量并添加默认 Slider。这个 Slider 用于修改进度值，并在圆形滑块上实现足够的代码以使拇指和进度弧响应。当前值显示在环形 Slider 的中心。

```swift
struct CircularSliderView2: View {
    @State var progress = 0.33
    let ringDiameter = 300.0
    
    private var rotationAngle: Angle {
        return Angle(degrees: (360.0 * progress))
    }
    
    var body: some View {
        ZStack {
            Color(hue: 0.58, saturation: 0.04, brightness: 1.0)
                .edgesIgnoringSafeArea(.all)
            
            VStack {
                ZStack {
                    Circle()
                        .stroke(Color(hue: 0.0, saturation: 0.0, brightness: 0.9), lineWidth: 20.0)
                        .overlay() {
                            Text("\(progress, specifier: "%.1f")")
                                .font(.system(size: 78, weight: .bold, design:.rounded))
                        }
                    Circle()
                        .trim(from: 0, to: progress)
                        .stroke(Color(hue: 0.0, saturation: 0.5, brightness: 0.9),
                                style: StrokeStyle(lineWidth: 20.0, lineCap: .round)
                        )
                        .rotationEffect(Angle(degrees: -90))
                    Circle()
                        .fill(Color.white)
                        .shadow(radius: 3)
                        .frame(width: 21, height: 21)
                        .offset(y: -ringDiameter / 2.0)
                        .rotationEffect(rotationAngle)
                }
                .frame(width: ringDiameter, height: ringDiameter)
                
                
                VStack {
                    Text("Progress: \(progress, specifier: "%.1f")")
                    Slider(value: $progress,
                           in: 0...1,
                           minimumValueLabel: Text("0.0"),
                           maximumValueLabel: Text("1.0")
                    ) {}
                }
                .padding(.vertical, 40)
                
                Spacer()
            }
            .padding(.vertical, 40)
            .padding()
        }
    }
}
```

![使用线性滑块确保圆形滑块响应值的变化](https://upload-images.jianshu.io/upload_images/2955252-a0fe53d6a84d243e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 添加触摸手势

[DragGesture](https://developer.apple.com/documentation/swiftui/draggesture/) 被添加到滑块圆圈，并且使用临时文本视图显示拖动手势的当前位置。可以看到 x 和 y 坐标围绕包含环形  Slider 的位置中心的变化情况。

```swift
struct CircularSliderView3: View {
    @State var progress = 0.33
    let ringDiameter = 300.0
    
    @State var loc = CGPoint(x: 0, y: 0)
    
    private var rotationAngle: Angle {
        return Angle(degrees: (360.0 * progress))
    }
    
    private func changeAngle(location: CGPoint) {
        loc = location
    }
    
    var body: some View {
        ZStack {
            Color(hue: 0.58, saturation: 0.04, brightness: 1.0)
                .edgesIgnoringSafeArea(.all)
            
            VStack {
                ZStack {
                    Circle()
                        .stroke(Color(hue: 0.0, saturation: 0.0, brightness: 0.9), lineWidth: 20.0)
                        .overlay() {
                            Text("\(progress, specifier: "%.1f")")
                                .font(.system(size: 78, weight: .bold, design:.rounded))
                        }
                    Circle()
                        .trim(from: 0, to: progress)
                        .stroke(Color(hue: 0.0, saturation: 0.5, brightness: 0.9),
                                style: StrokeStyle(lineWidth: 20.0, lineCap: .round)
                        )
                        .rotationEffect(Angle(degrees: -90))
                    Circle()
                        .fill(Color.blue)
                        .shadow(radius: 3)
                        .frame(width: 21, height: 21)
                        .offset(y: -ringDiameter / 2.0)
                        .rotationEffect(rotationAngle)
                        .gesture(
                            DragGesture(minimumDistance: 0.0)
                                .onChanged() { value in
                                    changeAngle(location: value.location)
                                }
                        )
                }
                .frame(width: ringDiameter, height: ringDiameter)
                
                Spacer().frame(height:50)
                
                Text("Location = (\(loc.x, specifier: "%.1f"), \(loc.y, specifier: "%.1f"))")
                
                Spacer()
            }
            .padding(.vertical, 40)
            .padding()
        }
    }
}
```


![使用临时状态变量来显示位置点如何随拖动手势变化](https://upload-images.jianshu.io/upload_images/2955252-30f6b213a7460cc4.gif?imageMogr2/auto-orient/strip)



## 为不同的坐标值设置滑块位置

圆形滑块上有两个表示进度的值，用于显示进度弧度的`progress`值和用于显示滑块光标的`rotationAngle`。应该只有一个属性来保存滑块进度。视图被提取到一个单独的结构中，该结构具有圆形滑块上进度的一个绑定值。



滑块的`range`的可选参数也是可用的。这需要对进度进行一些调整，以计算已设置的角度以及拇指在圆形滑块上位置的旋转角度。另外调用`onAppear`根据`View`出现前的进度值计算旋转角度。


```swift
struct CircularSliderView: View {
    @Binding var progress: Double

    @State private var rotationAngle = Angle(degrees: 0)
    private var minValue = 0.0
    private var maxValue = 1.0
    
    init(value progress: Binding<Double>, in bounds: ClosedRange<Int> = 0...1) {
        self._progress = progress
        
        self.minValue = Double(bounds.first ?? 0)
        self.maxValue = Double(bounds.last ?? 1)
        self.rotationAngle = Angle(degrees: progressFraction * 360.0)
    }
    
    private var progressFraction: Double {
        return ((progress - minValue) / (maxValue - minValue))
    }
    
    private func changeAngle(location: CGPoint) {
        // 为位置创建一个向量（在 iOS 上反转 y 坐标系统）
        let vector = CGVector(dx: location.x, dy: -location.y)
        
        // 计算向量的角度
        let angleRadians = atan2(vector.dx, vector.dy)
        
        // 将角度转换为 0 到 360 的范围（而不是负角度）
        let positiveAngle = angleRadians < 0.0 ? angleRadians + (2.0 * .pi) : angleRadians
        
        // 根据角度更新滑块进度值
        progress = ((positiveAngle / (2.0 * .pi)) * (maxValue - minValue)) + minValue
        rotationAngle = Angle(radians: positiveAngle)
    }
    
    var body: some View {
        GeometryReader { gr in
            let radius = (min(gr.size.width, gr.size.height) / 2.0) * 0.9
            let sliderWidth = radius * 0.1
            
            VStack(spacing:0) {
                ZStack {
                    Circle()
                        .stroke(Color(hue: 0.0, saturation: 0.0, brightness: 0.9),
                                style: StrokeStyle(lineWidth: sliderWidth))
                        .overlay() {
                            Text("\(progress, specifier: "%.1f")")
                                .font(.system(size: radius * 0.7, weight: .bold, design:.rounded))
                        }
                    // 取消注释以显示刻度线
                    //Circle()
                    //    .stroke(Color(hue: 0.0, saturation: 0.0, brightness: 0.6),
                    //            style: StrokeStyle(lineWidth: sliderWidth * 0.75,
                    //                               dash: [2, (2 * .pi * radius)/24 - 2]))
                    //    .rotationEffect(Angle(degrees: -90))
                    Circle()
                        .trim(from: 0, to: progressFraction)
                        .stroke(Color(hue: 0.0, saturation: 0.5, brightness: 0.9),
                                style: StrokeStyle(lineWidth: sliderWidth, lineCap: .round)
                        )
                        .rotationEffect(Angle(degrees: -90))
                    Circle()
                        .fill(Color.white)
                        .shadow(radius: (sliderWidth * 0.3))
                        .frame(width: sliderWidth, height: sliderWidth)
                        .offset(y: -radius)
                        .rotationEffect(rotationAngle)
                        .gesture(
                            DragGesture(minimumDistance: 0.0)
                                .onChanged() { value in
                                    changeAngle(location: value.location)
                                }
                        )
                }
                .frame(width: radius * 2.0, height: radius * 2.0, alignment: .center)
                .padding(radius * 0.1)
            }
            
            .onAppear {
                self.rotationAngle = Angle(degrees: progressFraction * 360.0)
            }
        }
    }
}

```

`CircularSliderView` 的三种不同视图被添加到View中以测试和演示 Circular Slider 视图的不同功能。

```swift
struct CircularSliderView5: View {
    @State var progress1 = 0.75
    @State var progress2 = 37.5
    @State var progress3 = 7.5
    
    var body: some View {
        ZStack {
            Color(hue: 0.58, saturation: 0.06, brightness: 1.0)
                .edgesIgnoringSafeArea(.all)

            VStack {
                CircularSliderView(value: $progress1)
                    .frame(width:250, height: 250)
                
                HStack {
                    CircularSliderView(value: $progress2, in: 1...10)

                    CircularSliderView(value: $progress3, in: 0...100)
                }
                
                Spacer()
            }
            .padding()
        }
    }
}
```



![不同类型的环形视图](https://upload-images.jianshu.io/upload_images/2955252-df9e795eb7b9fa2a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结

本文展示了如何定义响应拖动手势的圆环滑块控件。可以设置滑块视图的大小，并且滑块按预期工作。可以向控件添加更多参数以设置颜色或圆环内显示的值的格式。 GitHub 上提供了 [Circular Slider](https://github.com/SwiftCommunityRes/swift) 的代码。

快速获取源码链接方式，公众号后台回复「**20230314**」获取。

> 来自：[Create a circular slider in SwiftUI](https://swdevnotes.com/swift/2022/create-a-circular-slider-in-swiftui/)

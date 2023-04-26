## 前言

Async/await语法是在Swift 5.5 引入的，在 WWDC 2021中的 [Meet async/await in Swift](https://developer.apple.com/videos/play/wwdc2021/10132/ "Meet async/await in Swift - introduction video from Apple") 对齐进行了介绍。它是编写异步代码的一种更可读的方式，比调度队列和回调函数更容易理解。Async/await 语法与其他编程语言（如C#或JavaScript）中使用的语法类似。使用 "async let "是为了并行的运行多个后台任务，并等待它们的综合结果。

Swift异步编程是一种编写允许某些任务并发运行而不是按顺序运行的代码的方法。这可以提高应用程序的性能，允许它同时执行多个任务，但更重要的是，它可以用来确保用户界面对用户输入的响应，同时任务在后台线程上执行。

## 长期运行的任务阻塞了UI

在一个同步的程序中，代码以线性的、从上到下的方式运行。程序等待当前任务完成后再进入下一任务。这在用户界面（UI）方面会产生问题，因为如果一个长期运行的任务被同步执行，程序就会阻塞，UI就会变得没有反应，直到任务完成。

下面的代码模拟了一个长期运行的任务，如以同步方式下载一个文件，其结果是UI 变得没有反应，直到任务完成。这样的用户体验是不可接受的。

Model:

```swift
struct DataFile : Identifiable, Equatable {
    var id: Int
    var fileSize: Int
    var downloadedSize = 0
    var isDownloading = false
    
    init(id: Int, fileSize: Int) {
        self.id = id
        self.fileSize = fileSize
    }
    
    var progress: Double {
        return Double(self.downloadedSize) / Double(self.fileSize)
    }
    
    mutating func increment() {
        if downloadedSize < fileSize {
            downloadedSize += 1
        }
    }
}
```

ViewModel:

```swift
class DataFileViewModel: ObservableObject {
    @Published private(set) var file: DataFile
    
    init() {
        self.file = DataFile(id: 1, fileSize: 10)
    }
    
    func downloadFile() {
        file.isDownloading = true

        for _ in 0..<file.fileSize {
            file.increment()
            usleep(300000)
        }

        file.isDownloading = false
    }
    
    func reset() {
        self.file = DataFile(id: 1, fileSize: 10)
    }
}
```

View:

```swift
struct TestView1: View {
    @ObservedObject private var dataFiles: DataFileViewModel
    
    init() {
        dataFiles = DataFileViewModel()
    }
    
    var body: some View {
        VStack {
            /// 从文末源代码获取其实现
            TitleView(title: ["Synchronous"])
            
            Button("Download All") {
                dataFiles.downloadFile()
            }
            .buttonStyle(BlueButtonStyle())
            .disabled(dataFiles.file.isDownloading)
            
            HStack(spacing: 10) {
                Text("File 1:")
                ProgressView(value: dataFiles.file.progress)
                    .frame(width: 180)
                Text("\((dataFiles.file.progress * 100), specifier: "%0.0F")%")

                ZStack {
                    Color.clear
                        .frame(width: 30, height: 30)
                    if dataFiles.file.isDownloading {
                        ProgressView()
                            .progressViewStyle(CircularProgressViewStyle(tint: .blue))
                    }
                }
            }
            .padding()
            
            Spacer().frame(height: 200)

            Button("Reset") {
                dataFiles.reset()
            }
            .buttonStyle(BlueButtonStyle())

            Spacer()
        }
        .padding()
    }
}
```

![模拟同步下载一个文件--没有实时更新UI](https://upload-images.jianshu.io/upload_images/2955252-b17dd01168f2e45e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 使用 async/await 在后台执行任务

将 ViewModel 中的`downloadFile`方法修改为异步的。请注意，由于`DataFile`模型是被视图监听的，对模型的任何改变都需要在UI线程上执行。这是通过使用 [MainActor](https://developer.apple.com/documentation/swift/mainactor/ "MainActor") 队列来完成的，即用`MainActor.run`包裹所有的模型更新。

ViewModel

```swift
class DataFileViewModel2: ObservableObject {
    @Published private(set) var file: DataFile
    
    init() {
        self.file = DataFile(id: 1, fileSize: 10)
    }
    
    func downloadFile() async -> Int {
        await MainActor.run {
            file.isDownloading = true
        }
        
        for _ in 0..<file.fileSize {
            await MainActor.run {
                file.increment()
            }
            usleep(300000)
        }
        
        await MainActor.run {
            file.isDownloading = false
        }
        
        return 1
    }
    
    func reset() {
        self.file = DataFile(id: 1, fileSize: 10)
    }
}
```

View：

```swift
struct TestView2: View {
    @ObservedObject private var dataFiles: DataFileViewModel2
    @State var fileCount = 0
    
    init() {
        dataFiles = DataFileViewModel2()
    }
    
    var body: some View {
        VStack {
            TitleView(title: ["Asynchronous"])
            
            Button("Download All") {
                Task {
                    let num = await dataFiles.downloadFile()
                    fileCount += num
                }
            }
            .buttonStyle(BlueButtonStyle())
            .disabled(dataFiles.file.isDownloading)
            
            Text("Files Downloaded: \(fileCount)")
            
            HStack(spacing: 10) {
                Text("File 1:")
                ProgressView(value: dataFiles.file.progress)
                    .frame(width: 180)
                Text("\((dataFiles.file.progress * 100), specifier: "%0.0F")%")
                
                ZStack {
                    Color.clear
                        .frame(width: 30, height: 30)
                    if dataFiles.file.isDownloading {
                        ProgressView()
                            .progressViewStyle(CircularProgressViewStyle(tint: .blue))
                    }
                }
            }
            .padding()
            
            Spacer().frame(height: 200)
            
            Button("Reset") {
                dataFiles.reset()
            }
            .buttonStyle(BlueButtonStyle())
            
            Spacer()
        }
        .padding()
    }
}
```


![使用 async/await 来模拟下载一个文件，同时更新UI](https://upload-images.jianshu.io/upload_images/2955252-e91bd9c45300ffe6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 在后台执行多个任务

现在我们有一个文件在后台下载，UI显示进度，让我们把它改为多个文件。`ViewModel`被改为持有一个`DataFiles`数组，而不是一个单一的文件。添加一个`downloadFiles`方法来遍历所有文件并下载每一个。

视图被绑定到`DataFiles`数组，并更新显示每个文件的下载进度。下载按钮被绑定到异步的`downloadFiles`中。

ViewModel:

```swift
class DataFileViewModel3: ObservableObject {
    @Published private(set) var files: [DataFile]
    @Published private(set) var fileCount = 0
    
    init() {
        files = [
            DataFile(id: 1, fileSize: 10),
            DataFile(id: 2, fileSize: 20),
            DataFile(id: 3, fileSize: 5)
        ]
    }
    
    var isDownloading : Bool {
        files.filter { $0.isDownloading }.count > 0
    }
    
    func downloadFiles() async {
        for index in files.indices {
            let num = await downloadFile(index)
            await MainActor.run {
                fileCount += num
            }
        }
    }
    
    private func downloadFile(_ index: Array<DataFile>.Index) async -> Int {
        await MainActor.run {
            files[index].isDownloading = true
        }
        
        for _ in 0..<files[index].fileSize {
            await MainActor.run {
                files[index].increment()
            }
            usleep(300000)
        }
        await MainActor.run {
            files[index].isDownloading = false
        }
        return 1
    }
    
    func reset() {
        files = [
            DataFile(id: 1, fileSize: 10),
            DataFile(id: 2, fileSize: 20),
            DataFile(id: 3, fileSize: 5)
        ]
    }
}
```

View:

```swift
struct TestView3: View {
    @ObservedObject private var dataFiles: DataFileViewModel3
    
    init() {
        dataFiles = DataFileViewModel3()
    }
    
    var body: some View {
        VStack {
            TitleView(title: ["Asynchronous", "(multiple Files)"])
            
            Button("Download All") {
                Task {
                    await dataFiles.downloadFiles()
                }
            }
            .buttonStyle(BlueButtonStyle())
            .disabled(dataFiles.isDownloading)
            
            Text("Files Downloaded: \(dataFiles.fileCount)")
            
            ForEach(dataFiles.files) { file in
                HStack(spacing: 10) {
                    Text("File \(file.id):")
                    ProgressView(value: file.progress)
                        .frame(width: 180)
                    Text("\((file.progress * 100), specifier: "%0.0F")%")
                    
                    ZStack {
                        Color.clear
                            .frame(width: 30, height: 30)
                        if file.isDownloading {
                            ProgressView()
                                .progressViewStyle(CircularProgressViewStyle(tint: .blue))
                        }
                    }
                }
            }
            .padding()
            
            Spacer().frame(height: 150)
            
            Button("Reset") {
                dataFiles.reset()
            }
            .buttonStyle(BlueButtonStyle())
            
            Spacer()
        }
        .padding()
    }
}
```

![使用async await来模拟按顺序下载多个文件](https://upload-images.jianshu.io/upload_images/2955252-86fb025a5d3d6c02.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 使用 "async let " 下载多个文件

**使用 "async let "来模拟并发下载多个文件的情况**

上面的代码可以被改进，以并行地执行多个下载，因为每个任务都是独立于其他任务的。在Swift并发中，这是用`async let`实现的，它用一个承诺立即给一个变量赋值，允许代码执行下一行代码。然后，代码等待这些承诺，等待最终结果的完成。

async/await：

```swift
    func downloadFiles() async {
        for index in files.indices {
            let num = await downloadFile(index)
            await MainActor.run {
                fileCount += num
            }
        }
    }
```

async let

```swift
    func downloadFiles() async {
        async let num1 = await downloadFile(0)
        async let num2 = await downloadFile(1)
        async let num3 = await downloadFile(2)
        
        let (result1, result2, result3) = await (num1, num2, num3)
        await MainActor.run {
            fileCount = result1 + result2 + result3
        }
    }
```

ViewModel

```swift
class DataFileViewModel4: ObservableObject {
    @Published private(set) var files: [DataFile]
    @Published private(set) var fileCount = 0
    
    init() {
        files = [
            DataFile(id: 1, fileSize: 10),
            DataFile(id: 2, fileSize: 20),
            DataFile(id: 3, fileSize: 5)
        ]
    }
    
    var isDownloading : Bool {
        files.filter { $0.isDownloading }.count > 0
    }
    
    func downloadFiles() async {
        async let num1 = await downloadFile(0)
        async let num2 = await downloadFile(1)
        async let num3 = await downloadFile(2)
        
        let (result1, result2, result3) = await (num1, num2, num3)
        await MainActor.run {
            fileCount = result1 + result2 + result3
        }
    }
    
    private func downloadFile(_ index: Array<DataFile>.Index) async -> Int {
        await MainActor.run {
            files[index].isDownloading = true
        }
        
        for _ in 0..<files[index].fileSize {
            await MainActor.run {
                files[index].increment()
            }
            usleep(300000)
        }
        await MainActor.run {
            files[index].isDownloading = false
        }
        return 1
    }
    
    
    func reset() {
        files = [
            DataFile(id: 1, fileSize: 10),
            DataFile(id: 2, fileSize: 20),
            DataFile(id: 3, fileSize: 5)
        ]
    }
}
```

View

```swift
struct TestView4: View {
    @ObservedObject private var dataFiles: DataFileViewModel4
    
    init() {
        dataFiles = DataFileViewModel4()
    }
    
    var body: some View {
        VStack {
            TitleView(title: ["Parallel", "(multiple Files)"])
            
            Button("Download All") {
                Task {
                    await dataFiles.downloadFiles()
                }
            }
            .buttonStyle(BlueButtonStyle())
            .disabled(dataFiles.isDownloading)
            
            Text("Files Downloaded: \(dataFiles.fileCount)")
            
            ForEach(dataFiles.files) { file in
                HStack(spacing: 10) {
                    Text("File \(file.id):")
                    ProgressView(value: file.progress)
                        .frame(width: 180)
                    Text("\((file.progress * 100), specifier: "%0.0F")%")
                    
                    ZStack {
                        Color.clear
                            .frame(width: 30, height: 30)
                        if file.isDownloading {
                            ProgressView()
                                .progressViewStyle(CircularProgressViewStyle(tint: .blue))
                        }
                    }
                }
            }
            .padding()
            
            Spacer().frame(height: 150)
            
            Button("Reset") {
                dataFiles.reset()
            }
            .buttonStyle(BlueButtonStyle())
            
            Spacer()
        }
        .padding()
    }
}

```

![使用 "async let "来模拟并行下载多个文件的情况](https://upload-images.jianshu.io/upload_images/2955252-074894f64e642cc3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![使用 "async let "来模拟并行下载多个文件的情况](https://upload-images.jianshu.io/upload_images/2955252-bb028c1b0a2aab28.gif?imageMogr2/auto-orient/strip)


# 结论
在后台执行长期运行的任务并保持UI的响应是很重要的。async/await提供了一个干净的机制来执行异步任务。有的时候，一个方法在后台调用多个方法，默认情况下是按顺序进行这些调用。async 让其立即返回，允许代码进行下一个调用，然后所有返回的对象可以一起等待。这使得多个后台任务可以并行进行。


GitHub 上提供了 [AsyncLetApp](https://github.com/SwiftCommunityRes/swiftAsyncBackgroundTasks "Async Let app source code on GitHub") 的源代码。


> 译自 [Use async let to run background tasks in parallel in Swift](https://swdevnotes.com/swift/2023/use-async-let-to-run-background-tasks-in-parallel-in-swift/)

# InjectionIII.app Project

## 同时支持 Swift, Objective-C & C++ 的代码热重载工具！ 

![Icon](http://johnholdsworth.com/Syringe_128.png)

Injection 能够让你在 iOS 模拟器、真机、Arm 芯片 Mac 直接运行的 iOS app 上无需重新构建或者重启你的 app 就实现更新 class 的实现、方法，添加 struct 或者 enum。节省开发者大量调试代码和设计迭代的时间。它把 Xcode 的职责从“源代码编辑器”变成“程序编辑器”，源码的修改不再仅仅保存在磁盘中而是会直接注入到运行时的程序中

### 如何使用

你可以在 [github
releases](https://github.com/johnno1962/InjectionIII/releases) 下载最新的 app 
也可以选择通过 [Mac App Store](https://itunes.apple.com/app/injectioniii/id1380446739?mt=12) 下载，然后你需要把下面这些代码添加到你的工程中并且在 app 启动时执行它（比如在 didFinishLaunchingWithOptions 的时候），这些配置工作就完成了。


```Swift
#if DEBUG
Bundle(path: "/Applications/InjectionIII.app/Contents/Resources/iOSInjection.bundle")?.load()
//for tvOS:
Bundle(path: "/Applications/InjectionIII.app/Contents/Resources/tvOSInjection.bundle")?.load()
//Or for macOS:
Bundle(path: "/Applications/InjectionIII.app/Contents/Resources/macOSInjection.bundle")?.load()
#endif
```
另外一个非常重要的事情是添加 `-Xlinker` and `-interposable` 这两个参数到 "Other Linker Flags" 你的工程文件中（注意只修改 `Debug` 配置如下图所示）

![Icon](interposable.png)

配置完成以后，当 app 运行起来以后控制台会输出一条关于文件监视器监听目录的消息，当前工程中包含的源文件保存的同时它也会同时被注入到您的设备中。所有旧的代码实现都会被替换为最新的代码实现

通常来说，你想要在屏幕中立刻看到最新的效果，可能需要让某些函数被重新调用一次。比如你在 view controller 中注入了代码，想要让它被重新渲染。你可以实现 `@objc func injected()` 方法，这个函数将会被框架自动调用。在项目中使用可以参考下面这个样例代码

```Swift
#if DEBUG
extension UIViewController {
    @objc func injected() {
        viewDidLoad()
    }
}
#endif
```
另外一个解法是用 "hosting"，使用的是
[Inject](https://github.com/krzysztofzablocki/Inject) 这个 Swift Package，用法参考[这篇博客](https://merowing.info/2022/04/hot-reloading-in-swift/).

### 哪些做不到的？

你不能修改数据在内存中的布局，比如你不能添加、删除、排序属性。对于非最终类（non-final classes），增加或删除方法也是一样的到底，因为用于分派的虚表（vtable）本身就是一种数据结构，不能靠 Injection 修改。Injection 也无法判断哪些代码段需要重新执行以更新显示，如上所述你需要自己判断。此外，不要过度使用访问控制。私有属性和方法不能直接被注入，特别是在扩展中，因为它们不是全局可替换的符号。它们通常通过间接方式进行注入，因为它们只能在被注入的文件内部访问，但这可能会引起混淆。最后，代码注入的同时又在对源文件执行添加、重命名或删除的操作可能会出问题。您可能需要重新构建并重新启动您的应用程序，甚至关闭并重新打开您的项目以清除旧的 Xcode 构建日志。

### Injection of SwiftUI

如果说有什么区别的话，SwiftUI 比 UIKit 更适合注入，因为它有特定的机制来更新显示，但你需要对每个想要注入的 `View` 结构体做一些修改。为了强制重新绘制，最简单的方法是添加一个属性来观察注入何时发生：

```
    @ObserveInjection var forceRedraw
```
这个属性包装器可以在 [HotSwiftUI](https://github.com/johnno1962/HotSwiftUI) 或 [Inject](https://github.com/krzysztofzablocki/Inject) Swift 包中找到。它实际上就是包含了一个 @Published 整数，视图可以观察这个整数，它会在每次注入时递增。您可以使用以下任一方法保证相关的代码在整个项目中可用：

```
@_exported import HotSwiftUI
or
@_exported import Inject
```

让 SwiftUI 注入所需的第二个更改是调用 View 的 `.enableInjection()` 方法将 body 属性的返回类型“擦除”为 `AnyView` ——这个技巧叫做"erase the return type"。这是因为，在添加或删除 SwiftUI 元素时，body 属性的具体返回类型可能会发生变化，这相当于内存布局的更改，可能会导致崩溃。总的来说，每个 body 的末尾都应该看起来像这样：

```
    var body: some View {
         VStack or whatever {
        // Your SwiftUI code...
        }
        .enableInjection()
    }

    @ObserveInjection var redraw
```
你可以保留这些修改到生产环境中，`Release` 构建时这个调用会被优化为一个无操作（no-op）

#### Xcode 16
Xcode 16 中新增了 SWIFT_ENABLE_OPAQUE_TYPE_ERASURE 构建设置。这个设置默认是开启的，你不再需要显式地擦除视图的 body。但是，你仍然需要使用 `@ObserveInjection` 来强制重新绘制。

更多的信息可以参考 [Xcode 16.2 release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-16_2-release-notes).

### 关于 Injection 在 iOS, tvOS or visionOS 设备上的运行
[github 
4.8.0+ releases](https://github.com/johnno1962/InjectionIII/releases) 版本以上的 InjectionIII.app 需要通过修改 user default 并且重启 mac 端的 InjectionIII.app 明确表示需要真机调试，在命令行执行下面代码可以修改 user default

```
$ defaults write com.johnholdsworth.InjectionIII deviceUnlock any
```

还需要在在“Build Phase” 添加一个 run script, 并且关闭 "User Script Sandboxing"

```
RESOURCES=/Applications/InjectionIII.app/Contents/Resources
if [ -f "$RESOURCES/copy_bundle.sh" ]; then
    "$RESOURCES/copy_bundle.sh"
fi
```
最后在 app 启动以后加载相关的 bundle，具体参考下面的示例代码

```
    #if DEBUG
    if let path = Bundle.main.path(forResource:
            "iOSInjection", ofType: "bundle") ??
        Bundle.main.path(forResource:
            "macOSInjection", ofType: "bundle") {
        Bundle(path: path)!.load()
    }
    #endif
```
这样配置以后，模拟器和真机就都可以工作了。有关如何通过 Wi-Fi 连接到 InjectionIII.app 进行调试的详细信息，请查阅 [HotReloading project](https://github.com/johnno1962/HotReloading) 项目 的 README。你还需要从下拉菜单中手动选择项目目录以供文件监视器使用。

### 在 macOS 上工作
macOS 也可以工作，但在开发过程中，你需要暂时关闭 "app sandbox" 和 "hardened runtime-library validation" ，以便能够动态加载代码。为了避免代码签名问题，请按照上述在真实设备上进行注入的说明，使用新的 `copy_bundle.sh` 脚本。

### 工作原理

Injection has worked various ways over the years, starting out using 
the "Swizzling" apis for Objective-C but is now largely built around 
a feature of Apple's linker called "interposing" which provides a 
solution for any Swift method or computed property of any type.

When your code calls a function in Swift, it is generally "statically
dispatched", i.e. linked using the "mangled symbol" of the function being called.
Whenever you link your application with the "-interposable" option
however, an additional level of indirection is added where it finds 
the address of all functions being called through a section of 
writable memory. Using the operating system's ability to load 
executable code and the [fishhook](https://github.com/facebook/fishhook) 
library to "rebind" the call it is therefore possible to "interpose"
new implementations of any function and effectively stitch 
them into the rest of your program at runtime. From that point it will 
perform as if the new code had been built into the program. 

Injection uses the `FSEventSteam` api to watch for when a source
file has been changed and scans the last Xcode build log for how to
recompile it and links a dynamic library that can be loaded into your
program. Runtime support for injection then loads the dynamic library 
and scans it for the function definitions it contains which it then
"interposes" into the rest of the program. This isn't the full story as
the dispatch of non-final class methods uses a "vtable" (think C++ 
virtual methods) which also has to be updated but the project looks 
after that along with any legacy Objective-C "swizzling".

If you are interested knowing more about how injection works
the best source is either my book [Swift Secrets](http://books.apple.com/us/book/id1551005489) or the new, start-over reference implementation
in the [InjectionLite](https://github.com/johnno1962/InjectionLite) 
Swift Package. For more information about "interposing" consult [this
blog post](https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html) 
or the README of the [fishhook project](https://github.com/facebook/fishhook). 
For more information about the organisation of the app itself, consult [ROADMAP.md](https://github.com/johnno1962/InjectionIII/blob/main/ROADMAP.md).

### A bit of terminology

Getting injection to work has three components. A FileWatcher, the code to
recompile any changed files and build a dynamic library that can be loaded 
and the injection code itself which stitches the new versions of your code
into the app while it's running. How these three components are combined
gives rise to the number of ways injection can be used.

"Injection classic" is where you download one of the [binary releases](https://github.com/johnno1962/InjectionIII/releases)
from github and run the InjectionIII.app. You then load one of the bundles
inside that app into your program as shown above in the simulator. 
In this configuration, the file watcher and source recompiling is done 
inside the app and the bundle connects to the app using a socket to 
know when a new dynamic library is ready to be loaded.

"App Store injection" This version of the app is sandboxed and while
the file watcher still runs inside the app, the recompiling and loading
is delegated to be performed inside the simulator. This can create 
problems with C header files as the simulator uses a case sensitive 
file system to be a faithful simulation of a real device.

"HotReloading injection" was where you are running your app on a device
and because you cannot load a bundle off your Mac's filesystem on a real 
phone you add the [HotReloading Swift Package](https://github.com/johnno1962/HotReloading)
to your project (during development only!) which contains all the code that
would normally be in the bundle to perform the dynamic loading. This 
requires that you use one of the un-sandboxed binary releases. It has
also been replaced by the `copy_bundle.sh` script described above.

"Standalone injection". This was the most recent evolution of the project 
where you don't run the app itself anymore but simply load one of the 
injection bundles and the file watcher, re-compilation and injection are 
all performed inside the simulator. By default this watches for changes 
to any Swift file inside your home directory though you can change this
using the environment variable `INJECTION_DIRECTORIES`.

[InjectionLite](https://github.com/johnno1962/InjectionLite) is a start-over
minimal implementation of standalone injection for reference. Just add
this Swift package and you should be able to inject in the simulator.

[InjectionNext](https://github.com/johnno1962/InjectionNext) is a 
currently experimental version of Injection that should be faster and 
more reliable for large projects. It integrates into a debugging flag of 
Xcode to find out how to recompile files to avoid parsing build logs
and re-uses the client implementation of injection from `InjectionLite`.
To use with external editors such as `Cursor`, InjectionNext can also
use a file watcher to detect edits and fall back to build log parsing code.

All these variations require you to add the "-Xlinker -interposble" linker flags 
for a Debug build or you will only be able to inject non-final methods of classes
and all can be used in conjunction with either of the higher level 
[Inject](https://github.com/krzysztofzablocki/Inject) or
[HotSwiftUI](https://github.com/johnno1962/HotSwiftUI).

### Further information

Consult the [old README](https://github.com/johnno1962/InjectionIII/blob/main/OLDME.md) which if anything contained 
simply "too much information" including the various environment
variables you can use for customisation. A few examples:

| Environment var. | Purpose |
| ------------- | ------------- |
| **INJECTION_DETAIL** | Verbose output of all actions performed |
| **INJECTION_TRACE** | Log calls to injected functions (v4.6.6+) |
| **INJECTION_HOST** | Mac's IP address for on-device injection |

With an **INJECTION_TRACE** environment variable, injecting 
any file will add logging of all calls to functions and methods in
the file along with their argument values as an aid to debugging.

A little known feature of InjectionIII is that provided you have 
run the tests for your app at some point you can inject an 
individual XCTest class and have if run immediately – 
reporting if it has failed each time you modify it.

### Acknowledgements:

This project includes code from [rentzsch/mach_inject](https://github.com/rentzsch/mach_inject),
[erwanb/MachInjectSample](https://github.com/erwanb/MachInjectSample),
[davedelong/DDHotKey](https://github.com/davedelong/DDHotKey) and
[acj/TimeLapseBuilder-Swift](https://github.com/acj/TimeLapseBuilder-Swift) under their
respective licenses.

The App Tracing functionality uses the [OliverLetterer/imp_implementationForwardingToSelector](https://github.com/OliverLetterer/imp_implementationForwardingToSelector) trampoline implementation via the [SwiftTrace](https://github.com/johnno1962/SwiftTrace) project under an MIT license.

SwiftTrace uses the very handy [https://github.com/facebook/fishhook](https://github.com/facebook/fishhook).
See the project source and header file included in the app bundle
for licensing details.

This release includes a very slightly modified version of the excellent
[canviz](https://code.google.com/p/canviz/) library to render "dot" files
in an HTML canvas which is subject to an MIT license. The changes are to pass
through the ID of the node to the node label tag (line 212), to reverse
the rendering of nodes and the lines linking them (line 406) and to
store edge paths so they can be coloured (line 66 and 303) in "canviz-0.1/canviz.js".

It also includes [CodeMirror](http://codemirror.net/) JavaScript editor
for the code to be evaluated using injection under an MIT license.

The fabulous app icon is thanks to Katya of [pixel-mixer.com](http://pixel-mixer.com/).

$Date: 2024/11/22 $

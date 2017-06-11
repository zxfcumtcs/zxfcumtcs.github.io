title: iOS App Life Cycle
date: 2014-10-26 19:01:00
tags:
- iOS
---
本文主要介绍 iOS App 的5种状态以及它们之间的转换过程。
<!--more-->
©原创文章，转载请注明出处！

# Overview
_______________

在 iOS 中，每个 app 在每个时刻都有一个特定的状态，其或在*前台运行(Active)*、或在*后台运行(background)*、或在*奔往前后台的路上(Inactive)*、或在*后台睡大觉(Suspended)*，亦或*早已 over(Not running)*。

ok，我们来个系统、直观的表述：

![](/img/APPStatus.png)

App 在5种状态间的转换流程[图](https://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Introduction/Introduction.html)

![](/img/StatechangesiOSapp.png)


骄傲的iOS君本着对用户负责的态度，对其内的每个 app 能做什么、不能做什么都有严格的限制，任何 app 都不能越雷池半步！

今天我们讨论的话题是 app 的生命周期，那么总结一下与 app 生命周期有关的几个时间规则：
![](/img/apptimelimit.png)

其中，对启动时间的限制是为了追求更好的用户体验，而其它几项基本是为了更好地管理、使用系统有限的资源。

# Launch​ Options Keys
_______________

无论 app 是在 foreground 启动还是在 background 启动，我们能参与控制启动的两个点分别是 app delegate 的两个回调`application:willFinishLaunchingWithOptions:`及`application:didFinishLaunchingWithOptions:`。

在这两个回调中，我们首先要确认的是 app 因何而启动。同时还要准备 app 在启动过程中所需的核心数据结构以及 app 需要显示的 UI。

当`application:didFinishLaunchingWithOptions:`返回后，UIKit 才会将 window 显示在屏幕上，也就是说直到此时 app 的界面都会显示出来。

通过上面的图表，我们知道iOS 对 app 的启动时间有严格限制，因此上述两个回调函数要尽量简单，只做必要的事情！

启动 app 的原因有很多，最直接的就是用户点击 app icon，除此之外还有如响应 push 消息、其他 app 通过 openurl 启动等等。

`application:willFinishLaunchingWithOptions:`及`application:didFinishLaunchingWithOptions:`的第二个参数`launchOptions`准确记录了 app 启动原因以及相应的参数信息。如，若`launchOptions`中存在以`UIApplicationLaunchOptionsURLKey`为 key 的键值对，表明 app 是由 openurl 启动的。

![](/img/LaunchOptions.png)



# Launch app into foreground
_______________

大多数情况下，app 启动后都会在前台运行，app 的状态也将完成由not running到 inactvie 再到 active 的转换。为了启动 app，系统将创建一个新的进程以及主线程，并在主线程上调用`main`函数，由 Xcode生成的默认`main`函数只是简单地将控制权交给UIKit framework(通过调用`UIApplicationMain`函数)。

在前台启动 app 的事件流如下图所示([来自](https://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/StrategiesforHandlingAppStateTransitions/StrategiesforHandlingAppStateTransitions.html))：

![](/img/Launchinganappintotheforeground.png)



# Launch app into background
_______________

对于支持在后台运行的 app，当某些 background event 发生时，app 将会在后台启动。此时，app 将进入background state，并能处理各种 event，随后可能在某个时间点被挂起。app 在后台启动时，系统依然会完成相应的 UI 初始化工作，但不会将其显示到设备屏幕上。

ps：**app 能在后台启动的前提是用户没有在任务中心强制将 app kill 掉(iOS8 下的location apps除外)。**

主要的background event 主要有：

+ 系统接收到silent push；
+ 系统启动 app 执行 background fetch 任务；
+ background transfer 相关的事件，如任务完成、失败或需要鉴权等；
+ newsstand apps、location apps、audio apps以及bluetooth apps收到相应的事件。

app 在后台启动时的事件流如下图所示([来自](https://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/StrategiesforHandlingAppStateTransitions/StrategiesforHandlingAppStateTransitions.html))：

![](/img/Launchinganappintothebackground.png)

在代码中，我们如何判断 app 是在前台启动还是后台启动呢？

答案是在`application:willFinishLaunchingWithOptions:`或`application:didFinishLaunchingWithOptions:`函数中判断`UIApplication`对象的`applicationState`属性值，若该值等于`UIApplicationStateInactive`则表明app 是在前台启动，若等于`UIApplicationStateBackground`表明 app 是在后台启动。

# Launch app to open a URL
_______________

前面我们介绍过，app A 可以通过 openUrl 启动 app B，通过 openUrl 启动 app 的流程与前面介绍的流程略有不同：

首先，我们需要在`application:willFinishLaunchingWithOptions:`或`application:didFinishLaunchingWithOptions:`函数中，判断是否可以处理该 url(通过`UIApplicationLaunchOptionsURLKey`获取该 url)，若能处理，上述函数返回 YES，否则返回 NO。只有当它们都返回 YES 时，流程才会进入处理 openUrl 事件的回调中(`application:openURL:sourceApplication:annotation:`)

app 因 openUrl 而启动时的事件流如下图所示([来自](https://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/StrategiesforHandlingAppStateTransitions/StrategiesforHandlingAppStateTransitions.html))：

![](/img/LaunchinganapptoopenaURL.png)

当 app 在后台时，将唤醒之，使其进入前台运行：

![](/img/WakingabackgroundapptoopenaURL.png)

# Temporary Interruptions
_______________

当 app 正在前台运行时，有电话进来或用户打开通知中心等，对于当前 app 来说都会出现Temporary Interruptions，此时 app 依然在前台运行，但其状态转换为 inactive，不能接收 touch events，可以继续接收 notifactions 以及其他 events。

当出现中断时，系统会调用`applicationWillResignActive:`回调，在该方法中需要做的事情：
+ 保存关键数据以及状态信息；
+ 停止定时器以及其他周期性的任务；
+ 停止查询 metadata 任务；
+ 不要发起新任务；
+ 暂停音视频的播放；
+ 对于游戏类的 app 进入暂停状态；
+ 停止任何 OpenGL 的操作(否则会 crash)；
+ 暂停dispatch 或 operation 队列中的任何non-critical任务。

当 app 再次进入 active 状态时，会调用`applicationDidBecomeActive:`回调，在该方法中需要恢复在`applicationWillResignActive:`执行的操作。

Temporary Interruptions 的处理流程如下图所示：

![](/img/Handlingalertbasedinterruptions.png)

# 小结
_______________

了解 app 的生命周期，了解 app 5种状态的转换过程，了解 app 在状态转换中需要注意的问题，对提高 app 的稳定性、健壮性十分重要。在 app 状态转换过程中，若处理不当，可能会造成重要数据丢失、出现莫名 bug、甚至 crash 等严重问题！

title: iOS8新特性之Extension
date: 2015-01-10 16:41:25
tags:
- iOS
---
本文简单介绍了 Extension 的基本原理，包括：life cycle、communicates、share data、share code 以及开发 widget 时需要注意的点。
<!--more-->
©原创文章，转载请注明出处！

# Overview
________________________________

此刻写这篇文章都感觉不好意思了，Apple 自 WWDC2014 推出 Extension 这项引人注目的新特性已半年之久，广大前辈及同行以迅雷不及掩耳之速写下大量关于这方面的优秀文章。在这个拥抱变化、适应变化、快速学习、注重分享的互联网时代，足以令我自愧不如！后知后觉的我，作为自娱自乐的方式之一，还是鼓起勇气写下这篇文章，作为自我学习的总结。

言归正传，很明显，Extension 即扩展之意也！说白了，Extension 使 App 的功能可以轻松地"入侵"系统或三方 App。当然，在对方眼里 Extension 自然就是所谓的插件了！

官方一点的说法：Extension 的目的是使用户在使用系统功能或三方app时还能使用我们的app提供的功能。

iOS 支持的 Extension 类型有：`Today Extension(俗称widget)`、 `Share Extension`、 `Action Extension`、 `Photo Editing Extension`、 `Document Provider Extension` 以及 `Custom Keyboard Extension`。

广大 App 比较感兴趣的应该是 `Today Extension`，这也是本文要拿来说道的主角(其他的还不会ಥ_ಥ)。

ok，在继续之前先来两个名词解释：

+ `Host App` —— 宿主 App，即 Extension"入侵"的 App，用户最终是在 Host App 中使用 Extension；
+ `Container App` —— 容器 App，即 Extension 的主人，Extension 包含在Container App中，并通过Container App发布。

# Life Cycle of an App Extension
__________________________________

虽然，Extension 包含于其 Container App 中，并通过 Container App 发布到 App Store 最终安装到用户设备上。但 Extension 有独立于其 Container App 的二进制可执行文件，也就是说 Extension 和 Container App 在运行时分属于二个独立进程(相当于二个独立 App)。这就导致在 iOS 这个"隔离"至上的系统上有两个问题需要解决：Extension 和 Container App 之间的通讯以及彼此间的数据共享(Sandbox)。

OK, 既然 Extension 和 Container App 有各自独立的可执行文件，其生命周期也就互不干扰，Extension 在运行时 Container App 可能早已被 kill。但 Extension 终究是 Extension，在 App 面前似乎低人一等，属于 iOS 中的"二等"公民.

Extension 的生命周期起于 Host App 响应用户事件令其执行一项任务，止于其完成该任务。当然，所谓完成任务可以是即时的同步完成该任务，也可以是发起一个异步的、后台执行的任务。

Extension 生命周期如下图所示(来自[App Extension Programming Guide](https://developer.apple.com/library/prerelease/ios/documentation/General/Conceptual/ExtensibilityPG/ExtensionOverview.html#//apple_ref/doc/uid/TP40014214-CH2-SW2))：

![](/img/lifecycleofanappextension.png)

是的，Extension 在其完成任务后，系统会无情地将其 kill 掉(没办法，"二等"公民)。

嗯，关于 Extension 的"二等"公民身份，咱们再多捞道几句(ｖ_ｖ)：

+ Extension 应该是轻量级的、启动要迅速，最好在1秒之内，启动过慢将被系统终止；
+ 在使用内存方面，对 Extension 也有更多的限制，尤其是 widget，因为在 Notification Center 几乎会同时存在多个 widget。在系统内存不足时，会优先 kill Extension；
+ 在使用系统共享资源上(如GPU)，Extension 也没有优先权；
+ 一些 API 在 Extension 上不可用；
+ Extension 可以利用 NSURLSession 开启后台网络任务，但仅此而已，如VoIP、playing background audio等其它类型的后台任务在 Extension 中是不允许的；
+ Extension 必须支持arm64；
+ Extension 的 UI 应该是简洁流畅的、专注于单一功能。

# About App Extension Communicates
_____________________________________

理解 Extension、Container App 以及 Host App 间你中有我、我中有你的关系是理解 Apple Extension 机制的关键所在。正如前文所述，上述三者各自有自己的二进制可执行文件，运行时分属于三个独立的进程。而我们知道在 iOS 系统中，进程间(App间)有着很深的"隔阂"，Sandbox 机制不允许 App 间有过多的交流，数据共享更是不可能。

那么 Extension、Container App 以及 Host App 间是如何交流的呢？

+ Host App and Container App —— 分属于两个正常的 App，没啥好说的；
+ Extension and Container App —— 虽然两者独立运行，但毕竟是"同根生"，有着"血浓于水"的亲情。Apple 也为它们间的数据共享提供了很好的支持，即 App Group。当然在必要时 Extension 还可以通过`NSExtensionContext`类的实例方法`openUrl`呼起 Container App(似乎在 widget 中这也是一项必备技能)；
+ Extension and Host App —— 两者间似乎有种"主仆"关系，Host App 通过`NSExtensionContext`向 Extension 发出指令要求其完成某项任务，Extension 则通过`NSExtensionContext`向 Host App 反馈任务响应。

三者间的关系如下图所示(来自[App Extension Programming Guide](https://developer.apple.com/library/prerelease/ios/documentation/General/Conceptual/ExtensibilityPG/ExtensionOverview.html#//apple_ref/doc/uid/TP40014214-CH2-SW2)):
![](/img/CommunicatesBetweenExtension.png)

# Share Data between Extension and Container App
_______________________________________

Extension 作为 Container App 的延伸，两者间的数据共享在所难免。然而，iOS 的 Sandbox 机制又将该需求拒之门外。
![](/img/appsandbox.png)
![](/img/AppCrashed.jpg)
不过，不用桑心，Apple 君自然会考虑到该问题。

是的，为了解决 Extension 与 Container App 间数据共享的问题，Apple 在 iOS8 中引入了 App Group 的概念。在同一 App Group 中的 App 可以共享Shared Container中的数据。

来自[App Extension Programming Guide](https://developer.apple.com/library/prerelease/ios/documentation/General/Conceptual/ExtensibilityPG/ExtensionScenarios.html#//apple_ref/doc/uid/TP40014214-CH21-SW1):
![](/img/sharedcontainer.png)

来自[iOS8 by Tutorias](http://www.raywenderlich.com/store/ios-8-by-tutorials):
![](/img/sharedcontainer1.png)

通过上面两张图可以看出Extension 与 Container App 可以通过 Shared Container 共享数据。
![](/img/good.png)

## App Group

我们知道，Extension 与 Container App 间之所以能共享数据，App Group 起到关键性的作用。

处于同一 App Group 下的 App 可以绕过 Sandbox 机制共享数据，正是由于 App Group 的特殊性，Apple 对 App Group 有着严格的管理，App Group 与开发者帐号绑定，以确保只有同一 team 下的 App 才可以共享数据。并且只有管理员才可以配置 App ID 下的App Group。

App Group 由 App Group ID 唯一标识，其格式通常为：group.bundle identifier。

开启 App Group 需要同时在 Container App 与 Extension 的 target 上进行相应的设置：
![](/img/appgroup.png)

开启 App Group 后，XCode 会分别为 Extension 及 Container App 生成相应的 entitlements 文件。

OK，有了 App Group，我们就可以任性地在 Extension 和 Container App 间共享数据了。通过 Shared Container 共享数据有两种方式：NSUserDefaults 以及文件系统。

当然，NSUserDefaults 通常用于共享简单的数据。对于复杂的数据还是要通过 Shared Container(即以及文件的方式)共享。

以 NSUserDefaults 方式共享数据简单明了：只要将我们熟悉的方式
![](/img/NSUserDefaults.png)
改为：
![](/img/NewNSUserDefaults.png)

然后就能方便的通过 NSUserDefault 在 Extension 和 Container App 间共享数据了。但有些数据如图片、特定格式的数据则无法通过 NSUserDefault 分享，此时就需要使用文件系统了。

通过文件系统分享，其原理也非常简单，就是让 Extension 和 Container App 共同访问某个特定的共享目录。
正常情况下，我们要获得 Sandbox 中的某个路径，可以通过下述方式：
![](/img/normalfilemanager.png)
在访问共享目录时，需要通过以下方式，可以看到共享目录是由 app group id唯一标识的：
![](/img/appgroupfilemanager.png)

通过共享目录在 Extension 和 Container App 间共享数据固然方便，但我们知道 Extension 和 Container App 是两个独立的进程。是的，没错，存在数据同步问题，我们要确保共享数据的完整性。

> Use Core Data, SQLite, or Posix locks to help coordinate data access in a shared container.

这就是 Apple 君给我们的建议！

最简单的方式恐怕就是通过`writeToFile:atomically:encoding:error:`接口写数据了。

## Share Core Data between Extension and Container App

正常情况下 Core Data 会在 App Sandbox 中按指定的路径生成相应的数据存储文件。为了在 Extension 和 Container App 间共享 Core Data，关键点就在于将 Core Data 对应的数据存储文件保存到共享目录下。

为了实现共享 Core Data 有以下几个步骤：
+ 在 Extension、Container App 中将 Core Data 的数据存储文件保存到共享目录下的同一路径上；
+ 将 xcdatamodeld 文件以及相应的 model 文件分别添加到 Extension 以及 Container App 的 target 中；
+ 在 Extension 的 Copy Bundle Resources 中添加 xcdatamodeld 文件。

OK，通过上述简单的几步，就可以在 Extension 和 Container App 间通过 Core Data 共享数据了。

那且不是可以在 Extension 中共享 Container App 已有的 Core Data 数据！

然而，"前途是光明的，道路是曲折的"！

别忘了，App Group 是 iOS8 才有的新特性！将 Core Data 数据存储文件保存到共享目录在 iOS8 之前的系统上是没法用的！因此，在 App 还支持 iOS8 以下的系统时，Core Data 最多只能作为两者共享数据的桥梁，但这似乎有点重，可能普通的文件更适合，当然也要视具体情况而定。

因此，真正意义上，要想在 Extension 和 Container App 间共享 Core Data 数据，我们只能坐等 App 从 iOS8开始支持了！

# Share Code between Extension and Container App
__________________________________

Extension 作为 Container App 功能触角的延伸而"入侵"系统或三方 App，更多时侯 Extension 提供的功能是 Container App 已有的功能。因此，在 Extension 和 Container App 间存在code共享问题。然而，Extension 和 Container App 分属于两个 target，共享 code 似乎有点棘手。

最笨的方法可能就是 copy code，恭喜你，如果这么做，将会获得"劣质"程序猿的"光荣称号"！

那么我们还可以将需要共享的 code 对应的源文件分别添加到 Extension 和 Container App 的 target 中。虽然没有 copy code，但将会存在两个完全相同的二进制文件，此仍非上策也！

为了，完美解决该问题，Apple 君在 iOS8 引入了 Embedded Frameworks的概念。

具体如何操作 Embedded Frameworks 就不过多的阐述了。需要注意的，有些 API 在 Embedded Framework 中是不可用的，具体参见[App Extension Programming Guide](https://developer.apple.com/library/ios/documentation/General/Conceptual/ExtensibilityPG/ExtensionOverview.html#//apple_ref/doc/uid/TP40014214-CH2-SW6)

可以让 XCode 在 Embedded Framework 使用了不该使用的 API 时发出 warning，只需要在 Embedded Framework Target 的 General 选项中作如下设置：
![](/img/allowappextensionapionly.png)

还有一个要注意的问题，如果 Container App link 了 Embedded Framework，该Container App同样要支持arm64。

遗憾的是，同App Group一样，Embedded Framework也是 iOS8 才支持的新特性！如果，你的 App 还要支持 iOS7 之类的系统，那么基本可以和Embedded Framework说再见了！

为了解决该问题，Apple 君给我们的建议是不要在 XCode 中为 Container App link Embedded Framework，而是在运行时根据系统版本号动态地在 iOS8 系统中通过`dlopen` link 所需的 Embedded Framework。此时，就可以通过`CFBundleGetFunctionPointerForName`、`CFBundleGetFunctionPointersforNames`以及`NSBundle`类的`load`、`loadAndReturnError:`、`classNamed:` 等 API 获取 Embedded Framework 中的接口。

Apple 君特定说明的是，这些 API 只能在 iOS8 及以上的系统中调用！

这样看似解决了iOS8 以下系统不支持 Embedded Frameworks 的问题，但并未解决Share Code的核心问题！试想，Embedded Frameworks 中包含的功能在 iOS7 上如何使用？最终还是要将这些功能包含在 Container App 的 target 中以供 iOS8 以下的系统享用。是不是有种"一夜回到解放前"的凄凉之感！要想真正使用 Embedded Frameworks 这个新特性只能坐等 App 从 iOS8 开始支持了！

# Today Extension
_________________________________

唠唠叨叨讲了不少，最后看个实例吧——Today Extension(江湖人称 widget)。

关于如何在 XCode 中创建 Today Extension 就不截图了(省点流量)。

下面的文字可能没有一个良好的逻辑顺序，想到哪写到哪，都是一些我认为有意思的点。

通过 XCode Extension Template 生成的 Today Extension，会自动生成 UI 文件*MainInterface.storyboard*。如果你不习惯或者是不会使用storyboard，而是希望通过 code 的方式实现 UI，可以在 Extension 的 Info.plist 文件中作相应的设置：
![](/img/ExtensionUIplist.png)

是的，没错，`TodayViewController`就是Today Extension 实现 UI 逻辑的类。可以看到，Today Extension UI 的载体是我们十分熟悉的 `UIViewController`。

对于 UI 的布局 Apple 君推荐使用 autolayout，如果不使用要注意在各种尺寸的屏幕上 Extension 都能正常显示。

要改变Today Extension的高度，可以设置`TodayViewController`的`preferredContentSize`属性。如：`self.preferredContentSize = CGSizeMake(0, 100);`

需要注意的是，在 Today Extension 中尽量不要包含能滚动的 scrollview，这给用户带来的体验将非常糟糕！

XCode 自动生成的`TodayViewController`声明支持`NCWidgetProviding`协议。

在`NCWidgetProviding`协议中包含两个方法：

- `- (UIEdgeInsets)widgetMarginInsetsForProposedMarginInsets:(UIEdgeInsets)defaultMarginInsets;`

如果需要修改 widget 在通知中心的左右边距可以实现该方法。

- `- (void)widgetPerformUpdateWithCompletionHandler:(void (^)(NCUpdateResult result))completionHandler;`

该方法用于 widget 更新其内容，当加载 widget 时系统会调用该方法获取新数据，更重要的是，当Notification Center不在前台显示时，系统也会"时不时"的调用该方法在后台获取新数据，并在获取到新数据时更新 widget snapshot。

要注意的是在该方法中需要调用其参数传递进来的`completionHandler`，调用参数如下：
+ NCUpdateResultNewData —— The new content required you to redraw the view;
+ NCUpdateResultNoData —— The widget doesn’t require updating;
+ NCUpdateResultFailed —— An error occurred during the update process.

当以`NCUpdateResultNewData`调用`completionHandler`时，系统会更新 widget snapshot。此刻是不是想到了 `NSURLSession`，是的，它们有异曲同工之妙——让用户看到的数据永远都是最新的！

widget 最喜欢做的一件事情恐怕就是呼起 Container App 了，方法也很简单，通过 `NSExtensionContext` 的 `openUrl` 方法实现：
![](/img/extensionopenurl.png)

有时候，我们可能不希望在Notification Center显示 widget，如没有可用的数据等，此时，我们可以调用 `NCWidgetController` 的类方法： `setHasContent:forWidgetWithBundleIdentifier:`，通过该方法可以轻松控制 widget 是否要显示，该方法既可以在 Container App 中调用，也可以在 widget 中调用。


# Conclusion
______________________________

Extension 作为 iOS8 推出的新特性，深受广大 App 欢迎。但我们始终要记住 Extension 不是正常的 App，只是 Container App 功能触角的延伸，其功能要简单、专注，UI 要简洁、明了，切不可滥用。


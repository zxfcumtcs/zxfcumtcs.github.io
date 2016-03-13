title: iOS Multitasking——Background Fetch
date: 2014-08-17 13:24:00
categories:
- IOS
tags:
- IOS
---
本文主要介绍 iOS7 引入的多任务处理机制之Background Fetch。
<!--more-->
©原创文章，转载请注明出处！

# Overview
__________
从 IOS4到 IOS7，Apple 在多任务处理上不断改进、优化。但 Apple 的态度一直很谨慎，其考虑的因素主要有两个：内存和电池续航能力。

是的，如果 Apple 完全放开多任务机制，那么，首当其冲的就是电池了，有限的内存恐怕也很快就会消耗殆尽，从而影响当前正在前台运行的程序。

在介绍 Apple 是如何处理多任务前，先了解一下 App 的几种不同状态：
![](/img/APPStatus.png)

ok，App 不同状态间转换可概括为[下图](https://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/ManagingYourApplicationsFlow/ManagingYourApplicationsFlow.html#//apple_ref/doc/uid/TP40007072-CH4)：
![](/img/StatechangesiOSapp.png)

下面，我们回顾一下 Apple 在多任务上的改进、优化之路：

1. iOS4之前想实现多任务，Apple 只会骄傲地说一句："no way!"，只要按 HOME 键切后台，app 立马被终止；
2. 严格的单任务运行似乎对不起"智能机""这个荣誉称号，因此从 iOS4开始 Apple 允许某些特定功能的 app 在后台运行，如：Play audio、VoIP以及Receive location updates等；
3. 从 iOS5开始，按 HOME 键只是暂停 app 的运行，而不是终止，其最大的好处在于保留了 app 的状态；
4. iOS6引入了"State Preservation and Restoration"的概念，在 app 终止前保存状态、下次启动时再恢复状态，从而给用户一个错觉——好像 app从来没有退出过一样；
5. 高大上的 iOS7则引入更多的多任务 API，包括：Background fetch、Background transfer以及Silent push notifications。


# The new app switcher
_______________
在介绍 iOS7的多任务机制前，还要介绍一下 iOS7对 app switcher 的改进。

从iOS4到iOS6，app switcher非常简单，只列出了那些曾经打开过的 app
![](/img/ios6appswitcher.png)

iOS7对 app switcher 进行了改进，在 app switcher 中能展示 app 最新的状态
![](/img/ios7appswitcher.png)

每当 app 因后台执行任务而被唤醒时，系统都会提供一个completion handler，当后台任务完成时，我们必须执行这个completion handler，其作用不仅在于告诉设备可以进入睡眠状态，同时会更新 app switcher 的snapshot。
app swithcer 还扮演着任务控制中心的角色，当用户从 app switcher清除某个 app 时，该 app 的大多数后台任务都将结束，如：background fetches, play music等，但background transfers依然可以进行。

ok, 下面我们就要今天的主题了！

# Background fetch
_____________
在 iOS7之前，当我们打开某个 app 时，该 app 总是先显示缓存的老数据(没做缓存的恶劣 app 恐怕只能以转菊花来消磨时间了！)，然后耐心地等待网络返回新数据。
这无疑是在浪费用户的时间，而"浪费别人的时间，就等于..."
![](/img/AppCrashed.jpg)

iOS7给了我们一个改过自新的机会——通过Background fetch可以在后台悄悄地把数据拉回来！

Background fetch通过特定的算法周期性地唤醒 frequently-used apps 在后台下载新数据。因此，当用户再次打开 app 时，看到的数据已经是最新的了，这是多么 cool 的体验！
![](/img/good.png)

那么，要实现这么 cool 的功能是不是很复杂？

no，only need three steps，its so easy！

1. 允许 app 通过 Background fetch 下载数据；
2. 在`application:didFinishLaunchingWithOptions: `方法中设置周期性拉取数据的时间间隔；
3. 实现`application:performFetchWithCompletionHandler:`方法，在该方法中处理 Background fetch 的具体任务。

### Background fetch的开关

![](/img/setBackgroundfetch.png)

这一步很简单，点两下鼠标就搞定！

### 设置时间间隔

`- (void)setMinimumBackgroundFetchInterval:(NSTimeInterval)minimumBackgroundFetchInterval`通过该函数可以设置系统唤醒 app 在后台执行Background fetch操作的时间间隔。

关于时间间隔有几个要注意的问题：
1. 该属性的默值是`UIApplicationBackgroundFetchIntervalNever`，也就是永远不会执行Background fetch操作。因此，如果期望 app 能执行Background fetch，就必须设置该属性为一个合理的值；
2. 该属性值只是一个建议值，而不是精确的值，系统将会综合各种因素决定什么时侯执行Background fetch操作，但可以肯定的是时间间隔肯定不会小于该值(图片来自 WWDC2013)；

![](/img/MinimumBackgroundFetchIntervalCustomValue.png)

系统会收集用户使用 app 的习惯，预测用户在什么时侯使用 app，从而决定什么时间执行 app 的Background fetch 操作。
比如：系统收集到用户习惯在早上9：00左右打开某新闻 app，则系统可能会在8：45唤醒该 app 执行Background fetch操作拉取最新的新闻数据(图片来自 WWDC2013)。
![](/img/AdaptstoUserActivity.png)

为了不干扰系统的这种自我学习能力，不要频繁的执行Background fetch操作，尽量使用系统预定义的值：`UIApplicationBackgroundFetchIntervalMinimum`
```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    [application setMinimumBackgroundFetchInterval:UIApplicationBackgroundFetchIntervalMinimum];
    
    return YES;
}
```

### 实现performFetchWithCompletionHandler方法

当系统允许 app 执行 Background fetch 操作时，会在后台唤醒该 app，流程如下(图片来自 WWDC2013)：
![](/img/Backgroundfetch.png)

在`application:performFetchWithCompletionHandler:`方法中要完成的任务有：获取数据、管理数据以及更新 UI 等。

同样，作为善后工作，我们必须调用系统提供的`completion handler`，该`completion handler`接受一个`UIBackgroundFetchResult`类型的参数。

系统为`UIBackgroundFetchResult`预定义了3个值：
1. UIBackgroundFetchResultNoData--没有获取到新数据；
2. UIBackgroundFetchResultNewData--获取到了新数据；
3. UIBackgroundFetchResultFailed--获取失败。

# Testing the Background Fetch
_______________
对于Background Fetch，我们该如何去测试呢？

有两种方法可以去测试Background Fetch

第一种方法稍微复杂一点，需要生成新的scheme。

![](/img/scheme.jpg)
![](/img/duplicatescheme.jpg)
![](/img/schemename.jpg)
![](/img/schemerun.jpg)

ok，通过这几步，app 直接在后台被唤醒，执行Background Fetch操作。

第二种方法是 app 正在前台运行时，操作Background Fetch操作。
![](/img/xcodebackgroundfetch.png)

是不是很简单！

ok，今天就到这里了，下期将介绍另外一种Multitasking机制：Silent push notifications


title: iOS Multitasking——Silent Push Notification
date: 2014-10-06 17:20:51
categories:
- IOS
tags:
- IOS
---
本文主要介绍 iOS7 引入的多任务处理机制之Silent Push Notifications
<!--more-->
©原创文章，转载请注明出处！


# Overview
_______________

IOS7引入了三种新的多任务机制，分别是`Background Fetch`, `Background Transfer` 以及 `Silent Push Notification`。

其中， [Background Fetch](http://zhaoxuefeng.gitcafe.com/2014/08/17/multitasking/)、[Background Transfer](http://zhaoxuefeng.gitcafe.com/2014/08/03/nsurlsession3/) 在前文已经介绍过，本文重点介绍`Silent Push Notification`。


OK, 那么何为`Silent Push Notification`？ 听起来有点玄乎！

很显然，重点在**Silent**, 简单地说就是用户无感知的 push，专业说法是当系统收到**Silent Push**时在后台启动 app 去完成相应的任务。

ps: 为了描述方便，在下文我们将正常的 push 称为 `Normal Puah`

# Silent Push VS. Normal Push
_______________

我们已经知道，对于Normal Push，当用户去响应该 push 时，系统会启动 app 并在前台运行；Silent Push 对用户无感知，当该类型 push 到来后，系统在后台启动 app。这恐怕就是它们最直观的区别了，当然对于一个专业的程序猿来说，只知道这些太少！

那就继续吧……

首先，我们看看 Normal Push 的流程([WWDC2014](https://developer.apple.com/videos/wwdc/2014/))：

![](/img/NormalPush.png)

有没有觉得不对劲？

是的，push 直接由系统接收、显示，在用户响应 push 之前，app 只能在一旁凉快着，无法登台！只有当用户响应 push 时，系统才会启动 app。

问题在于：当用户响应该 push，系统启动 app 时，与该 push 相关的内容还远在天边的 sever 上，需要用户等待新内容的到来！！！

世界上最远的距离不是 sever 与 client 间的距离，而是已收到的 push 与 app 间的距离！

在追求极致体验的 Apple 看来，该问题需要改进！

喂！铺垫够了吧，该让主角登台了！

OK, 我们再来看看 Silent Push：

![](/img/SilentPush.png)

是的，系统接收到 Silent Push 之后，无需用户响应(当然用户也无法响应，因为他根本不知道！)，直接在后台启动 app 去获取内容

是不是很 cool！:)

那么，是什么因素决定了一个 push 是 silent or normal？

嗯，关键点在于push 的 payload：

Normal push：

![](/img/NormalPushPlayload1.png)

Silent push:

![](/img/SilentPushPlayload1.png)

富有想象力的鞋童可能会想，能否珠联璧合？

没错，这是可行的！

![](/img/SilentAndNormalPushPlayload1.png)

通过它，可以很好的解决上面提到的normal push的问题

![](/img/SlientAndNormalPush.png)

简单地讲，就是系统接收到 push 时，一边通知用户，一边在后台启动 app 去获取数据，当用户响应该 push 时，数据可能已经准备好了！
 
如果你非要钻牛角尖，说："push 来了我立马点它，这时即使已经在后台获取数据了，但可能数据还没返回，我不还是要等吗！"

我只能说："严谨的态度值得学习！"

看看 Apple 官方文档咋说的(来自[App Programming Guide for iOS](https://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/BackgroundExecution/BackgroundExecution.html#//apple_ref/doc/uid/TP40007072-CH4-SW57))：

> **Using Push Notifications to Initiate a Download**
> If your server sends push notifications to a user’s device when new content is available for your app, you can ask the system to run your app in the background so that it can begin downloading the new content right away. The intent of this background mode is to minimize the amount of time that elapses between when a user sees a push notification and when your app is able to able to display the associated content. Apps are typically woken up at roughly the same time that the user sees the notification but that still gives you more time than you might have otherwise.



(来自[UIApplicationDelegate Protocol Reference](https://developer.apple.com/library/ios/documentation/uikit/reference/uiapplicationdelegate_protocol/index.html#//apple_ref/occ/intfm/UIApplicationDelegate/application:didReceiveRemoteNotification:fetchCompletionHandler:))
>When a push notification arrives, the system displays the notification to the user and launches the app in the background (if needed) so that it can call this method. Launching your app in the background gives you time to process the notification and download any data associated with it, minimizing the amount of time that elapses between the arrival of the notification and displaying that data to the user.

>As soon as you finish processing the notification, you must call the block in the handler parameter or your app will be terminated. Your app has up to 30 seconds of wall-clock time to process the notification and call the specified completion handler block. In practice, you should call the handler block as soon as you are done processing the notification. The system tracks the elapsed time, power usage, and data costs for your app’s background downloads. Apps that use significant amounts of power when processing push notifications may not always be woken up early to process future notifications.

是的，对于 silent push， app 有30秒钟的机会在后台运行，一般可以启动一个 background download 任务去获取数据。

为了实现 silent push，除了上面提到的在 payload 中添加content-available: 1外，还需要设置相应的 background model：

要么在 info.plist 中：

![](/img/SilentpushInfoplist.png)

或者在工程文件中：

![](/img/SilentPushProject.png)

# Push and Delegate
_______________

系统收到 push 时 app 大概处于3种状态：

1. app 正在前台运行；
2. app 正在后台运行或处于挂起状态，总之 app 还处于内存中；
3. app 已终止运行。

当 app 处于前台运行时，收到 push，无需用户响应，系统直接调用`application:didReceiveRemoteNotification:fetchCompletionHandler:`

当 app 处于后台或挂起状态时，收到 push，需要用户响应，此时系统也会调用`application:didReceiveRemoteNotification:fetchCompletionHandler:`

那么，如何区分上述两种情况呢？

可以在`application:didReceiveRemoteNotification:fetchCompletionHandler:`中判断 app的状态，如果`applicationState == UIApplicationStateActive`, 则属于第一种情况，若`applicationState == UIApplicationStateInactive`, 则属于第二种情况。

当 app 已经终止时，收到 push，对于普通 push，当用户响应时会调用`application:didFinishLaunchingWithOptions:`, 对于 silent push：
![](/img/slientpushdelegate.png)

ps:Apple 会控制 silent push 的频率
![](/img/Silentpushesareratelimited.png)

# Backgroud Fetch VS. Silent Push
_______________

[前文](http://zhaoxuefeng.gitcafe.com/2014/08/17/multitasking/)，我们介绍过Backgroud Fetch，可以通过Backgroud Fetch在后台获取数据。

Silent Push 也可以在后台触发 app 获取数据。

那么问题来了：在后台获取数据哪家强？

什么时候用Backgroud Fetch，什么时候用 Silent Push？

ok，看看 Apple 君是如何说的：

![](/img/Backgroundfetchandsilentpush.png)

还有一个区别：Background Fetch 何时发起是由系统控制的，而 Silent Push 基本是由我们自己控制的。


# 总结
_______________

Silent Push 是一项非常cool 的特性，在特定的应用中能明显提升用户体验。

如 Apple 列举的几个例子：
![](/img/SlinetPushuserfor.png)

在日常项目中，我们还可以用 silent push 来抓取 log，如：某用户返回问题，为了定位问题可以下发 silent push 抓取该用户的 log。

最后一个需要注意的问题：`Background Fetch`、`Background Transfer`以及`Silent Push Notification`都会在后台启动 app，但如果用户在app switcher中强制退出 app，则是不会自动启动的。总之，除了在 IOS8下极个别的情况，只要用户 kill 掉 app，系统是不会自动重启的。
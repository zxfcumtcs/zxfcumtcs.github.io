---
title: NSURLSession 拾遗
date: 2016-06-09 17:51:28
tags:
- iOS
- NetWorking
---
本文总结了实际开发过程中在 NSURLSession 上遇到的各种问题。
<!--more-->
©原创文章，转载请注明出处！

# Overview
____________________________
NSURLSession 是 Apple 在 iOS7 推出的一套网络组件库，用于取代 NSURLConnection(在 iOS9 中已被废弃)。
> 本文并不是 NSURLSession 的科普文章(这类文章网上已有很多，也可以参考之前的文章：[NSURLSession—网络框架新生代](http://zxfcumtcs.github.io/2014/08/02/NSUrlSession/))。

最近在项目中对下载模块做了一次重构，其中用到 NSURLSession，期间遇到了一些有趣的问题，故小结一下。

# 网络错误
____________________________
在调试过程中发现，当断开网络时并没有收到回调：`URLSession:task:didCompleteWithError:`。
原来，从 iOS8 开始对于`background mode`的 NSURLSession，当无法连接服务器时，NSURLSession 并不会调用`URLSession:task:didCompleteWithError:`，而是让上传、下载任务处于空闲等待状态，当网络恢复后，NSURLSession 自动恢复之前的任务。

iOS7 上若无法连接服务器会立即调用上述回调方法并返回错误：
Error Domain=NSURLErrorDomain Code=-1009 "The Internet connection appears to be offline."

# timeoutIntervalForResource
____________________________
上小节了解到当无法连接服务器时，NSURLSession 并不会将这类问题反馈给业务层，但有时业务层可能需要了解这一事实，从而在 UI 上给出相应的提示。此时可以通过 NSURLSessionConfigure 的`timeoutIntervalForResource`属性设置一个超时时间。

在此，需要区分清楚 NSURLSessionConfigure 的两个属性`timeoutIntervalForResource`与`timeoutIntervalForRequest`的区别：
+ timeoutIntervalForResource：整个任务(上传或下载)完成的最长时间，默认值为1周；
+ timeoutIntervalForRequest：两个数据包之间的最大时间间隔；

timeoutIntervalForResource指整个任务完成的最大时间，简单讲就是 NSURLSession 里面所有 task 完成的总时间(包括因控制最大并发数而排队等待的时间)。

# 自动重试
____________________________
在调试过程中还发现一个有趣的问题，当把网络调成弱网时：
![](/img/downloadTask-didWriteData-totalBytesWritten-totalBytesExpectedToWrite.png)
上图是在 `URLSession:downloadTask:didWriteData:totalBytesWritten:totalBytesExpectedToWrite:` 中打印的 log，理论上totalBytesWritten应该是递增的，但从上图可以看到出现了断裂，从168640降到了3906，而3906恰是本次接收的字节数(bytesWritten)。
这点说明 NSURLSession 在弱网络下出现失败时会自动重试。

那么，这点对我们有何影响呢？
+ 业务层基本上不需要再进行重试操作了(以前我们经常会在失败时进行多次重试)；
+ 如果需要显示进度，需要作特殊处理，不然可能会出现抖动。

# background mode
_____________________________

background 模式下的 NSURLSession 可以在后台执行相应的任务，此时 app 可能处于 background、suspended 或 terminated 状态。当 app 处于 suspended 或 terminated 状态时是不能执行任何代码的，那么下载任务是如何执行的呢？
对于 background mode NSURLSession，所有的上传、下载任务都会交由系统在一个独立的进程中执行。
通过 instruments 可以看到这个进程：
有下载任务时：![](/img/NSURLSessionProcess.png)
处于空闲状态时：![](/img/NSURLSessionProcess1.png)

*ps：如果用户在任务中心强制将 app kill 掉，所有 background task 都会被 cancel。*

# identifier
_____________________________
在创建 background 模式的 NSURLSession 时需要提供一个唯一的`identifier`，***这里的『唯一』不仅仅是在 app 内部，而是在整个系统中都要求是唯一的(因为所有的 background session 都是在同一个进程中处理)。***因此，可以将bundle ID拼接到identifier中。

同时，需要将正在执行中的 session identifier存储起来，当 app crash、suspended 或 terminated 后可以通过存储的 identifier 恢复尚未完成的 sesssion。

# delegate
_____________________________
NSURLSession 会对 delegate 保持强引用，因此像 NSTimer 一样，在任务完成时需要调用 `invalidateAndCancel` 或 `finishTasksAndInvalidate` 方法，否则会出现内存泄漏。

# deep copy NSURLSessionConfigure
____________________________
在创建 NSURLSession 时，会对 NSURLSessionConfigure 进行深拷贝，也就是在创建 NSURLSession 后再对 configure 进行修改不会影响之前创建的 session。

# URLSession:downloadTask:didFinishDownloadingToURL:
_____________________________
在下载任务完成后，NSURLSession 会回调该方法，在该方法返回前需要将下载文件(保存在`location`参数指定的位置)读取到内存或拷贝到其他地方，在该方法返回后系统会自动删除`location`处的下载文件。

# application:handleEventsForBackgroundURLSession:completionHandler:
_____________________________
当 app 处于 suspended 或 terminated 状态，若此时有 background session 完成任务或需要鉴权，系统会调用该方法。通过该方法系统会传递一个 block 类型的 `completionHandler`过来，在我们处理完 session 相关的任务后，需要执行该 block，在执行完该 block 后，系统大概在5秒后再次被挂起。

# cancel
_____________________________
当 cacel 正在执行中的 session task 时，NSURLSession 会回调 `URLSession:task:didCompleteWithError:` 方法，并返回错误信息：
Error Domain=NSURLErrorDomain Code=-999 "cancelled"

# HTTP/2
_____________________________
在 NSURLSession 带来的诸多利好中还有一个就是在 iOS9 上自动支持 HTTP/2，无需使用方额外做任何事情。
HTTP/2 与 HTTP/1.1、HTTP/1.0相比，特大的特点就是『快』。
[HTTP/2 makes media loading 3–15 times faster on mobile](https://medium.com/apps-and-networking/http-2-makes-media-loading-3-15-times-faster-on-mobile-a455c3e68135#.1ptt4gx4s) 这篇文章对 HTTP/2 与 HTTP/1.1 在速度上做了对比：
![](/img/http2vs1wifi.jpg)
![](/img/http2vs13g.png)

HTTP/2 与 HTTP/1.1相比主要区别有：
+ 每个 Host 只需一个 TCP connection；
+ 连接多路复用(HTTP/1.0 中一个连接一次只能有一个请求、HTTP/1.1 中一个连接同时可以有多个请求，但一次只能有一个响应，HTTP/2则完全是多路复用的)；
+ request 可以设置优先级；
+ 二进制协议(HTTP：超文本传输协议，以前是文本协议)；
+ 头部压缩；
+ 服务器可以主动 push。


### 参考资料
[URL Session Programming Guide](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html)
[NSURLSession Class Reference](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSession_class/)
[NSURLSessionTask never calls back after timeout when using background configuration](http://stackoverflow.com/questions/23288780/nsurlsessiontask-never-calls-back-after-timeout-when-using-background-configurat)

[HTTP/2 Frequently Asked Questions](https://http2.github.io/faq/)
[HTTP/2 makes media loading 3–15 times faster on mobile](https://medium.com/apps-and-networking/http-2-makes-media-loading-3-15-times-faster-on-mobile-a455c3e68135#.1ptt4gx4s)


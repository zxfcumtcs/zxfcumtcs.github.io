title: NSURLSession——网络框架新生代(3/3)
date: 2014-08-03 15:51:08
categories:
- IOS
tags:
- NetWorking
- IOS
---
本系列文章将探讨Apple在IOS7中引入的网络新框架——NSURLSession。
<!--more-->
©原创文章，转载请注明出处！

# NSURLSession vs. NSURLConnection

NSURLSession 打出的旗号是推翻 NSURLConnection，取而代之！嗯，是的，这是个弱肉强食的时代，但同时也要遵守达尔文的进化论——优胜劣汰！

少年，你的优势在哪，凭什么取代别人！

ok，那我们来总结一下：

1. Background Transfers——当 app 处于crashes、terminated or suspended status 还能 upload 或 download data。而要实现 background transfers即非常简单，只需配置一个 background configuration。

2. Pause and Resume——NSURLSession提供了原生的 API 支持 pause 以及 resume 网络操作。

3. 面向 session 的 configuration——每个 NSURLSession 都有自己的configuration，该configuration对 session 内的所有 tasks 有效，而与其他的 session 相互独立，互不干扰。而NSURLConnection时代的 configuration 是全局的。

4. 隐私性——通过 ephemeral configuration可以创建更加隐私的 session，进行一些 private操作。
5. 改进的authentication处理方法——NSURLSession 的authentication是基于连接的，而 NSURLConnection 的连接是基于请求的。
6. 更加丰富的 delegate model——在 NSURLConnection 时代，要么使用 delegate，要么使用 block，休想鱼和熊掌兼而得之。当 block 遇到 authentication 时，注定是悲剧的结局。但在 NSURLSession 时代，可以放心地使用 block，在遇到 authentication 时同样可以使用 delegate 去处理。
7. 支持文件系统的upload and download——可以直接上传文件或下载数据到文件，更好的分离了data and metadata。

ok，NSURLConnection 君虽然曾经你作为基础网络框架服务于千万 iOS and Mac OS 程序，但那已经是曾经了……，该回家养老了！

# NSURLSession vs. AFNetworking

说到网络框架，那不得不提到AFNetworking君，作为著名的第三方开源Networking Framework，已得到广泛应用。

随着 NSURLSession 的到来，AFNetworking也进入2.0时代。

那么什么时候使用NSURLSession，什么时候使用AFNetworking呢？

基本指导原则是，如果没有复杂的操作，可以直接使用NSURLSession，这样能避免引入第三方库，而如果希望使用 AFNetworking 2.0的新特性，如：serialization、 further UIKit integration等，那就用之吧。

# Background Transfer实战

为了避免有纸上谈兵之嫌，我们来一次Background Transfer实战吧。

由于background transfer可能发生在 app 处于 out-process 状态，因此 background transfer 是在一个独立的进程中进行的。这就导致background transfer有以下限制：
1. session 必须提供 delegate；
2. 只支持 HTTP、HTTPS 协议；
3. 只支持upload and download tasks；
4. 总是允许重定向；
5. 若background transfer是 app 处于background状态发起的时，configuration的discretionary属性当作 YES 处理。

那么 background transfer 时，与 app 的交互是咋样的呢？

1. 当background transfer完成或需要鉴权时，若 app 处于out-process 状态，此时 iOS 自动在后台启动 app，并调用`UIApplicationDelegate`的`application:handleEventsForBackgroundURLSession:completionHandler:`方法，该方法的`identifier`参数表示由吗个 session 导致启动 app，在该方法中我们需要保存`completionHandler`参数传递过来的 block；
2. 当 session 完成所有background transfer tasks 时 `URLSessionDidFinishEventsForBackgroundURLSession:`，调用 session 的`URLSessionDidFinishEventsForBackgroundURLSession:`方法，在该方法中必须执行第一步中保存的`completionHandler` block。

少年，至少要上些 code 才能叫实战吧！

![](/img/handleEventsForBackgroundURLSession.png)

![](/img/URLSessionDidFinishEventsForBackgroundURLSession.png)

![](/img/urlsessiondownloadTask.png)

关键代码就这些，为了更好地了解完成情况，我们加了个LocalNotification。

![](/img/LocalNotification.jpg)

background transfer可以广泛应用到我们的 app 中，不断提升用户体验。如：有内容更新时，可以由服务器下发 remote notification， app 收到 notification 后在后台启动background transfer task，当用户再次打开 app 时更新的内容已经下载，可以直接阅读！

今天就到此结束，网络开发还有很多内容值得探讨，后会有期……

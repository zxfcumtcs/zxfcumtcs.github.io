title: NSURLSession——网络框架新生代(2/3)
date: 2014-08-02 23:03:47
tags:
- NetWorking
- iOS
---
本系列文章将探讨Apple在IOS7中引入的网络新框架——NSURLSession。
<!--more-->
©原创文章，转载请注明出处！

# delegate vs. completionHandler

前文我们介绍了用于创建`NSURLSession`的三个类方法，其中一个需要指定`delegate`，而另外两个方法创建的`NSURLSession`则需要其内部的task提供`completionHandler`。

那么，什么时侯用`delegate`，什么时侯用`completionHandler`呢？

首先，需要说明的是，当我们没有提供`delegate`时，`NSURLSession`会使用`system-provided delegate`处理各种网络逻辑，但不会处理业务层逻辑，因此此时必须由task提供`completionHandler`处理业务层的细节。

喂，少年，刚才那个问题还没回答呢！

+ 当App处于background、suspended以及out-of-process状态时，需要由background sessions上传或下载数据；

+ 自定义网络鉴权；

+ 自定义SSL验证；

+ 由服务器返回的MIME type决定一个网络传输是下载到磁盘还是直接display；

+ 通过body stream上传数据；

+ 自定义cache；

+ 自定义HTTP redirects；

嗯，除了上述情况之外，我们可以只提供`completionHandler`，其他繁琐之事就交由 `system-provided delegate`去处理吧^_^

# URL Session的生命周期

## URL Session with System-Provided Delegates

前面讲到，当我们没有为 session 提供 delegate 时，session 会使用system-provided delegate。

使用`System-Provided Delegate`执行网络任务的流程如下：

1. 创建session configuration----对于background session，需要提供唯一的identifier，当app crashes、terminated or suspended后，可以用该identifier重新关联background session；

2. 创建session----用第一步得到的configuration创建session，并且不要提供delegate；

3. 创建session task----task创建后处于suspended状态，需要调用`resume`方法启动该task；

4. 对于download task，可以暂停该task，之后又能恢复；

5. task完成后，NSURLSession调用task的completionHandler处理业务逻辑；

6. 当session完成其使命时，需要调用`invalidateAndCancel` 或 `finishTasksAndInvalidate`使之失效。

## URL Session with Custom Delegates

嗯， 使用system-provided delegate感觉非常之良好！我们只需在创建 task 时提供一个`completionHandler`就行了。but，想法是美好的，现实是残酷的，前面已经提到某些情况是不能使用system-provided delegate！

少年，既然不能改变世界，那就去适应它吧！

首先，我们来见识一下与 session 相关的 delegate：
![](/img/nsurlsessiondelegate.png)

那么，这些 delegate 都能做些什么事情呢？

1. 对于 download tasks，session 使用 delegate 为我们提供 downloaded data 所在的文件路径；

2. 处理鉴权；

3. 为stream-based upload task 提供body streams；

4. 决定是否允许 HTTP 重定向；

5. session 使用 delegate 向我们提供 task 执行的各种状态；

6. session 通过 delegate 告诉我们 task 完成。

ok，使用custom delegate流程如下：

1. 创建 configuration；
2. 创建 session，同时指定custom delegate；
3. 创建 task；
4. 如果服务器返回状态码表示要鉴权，并且需要connection-level鉴权时(如SSL验证)，session调用与鉴权相关的 delegate 方法；
5. 如果服务器需要重定向，对于background session，重定向是自动允许的，无需调用任何 delegate 方法。对于default and ephemeral sessions，会调用`URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler`方法，在该方法内允许重定向时需要以`request`为参数调用`completionHandler`，不允许时以`nil`调用`completionHandler`。允许重定向，相当于发起新请求，返回第4步，不允许重定向时，整个请求结束；
6. 如果 download task 是通过`downloadTaskWithResumeData:` or `downloadTaskWithResumeData:completionHandler:`方法创建的，在该task 开始时，会调用`URLSession:downloadTask:didResumeAtOffset:expectedTotalBytes:`方法；
7. 对于data task，当收到服务器响应时调用`URLSession:dataTask:didReceiveResponse:completionHandler:`方法，在该方法中可收决定是否要将 data task 转换为 download task；
8. 如果 task 是由` uploadTaskWithStreamedRequest:`方法创建，session 调用`URLSession:task:needNewBodyStream:`方法来提供body data；
9. 对于 upload task，在上传数据过程中，session 周期性的调用`URLSession:task:didSendBodyData:totalBytesSent:totalBytesExpectedToSend:`反馈上传进度；
10. 对于 download task，在从服务器传输数据过程中，session 周期性的调用`URLSession:downloadTask:didWriteData:totalBytesWritten:totalBytesExpectedToWrite: `反馈下载进度,对于 data task， 当有数据到达时session调用 `URLSession:dataTask:didReceiveData:`接收数据；
11. 对于 data task，在其接收完所有expected data后，session 调用`URLSession:dataTask:willCacheResponse:completionHandler:`方法处理与 cache 相关的事情。如果 delegate 没有实现该方法，则使用session configuration提供的默认 cache 行为。如果，我们实现了该方法，必须在该方法内部调用`completionHandler`，否则会引起**内存泄漏**。我们可以以original proposed response、a modified version of that response or NULL to prevent caching the response为参数来调用`completionHandler`。那么什么情况下，需要实现该方法呢？a. 不 cache 特定 url 的 response、b. 修改 response 的userInfo dictionary；
12. 当 download task 完成时，session 调用`URLSession:downloadTask:didFinishDownloadingToURL:`方法，在该方法中必须对下载的临时数据文件作处理(read or move)，task 完成后 session 将删除该临时文件；
13. 任何 task 完成时，session 都会调用` URLSession:task:didCompleteWithError:`方法；
14. 如果 response 是multipart encoded，session可能会再次调用`URLSession:dataTask:didReceiveResponse:completionHandler:`方法，随后可能会多次调用`URLSession:dataTask:didReceiveData:`方法；
15. 当session完成使命时，调用`invalidateAndCancel`或`finishTasksAndInvalidate`使之失效，随后当所有outstanding tasks都被取消或完成时，session 调用`URLSession:didBecomeInvalidWithError: `方法，该方法返回时 session 解除对 delegate 的strong reference。自此，session 彻底退出历史舞台。

呵呵，有点长，小二上几张图清醒一下(来自 WWDC2014)：

![](/img/datatask.png)

![](/img/downloadtask.png)

title: NSURLSession——网络框架新生代(1/3)
date: 2014-08-02 10:04:31
categories:
- IOS
tags:
- NetWorking
- IOS
---
本系列文章将探讨Apple在IOS7中引入的网络新框架——NSURLSession。
<!--more-->
©原创文章，转载请注明出处！

#Overview

`NSURLSession`作为网络框架新生代，由Apple在IOS7引入，其终极目标是取代之前一统网络江湖的`NSURLConnection`。

ok，首先看看`NSURLSession`在IOS体系中的地位(图片来自WWDC2014)。

![](/img/nsurlconnection.png)  ![](/img/nsurlsession.png)

嗯，从上面两张图也能再次看到`NSURLConnection`最终会被取代的悲惨命运。
同时，我们还能看到，无论是`NSURLSession`还是`NSURLConnection`都是构建在BSD Networking基础之上。

严格地说，`NSURLSession`不是孤身一人在战斗，而是一个家族，一个框架。
![](/img/nsurlsessionclassdigram.png)

上图中除了熟悉的老朋友`NSURLRequest`和`NSURLResponse`之外，其他都是Apple在IOS7引入的新面孔，下面我们一一简单介绍一下。

#NSURLSession家族

###NSURLSessionConfiguration——网络大管家

顾名思义，`NSURLSessionConfiguration`就是用于配置`NSURLSession`在网络交互过程中的各种属性。

![](/img/NSURLSessionConfiguration.png)

从其属性可以看出，其能配置的网络属性非常之丰富。我们常用的属性如：`allowsCellularAccess`----是否允许访问蜂窝网、`HTTPMaximumConnectionsPerHost`----最大连接数、`requestCachePolicy`----cache策略等等。

根据实际情况需要，Apple提供三种类型的`NSURLSessionConfiguration`：

+ Default Configuration----通过Default Configuration创建的`NSURLSession`其行为与`NSURLConnection`类似，使用全局的、disk-based的cache、cookie、credential等。

+ Ephemeral Configuration----通过Ephemeral Configuration创建的`NSURLSession`仅将cache、cookie、credential等信息保存在内存中，当session失效时这些信息也一并释放。因此该类型的session可以用于一些隐私的网络操作。

+ Background Configuration----顾名思义，通过Background Configuration创建的`NSURLSession`主要用于那些当App处于background、suspended以及out-of-process状态时还需要进行网络操作的任务。

`NSURLSessionConfiguration`提供了三个类方法用于分别创建上述Configuration：

   `+ (NSURLSessionConfiguration *)defaultSessionConfiguration;`
	
   `+ (NSURLSessionConfiguration *)ephemeralSessionConfiguration;`
	
   `+ (NSURLSessionConfiguration *)backgroundSessionConfiguration:(NSString *)identifier;`
	
其中`backgroundSessionConfiguration:`的`identifier`参数在整个app中必须是唯一的，如果出现重复其后果是严重的！
通过该`identifier`，我们可以创建新的background session，用于获取与其绑定的download or upload response。


###NSURLSessionTask——网络办事员

在每个组织、单位都会有一些领导、当然更多的是办事员了。呵呵，没错，`NSURLSessionTask`家族就处于社会底层——苦逼的办事员！

![](/img/NSURLSessionTaskMethod.png)

我们惊奇地发现，`NSURLSessionTask`家族竟然没有自己的初始化方法！

是的，官方资料显示，`NSURLSessionTask`必须归属于`NSURLSession`，由后者提供相应的creation methods。从而`NSURLSessionTask`也就失去了人身自由，一辈子也无法跳槽！

![](/img/AppCrashed.jpg)

根据任务的不同，又演化出三种类型的task，它们间的关系如下([来源](http://www.raywenderlich.com/51127/nsurlsession-tutorial))：

![](/img/nsurlsessiontask.png)

+ NSURLSessionDataTask----用于向服务器请求资源(如：json、xml格式的数据等)，服务器返回的资源以`NSData`的形式提供给业务层解析。其只能用于default和ephemeral sessions，而不支持background sessions。

	`NSURLSession`类中用于创建`NSURLSessionDataTask`的方法有：
	```
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request;
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;
- (NSURLSessionDataTask *)dataTaskWithURL:(NSURL *)url;
- (NSURLSessionDataTask *)dataTaskWithURL:(NSURL *)url completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;
	```
+ NSURLSessionUploadTask----用于通过HTTP协议向服务器上传数据或文件，其继承于`NSURLSessionDataTask`，刚开始对这点感到很奇怪，感觉`NSURLSessionDataTask`和`NSURLSessionUploadTask`有点风马牛不相及，其原因在于upload过程中服务器可能会向client反馈一些信息。但`NSURLSessionUploadTask`是支持background sessions的。

	`NSURLSession`类中用于创建`NSURLSessionUploadTask`的方法有：
	```
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromData:(NSData *)bodyData;
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromData:(NSData *)bodyData completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromFile:(NSURL *)fileURL;
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromFile:(NSURL *)fileURL completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;
- (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request;
	```
	
+ NSURLSessionDownloadTask----用于download文件，在download过程中会将服务器返回的数据写入临时文件。注意：在download完成时需要我们手动将文件移走，临时文件在task完成后会被删除。download tasks可以pause、resume。

	我们可以看一下这些临时文件是保存在哪里的：
	![](/img/nsurlsessiondownloadpath.png)

	`NSURLSession`类中用于创建`NSURLSessionDownloadTask`的方法有：
	```
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request;
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler;
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData;
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler;
- (NSURLSessionDownloadTask *)downloadTaskWithURL:(NSURL *)url;
- (NSURLSessionDownloadTask *)downloadTaskWithURL:(NSURL *)url completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler;
	```
	
###NSURLSession——最高领导

ok，领导终于露面了！

简单地讲，`NSURLSession`通过`NSURLSessionConfiguration`管理网络配置，通过`NSURLSessionTask`执行具体的网络任务。

在一个session中可以有多个task，而configuration是针对于session的，该session中的所有task共享configuration。

![](/img/sessiontaskconfigure.png)

上图显示(来自WWDC2014)，在一个session中有三个task，它们都服从session中configuration的管理。

处于session中的task，其行为由三个因素决定：

+ session的类型，更准确地说是session所使用的configuration的类型(default、ephemeral以及background)；

+ task本身的类型(datatask、uploadtask以及downloadtask)；

+ task创建时app所处的状态(是否处于foreground)。

那么，如何创建session呢？

`NSURLSession`提供了三个类方法：

```
// 使用default configuration并且该session中的task必须提供completionHandler
+ (NSURLSession *)sharedSession;

// 该session中的task必须提供completionHandler
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration;

+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration delegate:(id<NSURLSessionDelegate>)delegate delegateQueue:(NSOperationQueue *)queue;

```

在使用`NSURLSession`过程中有两个需要注意的问题：

+ 在创建`NSURLSession`时会copy其使用的configuration，因此在此之后修改configuration，对该session无效；

+ `NSURLSession`对delegate是strong reference，直到我们显式地调用 `invalidateAndCancel` 或 `resetWithCompletionHandler:`方法。因此，在使用过程中要注意retain cycle。
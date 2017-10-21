---
title: 细说 NSOperation
date: 2016-05-17 23:25:12
tags:
- iOS
---
本文详细介绍了在实现同步、异步 NSOperation 时分别需要实现哪些方法、注意哪些问题。最后对 GCD 与 NSOperation Queue 作了一个简单的对比。
<!--more-->
©原创文章，转载请注明出处！

# Overview
____________________________
在 iOS 中实现并发编程主要有三种方式：GCD、NSOperation Queue以及Thread，其中前两者使用广泛。
在正式开始之前有必要区分两组概念：同步、异步与串行、并行。
+ 同步(Synchronous)、异步(Asynchronous)通常指方法(或函数)，同步方法表示直到任务完成才返回(如：`dispatch_sync`)，异步方法则是将任务抛出去，在任务没完成前就返回(如：`dispatch_async`)；
+ 串行(Serial)、并行(Concurrent)通常指 App 执行一组任务的模式，串行表示一次只能执行一个任务，只有当前一个任务完成后才启动下一个任务，而并行指可以同时执行多个任务。最常见的莫过于 GCD 中的串行、并行队列。

NSOperation Queue + NSOperation 作为 iOS 中『高级的、面向对象的并发编程方式』耳熟能详，但具体到一些细节问题上认识往往又比较模糊。本文在苹果官方文档 [Concurrency Programming Guide](https://developer.apple.com/library/ios/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html)、[NSOperation Class Reference](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/NSOperation_class/index.html) 以及 [NSOperationQueue Class Reference](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/NSOperationQueue_class/index.html) 的基础上做了一次疏理和总结。

# NSOperation
______________________________
`NSOperation` 本身是个抽象类，在使用前必须子类化(系统预定义了两个子类：`NSInvocationOperation` 和 `NSBlockOperation`)。那问题来了，在子类化过程中，需要重写父类的哪些方法？

这首先就要了解一下`NSOperation`类中几个重要方法的默认实现：
![](/img/NSOperationDefault.png)
在 NSOperation 中还有一个重要概念：operation 的状态，并且当状态变化时需要通过 KVO 的方式通知外：
![](/img/NSOperationKeypath.png)
回到前面那个问题：子类化 NSOperation 时需要重写哪些方法？
这取决于子类化后的 operation 是 Synchronous 还是 Asynchronous(NSOperation 默认是Synchronous)。

## Synchronous VS. Asynchronous Operations
_______________________________
由于操作 NSOperation 与 NSOperation 任务的执行往往在不同的线程上进行，在继续之前需要强调线程安全问题：『NSOperation 本身是 thread-safe，*当我们在子类重写或自定义方法时同样需要保证 thread-safe*』。

### Synchronous Operations
对于 Synchronous Operation，在调用其 `start` 方法的线程上同步执行该 operation 的任务，`start` 方法返回时 operation 执行完成。因此，对于 Synchronous Operation 一般只需重写 `main` 方法即可(`start`方法的默认实现已实现相关 KVO 功能)。

### Asynchronous Operations
然而对于 Asynchronous Operation，调用其 `start` 方法后，在 `start` 返回时 operation 的任务可能还没完成(为了实现异步，一般需要在其他线程执行 operation 的具体任务)。因此 `start` 方法默认实现不能满足异步需要(默认实现会在`start`返回前将 `isExecuting` 置为 NO、`isFinished` 置为 YES，并产生 KVO 通知)。此时至少需要重写以下方法：
+ start：
	我们知道 NSOperation 本身不具备并发(或者说异步执行)能力，因此需要 `start` 方法来实现，可以通过创建子线程或其他异步方式完成。同时需要在任务开始前将 `isExecuting` 置为YES 并抛出 KVO 通知。
	***『重写的 `start` 方法一定不能调用 `[super start]`』***
+ asynchronous
	返回 YES，一般不需要抛出 KVO 通知
+ executing
	返回 operation 的执行状态，在其值发生变化时需要在 `isExecuting` 上抛出 KVO 通知
+ finished
	返回 operation 的完成状态，同样值变化时需要在 `isFinished` 上抛出 KVO 通知

这里我们看看著名的网络框架 AFNetworking 中关于 NSOperation 的使用：
> AFNetworking 3.0 全面使用 `NSURLSession`，而 `NSURLSession` 本身是异步的、且没有 `NSURLConnection` 需要 runloop 配合的问题，因此在3.0版本中并没有使用 NSOperation，代码得到很大的简化。这里我们说的是 AFNetworking 2.3.1 版本。

在 AFNetworking 中 AFURLConnectionOperation 是个异步的 NSOperation 子类，其 `start` 方法如下：
![](/img/AFURLConnectionOperation-start.png)
从上面 `start` 方法的实现可以看到：
1. 用 lock(递归锁) 保证了thread-safe；
2. 检查了 operation 是否已被 cancel；
3. 检查了 operation 是否已 ready；
4. 通过子线程实现并发；
5. 在 state setter 中实现了 KVO。
![](/img/AFURLConnectionOperation-statesetter.png)

再来看看 AFURLConnectionOperation 使用的子线程：
![](/img/AFURLConnectionOperation-thread.png)
可以看到，所有 AFURLConnectionOperation 实例底层使用的是同一个子线程，并在该线程中启动了 runloop（`NSURLConnection` 的网络回调必须要有 runloop 的配合，通过port-based input source 唤醒 runloop 处理网络事件），也就是说 AFURLConnectionOperation 是在一条常驻子线程中处理网络回调。

前面我们提到 operation 被 cancel 时也被认为是完成，这点在自定义 `start` 时同样需要注意：
![](/img/AFURLConnectionOperation-cancel.png)
在 AFURLConnectionOperation 的 `cancelConnection` 以及 `connection:didFailWithError:` 方法中都会调用其 `finish` 方法：
![](/img/AFURLConnectionOperation-finish.png)
ps：虽然 NSOperation 支持 cancel，但在调用 `cancel` 方法后该如何处理完全由我们自定义的 `start` 方法决定(当然良好的设计应该要符合 cancel 的语义)。

同时，AFURLConnectionOperation 也实现了以下方法：
![](/img/AFURLConnectionOperation-isReady-isFinished.png)

### 关于 NSOperation 其他细节问题
________________________
+ dependencies:
	我们可以在 operation 间添加依赖关系，在某个 operation 所依赖的 operations 完成之前，其一直处于未就绪状态(`isReady` 为 NO)。
	需要注意的是，依赖关系是 operation 自身的状态，也就是说有依赖关系的 operations 可以处在不同的 NSOperationQueue 中。
	
+ isReady:
	`isReady` 默认实现主要处理 operation 间的依赖关系，当我们自定义该方法时需要考虑 `super` 的值，如 AFURLConnectionOperation中关于 `isReady` 的实现：![](/img/AFURLConnectionOperation-isReady.png)
	
+ qualityOfService:
	用于表示 operation 在获取系统资源时的优先级，默认值：`NSQualityOfServiceBackground`，我们可以根据需要给 operation 赋不同的优化级，如最高优化级：`NSQualityOfServiceUserInteractive`。

+ queuePriority:
	用于设置 operation 在 operation queue 中的相对优化级，同一 queue 中优化级高的 operation(`isReady` 为 YES) 会被优先执行。需要注意区分`qualityOfService`(在系统层面，operation 与其他线程获取资源的优先级)与`queuePriority`(同一 queue 中 operation 间执行的优化级)的区别。
	同时，需要注意`dependencies`(严格控制执行顺序)与`queuePriority`(queue 内部相对优先级)的区别。
	
# NSOperation Queue
___________________________
NSOperation Queue 用于管理、执行 NSOperation，无论其中的 operation 是并行还是串行，queue 都会在子线程(借用 GCD)中执行 operation。
从上小节我们知道，实现异步 operation 比同步 operation 要复杂许多，因此如果打算将 operation 加入 queue 中，则完全可以将 operation 实现为同步方式。
对于 queue 中已就绪的 operation，queue 会选择 `queuePriority` 值最大的 operation 执行。

关于 NSOperation Queue 有两点需要强调：
+ cancelAllOperations：用于取消队列中的 operations，对 queue 中所有 operations 调用 `cancel`方法。(从上小节我们知道，对 operation 调用 `cancel` 方法后的效果完全由 operation 自己决定。`cancel` 唯一能影响的就是清除 operation 的依赖关系，使其立即可以被执行)。此时 queue 并不会 remove 其中的 operations，remove 操作仅发生在 operation 完成时。
+ suspended：将该属性置为 YES，会阻止 queue 执行新的 operation，但已经在执行中的 operation 不受此影响。

# GCD vs. NSOperation Queue
_______________________________
GCD 与 NSOperation Queue 作为常见的并发编程方式，在使用时该如何选择？
首先，对比一下我们关心的几个问题：
![](/img/GCDvsNSOperationQueue.png)
我们可以看到，NSOperation Queue 作为高级 API，有很多 GCD 没有的功能，如需要支持：控制并发数、取消、添加依赖关系等需要使用 NSOperation Queue。
另外，由于 block 可复用性没有 NSOperation 好，对于独立性强、可复用性高的任务建议使用 NSOperation 实现。
当然，NSOperation 在使用时需要 sub-classing，工作量较大，对于简单的任务使用 GCD 即可。

别忘了，我们还有第三种选择：NSThread。由于使用 NSThread 时需要处理线程相关的问题，一般很少使用。但无论是 GCD 还是 NSOperation Queue，其中的任务具体何时执行是由系统控制的，对于实时性要求很高的任务则可以使用 NSThread。

# 小结
_________________________________
本文简单讨论了在使用 NSOperation 时需要重写哪些方法、注意哪些问题。同时也对 GCD 与 NSOperation Queue 作了简单对比，在清楚了它们各自的特点之后再做选择时会更加清晰。

# 参考资料
 [Concurrency Programming Guide](https://developer.apple.com/library/ios/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html)
 [NSOperation Class Reference](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/NSOperation_class/index.html)
 [NSOperationQueue Class Reference](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/NSOperationQueue_class/index.html)

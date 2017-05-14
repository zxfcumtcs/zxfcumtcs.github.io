---
title: Multithreading Traps
date: 2017-05-07 18:02:11
tags:
- iOS
- Multithreading
---
本文首先介绍了多线程的一些基本概念，如：atomicity、Out-of-order execution、Memory barrier等。然后结合 iOS 实际开发，分析了Property、dealloc、target-action、block、mutable containers等在多线程下的问题。最后，分享了几个小技巧。
<!--more-->
©原创文章，转载请注明出处！

# 1. overriew
__________________________________________
多线程使 CPU 的计算能力得到更加充分的利用，尤其在多核时代，程序因此变得更加流畅、高效。在 iOS 开发中，通过 GCD 更是能够零成本实现多线程，动不动就将某些耗时操作通过 GCD 分发到子线程执行，正是由于其廉价性，使得我们通常会忽略其引起的多线程问题。

多线程其实是把双刃剑，在带来流畅、高效的同时，也带来了很多问题，大大增加了程序复杂性。在开发中不难发现，很多bug、crash 都是多线程引发的。
多线程及其相关的问题，大致有以下特点：
+ 难重现(能重现的问题都不是问题)，多线程问题一般与特定的执行时序有关，重现难度大，一般在测试阶段很难发现，而发布之后通过大量用户才会曝露出来；
+ 难理解，很多多线程问题除了看上去很奇怪、百思不得其解，找不到其他毛病，没有足够的经验很难发现是多线程引起的；
+ 难意识，当我们在很嗨地写着多线程代码时，很难察觉正在挖坑！

本文重点不是介绍多线程编程，而是多线程可能引发的问题，在继续之前有必要介绍几个重要概念：原子性(atomicity)、Out-of-order execution 以及 compiler reordering。

# 2. atomicity
____________________________________________
说到多线程，不得不提原子性(atomicity)，我们知道原子(atom)是化学中的概念，表示不可再分的基本粒子。在计算机领域，原子操作(原子性)是指在一次执行中不能被中断的操作。由于中断只发现在指令之间，因此在单处理器(UniProcessor)系统中，能通过一条指令完成的操作都具有原子性，然而在对称多处理器(Symmetrical Multi-Processing, SMP)系统中，由于同时存在多个处理器独立运行，单条指令操作也可能会受到干扰。因此，原子性通过软件是无法实现的，需要硬件层面的支持(架构相关的)。简单讲，CPU 通过缓存锁、总线锁在硬件层面可以实现原子性。

在目前大多数的 CPU 构架(如：x86、PowerPC、ARM)上，读写对齐的一个字长数据，能保证是原子的。如，在64位的系统中，读写对齐的 `int` 型数据的操作是原子操作。

原子性在多线程中是一个非常重要的概念，是实现线程同步(锁)的前提。

# 3. Data Races
____________________________________________
我们平时所说的多线程问题，其实绝大多数时候就是在讲 Data Races，出现 Data Races 有两个条件：
+ 在没有同步的情况下，多线程访问同一块内存；
+ 至少有一个是写操作。

其带来的后果可能是 crash、非预期的运行结果，当然也有可能是『无害』的。
如上文所说，在目前大多数 CPU 上，读写对齐的字长数据是原子操作，即有多个线程同时操作这一内存时不需要任何同步，也有人将其称之为『Benign Race』。但苹果工程师并不认同这种说法，主要有两点理由[WWDC2016_412](http://devstreaming.apple.com/videos/wwdc/2016/412jzguxz4h8hykgjlm/412/412_thread_sanitizer_and_static_analysis.pdf)：
+ 在 C 语言标准中，Benign Race 属于 undefined behavior；
+ 在新的编译器或处理器上可能会引起问题(毕竟这不是一个公认标准)。

因此，对于任何可能会出现 data races 的地方都要做好同步。

# 4. Out-of-order execution、compiler reordering
____________________________________________
Out-of-order execution、compiler reordering，其实两者从代码执行的角度看，本质上是一样的，都是改变了代码原有的执行顺序。
只不过两者的“幕后黑手”以及发生时期不同：
+ Out-of-order execution——由硬件(CPU)实现，也即在指令执行时(运行期)CPU改变了指令原有的顺序；
+ compiler reordering——由编译器实现，在编译过程中(编译期)改变了代码顺序。

为什么会这样？此时大家可能更关心的是这个问题。很简单，两个字：『优化』，使得处理器能够更加高效的运行，具体细节已超出本文范围，不深入讨论。
那么问题又来了，这么“任性”地改变代码顺序不会出问题吗？
当然，它们(CPU、Compiler)是在『深入』分析的基础上，认为『安全』的前提下，才做的优化。

比如有这样一段代码：![](/img/thread1.jpg)


出于优化的目的，`setData`方法最终执行顺序可能就变成了：![](/img/thread1_outofreorder.jpg)

即改变了两条赋值语句的执行顺序。由于两者之间没有依赖关系(正是CPU、Compiler优化的前提)，因此这种优先是无害的。
但是，这是有前提的：单线程程序。

如果此时还存在Thread 2：![](/img/thread2.jpg)

如果两个线程执行顺序如下，那么上面的优化就存在问题了：
![](/img/Out-of-order.jpeg)
最终输出的结果显然不是期望值。

之所以会出问题，在于 CPU、Compiler 没有能力处理多线程问题，它们一直停留在单线程模式中。因此，这个锅只有程序猿来接了！

# 5. Memory barrier
___________________________________________
Memory barrier 正是解决由 Out-of-order execution、compiler reordering 引起问题的方案。
Memory barrier 从字面(内存栅栏)即可理解其用途：强制要求处于 barrier 前的读、写操作的执行先于其后的读、写操作。
![](/img/Memorybarrier.jpg)

如上图，Memory operation1、Memory operation2 在没有 Memory barrier 『保护』的情况下，其执行顺序可能会发生变化，而若在其间添加 Memory barrier，则可以保证他们的执行顺序不会改变。

系统也为我们提供了设置 Memory barrier 的接口：`OSMemoryBarrier`。

因此，上面例子中，我们只需在 thread1 两条写操作间添加 Memory barrier 即可：![](/img/thread1_memorybarrier.png)

> 通过各种锁实现的线程同步，基础都实现了 Memory barrier。

得益于锁内部实现了 Memory barrier，说实话，在实际编程中，我们很少需要考虑 Out-of-order execution、compiler reordering、Memory barrier。但并不代表可以无视它，在遇到一些『匪夷所思』的问题时，或许能为你提供一些思路。其实，任何问题都是这样，研究深度决定在分析、解决问题时的思考深度、视野广度。

介绍完这些多线程的通用问题，我们来看看与 iOS 开发相关的问题。

# 6. Property
__________________________________________
在继续之前，有必要先复习几个概念：变量、指针、指针变量。
+ 变量——本质就是在内存上分配的一块区域，在代码中一般通过一个名称(变量名)操作它；
+ 指针——本质就是一个整型数值，标识一块内存的起始地址；
+ 指针变量——存储指针数值的变量。

通过一个简单的例子看一下：![](/img/ipointer.png)
`pi`是一个指向 `int` 型的指针变量，其指向了变量`i`，如下图，我们假设变量`i`的内存地址为`0x0ffab1234678`，则变量`pi`的值就等于`0x0ffab1234678`。当然，变量`pi`也有自己的内存地址(假设为：`0x0ffab1234660`)，也可以有一个指针来指向它，称之为指针的指针(`int **`)。

![](/img/pointer.jpg)

对象的属性可分为两种类型：
+ 值类型——即非指针类型，如`int`、`bool`、`double`；
+ 引用类型——所有声明为指针类型的属性(OC 中所有对象都是此类型)。

说到Property线程安全问题，必然会想到在定义Property时可以选择`atomic`或`nonatomic`这两个attribute之一，默认为`atomic`。
若在定义 Property 时选择`atomic`，则系统在实现默认 getter、setter 方法时会加锁。那么`atomic`一定能实现我们想要的线程安全吗？
对于值类型的属性，`atomic`确实能保证对该属性的操作是 thread-safety。但对于引用类型的属性，则情况更加复杂，不一定能保证 thread-safety。

我们来看个例子：
![](/img/array_objs.jpg)![](/img/array_objs_addobjects.jpg)![](/img/array-objs-result.jpg)
分别从主线程和子线程往具有`atomic`attribute 的属性`objs`中添加了200000个元素，但最终`objs`中元素的个数小于200000个，明显是由于多线程的 data race 引起的。
可以看到此处的`atomic`并没有解决多线程问题。原因也很简单，对于引用类型的属性，`atomic`『保护』的是属性本身(本质是一个指针变量)，而我们操作的是属性(指针)所指向的那块内存，其并不在`atomic`保护的范围之内。

由于`atomic`并没有想像中那么有用，并且会有一定的性能问题，因此，一般定义属性时并不会使用它，而是在需要同步时，手动实现。
> 在使用`atomic`属性时，还有两个点需要注意：
> 1. 若自定义了 getter、setter 方法，则需要自己实现 atomic 语义；
> 2. 若直接访问属性的存储变量，则失去了 atomic 语义。

# 7. dealloc
__________________________________________
我们知道，object 在哪个线程上被最终释放，其`dealloc`方法就会在哪个线程上执行。这里主要的问题在于有些类的`dealloc`方法在子线程上执行是不安全的，如 `UIKit object`。

当我们开启新的子线程时，子线程一般会 retain 其 target，如：
+ 通过`performSelectorInBackground:`、`performSelector:onThread:`方法开启子线程；
+ 通过`NSThread`开启子线程；
+ 通过 GCD 开启子线程，其block引用了 target。

当子线程 retain 了 target 时，必须要保证该线程对 target 的释放先于主线程。否则，若子线程持有对 target 的最后一个引用，target 的`dealloc`方法必定会在子线程上执行，这对于 UI 来说是不允许的。

但有意思的是，我们发现从 iOS8 开始，即使是在子线程上释放 UI 对象，系统也会将其`dealloc`方法分发到主线程上执行。
![](/img/vcrelease.jpg)
如上图代码，在子线程释放了一个`UIViewController`的object，按理该 object 的`dealloc`方法会在子线程中执行，我们在其`dealloc`方法添加了断点，结果如下：
![](/img/uideallocmainthread.jpg)
可以看到，系统最终还是在主线程上执行了`UIViewController`的`dealloc`方法。
为了进一步验证，在`UIViewController`的`release`方法上添加断点：
![](/img/releasebreakpoint.jpg)![](/img/controllerreleaseresult.png)
可以看到，系统在实现`UIViewController`的`release`方法时，做了一定的处理：
+ 在上图所示的1处，通过`pthread_main_np`方法判断当前是否是在主线程上执行；
+ 2处，进行了判断，若不是在主线程上执行，则直接跳转到`0x117a18a1a`处(跳过了直接执行`dealloc`方法)；
+ 3处(`0x117a18a1a`)，dispatch 到主线程上执行`dealloc`方法。

> 需要注意的是，到目前为止还没找到官方文档明确这件事。因此，我们最好不要做这样的假设，还是要从代码角度确保 UI 的 dealloc 方法永远在 main thread 上执行。

# 8. target-action
___________________________________________
在 Objective-C 中，实现回调(callback)，主要有两种方式：target-action(observer-selector)、block。
这一小节，我们来谈谈 target-action 模式在多线程下存在的问题。
target-action 模式通常情形如下：(有两个对象：`objA`、`objB`)
+ `objA`关心`objB`上某个特定事件EventA；
+ `objB`包含`objA`的弱引用(weak/assign);
+ 当`objB`上 EventA 事件发生时，其回调`objA`的某个方法；
+ 通常 EventA 事件发生在子线程上。

由于，objB 没有强引用 objA，使得 objA 在执行回调时，objA 可能在另外一个线程中被释放，从而出现野指针。
![](/img/target-action.jpg)

细想一下，NSNotification、KVO 都属于 target-action(observer-selector) 模式，它们有两个共同点：
+ 在哪个线程触发，就在哪个线程调用 observer-selector，其引发的最常见问题就是在子线程修改 UI；
+ observer 的 selector 是同步执行的。

进一步深入分析发现：
+ 从 iOS9 开始，NSNotificaionCenter 在抛通知时会强持有 observer，直到 observer 执行完其 action；
![](/img/notification.png)![](/img/notificationdealloc.jpg)
如上代码，在子线程抛出通知，此后我们的代码就再也没有强引用消息的 observer，但从执行的结果可以看出，首先没有 crash(没有出现野指针)，其次 observer 的`dealloc`方法在`notificationHandler`后执行。
![](/img/notificationdeallocstack.jpg)
通过在 observer 的`dealloc`方法添加断点可以看到，是在`postNotificationName:`方法中触发了其`dealloc`方法的执行。
+ KVO 在`NSKeyValueWillChange`方法调用了 observer 的`retain`方法，从而确保在 KVO 回调执行过程中，observer 不会被释放，出现野指针的问题：
![](/img/KVORetain.jpg)

> 与 KVO 不同的是 NSNotificationCenter 并没有直接调用 observer 的`retain`方法，猜测是通过更加低层的方式对 observer 进行了 retain 操作。

NSNotificationCenter、KVO 的这种处理方式给我们很大的启发：
***在执行回调前先对 target 进行 retain 操作，防止出现野指针问题，在回调完成后再 release。***

然而，事情远没这么简单，NSNotificationCenter、KVO 并非线程安全的。

### 8.1 KVO
上面讲到，KVO 在触发时会 retain observer，防止出现野指针，但线程安全问题依然存在。
![](/img/KVOObserverDealloc.jpg)
如上图，observer 在 thread1 上执行 `dealloc` 时(在调用`removeObserver:`前) thread2 触发 KVO，此时会在 thread2 上执行 KVO 回调`observeValueForKeyPath:`，此刻 observer 已成野指针了(虽然 KVO 会调用 observer 的 `retain` 方法，但由于 observer 的`dealloc`方法已开始执行，`retain`也无力回天了！)。

怎么解决？
+ 不要使用 KVO，当时在 QQ 阅读项目开发红包模块时，在灰度中发现这类 crash，最终放弃了 KVO 这一方案；
+ 不要跨线程使用 KVO；
+ apple 建议将 observer 设计成一个永不被释放的对象，facebook 著名 KVO 开源框架 [FBKVOController](https://github.com/facebook/KVOController) 就采用了这一方法，其内部有一个单例`_FBKVOSharedController`，专门用于接收所有的 KVO 回调。


### 8.2 NSNotificationCenter
说到 NSNotificationCenter，第一反应是需要在 observer 的`dealloc`方法中将其从 Notification Center 移除，否则会 crash。但从 iOS9 开始，并不需要手动在`dealloc`中移除，原因是在 Notification Center 中 observer 被存储为`weak`，其最大、最有用的特点就是在其所指 object 释放前会被置为 `nil`。因此，从 iOS9 开始，NSNotificationCenter 是线程安全的，不会出现像 KVO 那样在 `dealloc` 执行过程中触发回调的问题。

> 总结 NSNotificationCenter 在 iOS8、9上的表现，以及 KVO 的表现，可以得出在实现 target-action(delegate) 模式时可借鉴的经验：
> + target 一定要是 weak——防止在 target dealloc 过程中触发回调；
> + 在触发回调前先对 target 进行 retain 操作——防止在回调执行过程中，target 被释放，出现野指针。

# 9. block
_____________________________________________
作为 callback 的实现方式之一，block 由于具有保存 context 的优势，其使用的广泛程度甚至高于 target-action 模式。说到 block，一定会想到其引发的 retain cycle 问题。
为了解决 retain cycle 问题，在 MRC 下一般使用`__block`，ARC 下使用`__weak`。
在 MRC 下使用`__block`非常危险(极容易出现野指针)，必须保证在 block 执行时，`__block`指向的 object 还存在。
![](/img/MRCBlockCrash.jpg)
如上面这个例子，在 dispatch_async 执行过程中，self 被释放，出现野指针。
ps：当然这里仅是个例子，实际中 GCD api 一般是不需要考虑 retain cycle。

ARC 下，由于有 weak，一般不会出现野指针，但 block 在执行时，`self` 可能已是 `nil`，处理不好也有可能会 crash。比较好的做法是，将 block 体封装成方法，这时如果 `self` 为 `nil`，方法就不会被执行。
![](/img/blocktomethod.jpg)

# 10. mutable containers
_____________________________________________
Objective-C 中可变容器是非线程安全的，其导致的多线程问题也是在实际开发中遇到最多的一类线程安全问题。而这其中，`*** was mutated while being enumerated.`最为常见，并且大多数都是由多线程引起的(一个线程遍历，另一个线程写)。
解决这类问题需要注意两点：
+ 一定不能对外提供接口能直接访问类内部的mutable containers；
+ 对 containers 的读(包括整个遍历过程)、写都需要加锁做好同步。

我们都知道 UI 操作需要在主线程执行，除了这点 `UITableView` 似乎与多线程没啥关系，实则不然。
在使用 UITableView 时，我们经常是将其数据(datasource)保存在数组中，而数据来源要么是磁盘、要么是网络，基本都是在子线程完成。
![](/img/tableviewthreadsafe.jpg)
如上图所示，经常犯的一个错误是在子线程获得数据后，直接修改了 UITableView 的 dataSource，而此时可能主线程上 UITableView 的回调也正在读 dataSoure，从而出现 data race 问题。
一定要记住，UITableView datasource 的刷新必须要在主线程上完成(当然，请求数据的过程可以在，也应该在子线程上执行)。

# 11. 技巧
_____________________________________________
最后分享几个小技巧。

### 11.1 Thread Sanitizer (TSan)
针对多线程问题，Apple 在 Xcode8 中推出了扫描工具(目前只支持 64-bit 的模拟器)：Thread Sanitizer (TSan)，其具有以下功能：
+ Use of uninitialized mutexes；
+ Thread leaks (missing pthread_join)；
+ Unsafe calls in signal handlers (ex: malloc)；
+ Unlock from wrong thread；
+ Data races。

其中，对 Data races 的扫描功能非常棒！建议大家在每个版本都用 TSan 扫描一次。更多信息可以参看 [WWDC2016-412 Thread Sanitizer and Static Analysis](https://developer.apple.com/videos/play/wwdc2016/412/)。

### 11.2 放大法定位问题
由于多线程问题，一般复现难度大，只有在特定的执行时序下才能重现，无疑增加了排查、分析、解决问题的难度。
此时，我们可以通过放大法把小概率事件变成大概率事件，如：通过循环反复执行某一操作、通过 sleep 增大不安全的窗口期等。

### 11.3 提供接口让调用方指定callback thread
callback 应该在哪个 thread 上执行，在开发过程中经常有这样的问题。通常的做法是：要么粗暴的 dispatch 到 main thread，要么直接在当前线程执行。
其实，更好的做法是提供接口让调用方指定在哪个 thread callback。
目前，有不少开源库都是这么处理的，如：facebook 著名的 websocket 开源库 [SocketRocket](https://github.com/facebook/SocketRocket)：
![](/img/RocketSocket.jpg)
如上图，其在提供`delegate`接口的同时，提供了两个接口：`delegateDispatchQueue`、`delegateOperationQueue`。

### 11.4 通过 queue specific 判断当前是否在某个队列中执行任务
为了在某队列上同步执行任务，经常需要判断当前是否已在该队列上，否则容易出现 deadlock。通过`dispatch_queue_get_specific`、`dispatch_queue_set_specific`这两个 api 可以方便的实现：
![](/img/queuespecific.png)

# 小结
_____________________________________________
多线程在带来流畅、高效的同时，也带来了无尽的问题。多线程问题本身非常复杂，本文也只是简单分析了几类常见的多线程问题。
总的来说，多线程问题并没有什么统一解决方案，因人因事而异，更多的需要依赖开发人员的经验和严谨的态度。

# 参考资料
[Threading Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html)
[Simple and Reliable Threading with NSOperation](https://developer.apple.com/library/content/technotes/tn2109/_index.html)
[WWDC2016-412 Thread Sanitizer and Static Analysis](https://developer.apple.com/videos/play/wwdc2016/412/)
[Observers and Thread Safety](http://inessential.com/2013/12/20/observers_and_thread_safety)
[Out-of-order execution](https://en.wikipedia.org/wiki/Out-of-order_execution)
[Memory barrier](https://en.wikipedia.org/wiki/Memory_barrier)
[Why is UIViewController deallocated on the main thread?](http://stackoverflow.com/questions/32006565/why-is-uiviewcontroller-deallocated-on-the-main-thread)
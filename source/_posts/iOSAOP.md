title: AOP漫谈
date: 2016-01-23 19:03:00
categories:
- Objective-c
tags:
- IOS
---
本文首先简要讨论了 Objective-C 消息传递机制，随后重点讨论了在 Objective-C 中实现 AOP 的三种方式：*Method Swizzling*、*Aspects*以及*Proxy*。
<!--more-->
©原创文章，转载请注明出处！

# Overview
__________________
AOP(Aspect Oriented Programming, 面向切面编程)作为一种编程范式(Programming Paradigm)，主要用于解决*cross-cutting concerns(横向的通用逻辑)*问题，将非模块内部逻辑抽离出来，实现高内聚、低耦合的目标。

AOP 主要通过预编译或运行期动态代理实现在不修改源代码的前提下添加功能，我们知道 Objective-C 作为典型的动态语言，在实现 AOP 上有着先天优势，下文将着重介绍 Objective-C 现实 AOP 的三种方式。

在项目开发中，像统计、日志、安全鉴权、异常处理等都是不可避免的，同时它们与模块的业务逻辑并不相关，若将它们揉和在一起十分不利于代码的维护，而通过 AOP 可以很方便的将这些功能从业务模块中抽离出来。

在[KVO漫谈](http://zxfcumtcs.github.io/2015/09/18/KVO/)这篇文章中我们讨论了 KVO 的实现机制，简单地讲就是 Objective-C runtime 对被观察对象进行动态的功能添加，使得在被观察属性值发生变化时通知观察者，这也是 AOP 的应用。

# 简述Objective-C消息传递
___________________
Objective-C 语言的动态性在很大程度上取决于其消息传递机制，我们知道在 Objective-C 中调用某个实例的方法(如：`[receiver message]`)，称之为一条消息，最终会转换为调用`objc_msgSend`函数，其原型为：`objc_msgSend(receiver, selector, arg1, arg2, ...)`。该函数在运行时通过查表的方式*动态地*查找最终要执行的方法的入口地址(而 C 函数、C++普通实例方法的调用是在编译期就确定了方法的入口地址，是一个静态过程，当然 C++中的虚函数也是通过查表的方式动态地查找方法入口地址)。

在Objective-C中实现 AOP 主要依赖于其消息传递机制。

在继续之前，有必要先温习几个相关的概念：

+ selector: 我们最熟悉的，可以简单地理解为方法名；
+ NSMethodSignature: 方法签名，主要定义了方法的返回值类型，参数个数及类型；
![](/img/NSMethodSignature.png)
*函数原型 = selector + Method Signature*
+ IMP: 函数指针，其定义为`typedef id(*IMP)(id, SEL, ...);`，定义函数的入口地址;
+ Method: 其定义为`typedef struct objc_method *Method;`;
![](/img/objc_method.png)
从`objc_method`定义可以看出：
*Method = selector + Method Signature + IMP*
+ NSInvocation: 一次实例方法调用的面向对象的封装；
*NSInvocation = object + selector + method signature + arguments*

有了上述铺垫，下面我们进入本文主题，讨论在 Objective-C 中实现 AOP 的三种方式。

# AOP 之 Method Swizzling
_____________________

对于 swizzling 大家应该都不陌生，其利用 Objective-C 的 runtime 动态的替换已有方法的实现，常见的使用场景就是替换系统方法，用于打日志，做统计等。
在一个版本上线后，会收到不少 crash log，但有些 log 无法定位到问题所在，此时如果能知道 crash 发生在哪个页面或许有助于问题的解决，为此，我们 swizzle 了 UIViewController 的 `viewWillAppear:`方法，在 swizzling 方法中记录下该 controller 的名称，同时还 swizzle 了 button 的 action 方法。在出现 crash 时，将这些信息连同 crash log 一起上报，这样我们就能很清楚的看到用户的操作路径。通过该方法解决了不少疑难杂症。

Method Swizzling 的对象是类，因此会影响该类的所有实例，主要用于实现简单的 AOP 场景。
>在一次内部分享时讨论过一个有趣的问题：
>下图所示代码是一个典型的 swizzling 的实现方式，问题是在29行为何要先 addMethod，在失败的情况下再 exchange，而不是直接 exchange？(通过17、23行 if 的判断可以确定该类已经实现了 origSel 和 newSel 两个 selector)
>问题就出在`class_getInstanceMethod`方法的『搜索域』上，该方法首先在第一个参数所指定的类中查找，若没有找到将沿着继承链逆向查找，直到找到指定的方法。也就是说，其返回的方法可能是父类的，若此时直接 exchange，则 swizzle 的是父类中的方法，这不是我们想看到的，故需要先调用`addMethod`方法，而该方法的『执行域』则是类本身，不会进入父类。

![](/img/swizzleMethod.png)

# AOP 之 Aspects
_______________________
[Aspects](https://github.com/steipete/Aspects) 是一个非常优秀的 AOP 开源库，在[github](https://github.com/)已有3000多个 star，可以说是『久经考验』。其在原理上与普通的Method Swizzling是一致的，都是利用 Objective-C runtime 动态的注入新功能。
![](/img/NSObject_Aspects.png)
Aspects的接口非常简单通过为 NSObject 添加 category，提供两个接口分别用于为类、实例 AOP 功能。

通过 Aspects 即可以在类层次为该类的所有实例做切面，也可以针对某个特定的实例：
![](/img/AOPViewController.png)

好奇的是Aspects如何针对实例做切面，为了更方便的阐述问题，以下面的代码为例：
![](/img/AspectsDome.png)
+ Aspects 会动态创建`AspectsDome`类的子类`_Aspects_AspectsDome`；
+ swizzle `_Aspects_AspectsDome`类的`forwardInvocation:`方法，使其指向`__ASPECTS_ARE_BEING_CALLED__`函数(在该函数中会执行指定的切面操作并执行被 hook 的方法)；
+ swizzle `_Aspects_AspectsDome`类的`class`方法，使其指向父类的`class`方法(为了对外界隐藏其实现细节，系统在实现 KVO 时使用了同样的方式)；
+ 在`_Aspects_AspectsDome`类中添加新方法`aspects__testAspects`，其实现指向父类`AspectsDome`的`testAspects`方法；
+ 在`_Aspects_AspectsDome`类中将`testAspects`方法的实现指向`_objc_msgForward`;
+ 将实例`aspectsDome`的类属性(isa)切换到`_Aspects_AspectsDome`类上(即，此时`aspectsDome`的类型为在`_Aspects_AspectsDome`，而不是初始化时的`AspectsDome`)。

看到这里，似乎有种熟悉的味道，没错！在[KVO漫谈](http://zxfcumtcs.github.io/2015/09/18/KVO/)这篇文章中我们讨论了 KVO 的实现机制，和 Aspects 的做法十分类似，都会创建新的子类。

此时，执行`[aspectsDome testAspects]；`的流程如下：
[aspectsDome testAspects]->_objc_msgForward->[aspectsDome forwardInvocation]->__ASPECTS_ARE_BEING_CALLED__->执行切面操作并执行`testAspects`方法的初始实现。

详细的现实建议大家去看[源码](https://github.com/steipete/Aspects)。
在阅读源码过程中，最让我震撼的是下面两段代码：
![](/img/AspectBlock.png)
![](/img/aspect_blockMethodSignature.png)

在[block那些事](http://zxfcumtcs.github.io/2014/07/14/block/)系列文章中我们详细讨论过 block 的内部结构。Aspects的作者根据 block 内部结构，从 block 中抽取出一个 NSMethodSignature，这样做最大的目的是获取 block 参数的个数及类型。

然而下面这段代码却给我很大的困扰，无法理解：
![](/img/invokeWithInfo.png)
这段代码的作用是根据由 block 得到的 method signature，实例化一个 NSInvocation，设置相应的参数，最后在 block 上执行该 invocation。

在 NSInvocation 中第0个、第1个参数分别是 self，_cmd。其中第0个参数通过`invokeWithTarget:`方法可以设置，但_cmd，也就是真正要执行的 selector 却没有设置。

在844行，当 block 的参数个数大于1时，将第1个参数设为 info。

*简单地说，问题就是没有设置 NSInvocation的 selector，最终是如何找到相应的 selector 并执行的？
我想到的唯一的理由是，此处的 target 是 block 类型，在 NSInvocation 内部作了特殊处理，仅仅是猜测，没找到相关的资料。*

Aspects 通过 Objective-C 的动态性，在运行时 hook 原有方法，并注入新功能，实现了 AOP，其通常用于打日志、做统计等。

# AOP 之 Proxy
___________________________
通过前文的阐述我们知道，在 Objective-C 中方法调用最终都要通过查表的方式找到方法入口地址，那如果没有找到要执行的方法如何？
*unrecognized selector sent to instance...*，这是我们经常看到的在没有找到要执行的方法时系统报的 crash。

其实，在 crash 之前，runtime 给了我们3次拯救的机会：
+ Method resolution；
+ Fast forwarding；
+ Normal forwarding。

前两种方法与文本讨论无关，不多赘述。

Normal forwarding：这也是 runtime 给的最后一次机会，此时，rumtime 首先发送`methodSignatureForSelector:`消息，用于获取方法签名，若在该重载方法中返回 `nil`，则 rumtime 抛出`doesNotRecognizeSelector`消息，程序就 crash 了。否则，runtime 发送`-forwardInvocation:`消息，实现转发。

这是一个很好的实现 AOP 的契机。
但，很快会发现，若直接使用Normal forwarding实现 AOP，还是无法降低模块间的耦合，将不相关的逻辑从业务逻辑分离的目的。

此时，NSProxy 就要『粉墨登场』了。

NSProxy 或许有点陌生，但从名称能大概知道其主要作用是代理(简单说就是消息转发)。
![](/img/NSProxy.png)
从上图可以看出，NSProxy 的实现非常简单(代码已稍加整理)，也是为数不多的不是继承自 NSObject 的类。之所以要如此设计，原因在于一旦类实现了某个方法，那么在调用该方法时就不会触发消息转发机制，而是直接调用已实现的方法。NSProxy 作为代理类，这样的干扰自然是越少越好。

在开发过程中，经常有这样的需求，执行某些操作前必须要鉴权(登录)，下面以此为例简单的演示一下：
![](/img/TTAuthenticationProxyh.png)
![](/img/TTAuthenticationProxym.png)
首先，定义了一个鉴权 proxy，在该类的`forwardInvocation:`方法中判断当前是否需要鉴权，若不需要鉴权则直接在 target 上执行操作，否则通过鉴权中心`TTAuthenticationCenter`进行鉴权操作，同时保存当前请求。
![](/img/TTAuthenticationCenter.png)
在`TTAuthenticationCenter`中，通过 maptable 记录当前需要鉴权的请求(注意：maptable 的 key 使用了`NSPointerFunctionsWeakMemory`关键字，这样就不会retain target，当 tagret 被 dealloc 时，会自动从 maptable 中清除)
在鉴权完成后，即在`didAuthentication`方法中，依次执行被 hook 的请求。
![](/img/NSObjectAuthentication.png)
![](/img/testOperate.png)
上图在`testOperate`方法中要调用`operateNeedAuthentication`方法，但该方法需要先鉴权，故将该调用直接发送给`TTAuthenticationProxy`。
通过这种方式，完全将鉴权逻辑与业务逻辑隔离，降低模块间的耦合。

# 小结
_______________________
在很多场景下，通过 AOP 可以隔离不相关的逻辑、降低模块间的耦合。通过上文可知，在 Objective-C 中实现 AOP 主要是借助于 runtime，动态的添加新功能，因此有一定的性能损耗，在使用之前需要了解具体的业务场景。
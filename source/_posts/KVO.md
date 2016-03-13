title: KVO漫谈
date: 2015-09-18 23:02:28
categories:
- Objective-c
tags:
- IOS
---
本文讨论了 KVO 若干有趣的特性，并将 KVO 与其他通信方式进行了比较，最后深入分析了 KVO 内部实现机制。
<!--more-->
©原创文章，转载请注明出处！

#Overview
__________________

KVO(Key-Value Observing)对于 iOS 开发者来说应该并不陌生。通过 KVO 机制，当被观察对象的属性值发现变化时观察者能接到相应的通知。为了能接收到正确的 KVO 消息，需要满足以下三点：
- 被观察对象的相应属性必须是 KVO 兼容；
- 通过`addObserver:forKeyPath:options:context:`方法注册观察者；
- 观察者必须实现`observeValueForKeyPath:ofObject:change:context:`方法。

那么何为 KVO 兼容的属性？
同样需要满足三点：
- 相应属性需要是 KVC 兼容；
- 相应属性值改变时需要触发 KVO 消息；
- 相关的dependent keys设置正确。

其中，对于 KVC 兼容的属性，NSObject 已支持在属性值变化时自动发送 KVO 消息。当然也可以禁止 NSObject 的该功能，改为由手动实现，此时，只需重载`automaticallyNotifiesObserversForKey:`方法，对于需要禁止 KVO 的 key 返回 NO 即可。

关于 KVO 有几个点值得一提：
- Dependent Keys：
 在 KVO 所有特性中，最让我心动的莫过于依赖键(Dependent Keys)，简单地说就是某个属性值可以依赖于其他若干个属性，当任何被依赖的属性值改变时，依赖属性就会触发 KVO 消息。对于To-one Relationships，可以通过`keyPathsForValuesAffectingValueForKey:`或`keyPathsForValuesAffecting<Key>`方法注册依赖键。

- NSKeyValueObservingOptionInitial：
 我们知道 KVO 的触发时机是被观察的属性值发生变化时，但有时可能需要在初始化时就能通过 KVO 获取被观察属性的值，此时就可以在注册观察者时添加`NSKeyValueObservingOptionInitial`选项。通过该选项，在调用`addObserver:forKeyPath:options:context:`方法时会同步触发 KVO 消息。

- NSKeyValueObservingOptionPrior：
 正常情况下，是在被观察属性的值改变后通过`didChangeValueForKey:`方法触发 KVO 消息。若在注册观察者时添加`NSKeyValueObservingOptionPrior`选项，则在属性值改变前，即在`willChangeValueForKey:`方法中也会触发一次 KVO 消息。此时，被观察属性值的改变会触发两次 KVO 消息，可以通过`change`中的`NSKeyValueChangeNotificationIsPriorKey`进行区分。

- KVO 消息的同步性：
 首先，KVO 消息的处理与被观察属性值的改变发生在同一线程上，因此`observeValueForKeyPath:ofObject:change:context:`方法可能会在不同的线程上执行，需要考虑线程安全问题。其次KVO 消息与被观察属性值的改变是同步的，也就是在被观察属性值的`setter`方法返回前所有的 KVO 消息都已处理完成。

由于本文的定位不是 KVO 教程，关于[KVO](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html)、[KVC](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/KeyValueCoding.html)，Apple 官方有详细的描述，在此不再赘述。

#KVO、Notification、delegate、block
____________________
KVO、NSNotfication、delegate、block这四种方式都可以用于对象间的通信。
KVO 与 NSNotification 有很多相似之处，都是一对多的、单向的通信方式，通过它们可以使得模块间较好的解耦。同时它们也有许多相似的缺点。
delegate、block 则是一对一的、『半双向』的通信方式。之所以说是『半双向』是因为它们可以有返回值。
![](/img/MVC.jpg)
这是斯坦福大学 iOS 开发公开课里关于讲解 MVC 的一页 PPT，很好地总结了 M、V、C 间通信的规则与方式：
- Model 与 View 间不允许直接通信；
- Controller 作为控制中枢，一般持有 Model 与 View 的 object，能直接与它们通信；
- View 可以通过 delegate 与 Controller 通信以及通过将 Controller 设置为 target 响应来自 View 的事件；
- Model 可以通过 KVO 或 Notification向 Controller 发送消息。

可以看到，为了充分解耦，Model 与 Controller 间的通信有严格的限制，必须是单向的。
由于 View 与 Controller 严格要求是一对一的，两者的关系也更加紧密，故其间的通信 delegate 更加适合。

个人认为，从设计的角度出发，KVO 与 Notification 没有本质的区别。但前面提到 KVO 有个很棒的特性：Dependent Keys。有时，通过 KVO 的Dependent Keys特性能写出很漂亮的代码。

比如：界面中某个元素 V 的显示依赖于 Model 中 A、B、C、D 四个属性值，当其中任意属性值发生改变时，界面元素 V 的状态都需要变化。
如果，此时选用 Notification 的方式由 Modle 通知 Controller 刷新 UI，需要在A、B、C、D 四个属性值发生变化的地方都抛出 Notification，同时需要将4个属性值通过消息传递给 Controller 以便计算 V 的新状态。
若采用 KVO 机制，可以在 Model 中合成一个新的属性M，并且使得 M 依赖于 A、B、C、D，再在 Controller 中观察属性 M 即可。

#KVO陷阱
____________________
KVO 本身存在不少争议，也有人为其开出了长长的『罪行』清单。
个人认为 KVO 值得我们关注的缺点有：
- string 类型的 key：
 在 KVO 中所有 key path 都是 string 类型，也就意味着无法通过编译器在编译期发现像拼写错误这类的 warning、error；
 
- 无法指定响应 KVO 的selector：
 一个类只能集中在`observeValueForKeyPath:ofObject:change:context:`方法中处理 KVO 消息，而无法像 NSNotification 那样在注册观察者时指定 selector，这就导致如果某个类观察的属性较多时，在该方法中就会出现长长的`if...else...`；
 
- KVO 消息是隐式的(implicit)：
 一旦 KVO 相关的观察者注册完毕，其余的事情都由 runtime 完成，虽然符合数据单向流通的设计要求，但在实际开发过程中还是有些困扰，尤其是新人接手老项目时，可能无法意识到修改某个属性的值会产生一系列的影响，不便于问题的调试、跟踪。

当然，使用 KVO 还有一些要注意的地方，处理不好可能会 crash、产生意料之外的结果等，这些都属于编码规范一类的，在此不多作讨论。

#KVO背后的故事
____________________
我们知道被观察属性值发生变化时，观察者能自动收到 KVO 消息，这是如何实现的呢？
这一切都要归功于 Objective-C 语言的动态 runtime，[mikeash](https://www.mikeash.com/pyblog/friday-qa-2009-01-23.html)大神对此有过具体分析，在此我们再来梳理一下。
![](/img/printClassInfo.png)
![](/img/KVO1.png)
![](/img/KVOResult1.png)
从上述结果我们能得出：
+ 在某个对象(`_observedObject`)上添加观察者时，runtime 会为其动态生成一个子类(`NSKVONotifying_KVOObserved`)，并将被观察对象(`_observedObject`)的`isa`指针切换到新生成的子类上；
+ 在被观察对象(`_observedObject`)上移除观察者时，其`isa`指针又会切换回来；
+ Apple 君为了隐藏 KVO 的实现细节，在动态生成的子类中重写了`class`方法，使其返回原有类的信息(当然通过`object_getClass`获取的`isa`指针信息是无法隐藏的)。

我们再来对比一下这两个类都包含了哪些方法(注：printNamesStr只会输出该类自身的方法，不会包含父类的方法)：
![](/img/printNamesStr.png)
![](/img/printNamesStrResult.png)

可以看到在`NSKVONotifying_KVOObserved`类中重载了`setX:`、`class`、`dealloc`方法，添加了`_isKVOA`方法。
更进一步，我们发现被`NSKVONotifying_KVOObserved`重载的`setX:`实际指向全局的：`_NSSetLongLongValueAndNotify`方法。
![](/img/printIMP.png)

针对不同的类型，Apple 实现了以下方法：
_NSSetBoolValueAndNotify、_NSSetCharValueAndNotify、_NSSetDoubleValueAndNotify、_NSSetFloatValueAndNotify、_NSSetIntValueAndNotify、_NSSetLongLongValueAndNotify、_NSSetLongValueAndNotify、_NSSetObjectValueAndNotify、_NSSetPointValueAndNotify、_NSSetRangeValueAndNotify、_NSSetRectValueAndNotify、_NSSetShortValueAndNotify、_NSSetSizeValueAndNotify、_NSSetUnsignedCharValueAndNotify、_NSSetUnsignedIntValueAndNotify、_NSSetUnsignedLongLongValueAndNotify、_NSSetUnsignedLongValueAndNotify、_NSSetUnsignedShortValueAndNotify

这些方法虽没有开源，但其内部实现通过调用栈能所有了解：
![](/img/NSSetLongLongValueAndNotify.png)
在该方法中我们看到了熟悉的`willChangeValueForKey:`以及`didChangeValueForKey:`方法的调用。

#小结
____________________
KVO 通过 runtime 动态机制实现了自动发送 KVO 消息，但正如前文所说 KVO 也存在不少争议，在使用前需要仔细思考，其是否是最佳选项。

参考资料：
[Friday Q&A 2009-01-23](https://www.mikeash.com/pyblog/friday-qa-2009-01-23.html)
[Key-Value Observing Done Right](https://www.mikeash.com/pyblog/key-value-observing-done-right.html)
[Key-Value Observing](http://nshipster.com/key-value-observing/)
[Key-Value Observing Programming Guide](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html)
[Key-Value Coding Programming Guide](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/KeyValueCoding.html)
[Key-Value Coding and Observing](https://www.objc.io/issues/7-foundation/key-value-coding-and-observing/)
[KVO Considered Harmful](http://khanlou.com/2013/12/kvo-considered-harmful/)
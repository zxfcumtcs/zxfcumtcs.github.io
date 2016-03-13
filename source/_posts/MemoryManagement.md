title: Inside Memory Management for iOS
date: 2015-09-03 16:28:55
categories:
- Objective-c
tags:
- IOS
----

本文探讨了在 Memory Management 中的命名规则、引用计数的实现机制以及 weak 变量的内部实现。
<!--more-->
©原创文章，转载请注明出处！

#Overview
_________________

Memory Management 在 C 语言体系中一直是个重要的话题，它们没有像 Java 那样的 Garbage Collection，内存管理完全由程序员负责。因此，稍有不慎就会出现 Memory Leak 或 Dangling Pointer 等严重问题，这对于程序来说是致命的！

Objective-C 作为在 C 语言基础上发展起来的面向对象语言，自身自然也没有内存管理机制。因此，作为 iOS 程序员的我们也需要小心翼翼地处理着内存问题。然而，这一切随着 ARC 的到来有很大的改观。

由 iOS5 和 Xcode4.2 内置的编译器 LLVM3.0 共同支持的 ARC(Automatic Reference Counting)，如其名称所示实现了内存的自动管理。简单地说，其实质就是将内存管理的工作由程序员转交给编译器来完成，当然某些特性需要 runtime 的支持。

#内存管理中的命名规则
_________________

与 ARC 相比，我们将手动内存管理机制称作 MRC(Mannul Reference Counting)，就像 ARC 与 MRC 名称所展示的那样，两者从内存管理的本质上讲没有区别，都是通过引用计数(Reference Counting)机制管理内存。不同的是，在 ARC 中内存管理相关的代码由编译器在编译代码时自动插入。

那么问题来了~

ARC 下编译器如何自动插入内存管理代码？更直白点，编译器如何知道在某处需要插入相关的代码？

首先，能想到的是在类的 `dealloc` 方法中，对该类的实例对象所持有的成员变量(strong)执行 release 操作。

那么，对于局部变量，编译器如何管理内存？
```
{
    NSString *str = [[NSString alloc] initWithFormat:@"test ARC"];
    NSLog(@"%@", str);
}
```

我们知道，在 ARC 下，上面的代码片段会被编译器处理成(仅是示例，编译器最终的处理可能不完全一致)：

```
{
    NSString * str = [[NSString alloc] initWithFormat:@"test ARC"];
    NSLog(@"%@", str);
    [str release];			// 编译器插入了 release
}
```

而下面的代码片段，编译器没有为内存管理添加任何代码：

```
{
    NSString *str = [NSString stringWithFormat:@"test ARC"];
    NSLog(@"%@", str);
}
```

为什么编译器在处理上述两段代码时，采取不同的态度？

答案很明显，代码2返回的是 autorelese 对象，其已被纳入内存管理之中，故不需要编译器再作处理，而代码1却没有。

嗯，问题似乎已得到完美的解答！
然而，这一切都是站在人的角度去分析的，编译器如何知道代码1需要其管理内存，而代码2不需要？

ok，这就是本节主题：『命名规则』要解决的问题。

任何以下列名称为前缀的方法，若其返回值为 object，则方法调用者持有该 object：
- alloc
- new
- copy
- mutableCopy

还有一个更为严格的规则：任何以 init 为前缀的方法必须遵守下列规则：
- 该方法必须是实例方法；
- 该方法必须返回类型为`id`或其所属class、superclass、subclass 的对象；
- 该方法返回的 object 不能是 autorelese，即方法调用者持有返回的 object。

『方法调用者持有该 object』也就意味着该 object 的内存问题需要调用方管理。

在此之外的任何方法返回的 object，其调用方都不持有，即返回的应该是 autorelease object。

>例外：以 allocate、newer、copying、mutableCopyed 为前缀的方法以及 initialize 方法不在上述规则之内。

编译器根据上述规则，很容易就能判断出在代码1中需要处理对象`str`的内存问题，而代码2返回的是 autorelease object，故不需要其处理。

在编码过程中，应该严格按照上述规则执行！

那问题来了，如果在编码过程中硬是不遵守上述规则如何？
首先，你就被排除在优秀程序员之外了！MRC 时代，不按上述规则写出的代码在内存管理上极难维护，很容易出现内存泄漏或多次释放的问题。
![](/img/MRCAlloc.png)
在 MRC 下，上述代码通过 Analyze 做静态分析时会给出如上图所示的 warning。

####ARC 与 MRC 混用
_______________________
由于很多项目经历了 MRC 到 ARC 的时代，因此 ARC 与 MRC 在项目中同时存在的情况大有所在。
如果，此时不遵守上述命名规则会出现问题吗？

![](/img/MRCAllocString.png)
![](/img/ARCuseMRC.png)
运行结果：
![](/img/ARCUseMRCCrash.png)
crash!

原因在于在 ARC 的文件中调用了以 `alloc` 为前缀的方法，根据命名规则，此时编译器认为 `allocString` 方法将返回一个需要其管理内存(release)的对象。上述 ARC 的代码将被编译器处理为：
![](/img/ARCPseudoconde.png)
然而，在 MRC 中的`allocString`方法并未遵守命名规则，其返回的是一个 autorelease object。最终效果就是 release 了一个 autorelease object！这类问题，我们除了知道出问题对象的类型，crash 堆栈没有任何帮助，因此在大型项目中排查此类问题有一定的难度(工作量)。
![](/img/CrashStack.png)
那么，在 ARC 的文件中不遵守命名规则会出问题吗？
![](/img/ARCAllocString.png)
这段代码运行正常没有 crash，原因在于 `allocString` 方法本身及调用者都在 ARC 管理范畴之类，编译器很清楚该如何处理。即便如此，作为优秀的程序员，在日常 coding 过程中还是应该遵守命名规则。

#Inside Reference Counting
____________________
前文已提到，无论是 ARC 还是 MRC，其本质都是通过引用计数(Reference Counting)来管理内存。

如果让你设计一套引用计数机制，你会怎么做？
嗯，这是个不错的面试题！
其实，该问题的答案不外乎两种：
- 在对象内部管理引用计数；
- 通过外部结构(如：hash 表)统一管理引用计数。

####GNUstep’s Implementation of Reference Counting
________________________
GUNstep 实现了一套兼容 Cocoa Framework 的 Framework，作为开源代码我们来看看它是如何处理引用计数的：
![](/img/GNUstepReferenceCounting.png)
通过整理，删除非必要的代码，GUNstep 实现的`alloc`方法如上所示。可以看到，其使用了一个结构体`obj_layout`来保存引用计数，同时该结构体被附在所生成 object 的头部。
object内存布局如下图所示(引自《Objective-C高级编程》)：
![](/img/GNUstepObjectMemory.png)

####Apple’s Implementation of Reference Counting
_________________________
由于 Apple 现已来源了相关的代码，使得我们可以进一步一探究竟。Apple 所有的来源代码都可以在此找到：[Apple Opensource](http://www.opensource.apple.com/source/objc4/)。

首先，我们来看看 Apple 是如何实现 `retain` 方法的：
![](/img/AppleRetain.png)
如果抛开 Apple 所做的优化，其`retain`方法最终会调用上图所示的`sidetable_retain`方法。

在继续之前有必要介绍一下新朋友：`SideTable`
![](/img/SideTable.png)
在类`SideTable`的成员变量中，似乎看到了熟悉的味道！是的，没错，其中的 `RefcountMap` 就是引用计数表，而`weak_table_t`则是弱引用表(weak table).

RefcountMap 则是一个简单的 map，其 key 为 object 内存地址，value 为引用计数值。

通过`SideTable`源码，还可以得出如下结论：
- 存在全局的若干个`SideTable`实例，它们保存在 static 成员变量`table_buf`中；[在 iOS 平台上有8个这样的实例(SIDE_TABLE_STRIPE = 8)]
- 程序运行过程中生成的所有对象都会通过其内存地址映射到`table_buf`中相应的`SideTable`实例上。

这里之所以会存在多个`SideTable`实例，object 映射到不同`SideTable`实例上，猜测是出于性能优化的目的，避免`SideTable`中的 reference table、weak table 过大。

回到上面的`sidetable_retain`方法，其首先通过 object 的地址找到对应的 sidetale，然后通过 `RefcountMap`将该 object 的引用计数加1.

`release`、`retainCount`等相关方法的代码在该开源代码中也能找到，在此不再赘述。

简单地说，Apple 通过全局的 map 来记录Reference Counting，其key 为 object 地址，value 为引用计数值。

那么，GNUstep 与 Apple 的实现方案各有什么优劣点？
GNUstep 的方案从实现的角度看简单明了，而 Apple 的方案能够更好的把控系统内存使用情况，对调试有一定的帮助。

#Inside Weak
_________________________

weak 无疑是 ARC 送给我们的一大利器，通过它基本能消灭 delegate 引起的 dangling pointer 问题。这得益于 weak 指针指向的 object 在 dealloc 时，该指针会被置为 nil。
那么，系统是如何处理 weak 变量的呢？
![](/img/weak.png)
上面这个简单的代码片段会被编译器处理成如下所示的 pseudo code：
![](/img/weakpseudocode.png)
注：以下代码已经过整理以便阅读、抓住重点。

通过上述代码可以看到，对于 weak 变量，系统将以 weak 指针变量的地址(&weakNum)、用于赋值的 object(num)为参数调用`objc_initWeak`方法：
![](/img/objc_initWeak.png)
`objc_initWeak`方法进一步调用`objc_storeWeak`方法，在该方法中，以赋值object(num)对应的 sidetable 中的 weaktable、赋值object(num)以及 weak 变量的指针为参数调用`weak_register_no_lock`方法。
在`objc_storeWeak`方法中，还可以看到，对于 weak 引用会在赋值object的引用计数表中设置弱引用标志位(SIDE_TABLE_WEAKLY_REFERENCED)，具体原因有待深究。
当然，在`objc_storeWeak`方法中做的最后一件事情就是将赋值对象的地址赋给 weak 指针。
![](/img/objc_storeWeak.png)
`weak_register_no_lock`方法首先检查赋值object在 weak table 中是否存在相应的条目，若存在则直接在其中添加该 weak 变量的信息，若不存在则插入赋值 object 对应的条目。
![](/img/weak_register_no_lock.png)

####weaktable
__________________________
谈到 weak，自然少不了要说到 weaktable：
![](/img/weaktable.png)
![](/img/weak_entry_t.png)

在 weaktable 中，最重要的成员莫过于`weak_entry_t`类型的数组:weak_entries。在`weak_entry_t`结构体中，包含赋值 object 的指针以及所有指向该赋值 object 的 weak 变量列表(weak_referrer_t *referrers)。

那么，在 weaktable 中如何通过object 找到其对应的 entry？
![](/img/weak_entry_for_referent.png)


最后一个问题，在 weak 指针所指向的 object 被 dealloc 时，weak 指针会被置为 nil。那么问题来了，如果 weak 变量的生命周期在其指向的 object 之前就结束了，会如何？

若在 weak 变量生命周期结束后，其所使用的内存块被重新利用赋上了新值，而此时上述 object 被 dealloc，若根据 weak table 中的条目将其对应的 weak 变量一一置为 nil，则上述被重新利用的变量也将被清0，这显然是不合适的。

在上面提到的 pseudo code 中，我们看到在 weak 变量 `weakNum` 生命周期结束时，调用了`objc_destroyWeak`方法，没错，该方法就是用于解决上述问题的。
`objc_destroyWeak`方法最终会调用`weak_unregister_no_lock`方法，其会将 weak 变量从 weak table 中移除掉。
![](/img/weak_unregister_no_lock.png)

#Toll-Free Bridge cast
____________________________
####__bridge cast
在 ARC 下，id 与 `void*`之间不再像 MRC 时代可以任意转换，在 ARC 下若需在两者之间转换可以使用`__bridge cast`。
在使用时需注意，在将 id 转换为 `void*`时，其不再在 ARC 的内存管理范畴内，极有可能出现dangling pointer。

####__bridge_retained cast
__bridge_retained的作用是使得被赋值变量持有赋值 object。
![](/img/bridge_retainedARC.png)
上述 ARC 代码与下面的 MRC 代码在内存管理上是等价的
![](/img/bridge_retainedMRC.png)

####__bridge_transfer cast
__bridge_transfer的作用是使得赋值 object 在赋值后被 release。
![](/img/bridge_transferARC.png)
![](/img/bridge_transferMRC.png)
上面两段代码是等价的，正如__bridge_transfer名称所示，其作用是将持有权从赋值 object 转到被赋值变量。

在 ARC 下，需要转换的情况发生在Objective-C object 与 Core Foundation object 间，此时可以使用以下方法：
![](/img/CFBridgingRetainRelease.png)

#小结
_____________________________
关于内存管理有很多的问题值得探讨，随着 Apple 相关代码的开源，一切的私密都能在源码中找到答案。

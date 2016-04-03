title: block那些事——block误用与终极总结(5/5)
date: 2014-07-17 21:46:35
categories:
- Objective-c
tags:
- OC
- IOS
---

本文主要介绍错误使用block的两种情况以及对该系列的总结。
<!--more-->
©原创文章，转载请注明出处！

# block误用
______________
### block作为函数参数

[前文](http://zxfcumtcs.github.io/2014/07/15/block2/)介绍过除了两种特殊情况(ARC下三种特殊情况)，block作为函数参数时不会被copy到heap，需要手动copy block。

下面这个例子在非ARC下会crash：

![](/img/errorblock4.jpg)

正确用法：

![](/img/blockright.jpg)

误用1：将`NSConcreteStackBlock`类型的block作为函数参数时，没有copy该参数——在需要时没有将stack上block copy到heap上；

### 错误的使用__block解决retain cycle

在使用block时，大家都紧绷着一根弦：**retain cycle**。

是的，如果处理不当，会引起内存漏泄！

我们知道，在非ARC下，解决retain cycle问题的办法之一就是用__block打破这个cycle。

但在使用过程中，有很多人过度使用__block，导致出现crash问题。

其中，一个典型的错误用法就是：

```
__block ClassType *weakSelf = self;
dispatch_asyn(_queue, ^{
	[weakSelf doSomething];
});
```

咋看起来，好像没问题，还妥妥地解决了retain cycle问题。

but...，非ARC下，block不会retain其引用的__block变量！！！因此在异步执行这个block时，self可能已经dealloc了，出现野指针！crash！！！

![](/img/AppCrashed.jpg)

看个例子：

![](/img/errorblock1.jpg)
![](/img/errorblock2.jpg)

这里之所以会crash就是因为在block执行前，self已经野掉了。

那么这个retain cycle怎么解决呢？

等等...，这里真的会引起retain cycle吗？

以GCD API为例，我们知道GCD API会copy其参数block，此时block会retain其引用的外部object，但当该block执行完时，系统会release该block，进而也会解除对其引用object的retain。

**因此，对于GCD API来说，绝大多数情况下不需要考虑retain cycle问题。**并且处理不好还会引起crash！

![](/img/errorblock3.jpg)

上面这个例子中，我们直接在block里面使用`self`，但没有引起retain cycle，其`dealloc`方法执行了！

误用2：非ARC下，在异步执行的block中错误的使用__block变量。

# 总结
______________

+ **`block`被会编译器转换为`struct`；**

+ **`_NSConcreteStackBlock`类型的`block`在其作用域结束时会被释放，如果后续还要用到该`block`，需要将其`copy`到heap上；**

+ **`__block`类型的变量会被编译器转换为`struct`，其还会随`block` `copy`到heap上而`copy`到heap上；**

+ **`__forwarding`指针主要用于保证stack上的`__block`变量与heap上的`__block`变量最终访问的是同一内存(heap上)；**

+ **使用GCD API时，绝大多数情况不需要考虑reatin cycle问题，注意因异步访问`__block`变量引起的crash。**


OK，最后检验一下学习成果，做几道题吧^_^

[测试题](http://blog.parse.com/2013/02/05/objective-c-blocks-quiz/) ![](/img/Tsan_angry.jpg)

title: 'Objective-C Object Model(3/4)'
date: 2014-07-11 16:31:25
tags:
- Objective-C
- iOS
---
本文主要讨论Objective-C对象内存布局并与C++内存布局进行比较，最后讨论了Apple针对64位处理器引入的`Tagged Poniters`。
<!--more-->
©原创文章，转载请注明出处！

# Objective-C对象内存布局
______________
由[前文](http://zxfcumtcs.github.io/2014/07/10/Objective-C-Object-Model2/)我们知道Objective-C的类、对象本质上都是C语言的`struct`，且第一个成员变量是指向元类的`isa`指针。

那么Objective-C对象在内存中的布局是怎样的？

![](/img/oc_objective_c_memory_layout.jpg)

可以看到对象的成员变量从根类开始依次排列(`isa`指针就是根类`NSObject`的成员变量)。

OK，我们可以验证一下：
```mm
@interface A : NSObject
{
    NSInteger   _a;
}

@end

@interface B : A
{
    NSInteger   _b;
}

@end

@interface C : B
{
    NSInteger   _c;
}

@end
```

结果：

![](/img/oc_objective_c_memory_layout_test.jpg)

呵呵，结果很清楚了！

**我们来看看C++对象的内存布局：**

![](/img/c_plus_plus_memory_layout.jpg)

**对比这两个内存布局图，对于包含虚函数的C++对象来说，其在内存上的布局与Objective-C对象有几分神似，前者含有指向虚函数表的`vfptr`指针，后者含有指向元类的`isa`指针。**

# Tagged Pointer
______________
我们知道指针在本质上就是一个整数，在32位系统上它就是一个32位的整数，那么在64位的系统上它自然就是一个64位的整数了。

为了跨平台和处理器的访问效率，一般要求内存中的数据以特定的字节对齐。以32位平台为例，若一个整数存储在以`0x00000003`起始的4字节内，则读取这个整数需要2个CPU周期，而如果该整数存储在以`0x00000004`起始的4字节内则只需1个CPU周期。

在IOS中，数据都是16字节对齐的，也就是指针的后4位都是0。

![](/img/pointeralign.png)

Apple从iPhone5s开始采用64-bit A7双核处理器，因此指针也扩展到64位，其寻址能力：2^64，远远超过目前移动设备的实际需要，最终导致指针的前半部分出现大量的0，造成内存空间的浪费。

此时，`Tagged Pointer`应运而生。

那么什么是`Tagged Pointer`呢？

简单地讲，如果一个指针存储的是对象本身，而不是对象的地址，那么它就是`Tagged Pointer`。

对比一下普通指针和Tagged指针(来自WWDC2013 session404)：

![](/img/normalandtaggedpointer.png)

![](/img/taggedpointer.png)

从Apple的介绍我们可以知道：

+ `Tagged Poniter`主要用于处理小型数据，如：`NSNumber`、`NSData`

+ `Tagged Pointer`能节省内存提高效率(省去多余的`allocate/destroy`)


由于`Tagged Pointer`不再指向`object`，首当其冲的就是 `isa` 指针

![](/img/taggedpointerisa1.png)

![](/img/taggedpointerisa2.png)

最后一个问题，编译器怎么判断一个指针是否是`Tagged Pointer`？

```mm
struct objc_object 
{
    bool isTaggedPointer() 
    {
        return ((uintptr_t)this & 1);
    }
};

```

很明显，编译器利用了普通指针4字节对齐的特征。

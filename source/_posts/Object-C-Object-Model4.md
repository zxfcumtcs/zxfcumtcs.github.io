title: 'Objective-C Object Model(4/4)'
date: 2014-07-13 00:26:28
categories:
- Objective-c
tags:
- OC
- IOS
---
本文是Objective-C Object Model系列的最后一篇文章，主要讨论Objective-C消息机制。
<!--more-->
©原创文章，转载请注明出处！

#C++成员函数
______________
首先，我们回顾一下C++是如何处理成员函数调用的:

C++编译器在编译C++源码时会对成员函数进行*Name Mangling*，[Name Mangling主要用于解决名称唯一问题](http://en.wikipedia.org/wiki/Name_mangling#Standardised_name_mangling_in_C.2B.2B)。

> In compiler construction, name mangling (also called name decoration) is a technique used to solve various problems caused by the need to resolve unique names for programming entities in many modern programming languages.

> It provides a way of encoding additional information in the name of a function, structure, class or another datatype in order to pass more semantic information from the compilers to linkers.


经过*Name Mangling*处理的函数名在整个项目中是唯一的，链接器就是通过这个名称来查找函数地址，并且函数调用是一个静态过程，在编译阶段就已确定(虚函数除外)。

由于C++支持*NameSpacing*，为了确保类名的唯一性，编译器同样要为类名进行*Name Mangling*处理。

如：

```
void MyClass::methodA(int param){    printf("%p %d\n", this, param);}
```

会被编译器转换为：

```
void __ZN7MyClass6methodAEi(MyClass *this, int param){    printf("%p %d\n", this, param);}
```
其中函数名__ZN7MyClass6methodAEi就是编译器对MyClass::methodA进行*Name Mangling*处理的结果。

#Objective-C消息机制
___________________
虽然C++与Objective-C同为面向对象编程语言，但在处理成员函数调用上有着本质区别，前者是*static binding*，后者是*dynamic binding*。

Objective-C中所有函数调用(或者说发送消息)都会被转换为调用`objc_msgSend`函数

例如：

```
[self addWithLeftOperan:1 rightOperand:2];
```

将会被编译器转化为：

```
objc_msgSend(self, @selector(addWithLeftOperan:rightOperand:), 1, 2);
```

来个复杂点的：

```
NSNumber *num = [[NSNumber alloc] initWithInt:10];
```

转换为：

```
NSNumber *num = ((id (*)(id, SEL, int))(void *)objc_msgSend)((id)((id (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSNumber"), sel_registerName("alloc")), sel_registerName("initWithInt:"), 10);
```

objc_msgSend函数原型如下：

```
objc_msgSend(receiver, selector, arg1, arg2, ...)
```

同样，编译器会对Objective-C的成员函数进行*NameSpacing*处理，如：

`- (int)addWithLeftOperan:(int)leftOpera rightOperan:(int)rightOpera`

会被编译器转换为：

`int className_addWithLeftOperan_rightOperan(id self, SEL _cmd, int leftOpera, int rightOpera)`

`objc_msgSend`的处理流程如下([来自](http://www.friday.com/bbum/2009/12/18/objc_msgsend-part-1-the-road-map/)):

1. Check for ignored selectors(GC) and short-circuit.

2. Check for nil target.
    If nil & nil receiver handler configured, jump to handler
    If nil & no handler (default), cleanup and return.
    
3. Find the IMP on the class of the target and jump to it

4. Search the class’s method cache for the method IMP
    If found, jump to it.

5. Not found: lookup the method IMP in the class itself 
    If found, jump to it.
    
6. If not found, jump to forwarding mechanism.

其中重点是第5步:通过`sel`查找`method IMP`，其查找过程是通过target的`isa`指针找到target所属的类，从而找到类的dispatch table(即方法表)，如果在该类的dispatch table中没有找到`sel`对应的`IMP`，则继续在其父类中找，依此类推，走到找到或找不到。

好吧，说到这不得不上一张见过N次的图片([来自](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtHowMessagingWorks.html#//apple_ref/doc/uid/TP40008048-CH104-SW1))：

![](/img/MessagingFramework.png)

嗯，同时还要上段源码才过瘾(有点长，但非常清晰)：

```
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    Class curClass;
    IMP imp = nil;
    Method meth;
    bool triedResolver = NO;

    rwlock_assert_unlocked(&runtimeLock);

    // Optimistic cache lookup
    if (cache)
    {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

 retry:
    rwlock_read(&runtimeLock);

    // Ignore GC selectors
    if (ignoreSelector(sel))
    {
        imp = _objc_ignored_method;
        cache_fill(cls, sel, imp);
        goto done;
    }

    // Try this class's cache.
    imp = cache_getImp(cls, sel);
    if (imp) 
        goto done;

    // Try this class's method lists.
    meth = getMethodNoSuper_nolock(cls, sel);
    if (meth)
    {
        log_and_fill_cache(cls, cls, meth->imp, sel);
        imp = meth->imp;
        goto done;
    }

    // Try superclass caches and method lists.
    curClass = cls;
    while ((curClass = curClass->superclass)) 
    {
        // Superclass cache.
        imp = cache_getImp(curClass, sel);
        if (imp) 
        {
            if (imp != (IMP)_objc_msgForward_impcache) 
            {
                // Found the method in a superclass. Cache it in this class.
                log_and_fill_cache(cls, curClass, imp, sel);
                goto done;
            }
            else
            {
                // Found a forward:: entry in a superclass.
                // Stop searching, but don't cache yet; call method 
                // resolver for this class first.
                break;
            }
        }

        // Superclass method list.
        meth = getMethodNoSuper_nolock(curClass, sel);
        if (meth) 
        {
            log_and_fill_cache(cls, curClass, meth->imp, sel);
            imp = meth->imp;
            goto done;
        }
    }

    // No implementation found. Try method resolver once.
    if (resolver  &&  !triedResolver) 
    {
        rwlock_unlock_read(&runtimeLock);
        _class_resolveMethod(cls, sel, inst);
        // Don't cache the result; we don't hold the lock so it may have 
        // changed already. Re-do the search from scratch instead.
        triedResolver = YES;
        goto retry;
    }

    // No implementation found, and method resolver didn't help. 
    // Use forwarding.

    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp);

 done:
    rwlock_unlock_read(&runtimeLock);

    // paranoia: look for ignored selectors with non-ignored implementations
    assert(!(ignoreSelector(sel)  &&  imp != (IMP)&_objc_ignored_method));

    // paranoia: never let uncached leak out
    assert(imp != _objc_msgSend_uncached_impcache);

    return imp;
}

static method_t *getMethodNoSuper_nolock(Class cls, SEL sel)
{
    rwlock_assert_locked(&runtimeLock);

    FOREACH_METHOD_LIST(mlist, cls, {
        method_t *m = search_method_list(mlist, sel);
        if (m) return m;
    });

    return nil;
}

static method_t *search_method_list(const method_list_t *mlist, SEL sel)
{
    int methodListIsFixedUp = isMethodListFixedUp(mlist);
    int methodListHasExpectedSize = mlist->getEntsize() == sizeof(method_t);
    
    if (__builtin_expect(methodListIsFixedUp && methodListHasExpectedSize, 1)) 
    {
        return findMethodInSortedMethodList(sel, mlist);
    }
    else 
    {
        // Linear search of unsorted method list
        method_list_t::method_iterator iter = mlist->begin();
        method_list_t::method_iterator end = mlist->end();
        for ( ; iter != end; ++iter) 
        {
            if (iter->name == sel) return &*iter;
        }
    }

    return nil;
}
```

有源码在，就什么都不说了。

有没有发现，Objective-C中函数调用过程与C++中的虚函数是不是有几份神似。

与C++的静态绑定相比，Objective-C中函数调用无疑在效率上有所不足。

当然，我们也可以绕过`objc_msgSend`，直接调用函数，如下：
```
- (void)testIMP:(NSInteger)count
{
    for (int i = 0; i < count; i++)
    {
        NSLog(@"i = %d", i);
    }
}

SEL sel = @selector(testIMP:);
void (*funpt)(id, SEL, NSInteger) = (void (*)(id, SEL, NSInteger))[self methodForSelector:sel];
funpt(self, sel, 20);
```

为了提高效率，Apple也做了些优化，其中之一就是`cache`，即将已经调用过的方法的`IMP`缓存起来，下次再调用时就不需要查表了。在上面的源码以及之前介绍的底层数据结构中都体现，就不过多描述了。

另外一项优化就是：**tail-call optimization(尾调用优化)**

前面讲到`objc_msgSend`函数原型：`objc_msgSend(id, SEL, ...)`;
OC的成员函数也会被编译器处理为：`className_selector(id self. SEL _cmd, ...)`;

没错，它们长的很像，但这不是巧合，正是为了利用**tail-call optimization**技术。
实现**tail-call optimization**的前提是**tail-call**，即一个函数里的最后一个动作是一个函数调用。
如下面的函数调用就是**tail-call**：
```
int funA
{
    // some code
    return [self funB];
}
```
**tail-call optimization**就是在调用另一函数时，不需要为其压入新的栈帧(`Stack Frame`)，而是利用原有的栈帧，最多在原有栈桢基础上做些修改。

由于OC中所有的函数调用都要转化为对`objc_msgSend`函数的调用，因此**tail-call optimization**显得尤为重要。不然栈帧的规模将会大一倍，从而过早出现**栈溢出(stack overflow)**现象。

另外由于`objc_msgSend`与被编译器处理过的成员函数原型完全一致，在利用**tail-call optimization**技术时，连原有栈桢都不需要修改，可以直接使用，也就极大优化了效率。

说到Objective-C的消息机制，Message Forwarding也是一个重要的话题，由于Apple的[官方文档](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtForwarding.html#//apple_ref/doc/uid/TP40008048-CH105-SW1)对此有详细的描述，我们就不做这个翻译工作了，感兴趣的同学可以去了解一下。

OK，该系列一共分了四部分，是该结束了。

**最后来个终极总结：**

**1. 什么是Objective-C对象？凡是以`isa`指针开始的`struct`都可以看作对象；**

**2. Objective-C Object-Orinet特性是构建在C语言`struct`基础之上；**

**3. 类与元类一一对应，创建类的同时也创建了相应的元类；**

**4. Objective-C中方法调用需要经过复杂的查找过程(其中有些缓存优化)。**
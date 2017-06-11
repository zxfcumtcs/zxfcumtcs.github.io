title: block那些事——__forwarding(4/5)
date: 2014-07-17 09:35:53
tags:
- Objective-C
- iOS
---

本文主要介绍`__forwarding`指针，`retain`、`release`以及`copy`对三种block的影响。
<!--more-->
©原创文章，转载请注明出处！

# __forwarding
______________
在[前文](http://zxfcumtcs.github.io/2014/07/15/block3/)代码片段**codeF**中提到编译器为**__block变量**`blockArr`合成的结构体`struct __Block_byref_blockArr_0`，
在该结构体中有一个指向结构自身类型的指针`__Block_byref_blockArr_0 *__forwarding`。

ok，问题出来，这些`__forwarding`指针的作用是什么呢？

前面我们讲到**__block变量会随block copy到heap而同样被copy到heap**，因此，在此情况下就存在同时访问stack上和heap上__block变量的情况：

![](/img/forwardingvar.png)
___________

很明显，第一次访问的`var`位于heap上，第二次访问的`var`位于stack上，因此，如果不采用特殊机制，将导致`var`的值不正确。

嗯，这就是**__forwarding**指针的使命所在——**确保能正确的访问__block变量**。

**当__block变量从stack到copy到heap上时，stack上的__forwarding被修改为指向heap上的__block变量，通过该机制，使得无论是在stack上还是heap上都能访问到同一个__block变量。**

[前文](http://zxfcumtcs.github.io/2014/07/15/block3/)代码片段**codeH**中的第44行就是修改stack上__block变量的__forwarding指针，使其指向heap上的__block变量：`_Block_assign(src->forwarding, (void **)destp)`。

最后再亮出我们的看家法宝——【[图](http://book.douban.com/subject/10536953/)】：

![](/img/forwardcopytoheap.jpg)

# block与retain、release以及copy
______________
对block来说，`release`与`Block_release`、`copy`与`Block_copy`等同。

还是用源码说话吧：

``` objectivec Block_release
void _Block_release(void *arg) 
{
    struct Block_layout *aBlock = (struct Block_layout *)arg;
    int32_t newCount;
    if (!aBlock) 
        return;
        
    newCount = latching_decr_int(&aBlock->flags) & BLOCK_REFCOUNT_MASK;        // BLOCK_REFCOUNT_MASK = (0xffff)
    if (newCount > 0) 
        return;
        
    // Hit zero
    if (aBlock->flags & BLOCK_NEEDS_FREE)                                     // NSConcreteMallocBlock
    {
        if (aBlock->flags & BLOCK_HAS_COPY_DISPOSE)(*aBlock->descriptor->dispose)(aBlock);
        _Block_deallocator(aBlock);
    }
    else if (aBlock->flags & BLOCK_IS_GLOBAL)                                 // NSConcreteGlobalBlock
    {
        ;
    }
    else                                                                     // NSConcreteStackBlock
    {
        printf("Block_release called upon a stack Block: %p, ignored\n", (void *)aBlock);
    }
}

```

由`_Block_release`的源码可以得出如下结论：

+ block在其内部标记`flags`中保存了自己的引用计数，且引用计数占用`flags`的前16位(BLOCK_REFCOUNT_MASK = 0xffff)(第8行)；

+ release NSConcreteMallocBlock类型的block，会将其引用计数-1，当引用计数<=0时，先调用block的dispose函数，再释放block本身(第13~17行)；

+ release NSConcreteGlobalBlock或NSConcreteStackBlock类型的block没有任何实质操作。

block retain操作没有相关的源码，但从效果来看：

+ retain NSConcreteMallocBlock类型的block，其引用计数会+1；

+ retain NSConcreteGlobalBlock或NSConcreteStackBlock类型的block没有任何实质操作.


``` objectivec Block_copy
void *_Block_copy(const void *arg)
{
    return _Block_copy_internal(arg, WANTS_ONE);
}

static void *_Block_copy_internal(const void *arg, const int flags) 
{
    struct Block_layout *aBlock;
    const bool wantsOne = (WANTS_ONE & flags) == WANTS_ONE;

    if (!arg) return NULL;
    
    aBlock = (struct Block_layout *)arg;
    if (aBlock->flags & BLOCK_NEEDS_FREE)         // NSConcreteMallocBlock
    {
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }
    else if (aBlock->flags & BLOCK_IS_GLOBAL)     // NSConcreteGlobalBlock
    {
        return aBlock;
    }

                                                 // Its a stack block.  Make a copy.
    struct Block_layout *result = malloc(aBlock->descriptor->size);
    if (!result) return (void *)0;
    memmove(result, aBlock, aBlock->descriptor->size); // bitcopy first
    // reset refcount
    result->flags &= ~(BLOCK_REFCOUNT_MASK);    // XXX not needed
    result->flags |= BLOCK_NEEDS_FREE | 1;
    result->isa = _NSConcreteMallocBlock;
    if (result->flags & BLOCK_HAS_COPY_DISPOSE) 
    {
            (*aBlock->descriptor->copy)(result, aBlock); // do fixup
    }
        
    return result;
}

```

由Block_copy的源码可以看出：

+ NSConcreteMallocBlock类型的block在copy时引用计数+1；

+ NSConcreteGlobalBlock类型的block在copy时直接返回；

+ NSConcreteStackBlock类型的block在copy时才会进行真正的copy操作。

看个有趣的例子：

![](/img/blockretainrelease1.jpg)

![](/img/blockretainrelease2.jpg)

![](/img/blockretainrelease3.jpg)

应该很清楚了，不用多说。

ok，最后总结一下：
![](/img/blockretainreleasesummary.png)


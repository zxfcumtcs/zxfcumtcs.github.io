title: block那些事——__block storage-class-specifier(3/5)
date: 2014-07-15 23:17:48
tags:
- Objective-C
- iOS
---
本文主要介绍__block变量、block copy对外部引用变量的影响。
<!--more-->
©原创文章，转载请注明出处！

# __block storage-class-specifier
______________
我们知道在block内可以引用block外部的变量，那么能随意修改block外部的变量吗？

答案显然是否定的，不然就没有下文了^_^

在block内部能修改的变量有：
     
+ 全局变量、全局静态变量、静态变量以及类的成员变量；
+ 用`__block`修饰的自变量。

本文的主角就是`__block`，更准确的说是`__block storage-class-specifier`(C语言中`storage-class-specifier`有：`typedef`、`extern`、`static`、`auto`以及`register`)，其主要用于修饰需要在block中修改的自变量(*automatic variables*)。

那么，编译器是如何处理`__block`变量的呢？
_____________

![](/img/blockvarsource.jpg)

![](/img/blockvarresult.jpg) 

![](/img/blockref.jpg)

_____________

是不是有点复杂...

**__block类型的变量会被编译器转换为一个结构体。**

在最新的开源代码中，用于表示`__block`变量的结构`Block_byref`定义如下：

``` mm codeA

struct Block_byref 
{
    void *isa;
    struct Block_byref *forwarding;
    volatile int32_t flags; // contains ref count
    uint32_t size;
};

struct Block_byref_2 
{
    // requires BLOCK_BYREF_HAS_COPY_DISPOSE
    void (*byref_keep)(struct Block_byref *dst, struct Block_byref *src);
    void (*byref_destroy)(struct Block_byref *);
};

struct Block_byref_3 
{
    // requires BLOCK_BYREF_LAYOUT_EXTENDED
    const char *layout;
};


```

`Block_byref`与`Block_descriptor`类似，也是就3部分组成。

由*codeA*可以看到`Block_byref`也含有`isa`指针，因此也可以将其看作OC对象，但从编译器转换结果来看，该`isa`指针被置为`nil`。

通过前面的分析可以得出如下结论：

**block以及__block修饰的变量本质上都是结构体类型的自变量(an automatic variable of a struct)。**([来源](http://book.douban.com/subject/10536953/))

![](/img/blockandblockvar.jpg)

# block copy与其引用的外部变量
______________
## block copy与普通object类型变量

[前文](http://zxfcumtcs.github.io/2014/07/15/block2/)我们总结过，当block从stack copy到heap上时，会retain其引用的外部object。

我们看看编译器为此做了什么：

```mm codeB
int main()
{
    NSArray *arr = [[NSArray alloc] init];

    void (^blockA)(void) = ^{
        [arr addObject:@"test"];

    };
    
    return 0;
}
```

**block在copy时，会调用其`copy`方法完成对引用变量的copy，编译器为`blockA`合成的`copy`函数如下：**

```mm codeC
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src)
{
    _Block_object_assign((void*)&dst->arr, (void*)src->arr, 3//*BLOCK_FIELD_IS_OBJECT*//);
}

```

可以看到，合成的copy方法`__main_block_copy_0`调用了[runtime](http://opensource.apple.com/source/libclosure/libclosure-63/runtime.c)的`_Block_object_assign`方法来完成copy操作：

```mm codeD
void _Block_object_assign(void *destAddr, const void *object, const int flags) 
{
    if ((flags & BLOCK_BYREF_CALLER) == BLOCK_BYREF_CALLER) 
    {
        if ((flags & BLOCK_FIELD_IS_WEAK) == BLOCK_FIELD_IS_WEAK) 
        {
            _Block_assign_weak(object, destAddr);
        }
        else 
        {
            _Block_assign((void *)object, destAddr);
        }
    }
    else if ((flags & BLOCK_FIELD_IS_BYREF) == BLOCK_FIELD_IS_BYREF)  
    {
        _Block_byref_assign_copy(destAddr, object, flags);
    }
    else if ((flags & BLOCK_FIELD_IS_BLOCK) == BLOCK_FIELD_IS_BLOCK) 
    {
        _Block_assign(_Block_copy_internal(object, flags), destAddr);
    }
    else if ((flags & BLOCK_FIELD_IS_OBJECT) == BLOCK_FIELD_IS_OBJECT) 
    {
        _Block_retain_object(object);
        _Block_assign((void *)object, destAddr);
    }
}

```

第24行，retain了block内引用的对象，在本例中为`arr`。
第25行，将源block中`arr`的地址赋给了目标block中的`arr`变量。

## block copy与__block修饰的object类型变量

**我们知道，在ARC下block会retain __block修饰变量，而在非ARC下不会retain。**

还是通过代码来了解：

```mm codeE
int main()
{
    NSArray *arr = [NSArray array];
    __block NSArray *blockArr = arr;
    
    void (^blockB)(void) = ^{
        [blockArr addObject:@"obj"];
    };
    
    return 0;
}
```

编译器合成的代码：

```mm codeF
int main()
{
    NSArray *arr = ((NSArray *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSArray"), sel_registerName("array"));
    
    __attribute__((__blocks__(byref))) __Block_byref_blockArr_0 blockArr =
    {
        (void*)0,
        (__Block_byref_blockArr_0 *)&blockArr,
        33554432,    // BLOCK_HAS_COPY_DISPOSE 表示有自己的copy、dispose函数
        sizeof(__Block_byref_blockArr_0),
        __Block_byref_id_object_copy_131,
        __Block_byref_id_object_dispose_131,
        arr
    };

    void (*blockA)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0,
                                                            &__main_block_desc_0_DATA,
                                                            (__Block_byref_blockArr_0 *)&blockArr,
                                                            570425344);

    return 0;
}

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src)
{
    _Block_object_assign((void*)&dst->blockArr, (void*)src->blockArr, 8/*BLOCK_FIELD_IS_BYREF*/);
}

struct __Block_byref_blockArr_0 
{
    void *__isa;
    __Block_byref_blockArr_0 *__forwarding;
    int __flags; 
    int __size; 
    void (*__Block_byref_id_object_copy)(void*, void*); 
    void (*__Block_byref_id_object_dispose)(void*); 
    NSArray *blockArr;
};

static void __Block_byref_id_object_copy_131(void *dst, void *src) 
{
    _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}

```

为了讨论方便我们将`blockArr`称为**__block变量**，`arr`称为**原始__block变量**。

前面我们讲到在ARC下block会retain **__block修饰变量**，而在非ARC下不会retain，严格地讲这里所说的变量是指**原始__block变量**。

之所以ARC下会retain**原始__block变量**，是因为ARC下变量默认声明为`__strong`(上面代码的第37行：`NSArray *blockArr;`)。

通过比较可知：编译器为`blockB`合成的copy函数与为`blockA`合成的copy函数唯一的差别在调用`_Block_object_assign`函数时最后一个参数不一样。

```mm codeG

else if ((flags & BLOCK_FIELD_IS_BYREF) == BLOCK_FIELD_IS_BYREF)  
{
    _Block_byref_assign_copy(destAddr, object, flags);
}

```

```mm codeH
static void _Block_byref_assign_copy(void *dest, const void *arg, const int flags) 
{
    struct Block_byref **destp = (struct Block_byref **)dest;
    struct Block_byref *src = (struct Block_byref *)arg;
        
    if ((src->forwarding->flags & BLOCK_REFCOUNT_MASK) == 0) 
    {
        bool isWeak = ((flags & (BLOCK_FIELD_IS_BYREF|BLOCK_FIELD_IS_WEAK)) == (BLOCK_FIELD_IS_BYREF|BLOCK_FIELD_IS_WEAK));

        struct Block_byref *copy = (struct Block_byref *)_Block_allocator(src->size, false, isWeak);
        copy->flags = src->flags | _Byref_flag_initial_value; // non-GC one for caller, one for stack
        copy->forwarding = copy; // patch heap copy to point to itself (skip write-barrier)
        src->forwarding = copy;  // patch stack to point to heap copy
        copy->size = src->size;
        
        if (isWeak) 
        {
            copy->isa = &_NSConcreteWeakBlockVariable;  // mark isa field so it gets weak scanning
        }
        
        if (src->flags & BLOCK_HAS_COPY_DISPOSE) 
        {
            // Trust copy helper to copy everything of interest
            // If more than one field shows up in a byref block this is wrong XXX
            copy->byref_keep = src->byref_keep;
            copy->byref_destroy = src->byref_destroy;
            (*src->byref_keep)(copy, src);
        }
        else 
        {
            // just bits.  Blast 'em using _Block_memmove in case they're __strong
            _Block_memmove(
                (void *)&copy->byref_keep,
                (void *)&src->byref_keep,
                src->size - sizeof(struct Block_byref_header));
        }
    }
    else if ((src->forwarding->flags & BLOCK_NEEDS_FREE) == BLOCK_NEEDS_FREE)     // already copied to heap
    {
        latching_incr_int(&src->forwarding->flags);
    }
    
    // assign byref data block pointer into new Block
    _Block_assign(src->forwarding, (void **)destp);
}

```

对`blockB`的copy操作会执行到codeH中的第27行，表示执行**__block变量自己的copy函数**，即codeF中的`__Block_byref_id_object_copy_131`函数。

呵呵，最终又回到了`_Block_object_assign`函数，终极执行的代码是codeD中的第11行：`_Block_assign((void *)object, destAddr);`，

很明显，只是指针间的赋值，没有任何`retain`操作，也就说明了在非ARC下，block不会`retain`**原始__block变量**。

`_Block_object_assign`函数最后一个参数`flags`的含义：

```mm
enum 
{
    BLOCK_FIELD_IS_OBJECT   =  3,  // id, NSObject, __attribute__((NSObject)), block, ...
    BLOCK_FIELD_IS_BLOCK    =  7,  // a block variable
    BLOCK_FIELD_IS_BYREF    =  8,  // the on stack structure holding the __block variable
    BLOCK_FIELD_IS_WEAK     = 16,  // declared __weak, only used in byref copy helpers
    BLOCK_BYREF_CALLER      = 128, // called from __block (byref) copy/dispose support routines.
};
```

有点晕了吧，看几张图吧，清醒一下^_^

**注意：下面讲的都是__block变量本身，而非原始__block变量**

block从stack copy到heap上时对__block变量的影响([来源](http://book.douban.com/subject/10536953/))：
![](/img/blockvarcopytable.jpg)

当block从stack copy到heap上时，对于在其内部使用的、并且在stack上的__block变量也会被copy到heap上，并且会持有该变量([来源](http://book.douban.com/subject/10536953/)):

![](/img/blockvarcopytoheap.jpg)
___________________

当该__block变量已经被copy到heap上时，其引用计数会+1([来源](http://book.douban.com/subject/10536953/)):

![](/img/blockvaralreadyinheap.jpg)
___________________

当heap上的block被释放时，其内部引用到的__block变量也会被release([来源](http://book.douban.com/subject/10536953/)):

![](/img/disposedblockvar.jpg)
___________________

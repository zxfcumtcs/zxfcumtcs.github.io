title: block那些事——block 内部结构(1/5)
date: 2014-07-14 19:12:57
categories:
- Objective-c
tags:
- OC
- IOS
---
本文主要介绍block内部结构以及实现机制。
<!--more-->
©原创文章，转载请注明出处！

#block内部结构
______________
ok，假设大家已经了解了block语法。

呵呵，讲解技术问题的两大法宝：**源码**+**图**，在此也不例外。

[源码](http://opensource.apple.com/source/libclosure/libclosure-38/Block_private.h)：

```
// revised new layout
struct Block_descriptor
{
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
};

struct Block_layout 			// block对应的结构体
{
    void *isa;
    int flags;
    int reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    // imported variables
};

```

[最新的开源代码](http://opensource.apple.com/source/libclosure/libclosure-63/Block_private.h)对`Block_descriptor`做了优化：


```
struct Block_descriptor_1 
{
    uintptr_t reserved;
    uintptr_t size;
};

struct Block_descriptor_2 
{
    // requires BLOCK_HAS_COPY_DISPOSE
    void (*copy)(void *dst, const void *src);
    void (*dispose)(const void *);
};

struct Block_descriptor_3 
{
    // requires BLOCK_HAS_SIGNATURE
    const char *signature;
    const char *layout;     // contents depend on BLOCK_HAS_EXTENDED_LAYOUT
};

struct Block_layout 
{
    void *isa;
    volatile int32_t flags; // contains ref count
    int32_t reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 *descriptor;
    // imported variables
};

```

```
static struct Block_descriptor_1 * _Block_descriptor_1(struct Block_layout *aBlock)
{
    return aBlock->descriptor;
}

static struct Block_descriptor_2 * _Block_descriptor_2(struct Block_layout *aBlock)
{
    if (! (aBlock->flags & BLOCK_HAS_COPY_DISPOSE)) 
        return NULL;
        
    uint8_t *desc = (uint8_t *)aBlock->descriptor;
    desc += sizeof(struct Block_descriptor_1);
    
    return (struct Block_descriptor_2 *)desc;
}

```

其实也很简单，当block需要`copy`、`dispose`函数时，才在`descriptor`中包含相应的函数指针(`Block_descriptor_3`用于存储与ABI相关的metadata，具体细节不清楚)。

那么什么情况下block才需要`copy`、`dispose`函数呢？

**简单来说，就是在block中引用了外部的OC对象或其他block时。**后面会有更详细的讲解。

![](/img/source_noobjectinblock.jpg)
![](/img/result_noobjectinblock.jpg)

在上面这个例子中，由于`blockA`没有引用外部对象，因此编译器不会为其合成`copy`和`dispose`函数。

![](/img/source_hasobjectinblock.jpg)
![](/img/result_hasobjectinblock.jpg)

而在这个例子中由于`blockB`引用了`blockA`，故编译器为`blockB`合成了`copy`和`dispose`函数。

[图](http://www.galloway.me.uk/2013/05/a-look-inside-blocks-episode-3-block-copy/)：

![](/img/block_layout.png)

最后，我们用clang验证一下(利用clang的-rewrite-objc命令可以将Objectvie-C源码转换为C语言源码)：

``` objectivec 1.c
int main()
{
    void (^blockA)(void) = ^{
        printf("in blockA");
    };
    
    return 0;
}
```

利用clang的-rewrite-objc：`clang -rewrite-objc 1.c`会生成相应的cpp文件：

``` objectivec 1.cpp
struct __block_impl 
{
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

struct __main_block_impl_0
{
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int a;
  
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int flags=0) : a(_a) 
  {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) 
{
  int a = __cself->a; // bound by copy

  printf("in blockA a = %d", a);
}

static struct __main_block_desc_0 
{
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main()
{
    int a = 10;
    void (*blockA)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, a);

    return 0;
}
```

嗯，已经很清楚了，block会被编译器转换为C语言的`struct`。

额，熟悉的`isa`指针又出现在了表示block的结构体中，通过[《Objective-C Object Model》](http://localhost:4000/2014/07/13/Object-C-Object-Model4/)系列文章我们知道，含有`isa`指针的struct，在Objective-C中都可以被认为是Object。

那么block里面的`isa`指针指向谁呢？

上述代码17行就是答案所在：`impl.isa = &_NSConcreteStackBlock;`，即block对应的类是`_NSConcreteStackBlock`。

是的，根据block的不同，可以分为三种类型：`_NSConcreteStackBlock`、`_NSConcreteMallocBlock`以及`_NSConcreteGlobalBlock`。

<font color = red size = 4>***注意：在很多资料上看到，说在ARC下，只有`_NSConcreteMallocBlock`和`_NSConcreteGlobalBlock`两种类型的block。其实不是这样的……***</font>

![](/img/blocktypeinarc.jpg)

<font color = red>这是个ARC的例子，我们可以看到`__weak`类型的block以及作为函数参数的block都是`_NSConcreteStackBlock`类型！</font>

之所以大家会误认为ARC下只有`_NSConcreteMallocBlock`和`_NSConcreteGlobalBlock`类型的block，是因为将block赋给`strong`类型的变量时，编译器会自动将其copy到heap上！如上面的`strongBlk`。

根据这三种block类型的名称，我们也能猜出它们在内存上的分布情况([来源](http://book.douban.com/subject/10536953/))：

![](/img/blockmemoryarrangement.jpg)

###_NSConcreteGlobalBlock

很明显，`_NSConcreteGlobalBlock`类型的block分配在数据区域。以全局的方式定义的block，其类型为`_NSConcreteGlobalBlock`。

![](/img/globalblock1.png)
![](/img/globalblock2.png)

注意：在函数内部定义的block，如果该block没有引用block外的任何自变量，那么该block的类型同样是`_NSConcreteGlobalBlock`，并且存储在data section中。


因此，以下两种情况下生成的block为_NSConcreteGlobalBlock类型：

+ 以全局的方式定义的block；

+ 在block内没有引用block外的任何自变量。

###_NSConcreteMallocBlock

`_NSConcreteMallocBlock`类型的block存储在heap上，通常是通过copy函数将stack上的block copy到heap上。

###_NSConcreteStackBlock

`_NSConcreteStackBlock`类型的block存储在stack上，除了上述两种类型的block，其他block都属于`_NSConcreteStackBlock`。


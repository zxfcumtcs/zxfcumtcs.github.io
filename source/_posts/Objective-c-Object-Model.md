title: 'Objective-C Object Model(1/4)'
date: 2014-07-08 18:53:46
tags:
- Objective-C
- iOS
---
本系列文章我们一起探讨Objective-C对象模型(Object Model)，本文主要讨论与Object Model相关的底层数据结构。
<!--more-->
©原创文章，转载请注明出处！

# 什么是对象模型(Object Model)
______________
说起Object Model不得不提起*Stanley B. Lippman*以及他的著作[*《Inside the C++ Object Model》*](http://book.douban.com/subject/1091086/)。Lippman作为C++大师，参与设计了第一套C++编译程序cfront，其主要著作相信大家也耳熟能详：[*《C++ Primer》*](http://book.douban.com/subject/25708312/)，[*《Inside the C++ object model》*](http://book.douban.com/subject/1091086/)，[*《Essential C++》*](http://book.douban.com/subject/1456836/)等。

在*《Inside the C++ Object Model》*中认为Object Model主要阐述两个层次的问题：

+  在语言层面对Object-Oriented Programming的支持(广大程序猿天天要用到的各种高大上的OOP特性)

+  Object-Oriented Programming的底层实现机制(伟大的编译器如何处理这些高大上的OOP特性)

嗯，本文主要讨论第二个问题。

# Objective-C中与Object Model相关的数据类型
______________
### NSObject——万物之源

``` objectivec
@interface NSObject <NSObject> 
{
    Class isa  OBJC_ISA_AVAILABILITY;
}
```

我们都知道正常情况下所有的Objective-C类都是继承自`NSObject`这个原始基类，`isa`这个Objective-C Object的精神领袖就包含在其中。

继续...

### objc_class——类的元数据

``` objectivec
typedef struct objc_class *Class;
```

Class就是一个指向`objc_class struct`的指针。

``` objectivec
struct objc_class objc_class public
{
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```

在Objective-C runtime的公开接口中可以看到`objc_class`是一个指向自身类型的结构体(从Objective-C 2.0开始，该结构体中其余成员已不再可用)。

由于[Objective-C runtime](http://www.opensource.apple.com/tarballs/objc4/)已经开源，我们可以进一步一探究竟。

在最新的开源代码objc4-551.1中(以下代码只摘取了其中的成员变量)：

``` objectivec objc_class private
struct objc_class : objc_object
 {
    Class superclass;
    cache_t cache;
    uintptr_t data_NEVER_USE;  // class_rw_t * plus custom rr/alloc flags
 };
```

可以看到`objc_class`继承自`objc_object`，这也间接说明了Objective-C中一个重要的概念：*类也是对象*。

### objc_object
``` objectivec objc_object
struct objc_object
{
private:
    uintptr_t isa;
}：
```
在`objc_object`看到了久违的`isa`指针。

额...，此时或许有个疑问了，类的成员变量、方法等元信息咋还没登场？
不要急，他们来了...

### class_rw_t

或许已经注意到在`objc_class`结构中有个成员变量`uintptr_t data_NEVER_USE;`。
没错，它就是指向`class_rw_t`结构体的指针(其中做了一些优化，在此不作过多讨论，只简单的认为`data_NEVER_USE`指向`class_rw_t`结构体)。

``` objectivec objc_object class_rw_t\class_ro_t
struct class_rw_t 
{
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    union 
    {
        method_list_t **method_lists;  // RW_METHOD_ARRAY == 1
        method_list_t *method_list;    // RW_METHOD_ARRAY == 0
    };
    struct chained_property_list *properties;
    const protocol_list_t **protocols;

    Class firstSubclass;
    Class nextSiblingClass;
};

struct class_ro_t 
{
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    const method_list_t * baseMethods;
    const protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    const property_list_t *baseProperties;
};
```

其中，类名`name`、实例大小`instanceSize`、子实例相对整个实例的偏移`instanceStart`以及实例变量`ivars`等信息保存在`class_ro_t`中，方法列表`method_lists`、协议`protocols`保存在`class_rw_t`中。

这里有个小细节，当类只有一个方法时，该方法直接保存在`method_list`中，否则保存在方法列表中`method_lists`。

``` objectivec attachMethodLists
    if (newCount > 1)
    {
        assert(newLists != newBuf);
        cls->data()->method_lists = newLists;
        cls->setInfo(RW_METHOD_ARRAY);
    } 
    else
    {
        assert(newLists == newBuf);
        cls->data()->method_list = newLists[0];
        assert(!(cls->data()->flags & RW_METHOD_ARRAY));
    }

```

**我们都知道可以在分类中添加方法，而不能添加成员变量(或属性)，从其内部实现我们也能得出这样的结论：**
+ *方法是保存在方法表里面的(只有一个方法情况除外)，因此添加方法不会改变实例的大小，也不会引起实例内存布局的改变；*

+ *若添加成员变量将引起实例内存布局的改变(`class_ro_t`的`instanceStart`、`instanceSize`)，同时在`class_rw_t`中`class_ro_t`被声明为`const`，因此想改也改不了。*

### Other

``` objectivec
struct method_list_t 
{
    uint32_t entsize_NEVER_USE;  // high bits used for fixup markers
    uint32_t count;
    method_t first;
};

struct method_t 
{
    SEL name;
    const char *types;
    IMP imp;
}

typedef struct objc_object *id;

// objc_selector结构体的源码没有找到,可以简单的将其理解为由函数名组成的字符串
typedef struct objc_selector *SEL;
                               
typedef id (*IMP)(id, SEL, ...);          // IMP就是一个函数指针

struct ivar_t 
{
    int32_t *offset;
    const char *name;
    const char *type;

    uint32_t alignment_raw;
    uint32_t size;
};

struct ivar_list_t 
{
    uint32_t entsize;
    uint32_t count;
    ivar_t first;
};

struct property_t 
{
    const char *name;
    const char *attributes;
};

struct property_list_t 
{
    uint32_t entsize;
    uint32_t count;
    property_t first;
};

struct protocol_t : objc_object 
{
    const char *name;
    struct protocol_list_t *protocols;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    method_list_t *optionalInstanceMethods;
    method_list_t *optionalClassMethods;
    property_list_t *instanceProperties;
    uint32_t size;   // sizeof(protocol_t)
    uint32_t flags;
    const char **extendedMethodTypes;
};

struct protocol_list_t 
{
    uintptr_t count;
    protocol_ref_t list[0]; // variable-size
};

```

**我们可以看到Objective-C的Object-Orient是构建在C语言结构体基础之上，大到类、对象，小到成员变量、方法等等。**

***插个题外话：个人感觉C语言是一门简洁、优雅的语言，而这一切都归功于指针，可以说指针是C语言的灵魂。***

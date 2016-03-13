title: 'Objective-C Object Model(2/4)'
date: 2014-07-10 18:03:52
categories:
- Objective-c
tags:
- OC
- IOS
---
在[前文](http://zhaoxuefeng.gitcafe.com/2014/07/08/Objective-c-Object-Model/)我们详细讲解了Objective-C OOP的底层数据结构，本文我们一起了解一下Objective-C中类(`class`)、父类(`super class`)、元类(`meta class`)、根父类(`root super class`)以及根元类(`root meta class`)间的关系。
<!--more-->
©原创文章，转载请注明出处！

#类、元类、根父类、根元类
______________
从[前文](http://zhaoxuefeng.gitcafe.com/2014/07/08/Objective-c-Object-Model/)我们知道，在OC中类也是对象(`objc_class`继承自`objc_object`)，也有`isa`指针。实例的`isa`指针指向类(*确切的说是指向代表类的实例*)，那么类`isa`指针指向谁呢？

答案是指向一个叫元类的东东(`metaclass`)，显然`metaclass`也是对象，那它的`isa`又指向何方？

答案是指向一个叫根元类的东东(`root metaclass`), 显然`root metaclass`也是对象，那它的`isa`又指向哪里？

额，这是要进入死循环的节奏...

还好，`root metaclass`的`isa`指向它自己。

好吧，有点晕了，上个图吧([来源](http://www.sealiesoftware.com/blog/class%20diagram.pdf)):

![](/img/class_diagram.jpg)

相关的源码：

```
bool isMetaClass() 
{
    assert(this);
    assert(isRealized());
    return data()->ro->flags & RO_META;
}

// NOT identical to this->ISA when this is a metaclass
Class getMeta() 
{
    if (isMetaClass()) return (Class)this;
    else return this->ISA();
}

bool isRootClass() 
{
    return superclass == nil;
}

bool isRootMetaclass()
{
    return ISA() == (Class)this;
}
```

下面是创建一个类相关的代码(仅摘取核心代码，[完整代码](http://www.opensource.apple.com/source/objc4/objc4-551.1/runtime/objc-runtime-new.mm))：
```
Class objc_allocateClassPair(Class superclass, const char *name, size_t extraBytes)
{
    Class cls, meta;

    rwlock_write(&runtimeLock);

    // Allocate new classes.
    size_t size = sizeof(objc_class);
    size_t metasize = sizeof(objc_class);
    if (superclass) 
    {
        size = superclass->ISA()->alignedInstanceSize();
        metasize = superclass->ISA()->ISA()->alignedInstanceSize();
    }
    cls  = _calloc_class(size + extraBytes);
    meta = _calloc_class(metasize + extraBytes);

    objc_initializeClassPair_internal(superclass, name, cls, meta);

    rwlock_unlock_write(&runtimeLock);

    return cls;
}

static void objc_initializeClassPair_internal(Class superclass, const char *name, Class cls, Class meta)
{
    // Connect to superclasses and metaclasses
    cls->initIsa(meta);
    if (superclass) 
    {
        meta->initIsa(superclass->ISA()->ISA());
        cls->superclass = superclass;
        meta->superclass = superclass->ISA();
        addSubclass(superclass, cls);
        addSubclass(superclass->ISA(), meta);
    }
    else
    {
        meta->initIsa(meta);
        cls->superclass = Nil;
        meta->superclass = cls;
        addSubclass(cls, meta);
    }
}

```
可以看到，类和元类是一一对应的，创建类的同时创建了相应的元类，同时保持了它们间的继承关系

总结一下：

+ 对象的`isa`指针指向类对象

+ 类对象的`isa`指针指向元类对象

+ 元类对象的`isa`指针指向根元类

+ 根元类对象的`isa`指针自身

+ 类与元类一一对应

+ 类的继承关系在元类中也有对应的继承关系

+ 根类(`NSObject`)没有`super class`

+ 元根类继承自根类
title: block那些事——block copy(2/5)
date: 2014-07-15 17:06:54
categories:
- Objective-c
tags:
- OC
- IOS
---
本文主要介绍ARC和非ARC下block copy相关的问题。
<!--more-->
©原创文章，转载请注明出处！

# Block Copy
______________
通过[前文](http://zxfcumtcs.github.io/2014/07/14/block/)我们知道，起始大多数block都是创建在stack上，需要通过`Block_copy`或`copy`函数才能将其copy到heap上。

那么什么情况下需要我们手动调用`Block_copy`或`copy`函数将block copy到heap？

### ARC

ARC下，编译器在大多数情况下能自动识别并在需要的时候自动将stack上的block copy到heap上，如：

+ 函数返回值是block；

+ 对strong类型的block变量赋值(在[前文](http://zxfcumtcs.github.io/2014/07/14/block/)我们已经看到过这样的例子)。

**在ARC下，编译器不会自动copy的情况：block作为函数的参数、为`__weak`类型的block变量赋值**

**但如果函数内部会copy block参数，则调用者不需要手工copy作为参数的block了，如：**

+ 包含**usingBlock**的Cocoa Framework方法；

+ **GCD API**，如：disaptch_asyn会copy其block参数；

+ 内置容器自带API(如`NSArray`)。

**以上前两条也适用非ARC的情况，但第三条就不适用了。**

![](/img/blockstackcrashinnonarc.jpg)

以上这个例子在ARC下正常，但在非ARC下会crash！


### 非ARC

非ARC下，编译器不会自动copy block。

那么在什么情况下需要手动调用copy函数呢？

终极标准：**在block内引用了block外的对象类型的自变量时。**

![](/img/nonarcvar.jpg)

如上图这种情况，stack上的block没有retain其引用的object。

![](/img/nonarccopyvar.jpg)

将stack上的block copy到heap上时，会retain其引用的object。

![](/img/arcvar.jpg)

上图是ARC下的结果，可以看出编译器已经自动将其copy到heap上了。

上面讲到的都是对stack上的block进行copy，那么对其他两种类型的block进行copy操作会有什么结果呢？([来源](http://book.douban.com/subject/10536953/))

![](/img/howcopywork.png)

如果实在搞不清楚是否要copy block，那再来个傻瓜式的指导原则：([来源](http://book.douban.com/subject/10536953/))

![](/img/whencopyblock.jpg)

另外，无论是ARC还是非ARC，对block进行多次copy都不会有问题。

**小结：**

**+ _NSConcreteStackBlock类型的block不会retain其引用的外部object；**

**+ 将_NSConcreteStackBlock类型的block copy到heap上变成_NSConcreteMallocBlock类型的block时，会retain其引用的外部object(非ARC时__block变量除外)；**

**+ ARC下同样存在_NSConcreteStackBlock类型的block；**

**+ ARC下编译器不会自动copy作为函数参数的block，需要手动copy(包含usingBlock的Cocoa Framework方法、GCD API以及内置容器自带API除外)。**
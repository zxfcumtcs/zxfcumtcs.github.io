title: performSelector系列方法
date: 2014-07-19 18:19:19
categories:
- IOS
tags:
- IOS
---

本文讨论一下我们非常熟悉的performSelector系列方法。
<!--more-->
©原创文章，转载请注明出处！

# 普通系列
_________________________
+ \- (id)performSelector:(SEL)aSelector;
+ \- (id)performSelector:(SEL)aSelector withObject:(id)object;
+ \- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;

我们来看看这几个函数的[源码](http://www.opensource.apple.com/tarballs/objc4/)：

```
- (id)performSelector:(SEL)sel 
{
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL))objc_msgSend)(self, sel);
}

- (id)performSelector:(SEL)sel withObject:(id)obj 
{
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL, id))objc_msgSend)(self, sel, obj);
}

- (id)performSelector:(SEL)sel withObject:(id)obj1 withObject:(id)obj2 
{
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL, id, id))objc_msgSend)(self, sel, obj1, obj2);
}

```

可以看到它们只是简单的通过`obj_send`方法调用相应的方法，只是performSelector具有动态性。

# wait系列
_______________________
+ \- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thread withObject:(id)arg waitUntilDone:(BOOL)wait;
+ \- (void)performSelector:(SEL)aSelector onThread:(NSThread \*)thread withObject:(id)arg waitUntilDone:(BOOL)wait modes:(NSArray \*)array;
+ \- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(id)arg waitUntilDone:(BOOL)wait;
+ \- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(id)arg waitUntilDone:(BOOL)wait modes:(NSArray *)array;

其中`wait`参数为`YES`时会阻塞当前线程，如果当前线程与目标线程是同一线程时，且`wait`为`YES`，则`selector`立即被执行；

若`wait`为`NO`，performSelector在目标线程的runloop上将消息入队，并立即返回，无论目标线程是否是当前线程。且加入队列的消息不能被取消。

# afterDelay系列
________________________
+ \- (void)performSelector:(SEL)aSelector withObject:(id)anArgument afterDelay:(NSTimeInterval)delay;
+ \- (void)performSelector:(SEL)aSelector withObject:(id)anArgument afterDelay:(NSTimeInterval)delay inModes:(NSArray *)modes;

上述方法会在当前线程的runloop上安装timer，当timer触发时，当前线程企图从runloop中出队该消息并执行。如果runloop正在以指定的mode运行，则成功执行该消息，否则timer继续等待。

# other
________________________
+ \- (void)performSelectorInBackground:(SEL)aSelector withObject:(id)arg;

该方法将创建新的线程来执行selector，selector对应的方法必须准备好线程环境，就像自己开启一个新线程一样。

**以上方法都属于NSObject类。**


+ \- (void)performSelector:(SEL)aSelector target:(id)target argument:(id)anArgument order:(NSUInteger)order modes:(NSArray *)modes;

**该方法属于NSRunLoop类。**

该方法在当前线程的下一次runloop迭代时启动一个timer来执行selector，该方法在selector执行前就返回，因此将会retain target和anArgument object，直到timer触发，并作为cleanup的一部分而release。

那么什么时侯会用到这个方法呢？Apply 的原话：

Use this method if you want multiple messages to be sent after the current event has been processed and you want to make sure these messages are sent in a certain order.

---
title: ReactNative源码解析——通信机制详解(补充条款)
date: 2017-11-22 23:13:53
tags:
- 框架
- iOS
- ReactNative
---
本文通过分析 RN 源码，简要介绍了 JS to Native 的 callback 实现原理以及 RN 中的三个重要线程。
<!--more-->
©原创文章，转载请注明出处！
# callback
_______________________

前两篇文章[ReactNative源码解析——通信机制详解(1/2)
](https://zxfcumtcs.github.io/2017/10/08/ReactNativeCommunicationMechanism/)、[ReactNative源码解析——通信机制详解(2/2)](https://zxfcumtcs.github.io/2017/10/12/ReactNativeCommunicationMechanism2/)分别介绍了 RN 通信机制中的 JS to Native、Native to JS 的执行流程。为了集中注意力抓住主要流程，当时没有分析调用过程中的 callback 问题，下面简要分析一下 JS to Native callback 的实现原理。
> 文中所列代码均做了简化处理。

首先，来看一个具体的例子：
```mm
RCT_EXPORT_METHOD(showShareActionSheetWithOptions:(NSDictionary *)options
                  failureCallback:(RCTResponseErrorBlock)failureCallback
                  successCallback:(RCTResponseSenderBlock)successCallback)
{
    UIActivityViewController *shareController =
    [[UIActivityViewController alloc] initWithActivityItems:items applicationActivities:nil];

    shareController.completionWithItemsHandler = 
    ^(NSString *activityType, 
       BOOL completed, 
       __unused NSArray *returnedItems, 
       NSError *activityError) {
           if (activityError) {
               failureCallback(activityError);
           } 
           else {
               successCallback(@[@(completed), RCTNullIfNil(activityType)]);
           }
       };
}
```
`showShareActionSheetWithOptions:failureCallback:successCallback:`是`RCTActionSheetManager`曝露给 JS 的方法之一，其包含两个 callback：`failureCallback`、`successCallback`。
其中，`RCTResponseErrorBlock`、`RCTResponseSenderBlock`的定义如下：
```mm
/**
 * The type of a block that is capable of sending a response to a bridged
 * operation. Use this for returning callback methods to JS.
 */
typedef void (^RCTResponseSenderBlock)(NSArray *response);

/**
 * The type of a block that is capable of sending an error response to a
 * bridged operation. Use this for returning error information to JS.
 */
typedef void (^RCTResponseErrorBlock)(NSError *error);
```
JS 中的调用：
```js
ActionSheetIOS.showShareActionSheetWithOptions({
        url: uri,
        excludedActivityTypes: [
          'com.apple.UIKit.activity.PostToTwitter'
        ]
      },
      (error) => alert(error),
      (completed, method) => {
        var text;
        if (completed) {
          text = `Shared via ${method}`;
        } else {
          text = 'You didn\'t share';
        }
        this.setState({text});
      });
```
上述代码的第`7`行、`8~16`行，分别设置了两个 fail、success callback。

![](/img/RCTModuleMethodinvokeWithBridge.jpg)
上图所示是 JS to Native 中最终在 Native 侧调用相应方法的调用栈，在[ReactNative源码解析——通信机制详解(1/2)
](https://zxfcumtcs.github.io/2017/10/08/ReactNativeCommunicationMechanism/)中提到过，但限于篇幅没有展开讨论。

今天的分析就从`RCTModuleMethod#invokeWithBridge:module:arguments:`开始。
## RCTModuleMethod#invokeWithBridge:
```mm
- (id)invokeWithBridge:(RCTBridge *)bridge
                module:(id)module
             arguments:(NSArray *)arguments
{
    if (_argumentBlocks == nil) {
        [self processMethodSignature];
    }

    // Set arguments
    //
    NSUInteger index = 0;
    for (id json in arguments) {
        RCTArgumentBlock block = _argumentBlocks[index];
        if (!block(bridge, index, RCTNilIfNull(json))) {
            return nil;
        }
        index++;
    }

    // Invoke method
    //
    [_invocation invokeWithTarget:module];
    return nil;
}
```
在`invokeWithBridge:module:arguments:`内部(第`6`行)调用了`processMethodSignature`方法，从名称可知该方法是处理『方法签名』的(被处理的方法当然是被 JS 调用的 Native method 了)：
```mm
- (void)processMethodSignature
{
    NSArray<RCTMethodArgument *> *arguments;
    _selector = RCTParseMethodSignature(_methodInfo->objcName, &arguments);

    // Create method invocation
    NSMethodSignature *methodSignature =  [_moduleClass instanceMethodSignatureForSelector:_selector];
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
    invocation.selector = _selector;
    _invocation = invocation;

    // Process arguments
    //
    NSUInteger numberOfArguments = methodSignature.numberOfArguments;
    NSMutableArray<RCTArgumentBlock> *argumentBlocks =
    [[NSMutableArray alloc] initWithCapacity:numberOfArguments - 2];

    for (NSUInteger i = 2; i < numberOfArguments; i++) {
        const char *objcType = [methodSignature getArgumentTypeAtIndex:i];
        RCTMethodArgument *argument = arguments[i - 2];
        NSString *typeName = argument.type;
        SEL selector = RCTConvertSelectorForType(typeName);
        if ([RCTConvert respondsToSelector:selector]) {
            switch (objcType[0]) {
                case _C_CHR: {
                    char (*convert)(id, SEL, id) = (typeof(convert))objc_msgSend;
                    [argumentBlocks addObject:^(__unused RCTBridge *bridge, NSUInteger index, id json) {
                        char value = convert([RCTConvert class], selector, json);
                        [invocation setArgument:&value atIndex:(index) + 2];
                        return YES;}];
                        break;
                }
            }
        }
        else if ([typeName isEqualToString:@"RCTResponseSenderBlock"]) {
            [argumentBlocks addObject:^(__unused RCTBridge *bridge, NSUInteger index, id json) {
                void (^block)(NSArray *) = ^(NSArray *args) {
                    [bridge enqueueCallback:json args:args];
                };
                id value = json ? [block copy] : (id)^(__unused NSArray *_){};
                CFBridgingRetain(value);

                [invocation setArgument:&value atIndex:(index) + 2];
                return YES;
            }];
        } 
        // ...
    }
    _argumentBlocks = argumentBlocks;
}
```
`processMethodSignature`方法主要做的工作：
+ 从被调方法的名称(字符串)中解析出 selector 以及`RCTMethodArgument`格式的参数(第`4`行)；
+ 根据第一步中解析出的 selector 生成methodSignature、invocation(第`7~10`行)；
+ 为每个参数生成一个 block(`argumentBlock`)，并添加到`argumentBlocks`数组中：
	1\. 若参数的类型在 JS 与 Native 间可以转换(如：基础类型、字符串、数组等)(第`23~34`行)，在`argumentBlock`中完成 JS 类型参数 to Native 类型参数的转换(第`28`行)，并将得到的结果设置到`invocation`上(第`29`行)；
	2\. 若参数类型是 block(如：`RCTResponseSenderBlock`、`RCTResponseErrorBlock`等)(第`35~46`行)，则在`argumentBlock`中生成一个 bolck，用作调用方法时的实参(因为 block 类型无法从 JS 直接传给 Native)，在该 block 中调用了`bridge`的`enqueueCallback:args:`方法。

再回到`RCTModuleMethod#invokeWithBridge:module:arguments:`方法，第`12~18`行，执行了`processMethodSignature`方法为每个参数生成的 block，最终的效果就是将 JS 侧传入的参数值转换成 Native 类型并设置到`invocation`上。

通过上述分析可知，对于 block 类型的参数(callback)最终会调用了`RCTCxxBridge#enqueueCallback:args:`方法。
```mm
void JSCExecutor::invokeCallback(const double callbackId, const folly::dynamic& arguments) {
    auto result = [&] {
        try {
            return m_invokeCallbackAndReturnFlushedQueueJS->callAsFunction({
                Value::makeNumber(m_context, callbackId),
                Value::fromDynamic(m_context, std::move(arguments))
            });
        } catch (...) {}
    }();
    callNativeModules(std::move(result));
}
```
沿调用链最终来到`JSCExecutor::invokeCallback`，第`4`行通过 hook，实际调用的是 JS 侧的`MessageQueue#invokeCallbackAndReturnFlushedQueue`方法。
下面我们来看看 JS 侧如何处理 callback。
## NativeModules#genMethod
还记得`genMethod`方法吗？
```JS
function genMethod(moduleID: number, methodID: number, type: MethodType) {
    let fn = null;
    fn = function(...args: Array<any>) {
        const lastArg = args.length > 0 ? args[args.length - 1] : null;
        const secondLastArg = args.length > 1 ? args[args.length - 2] : null;
        const hasSuccessCallback = typeof lastArg === 'function';
        const hasErrorCallback = typeof secondLastArg === 'function';
        const onSuccess = hasSuccessCallback ? lastArg : null;
        const onFail = hasErrorCallback ? secondLastArg : null;
        const callbackCount = hasSuccessCallback + hasErrorCallback;
        args = args.slice(0, args.length - callbackCount);
        BatchedBridge.enqueueNativeCall(moduleID, methodID, args, onFail, onSuccess);
    };
    fn.type = type;
    return fn;
}
```
在[ReactNative源码解析——通信机制详解(1/2)
](https://zxfcumtcs.github.io/2017/10/08/ReactNativeCommunicationMechanism/)一文中介绍过，JS 在调用 Native 方法时，会在 JS 侧动态生成一个对应的 JS 方法。
在`genMethod`方法的第`4~9`行，处理了调用参数的最后2个，判断是否是 callback 类型(function 类型)。
从上述处理代码中可以得出：
+ 对于 callback 类型的参数，作了特殊处理，将其从参数列表中剥离出来；
+ 一个方法最多只能有两个 callback 类型的参数；
+ callback 类型的参数只能位于参数列表的最后。

下面再来看看`MessageQueue.enqueueNativeCall`：
```JS
enqueueNativeCall(
    moduleID: number,
    methodID: number,
    params: any[],
    onFail: ?Function,
    onSucc: ?Function,
  ) {
    if (onFail || onSucc) {
        // Encode callIDs into pairs of callback identifiers by shifting left and using the rightmost bit
        // to indicate fail (0) or success (1)
        // eslint-disable-next-line no-bitwise
        onFail && params.push(this._callID << 1);
        // eslint-disable-next-line no-bitwise
        onSucc && params.push((this._callID << 1) | 1);
        this._successCallbacks[this._callID] = onSucc;
        this._failureCallbacks[this._callID] = onFail;
    }
    this._callID++;
}
```
可以看到：
+ 对于 callback 类型的参数，在 JS 与 Native 间传递的是 callbackID(`_successCallbacks`、`_failureCallbacks`中的下标)；
+ JS 侧的 callback function 存储在`_successCallbacks`、`_failureCallbacks`中。

我们再回到调用流程中的`MessageQueue#invokeCallbackAndReturnFlushedQueue`方法，在该方法中调用了`__invokeCallback`方法：
```JS
__invokeCallback(cbID: number, args: any[]) {
    // The rightmost bit of cbID indicates fail (0) or success (1), the other bits are the callID shifted left.
    // eslint-disable-next-line no-bitwise
    const callID = cbID >>> 1;
    // eslint-disable-next-line no-bitwise
    const isSuccess = cbID & 1;
    const callback = isSuccess
      ? this._successCallbacks[callID]
      : this._failureCallbacks[callID];

    if (!callback) {
      return;
    }

    this._successCallbacks[callID] = this._failureCallbacks[callID] = null;
    callback(...args);
}
```
在`__invokeCallback`方法中，通过 cbID 在`_successCallbacks`、`_failureCallbacks`中找到相应的 callback function，并执行。至此，callback 的流程全部结束。
> RN 通过 callbackID 的最后一位是0还是1，确定callback 是 success 还是 fail。

## 小结
+ JS 与 Native 间传递的是 callbackID；
+ callback 参数只能位于方法参数列表的最后面并且最多只能有2个；
+ RN 通过 callbackID 二进制的最后一位是0还是1，确定是 success 还是 fail；
+ 由于 JS callback function 无法直接传递给 Native，Native 侧会生成一个 block。

# 线程模型
_______________________
在 RN 中，有3类线程需要关注：
+ JS Thread；
+ Native Module Thread；
+ UI Manager Thread(Shadow Thread)。

## JS Thread
JS Thread 是 JS 执行以及 JS 与 Native 通信线程。
简单讲，Native 在此线程执行 JS 代码，JS 调用 Native 接口也发生在此线程上。
JS Thread 的初始化发生在`RCTCxxBridge#start`方法中：
```mm
_jsThread = [[NSThread alloc] initWithTarget:[self class]
                                                   selector:@selector(runRunLoop)
                                                     object:nil];
_jsThread.name = RCTJSThreadName;
_jsThread.qualityOfService = NSOperationQualityOfServiceUserInteractive;
[_jsThread start];
```
在阅读 RN 源码时可能会发现`RCTMessageThread`类，它是对 JS Thread 的 C++封装。具体源码就不列了。
还会发现`RCTJSThread`变量：
```mm
dispatch_queue_t RCTJSThread;
RCTJSThread = (id)kCFNull;
```
> NOTE: RCTJSThread is not a real libdispatch queue

`RCTJSThread`的作用只是用于标识，确保需要在 JSThread 上执行的操作能在该线程上执行：
![](/img/RCTJSThread.png) ![](/img/dispatchJSThread.png)
## Native module Thread
JS 在调用 Native 方法时，Native 方法在哪个线程上执行？
Native Module 可以实现`methodQueue`方法，指定执行队列：`- (dispatch_queue_t)methodQueue`。
那如果 Native Module 没有实现`methodQueue`方法，会如何？
```mm
- (void)setUpMethodQueue
{
    BOOL implementsMethodQueue = [_instance respondsToSelector:@selector(methodQueue)];
    if (implementsMethodQueue && _bridge.valid) {
        _methodQueue = _instance.methodQueue;
    }
    if (!_methodQueue && _bridge.valid) {
        // Create new queue (store queueName, as it isn't retained by dispatch_queue)
        _queueName = [NSString stringWithFormat:@"com.facebook.react.%@Queue", self.name];
        _methodQueue = dispatch_queue_create(_queueName.UTF8String, DISPATCH_QUEUE_SERIAL);
    }
}
```
在`RCTModuleData#setUpMethodQueue`方法中可以看到：若Native Module 没有实现`methodQueue`方法，则为该 Native Module 生成一个串行队列。
那么在实现 `Native module#methodQueue` 方法时需要注意什么？
来了解一下 RN 自带的 module 实现情况：
+ main thread — 如 RCTActionSheetManager，在接口中有 UI 操作；
+ JSThread — 如 RCTTiming、RCTEventDispatcher，实时性要求较高的
(慎用，This can have serious implications for performance, so only use this if you're sure it's what you need)；
+ UI Manager thread — 如 RCTUIManager、RCTViewManager，UI 组件；
+ Custom thread — 如 RCTAsyncLocalStorage，耗时操作。

## UI Manager Thread(Shadow Thread)
UI Manager Thread，UI 组件(UI module)接口执行线程。
UI 不应该在 main thread？
RN 为了提高效率(如: 帧率)，会先在UI Manager Thread做一些预处理操作(如计算 frame)，最终在渲染上屏时会切到 main thread。
```mm
dispatch_queue_t RCTGetUIManagerQueue(void)
{
    static dispatch_queue_t shadowQueue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        if ([NSOperation instancesRespondToSelector:@selector(qualityOfService)]) {
            dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_USER_INTERACTIVE, 0);
            shadowQueue = dispatch_queue_create(RCTUIManagerQueueName, attr);
        }
        else {
            shadowQueue = dispatch_queue_create(RCTUIManagerQueueName, DISPATCH_QUEUE_SERIAL);
            dispatch_set_target_queue(shadowQueue, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0));
        }
    });
    return shadowQueue;
}
```
可以看到，UI Manager Thread 是一个高优先级的串行队列。

## 小结
+ 所有 JS 代码都会在独立线程 JSThread 上执行；
+ 可通过 methodQueue 方法自定义 Native module 执行线程；
+ 为了提高效率，所有 UI 组件都会在 UI Manager thread 上预处理，再在 main thread 上渲染上屏。
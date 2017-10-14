---
title: ReactNative源码解析——通信机制详解(1/2)
date: 2017-10-08 19:19:59
tags:
- 框架
- iOS
- ReactNative
---
本文通过分析源码，逐步解析 ReactNative 中 JS to Native 的通信机制。
<!--more-->
©原创文章，转载请注明出处！

# Overview
____________________________
本文详细分析了 JS to Native 的调用过程，包括：ReactNative 的初始化、native module 注册、JS 获取 native module 信息、JS 调用 native module。

> 本文分析的源码基于 ReactNative 0.47，为了叙述方便后文将 ReactNative 简称为 RN。
> 另外，为了精减篇幅聚焦重点，文中所列源码均做过简化，删除非关键代码。
> 由于本文具体分析的是 iOS 侧的实现，故后文所有内容都是针对 iOS 平台(其中有一大部分是共用的)。

# 准备工作
_______________________________
> 阅读源码大概有两类目的，一是想通过源码了解某个实现细节；二是想了解其实现的整体结构、原理。对于第一种情形可以直接深入源码，快速抓住关注点；而后者如果也一股脑扎入源码，很容易陷入各种实现细节而迷失方向(尤其是大型源码)。此时，应先了解代码整体结构，抓住关键路径，必要时辅以类图。

本文讨论的 RN 通信机制是整个 RN 的核心，贯穿始末，涉及`Objective-C`/`Java`、`C++`、`JavaScript`间的交互。因此，在深入分析前有必要先了解一下 RN 整体结构以及关键类的类图。
![](/img/RNLanguageComposition.jpeg)
RN 最大的优势在于***跨平台、热更新***。因此，RN 库本身也需要在多个平台(iOS、Android)上运行，从上图可知，在 RN 中 `C++`与`JS`部分的实现为多平台共用，在此基础上再分化为各平台实现。其中，iOS 平台特有的实现(`Objective-C`、`C++`)主要集中在 React 下，以`C++`语言实现的共有部分在 ReactCommon 下：
![](/img/ReactReactCommon.png)
JS侧与本文关系紧密的内容主要集中在：
![](/img/BatchedBridgePath.jpg)下的`BatchedBridge.js`、`MessageQueue.js`以及`NativeModules.js`中。

我们先大概了解几个关键类及其间的关系，对整体结构有个大致印象：
![](/img/RNClassDiagram.png)
如上图：
+ `RCTBridge`与`RCTCxxBridge`属于 iOS 平台特有，前者是 RN 对业务层接口(图中其他类都属于内部类，业务层无感知)，具体工作在其子类`RCTCxxBridge`中完成；
+ 整个 RN 的核心在跨平台的 C++层，其中很多类的功能从其名称即可略知一二，后文也会有详细的描述；
+ 在 RN 中肯定少不了 JS 的支持，从上图可知，JS 与 Native 的通信发生在 `JavaScript` 与 `C++`间，这也是本文分析的重点。

说到通信，无外乎 JS to Native、Native to JS，本篇我们重点分析JS to Native。
# JavaScript—>Native
_______________________________
通过 Apple 推出的 `JavaScriptCore`，要实现 JS to Native 的通信并非难事，主要有两条途径：
- `block`——在 RN 这样复杂的应用场景中`block`显得有些力不从心；
- `JSExport`协议——通过实现该协议可以向 JS 曝露 Native 接口(但不具备跨平台能力)。

因此，RN 并没有使用 `JavaScriptCore`提供的这两种方式，而是自己实现了一套通信机制。
我们先从一个简单的例子入手：`CalendarManager`封装了 iOS 平台的日历控件，供 JS 调用。
```mm
// CalendarManager.h
#import <React/RCTBridgeModule.h>

@interface CalendarManager : NSObject <RCTBridgeModule>
@end
```
```mm
// CalendarManager.m
@implementation CalendarManager

// To export a module named CalendarManager
RCT_EXPORT_MODULE();

RCT_EXPORT_METHOD(addEvent:(NSString *)name location:(NSString *)location) {
    RCTLogInfo(@"Pretending to create an event %@ at %@", name, location);
}
@end
```
我们知道，要将 Native module(类、接口)曝露给 JS，module需要实现`RCTBridgeModule`协议，并且在实现中要插入`RCT_EXPORT_MODULE`宏。具体曝露的方法也需要通过`RCT_EXPORT_METHOD`宏定义。
```js
// JS
import { NativeModules } from 'react-native';
var CalendarManager = NativeModules.CalendarManager;
CalendarManager.addEvent('Birthday Party', '4 Privet Drive, Surrey');
```
此时，在 JS 中就可以通过`NativeModules.CalendarManager.addEvent(...)`方式调用 Native 接口了。
下面将对这一过程逐一分析，先来了解上面提到的两个关键的宏：
## RCT_EXPORT_MODULE
```c
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }
```
可以看到，添加`RCT_EXPORT_MODULE`宏，相当定义了`load`、`moduleName`方法。正是在`load`方法中调用了`RCTRegisterModule`方法注册 Module(会影响 App 启动速度^_^)。
```mm
void RCTRegisterModule(Class moduleClass) {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        RCTModuleClasses = [NSMutableArray new];
    });
    // Register module
    [RCTModuleClasses addObject:moduleClass];
}
```
> `RCTRegisterModule`只是简单收集所有需要曝露给 JS 的类。

## RCT_EXPORT_METHOD
曝露给 JS 的接口需要通过`RCT_EXPORT_METHOD`宏来定义，上文中：
```mm
RCT_EXPORT_METHOD(addEvent:(NSString *)name location:(NSString *)location) {
    RCTLogInfo(@"Pretending to create an event %@ at %@", name, location);
}
```
最终展开后的样子(由于篇幅关系具体的展开过程不作描述)：
```mm
+ (NSArray *)__rct_export__531 {
    return @[@"", @"addEvent:(NSString *)name location:(NSString *)location", @NO];
}
- (void)addEvent:(NSString *)name location:(NSString *)location {
    RCTLogInfo(@"Pretending to create an event %@ at %@", name, location);
}
```
可以看到通过宏添加了一个新方法，其命名规则：`__rct_export__`+`__LINE__`+`__COUNTER__`。同时注意到，所有曝露给 JS 的方法返回值都是 `void`，要返回结果时，需通过 callback 方式实现。
> 优雅、巧妙地使用宏充满技巧。

上面介绍的`RCT_EXPORT_MODULE`以及`RCT_EXPORT_METHOD`宏属于编译阶段的处理，下面我们从执行角度一步一步进行分析。
![](/img/RNTimingDiagram.png)
上图是RN 初始化过程的时序图，我们将沿着图中的步骤逐步分析。
## 注册NativeModule
![](/img/RCTCxxBrigdeInitModulesWithDispatchGroup.png)
通过上述坐标，最终定位到`RCTCxxBridge._initModulesWithDispatchGroup`：
```mm
- (void)_initModulesWithDispatchGroup:(dispatch_group_t)dispatchGroup
{
    NSMutableArray<Class> *moduleClassesByID = [NSMutableArray new];
    NSMutableArray<RCTModuleData *> *moduleDataByID = [NSMutableArray new];
    NSMutableDictionary<NSString *, RCTModuleData *> *moduleDataByName = [NSMutableDictionary new];

    // Set up moduleData for automatically-exported modules
    for (Class moduleClass in RCTGetModuleClasses()) {
        NSString *moduleName = RCTBridgeModuleNameForClass(moduleClass);
        moduleData = [[RCTModuleData alloc] initWithModuleClass:moduleClass bridge:self];
       
        moduleDataByName[moduleName] = moduleData;
        [moduleClassesByID addObject:moduleClass];
        [moduleDataByID addObject:moduleData];       
    }

    // Store modules
    _moduleDataByID = [moduleDataByID copy];
    _moduleDataByName = [moduleDataByName copy];
    _moduleClassesByID = [moduleClassesByID copy];
}
```
+ 上述代码第`8`行`RCTGetModuleClasses()`即是获取通过`RCTRegisterModule`注册的 module 类(即所有曝露给 JS 的类)；
+ 通过`RCTRegisterModule`注册的 module 默认使用`init`方法进行初始化，若某个 module 的初始化需要参数，可通过`RCTBridgeDelegate`->`extraModulesForBridge`或`moduleProvider`提供已初始化的 module 实例。

至此，所有需要曝露给 JS 的 module 都已注册完成，并以`RCTModuleData`格式存储在`RCTCxxBridge`中。
> 大部分 module 都是懒加载，只有那些需要在主线程完成初始化以及有常量需要导出的 module才会在注册时实例化。

## Instance
![](/img/JSCExecutor.png)
坐标定位到`RCTCxxBridge._initializeBridge`方法中：
```mm
- (void)_initializeBridge:(std::shared_ptr<JSExecutorFactory>)executorFactory
{
    if (_reactInstance) {
        _reactInstance->initializeBridge(
          std::unique_ptr<RCTInstanceCallback>(new RCTInstanceCallback(self)),
          executorFactory,
          _jsMessageThread,
          [self _buildModuleRegistry]);
    }
}
```
`_initializeBridge`方法做的最重要的事情就是初始化`Instance`实例`_reactInstance`，此过程将所有曝露给 JS 的 module 由`RCTModuleData`格式转化为`ModuleRegistry`格式传入`Instance`。
```c
- (std::shared_ptr<ModuleRegistry>)_buildModuleRegistry
{
    auto registry = std::make_shared<ModuleRegistry>(createNativeModules(_moduleDataByID, self, _reactInstance));
    return registry;
}
```
此时，有必要介绍与 NativeModule 有关的几个类：
![](/img/NativeModuleClassDiagram.png)
+ `JSCNativeModules`——`C++`类，在`JSCExecutor`中使用；
+ `ModuleRegistry`——`C++`类，是`NativeModule`的集合；
+ `NativeModule`——`C++`抽象类，定义了与 NativeModule 有关的接口；
+ `RCTNativeModule`——实现了`NativeModule`中定义的接口；
+ `RCTModuleData`——`OC`类，是存储曝露的moudle 的数据结构；
+ `RCTBridgeMethod`——`OC`类，是存储曝露给 JS 接口(方法)的数据结构。

`Instance`是一个中转类，没有做太多的事情，我们继续向前。
## NativeToJsBridge
此时，来到` Instance::initializeBridge`->`NativeToJsBridge::NativeToJsBridge`，很明显`NativeToJsBridge`是 Native to JS 的桥接。
所有从 Native 到 JS 的调用都是从`NativeToJsBridge`中的接口发出去的。
在其构造函数中会初始化两个成员变量：
+ `m_executor`—`JSExecutor`类型的指针，从上文可知`JSExecutor`是个 `C++`抽象类，`m_executor`实际指向`JSCExecutor`的实例，作为 JS 的引擎，无论是 Native to JS 还是 JS to Native 最终都需要该类来处理，后面我们会逐一分析；
+ `m_delegate`—`JsToNativeBridge`类型的指针，顾名思义，JS to Native 的桥接，该成员变量仅用于初始化`JSCExecutor`实例。

## JSCExecutor
坐标定位到`NativeToJsBridge::NativeToJsBridge`->`JSCExecutor::JSCExecutor`：
```cpp
JSCExecutor::JSCExecutor(std::shared_ptr<ExecutorDelegate> delegate,
                         std::shared_ptr<MessageQueueThread> messageQueueThread,
                         const folly::dynamic& jscConfig) throw(JSException) :
                         m_delegate(delegate),
                         m_messageQueueThread(messageQueueThread),
                         m_nativeModules(delegate ? delegate->getModuleRegistry() : nullptr),
                         m_jscConfig(jscConfig) {
    initOnJSVMThread();
    installGlobalProxy(m_context, "nativeModuleProxy",
                       exceptionWrapMethod<&JSCExecutor::getNativeModule>());
}
```
`JSCExecutor`的构造函数做了一条非常重要的事情：***在 JS Context 中设置了一个全局代理`nativeModuleProxy`，其最终指向`JSCExecutor`类的`getNativeModule`方法***。
至于说`nativeModuleProxy`为什么非常重要，留个悬念，后文再解，我们先看看`installGlobalProxy`。
```cpp
void installGlobalProxy(
    JSGlobalContextRef ctx,
    const char* name,
    JSObjectGetPropertyCallback callback) {
  JSClassDefinition proxyClassDefintion = kJSClassDefinitionEmpty;
  proxyClassDefintion.attributes |= kJSClassAttributeNoAutomaticPrototype;
  proxyClassDefintion.getProperty = callback;

  const bool isCustomJSC = isCustomJSCPtr(ctx);
  JSClassRef proxyClass = JSC_JSClassCreate(isCustomJSC, &proxyClassDefintion);
  JSObjectRef proxyObj = JSC_JSObjectMake(ctx, proxyClass, nullptr);
  JSC_JSClassRelease(isCustomJSC, proxyClass);

  Object::getGlobalObject(ctx).setProperty(name, Value(ctx, proxyObj));
}
```
结合上面两段代码总结一下：
+ `nativeModuleProxy`在JS Context 中是一个具有`JSObjectGetPropertyCallback`属性的对象(object)；
+ `JSObjectGetPropertyCallback`非常神奇，其特点是在 JS 中访问对象属性(object.propertyName)时会触发 callback；
+ `nativeModuleProxy`对应的 callback 最终会调用`JSCExecutor::getNativeModule`。

在`JSCExecutor`构造函数中还调用了`initOnJSVMThread`方法：
```c
void JSCExecutor::initOnJSVMThread()  {
    installNativeHook<&JSCExecutor::nativeFlushQueueImmediate>("nativeFlushQueueImmediate");
}
```
`initOnJSVMThread`方法中有个值得关注的点：在 JS Context 中 hook 了`JSCExecutor::nativeFlushQueueImmediate`。
简单讲，hook 后，在 JS 中调用`global.nativeFlushQueueImmediate(...)`，实际调用 Native 的`JSCExecutor::nativeFlushQueueImmediate`方法。
另外，在构造函数中也将native module 注册信息转换为`JSCNativeModules`格式存储了下来。
## JsToNativeBridge
坐标定位到`NativeToJsBridge::NativeToJsBridge`->`JsToNativeBridge::JsToNativeBridge`：
```cpp
  JsToNativeBridge(std::shared_ptr<ModuleRegistry> registry,
                   std::shared_ptr<InstanceCallback> callback)
    : m_registry(registry)
    , m_callback(callback) {}
```
沿着调用链一路下来，最终 module 注册信息传入`JsToNativeBridge`，随后 JS to Native 的调用将用到这份信息。
至此，RN 初始化过程中与本文讨论的通信机制相关的内容基本结束。总结下来大概有几点：
+ 收集了所有曝露给 JS 的 module(也可称之为生成了一份 native module 注册表)；
+ 在 JS Context 中设置了`nativeModuleProxy`以及`nativeFlushQueueImmediate`；
+ 初始化了相关的类，如：`NativeToJsBridge`、`JsToNativeBridge`以及`JSCExecutor`等。

现在可以说是『万事具备，只欠东风』——由 JS 发起对 Native 的调用了。
下面我们沿着调用路径继续往下分析。
## JS NativeModules
> 这小节我们在 JS 中^-^

```js
// JS
import { NativeModules } from 'react-native';
var CalendarManager = NativeModules.CalendarManager;
CalendarManager.addEvent('Birthday Party', '4 Privet Drive, Surrey');
```
回到之前那个例子，其调用的是`CalendarManager`module 的`addEvent:location:`方法。
展开上面的调用，最终的形式是：
```js
NativeModules.CalendarManager.addEvent('Birthday Party', '4 Privet Drive, Surrey')
```
`NativeModules`定义在node_modules->react-native->Libraries->BatchedBridge->NativeModules.js。
```js
let NativeModules : {[moduleName: string]: Object} = {};
if (global.nativeModuleProxy) {
  NativeModules = global.nativeModuleProxy;
} 
```
`nativeModuleProxy`正是上文提到的，由 Native 注入 JS 的具有`JSObjectGetPropertyCallback`属性的 object。
因此，JS 中的`NativeModules.CalendarManager`(取属性值)等价于调用：
```cpp
JSCExecutor::getNativeModule(NativeModules, `CalendarManager`)
```
格式化：
JS中`NativeModules.moduleName` 等价于 Native 的：
```cpp
JSCExecutor::getNativeModule(NativeModules, `moduleName`)
```
也这是为什么说`nativeModuleProxy`非常重要的原因，所有从 JS to Native 的调用都需要其作为中间代理。
此时，又要切入 Native 环境了。
## ModuleRegistry::getConfig
由于接下来马上要用到 nativemodule 的信息，在此先提前准备好。
定位到`ModuleRegistry::getConfig`，通过上文可知在`ModuleRegistry`中存储了 nativemodule 的信息，
```cpp
folly::Optional<ModuleConfig> ModuleRegistry::getConfig(const std::string& name) {
    auto it = modulesByName_.find(name);
    NativeModule* module = modules_[it->second].get();

    // string name, object constants, array methodNames (methodId is index), [array promiseMethodIds], [array syncMethodIds]
    folly::dynamic config = folly::dynamic::array(name);
    std::vector<MethodDescriptor> methods = module->getMethods();
    for (auto& descriptor : methods) {
        methodNames.push_back(std::move(descriptor.name));
    }

    if (!methodNames.empty()) {
        config.push_back(std::move(methodNames));
    }
    return ModuleConfig({it->second, config});
}
```
`getConfig`方法最终将 native module 信息组装成一个数组，其格式：
[modulename, module 导出的 constants, [methodNames], [promiseMethodIds], [syncMethodIds]]
下面是`RCTWebSocketModule`导出的信息：
![](/img/NativeModuleConfig.jpg)

在这过程中`RCTModuleData.methods`起到关键作用：
ModuleRegistry::getConfig->RCTNativeModule::getMethods->RCTModuleData.methods
```mm
// RCTModuleData.m
- (NSArray<id<RCTBridgeMethod>> *)methods
{
    if (!_methods) {
        NSMutableArray<id<RCTBridgeMethod>> *moduleMethods = [NSMutableArray new];
        unsigned int methodCount;
        Class cls = _moduleClass;
        while (cls && cls != [NSObject class] && cls != [NSProxy class]) {
            Method *methods = class_copyMethodList(object_getClass(cls), &methodCount);

            for (unsigned int i = 0; i < methodCount; i++) {
                Method method = methods[i];
                SEL selector = method_getName(method);
                if ([NSStringFromSelector(selector) hasPrefix:@"__rct_export__"]) {
                    IMP imp = method_getImplementation(method);
                    NSArray *entries = ((NSArray *(*)(id, SEL))imp)(_moduleClass, selector);
                    id<RCTBridgeMethod> moduleMethod =
                    [[RCTModuleMethod alloc] initWithMethodSignature:entries[1]
                                                JSMethodName:entries[0]
                                                      isSync:((NSNumber *)entries[2]).boolValue
                                                 moduleClass:_moduleClass];

                    [moduleMethods addObject:moduleMethod];
                }
            }
        }
    }
    return _methods;
}
```
`__rct_export__`是不是很熟悉，`RCTModuleData.methods`会遍历所有以`__rct_export__`为前缀的方法并执行以导出曝露给 JS 的接口。

## JSCNativeModules::createModule
书接前文，JS to Native
NativeModules.moduleName(JS)->JSCExecutor::getNativeModule->JSCNativeModules::getModule->JSCNativeModules::createModule
沿调用链来到`JSCNativeModules::createModule`:
```cpp
folly::Optional<Object> JSCNativeModules::createModule(const std::string& name, JSContextRef context) {
  if (!m_genNativeModuleJS) {
    auto global = Object::getGlobalObject(context);
    m_genNativeModuleJS = global.getProperty("__fbGenNativeModule").asObject();
    m_genNativeModuleJS->makeProtected();
  }

  auto result = m_moduleRegistry->getConfig(name);
  Value moduleInfo = m_genNativeModuleJS->callAsFunction({
    Value::fromDynamic(context, result->config),
    Value::makeNumber(context, result->index)
  });

  folly::Optional<Object> module(moduleInfo.asObject().getProperty("module").asObject());

  return module;
}
```
 在`createModule`方法中，通过`ModuleRegistry::getConfig`(第`8`行)拿到了要调用的 native module 的信息(包括导出的常量、曝露的接口等)。
同时获取了 JS Context 中名为`__fbGenNativeModule`的属性(第`4`行)，从名称可知其作用是生成 JS 端的 Native Module 信息。
`__fbGenNativeModule`定义在`NativeModules.js`中
```js
// export this method as a global so we can call it from native
global.__fbGenNativeModule = genModule;
```
## NativeModules.genModule
NativeModules.moduleName(JS)->JSCExecutor::getNativeModule->JSCNativeModules::getModule->JSCNativeModules::createModule->NativeModules. genModule(JS)
```js
function genModule(config: ?ModuleConfig, moduleID: number): ?{name: string, module?: Object} {
    const [moduleName, constants, methods, promiseMethods, syncMethods] = config;

    const module = {};
    methods && methods.forEach((methodName, methodID) => {
        const methodType = isPromise ? 'promise' : isSync ? 'sync' : 'async';
        module[methodName] = genMethod(moduleID, methodID, methodType);
    });
    Object.assign(module, constants);
    return { name: moduleName, module };
}
```
`genModule`会遍历传入的`methods`数组，分别调用`genMethod`生成 JS 侧的方法：
```js
function genMethod(moduleID: number, methodID: number, type: MethodType) {
    let fn = null;
    fn = function(...args: Array<any>) {
        args = args.slice(0, args.length - callbackCount);
        BatchedBridge.enqueueNativeCall(moduleID, methodID, args, onFail, onSuccess);
    };
    fn.type = type;
    return fn;
}
```
`genMethod`生成的函数非常简单，只是将对 native 的调用信息入队。
至此，由`NativeModules.moduleName`发起的调用链终于结束，最终返回结果：以 `methodName` 为 key，`genMethod` 生成的 function 为 value 的 JS object。即`NativeModules.moduleName`等价于`{methodName: fn}`。
> JS 中的 moduleID、methodID 对应 native moudle注册表中的数组下标。

当然，事情并没有结束，继续扩展`NativeModules.moduleName.methodName(args)`。
通过上述分析，可知：
`NativeModules.moduleName.methodName(args)`等价于：
`BatchedBridge.enqueueNativeCall(moduleID, methodID, args, onFail, onSuccess)`
## MessageQueue.enqueueNativeCall
`NativeModules.moduleName.methodName`->`BatchedBridge.enqueueNativeCall`
```JS
const MessageQueue = require('MessageQueue');
const BatchedBridge = new MessageQueue();
```
可以看到`BatchedBridge`就是`MessageQueue`
```JS
  enqueueNativeCall(moduleID: number, methodID: number, params: Array<any>, onFail: ?Function, onSucc: ?Function) {
    this._queue[MODULE_IDS].push(moduleID);
    this._queue[METHOD_IDS].push(methodID);
    this._queue[PARAMS].push(params);

    const now = new Date().getTime();
    if (global.nativeFlushQueueImmediate &&
        (now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS ||
         this._inCall === 0)) {
        var queue = this._queue;
        this._queue = [[], [], [], this._callID];
        this._lastFlush = now;
        global.nativeFlushQueueImmediate(queue);
    }
}
```
在`enqueueNativeCall`方法中将调用信息(moduleID、methodID、params等入队)，若离上一次 flush queue 的时间超过5ms(`MIN_TIME_BETWEEN_FLUSHES_MS`)则立即 flush queue(出于性能考虑)。
对`nativeFlushQueueImmediate`是否还有影像？(在`JSCExecutor`构造函数中讲过了^_^)
`global.nativeFlushQueueImmediate(queue)`等价于`JSCExecutor::nativeFlushQueueImmediate(queue)`
## JsToNativeBridge::callNativeModules
`JSCExecutor::nativeFlushQueueImmediate`->`JSCExecutor::flushQueueImmediate`->`JsToNativeBridge::callNativeModules`
沿着调用链，定位到`JsToNativeBridge::callNativeModules`。
```cpp
void callNativeModules(
    JSExecutor& executor, folly::dynamic&& calls, bool isEndOfBatch) override {

    for (auto& call : parseMethodCalls(std::move(calls))) {
        m_registry->callNativeMethod(call.moduleId, call.methodId, std::move(call.arguments), call.callId);
    }
}
`callNativeModules`会逐个解析从 JS 传过来的 call queue 中的每个调用。
```cpp
void ModuleRegistry::callNativeMethod(unsigned int moduleId, unsigned int methodId, folly::dynamic&& params, int callId) {
    modules_[moduleId]->invoke(methodId, std::move(params), callId);
}
```
沿着调用链后面还有一些细节问题，由于篇幅关系，不再展开，主要如下：
![](/img/RCTNativeModuleinvoke.jpg)
![](/img/RCTModuleMethodinvokeWithBridge.jpg)
最终在`RCTModuleMethod.invokeWithBridge`中执行了调用`[_invocation invokeWithTarget:module];`
至此，从 JS to Native 的调用结束！
## 小结
JS to Native 的调用时序图：
![](/img/JSToNativeCallTimingDiagram.png)
整个过程大概分为两个阶段：
+ NativeModules.moduleName — 该过程主要是获取 native module 的信息(moduleID、methodID)，最终封装为 JS object ({methodName: fn})；
+ NativeModules.moduleName.methodName(params) — 执行调用。

今天就到这了，下篇我们将分析 Native to JS。
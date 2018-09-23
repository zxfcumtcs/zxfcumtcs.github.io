---
title: 简述 ReactNative Bundle
date: 2018-05-11 23:04:48
tags:
- 框架
- iOS
- ReactNative
---
本文因项目实际问题而起，简要分析了 RN Bundle 的结构。
<!--more-->
©原创文章，转载请注明出处！
# 引
__________________________________
原本计划在完成『[ReactNative源码解析——渲染机制详解](https://zxfcumtcs.github.io/2018/02/03/RNRendering/)』一文后，暂停 RN 相关的总结分享，谁料项目中通过RN分包同时加载两个业务 bundle 时出错了！索性对 RN Bundle 研究一番，遂总结出此文。
# Overview
__________________________________
使用 ReactNative 开发的业务，无论是通过内置还是动态下发的方式发布，都需要将业务 JavaScript 代码打包成 bundle。
JavaScript 作为一门静态脚本语言，为何需要打包这个过程？
打包主要有以下几个用途：
+ 开发 RN 业务时，一般会使用 JSX 语法『糖』描述 UI 视图，然而标准的 JS 引擎显然不支持 JSX，所以需要将 JSX 语法转换成标准的 JS 语法；
+  开发 RN 业务时，通常使用的是 ES 6，目前 iOS、Android 上的 JS 引擎还不支持 ES 6，因此需要转换；
+ JS 业务代码会依赖多个不同的模块(JS 文件)，RN 在打包时将所有依赖的模块打包到一个 bundle 文件中，较好地解决了这种复杂的依赖关系；
+ JS 代码的混淆。

> RN 打包过程中的转码主要依赖 [Babel](https://babeljs.io/) 实现。

# ReactNative Bundle
_______________________________________
RN Bundle 从本质上讲是一个 JS 文件，其主要由三部分组成：polyfills、module 定义、require 调用。
![](/img/RNBundle.png)

## Polyfills
polyfills 部分主要是在 JS context 中做一些准备工作，如：声明 ES 6 语法中新增接口、定义模块方法(如：模块声明方法`__d`、模块引用方法`require`等)、设置`global.__DEV__`变量等。
![](/img/Polyfills.jpeg)
如上图，polyfills 都闭包方法，定义的同时被调用。
polyfill具体规则定义在node_modules/metro-bundler/src/defaults.js中：
```JS
exports.polyfills = [
require.resolve('./Resolver/polyfills/Object.es6.js'),
require.resolve('./Resolver/polyfills/console.js'),
require.resolve('./Resolver/polyfills/error-guard.js'),
require.resolve('./Resolver/polyfills/Number.es6.js'),
require.resolve('./Resolver/polyfills/String.prototype.es6.js'),
require.resolve('./Resolver/polyfills/Array.prototype.es6.js'),
require.resolve('./Resolver/polyfills/Array.es6.js'),
require.resolve('./Resolver/polyfills/Object.es7.js'),
require.resolve('./Resolver/polyfills/babelHelpers.js')];
```
> defaults.js中还有其他有意思的信息^_^

## module definitions
为了更直观的了解 RN Bundle 中模块的定义，我们先来看一个例子：
![](/img/RNDemo.jpeg)
如上图一个非常简单的 RN Demo，在打包生成的 bundle 中变成如下的格式：
![](/img/RNMoudleBundle.jpeg)
很明显，为了看懂上图所示的打包结果，必须先了解一下`__d`为何物，细心的同学，可能已经在 polyfills 小节中发现了`__d`的定义(第`12`行)，即`__d`就是`define`方法，其完整的源码定义在：`node_modules/metro-bundler/src/Resolver/polyfills/require.js`中(代码略有删减)：
```JS node_modules/metro-bundler/src/Resolver/polyfills/require.js
global.require = require;
global.__d = define;

const modules = Object.create(null);

function define(factory, moduleId, dependencyMap) {
    if (moduleId in modules) {
        // that are already loaded
        return;
    }
    modules[moduleId] = {
        dependencyMap,
        exports: undefined,
        factory,
        hasError: false,
        isInitialized: false };
}
```
可以看到`define`方法前三个参数分别为：`factory` 方法、module ID以及dependencyMap。
**调用`define`方法定义模块，实质就是以 moduleID 为 key 向模块注册表(`modules`)中注册模块相关的信息(`exports`、模块`factory`方法、`isInitialized`等)。**
好了，下面我们再次回到 RN Bundle 中 module definition。
### module ID
在 RN 中，为了唯一标识每个模块，解决模块间的依赖问题，在打包生成 bundle 时，为每个 module 生成一个唯一的 moduleID，moudleID 为从0开始递增的数字。
另外，RN 在打包 bundle 时，按模块间依赖关系深度遍历(弦外之音就是，根组件的 moduleID 为0)。
### module factory
从上图可知，module factory 方法主要做了以下几件事：
+ **所有 `import` 转换为`require`方法调用(`import`是 ES 6新增语法，需要转换)(第`58`、`60`行)；**
+ **创建组件类(第`62~83`行)，其中最关键的方法就是`render`；**
+ **导出(exports)组件类(第`85`行)；**
+ **注册根组件(第`87~89`行)，详细信息请参看[ReactNative源码解析——渲染机制详解](https://zxfcumtcs.github.io/2018/02/03/RNRendering/)一文。**

> 通过 module factory 中的 `render`方法，再次看到 JSX 标签被转换成了`createElement`方法调用。

## require calls
RN Bundle 最后部分是 require calls：
```JS
;require(50);  // InitializeCore.js
;require(0);
```
`require`方法参数为 moduleID，RN Bundle 最后这两个`require`调用分别加载了`InitializeCore`以及`RNDemo`(根组件)模块。
下面我们来看看，`require`具体做了哪些事(与模块定义方法`define`定义在同一文件中)：
```JS node_modules/metro-bundler/src/Resolver/polyfills/require.js
function require(moduleId) {
    const module = modules[moduleId];
    return module && module.isInitialized ?
    module.exports :
    guardedLoadModule(moduleIdReallyIsNumber, module);
}

function guardedLoadModule(moduleId, module) {
    return loadModuleImplementation(moduleId, module);
}

function loadModuleImplementation(moduleId, module) {
    module.isInitialized = true;
    const exports = module.exports = {};
    var _module = module;
    const factory = _module.factory, dependencyMap = _module.dependencyMap;
    const moduleObject = { exports };
    factory(global, require, moduleObject, exports, dependencyMap);
    return module.exports = moduleObject.exports;
}
```
如上代码所示，`require`方法首先判断所要加载的模块是否已经存在并初始化完成。若是，则直接返回模块的`exports`，否则调用`guardedLoadModule`方法(最终调用的是`loadModuleImplementation`方法)。
`loadModuleImplementation`方法获得模块的`factory`方法并调用，最终返回模块的`exports`。
## 小结
从上文的分析可知，加载一个 RN Bundle，主要完成三件事：
+ 准备 JS 执行环境(polyfills)；
+ 定义所有需要的模块(module define)；
+ 加载`InitializeCore`以及根组件模块(require)。

通过上文分析，应该能清楚的区分模块定义与加载的关系：
+ 模块定义(define)：将模块相关的信息(其中最重要的就是`factory`方法)添加到模块注册表中，仅此而已；
+ 模块加载(require)：在 JS context 中调用模块`factory`方法，创建模块类并在组件注册表中注册根组件。

以盖房子作比喻：
+ 模块定义：将盖房子需要的材料运入场地；
+ 模块加载：真正地将房子盖起来。

ps：上文所示的 RN Bundle 都是开发环境下打出来的(打包命令中`--dev`为 true)，这样的 Bundle 是没有经过混淆的，其可读性较好。经过混淆的 Bundle，大概长这样：
```JS
!function(e){e.__DEV__=!1,e.__BUNDLE_START_TIME__=e.nativePerformanceNow?e.nativePerformanceNow():Date.now()}("undefined"!=typeof global?global:"undefined"!=typeof self?self:this);
!function(r){"use strict";function e(r,e,t){e in u||(u[e]={dependencyMap:t,exports:void 0,factory:r,hasError:!1,isInitialized:!1})}function t(r){var e=r,t=u[e];return t&&t.isInitialized?t.exports:i(e,t)}function i(e,t){if(!c&&r.ErrorUtils){c=!0;var i=void 0;try{i=n(e,t)}catch(e){r.ErrorUtils.reportFatalError(e)}return c=!1,i}return n(e,t)}function n(e,i){var n=r.nativeRequire;if(!i&&n&&(n(e),i=u[e]),!i)throw o(e);if(i.hasError)throw a(e,i.error);i.isInitialized=!0;var c=i.exports={},d=i,s=d.factory,f=d.dependencyMap;try{var l={exports:c};return s(r,t,l,c,f),i.factory=void 0,i.dependencyMap=void 0,i.exports=l.exports}catch(r){throw i.hasError=!0,i.error=r,i.isInitialized=!1,i.exports=void 0,r}}function o(r){var e='Requiring unknown module "'+r+'".';return Error(e)}function a(r,e){var t=r;return Error('Requiring module "'+t+'", which threw an exception: '+e)}r.require=t,r.__d=e;var u=Object.create(null),c=!1}("undefined"!=typeof global?global:"undefined"!=typeof self?self:this);
__d(function(e,t,r,l){Object.defineProperty(l,"__esModule",{value:!0});var n=t(12),s=babelHelpers.interopRequireDefault(n),a=t(24),o=function(e){function t(){return babelHelpers.classCallCheck(this,t),babelHelpers.possibleConstructorReturn(this,(t.__proto__||Object.getPrototypeOf(t)).apply(this,arguments))}return babelHelpers.inherits(t,e),babelHelpers.createClass(t,[{key:"render",value:function(){return s.default.createElement(a.View,{style:styles.container},s.default.createElement(a.Text,null,"This is a demo"))}}]),t}(s.default.Component);l.default=o,a.AppRegistry.registerComponent("RNDemo",function(){return o})},0);
```
# 问题
____________________________________
好了，本文既然是因问题而起，最后简要介绍一下遇到的问题：
![](/img/RNDemoError.jpeg)
问题的描述很清楚是根组件没有注册，由于刚刚分析完 RN 的渲染机制，知道这个错误描述的出处(来自`AppRegistry`模块的`runApplication`方法)：
![](/img/runApplicationError.jpeg)
问题也很明显是在组件注册表`runnables`中找不到要运行的根组件。
起初我们怀疑是因为两个业务 bundle 有冲突在加载时出错了。
但 debug 下来并没有出错提示。
最终请教相关人士，得知是两个业务 bundle ID 一样，导致第二个 bundle 没有被正确定义(`define`方法首先通过要定义的模块 ID 判断该 ID 是否已注册，若已注册直接返回)。
> 两个业务 bundle ID 会一样与我们使用的拆包方案有关。

问题清楚了，修改就简单了，在此不细述了。
> 通过这个问题，也说明了对底层实现方案了解的必要性。了解的越深，在遇到问题时思考的视角就会越广。


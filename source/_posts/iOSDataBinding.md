---
title: 简化 iOS Data Binding
date: 2016-11-05 18:55:46
tags:
- iOS
- Objective-C
- 构架
---
ZSKVOController 是一个基于 KVO 的 Data Binding 框架，本文简要介绍其实现原理。
<!--more-->
©原创文章，转载请注明出处！

# Overview
________________________________
在[《不谈框架，谈细节》](https://zxfcumtcs.github.io/2016/07/20/MobileArchitecture/)这篇文章中，简要分析了当前移动开发中几种常见的框架 ( MVC、MVP、MVVM 等)，其中数据绑定 ( Data Binding ) 是 MVVM 与其他 MV* 模式的一个显著区别。目前，iOS 中现实数据绑定主要有两种方式：
+ KVO；
+ 三方开源库 ReactiveCocoa。

其中，ReactiveCocoa 学习成本比较高，使用并不十分广泛。
在[《KVO漫谈》](https://zxfcumtcs.github.io/2015/09/18/KVO/)这篇文章中，我们分析了 KVO 实现的基本原理，也对 KVO 存在的问题作了简要的描述，其在使用中存在的问题主要有：
+ KeyPath 需要逐个注册，并且是字符串形式的硬编码，无法通过编译器检查错误；
+ KVO 回调都集中在`observeValueForKeyPath:ofObject:change:context:`方法中，通常使得该方法过于庞大；
+ observer 的注册与注销必须成对出现，否则极容易出现 crash。

作为一个优秀的框架或工具至少需要具备两点：
+ 能提升开发效率；
+ 不给使用者提供犯错误的机会。

因此，我们希望简化后的 Data Binding ( 基于 KVO ) 也具备这些优点，具体讲：
+ 简化注册过程；
+ 不同的 KeyPath 对应不同的回调方法；
+ 不需要手动注销 ( 使用者没有犯错的机会 )。

# ZSKVOController
__________________________________
ZSKVOController 是我们实现的基于 KVO 的 Data Binding 框架。
首先，我们看看 ZSKVOController 框架达到的效果：
假如：
在 ViewModel 中有属性：`title`、`desc`，
View 需要绑定`title`、`desc`，则在 View 中只需做两件事：
+ 绑定：将 ViewModel 绑定到 View 上`[_viewModel zs_addKVOObserver:self];`；
+ 实现方法：`zs_observeTitle:`、`zs_observeDesc:`。

如`zs_observeDesc:`的实现如下：
![](/img/zs_observeDesc.jpg)
是不是简单多了。

### 实现原理
___________________________________
我们知道，对于属性，可以重写 setter 方法，其 setter 方法名是通过一定的规则生成的：'set' + 属性名(大写首字母)
我们也可通过类似的方式生成 KeyPath 对应的 Data Binding 回调方法，如：'zs_observe' + KeyPath(大写首字母)。

其余的事情全由 ZSKVOController 框架完成，其结构如下所示：
![](/img/ZSKVOController.png)

核心思想：
+ 在将 observeder(ViewModel) 绑定到 observer(View) 上时 (即调用`zs_addKVOObserver:`方法)，动态遍历 observer 的实例方法，找到所有符合："zs_observe*:" 命名规则的方法，从中解析出需要绑定的 KeyPath；
+ 为了在 observer(View) dealloc 时不需要手动 remove，我们在 observer 上动态添加属性：ZSKVOController，由其注册对 observeder(ViewModel)的绑定，在其 `dealloc` 方法中 remove KVO；
+ 为了在 observeder(ViewModel) dealloc 时不需要手动 remove KVO，我们为 observeder 动态添加了一个监视其 dealloc 的『哨兵』属性：ZSObservederDeallocGuard，在其 dealloc 方法中 remove 所有 KVO。

具体实现细节，请参看源码：[ZSKVOController](https://github.com/zxfcumtcs/ZSKVOController)

# 小结
____________________________________
ZSKVOController 内部通过 KVO 实现 Data Binding，是对 KVO 的一次封装，从注册到数据变化的处理都有较大的简化，同时，无需手动 remove KVO。
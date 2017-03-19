---
title: 自定义 UI 组件库
date: 2017-03-04 22:34:44
tags:
- 构架
---
本文简单介绍了通过 View-ViewModel 模式构建 UI 组件，以便提高 UI 组件的可复用性。
<!--more-->
©原创文章，转载请注明出处！

# Overview
________________________________

『不要重复造轮子，要注意代码复用，提高开发效率』，这是在日常开发中经常挂在嘴边的话。对 copy code 的行为我们经常也报以鄙视的目光。
但是，copy 行为却又时常发生，经常是一言不合就 copy！

在继续之前，我们有必要思考两个问题：
+ 什么样的代码被复用的概率最高；
+ copy code 的行为为何如此普遍。

### 什么样的代码被复用的概率最高
代码复用可分为二个层级：APP 间以及 APP 内部。
APP 间被复用的代码通常是些基础组件，如：网络、数据库、日志系统、磁盘存储系统等。
APP 内部复用概率最高的代码个人认为是 UI 组件（本文所说的 UI 组件为 APP 内部自定义的 UI）。

### copy code 的行为为何经常发生
既然大家都会鄙视 copy 行为，为何又时常发生呢？
你可能会说 copy 的同学觉悟、追求不高。
但，从另一角度是否也说明代码的可复用性、可扩展性比较差。
现在一般都是敏捷开发、快速迭代，时间紧、任务重，在遇到相似、相近的代码时，大家通常都是拷一份过来，改吧改吧，而不是在原有基础上费时费力地扩展。这其中的弊端就不用多说了！

UI 组件作为 APP 内部复用概率最高的代码，更是 copy 的重灾区。究其原因，在于在 UI 组件中包含了大量的业务逻辑，导致一个 UI 组件要复用到另外一个业务下非常困难。主要体现在：
+ UI 组件与业务数据相绑定（与后台协议返回的数据结构相对应的 OC 对象，我们称之为 item）；
+ UI 组件内部处理了业务逻辑。

举个例子，在 QQ 阅读的信息流中有一种常见的 UI 样式：左图右文(QRFeedFlowBookView)
![](/img/FeedFLowLeftPictureRightText.jpg)
在当初实现该 UI 样式时，数据是通过 QRFeedFlowBookItem(业务数据结构) 传递给 QRFeedFlowBookView，即在 QRFeedFlowBookView 内部处理了具体的业务数据。这导致，QRFeedFlowBookView 在业务数据不是或不能通过 QRFeedFlowBookItem 表示时，无法复用。
还有，像一些角标要不要展现，具体逻辑也是在 QRFeedFlowBookView 中处理的，此时如各个业务对此的逻辑不同，QRFeedFlowBookView 也无法复用。


# 以 View-ViewModel 形式构建 UI 组件
_________________________________________
从上面分析出的 UI 组件难于复用的原因，可以看出要解决这一问题，就是要避免上文提到的2点：
+ UI 组件不能与具体业务数据相绑定；
+ UI 组件内部不能处理业务逻辑（其本职工作仅是 UI 布局）。

此时，MVVM 模式应该要进入我们的视线了，在该模式中 ViewModel 的存在是不是很好的解决了上面的问题。
我们知道在 MVVM 模式中，ViewModel 向上为 View 提供展示数据（该数据已经在 ViewModel 中处理好了，View 无需任何处理，只要展示即可），向下接收来自业务层的数据，处理相关的业务逻辑。

可以看出，ViewModel 作为中间层很好地将业务与 UI 隔离开。
说到 MVVM，很多同学并不喜欢，觉得其中的 Data-Binding 很麻烦，但我们构建 UI 组件时用到的是 View-ViewModel 结构，并不要求一定是 MVVM，在 MVC 等模式下也可使用。

同时，我们采用的是面向接口的模式，View 对外依赖的是接口（protocol），而不是某个具体的 ViewModel。每个 UI 组件其结构如下所示：
![](/img/ViewViewModel.png)
如上图所示，若某个 UI 组件被多个业务所复用，可以根据需求定义多个 ViewModel 以处理不同的业务逻辑，每个 ViewModel 都实现 ViewModelProtocol 协议为 View 提供数据。

另外，UI 组件也可以包含(复用)其他 UI 组件。

如上文提到的左图右文 UI，我们抽取为一个组件 QRLeftPictureRightTextView：
![](/img/QRLeftPictureRightTextView.png)![](/img/QRLeftPictureRightTextViewModel.png)
该控件在信息流以及老书城都有用到，为此我们也定义了两个 ViewModel，以处理各自的业务逻辑：
![](/img/QRLeftPictureRightText.jpg)

在日常开发中，经常有同学问『某某样子的 UI 我们现在有吗，是哪个类？』
为此，我们在构建 UI 组件库的同时也维护了一份文档，将所有组件都『登记造册』：
![](/img/QRLeftPictureRigthTextDoc.png)

# 小结
____________________________________________
通过 ViewModel 这个中间层很好地隔离了 UI 与业务逻辑，使 UI 复用性更好，不仅提高了开发效率，也规范了代码结构。
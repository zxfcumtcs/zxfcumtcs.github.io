---
title: GUI 架构简述
date: 2016-07-20 22:03:26
tags:
- 架构
- GUI
- iOS
---
本文从细节入手，尝试分析了几种常见的 GUI 架构：MVC、MVCS、MVP、MVVM。对于在实际开发中如何选择给出了一些参考意见。
<!--more-->
©原创文章，转载请注明出处！

# Overview
____________________________
移动开发架构，无论是 iOS、Andriod 还是 Web 都属于 GUI (Graphical User Interfaces) 架构范畴。2006年，Martin Fowler 的 [GUI Architectures](http://martinfowler.com/eaaDev/uiArchs.html) 一文可谓是经典之作。文中 Martin Fowler 提到 MVC 模式如何组织代码、划分模块职责，还提到 [Data Binding](http://martinfowler.com/eaaDev/DataBinding.html)、[Flow Synchronization](http://martinfowler.com/eaaDev/FlowSynchronization.html) 以及 [Observer Synchronization](http://www.martinfowler.com/eaaDev/MediatedSynchronization.html) 等核心概念。
纵观十年来 GUI 架构演变，无论是 MVCS、MVP 还是 MVVM，其实讨论的核心问题还是如何分层、如何划分模块职责、做好代码隔离。

谈到 iOS 上常见架构，相信只要有半年以上开发经验的同学都能侃侃而谈。但在实际交流过程中发现不少同学对关键细节问题却认知模糊，甚至是错误的。因此，本文尝试从细节入手对几种常见架构进行简单描述(对架构的认识智者见智、仁者见仁，我所描述的也不一定是正确的)。

# 为什么要分层
____________________________
上文提到各种架构虽各种不同，但它们其实都是在讨论一个问题：『如何分层』。
那么在继续之前，我们有必要思考一下：***为什么要分层？***

计算机界有一句大道至简的名言：
> All problems in computer science can be solved by another level of indirection.

之所以要分层，最终目的是降低系统整体复杂度。
通过分层我们至少能获得以下能力：
+ 提供良好的抽象，隐藏实现细节，降低耦合度；
+ 隔离变化；
+ 提高模块可复用性；
+ 增强系统可扩展性。

说到分层，可能最先想到的例子是 OSI 七层或 TCP/IP 五层网络模型：![](/img/OSI.jpg)
在分层网络模型中，不同协议工作在不同网络层，互不干扰，又协调有致，如：IP 协议工作在网络层，主要职责是网络寻址；TCP 协议工作在传输层，主要负责建立可靠的网络连接、负责拥塞控制等。正是因为良好的分层，IP 协议无需关心 TCP 协议的工作，反之亦然。同时 IP 协议也可以在 TCP、UDP 协议间复用。

如今对网络安全越来越重视，HTTPS 协议也慢慢普及，通过分层，只需在原有 HTTP 协议基础上添加一个 TLS/SSL 的安全传输层即可，由它来负责加解密，而原有的 HTTP、TCP 协议无需修改：![](/img/HTTPHTTPS.png)

# Model-View-Controller
_____________________________
MVC(Model-View-Controller)作为最经典的架构，广为人熟知，也是 Apple 官方推荐的移动架构。![](/img/MVC-Apple.jpg)
MVC模式的核心思想是数据层(Domain)与表现层(Presentation)的隔离。
> [Separated Presentation:](http://www.martinfowler.com/eaaDev/uiArchs.html)
> Ensure that any code that manipulates presentation only manipulates presentation, pushing all domain and data source logic into clearly separated areas of the program.

![](/img/DomainPresentation.png)
那么，在数据与展现被隔离之后，它们之间如何同步数据、状态？
这就涉及 MVC 模式另一个重要思想：观察者同步(Observer Synchronization)。
> [Observer Synchronization:](http://www.martinfowler.com/eaaDev/MediatedSynchronization.html)
> Synchronize multiple screens by having them all be observers to a shared area of domain data.

常用方法：在Presentation Object(Controller)中注册通知、设置delegate、传递block等。当数据需要更新时，Domain Object(Model)通过上述方式将数据自底向上的同步给Presentation Object。

下面简单介绍一下 Model、View、Controller：
## Model
_________________________
Apple: Model Objects Encapsulate Data and Basic Behaviors.
Stanford: Model = What your application is (but not how it is displayed).
简单讲：Model = Data + Manipulate Data

(ps:本文中的Stanford表示斯坦福的 iOS 公开课)
![](/img/QRBookShelfModel.png)
如：我们书架的 Model：`QRBookShelfModel`，包含了数据：`NSArray<QRBookShelfItem *> *books`以及对数据的操作：`addBook:`、`deleteBook:`等。

## Controller
_________________________
Apple: Controller Objects Tie the Model to the View.
Stanford: Controller = How your Model is presented to the user(UI logic).

Controller 是 Model 与 View 间的连接器，其核心职责有：
+ 处理用户事件；
+ 处理展示逻辑；
+ 连接 Model 与 View。

这里有个问题：到底什么是展示逻辑？
简单讲：将业务数据转换成UI数据，如：
+ 下载进度，从 Model 层返回的是 double 型，将其转换成可展示的 string 类型(0.811—>81.1%)；
+ 性别，从 Model 返回的是0、1这样的 int 型，将其转换成：1->男，0—>女；
+ 日期，将时间戳格式化：123456789923—>2016-07-01 10:09.

## View
_________________________
Apple: View Objects Present Information to the User
Stanford: View=Your Controller’s minions

总之，View 只做一件事：layout。

## MVC 规则
_________________________
为了实现 MVC 的核心思想：业务 (model) 与展示 (View) 的隔离，必须严格遵守一些规则：
+ Controller 依赖(持有) Model、View(可直接与它们通信)；
+ Model 与 View 互不可见(不可通信)；
+ View 只负责layout，且不能保存业务数据 (需要数据时通过 datasource 方式向 Controller 要)；
+ View 可通过 target、delegate 与 Controller 同步状态；
+ Model 不能主动与 Controller 通信，通过 Notification、KVO、delegate、block 等机制通知 Controller 数据变化。

看到这里，大家有没有一种熟悉的味道？
没错，UITableView 与外界 (Contoller) 的交互与此处的描述高度一致。
![](/img/MVC.jpg)

(关于 MVC 规则的描述，大家也可以参考 Stanford iOS 公开课中的相关内容)

## 有问题吗？
___________________________
此时，大家或许心中有些疑问：
1、在 MVC 模式中，网络请求、数据存储谁来完成？
2、Model、View、Controller 谁的可复用性最强？
3、展示逻辑为什么由 Controller 完成而不是 View？

+ 在 MVC 模式中，网络请求、数据存储谁来完成？
	Controller、Model 都可以，一般由 Model 完成。此时的 Model 已不再是简单的 Model Object，而是 Model layer，在我们项目中通常将其称为 Manager。

+ Model、View、Controller 谁的可复用性最强？
	View>Model>Controller

+ 展示逻辑为什么由 Controller 完成而不是 View？
	View可复用性高，不应关心具体展示逻辑，只专注于 layout
	
## Massive View Controller
_______________________________
MVC 模式被批评最多的就是 Controller 过于臃肿，那么 Controller 都做了什么？
+ 处理复杂的展示逻辑；
+ 处理用户事件；
+ 初始化 View、管理部分 View 的生命周期并提供数据；
+ 处理业务数据变化，转换为 UI 结果；
+ 获取、存储数据(可选)。

尤其是如今很多产品经理『擅长』做加法，页面、交互越来越复杂，这对于 Controller 来说无疑是雪上加霜。

# Model-View-Controller-Store
________________________________
前面提到，在 MVC 模式中，并没有讨论获取数据属于哪个模块的职责 (一般由 model 负责)。MVCS 模式就是在 MVC 基础上将数据单独提取为一层(Store)。
![](/img/MVCS.png)

# Model-View-Presenter
________________________________
在 MVC 模式中，展示逻辑被划分为 Controller 的职责范围。如今，展示逻辑越来越复杂，Controller 随之也变得越来越臃肿。同时，Controller 也被认为是 View 的一部分，这样 Model 与 View 间并没有完全隔离、解耦。
MVP (Model-View-Presenter) 就是在这样的背景下产生的，其将展示逻辑提取为一个单独的层(Presenter)，简化了 Controller，也彻底隔离了 Model 与 View。
![](/img/MVP.png)

新产生的 Presenter 层有以下特点：
+ UI 无关 (在 Presenter 中不能包含 UIKit 相关头文件)；
+ 处理展示逻辑；
+ Model 与 View 间的桥接者。

# Model View View-Model
___________________________________
最近两三年对 MVVM(Model View View-Model) 的讨论比较多，其提出的愿景也是为了简化 Controller、彻底将 View 与 Model 解耦、并提供 Data Binding。
![](/img/MVCMVVM.jpg)
在 MVVM 中 Controller 被认为是 View，更准确的说是：
![](/img/MVMCV.jpg)
## Rules
___________________________________
在 MVVM 模式中，各模块间的依赖关系、数据流向、数据传递的格式都有严格的规定：
![](/img/MVVMRules.jpg)
如上图所示，View、View Model 以及 Model 需要遵守以下规则：
+ View(UIViewController/UIView):
	1.可以依赖(持有) View、View Model，即可直接调用其方法；
	2.不能依赖(持有) Model、Model Object(Item)；
	3.UI 绑定到View Model上(如：titleLabel.text->viewModel.title)
	4.通过 RACCommands/RACActions 或直接调用 View Model 的方法将用户事件传递给 View Model。

+ View Model:
	1.可以依赖(持有) View Model、Model 以及 Model Object；
	2.不能依赖(持有) View、Raw Model Object；
	3.其公开属性只能是基础数据类型(NSInteger、NSString等)或其他 View Model；
	4.将 Model Object 转换成可直接在 View 上显示的属性或Sub View Model(展示逻辑)；
	5.接受来自 View 或 Sub View Model 的输入(用户事件)。
	
+ Model (Layer)：
	1.可以依赖(持有) 其他 Model、Model Object、Data Source、Raw Model Object；
	2.不能依赖(持有) View Model、View；
	3.将 Raw Model Object 转换为 Model Object；
	4.为 View Model 提供数据(异步)。
	
其中，View 与 View Model 类似 UIView 与 CALayer 的关系，一一对应(包括层次结构)：
![](/img/ViewViewModel.jpg)
## Data Binding
____________________________________
![](/img/MVVMDataBinding.jpg)
从上图可知，在 MVVM 中数据流方向与依赖关系正好相反，数据流的流动就是建立在 Observer Synchronization 思想基础之上。

从 Data Source 到 Model、Model 到 View Model 可采用一般的同步方法，如：Delegate、Notification 以及 block 等。而从 ViewModel 到 View 的 Data Binding 是 MVVM 模式与其他 MV* 模式最大的区别。

遗憾的是 iOS 并没有原生的 Data Binding 方式，目前大概只能通过两种方式实现 Data Binding：KVO 或 ReactiveCocoa。KVO/RAC是一种更加激进的 Observer Synchronization：
+ 优势：绑定关系确定后，同步更加方便；
+ 劣势：数据流不直观，调试较困难；

## MVVM VS. MVP
_____________________________________
MVVM 与 MVP 有很多相似的点：
+ 将展示逻辑从 Controller 中提取出来(分别放到 View Model 和 Presenter 中)；
+ 分别在 View Model、Presenter 中响应用户事件；
+ 分别通过 View Model、Presenter 连接 View 与 Model；
+ 解耦 View 与 Model。

两者最大的区别在于：MVVM 有 Data Binding 而 MVP 没有。

# 华山论剑——MV* VS. MVVM
______________________________________
根据是否有 Data Binding，可将常见 GUI 构架分为两大阵营：
+ 有 Data Binding：MVVM；
+ 没有 Data Binding：MVC、MVP、MVCS 等。

MVVM 的 Data Binding 在一定程度上增加了编码的复杂度、数据流也变得不够直观、调试难度也有所增加。但对于数据可变的场景，一旦通过 Data Binding 将 View 与 View Model 绑定起来，在数据变化时，会自动映射到 UI 上，十分方便。

根据展示逻辑是否独立于 Controller，可分为：
+ 独立：MVVM、MVP
+ 不独立：MVC、MVCS

MVVM、MVP 分别将展示逻辑从 Controller 中提取出来，使 Controller 得到一定程度的简化，在展示逻辑复杂的情况下，效果更加明显。
## 没有好坏，只有适合
________________________________________
通过上述分析，我们可以看到，常见几种架构：MVC、MVCS、MVP、MVVM 并没有绝对的好坏之分，只是各有不同的适用场景。
我们在选择时可以根据以下两点作为参考依据：
+ 数据是否可变(UI是动态还是静态)
	静态：MV*
	动态：MVVM
+ 展示逻辑是否复杂
	复杂：MVVM、MVP

## 万变不离其宗——MVC是根
________________________________________
MVCS、MVP、MVVM 等各种新生架构，虽各有不同，但都是源自于 MVC，它们的核心思想一直没变，也不能变：
+ Separated Presentation;
+ Observer Synchronization.

# 实例
________________________________________
理论的东西讲了不少，下面结合实际的项目，看看应该如何选择架构。

## 书籍详情页
__________________________
书籍详情页在整个 QQ 阅读 app 中，无论是展示还是业务逻辑都是最复杂的一个模块。
![](/img/DetailPage.jpg)
+ 在书籍下载过程中下载按钮需要显示下载状态(进度)——动态 UI；
+ 评分、作者分类、包月相关提示、打赏、粉丝榜等展示逻辑十分复杂。

因此，该模块采用 MVVM 架构比较合适。遗憾的是，当时设计该模块时没有充分意识到其复杂程序，而是选择了传统的 MVC 架构。结果造成详情页 Controller 十分复杂，下面这段就是 Controller 中根据下载状态修改 toolbar 上3个按钮状态的代码(ps：看不清没关系，只要能看出其很复杂即可^_^)：
![](/img/DetailPageChangeBookInfo.jpg)
同时，大量的展示逻辑也耦合在了各个 View 中：
![](/img/BookDetailHeaderView.jpg)
如果，采用 MVVM 架构，各种展示逻辑可以放到相应的 View Model 中，让 View 只专注于 layout。同时在 View Model 中处理下载相关逻辑，使下载逻辑与 View 解耦。

## 信息流
__________________________
信息流作为 QQ 阅读一大亮点，能为用户个性化推荐书籍，是整个 app 中最重要的一个页面：
![](/img/FeedFlow.jpg)
信息流是最重要的页面，但不是最复杂的页面：
+ 信息流的数据是静态的，在显示过程中不会改变——静态UI；
+ 展示逻辑相对简单。

因此，信息流模块没必要使用复杂的 MVVM 架构。
***很多场景属于此类情形：通过 UITableView 列举多行静态数据，若数据有更新时直接 reload tableview。***

# 无剑胜有剑 皆可为剑
___________________________
各种分层架构都是前辈充满智慧的宝贵经验，值得尊敬、借鉴、学习，但也不必拘泥于形式，重点是理解其背后的思想。设计模式有六大原则：
+ 单一职责原则
+ 里氏代换原则
+ 依赖倒转原则
+ 接口隔离原则
+ 迪米特法则
+ 开放-封闭原则

其中除了里氏代换原则，其他五大原则都是分层架构的指导思想。只要我们深刻理解并能严格遵守这些原则，无论我们选择哪种架构、或在其基础上进行衍化，都能设计出高质量的代码。

# 小结
___________________________
代码设计、架构选择及理解仁者见仁、智者见智，但经典的设计理念是公认的、也是经过时间检验的。

# 参考资料
[GUI Architectures](http://martinfowler.com/eaaDev/uiArchs.html)
[Model-View-Controller](https://developer.apple.com/library/ios/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html)
[Introduction to MVVM](https://www.objc.io/issues/13-architecture/mvvm/)
[Lighter View Controllers](https://www.objc.io/issues/1-view-controllers/lighter-view-controllers/)
[ReactiveCocoa and MVVM, an Introduction](http://www.sprynthesis.com/2014/12/06/reactivecocoa-mvvm-introduction/)
[On MVVM, and Architecture Questions](http://twocentstudios.com/2014/06/08/on-mvvm-and-architecture-questions/)
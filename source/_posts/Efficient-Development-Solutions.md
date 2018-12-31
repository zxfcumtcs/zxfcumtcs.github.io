---
layout: post
title: iOS 高效开发解决方案
date: 2018-12-22 17:36:54
tags:
- 架构
- iOS
- UI 组件
---
本文作为 QQ 阅读 7.0 改版总结，从架构、页面元素模块化、UI 组件化、基于 iOS 系统响应链的事件处理、业务模板化等方面阐述了一套高效的列表类应用开发解决方案。
<!--more-->
©原创文章，转载请注明出处！
# 概述
_____________________
QQ 阅读迎来了7.0版本，作为惯例大版本需要大动作——『UI大改版』。
本文主要是对这次改版的一个总结并提炼出一套通用的『列表类业务』开发解决方案。
本文将从以下几个方面展开讨论：
+ 架构
+ 页面元素模块化
+ UI 组件化
+ 基于响应链的事件处理
+ 业务模板化

> 本文部分内容来自[列表类应用场景模板化](https://zxfcumtcs.github.io/2018/09/17/ListSceneTemplating/)和[自定义 UI 组件库](https://zxfcumtcs.github.io/2017/03/04/CustomUIControls/)

# 架构
_____________________
列表类业务应该说是大多数 App 的主要业务场景，如朋友圈、新闻类 App 首页、各类个性化推荐页、微博首页以及我们的书城等等。

列表类业务其流程主要是：
+ 从网络或本地磁盘获取数据；
+ 再将数据以列表(`UITableView`、`UICollectionView`)形式展示出来；
+ 最主要的交互就是点击进入次级页面。

对于列表类业务每个项目团队可能都有一套架构，在 QQ 阅读不断迭代的过程中也演化出一套架构。
![](/img/ListSceneClassDiagram.png)
![](/img/ListSceneTimingDiagram.png)
上面分别是我们这套架构的关键类图和时序图。整体上是由经典 MVC 模式演化而来：
+ Manager(Interface)：对应 MVC 中的 Model 『层』，主要负责数据的获取、管理等业务逻辑；
+ Controller：各个模块的协调枢纽，页面的承载主体；
+ Cell\View：对应 MVC 中的 View，仅仅负责 UI 布局、展示逻辑；
+ ViewModel(Interface)：View 与具体业务的中间抽象层，使两者解耦，达到 View 只负责 UI 布局的目的，最终实现 View 的高可复用性；
+ Module(Interface)： 称其为『业务模块』，一个页面由多个不同或相同类型的模块组成。

# 页面元素模块化
_____________________
MVC 模式饱受诟病的一点就是：Controller 经常会变得过于臃肿(Massive View Controller)。
为了解决这一问题，业界提出了多种解决方案，大部分都是通过添加中间层，将 Controller 的功能分解到中间层上，如 MVP (Model View Presenter) 模式。

为了解决 Controller 臃肿问题，在我们的架构中将页面元素抽象成一个个的 Module。
![](/img/bookcitymodules.png)
如上图，红色虚线分隔的就是不同的 Module。从此，页面的生成过程就是拼接组装 Module 的过程。

在 TableView 中一个 Module 对应一个 section。
Module 的职责主要有：
+ 解析、存储业务数据(如今日必读 Module 需要负责解析、存储今日必读这块业务数据)；
+ 为 TableView 提供数据(即实现`UITableViewDataSource`协议)；
+ 处理用户事件；
+ 埋点；
+ ...

——即负责『模块』的所有逻辑(与 React Component 类似)。

## Manager 与 Module
通过上述分析可知，Module 解析、存储业务数据，Manager 存储、管理 Module。

这种做法也存在弊端，由于将解析业务数据、控制 UI 展示的逻辑(创建 cell 等)都放在了 Module 中。使得 Module 违反了『单一职责原则』。
> 『单一职责原则』(SRP)作为面向对象设计的五大原则『SOLID』之一，很容易理解，也很难把握！『就好像生活中的各种"适量"，适量放点盐、适量加点水…』
 Bob大叔在《敏捷软件开发》中，将类的单一职责原则描述为『应该仅有一个引起它变化的原因』。

在 Module 中，业务数据解析、UI 展示就是两个可变的因素——『同样的 UI 用于展示不同的网络协议返回的数据、同一协议返回的数据展示为不用的 UI』。
在 QQ 阅读中，书籍列表页就属于『同样的 UI 展示不同协议返回的数据』：
![](/img/BookList.jpeg)
针对这种情况，无非就是将其中一个变化因子抽取出来，如将业务数据解析抽取为一个单独的类。
由于 Module 中这两个变化因子变动的概率并不大，为了降低复杂度，只有在真正需要时才将这两者分离开。
>『敏捷开发』的原则之一就是尽量保持代码简单、并在必要时进行重构，防止代码变坏。

## 效果
```mm
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
{
    QRBaseModule *module = [self.manager moduleAtIndex:indexPath.section];
    return [module heightForRow:indexPath.row];
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
    QRBaseModule *module = [self.manager moduleAtIndex:section];
    return [module numberOfRows];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    QRBaseModule *module = [self.manager moduleAtIndex:indexPath.section];
    return [module cellForRow:indexPath.row tableView:tableView];
}

- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView
{
    return [self.manager moduleCount];
}
```
如上述代码，模块化后`UITableViewDelegate`、`UITableViewDataSource`的大部分方法都转发给相应的 Module 去处理，大大简化了 Controller 的复杂度。

另外，在页面上增删任何元素都只需在 Manager 中增删相应的 Module 即可，Controller 无需任何改动——在 Controller 层面遵守了开放-封闭原则『OCP』。

> 模块化不仅简化了 Controller，同时也提高了代码的复用性。Module 可以在不同页面间复用。如果这些逻辑全部放在 Controller 里，基本没有复用性可言。

模块化有没有缺点？
答案是肯定的😒
模块化会增加类的数量、方法的数量(每个 Module 都要实现`UITableViewDelegate`、`UITableViewDataSource`的部分方法)。

当然啦，个人认为利大于弊😊

# UI 组件化
_____________________
QQ 阅读7.0改版，UI 修改的工作量占大头，涉及200多个页面的修改。
此时，充分体现出 UI 复用的重要性。

虽然，我们很早就提出通过 View-ViewModel 的方式实现 UI 组件化，提高复用性。
遗憾的是，由于历史原因，在我们的工程中依然存在大量重复的实现，即『同一 UI 样式，N 份实现』。这对于 UI 大改版是灾难性了！——「不仅工作量成倍增加，还有漏改的可能性」

为了避免灾难再次上演(8.0、9.0...)，此次改版过程中，我们严格要求所有 UI 都必须以 View-ViewModel 模式做成 UI 组件。

## UI 组件
在继续之前，我们简单描述一下什么是 UI 组件：
+ 可复用的 UI 单元；
+ UI 组件可包含子 UI 组件；

同时，我们将 UI 组件分为外部 UI 组件、内部 UI 组件：
+ 外部 UI 组件——与视觉对接，默认含有上下左右边距，为了提高其复用性，需实现`QRExternalUIComponent`协议，使得业务方可灵活控制其边距；
```mm
@protocol QRExternalUIComponent <NSObject>

- (void)setEdgeInsets:(UIEdgeInsets)edgeInsets;

@end
```
+ 内部 UI 组件——肯定是子 UI 组件，用于构造更大的 UI 组件，为了提高其复用性，同时控制复杂度，切边实现。

![](/img/fourbookUIComponent.png)
如上图，整体是一个四书的对外 UI 组件，含有视觉要求的上下左右边距，业务方可直接使用。
其中，红色虚线框住的则是一个内部组件，切边实现——没有上下左右边距，四书组件就是由4个这样的内部组件拼接而成。

## 复用粒度
***复用没把握好火候就变成耦合了。***
**例1.**
![](/img/banner1.jpeg)
![](/img/banner2.jpeg)
我们书城顶部 banner 有如上图的推书样式、通栏广告图样式、还有柱状图动画样式。
在实现的时候，通通将这些样式塞到一个类里面，通过`if...else...`区分，这就是严重的耦合，给后面的维护造成很大的困难。
**例2.**
![](/img/fourbook.jpeg)
![](/img/threebook.jpeg)
三书与四书 UI 也是通过`if...else...`区分，内部还要处理六书、八书的情况，还要兼容 iPad，内部实现异常复杂，导致大家都不敢去碰这块代码。

为此，我们制定了如下规则：
+ UI 布局相同才复用内部实现，所谓布局相同是指 UI 组件在结构上是相同的，如左边都是一个书封，右边都是两行文字，但书封大小、文字字号不同，则认为布局相同；
+ UI 布局相同，内部细节不同的，通过 ***Template Method 模式***实现代码复用，但对外提供的 UI 组件是独立的(简化业务层的使用)；
> Template Method: Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure.
> 此例中 UI 布局就是 Template Method 中的算法结构，而布局的细节则可以通过子类去控制。

+ 横向展示数量可扩展、纵向固定不变，如三书、六书是同一个 UI 组件，四书、八书是同一个，因为它们可以通过传入的数量控制展示。

## 隔离变化
我们经常吐槽 QQ 阅读 UI 的多样性在业界能排 Top1。
![](/img/singlebook.jpeg)
如上图单书组件，其中红框框住的部分就有15、16种变***幻***。
为此，我们将这部分抽取出来，作为单书组件的一个子组件由使用方负责构造该子组件并传给单书组件去展示。

好了，下面进入本节的正题，如何构造出复用性高的 UI 组件。
## 以 View-ViewModel 形式构建 UI 组件
高可复用的 UI 组件，至少要满足以下两点：
+ UI 组件不能与具体业务数据相绑定；
+ UI 组件内部不能处理业务逻辑——其本职工作仅是 UI 布局。

总之，UI 组件要与业务解耦。
此时，MVVM 模式进入我们的视线，在该模式中 ViewModel 的存在是不是很好的解决了上面的问题。
在 MVVM 模式中，ViewModel 向上为 View 提供展示数据（该数据已经在 ViewModel 中处理好了，View 无需任何处理，只要展示即可），向下接收来自业务层的数据，处理相关的业务展示逻辑。

可以看出，ViewModel 作为中间层很好地将业务与 UI 隔离开。
说到 MVVM，很多同学并不喜欢，觉得其中的 Data-Binding 很麻烦，但我们构建 UI 组件时用到的是 View-ViewModel 结构，并不要求一定是 MVVM，在 MVC 等模式下也可使用。

同时，我们采用的是面向接口的模式，View 对外依赖的是接口（protocol），而不是某个具体的 ViewModel。每个 UI 组件其结构如下：
![](/img/ViewViewModel.png)
如上图所示，若某个 UI 组件被多个业务所复用，可以根据需求定义多个 ViewModel 以处理不同的业务逻辑，每个 ViewModel 都实现`ViewModelProtocol`协议为 View 提供数据。

如上文提到的单书 UI，我们抽取为一个组件`QRLeftPictureRightTextView`：
![](/img/QRLeftPictureRightTextView.png)![](/img/QRLeftPictureRightTextViewModel.png)
该组件在信息流以及书城都有用到，为此定义了两个 ViewModel，以处理各自的业务逻辑：
![](/img/QRLeftPictureRightText.jpg)

至此，UI 组件化部分的内容基本结束。
在 QQ 阅读7.0版本中，实现了『同一 UI 样式，只有一份实现』，个人看来是一件很有意义的事情：
+ 提高开发效率，不必重复造轮子，工程代码得到很好的规范；
+ 减轻了设计师的工作，对于复用的组件，设计师只需在设计稿中标出组件编号即可；
+ 降低了开发与设计师的沟通成本；
+ 为下次大改版奠定了很好的基础。

> Module 与 UI 组件在两个不同的层面实现复用。

# 基于响应链的事件处理
_____________________
现有的事件处理方案有两大痛点，于是提出了基于响应链『Chain of Responsibility』的事件处理方案。
+ 痛点1
![](/img/ViewHierarchy.png)
大多数场景下 View 的层级结构如上图所示。我们知道，View 一般不处理用户事件，需要逐级向上传递给 Controller，因此需要沿着上图的层级结构逐级传递处理事件的 delegate。这种单调、重复、琐碎的代码非常令人不悦：
```m
cell.delegate = controller;
view.delegate = cell;
…
```

+ 痛点2
随着版本的迭代，不同类型的 cell/view 极有可能出现不同的事件处理接口，如下图所示：
![](/img/celldelegate.jpeg)
这严重违反了面向对象设计的开闭原则『OCP』——每增加一种 cell 类型此处都需要修改。

尤其是第一点一直困扰着我。直到前不久在《Design Patterns》一书中看到在介绍『Chain of Responsibility』模式时的一句话：『Using existing links works well when the links support the chain you need. It saves you from defining links explicitly, and it saves space』。
`UIResponder` 中的 `nextResponder`不正是这个『existing links』吗！
最上层 View 的事件通过`nextResponder`链就可以顺利传到 ViewController 中，从而也就省去了 delegate 的逐级传递了，痛点1、2随之化解。
为此，我们为 `UIResponder`添加了传递、处理事件的分类：
```mm
@protocol ZSCEvent <NSObject>
@property (nonatomic, strong) __kindof UIResponder *sender;
@property (nonatomic, strong) NSIndexPath *indexPath;
@property (nonatomic, strong) NSMutableDictionary *userInfo;
@end

@interface UIResponder (ZSCEvent)
- (void)respondEvent:(NSObject<ZSCEvent> *)event;
@end

@implementation UIResponder (ZSCEvent)
- (void)respondEvent:(NSObject<ZSCEvent> *)event
{
    [self.nextResponder respondEvent:event];
}
@end
```
`UIResponder`的实现只是简单地将事件传递给`nextResponder`。
由于 View 不包含业务数据，所以事件传递的过程中需要不断添加一些信息。
> 因此，我们将`ZSCEvent#userInfo`定义为 mutable。正常情况下外露接口一般都是 immutable。

```mm
@implementation UITableViewCell (ZSCEvent)
- (void)respondEvent:(NSObject<ZSCEvent> *)event
{
    event.sender = self;
    [self.nextResponder respondEvent:event];
}
@end
```
如，在`UITableViewCell`的`respondEvent:`中需要将`sender`设置为`self`，以便在`UIViewController`中可以通过`cell`找到对应的 Module。
```mm
- (void)respondEvent:(NSObject<ZSCEvent> *)event
{
    NSAssert([event.sender isKindOfClass:UITableViewCell.class], @"event sender must be UITableViewCell");
    if (![event.sender isKindOfClass:UITableViewCell.class]) {
        return;
    }
    
    NSIndexPath *indexPath = [_tableView indexPathForCell:event.sender];
    id<ZSModule> module = [self.manager moduleAtIndex:indexPath.section];
    
    event.sender = self;
    event.indexPath = indexPath;
    [event.userInfo setObject:_tableView
                       forKey:ZSCEventUserInfoKeys.tableView];
    
    [module handleEvent:event];
}
```
在 View 中的事件处理代码可以这样：
```mm
- (void)_clickedButton:(id)sender
{
    ZSCEvent *event = [[ZSCEvent alloc] init];
    event.sender = self;
    [event.userInfo setObject:@(YES) forKey:@"clickedButton"];
    
    [self respondEvent:event];
}
```
> 如果一个 cell 中有多个事件需要处理，就需要在`userInfo`中加以区分，如上面代码第`5`行。

总之，通过`UIResponder`的`nextResponder`响应链，不必再在 view 的层级间传递 delegate，减少了琐碎的代码，提高了开发效率。同时也统一规范了事件处理方案。

# 业务模板化
_____________________
[列表类应用场景模板化](https://zxfcumtcs.github.io/2018/09/17/ListSceneTemplating/)一文对此有详细的描述，在此就不赘述了。
其效果还是不错的。
很多二级页，由于 Module 是完全复用的，通过模板化脚本***半小时***就能做好一个二级页✌️。

# 小结
_____________________
简单、高效一直是软件开发、工程管理追求的目标，本文从实际项目经验出发，从架构、解耦、复用等角度总结出一套开发解决方案。
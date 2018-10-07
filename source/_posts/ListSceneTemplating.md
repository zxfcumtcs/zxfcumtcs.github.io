---
title: 列表类应用场景模板化
date: 2018-09-17 23:05:02
tags:
- 框架
- iOS
---
由于列表类应用场景具有固定的流程和模式，本文首先简要介绍了 QQ 阅读中列表类应用场景的架构，然后提出对这一场景进行模板化，以便提高开发效率、减少沟通理解成本。
同时，提出一套基于 『Chain of Responsibility』 的事件处理方案，也在一定程度上提高了开发效率，减少了琐碎代码量。
<!--more-->
©原创文章，转载请注明出处！
# Overview
______________________________
2009年作为移动互联网开发元年至今已过去十年，移动客户端开发技术也已从拓荒时代进化到成熟稳定阶段。十年间，数以万计的 App 被创造出来。然而，细观市面上的 App 会发现大多数及至绝大多数 App 的大多数及至绝大多数应用场景都是列表类的。
所谓列表类应用场景主要有以下几个特征：
+ 以 UITableView 展示多条数据；
+ 数据不变，仅用于展示，或只有很少的状态变化；
+ 没有复杂的用户交互(如：UGC)。

从上述特征可知，所有列表类应用场景都具有『相同的代码结构』，也就意味着我们经常在做一些重复性的工作。
同时，『相同的代码结构』也意味着可以将其模板化。通过模板化列表类应用场景至少有以下两点收益：
+ 提高工作效率，减少重复性劳动；
+ 统一代码结构，减少项目组内理解沟通成本。

# QQ 阅读列表类应用场景架构简述
_________________________________
列表类应用场景其流程无外乎：从网络或本地磁盘获取数据，再将数据以列表(tableIview)的形式展示出来，其中最主要的交互就是点击进入次级页面。
每个项目团队可能都有一套用于列表类场景的架构，在 QQ 阅读不断迭代的过程中我们也演化出了一套相关的架构。本文会以这套架构为例讲述模板化的思路。
> 使用什么样的架构不是重点，重点是将使用的架构模板化的思路。

![](/img/ListSceneClassDiagram.png)
![](/img/ListSceneTimingDiagram.png)
上图分别是我们这套架构的关键类图和时序图。整体上是由经典 MVC 模式演化而来：
+ Manager(Interface)：对应 MVC 中的 Model 『层』，主要负责数据的获取、管理等；
+ Controller：各个模块的协调枢纽；
+ Cell/View：对应 MVC 中的 View，仅仅负责 UI 布局逻辑；
+ ViewModel：处理 UI 展示相关的业务逻辑(详细信息请参看之前的文章[『自定义 UI 组件库』](https://zxfcumtcs.github.io/2017/03/04/CustomUIControls/))；
+ Module(Interface)： 将其称之为『业务模块』，一个页面由多个不同或相同类型的模块组成。如 QQ 阅读精选页的『今日必读』、『今日秒杀』等都是模块。

![](/img/RecommendedToday.jpeg)
![](/img/TodaySecondKill.jpeg)
当然 Module 也可以是一个简单的样式：
![](/img/SimpleStyle.jpeg)
在 TableView 中一个 Module 对应一个 section。Module 的职责主要有：网络数据的解析、 为 TableView datasource 提供数据(如：创建 cell 等)、处理用户事件——即负责『模块』的所有业务逻辑(与 React Component 类似)。
> Module 的存在主要是减轻 Controller 的负担。

通常情况下，Manager(Model)存储的是纯粹的业务数据(从网络拉取的数据)，这样就需要在业务数据与 Module『模块』 间建立映射关系。为了省去这层映射，直接由 Module 解析、存储业务数据。
这种做法也存在弊端，由于将网络数据的解析、控制 UI 展示的逻辑(创建 cell 等)都放在了 Module 中。使得 Module 违反了『单一职责原则』。
> 『单一职责原则』(SRP)作为面向对象设计的五大原则『SOLID』之一，很容易理解，也很难把握！『就好像生活中的各种"适量"，适量放点盐、适量加点水…』
 Bob大叔在《敏捷软件开发》中，将类的单一职责原则描述为『应该仅有一个引起它变化的原因』。

在 Module 中，网络数据解析、UI 展示就是两个可变的原因——『同样的 UI 用于展示不同的网络协议返回的数据、同一协议返回的数据展示为不用的 UI』。
在 QQ 阅读中，书籍列表页就属于『同样的 UI 展示不同协议返回的数据』：
![](/img/BookList.jpeg)
针对这种情况，无非就是将其中一个变化因子抽取出来，可以将网络数据解析抽取为一个单独的类。
由于Module 中这两个变化因子变动的概率并不大，为了降低复杂度，在模板中并没有将这两者分离开。
>『敏捷开发』的原则之一就是尽量保持代码简单、并在必要时进行重构，防止代码变坏。

# 模板化
___________________________________
通过上述介绍可知，Controller、Manager、API 的代码基本是固定的——可以模板化，另外 View-ViewModel 是可以高度复用的。所以模板化后新增一个列表类应用场景的主要工作集中在 Module 上。
> 所谓模板化就是提供一套代码模板，在实例化时将模板中的『Template』关键字替换成业务名。

我们这套模板中有：Manager、Module、View 以及 API 四个目录，ZSTemplateManager.m(.h)、ZSTemplateViewController.m(.h) 以及 ZSTemplateAPI.m(.h)六个文件，其中可以模板化的代码主要有：
+ Controller：设置 tableview(含下拉、上拉、datasource、delegate)、设置导航栏、错误\空数据处理、向 manager 发送请求数据的调用、事件处理等；
+ Manager：管理 module、向 API 发送网络请求、缓存处理等；
+ API：发送网络请求。

也就是模板化后上述功能可以通过转换脚本一键生成，不用重复地写这些代码。尤其是 Controller 基本可以直接使用。
> 转换脚本、demo 已提交到 github 上[『ZSTemplatedListScene』](https://github.com/zxfcumtcs/ZSTemplatedListScene)。转换脚本的功能就是将模板中的『Template』关键字替换为业务名(包括代码和文件名中的)。如，demo 中的通信录业务：![](/img/transformsh.jpeg)注：demo 中的模板仅是个『demo』，其中的网络请求、缓存等功能可替换为项目中统一的模块。

总之，通过转换脚本可以一键生成部分代码，提高了开发效率。同时，通过模板也规范了代码结构，减少了项目组沟通理解成本。
# 基于 Chain of Responsibility 事件处理方案
___________________________________
目前的事件处理有2个痛点，于是才有了基于 Chain of Responsibility 的事件处理方案。
+ 痛点1
![](/img/ViewHierarchy.png)
大多数场景下 View 的层级结构如上图所示。我们知道，View 一般不处理用户事件，需要逐级传递给 Controller，因此需要沿着上图的层级结构逐级传递处理事件的 delegate。这种单调重复琐碎的代码有种令人不悦的感觉：
```m
cell.delegate = controller;
view.delegate = cell;
…
```

+ 痛点2
随着版本的迭代，不同类型的 cell/view 极有可能出现不同的事件处理接口，如下图所示：
![](/img/celldelegate.jpeg)
这严重违反了面向对象设计的开闭原则(Open-Closed)——每增加一种 cell 类型此处都需要修改。

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
如，在`UITableViewCell`的`respondEvent:`中需要将`sender`设置为`self`，以便在`UIViewController`中可以通过`cell`找到对应的 module。
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
> 如果一个 cell 中有多个事件需要处理，就需要在`userInfo`中加以区分，如上面代第`5`行。

总之，通过`UIResponder`的`nextResponder`响应链，不必再在 view 的层级间传递 delegate，减少了琐碎的代码，提高了开发效率。同时也统一规范了事件处理方案。
# 小结
___________________________________
提升开发效率、规范代码结构一直是我们追求的目标。文本通过对列表类应用场景模板化以及通过『Chain of  Responsibility』机制处理用户事件，在一定程度上提高了开发效率并规范了代码结构。
---
layout: post
title: 论面向接口编程
date: 2019-12-04 22:25:55
tags:
- 设计
---
本文首先从接口的实现方、使用方角度阐述了什么是接口，其次分析了面向接口编程的意义。
<!--more-->
©原创文章，转载请注明出处！

# 我眼中的接口
_________________
面向接口编程作为口号可以说是妇孺皆知，臭大街了！
但在日常开发、交流过程中发现很多同学对面向接口编程的理解还是有所偏差。
因此，想通过这篇小短文，谈谈我对面向接口编程的理解，希望对大家有所帮助。

首先要回答的问题就是：
接口是什么？
+ `C++`、`JavaScript`、`dart`的 (abstract)`class`——『形式上的接口』
+ `Objective-C`的`delegate`、`Swift`的`protocol`、`Java`的`interface`——『语义上的接口』

> 接口属于面向对象编程(OOP)的范畴

这样的答案无疑是『政治正确』的，但并非我们想要的，我想从另外一个角度来看待这个问题：
> 接口是一种抽象

+ 接口使用方：**对外界『依赖』的抽象，表明其依赖哪些能力**；
+ 接口实现方：**对自我『能力』的抽象，宣称其具备哪些能力**。

正是由于接口的抽象性，接口使用方与实现方才得以解耦。
下面，我们以`UITableViewDataSource`为例，来具体感受一下：
```mm
@protocol UITableViewDataSource<NSObject>

-(NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section;
-(UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;
...
@end
```
`UITableView`对 iOS 开发同学来说再熟悉不过了，其有一个需要实现`UITableViewDataSource`接口的属性`dataSource`。
`UITableView`作为`UITableViewDataSource`接口的**使用方**，很清楚的表明了其对外界的依赖：
+ `numberOfRowsInSection:`——依赖外界告诉它，某个 section 有几行；
+ `cellForRowAtIndexPath:`——依赖外界提供某个 indexPath 处的cell；
+ ...

总之，通过`UITableViewDataSource`清楚地表明了`UITableView`需要哪些能力的支持，也就清楚地说明了该如何去使用`UITableView`。

```mm
@interface ABCViewController () <UITableViewDelegate,UITableViewDataSource>
...
@end
```

通过上面这行代码，我们就能清晰地知道，`ABCViewController`具有`numberOfRowsInSection`、`cellForRowAtIndexPath`等能力，因为其实现了`UITableViewDataSource`接口！也就是其可以供`UITableView`使用。
> `UITableView`之所以有如此好的通用性，就是采用了面向接口编程，将其对外界的依赖抽象成2个接口：`UITableViewDelegate`、`UITableViewDataSource`，即只要是实现了这两个接口的都可以为其所用。

# 意义
_________________
+ **简洁、易维护**——如上节所提到的`UITableView`，其内部实现想必非常复杂，但由于有`UITableViewDelegate`、`UITableViewDataSource`两个抽象接口，表明了其对使用方的依赖，我们非常容易地就可以使用`UITableView`，无须关心其内部细节，只要实现这两个接口即可；
+ **解耦**——接口作为一个抽象层，很好地将使用方与实现方隔离开来，使得两者不再有直接依赖关系，双方的复用性、扩展性(尤其是接口使用方)得到极大提高，具体例子可以参考[自定义 UI 组件库](https://zxfcumtcs.github.io/2017/03/04/CustomUIControls/)这篇文章；
+ **分工协作**——通过接口使得原本相互依赖的双方得以很好的解耦，分工协作更加顺畅，双方可以并行开发，互不干扰；
+ **可测性更好**——这里主要指接口使用方的测试，因为通过接口可以更方便地 mock 数据供接口使用方测试用；
+ **拥抱变化**——对开发同学来说最憎恨的莫过于已经开发好的或开发中的需求又变了，除了『撕』之外，我们还可以采取积极的防御措施，通过接口『隔离变化』，将变化带来的影响降到最低。

![](/img/tab_page.png)
这是一个前不久在项目中真实遇到的例子，某个页面，当初只有 A、B 两个 tab，但在开发过程中又变成 A、B、C 三个 tab，之后又变成了 D、E 两个 tab。(每个 tab 的数据开源、UI 样式完成不一样)

刚开始时，A、B 两个 tab 的逻辑是直接放在主页面里面的
当产品需要增加 C tab 时，隐约感到情况不妙
为了，防止后续还有改动带来的影响，决定将 tab 相关的逻辑抽离出来，主页面不再依赖于某个特定的 tab，而是依赖于抽象后的接口。
需要放到该主页面的 tab 自行实现这套接口即可。
从此，tab 的增删改就与主页面无关了，即实现了 『OCP』，这也是23种经典设计模式之一的：『策略模式』。

总之，面向接口，即面向简洁、面向变化。
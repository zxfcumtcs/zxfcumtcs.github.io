title: 横屏不好惹
date: 2016-02-28 20:17:20
tags:
- iOS
---
本文简单讨论了从横屏页面 pop 回竖屏页面以及在横屏页面 push 竖屏页面的实现方式。
<!--more-->
©原创文章，转载请注明出处！

# Overview
__________________
最近由于项目需要又和横屏扛上了，但本文不打算过多介绍 iOS 中转屏相关的理论，而是简要的分析横竖屏在实际使用中的两个场景：
- 从仅支持竖屏的页面 push 一个支持横竖屏切换的页面，再从横屏状态返回；
- 从横屏状态 push 一个竖屏页面。

(上述问题通过 present 方式可以解决，但在某些场景 present 不是好的解决方案，本文讨论的是 push 方式)
假设 controllerA 仅支持竖屏，controllerB 支持横竖屏切换。

# 从横屏 pop 回竖屏
___________________
场景：从 controllerA push controllerB，controllerB 切到横屏，再 pop 掉 controllerB（即返回 controllerA）。
此时由于设备还处于横屏状态，原本只支持竖屏的 controllerA 也以横屏的方式显示了，这当然不是我们希望看到的结果。

一直对此感到很苦恼，没有找到好的解决办法。

『山穷水复疑无路，柳暗花明又一村』，一个偶然的机会发现在 pop controllerB 返回 controllerA 时系统会调用 controllerA 的`supportedInterfaceOrientations`方法，若在该方法中返回`UIInterfaceOrientationMaskPortrait`，即可顺利以竖屏方式显示 controllerA。

由于从iOS6开始，系统在处理转屏时有很大的改动，不再调用具体某个 controller 的转屏方法(如：`supportedInterfaceOrientations`、`shouldAutorotate`)，而是由 navigationController 接管，因此一般情况不会在 contoller 里面实现这些方法。


# 从横屏 push 仅支持竖屏的页面
_____________________
场景：controllerB 处于横屏状态，此时需要 push controllerA。
由于此时没有触发转屏操作，无论是 navigationController 还是涉及到的相关 controller(该场景下的 controllerA、controllerB)的转屏方法都没有调到。controllerA 自然也是以横屏姿态出现了。

此时，要是能触发一下转屏相关的事件就好了！
为了达此目的，需要使用点『奇技淫巧』的手段了，在 push controllerA 之前先 present 一个仅支持竖屏的页面，立马上又 dismiss 掉，这样就能触发转屏事件了，controllerA 也就变成竖屏了。

![](/img/PortraitViewController.png)

从 iOS 的开展来看，Apple 是希望淡化页面方向这个概念(尤其是 iOS8 之后)，但在实际项目中有时又十分关注这点，此时显得很无奈！

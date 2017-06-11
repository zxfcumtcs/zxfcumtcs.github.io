title: 性能调优那些事
date: 2015-05-07 22:19:48
tags:
- iOS
---
本文讨论了优化 UITableView 性能的几种常用方法，并对为何不应直接重载 UITableViewCell 的 drawRect 方法作了详细的分析说明。
<!--more-->
@原创文章，转载请注明出处！

# Overview
_____________________________

App 性能优劣将决定用户第一印象，最近在项目中对 app 做了一次性能上的简单优化，由此做一些相关的总结。在前一篇文章[Core Animation](http://zxfcumtcs.github.io/2015/03/21/CoreAnimation/)中，我们分析了影响性能的几个关键原因(Offscreen Rendering、Rasterization、drawRect:、Image等)，主要是理论上的分析，本文将从实战角度出发，结合 UITableView 讨论如何优化性能。

性能优化是一个纠结的过程，某个特定解决方案对性能的影响在不同的场景下有不同的表现，如：Rasterization特性利用恰当会优化性能，否则会有损性能。因此不能盲目猜测，优化前先通过工具(Instruments)找出性能瓶颈，有目标且针对性地优化，最终还要通过工具检测优化效果。

Apple 给我们的建议是(Performance Investigation Mindset)：
![](/img/PerformanceInvestigationMindset.png)

# 初级优化
_____________________________

### UITableViewCell 高度

设置 cell 的高度有两种方式：
+ 设置 tableview 的 `rowHeight` 属性；
+ 实现 UITableViewDelegate 中的 `tableView:heightForRowAtIndexPath:`方法。

其中，heightForRowAtIndexPath 方法的优化级高于 rowHeight 属性，在实现了 heightForRowAtIndexPath 方法的情况下，为了正确显示滚动条，tableview 需要计算其整体的高度，意味着针对每一行都会调用一次该方法。如果行数过多、heightForRowAtIndexPath 方法过于复杂，明显会影响 tableview 的性能。

ps：此情况下，tableview 对 heightForRowAtIndexPath 的调用发生在 tableview 加载过程中，影响的是加载速度。

**因此，对于固定、统一高度的 cell，直接对 rowHeight 属性进行赋值，不要实现 heightForRowAtIndexPath 方法。**

然而，实际情况并非如此简单，大多数情况下 cell 的高度都是动态的。

此时，我们能做些什么呢？

首先，从 iOS7 开始 Apple 提供了新的接口：`tableView:estimatedHeightForRowAtIndexPath:` 用于估算 cell 的高度，主要是用于显示滚动条，因此在该方法中可以返回一个大约的值而没必要经过严格的计算，节省了加载时间，直到 cell 需要显示出来时才调用 heightForRowAtIndexPath 计算精确的高度值。**可知，通过该方法只是将加载时间延迟到 table 滚动时间。**

其次，为了计算 cell 的高度，通常需要对 cell 进行 layout 操作，因此 layout 结果以及得到的高度都可以 cache 起来以免重复计算。

### Reuse cell

为了节省内存，提高性能，tableview 提供了 cell 的重用机制，对于滑出屏幕的 cell tableview 将其加入 reuse 队列，在需要显示新 cell 时从该队列取出重复利用。我们应该要充分利用好这一特性。

有时，为了提升来回滑动的性能，防止滑出的 cell 立即被重用，而在来回滑动时需要重新 layout cell，可以增加 reuseIdentifier 的间隔.
![](/img/reuseIdentifier.png)

如上图，将 cell 复用的周期设为5可以提升来回滑动的流畅度。

### 拒绝 Misalignment

在 [Core Animation](http://zxfcumtcs.github.io/2015/03/21/CoreAnimation/) 这篇文章中已经讨论过 misalignment 对性能的影响，当出现 misalignment 时，GPU 需要从 source texture 中 blending 多个像素点的值来生成一个像素，增加了额外的计算量。因此需要避免出现 Misalignment 的情况，最简单的方法无非是对坐标值取整，如 ceil。

我们也可以定义些宏方便使用：
![](/img/Misalignment.png)

像素不对齐问题，可以通过 Instruments 的 Core Animation 功能下的 Color Misaligned Images 选项检测出来。通过该选项，像素不对齐的情况将被标注为紫色，图片大小与最终显示时不一致的情况将被标注为黄色(即图片需要缩放)。
![](/img/coreanimationdebugoptions.png)
![](/img/Misalignmentdebugresult.png)

### 避免 Transparent

对 GPU 来说，透明意味着有大量额外的工作要做，在合成最终bitmap 时需要针对透明 layer 的每个像素计算叠加效果。这对于滑动时的性能来说是大忌。在代码中经常会出现：`xxView.backgroundColor = [UIColor clearColor];`，意味着 xxView 具有透明背景！通过 Instruments 的 Core Animation 功能下的 Color Blended Layers 选项可以检测哪些 layer 是透明的。透明层会被标注为红色，颜色越深说明叠加的透明层就越多。
![](/img/ColorBlendedLayersDebugResult.png)

# 中级优化
_________________________________

### 避免 Offscreen Rendering 

在 [Core Animation](http://zxfcumtcs.github.io/2015/03/21/CoreAnimation/) 这篇文章中对 Offscreen Rendering 有详细的讨论，其对性能有一定的影响，在大多数情况下需要尽力避免。

例1：生成阴影效果，可以通过多种方式实现：
![](/img/shadowoffset.png)
或：
![](/img/shadowpath.png)

然而，前者会产生 Offscreen Rendering：
![](/img/shadowoffsetoffscreenreadering.png)
而后者却不会：
![](/img/shadowpathnotoffscreenrendering.png)

上述两种方式对性能的影响：
![](/img/shadowperformance.png)

例2：生成圆形图片，首先想到的常规方法是通过结合 layer 的 cornerRadius 和 masksToBounds 属性：
![](/img/cornerRadius.png)
问题是该方式会产生 Offscreen Rendering。另外一种替代方式是在图片上方叠加一张中间有个透明圆形的图片，当然该方式会引起 blending，但据 Apple 在 WWDC 上的介绍，其性能优于前者：
![](/img/roundperformance.png)

例3：通过 Rasterization cache 耗时的计算结果：
其实这是一个权衡的过程，我们知道 Rasterization 会产生 Offscreen Rendering，有一定的性能损耗，同时还存在 Rasterization 失效的"危险"。

在例1中我们知道通过 shadowoffset 方式生成的阴影效果会导致 Offscreen Rendering，并且离屏渲染发生在每一帧上，带来性能损耗，此时，可以通过 Rasterization cache 生成的阴影效果，从而只需一次离屏渲染即可。

在 iOS Core Animation Advanced Techniques 这本书给出的例子中(12.2)，tableview 每行显示的图片和文字都使用 shadowoffset 的方式添加了阴影效果：
![](/img/shadowoffsetrasterizationexample.png)

在不使用 Rasterization 时，在 iphone5 上测出的帧率：
![](/img/shadowoffsetnorasterizationfps.png)
使用 Rasterization 时的帧率：
![](/img/shadowoffsetrasterizationfps.png)
后者较前者在帧率上有一定的提升。

需要注意的是，虽然在该例子中每个 cell 的内容是固定不变的，不会导致 Rasterization 的结果失效，但由于在滚动过程中 cell 是复用的，这将导致之前 Rasterization 的结果失效。

通过 Instruments 的 Core Animation 功能下的 Color Hits Green and Misses Red 选项可以检测 Rasterization 失效的情况(绿色为 Rasterization 有效，红色为失效)。

在 tableview 静止时，Rasterization 是有效的：
![](/img/RasterizationYes.png)
而在滑动过程中，由于 cell 的复用导致 Rasterization 失效：
![](/img/RasterizationNO.png)

总之，Offscreen Rendering 对性能的影响值得关注，大多数情况下需要尽力避免，在例3中也是以一次 Offscreen Rendering 换取避免多次。
![](/img/offscreenrenderingsummary.png)

### Lazy Load

延迟加载有时不失为提高性能的有效措施之一，UITableView 本身就采用了延迟加载的策略：只有当 cell 需要显示时才进行加载。
我们可以在 tableview 滑动过程中禁止某些非关键的、耗时操作，如：滑动过程中 gif 图片不解码，直到 tableview 停止滑动，才对可见范围内的 gif 解码。

# 高级优化
______________________________________

### 在子线程预处理图片

对图片的处理通常有较大的性能损耗，如：读取图片的 io 操作、图片解码、缩放、渲染等，若处理不当将严重影响性能。
为了提升图片处理的性能，其中 io、解码、缩放等操作可以在子线程中预处理，再将处理结果转交主线程显示。
其中，需要特别注意的是图片的缩放，如果图片本身的尺寸与需要显示的尺寸不一致时，需要做缩放操作，这是一个非常耗时的过程。因此，app 中使用的图标等尽量保持尺寸一致，对于下载的图片虽无法保证，但可以先在子线程按显示尺寸对图片做裁剪。图片在显示过程中需要缩放的情况可以通过 Color Misaligned Images 选项检测，需要缩放的图片会被标注为黄色：
![](/img/imagesize.png)
可以在子线程做如下处理(注：为了节省篇幅，在下面的代码中只处理了UIViewContentMode为UIViewContentModeScaleAspectFill的情况)：
![](/img/imagecode1.png)![](/img/imagecode2.png)

### 不可直接重载 UITableViewCell 的 drawRect 方法

无论是出于优化性能还是其他目的，经常会看到直接重载 cell 的 drawRect 方法。但这样做是有问题的，暂且不讨论其是否能达到优化性能的目的，从 cell 的结构上来讲，需要显示的内容应该是基于 cell 的 contentView，而不应该直接 draw 在 cell上。

首先，我们看看如下简单的代码在 iOS7 和 iOS8 上运行的结果：
![](/img/drawincellcode.png)
iOS7:
![](/img/ios7drawcellresult.png)
没错，什么也没有！
iOS8:
![](/img/ios8drawcellresult.png)
在 iOS8上显示正常！

那如何才能在7上显示正常呢？答案是将 cell 的 backgroundcolor 设为 clearcolor，也就意味着透明！
![](/img/ios7drawcelldebug.png)

两个问题：为什么 iOS7 不行而 iOS8 行、为什么在 iOS7 需要将 backgroundcolor 设为 clearcolor。

首先，讨论第二个问题，这是在 iOS7 上调试打印出来的结果：
![](/img/ios7debugcellviewhierarchy.png)

可以看到 iOS7 上 cell 的 view hierarchy：
![](/img/ios7cellviewhierarchy.png)

在 cell 上有个 subview：UITableViewCellScrollView，并且其默认背景色为白色。
若将 cell 的 backgroundcolor 设为 clearcolor：
![](/img/ios7cellclearcolor.png)
可以看到UITableViewCellScrollView的背景色也变为 clearcolor 了。
很明显，在 iOS7 上之所以 drawrect 绘制的内容没有显示，原因在于被UITableViewCellScrollView拦住了。

我们再看一下，iOS8 上 cell 的 view hierarchy：
![](/img/ios8cellviewhierarchy.png)
第一个问题有答案了：iOS8 上 cell 的 view hierarchy 变了，UITableViewCellScrollView不再存在了。
![](/img/ios8cellviewhierarchy1.png)

还有另外一个问题：无论是 iOS7 还是 iOS8 上，当 cell 高亮显示时，drawrect 绘制的内容还是会消失！
![](/img/cellhightlight.png)

看看此时的 view hierarchy(iOS8)：
![](/img/UITableViewCellSelectedBackground.png)

是的，就是UITableViewCellSelectedBackground在搞怪！

可以看到，直接重载 cell 的 drawrect 方法问题多多！

正确的做法是：将需要 draw 的内容放到一个单独的 view 上，然后将其作为 contentview 的 subview。
ps: 在[Core Animation](http://zxfcumtcs.github.io/2015/03/21/CoreAnimation/) 这篇文章中已经讨论过重载 drawrect 方法有一定的性能损耗，在决定重载前需要三思！

# 小结
________________________________________

性能优化是一个复杂的过程，也是一个权衡、折中的过程。在优化前需要明确性能瓶颈在哪，优化后需要通过工具检测优化效果。

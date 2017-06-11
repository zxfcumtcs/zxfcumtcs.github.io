title: Core Animation
date: 2015-03-21 09:41:13
tags:
- iOS
---
本文重点讲解了 Core Animation 中影响 app 性能的几个关键因素，如：Offscreen Rendering、Rasterization、drawRect:，以及 image decode。
<!--more-->
©原创文章，转载请注明出处！

# Overview
______________________________

Core Animation，核心动画，给人的第一印象是一项专注于高端动画的技术。事实情况并非如此，Core Animation是一个用于管理视图排版、合成、渲染以及动画的基础 framework，其在整个系统中的层级如下[图](https://robots.thoughtbot.com/designing-for-ios-graphics-performance)所示：
![](/img/coreanimationframework.png)

可以看到，UIKit 建立在 Core Animation 基础之上，在 Core Animation 之下是 OpenGL ES 和 Core Graphics，分别对应 GPU 和 CPU。Core Animation 本身并不做渲染工作，而是将这些工作转交给 Graphics Hardware(GPU)处理。

提及 Core Animation 在视图排版、合成上的工作，就绕不过Core Animation 的核心成员——CALayer。

# UIView and CALayer
_______________________________

对 iOS 开发者来说，UIView 恐怕是再熟悉不过了，通过 UIView(subclass) 可以在屏幕上展示内容、执行变换、动画以及响应用户事件。然而，这其中大多数事情并不是由 UIView 完成，而是由其幕后"黑手"CALayer 具体执行，如：layout、rendering以及animation等。

每个 view 背后都有一个 layer 与其一一对应，通过 view 的 layer 属性可以获取该 layer(backing layer)，当然我们也可以在 view 上添加额外的 layer。屏幕上的 view 间存在一定的层级关系(view hierarchy)，同样在 layer 间也存在与 view hierarchy 对应的层次关系(layer hierarchy tree).

![](/img/layertree.jpg)

事实上，view 可以看作是对 layer 的一次封装，view 与 layer的最大区别在于 view 可以响应用户事件而 layer 不能。

Apple 君为何要设计 UIView 和 CALayer 这两套系统？既生 view 何生 layer？

Apple 君的真实目的是希望做到功能分离，代码复用。用户交互在 iOS 和 Mac OS 上有着本质的区别，而像layout、drawing 以及 animation 在两者上的处理是一致的。因此，在 iOS 和 Mac OS 上分别存在UIKit 和 AppKit，它们处理各自平台上的用户交互，但幕后的 CALayer 是一致的。

我们已经知道，layer 与 view 相比不具备处理用户事件的能力，那么什么事情是 layer 可以完成而 view 不行：**阴影、圆角、彩色边框、3D 变换、非矩形结构、透明mask以及非线性动画等。**

由于本文的重点不是讲解 CALayer，关于CALayer 的内容就介绍这些。下文重点介绍 Core Animation 对 app 性能影响以及如何达到性能最优。

# CPU vs. GPU
______________________________

CPU 相信大家再熟悉不过了，执行各种指令的神经中枢。GPU(图形处理单元，Graphics Processing Unit)针对高并行浮点运算做了特定优化，因此在图形处理上比 CPU 更胜一筹。居于此，我们希望尽可能多地将图形渲染工作交由 GPU 完成，然而 GPU 的处理能力也是有限的，当 GPU 超负荷运转时，即使 CPU 还有空闲，app 的性能也将下降。因此，动画性能优化主要是考虑 CPU 与 GPU 间的负载均衡。

### The Stages of an Animation

一般情况下，我们认为animation都是发生在 app 内部的。其实不然，在 iPad 上通过4指左右滑动可以切换当前运行的 app，此过程就涉及 app 间的animation了，也就是在同一 animation 中需要显示来自两个不同 app 的内容。也就意味着，执行 animation 的代码不能在某个 app 内部完成(sandboxed所限)。

事实上，执行动画、将各个 layer 的内容合成到屏幕的过程是在一个单独的进程中完成的，该进程我们称之为render server。在 iOS5及之前的版本，该进程实际上是 SpringBoard Process(该进程同时运行 iOS 的 home screen)，从 iOS6开始，有一个特定的进程处理，即 BackBoard Process。

![](/img/coreanimationpipeline.jpg)
执行一个animation，实际上分为6个步骤：

1、Layout——生成 view、layer 以及管理其层次关系(如果重载了`layoutSubviews`，将执行该方法)，设置 layer 属性(如：frame、background color、border 等)；
2、Display——绘制 layer 的 backing image。如果重载了`drawRect:`或实现了`drawLayer:inContext:`，将执行这些方法；
3、Prepare——准备好要发送到 render server 的 animation data。在动画过程中要用到的 image，也将在该过程解码；
4、Commit——将 layer、animation properties 等数据通过 IPC(Inter-Process Communication) 发送给 render server 做动画和展示，如果 layer tree 非常复杂在该阶段将有较大的性能损耗；

上述4步在 app 内部执行，当 layer、animation data 到达 render server 后，被反序列化为render tree，render server 利用 render tree 针对动画的每一帧执行以下两步：
5、Calculates the intermediate values——动画执行过程中计算各个 layer 属性的中间值并设置 OpenGl geometry；
6、Render——将可见的形状渲染到屏幕上。

![](/img/animationstep.png)
其中，前5步由 CPU 完成，最后一步由 GPU 完成，而我们真正能控制的只有前2步。当然，在 layout、display 过程中，我们可以决定哪些工作提前由 CPU 完成，哪些工作交由 GPU 完成，这也是我们优化动画性能的关键点所在。


#### GPU-Bound Operations

GPU 在图形处理上做了特定优化，那么什么操作会在 GPU 上执行呢？

简单地的说，CALayer 的各个属性(如：background color、contents 等)都是通过 GPU 绘制到屏幕上的。虽然说 GPU 对图形处理有优化，但有些情况会影响 GPU 的绘制性能：

+ Too much geometry——在屏幕上存在过多的形状(layer)，这些 layer 最终都需要通过 GPU 作光栅化处理。最新的 GPU 能同时处理大量的layer，因此在此问题上 GPU 一般不会成为性能瓶颈。但 layer 需要预处理并通过 IPC 发送给 render server，过多的 layer 将使 CPU 成为性能瓶颈。
+ Too much overdraw——主要由透明的 layer 引起，当存在透明 layer 时 GPU 需要在每一帧中对同一像素执行多次填充，这将明显影响动画的性能。
+ Offscreen-Rendering——离屏渲染，需要为 offscreen image 分配额外的内存并切换 drawing context，这将会影响 GPU 的性能。
+ Too-large images——当要绘制的 image 超过 GPU 的处理能力时，必须通过 CPU 来处理，这将明显性能。

### CPU-Bound Operations

在 Core Animation 中，CPU 的工作大多发生在动画开始前，意味着在 CPU 上执行的工作基本不会影响帧率(frame rate)，但是会延迟动画的开始执行时间，使得 app 响应用户事件不够及时。

以下 CPU operation 会影响动画开始执行时间：

+ Layout calculations——在 view 将要展示或被修改时需要计算所有 subviews 的 frame，如果 view hierarchy 非常复杂，这将是一个比较耗时的过程，当使用了 iOS6 的 autolayout 特性时，情况更为糟糕。
+ Lazy view loading——为了节省内存、提高 app 的启动速度，直到 viewcontroller 的 view 第一次要显示到屏幕上时才会加载它。当 view 的显示需要大量计算，甚至 IO 操作时(如 nib file，image，db 等)，将会严重影响viewcontroller 的加载速度。
+ Core Graphics drawing——如果自定义 view 实现了`drawRect：`或`drawLayer:inContext:`方法将会引起大量的额外工作，影响性能。
+ Image decompression——png、jpg 等格式的 image 为了减少文件大小都进行了压缩，在 image 显示到屏幕前必须解压，这将是一项耗时的操作。

# Offscreen Rendering 
___________________________________

Offscreen Rendering(又称Offscreen drawing，即离屏渲染)发生在某些特效无法通过直接绘制到屏幕上获取而需要先在 offscreen image context上绘制时。CPU-based drawing、GPU-based drawing 都可能引起 Offscreen Rendering，出现 Offscreen Rendering 时，需要为 offscreen drawing context 分配额外的内存，并且在 onscreen drawing context 与 offscreen drawing context 间切换，这些对性能都有较大的影响。常见的引起 Offscreen Rendering 的原因有：
+ 使用 Core Graphics(任何以 CG 为前缀的类) 绘制；
+ 实现了`drawRect:`或`drawLayer:inContext:`方法，即使方法体为空；
+ CALayer 的 `shouldRasterize` 属性设为 YES；
+ CALayer 使用了 mask(setMaskToBounds) 或 shadow(setShadow*);
+ 将任何文本显示到屏幕上，包括 Core Text；
+ Group opacity(UIViewGroupOpacity)。

其中 Core Graphics、`drawRect：`、`drawLayer:inContext:`以及 Core Text 的绘制(Offscreen Rendering)由 CPU 执行。而其他情况由 GPU 执行，当 OpenGL renderer 绘制 layer 时，可能由于某些特殊的子层，需要将它们合成到一个单独的 buffer 里面(offscreen context)。虽然 GPU 在图形渲染等操作上比 CPU 要快，但对于 GPU 来说，从 onscreen drawing context 切换到 offscreen drawing context 是一个昂贵的操作(it must flush its pipelines and barrier)。因此，对于某些会引起 GPU 离屏渲染的简单绘制操作来说，GPU 因离屏渲染引起的开销可能比直接通过 Core Graphics 在 CPU 上绘制的开销还要大。 如：对于层级复杂的 view，你可能会考虑使用[CALayer setShouldRasterize:]来缓存渲染结果，但其引起的离屏开销不一定比直接通过 Core Graphics 绘制小，唯一的方法就是测试！

当我们通过 Core Graphics 绘制并将 result image 展示在需要 offscreen rendering 的 layer 上时就会同时遇到这两种类型的 offscreen rendering，如：通过`–[CALayer renderInContext:]`获取截屏并将该截屏显示在一个带 shadow 的 layer 上时。

另外，shouldRasterize 与 masking、shadows、edge antialiasing以及group opacity有很大的不同。后述任何一种情况触发时，由于没有 cache，在每一帧上都会发生 offscreen rendering。rasterization虽然会引起 offscreen rendering，但只要 rasterized layer 没有修改，rasterization 结果将被 cache，并在每帧中被重复利用。

简单地说就是，shouldRasterize引起的离屏渲染只会发生一次(layer 被修改除外)，而其他情况触发的离屏渲染在每一帧中都会发生一次。因此，可以将两者结合在一起，如使用 layer mask 的同时将 shouldRasterize 设为 YES，当然前提是该 layer 的内容不经常发生变化。

使用 Instrument 的*Color Offscreen-Rendered Yellow*选项可以检测哪些layer 发生了 offscreen rendering。

# Rasterization
______________________________________

如果将 CALayer 的 `shouldRasterize` 属性设为 YES，该 layer 及其 sublayer 的内容将被 offscreen rendering 到一张 flat image 上，最终在屏幕上该 rasterized image 显示在其对应 layer 的位置上，同时 rasterized image 会被 cache 在内存中。
![](/img/bitmapcache.jpg)
由于需要 offscreen rendering 以及 cache rasterized image，rasterization 有一定的性能开销。尤其需要注意的是，针对内容经常变化的 layer，要慎用该特性，因为 layer 的内容一旦改变，之前 cache 的 rasterization 结果失效，需要重新 rasterization。我们可以通过Instrument 的 *Color Hits Green and Misses Red* 选项检测rasterized image cache是否经常被修改。

另外，在使用 `shouldRasterize` 属性时，需要设置 `rasterizationScale` 属性值为屏幕的 scale。

在使用 layer 的 opacity 的属性时，为了 sublayer 能正常显示，需要 UIViewGroupOpacity 的配合，但UIViewGroupOpacity会引起性能上的损耗。一种替代的方法是先将该 layer 光栅化，再应用 opacity。

# drawRect:
_______________________________________

重载 view 的 `drawRect:` 方法可以实现 custom drawing，但这将带来严重的性能消耗(即使重载的 `drawRect:` 方法体为空)。

因为一旦重载了 `drawRect:` 方法，系统就要为其分配一个新的 backing image 并且会引起 offscreen rendering。因此，尽量不要重载 `drawRect:` 方法。在 XCode 为 UIView 的子类生成的 `drawRect:` 方法模板中含有这样的提示:
> An empty implementation adversely affects performance during animation.

![](/img/notdrawrect.png)
![](/img/drawrect.png)

![](/img/usecanotdrawrect.jpg)

**在开发过程中，我们要尽量少重载 drawRect:方法，尤其不应该存在方法体为空的 drawRect: 方法。**

# image
_______________________________________

在 iOS 上常用的图片格式有 png、jpg：
+ png——采用的是无损压缩方式，图片文件较大，但解压速度快，只能用 CPU 解压，XCode 对 app bundle 中的 png 图片在编译时会做优化处理，常用于 UI elements；
+ jpg——采用的是有损压缩，图片文件相对较小，解压速度慢，CPU、GPU 都能解压，常用于 photo。

简单地说，png 图片文件大，加载速度相对较慢，但解压快；jpg 图片文件小，加载速度快，但由于其采用的压缩算法复杂，解压过程也复杂，相对较慢。

iOS 为了节省内存，对加载的 image 通常会延迟解压，直到该 image 需要显示。但这样处理，在我们要将 image draw 到屏幕上时存在性能问题，因为在真正 draw 之前还要花时间去解压。因此，有时需要提前解压 image。
 
为了避免延迟解压，我们可以使用 UIImage 的`+imageName:` 方法加载图片，该方法会立即对加载的图片进行解压(UIImage其他的图片加载方法不会解压image，如：`+imageWithContentsOfFile:`)。但，`+imageName:` 方法只能用于加载 app resources bundle 中的图片。
 
通过将 image 赋值给 layer 的 `contents` 属性或 UIImageView 的 `image` 属性也能立即解压 image，但他们对图片的解压发生在 main thread。
 
第三种解压方法是通过 ImageIO framework 实现：
![](/img/imageio.png)
通过设置`kCGImageSourceShouldCache`选项，可以强制立即解压 image，并保存解压后的结果。
 
最后一种方式是通过 UIKit 加载图片，但立即将其 draw 到 CGContext 上，image 在 draw 之前是必须要解压的，通过该方式解压的一个优势是解压过程可以在 background thread 中进行。
![](/img/backgrounddecodeimage.png)
 
### UIImageView vs. -[UIImage draw...]

将 image 绘制到屏幕上通常有两种方式：通过 UIImageView 控件以及直接使用 UIImage 的 draw 方法。

Apple 推荐使用 UIImageView 方式显示图片，原因主要有三点：
+ CALayer 能直接从 CGImage 中获取 bitmap；
+ 充分利用 GPU；
+ bitmap 有 cache。

![](/img/uiimageviewandraw.jpg)
![](/img/imageviewandlayer.png)
![](/img/imageviewvsimagedraw.png)
![](/img/CPUAndGpuimage.png) 
 
# 影响性能的其他因素
_______________________________________

#### Transparent 

如果要绘制的 layer 是透明的，对于 GPU 来说，有大量额外的工作要做，在合成最终bitmap 时需要针对透明 layer 的每个像素计算叠加效果。这对于滑动时的性能来说是大忌。在代码中经常会出现：`xxView.backgroundColor = [UIColor clearColor];`，其实这就意味着 xxView 具有透明背景！通过 Instrument 的**Color Blended Layers**选项可以检测哪些 layer 是透明的。

#### Misalignment

对于 view 的 frame 中的成员存在小数值时，可能会出现像素不对齐的问题(在2x 屏幕中0.5不会出现misalignment问题，因为1个点对应2像素)。当出现misalignment时，GPU 需要从 source texture 中 blending 多个像素点的值来生成一个像素。通过 Instrument 的**Color Misaligned Images**选项可以检测misalignment。


# Measure, Measure, Measure, Don’t Guess
________________________________________

app 的性能问题只能通过工具(Instrument)去测试，找到性能瓶颈的真正原因，切忌通过猜测，盲目地优化，这很可能适得其反。

并且性能问题不像 memory leaks，找到了就能明确的进行修复，性能问题即使找到原因，在修改后还要通过 Instrument 测试是否达到优化效果。

由于 Instrument 的使用有详细的说明文档，在此不再重复。

关于 app 性能优化还有很多话题可以讨论，今天就到此结束。

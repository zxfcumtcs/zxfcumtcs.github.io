---
layout: post
title: 深入浅出 Flutter Framework 之 PipelineOwner
date: 2020-12-05 21:10:51
tags:
- flutter
- 移动开发
- 跨平台
---
本文是『 深入浅出 Flutter Framework 』系列文章的第六篇，详细介绍了 PipelineOwner 在整个 Rendering Pipeline 中是如何协助『 RenderObject Tree 』、『 RendererBinding』以及『 Window』完成 UI 刷新。

<!--more-->
©原创文章，转载请注明出处！

本系列文章将深入 Flutter Framework 内部逐步去分析其核心概念和流程，主要包括：
[『 深入浅出 Flutter Framework 之 Widget 』](https://zxfcumtcs.github.io/2020/05/01/deepinto-flutter-widget/)
[『 深入浅出 Flutter Framework 之 BuildOwner 』](https://zxfcumtcs.github.io/2020/05/16/deepinto-flutter-buildowner/)
[『 深入浅出 Flutter Framework 之 Element 』](https://zxfcumtcs.github.io/2020/05/17/deepinto-flutter-element/)
[『 深入浅出 Flutter Framework 之 PaintingContext 』](https://zxfcumtcs.github.io/2020/05/23/deepinto-flutter-paintingcontext/)
[『 深入浅出 Flutter Framework 之 Layer 』](https://zxfcumtcs.github.io/2020/06/07/deepinto-flutter-layer/)
『 深入浅出 Flutter Framework 之 PipelineOwner 』
[『 深入浅出 Flutter Framework 之 RenderObejct 』](https://zxfcumtcs.github.io/2021/03/27/deepinto-flutter-renderobject/)
[『 深入浅出 Flutter Framework 之自定义渲染型 Widget 』](https://zxfcumtcs.github.io/2021/08/28/deepinto-flutter-custom-renderobjectwidget/)

# Overview
_________________
`PipelineOwner`在 Rendering Pipeline 中起到重要作用：
+ 随着 UI 的变化而不断收集『 Dirty Render Objects 』
+ 随之驱动 Rendering Pipeline 刷新 UI

简单讲，`PipelineOwner`是『RenderObject Tree』与『RendererBinding』间的桥梁，在两者间起到沟通协调的作用。

# 关系
____________________
![](/img/PipelineOwner.PNG)
如上图：
+ `RendererBinding`创建并持有`PipelineOwner`实例，Code1-第`8~12`行
+ 同时，`RendererBinding`会创建『RenderObject Tree』的根节点，即：RenderView，并将其赋值给`PipelineOwner#rootNode`，Code1-第`13~24`行
+ 在『RenderObject Tree』构建过程中，每插入一个新节点，就会将`PipelineOwner`实例 attach 到该节点上，即『RenderObject Tree』上所有结点共享同一个`PipelineOwner`实例，Code2-第`4`行

```dart Code1-RendererBinding#init
// 代码有删减，下同
//
mixin RendererBinding {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    _pipelineOwner = PipelineOwner(
      onNeedVisualUpdate: ensureVisualUpdate,
      onSemanticsOwnerCreated: _handleSemanticsOwnerCreated,
      onSemanticsOwnerDisposed: _handleSemanticsOwnerDisposed,
    );
    initRenderView();
    addPersistentFrameCallback(_handlePersistentFrameCallback);
  }

  void initRenderView() {
    renderView = RenderView(configuration: createViewConfiguration(), window: window);
    renderView.prepareInitialFrame();
  }

  set renderView(RenderView value) {
    _pipelineOwner.rootNode = value;
  }
```

```dart Code2-RenderObject#adoptChild
  void adoptChild(covariant AbstractNode child) {
    child._parent = this;
    if (attached)
      child.attach(_owner);
    redepthChild(child);
  }

  void attach(covariant Object owner) {
    _owner = owner;
  }
```
> `RendererBinding`是 mixin，其背后真实的类是`WidgetsFlutterBinding`

如上所述，正常情况下在 Flutter 运行过程中只有一个`PipelineOwner`实例，并由`RendererBinding`持有，用于管理所有『 on-screen RenderObjects 』。
然而，如果有『 off-screen RenderObjects 』，则可以创建新的`PipelineOwner`实例来管理它们。
『on-screen PipelineOwner』与 『 off-screen PipelineOwner 』完全独立，后者需要创建者自己维护、驱动。

> `mixin RendererBinding on BindingBase, ServicesBinding, SchedulerBinding, GestureBinding, SemanticsBinding, HitTestable`
> 如上，`RendererBinding`要求其附属类 mixin `BindingBase`、`ServicesBinding`、`SchedulerBinding`、`GestureBinding`、`SemanticsBinding`以及`HitTestable`，为了描述方便，文本提到的`RendererBinding`上的方法也可能来自于其他几个 Binding。

# 驱动
____________________
## Dirty RenderObjects

Render Object 有4种『 Dirty  State』 需要 PipelineOwner 去维护：
+ Needing Layout：Render Obejct 需要重新 layout
+ Needing Compositing Bits Update：Render Obejct 合成标志位(Compositing)有变化
+ Needing Paint：Render Obejct 需要重新绘制
+ Needing Semantics：Render Object 辅助信息有变化

![](/img/RenderObject_PipelineOwner_RendererBinding.png)

如上图：
+ 当 RenderObject 需要重新 layout 时，调用`markNeedsLayout`方法，该方法会将当前 RenderObject 加入 `PipelineOwner#_nodesNeedingLayout`或传给父节点去处理；
+ 当 RenderObject 的 Compositing Bits 有变化时，调用`markNeedsCompositingBitsUpdate`方法，该方法会将当前 RenderObject 加入 `PipelineOwner#_nodesNeedingCompositingBitsUpdate`或传给父节点去处理；
+ 当 RenderObject 需要重新 paint 时，调用`markNeedsPaint`方法，该方法会将当前 RenderObject 加入`PipelineOwner#_nodesNeedingPaint`或传给父节点处理；
+ 当 RenderObject 的辅助信息(Semantics)有变化时，调用`markNeedsSemanticsUpdate`方法，该方法会将当前 RenderObject 加入 `PipelineOwner#_nodesNeedingSemantics`或传给父节点去处理

上述就是 PipelineOwner 不断收集『 Dirty RenderObjects 』的过程。

> RenderObject 内部的逻辑会在后续文章中详细分析。

## Request Visual Update
上述4个`markNeeds*`方法，除了`markNeedsCompositingBitsUpdate`，其他方法最后都会调用`PipelineOwner#requestVisualUpdate`。
之所以`markNeedsCompositingBitsUpdate`不会调用`PipelineOwner#requestVisualUpdate`，是因为其不会单独出现，一定是伴随其他3个之一一起出现的。

如上图，随着`PipelineOwner#requestVisualUpdate`->`RendererBinding#scheduleFrame`->`Window#scheduleFrame`调用链，UI 需要刷新的信息最终传递到了 Engine 层。
具体讲，`Window#scheduleFrame`主要是向 Engine 请求在下一帧刷新时调用`Window#onBeginFrame`以及`Window#onDrawFrame`方法。
> `Window#onBeginFrame`、`Window#onDrawFrame`本质上是 RendererBinding 向其注入的两个回调(`_handleBeginFrame`、`_handleDrawFrame`)：
> ```dart Code3-SchedulerBinding
   void ensureFrameCallbacksRegistered() {
    window.onBeginFrame ??= _handleBeginFrame;
    window.onDrawFrame ??= _handleDrawFrame;
  }
```
## Handle Draw Frame
![](/img/RendererBinding_handleDrawFrame.png)
如上图，Engine 在接收到 UI 需要更新后，在下一帧刷新时会调用`Window#onDrawFrame`，通过提前注册好的`PersistentFrameCallback`，最终调用到`RendererBinding#drawFrame`方法：
```dart Code4-RendererBinding#drawFrame
  void drawFrame() {
    pipelineOwner.flushLayout();
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();
    renderView.compositeFrame(); // this sends the bits to the GPU
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
  }
```
如上，`RendererBinding#drawFrame`依次调用`PipelineOwner`的`flushLayout`、`flushCompositingBits`、`flushPaint`以及`flushSemantics`方法，来处理对应状态下的 RenderObject。

### Flush Layout
```dart Code5-PipelineOwner#flushLayout
  void flushLayout() {
    while (_nodesNeedingLayout.isNotEmpty) {
      final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
      _nodesNeedingLayout = <RenderObject>[];
      for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {
        if (node._needsLayout && node.owner == this)
          node._layoutWithoutResize();
      }
    } 
  }
```
首先，`PipelineOwner`对于收集到的『 Needing Layout RenderObjects 』按其在『 RenderObject Tree 』上的深度升序排序，主要是为了避免子节点重复 Layout (因为父节点 layout 时，也会递归地对子树进行 layout)；
其次，对排好序的且满足条件的 RenderObjects 依次调用`_layoutWithoutResize`来执行 layout 操作。
> 在父节点 layout 完成时，其所有子节点也 layout 完成，它们的`_needsLayout`标志会被置为`flase`，因此在 Code5 中需要第`6`行的判断，避免重复 layout。

### Flush Compositing Bits
```dart Code6-PipelineOwner#flushCompositingBits
  void flushCompositingBits() {
    _nodesNeedingCompositingBitsUpdate.sort((RenderObject a, RenderObject b) => a.depth - b.depth);
    for (RenderObject node in _nodesNeedingCompositingBitsUpdate) {
      if (node._needsCompositingBitsUpdate && node.owner == this)
        node._updateCompositingBits();
    }
    _nodesNeedingCompositingBitsUpdate.clear();
  }
```
同理，先对『 Needing Compositing Bits RenderObjects 』排序，再调用`RenderObjects#_updateCompositingBits`

### Flush Paint
```dart Code7-PipelineOwner#flushPaint
  void flushPaint() {
    final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
    _nodesNeedingPaint = <RenderObject>[];
    // Sort the dirty nodes in reverse order (deepest first).
    for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => b.depth - a.depth)) {
      if (node._needsPaint && node.owner == this) {
        if (node._layer.attached) {
            PaintingContext.repaintCompositedChild(node);
        } 
      }
    }
  }
```
对于 Paint 操作来说，父节点需要用到子节点绘制的结果，故子节点需要先于父节点被绘制。
因此，不同于前两个 flush 操作，此时需要对『 Needing Paint RenderObjects 』按深度降序排序。
如下图，在[深入浅出 Flutter Framework 之 PaintingContext](https://zxfcumtcs.github.io/2020/05/23/deepinto-flutter-paintingcontext/)一文中详细分析了从`PipelineOwner#flushPaint`到`PaintingContext`内部操作的过程，在此不再赘述。
![](/img/PaintingPipeline.png)

### Flush Semantics
```dart Code8-PipelineOwner#flushSemantics
  void flushSemantics() {
    final List<RenderObject> nodesToProcess = _nodesNeedingSemantics.toList()
      ..sort((RenderObject a, RenderObject b) => a.depth - b.depth);
    _nodesNeedingSemantics.clear();
    for (RenderObject node in nodesToProcess) {
      if (node._needsSemanticsUpdate && node.owner == this)
        node._updateSemantics();
    }
    _semanticsOwner.sendSemanticsUpdate();
  }
```

Flush Semantics 所做操作与 Flush Layout 完全相似，不再赘述。

至此，PipelineOwner 相关的内容就介绍完了。

# 小结
PipelineOwner 作为『 RenderObject Tree』与『 RendererBinding/Window』间的沟通协调桥梁，在整个 Rendering Pipeline 中起到重要作用。
在 Flutter 应用生命周期内，不断收集『 Dirty RenderObjects 』并及时通知 Engine。
在帧刷新时，通过来自 RendererBinding 的回调依次处理收集到的：
+ 『 Needing Layout RenderObjects 』
+ 『 Needing Compositing Bits Update RenderObjects 』
+ 『 Needing Paint RenderObjects 』
+ 『 Needing Semantics RenderObjects 』

最终完成 UI 的刷新。
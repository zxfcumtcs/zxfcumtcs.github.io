---
layout: post
title: 深入浅出 Flutter Framework 之 Layer
date: 2020-06-07 12:01:28
tags:
- flutter
- 移动开发
- 跨平台
---
本文是『 深入浅出 Flutter Framework 』系列文章的第五篇，对 Layer 的类层级结构以及 Layer 的状态管理进行了简要的分析介绍。

<!--more-->
©原创文章，转载请注明出处！

本系列文章将深入 Flutter Framework 内部逐步去分析其核心概念和流程，主要包括：
[『 深入浅出 Flutter Framework 之 Widget 』](https://zxfcumtcs.github.io/2020/05/01/deepinto-flutter-widget/)
[『 深入浅出 Flutter Framework 之 BuildOwner 』](https://zxfcumtcs.github.io/2020/05/16/deepinto-flutter-buildowner/)
[『 深入浅出 Flutter Framework 之 Element 』](https://zxfcumtcs.github.io/2020/05/17/deepinto-flutter-element/)
[『 深入浅出 Flutter Framework 之 PaintingContext 』](https://zxfcumtcs.github.io/2020/05/23/deepinto-flutter-paintingcontext/)
『 深入浅出 Flutter Framework 之 Layer 』
[『 深入浅出 Flutter Framework 之 PipelineOwner 』](https://zxfcumtcs.github.io/2020/12/05/deepinto-flutter-pipelineowner/)
『 深入浅出 Flutter Framework 之 RenderObejct 』
『 深入浅出 Flutter Framework 之 Binding 』
『 深入浅出 Flutter Framework 之 Rendering Pipeline 』
『 深入浅出 Flutter Framework 之 自定义 Widget 』

# Overview
_________________
前面的文章中我们介绍过在 Flutter build、layout、render 过程中会生成 3 棵树：
+ Element Tree
+ RenderObject Tree
+ Layer Tree

可以说 Layer Tree 是 Flutter Framework 最终的输出产物，之后的流程就进入到 Flutter Engine 了。
![](/img/Element_RenderObject_LayerTree.png)
如上图：
+ 在`build`过程中，由 Element Tree 生成 RenderObject Tree (在 [深入浅出 Flutter Framework 之 Element](https://zxfcumtcs.github.io/2020/05/17/deepinto-flutter-element/) 一文中介绍过只有 RenderObject_Element 才会有对应的 RenderObject)
+ 在`paint`阶段，由 RenderObject Tree 生成 Layer Tree (在 [深入浅出 Flutter Framework 之 PaintingContext](https://zxfcumtcs.github.io/2020/05/23/deepinto-flutter-paintingcontext/) 一文中介绍过只有当`RenderObject#isRepaintBoundary`为`true`时才会生成独立的 Layer 节点)

# 分类
_________________
![](/img/Layer.png)

如上图，`Layer`是抽象基类，其内部实现了基本的 Layer Tree 的管理逻辑以及对渲染结果复用的控制逻辑。
具体的 Layer 大致可以分为2类：
+ Container Layer：正如其名，作为 Layer 容器，用于管理一组 Layers，是唯一可以拥有 child layer 的 Layer；
+ 非 Container Layer：真正用于承载渲染结果的 layer，在 Layer Tree 中属于叶结点，如：`PictureLayer`承载的是图片的渲染结果，`TextureLayer`承载的是纹理的渲染结果。

# 状态管理
_________________
抽象基类`Layer`一个非常重要的职责就是管理 Layer Tree 的状态
简单讲，就是控制什么情况下可以复用 engine 在前一帧渲染的结果，什么情况下需要刷新，即需要 engine 重新渲染。当然，这是出于性能考虑，避免因不必要的渲染操作而浪费资源。

## EngineLayer
每个 Layer 实例都有一个与之对应的`EngineLayer`实例，其属于 engine 层范畴，对 framework 来说是个黑盒。可以简单理解 EngineLayer 为 engine 渲染的结果。
![](/img/EngineLayer.png)

在 [深入浅出 Flutter Framework 之 PaintingContext](https://zxfcumtcs.github.io/2020/05/23/deepinto-flutter-paintingcontext/) 一文中介绍过的`SceneBuilder`类有一系列的`push`方法 (如：`pushOffset`、`pushClipRect`等)，这些方法的返回值即为`EngineLayer`实例。

如：`ColorFilterLayer#addToScene`方法在调用`SceneBuilder#pushColorFilter`方法时就保存了其返回的`engineLayer`：
```Dart
  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    engineLayer = builder.pushColorFilter(colorFilter, oldLayer: _engineLayer);
    addChildrenToScene(builder, layerOffset);
    builder.pop();
  }
```

## needsAddToScene
我们知道，Layer上的内容需要借助`SceneBuilder`生成`Scene`，之后才能被渲染在屏幕上。
这一过程被称之为`addToScene`。

在`Layer`内有一个非常重要的变量：`_needsAddToScene`，用于记录该 Layer 自上次渲染后(`addToScene`)是否发生了变化。
即，该 layer 背后的 EngineLayer 是否可以复用。
> Whether this layer has any changes since its last call to [addToScene].

在`Layer`刚初始化时，`_needsAddToScene`为`true`，在第一次调用`addToScene`后置为`false`，之后有几种情况可能会被再次置为`true`，表明该 layer需要 engine 重新渲染，如下图所示：
+ 自身发生了变化，如`PictureLayer#picture`、`TransformLayer#transform`被重新赋值；
+ 子节点有增删；
+ 子节点的`needsAddToScene`变为`true`；

![](/img/needsAddToScene.png)

> `Layer#alwaysNeedsAddToScene`为`true`时，表示该 layer 在每帧刷新时都需要重新渲染。其对`_needsAddToScene`的影响如下：
> ```Dart
  @protected
  @visibleForTesting
  void updateSubtreeNeedsAddToScene() {
    _needsAddToScene = _needsAddToScene || alwaysNeedsAddToScene;
  }
```

## addToScene
![](/img/addToScene.png)
`addToScene`方法可以说是`Layer`中最重要的方法之一，用于将 layer 送入 engin 进行渲染。
由具体的子类去实现该方法，Container 类型的 Layer 与非 Container 类型的 Layer 在实现上还是有较大区别，细节在此不再赘述，感兴趣的同学可以看看源码。

# 串起来
_________________
![](/img/LayerBuild.png)

如上面这张长长的图，现在要描述的一切都始于`RenderView.compositeFrame()`：
> RenderView 是 RenderObject Tree 的根节点，在每帧刷新时都会调用其`compositeFrame`方法去合成新的帧
> RenderView 对应的 Layer 是ContainerLayer

几个关键点：
+ `ContainerLayer.buildScene()`方法首先去更新`needsAddToScene`标志位 (对 Layer Tree 进行深度遍历)，子节点的值会影响父节点 (子节点有更新时，父节点肯定也要刷新)；
+ 之后，调用`ContainerLayer.addToScene()`方法，该方法会对子节点进行递归操作；
+ 注意，在`ContainerLayer._addToSceneWithRetainedRendering()`方法中，当`_ needsAddToScene`为`false`且`_engineLayer!=nil`时直接复用上次的渲染结果。

好了，今天就先到这里了！

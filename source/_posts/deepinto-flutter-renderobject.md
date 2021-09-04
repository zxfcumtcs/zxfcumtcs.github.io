---
layout: post
title: 深入浅出 Flutter Framework 之 RenderObject
date: 2021-03-27 16:14:34
tags:
- flutter
- 移动开发
- 跨平台
---
本文是『 深入浅出 Flutter Framework 』系列文章的第七篇。

<!--more-->
©原创文章，转载请注明出处！

本系列文章将深入 Flutter Framework 内部逐步去分析其核心概念和流程，主要包括：
[『 深入浅出 Flutter Framework 之 Widget 』](https://zxfcumtcs.github.io/2020/05/01/deepinto-flutter-widget/)
[『 深入浅出 Flutter Framework 之 BuildOwner 』](https://zxfcumtcs.github.io/2020/05/16/deepinto-flutter-buildowner/)
[『 深入浅出 Flutter Framework 之 Element 』](https://zxfcumtcs.github.io/2020/05/17/deepinto-flutter-element/)
[『 深入浅出 Flutter Framework 之 PaintingContext 』](https://zxfcumtcs.github.io/2020/05/23/deepinto-flutter-paintingcontext/)
[『 深入浅出 Flutter Framework 之 Layer 』](https://zxfcumtcs.github.io/2020/06/07/deepinto-flutter-layer/)
[『 深入浅出 Flutter Framework 之 PipelineOwner 』](https://zxfcumtcs.github.io/2020/12/05/deepinto-flutter-pipelineowner/)
『 深入浅出 Flutter Framework 之 RenderObejct 』
[『 深入浅出 Flutter Framework 之自定义渲染型 Widget 』](https://zxfcumtcs.github.io/2021/08/28/deepinto-flutter-custom-renderobjectwidget/)

# Overview
_________________
可以说，RenderObject 在整个 Flutter Framework 中属于核心对象，其职责概括起来主要有三点：
『 Layout 』、『 Paint 』、『 Hit Testing 』。
然而，RenderObject 是抽象类，具体工作由子类去完成。
<img src="/img/RenderObjectClassDiagram.png" width="80%" height="80%">
如上图，`RenderSliver`、`RenderBox`、`RenderView`以及`RenderAbstractViewport`是在 Flutter Framework 中定义的4个子类：
+ `RenderSliver`，『 Sliver-Widget 』对应的 Base RenderObject；
+ `RenderBox`，除『 Sliver-Widget 』外几乎所有常见 Render-Widget 对应的 Base RenderObject；
+ `RenderView`，是一种特殊的 Render Object，是『 RenderObect Tree 』的根节点；
+ `RenderAbstractViewport`，主要用于『 Scroll-Widget 』。

![](/img/RenderObject_XMind.png)
如上图，概括了`RenderObject`中与 Layout、Paint 相关的主要属性与方法(也是本文将要讨论的主要内容)。
其中，用虚线框起来的是`RenderObject`子类需要重写的方法。
正如上面所说，`RenderObject`是抽象类，它即没有明确使用哪种坐标系 (Cartesian coordinates or Polar coordinates)，也没有指定使用哪种排版算法 (width-in-height-out or constraint-in-size-out)。

> `RenderBox`采用笛卡尔坐标系、排版算法是 constraint-in-size-out，即根据父节点传递的排版约束来计算 Size。

下面我们从`RenderObject`生命周期中几个关键节点展开介绍：创建、布局、渲染。
> 本文所示代码基于 Flutter 1.12.13，同时对代码做了精简处理，以便突出所要讨论的重点。

# 创建
_________________
在[『深入浅出 Flutter Framework 之 Element』](https://zxfcumtcs.github.io/2020/05/17/deepinto-flutter-element/)一文中我们介绍过，当 RenderObjectElement 被挂载(mount) 到『 Element Tree 』上时，会创建对应的 Render Object 。
同时，会将其 attach 到『 RenderObject Tree 』上，也就是在『 Element Tree 』创建过程中『 RenderObject Tree 』也被逐步创建出来：
```Dart
// RenderObjectElement
void mount(Element parent, dynamic newSlot) {
  super.mount(parent, newSlot);
  _renderObject = widget.createRenderObject(this);
  attachRenderObject(newSlot);
  _dirty = false;
}
```
# 布局
_________________
在[『深入浅出 Flutter Framework 之 PipelineOwner』](https://zxfcumtcs.github.io/2020/12/05/deepinto-flutter-pipelineowner/)一文中介绍过，当 RenderObject 需要(重新)布局时调用`markNeedsLayout`方法，从而被`PipelineOwner`收集，并在下一帧刷新时触发 Layout 操作。

## markNeedsLayout调用场景
+ Render Object 被添加到『 RenderObject Tree 』;
+ 子节点 adopt、drop、move；
+ 由子节点的`markNeedsLayout`方法传递调用；
+ Render Object 自身与布局相关的属性发生变化，如`RenderFlex`的排版方向有变化时：
```Dart
set direction(Axis value) {
  assert(value != null);
  if (_direction != value) {
    _direction = value;
    markNeedsLayout();
  }
}
```
## Relayout Boundary
若某个 Render Object 的布局变化不会影响到其父节点的布局，则该 Render Object 就是『 Relayout Boundary 』。
Relayout Boundary 是一项重要的优化措施，可以避免不必要的 re-layout。

当某个 Render Object 是 Relayout Boundary 时，会切断 layout dirty 向父节点传播，即下一帧刷新时父节点无需 re-layout。
<img src="/img/RelayoutBoundary.png" width="50%" height="50%">
如上图：
+ 若`RD`节点出现 layout dirty，由于其自身、其父节点`RA`、`RRoot`都不是 Relayout Boundary，最终 layout dirty 传播到根节点`RenderView`，导致整颗『 RenderObject Tree 』重新布局；
+ 若`RF`节点出现 layout dirty，由于其父节点`RB`为 Relayout Boundary，layout dirty 传播到`RB`即结束，最终需要重新布局的只有`RB`、`RF`两个节点；
+ 若`RG`节点出现 layout dirty，由于其自身就是 Relayout Boundary，最终需要重新布局的只有`RG`自己。

那么，具体来说要成为 Relayout Boundary 需要满足什么条件呢？
```Dart
void layout(Constraints constraints, { bool parentUsesSize = false }) {
  RenderObject relayoutBoundary;
  if (!parentUsesSize || sizedByParent || constraints.isTight || parent is! RenderObject) {
    relayoutBoundary = this;
  } 
  else {
    relayoutBoundary = (parent as RenderObject)._relayoutBoundary;
  }

  if (!_needsLayout && constraints == _constraints && relayoutBoundary == _relayoutBoundary) {
    return;
  }

  if (_relayoutBoundary != null && relayoutBoundary != _relayoutBoundary) {
    visitChildren(_cleanChildRelayoutBoundary);
  }
  _relayoutBoundary = relayoutBoundary;
}
```
以上是`RenderObject#layout`中与 Relayout Boundary 有关的代码，
可知满足以下 4 个条件之一即可成为 Relayout Boundary：
+ `parentUsesSize`为`false`，即父节点在 layout 时不会使用当前节点的 size 信息(也就是当前节点的排版信息对父节点无影响)；
+ `sizedByParent`为`true`，即当前节点的 size 完全由父节点的 constraints 决定，即若在两次 layout 中传递下来的 constraints 相同，则两次 layout 后当前节点的 size 也相同；
+ 传给当前节点的 constraints 是紧凑型 (Tight)，其效果与`sizedByParent`为`true`是一样的，即当前节点的 layout 不会改变其 size，size 由 constraints 唯一确定；
+ 父节点不是 RenderObject 类型(主要针对根节点，其父节点为nil)。

> 每个 Render Object 都有一个`relayoutBoundary`属性，其值要么等于自己，要么等于父节点的`relayoutBoundary`。

## markNeedsLayout
```Dart
  void markNeedsLayout() {
    if (_needsLayout) {
      return;
    }
    if (_relayoutBoundary != this) {
      markParentNeedsLayout();
    } 
    else {
      _needsLayout = true;
      if (owner != null) {
        owner._nodesNeedingLayout.add(this);
        owner.requestVisualUpdate();
      }
    }
  }
```
从上述代码可以看到：
+ 若当前 Render Object 不是 Relayout Boundary，则 layout 请求向上传播给父节点(即 layout 范围扩大到父节点，这是一个递归过程，直到遇到 Relayout Boundary)；
+ 若当前 Render Object 是 Relayout Boundary，则 layout 请求到该节点为此，不会传播到其父节点。

> 通过 PipelineOwner 收集所有 layout dirty 节点，并在下一帧刷新时批量处理，而不是实时更新 dirty layout，从而避免不必要的重复 re-layout。

## layout
```Dart
  void layout(Constraints constraints, { bool parentUsesSize = false }) {
    RenderObject relayoutBoundary;
    if (!parentUsesSize || sizedByParent || constraints.isTight || parent is! RenderObject) {
      relayoutBoundary = this;
    } 
    else {
      relayoutBoundary = (parent as RenderObject)._relayoutBoundary;
    }

    if (!_needsLayout && constraints == _constraints && relayoutBoundary == _relayoutBoundary) {
      return;
    }
    _constraints = constraints;
    if (_relayoutBoundary != null && relayoutBoundary != _relayoutBoundary) {
      visitChildren(_cleanChildRelayoutBoundary);
    }
    _relayoutBoundary = relayoutBoundary;

    if (sizedByParent) {
      performResize();
    }

    performLayout();
    markNeedsSemanticsUpdate();
    
    _needsLayout = false;
    markNeedsPaint();
  }
```
`layout`方法是触发 Render Object 更新布局信息的主要入口点。
一般情况下，由父节点调用子节点的`layout`方法来更新其整体布局。
`RenderObject`的子类不应重写该方法，可按需重写`performResize`或/和`performLayout`方法。

当前 Render Object 的布局受到`layout`方法参数`constraints`的约束。
<img src="/img/LayputDataFlow.png" width="60%" height="60%">
如上图，『 Render Object Tree 』的 layout 是一次深度优先遍历的过程。
优先 layout 子节点，之后 layout 父节点。
父节点向子节点传递 layout constraints，子节点在 layout 时需遵守这些约束。
作为子节点 layout 的结果，父节点在 layout 时可以使用子节点的 size。

在上述`layout`代码第`19~21`行，若`sizedByParent`为`true`，则调用`performResize`来计算该 Render Object 的 size。

`sizedByParent`为`true`的 Render Object 需重写`performResize`方法，在该方法中仅根据`constraints`来计算 size。
如`RenderBox`中定义的`performResize`的默认行为：取`constraints`约束下的最小 size：
```Dart
  @override
  void performResize() {
    // default behavior for subclasses that have sizedByParent = true
    size = constraints.smallest;
    assert(size.isFinite);
  }
```

若父节点 layout 依赖子节点的 size，在调用`layout`方法时需将`parentUsesSize`参数设为`true`。
因为，在这种情况下若子节点 re-layout 导致其 size 发生变化，需要及时通知父节点，父节点也需要 re-layout (即 layout dirty 范围需要向上传播)。
这一切都是通过上节介绍过的 Relayout Boundary 来实现。

## performLayout
本质上，`layout`是一个模板方法，具体的布局工作由`performLayout`方法完成。
`RenderObject#performLayout`是一个抽象方法，子类需重写。

关于`performLayout`有几点需要注意：
+ 该方法由`layout`方法调用，在需要 re-layout 时应调用`layout`方法，而不是`performLayout`；
+ 若`sizedByParent`为`true`，则该方法不应改变当前 Render Object 的 size ( 其 size 由`performResize`方法计算)；
+ 若`sizedByParent`为`false`，则该方法不仅要执行 layout 操作，还要计算当前 Render Object 的 size；
+ 在该方法中，需对其所有子节点调用`layout`方法以执行所有子节点的 layout 操作，如果当前 Render Object 依赖子节点的布局信息，需将`parentUsesSize`参数设为`true`。

```Dart
// RenderFlex
void performLayout() {
  RenderBox child = firstChild;
  while (child != null) {
    final FlexParentData childParentData = child.parentData;
    BoxConstraints innerConstraints = BoxConstraints(minHeight: constraints.maxHeight, maxHeight: constraints.maxHeight);
    child.layout(innerConstraints, parentUsesSize: true);
    child = childParentData.nextSibling;
  }
  
  size = constraints.constrain(Size(idealSize, crossSize));
  
  child = firstChild;
  while (child != null) {
    final FlexParentData childParentData = child.parentData;
    double childCrossPosition = crossSize / 2.0 - _getCrossSize(child) / 2.0;
    childParentData.offset = Offset(childMainPosition, childCrossPosition);
    child = childParentData.nextSibling;
  }
}
```
上述代码片段截取自`RenderFlex`，可以看到它大概做了3件事：
+ 对所有子节点逐个调用`layout`方法；
+ 计算当前 Render Object 的 size；
+ 将与子节点布局有关的信息存储到相应子节点的`parentData`中。

> `RenderFlex`继承自`RenderBox`，是常用的`Row`、`Column`对应的 Render Object。

# 绘制
_________________
与`markNeedsLayout`相似，当 Render Object 需要重新绘制 (paint dirty) 时通过`markNeedsPaint`方法上报给`PipelineOwner`。
## markNeedsPaint
```Dart
  void markNeedsPaint() {
    if (isRepaintBoundary) {
      assert(_layer is OffsetLayer);
      if (owner != null) {
        owner._nodesNeedingPaint.add(this);
        owner.requestVisualUpdate();
      }
    } 
    else if (parent is RenderObject) {
      final RenderObject parent = this.parent;
      parent.markNeedsPaint();
    } 
    else {
      if (owner != null)
        owner.requestVisualUpdate();
    }
  }
```
`markNeedsPaint`内部逻辑与`markNeedsLayout`都非常相似：
+ 若当前 Render Object 是 Repaint Boundary，则将其添加到`PipelineOwner#_nodesNeedingPaint`中，Paint request 也随之结束；
+ 否则，Paint request 向父节点传播，即需要 re-paint 的范围扩大到父节点(这是一个递归的过程)；
+ 有一个特例，那就是『 Render Object Tree 』的根节点，即 RenderView，它的父节点为 nil，此时只需调用`PipelineOwner#requestVisualUpdate`即可。

> `PipelineOwner#_nodesNeedingPaint`收集的所有 Render Object 都是 Repaint Boundary。

## Repaint Boundary
对 Repaint Boundary 从上面`markNeedsPaint`的实现可略知一二，
若某 Render Object 是 Repaint Boundary，其会切断 re-Paint request 向父节点传播。

更直白点，Repaint Boundary 使得 Render Object 可以独立于父节点进行绘制，
否则当前 Render Object 会与父节点绘制在同一个 layer 上。
总结一下，Repaint Boundary 有以下特点：
+ 每个 Repaint Boundary 都有一个独属于自己的 OffsetLayer (ContainerLayer)，其自身及子孙节点的绘制结果都将 attach 到以该 layer 为根节点的子树上；
+ 每个 Repaint Boundary 都有一个独属于自己的 PaintingContext (包括背后的 Canvas)，从而使得其绘制与父节点完全隔离开。

<img src="/img/RepaintBoundary.png" width="100%" height="100%">
如上图，由于`Root`/`RA`/`RC`/`RG`/`RI`是 Repaint Boundary，所以它们都有对应的 OffsetLayer。
同时，由于每个 Repaint Boundary 都有属于自己的 PaintingContext，所以它们都有对应的 PictureLayer，用于呈现具体的绘制结果。
对于那些不是 Repaint Boundary 的节点，将会绘制到最近的 Repaint Boundary 祖先节点提供的 PictureLayer 上。

> Repaint Boundary 会影响兄弟节点的绘制，如由于`RC`是 Repaint Boundary，导致`RB`、`RD`被绘制到不同的 PictureLayer 上。

> 实现中，『 Layer Tree 』往往会比上图所示更复杂，由于每个 Render Object 在绘制过程中都可以自主引入更多的 layer。

Repaint Boundary 的目标是优化性能，但从上面的讨论我们也可以看出 Repaint Boundary 会增加『 Layer Tree 』的复杂度。
因此，Repaint Boundary 并不是越多越好。
只适用于那些需要频繁重绘的场景，如视频。
> Flutter Framework 为开发者预定义了`RepaintBoundary` widget，其继承自`SingleChildRenderObjectWidget`，在有需要时我们可以通过`RepaintBoundary` widget 来添加 Repaint Boundary。

## Paint
```Dart
void paint(PaintingContext context, Offset offset) { }
```
抽象基类`RenderObject`中的`paint`是个空方法，需要子类重写。
`paint`方法主要有2项任务：
+ 当前 Render Object 本身的绘制，如：`RenderImage`，其`paint`方法主要职责就是 image 的渲染
```Dart
  void paint(PaintingContext context, Offset offset) {
    paintImage(
      canvas: context.canvas,
      rect: offset & size,
      image: _image,
      ...
    );
  }
```
+ 绘制子节点，如：`RenderTable`，其`paint`方法主要职责是依次对每个子节点调用`PaintingContext#paintChild`方法进行绘制：
```Dart
  void paint(PaintingContext context, Offset offset) {
    for (int index = 0; index < _children.length; index += 1) {
      final RenderBox child = _children[index];
      if (child != null) {
        final BoxParentData childParentData = child.parentData;
        context.paintChild(child, childParentData.offset + offset);
      }
    }
  }
```
## 串起来
![](/img/RenderObject-Paint-Line.png)
下面我们将整个绘制流程串一串，如上图：
### PipelineOwner#flushPaint
当新一帧开始时，会触发`PipelineOwner#flushPaint`方法，进而对收集的所有『 paint-dirty Render Obejcts 』进行 re-paint：
```Dart
void flushPaint() {
    try {
      final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
      _nodesNeedingPaint = <RenderObject>[];
      
      // Sort the dirty nodes in reverse order (deepest first).
      for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => b.depth - a.depth)) {
        assert(node._layer != null);
        if (node._needsPaint && node.owner == this) {
          PaintingContext.repaintCompositedChild(node);
        }
      }
    } 
  }
```
### PaintingContext#repaintCompositedChild
`PaintingContext#repaintCompositedChild`是一个非常重要的方法，一是为『 paint-dirty Render Obejcts 』创建 Layer (若没有)、二是为 RenderObject 的绘制准备`context`并发起绘制流程：
```Dart
  static void _repaintCompositedChild(
    RenderObject child, {
    bool debugAlsoPaintedParent = false,
    PaintingContext childContext,
  }) {
    assert(child.isRepaintBoundary);
    OffsetLayer childLayer = child._layer;
    if (childLayer == null) {
      child._layer = childLayer = OffsetLayer();
    } 
    else {
      assert(childLayer is OffsetLayer);
      childLayer.removeAllChildren();
    }
    
    // 在正常的绘制流程中通过参数传递过来的 childContext 都是 null
    // 因此，此处总是会创建新的 PaintingContext
    //
    childContext ??= PaintingContext(child._layer, child.paintBounds);
    child._paintWithContext(childContext, Offset.zero);

    childContext.stopRecordingIfNeeded();
  }
```
### RenderObject#_paintWithContext
`RenderObject#_paintWithContext`逻辑比较简单，主要就是调用`paint`方法；
```Dart
  void _paintWithContext(PaintingContext context, Offset offset) {
    if (_needsLayout)
      return;
    _needsPaint = false;
    paint(context, offset);
  }
```
`paint`方法正如上节所讲，通过`Canvas#draw***`系列方法进行具体的绘制操作，对于子节点(若有)调用`PaintingContext#paintChild`方法；
### PaintingContext#paintChild
对于当前绘制子节点，若是 Repaint Boundary，则需要在独立的 layer 上进行绘制，
否则直接调用子节点的`_paintWithContext`方法在当前上下文(paint context)中绘制：
```Dart
  void paintChild(RenderObject child, Offset offset) {
    if (child.isRepaintBoundary) {
      stopRecordingIfNeeded();
      _compositeChild(child, offset);
    } 
    else {
      child._paintWithContext(this, offset);
    }
  }
```

下面重点分析对 Repaint Boundary 的处理，如上`paintChild`所示，首先会调用`PaintingContext#stopRecordingIfNeeded`来停止当前的绘制工作：
```Dart
  void stopRecordingIfNeeded() {
    if (!_isRecording)
      return;
    _currentLayer.picture = _recorder.endRecording();
    _currentLayer = null;
    _recorder = null;
    _canvas = null;
  }
```
`stopRecordingIfNeeded`首先将当前绘制结果保存到`_currentLayer.picture`中，之后对上下文做一些清理。
将`_currentLayer`置为 null，初看感觉难于理解，保存在`_currentLayer.picture`中的绘制结果且不是就丢了？
其实，`_currentLayer`早在`_startRecording`方法中就被添加到『 Layer Tree 』上，这里只是将`PaintingContext`中的引用置为 null 而以。
```Dart
  void _startRecording() {
    assert(!_isRecording);
    _currentLayer = PictureLayer(estimatedBounds);
    _recorder = ui.PictureRecorder();
    _canvas = Canvas(_recorder);
    _containerLayer.append(_currentLayer);
  }
```
由于已经将`_canvas`置为 null 了，下次使用时会触发对`_startRecording`方法的调用：
```Dart
  Canvas get canvas {
    if (_canvas == null)
      _startRecording();
    return _canvas;
  }
```
### PaintingContext#_compositeChild
在`_compositeChild`中，通过`repaintCompositedChild`对子节点发起新一轮的绘制，并将绘制结果(`child._layer`)添加到『 Layer Tree 』中：
```Dart
  void _compositeChild(RenderObject child, Offset offset) {
    assert(!_isRecording);
    assert(child.isRepaintBoundary);
    assert(_canvas == null || _canvas.getSaveCount() == 1);

    repaintCompositedChild(child, debugAlsoPaintedParent: true);

    final OffsetLayer childOffsetLayer = child._layer;
    childOffsetLayer.offset = offset;
    appendLayer(child._layer);
  }
```
### 小结
整个绘制过程其实就是对『 RenderObject Tree 』进行深度遍历的过程；
Repaint Boundary 会独立于父节点进行绘制，因此需要独立的 ContainerLayer(OffsetLayer) 以及 PaintingContext。

# 跑起来
_________________
下面我们简单的分析一下，一个 Flutter App 是怎么跑起来的。

```Dart
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..scheduleAttachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```
`runApp`是 Flutter 项目的入口点，完成 Binding 的初始化、attach root widget、安排首帧的调度。

![](/img/RenderView-create.png)
如上图，在`RendererBinding#initInstances`中会创建『 RenderObject Tree 』的根节点：`RenderView`。

如下代码，在`RenderView`初始化过程中为首帧渲染作准备，『 Layer Tree 』根节点也是在此过程中创建的。
```Dart
  // RenderView
  //
  void prepareInitialFrame() {
    scheduleInitialLayout();
    scheduleInitialPaint(_updateMatricesAndCreateNewRootLayer());
  }
  
  Layer _updateMatricesAndCreateNewRootLayer() {
    _rootTransform = configuration.toMatrix();
    final ContainerLayer rootLayer = TransformLayer(transform: _rootTransform);
    rootLayer.attach(this);
    return rootLayer;
  }
```

```Dart
  // RenderObject
  //
  void scheduleInitialLayout() {
    _relayoutBoundary = this;
    owner._nodesNeedingLayout.add(this);
  }
  
  void scheduleInitialPaint(ContainerLayer rootLayer) {
    _layer = rootLayer;
    owner._nodesNeedingPaint.add(this);
  }
```

如下图，在`WidgetsFlutterBinding#scheduleAttachRootWidget`->`WidgetsFlutterBinding#attachRootWidget`调用链上会创建 Root Widget：`RenderObjectToWidgetAdapter`。

![](/img/attachRootWidget.png)

在`RenderObjectToWidgetAdapter#attachToRenderTree`方法中『 Element Tree 』的根节点`RenderObjectToWidgetElement`被创建出来。
至此：
+  Root Widget(`RenderObjectToWidgetAdapter`)
+ 『 Element Tree 』的根节点(`RenderObjectToWidgetElement`)
+ 『 RenderObject Tree 』的根节点(`RenderView`)
+ 『 Layer Tree 』的根节点(`TransformLayer`)

创建完成。

```Dart
  // RenderObjectToWidgetAdapter
  RenderObjectToWidgetElement<T> attachToRenderTree(BuildOwner owner, [ RenderObjectToWidgetElement<T> element ]) {
	  owner.lockState(() {
	    element = createElement();
	    element.assignOwner(owner);
	  });
	  
	  owner.buildScope(element, () {
	    element.mount(null, null);
	  });
	  
	  // This is most likely the first time the framework is ready to produce
	  // a frame. Ensure that we are asked for one.
	  SchedulerBinding.instance.ensureVisualUpdate();

    return element;
  }
  
  RenderObjectToWidgetElement<T> createElement() => RenderObjectToWidgetElement<T>(this);
```
上述代码第`9`行，对 Root Element 进行挂载(mount)。
随之，『 Element Tree 』被逐步创建出来。
> 关于『 Element Tree 』创建过程更详细的分析可参看[『深入浅出 Flutter Framework 之 Element』](https://zxfcumtcs.github.io/2020/05/17/deepinto-flutter-element/)一文

![](/img/RenderObjectToWidgetElement-mount.png)
```Dart
  // RenderObjectToWidgetAdapter
  void _rebuild() {
    _child = updateChild(_child, widget.child, _rootChildSlot);
  }
```
随着『 Element Tree 』的构建，『 RenderObject Tree 』也被创建出来。
```Dart
  // RenderObjectElement
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _renderObject = widget.createRenderObject(this);
    attachRenderObject(newSlot);
    _dirty = false;
  }
```

# 小结
_________________
本文对 RenderObject 生命周期中几个重要节点：创建、布局、绘制等作了简要介绍。
对一些重要概念，如：Relayout Boundary、Repaint Boundary 也进行了较详细的分析。
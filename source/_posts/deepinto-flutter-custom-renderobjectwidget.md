---
layout: post
title: 深入浅出 Flutter Framework 之自定义渲染型 Widget
date: 2021-08-28 11:07:58
tags:
- flutter
- 移动开发
- 跨平台
---
本文是『 深入浅出 Flutter Framework 』系列文章的第八篇，也是收官之作。通过自定义渲染型 Widget，我们一步步地实现了一个评分组件。

<!--more-->
©原创文章，转载请注明出处！

本系列文章将深入 Flutter Framework 内部逐步去分析其核心概念和流程，主要包括：
[『 深入浅出 Flutter Framework 之 Widget 』](https://zxfcumtcs.github.io/2020/05/01/deepinto-flutter-widget/)
[『 深入浅出 Flutter Framework 之 BuildOwner 』](https://zxfcumtcs.github.io/2020/05/16/deepinto-flutter-buildowner/)
[『 深入浅出 Flutter Framework 之 Element 』](https://zxfcumtcs.github.io/2020/05/17/deepinto-flutter-element/)
[『 深入浅出 Flutter Framework 之 PaintingContext 』](https://zxfcumtcs.github.io/2020/05/23/deepinto-flutter-paintingcontext/)
[『 深入浅出 Flutter Framework 之 Layer 』](https://zxfcumtcs.github.io/2020/06/07/deepinto-flutter-layer/)
[『 深入浅出 Flutter Framework 之 PipelineOwner 』](https://zxfcumtcs.github.io/2020/12/05/deepinto-flutter-pipelineowner/)
[『 深入浅出 Flutter Framework 之 RenderObejct 』](https://zxfcumtcs.github.io/2021/03/27/deepinto-flutter-renderobject/)
『 深入浅出 Flutter Framework 之自定义渲染型 Widget 』

# Overview
_________________
本文作为『 深入浅出 Flutter Framework 』系列文章的收官之作，为了对本系列文章所涉重点内容的回顾和总结，动手实现一个渲染型 Widget (Render-Widget)。
如下图，最终成品是一个评分组件 (源码已上传 Github: [Score](https://github.com/zxfcumtcs/Score))：
![](/img/Score.gif)

通过前面系列文章的介绍，我们知道 Render-Widget 大致有三类：
+ 作为『 Widget Tree 』的叶节点，也是最小的 UI 表达单元，一般继承自`LeafRenderObjectWidget`；
+ 有一个子节点 ( Single Child )，一般继承自`SingleChildRenderObjectWidget`；
+ 有多个子节点 ( Multi Child )，一般继承自`MultiChildRenderObjectWidget`。

Widget 间的继承关系如下图：
![](/img/Widget.png)

Widget、Element、RenderObject 间的对应关系如下：
![](/img/Widget-Element-RenderObject.png)
> 其中，Element 与 RenderObject 间用的是虚线，因为它们间的对应关系是基于 RenderBox 系列下的一种建议 (不是强制)。
> Sliver 系列就不是基于`RenderBox`，而是`RenderSliver`。
> 通过`Render-Widget#createRenderObject`方法可以返回任意 RenderObject (如果你愿意)。

对于`RenderBox`系列来说，如果要自定义子类，根据自定义子类子节点模型的不同需要有不同的处理：
+ 自定义子类本身是『 Render Tree 』的叶子节点，一般直接继承自`RenderBox`；
+ 有一个子节点 (Single Child)，且子节点属于`RenderBox`系列：
	+ 如果其自身的 size 完全 match 子节点的 size，则可以选择继承自`RenderProxyBox`(如：`RenderOffstage`)；
	+ 如果其自身的 size 大于子节点的 size，则可以选择继承自`RenderShiftedBox`(如：`RenderPadding`)；
+ 有一个子节点 (Single Child)，但子节点不属于`RenderBox`系列，自定义子类可以 mixin `RenderObjectWithChildMixin`，其提供了管理一个子节点的模型；
+ 有多个子节点 (Multi Child)，自定义子类可以 mixin `ContainerRenderObjectMixin`、`RenderBoxContainerDefaultsMixin`，前者提供了管理多个子节点的模型，后者提供了基于`ContainerRenderObjectMixin`的一些默认实现。

下面，我们一步步地来实现上面提到的评分组件。

# Custom Leaf Render Widget
_________________
首先，我们来实现评分组件里的五星部分 (ScoreStar Widget)：
<img src="/img/Stars.png" width="40%" height="40%">

## LeafRenderObjectWidget
`ScoreStar`作为叶子节点，继承自`LeafRenderObjectWidget`，并实现了2个重要方法：`createRenderObject`、`updateRenderObject`：
```Dart
class ScoreStar extends LeafRenderObjectWidget {
  final Color backgroundColor;
  final Color foregroundColor;
  final double score;

  ScoreStar(this.backgroundColor, this.foregroundColor, this.score);

  @override
  RenderObject createRenderObject(BuildContext context) {
    return RenderScoreStar(backgroundColor, foregroundColor, score);
  }

  @override
  void updateRenderObject(BuildContext context, covariant RenderScoreStar renderObject) {
    renderObject
      ..backgroundColor = backgroundColor
      ..foregroundColor = foregroundColor
      ..score = score;
  }
}
```
其中，`updateRenderObject`方法会在 Widget re-build 时调用，用于更新复用的 Render Object 的属性。
在本例中，`score`会随着用户点击不同的区域而变化，就需要通过`updateRenderObject`方法来更新`RenderScoreStar#score`，以便刷新 UI。

## Leaf Render Object
从上一小节`ScoreStar#createRenderObject`可知，`ScoreStar` 对应的 Render Object 是`RenderScoreStar`。
`RenderScoreStar`继承自`RenderBox`。

如下代码：
+ 在 `socre setter` 中调用了`markNeedsPaint`方法，以便在`score`变化后及时 re-paint (由于 socre 变化不会引起 layout 变化，故此处只需调用`markNeedsPaint`，若会引起 layout 变化，则需要调用`markNeedsLayout`)；
+ 关于`sizedByParent`，在该例子中设为`true` or `false`都可以，因为`RenderScoreStar#size`完全由`constraints`决定：
	+ 从性能角度考虑，`sizedByParent`应设为`true`，以便满足`RepaintBoundary`的条件 (详情请参见[ [深入浅出 Flutter Framework 之 RenderObject](https://zxfcumtcs.github.io/2021/03/27/deepinto-flutter-renderobject/) ] )；
	+ 若`sizedByParent`设为`true`，需要重写`performResize`方法来计算 size，由于`RenderScoreStar`没有 layout 操作需要执行，故不需要重写`performLayout`；
	+ 若`sizedByParent`设为`false`，则需要重写`performLayout`，并在该方法中完成 size 的计算；
	+ `22~30`、`32~40`两个代码片段随便使用哪个都可以。
+  关于IntrinsicWidth/Height，若重写了`performLayout`方法，则进而需要重写以下四个方法：
	+ `double computeMaxIntrinsicWidth(double height)`：用于计算一个最小宽度(没错，是最小宽度)，在最终 size.width 超过该宽度时，也不会减少 size.height (如，对 text 排版，将 text 排成一行需要的最小宽度就是这里的 MaxIntrinsicWidth，因为再增加宽度也不会减少 text 的高度)；
	+ `double computeMinIntrinsicWidth(double height)`：排版需要的最小宽度，若小于这个宽度内容就会被裁剪；
	+ `computeMinIntrinsicHeight`、`computeMaxIntrinsicHeight`与上面介绍的`computeMinIntrinsicWidth`、`computeMaxIntrinsicWidth`类似，不再赘述；
	+ 在一些特殊 RenderObject 排版时才会用到这些方法，在此我们根据 constraints 简单计算了一下。
+ 为了响应点击事件，重写`hitTestSelf`方法，并返回`true`，表示该 Render Object 需要响应用户事件； 
+ 关于`paint`方法中五角星 ★★★★★ 的绘制：
	+  对于背景 ★★★★★，设置好 path 后，直接通过`context.canvas.drawPath`绘制即可；
	+  对于前景 ★★★★★，先通过`context.pushClipRect`对画布进行裁剪 ( rect.width 由 score 决定 )，再行绘制。

```
// 为了缩减篇幅，精简了部分代码
//
class RenderScoreStar extends RenderBox {
  Color _backgroundColor;
  ...

  Color _foregroundColor;
  ...

  double _score;
  double get score => _score;
  set score(double value) {
    _score = value;
    
    // score 变化时需要re-paint
    //
    markNeedsPaint();
  }

  RenderScoreStar(this._backgroundColor, this._foregroundColor, this._score);

  @override
  bool get sizedByParent => false;

  @override
  void performLayout() {
    double height = min(constraints.biggest.height, constraints.biggest.width / 5);
    height = max(height, constraints.smallest.height);
    size = Size(constraints.biggest.width, height);
  }

  // @override
  // bool get sizedByParent => true;
  //
  // @override
  // void performResize() {
  //   double height = min(constraints.biggest.height, constraints.biggest.width / 5);
  //   height = max(height, constraints.smallest.height);
  //   size = Size(constraints.biggest.width, height);
  // }

  @override
  double computeMaxIntrinsicWidth(double height) {
    return constraints.biggest.width;
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    double height = min(constraints.biggest.height, constraints.biggest.width / 5);
    height = max(height, constraints.smallest.height);

    return height;
  }

  @override
  bool hitTestSelf(Offset position) {
    return true;
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    void _backgroundStarPainter(PaintingContext context, Offset offset) {
      _starPainter(context, offset, backgroundColor);
    }

    void _foregroundStarPainter(PaintingContext context, Offset offset) {
      _starPainter(context, offset, foregroundColor);
    }

    _backgroundStarPainter(context, offset);
    context.pushClipRect(
        needsCompositing,
        offset,
        Rect.fromLTRB(0, 0, size.width * score / 5, size.height),
        _foregroundStarPainter
    );
  }

  void _starPainter(PaintingContext context, Offset offset, Color color) {
    Paint paint = Paint();
    paint.color = color;
    paint.style = PaintingStyle.fill;

    double radius = min(size.height / 2, size.width/ (2 * 5));

    Path path = Path();
    _addStarLine(radius, path);
    for (int i = 0; i < 4; i++) {
      path = path.shift(Offset(radius * 2, 0.0));
      _addStarLine(radius, path);
    }

    path = path.shift(offset);
    path.close();

    context.canvas.drawPath(path, paint);
  }

  void _addStarLine(double radius, Path path) {
    ...
  }
}
```
至此，`RenderScoreStar`基本完成，完整代码请参见 [ Github：[Score](https://github.com/zxfcumtcs/Score) ]

# 动态评分
_________________
如下图，我们希望评分组件不仅能展示分数，还能评分：
![](/img/Score_small.gif)

在 Flutter UI 中，一个重要的思想就是：『 组合 』。
为了实现上图所示效果，只需组合`StatefulWidget` +`ScoreStar`即可：
```Dart
typedef ScoreCallback = void Function(double score);

class Score extends StatefulWidget {
  final double score;
  final ScoreCallback callback;

  const Score({Key key, this.score = 0, this.callback}) : super(key: key);

  @override
  _ScoreState createState() => _ScoreState();
}

class _ScoreState extends State<Score> {

  double score;

  @override
  void initState() {
    super.initState();

    score = widget.score ?? 0;
  }

  @override
  void didUpdateWidget(Score oldWidget) {
    super.didUpdateWidget(oldWidget);

    score = widget.score ?? 0;
  }

  @override
  Widget build(BuildContext context) {
    void _changeScore(Offset offset) {
      Size _size = context.size;
      double offsetX = min(offset.dx, _size.width);
      offsetX = max(0, offsetX);

      setState(() {
        score = double.parse(((offsetX / _size.width) * 5).toStringAsFixed(1));
      });

      if (widget.callback != null) {
        widget.callback(score);
      }
    }

    return GestureDetector(
      child: ScoreStar(Colors.grey, Colors.amber, score),
      onTapDown: (TapDownDetails details) {
        _changeScore(details.localPosition);
      },
      onLongPressMoveUpdate:(LongPressMoveUpdateDetails details) {
        _changeScore(details.localPosition);
      },
    );
  }
}
```
代码比较简单，就不赘述了。
其中的关键还是上节介绍的`RenderScoreStar#hitTestSelf`需要返回`true`。

# Custom MultiChild RenderObject Widget
_________________
![](/img/Score.gif)
我们希望通过自定义 MultiChild RenderObject Widget 实现如上图所示的效果。
没错，就是加了一个显示分数的 Text。
> 本来，这完全没必要通过自定义 MultiChild RenderObject Widget 来实现，一般的 Widget 组合即可。
> 我们只是为了实践自定义 MultiChild RenderObject Widget 才这么做的。

## MultiChildRenderObjectWidget 
`RichScore`继承自`MultiChildRenderObjectWidget`。
在其初始化方法中，向父类传递了2个 children：`Score`、`Text`。
重写了`createRenderObject`方法，以便返回`RenderRichScore`实例。
由于`RenderRichScore`没有属性，故无需重写`updateRenderObject`方法。

```Dart
class RichScore extends MultiChildRenderObjectWidget {
  RichScore({
    Key key,
    double score,
    ScoreCallback callback,
  }) : super(
    key: key,
    children: [
      Score(score: score, callback: callback),
      Text('$score分', style: TextStyle(fontSize: 28)),
    ]
  );

  @override
  RenderObject createRenderObject(BuildContext context) {
    return RenderRichScore();
  }
}
```

## RichScoreParentData
还记得 ParentData 吗？
> 在[『 深入浅出 Flutter Framework 之 Widget 』](https://zxfcumtcs.github.io/2020/05/01/deepinto-flutter-widget/)中有过简单介绍。

对于含有子节点的 RenderObject，一般都需要自定义自己的 ParentData 子类，用于辅助 layout。
```Dart
class RichScoreParentData extends ContainerBoxParentData<RenderBox> {
  double scoreTextWidth;
}
```
`RichScoreParentData`继承自`ContainerBoxParentData`：
```Dart
/// Abstract ParentData subclass for RenderBox subclasses that want the
/// ContainerRenderObjectMixin.
///
/// This is a convenience class that mixes in the relevant classes with
/// the relevant type arguments.
abstract class ContainerBoxParentData<ChildType extends RenderObject> extends BoxParentData with ContainerParentDataMixin<ChildType> { }
```
`ContainerBoxParentData`是抽象类，但其 mixin`ContainerParentDataMixin`：
```Dart
/// Parent data to support a doubly-linked list of children.
mixin ContainerParentDataMixin<ChildType extends RenderObject> on ParentData {
  /// The previous sibling in the parent's child list.
  ChildType previousSibling;
  /// The next sibling in the parent's child list.
  ChildType nextSibling;
}
```
`ContainerParentDataMixin`在子节点间提供了双向链接的支持。

在`RichScoreParentData`中定义了唯一一个属性：`scoreTextWidth`，其作用在后面再介绍。

## MultiChild RenderObject
`RenderRichScore`继承自`RenderBox`并 minix 了`ContainerRenderObjectMixin`以及`RenderBoxContainerDefaultsMixin`：
+ 由于`RenderRichScore#size`受子节点的影响，即不完全由 Constraints 决定，故`sizedByParent`设为`false`，同时在调用子节点的`layout`方法时`parentUsesSize`参数需设为`true` (下面代码第`40`、`55`行)；
+ 由于其子节点 (`RenderScoreStar`)需要响应用户事件，故重写了`hitTestChildren`方法；
+ 在`performLayout`方法中，完成了所有子节点的排版、设置相应的 ParentData 并计算出了 size；
+ 对于有子节点的 RenderObject 需要重写`computeDistanceToActualBaseline`方法，这里我们用了`RenderBoxContainerDefaultsMixin`提供的默认实现；
+ `paint`方法的功能很简单，依次绘制每个子节点(`defaultPaint`由`RenderBoxContainerDefaultsMixin`提供)；
+ `setupParentData`用于给子节点设置`parentData`。

```Dart
class RenderRichScore extends RenderBox with ContainerRenderObjectMixin<RenderBox, RichScoreParentData>,
    RenderBoxContainerDefaultsMixin<RenderBox, RichScoreParentData>,
    DebugOverflowIndicatorMixin {

  RenderRichScore({
    List<RenderBox> children,
  }) {
    addAll(children);
  }

  @override
  bool get sizedByParent => false;

  final double horizontalSpace = 10;
  final double scoreTextWidthDifference = 10;

  @override
  bool hitTestChildren(BoxHitTestResult result, { Offset position }) {
    assert(childCount == 2);

    RenderBox scoreChild = firstChild;
    return scoreChild?.hitTest(result, position: position) ?? false;
  }

  @override
  void performLayout() {
    assert(childCount == 2);

    RenderBox scoreStarChild = firstChild;
    RenderBox scoreTextChild = lastChild;

    if (scoreStarChild == null || scoreTextChild == null) {
      size = constraints.smallest;
      return;
    }

    // infinity constraints
    //
    BoxConstraints descConstraints = BoxConstraints();
    scoreTextChild.layout(descConstraints, parentUsesSize: true);

    final RichScoreParentData descChildParentData = scoreTextChild.parentData as RichScoreParentData;
    double descWidth = descChildParentData.scoreTextWidth;
    if (descWidth == null) {
      descWidth = scoreTextChild.size.width + scoreTextWidthDifference;
      descChildParentData.scoreTextWidth = descWidth;
    }

    BoxConstraints scoreConstraints = BoxConstraints(
      minWidth: 0,
      maxWidth: max(constraints.maxWidth - descWidth - horizontalSpace, 0),
      minHeight: 0,
      maxHeight: constraints.maxHeight
    );
    scoreStarChild.layout(scoreConstraints, parentUsesSize: true);

    descChildParentData.offset = Offset(
      scoreStarChild.size.width + horizontalSpace,
      (scoreStarChild.size.height - scoreTextChild.size.height) / 2
    );

    if (constraints.isTight) {
      size = constraints.biggest;
    }
    else {
      double width = min(constraints.biggest.width, scoreStarChild.size.width + descWidth + horizontalSpace);
      width = max(constraints.smallest.width, width);

      double height = max(scoreStarChild.size.height, scoreTextChild.size.height);
      height = min(constraints.biggest.height, height);
      height = max(constraints.smallest.height, height);

      size = Size(width, height);
    }
  }
  
  ...

  @override
  double computeDistanceToActualBaseline(TextBaseline baseline) {
    return defaultComputeDistanceToFirstActualBaseline(baseline);
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    assert(childCount == 2);

    if (childCount != 2) {
      return;
    }

    defaultPaint(context, offset);
  }

  @override
  void setupParentData(RenderObject child) {
    if (child.parentData is! RichScoreParentData) {
      child.parentData = RichScoreParentData();
    }
  }
}
```

## RichScoreParentData#scoreTextWidth
上面我们提到`RichScoreParentData`有唯一一个属性：`scoreTextWidth`。
那么它的作用是啥呢？
根据`RenderRichScore`的排版算法，先计算 text 的宽度，★★★★★ 的宽度等于 constraints.biggest.width - textWidth。
这个算法有点小问题：
![](/img/bad_score.gif)
由于 textWidth 会因分数的不同，而有细微的差异，最终导致 ★★★★★ 有点闪烁。
为了解决这个问题，我们将 textWidth 的宽度固定为首次计算的 text 宽度+10，并将其存储在`RichScoreParentData`中(上述代码第`42~47`行)。
> 这种解决方法不一定是最好的，这里主要是演示一下 ParentData 的作用。

至此，自定义 MultiChild RenderObject 基本完成了。
# 小结
_________________
本文通过实现评分组件，逐步实践了如何自定义 Leaf Render Widget 以及 MultiChild Render Widget。
在这过程中，自定义了 Widget 以及 Render Object，但并没有涉及 Element。
原因是 Element 作为 Widget 与 Render Object 间的桥梁，逻辑相对内聚、独立。
当自定义 Widget 继承自`LeafRenderObjectWidget`、`SingleChildRenderObjectWidget`或`MultiChildRenderObjectWidget`时，一般不用自定义 Element。

自定义 Leaf Render Widget，一般需要以下步骤：
+ 自定义 Widget 继承自`LeafRenderObjectWidget`，并重写`createRenderObject`、`updateRenderObject`方法；
+ 自定义 Render Object 继承自`RenderBox`：
	+ 确定`sizedByParent`为`true` or `false`；
	+ 为`false`，重写`performLayout`方法，执行 layout 并计算 size；
	+ 为`true`，重写`performResize`方法计算 size、重写`performLayout`方法执行 layout (若需要)；
	+ 如果重写了`performLayout`方法，则需进一步重写`computeMax/MinIntrinsicWidth/Height`系列方法；
	+ 如需处理用户事件，重写`hitTestSelf`方法；
	+ 重写`paint`方法，完成最终的绘制。

自定义 MultiChild Render Widget，一般需要以下步骤：
+ 自定义 Widget 继承自`MultiChildRenderObjectWidget`，并重写`createRenderObject`、`updateRenderObject`方法；
+ 自定义 Render Object 继承自`RenderBox`，并 minix `ContainerRenderObjectMixin`、`RenderBoxContainerDefaultsMixin`：
	+ 确定`sizedByParent`为`true` or `false`；
	+ 为`false`，重写`performLayout`方法，对子节点逐个执行 layout 操作并计算 size；
	+ 为`true`，重写`performResize`方法计算 size、重写`performLayout`方法执行 layout；
	+ 如果重写了`performLayout`方法，则需进一步重写`computeMax/MinIntrinsicWidth/Height`系列方法；
	+ 重写`computeDistanceToActualBaseline`方法计算 baseline;
	+ 如需处理用户事件，重写`hitTestSelf`或/和`hitTestChildren`方法；
	+ 自定义 ContainerBoxParentData 子类，用于存储 layout 过程中需要的辅助信息；
	+ 重写`setupParentData`方法，为子节点设置 ParentData；
	+ 重写`paint`方法，对子节点逐个执行 paint 操作。
	
『 深入浅出 Flutter Framework 』系列文章至此全部完成！
这一系列文章围绕 Widget、Element 以及 RenderObject 展开讨论，对 Flutter Framework 有了一个简单的认识。
在此过程中对相关的 BuildOwner、PaintingContext、Layer 以及 PipelineOwner 等也进行了一定的讨论。
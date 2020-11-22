---
layout: post
title: 深入浅出 Flutter Framework 之 Widget
date: 2020-05-01 21:29:16
tags:
- flutter
- 移动开发
- 跨平台
---
本文是『 深入浅出 Flutter Framework 』系列文章的第一篇，主要以不同类型 Widget 的核心方法为切入点，对其展开详细分析。

<!--more-->
©原创文章，转载请注明出处！

# Overview
_________________
Flutter 作为一种新兴跨平台解决方案，自 2017 年 Google 在 I/O 大会上推出后，尤其是在 2018 年 I/O 大会上发布第一个预览版后，迅速引起移动开发者的广泛关注，并成为时下最热门的跨平台解决方案 ( 没有之一 ) ！

本系列文章将深入 Flutter Framework 内部逐步去分析其核心概念和流程，主要包括：
『 深入浅出 Flutter Framework 之 Widget 』
[『 深入浅出 Flutter Framework 之 BuildOwner 』](https://zxfcumtcs.github.io/2020/05/16/deepinto-flutter-buildowner/)
[『 深入浅出 Flutter Framework 之 Element 』](https://zxfcumtcs.github.io/2020/05/17/deepinto-flutter-element/)
[『 深入浅出 Flutter Framework 之 PaintingContext 』](https://zxfcumtcs.github.io/2020/05/23/deepinto-flutter-paintingcontext/)
[『 深入浅出 Flutter Framework 之 Layer 』](https://zxfcumtcs.github.io/2020/06/07/deepinto-flutter-layer/)
『 深入浅出 Flutter Framework 之 PipelineOwner 』
『 深入浅出 Flutter Framework 之 RenderObejct 』
『 深入浅出 Flutter Framework 之 Binding 』
『 深入浅出 Flutter Framework 之 Rendering Pipeline 』
『 深入浅出 Flutter Framework 之 自定义 Widget 』

其中，前 7 篇属于基础篇，分别介绍 Flutter 中几个最核心的概念。Rendering Pipeline 篇则是在此基础上从 Build、Layout 到 Paint 的流程将它们串起来，分析 Flutter UI 是如何创建的、如何更新的。最后，自定义 Widget 属于回顾、实践篇，分析自定义一个 Render Widget 至少需要哪些步骤。

> 如[下图](https://flutter.dev/docs/resources/technical-overview#layer-cakes-are-delicious)所示，Flutter 整体分为三层：Framework (dart)、Engine (C/C++)、Embedder (Platform)，上述文章主要集中在 Framework 这一层。
> ![](/img/flutter-layercake.png)



# Widget
_________________
> Everything’s a widget.

在开发 Flutter 应用过程中，接触最多的无疑就是`Widget`，是『描述』 Flutter UI 的基本单元，通过`Widget`可以做到：
+ 描述 UI 的层级结构 (通过`Widget`嵌套)；
+ 定制 UI 的具体样式 (如：`font`、`color`等)；
+ 指导 UI 的布局过程 (如：`padding`、`center`等)；
+ ...

Google 在设计`Widget`时，还赋予它一些鲜明的特点：
+ 声明式 UI —— 相对于传统 Native 开发中的命令式 UI，声明式 UI 有不少优势，如：开发效率显著提升、UI 可维护性明显加强等；

+ 不可变性 —— Flutter 中所有`Widget`都是不可变的(immutable)，即其内部成员都是不可变的(`final`)，对于变化的部分需要通过「Stateful Widget-State」的方式实现；

+ 组合大于继承 —— `Widget`设计遵循组合大于继承这一优秀的设计理念，通过将多个功能相对单一的`Widget`组合起来便可得到功能相对复杂的`Widget`。

在`Widget`类定义处有这样一段注释：
![](/img/Widget_Comment.png)
这段注释阐明了`Widget`的本质：**用于配置`Element`的，`Widget`本质上是 UI 的配置信息 (附带部分业务逻辑)。**
> 我们通常会将通过`Widget`描述的 UI 层级结构称之为「Widget Tree」，但与「Element Tree」、「RenderObject Tree」以及「Layer Tree」相比，实质上并不存在「Widget Tree」。为了描述方便，将 Widget 组合描述的 UI 层级结构称之为「Widget Tree」，也未尝不可。

## 分类
![](/img/Widget.png)
如上图所示，按照功能划分`Widget`大致可以分为 3 类：
+ *「Component Widget」* —— 组合类 Widget，这类 Widget 都直接或间接继承于`StatelessWidget`或`StatefulWidget`，上一小节提到过在 Widget 设计上遵循组合大于继承的原则，通过组合功能相对单一的 Widget 可以得到功能更为复杂的 Widget。平常的业务开发主要是在开发这一类型的 Widget；

+ *「Proxy Widget」* —— 代理类 Widget，正如其名，「Proxy Widget」本身并不涉及 Widget 内部逻辑，只是为「Child Widget」提供一些附加的中间功能。典型的如：`InheritedWidget`用于在「Descendant Widgets」间传递共享信息、`ParentDataWidget`用于配置「Descendant Renderer Widget」的布局信息；

+ *「Renderer Widget」* —— 渲染类 Widget，是最核心的`Widget`类型，会直接参与后面的「Layout」、「Paint」流程，无论是「Component Widget」还是「Proxy Widget」最终都会映射到「Renderer Widget」上，否则将无法被绘制到屏幕上。这 3 类 Widget 中，**只有「Renderer Widget」有与之一一对应的「Render Object」**。

## 核心方法源码分析
下面，我们重点介绍各类型 Widget 的核心方法，以便更好地理解 Widget 是如何参与整个 UI 的构建过程。

### Widget
`Widget`，所有 Widget 的基类。

<div style="width: 400px; margin: auto">![](/img/Widget_Method.png)</div>


如上图所示，在 `Widget`基类中有 3 个重要的方法 (属性)：
+ *Key key* —— 在同一父节点下，用作兄弟节点间的唯一标识，主要用于控制当 Widget 更新时，对应的 Element 如何处理 (是更新还是新建)。若某 Widget 是其「Parent Widget」唯一的子节点时，一般不用设置 key；
> GlobalKey 是一类较特殊的 key，在介绍 Element 时会附带介绍。

+ *Element createElement()* —— 每个`Widget`都有一个与之对应的`Element`，由该方法负责创建，`createElement`可以理解为设计模式中的*工厂方法*，具体的`Element`类型由对应的`Widget`子类负责创建；

+ *static bool canUpdate(Widget oldWidget, Widget newWidget)* —— 是否可以用 new widget 修改前一帧用 old widget 生成的 Element，而不是创建新的 Element，`Widget`类的默认实现为：2个`Widget`的`runtimeType`与`key`都相等时，返回`true`，即可以直接更新 (key 为 null 时，认为相等)。
> 上述更新流程，同样在介绍 Element 时会重点分析。

### StatelessWidget
无状态-组合型 Widget，由其`build`方法描述组合 UI 的层级结构。在其生命周期内状态不可变。
![](/img/StatelessWidget_Comment.png)

<div style="width: 400px; margin: auto">![](/img/StatelessWidget_Method.png)</div>

> ps: 对于有父子关系的类，在子类中只会介绍新增或有变化的方法

+ *StatelessElement createElement()* ——「Stateless Widget」对应的 Element 为`StatelessElement`，一般情况下`StatelessWidget`子类不必重写该方法，即子类对应的 Element 也是`StatelessElement`；

+ *Widget build(BuildContext context)* —— 算是 Flutter 体系中的核心方法之一，以『声明式 UI』的形式描述了该组合式 Widget 的 UI 层级结构及样式信息，也是开发 Flutter 应用的主要工作『场所』。该方法在 3 种情况下被调用：
	+ Widget 第一次被加入到 Widget Tree 中 (更准确地说是其对应的 Element 被加入到 Element Tree 时，即 Element 被挂载『mount』时)；
	+ 「Parent Widget」修改了其配置信息；
	+ 该 Widget 依赖的「Inherited Widget」发生变化时。

当「Parent Widget」或 依赖的「Inherited Widget」频繁变化时，`build`方法也会频繁被调用。因此，提升`build`方法的性能就显得十分重要，Flutter 官方给出了几点建议：
+ *减少不必要的中间节点，即减少 UI 的层级，*如：对于「Single Child Widget」，没必要通过组合「Row」、「Column」、「Padding」、「SizedBox」等复杂的 Widget 达到某种布局的目标，或许通过简单的「Align」、「CustomSingleChildLayout」即可实现。又或者，为了实现某种复杂精细的 UI 效果，不一定要通过组合多个「Container」，再附加「Decoration」来实现，通过 「CustomPaint」自定义或许是更好的选择；

+ *尽可能使用`const` Widget，*为 Widget 提供`const`构造方法；
	> 关于 const constructor 推荐 [Dart Constant Constructors](https://japhr.blogspot.com/2012/12/dart-constant-constructors.html) 看看这篇文章的评论。

+ 必要时，*可以将「Stateless Widget」重构成「Stateful Widget」，*以便可以使用「Stateful Widget」中一些特定的优化手法，如：缓存「sub trees」的公共部分，并在改变树结构时使用`GlobalKey`；

+ *尽量减小 rebuilt 范围，*如：某个 Widget 因使用了「Inherited Widget」，导致频繁 rebuilt，可以将真正依赖「Inherited Widget」的部分提取出来，封装成更小的独立 Widget，并尽量将该独立 Widget 推向树的叶子节点，以便减小 rebuilt 时受影响的范围。

### StatefulWidget
有状态-组合型 Widget，但要注意的是`StatefulWidget`本身还是不可变的，其可变状态存在于`State`中。
![](/img/Stateful_Comment.png)

<div style="width: 400px; margin: auto">![](/img/StatefulWidget_Method.png)</div>

+ *StatefulElement createElement()* ——「Stateful Widget」对应的 Element 为`StatefulElement`，一般情况下`StatefulWidget`子类不用重写该方法，即子类对应的Element 也是`StatefulElement`；

+ *State createState()* —— 创建对应的 State，该方法在`StatefulElement`的构造方法中被调用。可以简单地理解为当「Stateful Widget」被添加到 Widget Tree 时会调用该方法。
	```dart
	// 代码已精简处理(本文中其他代码会做同样的简化处理)
	StatefulElement(StatefulWidget widget)
      : _state = widget.createState(),
        super(widget) {
    _state._element = this;
    _state._widget = widget;
  }```

> 实际上是「Stateful Widget」对应的「Stateful Element」被添加到 Element Tree 时，伴随「Stateful Element」的初始化，`createState`方法被调用。从后文可知一个 Widget 实例可以对应多个 Element 实例 (也就是同一份配置信息 (Widget) 可以在 Element Tree 上不同位置配置多个 Element 节点)，因此，`createState`方法在「Stateful Widget」生命周期内可能会被调用多次。
> 另外，需要注意的是配有`GlobalKey`的 Widget 对应的 Element 在整个 Element Tree 中只有一个实例。

### State
> The logic and internal state for a 「Stateful Widget」.

State 用于处理「Stateful Widget」的业务逻辑以及可变状态。
由于其内部状态是可变的，故 State 有较复杂的生命周期：
![](/img/StateLifecycle.png)
如上图，State 的生命周期大致可以分为 8 个阶段：
+ 在对应的「Stateful Element」被挂载 (mount) 到树上时，通过`StatefulElement.constructor` --> `StatefulWidget.createState`创建 State 实例；
 > 从`StatefulElement.constructor`中的`_state._element = this;`可知，`State._emelent`指向了对应的 Element 实例，而我们熟知的`State.context`引用的就是这个`_element`：`BuildContext get context => _element;`。
 `State`实例与`Element`实例间的绑定关系一经确定，在整个生命周期内不会再变了 (**Element 对应的 Widget 可能会变，但对应的 State 永远不会变**)，期间，`Element`可以在树上移动，但上述关系不会变 (即「Stateful Element」是带着状态移动的)。

+ StatefulElement 在挂载过程中接着会调用`State.initState`，子类可以重写该方法执行相关的初始化操作 (此时可以引用`context`、`widget`属性)；

+ 同样在挂载过程中会调用`State.didChangeDependencies`，该方法在 State 依赖的对象 (如：「Inherited Widget」) 状态发生变化时也会被调用，*子类很少需要重写该方法，*除非有非常耗时不宜在`build`中进行的操作，因为在依赖有变化时`build`方法也会被调用；

+ 此时，State 初始化已完成，其`build`方法此后可能会被多次调用，在状态变化时 State 可通过`setState`方法来触发其子树的重建；

+ 此时，「element tree」、「renderobject tree」、「layer tree」已构建完成，完整的 UI 应该已呈现出来。此后因为变化，「element tree」中「parent element」可能会对树上该位置的节点用新配置 (Widget) 进行重建，当新老配置 (oldWidget、newWidget)具有相同的「runtimeType」&&「key」时，framework 会用 newWidget 替换 oldWidget，并触发一系列的更新操作 (在子树上递归进行)。同时，`State.didUpdateWidget`方法被调用，子类重写该方法去响应 Widget 的变化；
> 上述 3 棵树以及更新流程在后续文章中会有详细介绍

+ 在 UI 更新过程中，任何节点都有被移除的可能，State 也会随之移除，(如上一步中「runtimeType」||「key」不相等时)。此时会调用`State.deactivate`方法，由于被移除的节点可能会被重新插入树中某个新的位置上，故子类重写该方法以清理与节点位置相关的信息 (如：该 State 对其他 element 的引用)、同时，不应在该方法中做资源清理；
> 重新插入操作必须在当前帧动画结束之前

+  当节点被重新插入树中时，`State.build`方法被再次调用；

+  对于在当前帧动画结束时尚未被重新插入的节点，`State.dispose`方法被执行，State 生命周期随之结束，此后再调用`State.setState`方法将报错。子类重写该方法以释放任何占用的资源。

![](/img/State_Method.png)
至此，State 中的核心方法基本都已在上述过程中介绍了，下面重点看一下`setState`方法：
```dart
  void setState(VoidCallback fn) {
    assert(fn != null);
    assert(() {
      if (_debugLifecycleState == _StateLifecycle.defunct) {
        throw FlutterError.fromParts(<DiagnosticsNode>[...]);
      }
      if (_debugLifecycleState == _StateLifecycle.created && !mounted) {
        throw FlutterError.fromParts(<DiagnosticsNode>[...]);
      }
      return true;
    }());
    final dynamic result = fn() as dynamic;
    assert(() {
      if (result is Future) {
        throw FlutterError.fromParts(<DiagnosticsNode>[...]);
      }
      return true;
    }());
    _element.markNeedsBuild();
  }
```
从上述源码可以看到，关于`setState`方法有几点值得关注：
+ 在`State.dispose`后不能调用`setState` (第 4 行)；

+ 在 State 的构造方法中不能调用`setState` (第 7 行)；

+ `setState`方法的回调函数 (`fn`) 不能是异步的 (返回值为`Future`)，原因很简单，因为从流程设计上 framework 需要根据回调函数产生的新状态去刷新 UI (第 14 行)；

+ 通过`setState`方法之所以能更新 UI，是在其内部调用`_element.markNeedsBuild()`实现的 (具体过程在介绍 Element 时再详细分析)。

关于 State 最后再强调 2 点：
+ 若`State.build`方法依赖了自身状态会变化的对象，如：`ChangeNotifier`、`Stream`或其他可以被订阅的对象，需要确保在`initState`、`didUpdateWidget`、`dispose`等 3 方法间有正确的订阅 (subscribe) 与取消订阅 (unsubscribe) 的操作：
	+ 在`initState`中执行 subscribe；
	+ 如果关联的「Stateful Widget」与订阅有关，在`didUpdateWidget`中先取消旧的订阅，再执行新的订阅；
	+ 在`dispose`中执行 unsubscribe。


+ 在`State.initState`方法中不能调用`BuildContext.dependOnInheritedWidgetOfExactType`，但`State.didChangeDependencies`会随之执行，在该方法中可以调用。

### ParentDataWidget
`ParentDataWidget`以及下面要介绍的`InheritedElement`都继承自`ProxyWidget`，由于`ProxyWidget`作为抽象基类本身没有任何功能，故下面直接介绍`ParentDataWidget`、`InheritedElement`。
![](/img/ParentDataWidget_Comment.png)
`ParentDataWidget`作为 Proxy 型 Widget，其功能主要是为其他 Widget 提供`ParentData`信息。虽然其 child widget 不一定是 RenderObejctWidget 类型，但其提供的`ParentData`信息最终都会落地到 RenderObejctWidget 类型子孙 Widget 上。
> ParentData 是『parent renderobject』在 layout『child renderobject』时使用的辅助定位信息，详细信息会在介绍 RenderObject 时介绍。

```dart
  void attachRenderObject(dynamic newSlot) {
    assert(_ancestorRenderObjectElement == null);
    _slot = newSlot;
    _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
    _ancestorRenderObjectElement?.insertChildRenderObject(renderObject, newSlot);
    final ParentDataElement<RenderObjectWidget> parentDataElement = _findAncestorParentDataElement();
    if (parentDataElement != null)
      _updateParentData(parentDataElement.widget);
  }

  ParentDataElement<RenderObjectWidget> _findAncestorParentDataElement() {
    Element ancestor = _parent;
    while (ancestor != null && ancestor is! RenderObjectElement) {
      if (ancestor is ParentDataElement<RenderObjectWidget>)
        return ancestor;
      ancestor = ancestor._parent;
    }
    return null;
  }

  void _updateParentData(ParentDataWidget<RenderObjectWidget> parentData) {
    parentData.applyParentData(renderObject);
  }
```
上面这段代码来自`RenderObjectElement`，可以看到在其`attachRenderObject`方法第 6 行从祖先节点找`ParentDataElement`，如果找到就用其 Widget(ParentDataWidget) 中的 parentData 信息去设置 Render Obejct。在查找过程中如查到`RenderObjectElement` (第 13 行)，说明当前 RenderObject 没有 Parent Data 信息。
最终会调用到`ParentDataWidget.applyParentData(RenderObject renderObject)`，子类需要重写该方法，以便设置对应`RenderObject.parentData`。

<div style="width: 400px; margin: auto">![](/img/ParentDataWidget_Method.png)</div>

来看个例子，通常配合`Stack`使用的`Positioned`(继承自ParentDataWidget)：
```dart
  void applyParentData(RenderObject renderObject) {
    assert(renderObject.parentData is StackParentData);
    final StackParentData parentData = renderObject.parentData;
    bool needsLayout = false;

    if (parentData.left != left) {
      parentData.left = left;
      needsLayout = true;
    }
    ...
    if (parentData.width != width) {
      parentData.width = width;
      needsLayout = true;
    }
    ...
    if (needsLayout) {
      final AbstractNode targetParent = renderObject.parent;
      if (targetParent is RenderObject)
        targetParent.markNeedsLayout();
    }
  }
```
可以看到，`Positioned`在必要时将自己的属性赋值给了对应的`RenderObject.parentData` (此处是`StackParentData`)，并对「parent render object」调用`markNeedsLayout`(第 19 行)，以便重新 layout，毕竟修改了布局相关的信息。


```dart
abstract class ParentDataWidget<T extends RenderObjectWidget> extends ProxyWidget
```
如上所示，`ParentDataWidget`在定义上使用了泛型`<T extends RenderObjectWidget>`，其背后的含义是：
*从当前`ParentDataWidget`节点向上追溯形成的祖先节点链(『parent widget chain』)上，在 2 个`ParentDataWidget`类型的节点形成的链上至少要有一个『RenderObject Widget』类型的节点。因为一个『RenderObject Widget』不能接受来自 2 个及以上『ParentData Widget』的信息。*

<div style="width: 400px; margin: auto">![](/img/ParentDataWidget_Chain.png)</div>

### InheritedWidget
![](/img/InheritedWidget_Comment.jpg)
InheritedWidget 用于在树上向下传递数据。
通过`BuildContext.dependOnInheritedWidgetOfExactType`可以获取最近的「Inherited Widget」，需要注意的是通过这种方式获取「Inherited Widget」时，当「Inherited Widget」状态有变化时，会导致该引用方 rebuild。
> 具体原理在介绍 Element 时会详细分析。

通常，为了使用方便会「Inherited Widget」会提供静态方法`of`，在该方法中调用`BuildContext.dependOnInheritedWidgetOfExactType`。`of`方法可以直接返回「Inherited Widget」，也可以是具体的数据。

有时，「Inherited Widget」是作为另一个类的实现细节而存在的，其本身是私有的(外部不可见)，此时`of`方法就会放到对外公开的类上。最典型的例子就是`Theme`，其本身是`StatelessWidget`类型，但其内部创建了一个「Inherited Widget」：`_InheritedTheme`，`of`方法就定义在上`Theme`上：
```dart
  static ThemeData of(BuildContext context, { bool shadowThemeOnly = false }) {
    final _InheritedTheme inheritedTheme = context.dependOnInheritedWidgetOfExactType<_InheritedTheme>();

    return ThemeData.localize(theme, theme.typography.geometryThemeFor(category));
  }
```
该`of`方法返回的是`ThemeData`类型的具体数据，并在其内部首先调用了`BuildContext.dependOnInheritedWidgetOfExactType`。

我们经常使用的「Inherited Widget」莫过于`MediaQuery`，同样提供了`of`方法：
```dart
  static MediaQueryData of(BuildContext context, { bool nullOk = false }) {
    final MediaQuery query = context.dependOnInheritedWidgetOfExactType<MediaQuery>();
    if (query != null)
      return query.data;
    if (nullOk)
      return null;
  }
```

<div style="width: 400px; margin: auto">![](/img/InheritedWidget_Method.png)</div>

+ *InheritedElement createElement()* ——「Inherited Widget」对应的 Element 为`InheritedElement`，一般情况下`InheritedElement`子类不用重写该方法；

+ *bool updateShouldNotify(covariant InheritedWidget oldWidget)* —— 在「Inherited Widget」rebuilt 时判断是否需要 rebuilt 那些依赖它的 Widget；

如下是`MediaQuery.updateShouldNotify`的实现，在新老`Widget.data` 不相等时才 rebuilt 那依赖的 Widget。
```dart
bool updateShouldNotify(MediaQuery oldWidget) => data != oldWidget.data;
```
### RenderObjectWidget
真正与渲染相关的 Widget，属于最核心的类型，一切其他类型的 Widget 要渲染到屏幕上，最终都要回归到该类型的 Widget 上。

<div style="width: 400px; margin: auto">![](/img/RenderObjectWidget_Method.png)</div>

+ *RenderObjectElement createElement()* ——「RenderObject Widget」对应的 Element 为`RenderObjectElement`，由于`RenderObjectElement`也是抽象类，故子类需要重写该方法；

+ *RenderObject createRenderObject(BuildContext context)* —— 核心方法，创建 Render Widget 对应的 Render Object，同样子类需要重写该方法。该方法在对应的 Element 被挂载到树上时调用(`Element.mount`)，即在 Element 挂载过程中同步构建了「Render Tree」(详细过程后续文章会详细分析)；
```dart
  @override
  RenderFlex createRenderObject(BuildContext context) {
    return RenderFlex(
      direction: direction,
      mainAxisAlignment: mainAxisAlignment,
      mainAxisSize: mainAxisSize,
      crossAxisAlignment: crossAxisAlignment,
      textDirection: getEffectiveTextDirection(context),
      verticalDirection: verticalDirection,
      textBaseline: textBaseline,
    );
  }
```
	上面是`Flex.createRenderObject`的源码，真实感受一下 (还是代码更有感觉)。可以看到，用`Flex`的信息(配置)初始化了`RenderFlex`。
	> `Flex`是`Row`、`Column`的基类，`RenderFlex`继承自`RenderBox`，后者继续自`RenderObject`。

+ *void updateRenderObject(BuildContext context, covariant RenderObject renderObject)* —— 核心方法，在 Widget 更新后，修改对应的 Render Object。该方法在首次 build 以及需要更新 Widget 时都会调用；
```dart
  @override
  void updateRenderObject(BuildContext context, covariant RenderFlex renderObject) {
    renderObject
      ..direction = direction
      ..mainAxisAlignment = mainAxisAlignment
      ..mainAxisSize = mainAxisSize
      ..crossAxisAlignment = crossAxisAlignment
      ..textDirection = getEffectiveTextDirection(context)
      ..verticalDirection = verticalDirection
      ..textBaseline = textBaseline;
  }
```
	`Flex.updateRenderObject`的源码也很简单，与`Flex.createRenderObject`几乎一一对应，用当前`Flex`的信息修改`renderObject`。

+ *void didUnmountRenderObject(covariant RenderObject renderObject)* —— 对应的「Render Object」从「Render Tree」上移除时调用该方法。

> `RenderObjectWidget`的几个子类：`LeafRenderObjectWidget`、`SingleChildRenderObjectWidget`、`MultiChildRenderObjectWidget`只是重写了`createElement`方法以便返回各自对应的具体的 Element 类实例。

# 小结
_________________

至此，重要的基础型 Widget 基本介绍完了，总结一下：
+ Widget 本质上是 UI 的配置信息 (附加部分业务逻辑)，并不存在一颗真实的「Widget Tree」(与「Element Tree」、「RenderObject Tree」以及「Layer Tree」相比)；

+ Widget 从功能上可以分为 3 类：「Component Widget」、「Proxy Widget」以及「Renderer Widget」；

+ Widget 与 Element 一一对应，Widget 提供创建 Element 的方法 (`createElement`，本质上是一个工厂方法)；

+ 只有「Renderer Widget」才会参与最终的 UI 生成过程(Layout、Paint)，只有该类型的 Widget 才有与之对应的「Render Object」，同样由其提供创建方法(`createRenderObject`)。

下篇再见！


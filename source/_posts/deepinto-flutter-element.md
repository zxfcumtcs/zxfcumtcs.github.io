---
layout: post
title: 深入浅出 Flutter Framework 之 Element
date: 2020-05-17 10:08:21
tags:
- flutter
- 移动开发
- 跨平台
---
本文是『 深入浅出 Flutter Framework 』系列文章的第三篇，主要围绕 Element 相关内容进行分析介绍，包括 Element 分类、Element 与其他几个核心元素的关系、Element 生命周期以及核心方法解读等。

<!--more-->
©原创文章，转载请注明出处！

本系列文章将深入 Flutter Framework 内部逐步去分析其核心概念和流程，主要包括：
[『 深入浅出 Flutter Framework 之 Widget 』](https://zxfcumtcs.github.io/2020/05/01/deepinto-flutter-widget/)
[『 深入浅出 Flutter Framework 之 BuildOwner 』](https://zxfcumtcs.github.io/2020/05/16/deepinto-flutter-buildowner/)
『 深入浅出 Flutter Framework 之 Element 』
[『 深入浅出 Flutter Framework 之 PaintingContext 』](https://zxfcumtcs.github.io/2020/05/23/deepinto-flutter-paintingcontext/)
[『 深入浅出 Flutter Framework 之 Layer 』](https://zxfcumtcs.github.io/2020/06/07/deepinto-flutter-layer/)
[『 深入浅出 Flutter Framework 之 PipelineOwner 』](https://zxfcumtcs.github.io/2020/12/05/deepinto-flutter-pipelineowner/)
『 深入浅出 Flutter Framework 之 RenderObejct 』
『 深入浅出 Flutter Framework 之 Binding 』
『 深入浅出 Flutter Framework 之 Rendering Pipeline 』
『 深入浅出 Flutter Framework 之 自定义 Widget 』

# Overview
_________________
通过[『 深入浅出 Flutter Framework 之 Widget 』](https://zxfcumtcs.github.io/2020/05/01/deepinto-flutter-widget/)的介绍，我们知道 Widget 本质上是 UI 的配置数据 (静态、不可变)，Element 则是通过 Widget 生成的『实例』，两者间的关系就像是 json 与 object。
> 同一份配置 (Widget) 可以生成多个实例 (Element)，这些实例可能会被安插在树上不同的位置。

UI 的层级结构在 Element 间形成一棵真实存在的树「Element Tree」，Element 有 2 个主要职责：
+ 根据 UI (「Widget Tree」) 的变化来维护「Element Tree」，包括：节点的插入、更新、删除、移动等；
+ Widget 与 RenderObject 间的协调者。

# 分类
_________________
![](/img/Element.png)
如图所示，Element 根据特点可以分为 2 类：
+ *「Component Element」* —— 组合型 Element，「Component Widget」、「Proxy Widget」对应的 Element 都属于这一类型，其特点是子节点对应的 Widget 需要通过`build`方法去创建。同时，该类型 Element 都只有一个子节点 (single child)；
+ *「Renderer Element」* —— 渲染型 Element，对应「Renderer Widget」，其不同的子类型包含的子节点个数也不一样，如：LeafRenderObjectElement 没有子节点，RootRenderObjectElement、SingleChildRenderObjectElement 有一个子节点，MultiChildRenderObjectElement 有多个子节点。
> 原生型 Element，只有 MultiChildRenderObjectElement 是多子节点的，其他都是单子节点。

同时，可以看到，`Element`实现了`BuildContext`接口 —— 我们在 Widget 中遇到的`context`，其实就是该 Widget 对应的 Element。

# 关系
_________________
在继续之前有必要先了解一下 Element 与其他几个核心元素间的关系，以便在全局上有个认识。
![](/img/Element_Relationship.png)

如图：
+ Element 通过 parent、child 指针形成「Element Tree」；
+ Element 持有 Widget、「Render Object」；
+ State 是绑定在 Element 上的，而不是绑在「Stateful Widget」上(这点很重要)。
  > 上述这些关系并不是所有类型的 Element 都有，如：「Render Object」只有「RenderObject Element」才有，State 只有「Stateful Element」才有。

# 生命周期
_________________
Element 作为『实例』，随着 UI 的变化，有较复杂的生命周期：
+ parent 通过`Element.inflateWidget`->`Widget.createElement`创建 child element，触发场景有：UI 的初次创建、UI 刷新时新老 Widget 不匹配(old element 被移除，new element 被插入)；

+ parent 通过`Element.mount`将新创建的 child 插入「Element Tree」中指定的插槽处 (slot);
> `dynamic Element.slot`——其含意对子节点透明，父节点用于确定其下子节点的排列顺序 (兄弟节点间的排序)。因此，对于单子节点的节点 (single child)，child.slot 通常为 null。
另外，slot 的类型是动态的，不同类型的 Element 可能会使用不同类型的 slot，如：Sliver 系列使用的是 int 型的 index，MultiChildRenderObjectElement 用兄弟节点作为后一个节点的 slot。
对于「component element」，`mount`方法还要负责所有子节点的 build (这是一个递归的过程)，对于「render element」，`mount`方法需要负责将「render object」添加到「render tree」上。其过程在介绍到相应类型的 Element 时会详情分析。

+ 此时，(child) element 处于 **active** 状态，其内容随时可能显示在屏幕上；

+ 此后，由于状态更新、UI 结构变化等，element 所在位置对应的 Widget 可能发生了变化，此时 parent 会调用`Element.update`去更新子节点，update 操作会在以当前节点为根节点的子树上递归进行，直到叶子节点；(执行该步骤的前提是新老 Widget.[key && runtimeType] 相等，否则创建新 element，而不是更新现有 element)；

+ 状态更新时，element 也可能会被移除 (如：新老 Widget.[key || runtimeType] 不相等)，此时，parent 将调用`deactivateChild`方法，该方法主要做了 3 件事：
  + 从「Element Tree」中移除该 element (将 parent 置为 null)；
  + 将相应的「render object」从「render tree」上移除；
  + 将 element 添加到`owner._inactiveElements`中，在添加过程中会对『以该 element 为根节点的子树上所有节点』调用`deactivate`方法 (移除的是整棵子树)。
  ```dart
  void deactivateChild(Element child) {
    child._parent = null;
    child.detachRenderObject();
    owner._inactiveElements.add(child); // this eventually calls child.deactivate()
  }
  ```

+ 此时，element 处于 "inactive" 状态，并从屏幕上消失，该状态一直持续到当前帧动画结束；

+ 从 element 进入 "inactive" 状态到当前帧动画结束期间，其还有被『抢救』的机会，前提是『带有「global key」&& 被重新插入树中』，此时：
  + 该 element 将会从`owner._inactiveElements`中移除；
  + 对该 element subtree 上所有节点调用`activate`方法 (它们又复活了！)；
  + 将相应的「render object」重新插入「render tree」中；
  + 该 element subtree 又进入 "active" 状态，并将再次出现在屏幕上。
  > 上述过程经历这几个方法：`Parent Element.inflateWidget`-->`Parent Element._retakeInactiveElement`-->`BuildOwner._inactiveElements.remove`-->`Child Element._activateWithParent`...

+ 对于所有在当前帧动画结束时未能成功『抢救』回来的「Inactive Elements」都将被 unmount；
+ 至此，element 生命周期圆满结束。
![](/img/Element_Lifecycle.jpg)

# 核心方法
_________________
下面对 Element 中的几个核心方法进行简单介绍：

## updateChild
`updateChild`是 flutter framework 中的核心方法之一：
```dart
Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
  if (newWidget == null) {
    if (child != null)
      deactivateChild(child);
    return null;
  }

  if (child != null) {
    if (child.widget == newWidget) {
      if (child.slot != newSlot)
        updateSlotForChild(child, newSlot);
      return child;
    }

    if (Widget.canUpdate(child.widget, newWidget)) {
      if (child.slot != newSlot)
        updateSlotForChild(child, newSlot);
      child.update(newWidget);
      assert(child.widget == newWidget);

      return child;
    }

    deactivateChild(child);
    assert(child._parent == null);
  }

  return inflateWidget(newWidget, newSlot);
}
```
在「Element Tree」上，**父节点通过该方法来修改子节点对应的 Widget**。
![](/img/updateChild.jpg)

根据传入参数的不同，有以下几种不同的行为：
+ `newWidget` == `null` —— 说明子节点对应的 Widget 已被移除，直接 remove child element (如有)；
+ `child` == `null` —— 说明 newWidget 是新插入的，创建子节点 (inflateWidget)；
+ `child` != `null` —— 此时，分为 3 种情况：
  + 若 child.widget == newWidget，说明 child.widget 前后没有变化，若 child.slot != newSlot 表明子节点在兄弟结点间移动了位置，通过`updateSlotForChild`修改 child.slot 即可；
  + 通过`Widget.canUpdate`判断是否可以用 newWidget 修改 child element，若可以，则调用`update`方法；
  + 否则先将 child element 移除，并通 newWidget 创建新的 element 子节点。

> 子类一般不需要重写该方法，该方法有点类似设计模式中的『模板方法』。

## update
在更新流程中，若新老 Widget.[runtimeType && key] 相等，则会走到该方法。
子类需要重写该方法以处理具体的更新逻辑：

### Element 基类
```dart
@mustCallSuper
void update(covariant Widget newWidget) {
  _widget = newWidget;
}
```
基类中的`update`很简单，只是对`_widget`赋值。
> 子类重写该方法时必须调用 super.

### StatelessElement
> 父类`ComponentElement`没有重写该方法

```dart
void update(StatelessWidget newWidget) {
  super.update(newWidget);
  _dirty = true;
  rebuild();
}
```
通过`rebuild`方法触发重建 child widget (第 4 行)，并以此来 update child element，期间会调用到`StatelessWidget.build`方法 (也就是我们写的 Flutter 代码)。
> 组合型 Element 都会在`update`方法中触发`rebuild`操作，以便重新 build child widget。

### StatefulElement
```dart
void update(StatefulWidget newWidget) {
  super.update(newWidget);
  final StatefulWidget oldWidget = _state._widget;
  _dirty = true;
  _state._widget = widget;
  try {
    _state.didUpdateWidget(oldWidget) as dynamic;
  }
  finally {
  }
  rebuild();
}
```
相比`StatelessElement`，`StatefulElement.update`稍微复杂一些，需要处理`State`，如：
+ 修改 State 的 `_widget`属性；
+ 调用`State.didUpdateWidget` (熟悉么)。

最后，同样会触发`rebuild`操作，期间会调用到`State.build`方法。

### ProxyElement
```dart
void update(ProxyWidget newWidget) {
  final ProxyWidget oldWidget = widget;
  super.update(newWidget);
  updated(oldWidget);
  _dirty = true;
  rebuild();
}

void updated(covariant ProxyWidget oldWidget) {
  notifyClients(oldWidget);
}

Widget build() => widget.child;
```
`ProxyElement.update`方法需要关注的是对`updated`的调用，其主要用于通知关联对象 Widget 有更新。
具体通知逻辑在子类中处理，如：`InheritedElement`会触发所有依赖者 rebuild (对于 StatefulElement 类型的依赖者，会调用`State.didChangeDependencies`)。

ProxyElement 的`build`操作很简单：直接返回`widget.child`。

### RenderObjectElement
```dart
void update(covariant RenderObjectWidget newWidget) {
  super.update(newWidget);
  widget.updateRenderObject(this, renderObject);
  _dirty = false;
}
```
`RenderObjectElement.update`方法调用了`widget.updateRenderObject`来更新「Render Object」(熟悉么)。

### SingleChildRenderObjectElement
> `SingleChildRenderObjectElement`、`MultiChildRenderObjectElement`是`RenderObjectElement`的子类。

```dart
void update(SingleChildRenderObjectWidget newWidget) {
  super.update(newWidget);
  _child = updateChild(_child, widget.child, null);
}
```
第 3 行，通过`newWidget.child`调用`updateChild`方法递归修改子节点。

### MultiChildRenderObjectElement
```dart
void update(MultiChildRenderObjectWidget newWidget) {
  super.update(newWidget);
  _children = updateChildren(_children, widget.children, forgottenChildren: _forgottenChildren);
}
```
上述实现看似简单，实则非常复杂，在`updateChildren`方法中处理了子节点的插入、移动、更新、删除等所有情况。

## inflateWidget
```dart
Element inflateWidget(Widget newWidget, dynamic newSlot) {
  final Key key = newWidget.key;
  if (key is GlobalKey) {
    final Element newChild = _retakeInactiveElement(key, newWidget);
    if (newChild != null) {
      newChild._activateWithParent(this, newSlot);
      final Element updatedChild = updateChild(newChild, newWidget, newSlot);
      return updatedChild;
    }
  }

  final Element newChild = newWidget.createElement();
  newChild.mount(this, newSlot);
  return newChild;
}
```
> `inflateWidget` 属于模板方法，故一般情况下子类不用重写。

该方法的主要职责：通过 Widget 创建对应的 Element，并将其挂载 (mount) 到「Element Tree」上。
> 如果 Widget 带有 GlobalKey，首先在 Inactive Elements 列表中查找是否有处于 inactive 状态的节点 (即刚从树上移除)，如找到就直接复活该节点。

主要调用路径来自上面介绍的`updateChild`方法。

## mount
当 Element 第一次被插入「Element Tree」上时，调用该方法。由于此时 parent 已确定，故在该方法中可以做依赖 parent 的初始化操作。经过该方法后，element 的状态从 "initial" 转到了 "active"。

### Element
```dart
@mustCallSuper
void mount(Element parent, dynamic newSlot) {
  _parent = parent;
  _slot = newSlot;
  _depth = _parent != null ? _parent.depth + 1 : 1;
  _active = true;
  if (parent != null) // Only assign ownership if the parent is non-null
    _owner = parent.owner;

  if (widget.key is GlobalKey) {
    final GlobalKey key = widget.key;
    key._register(this);
  }

  _updateInheritance();
}
```
还记得`BuildOwner`吗，正是在该方法中父节点的 owner 传给了子节点。
如果，对应的 Widget 带有 GlobalKey，进行相关的注册。
最后，继承来自父节点的「Inherited Widgets」。
> 子类重写该方法时，必须调用 super。
> 关于「Inherited Widgets」，后文会详细分析

### ComponentElement
```dart
void mount(Element parent, dynamic newSlot) {
  super.mount(parent, newSlot);
  _firstBuild();
}

void _firstBuild() {
  rebuild();
}
```
组合型 Element 在挂载时会执行`_firstBuild->rebuild`操作。

### RenderObjectElement
```dart
void mount(Element parent, dynamic newSlot) {
  super.mount(parent, newSlot);
  _renderObject = widget.createRenderObject(this);
  attachRenderObject(newSlot);
  _dirty = false;
}
```
在`RenderObjectElement.mount`中做的最重要的事就是通过 Widget 创建了「Render Object」(第 3 行)，并将其插入到「RenderObject Tree」上 (第 4 行)。

### SingleChildRenderObjectElement
```dart
@override
void mount(Element parent, dynamic newSlot) {
  super.mount(parent, newSlot);
  _child = updateChild(_child, widget.child, null);
}
```
`SingleChildRenderObjectElement`在 super (`RenderObjectElement`) 的基础上，调用`updateChild`方法处理子节点，其实此时`_child`为`nil`，前面介绍过当 child 为`nil`时，`updateChild`会调用`inflateWidget`方法创建 Element 实例。

### MultiChildRenderObjectElement
```dart
void mount(Element parent, dynamic newSlot) {
  super.mount(parent, newSlot);
  _children = List<Element>(widget.children.length);
  Element previousChild;
  for (int i = 0; i < _children.length; i += 1) {
    final Element newChild = inflateWidget(widget.children[i], previousChild);
    _children[i] = newChild;
    previousChild = newChild;
  }
}
```
`MultiChildRenderObjectElement`在 super (`RenderObjectElement`) 的基础上，对每个子节点直接调用`inflateWidget`方法。

## markNeedsBuild
```dart
void markNeedsBuild() {
  if (!_active)
    return;

  if (dirty)
    return;

  _dirty = true;
  owner.scheduleBuildFor(this);
}
```
`markNeedsBuild`方法其实在介绍[BuildOwer](https://zxfcumtcs.github.io/2020/05/16/deepinto-flutter-buildowner/)时已经分析过，其作用就是将当前 Element 加入`_dirtyElements`中，以便在下一帧可以rebuild。
那么，哪些场景会调用`markNeedsBuild`呢？
+ `State.setState` —— 这个在介绍 [Widget](https://zxfcumtcs.github.io/2020/05/01/deepinto-flutter-widget/) 时已分析过了；
+ `Element.reassemble` —— debug hot reload；
+ `Element.didChangeDependencies` —— 前面介绍过当依赖的「Inherited Widget」有变化时会导致依赖者 rebuild，就是从这里触发的；
+ `StatefulElement.activate` —— 还记得`activate`吗？前文介绍过当 Element 从 "inactive" 到 "active" 时，会调用该方法。为什么`StatefulElement`要重写`activate`？因为`StatefulElement`有附带的 State，需要给它一个`activate`的机会。

> 子类一般不必重写该方法。

## rebuild
```dart
void rebuild() {
  if (!_active || !_dirty)
    return;

  performRebuild();
}
```
该方法逻辑非常简单，对于活跃的、脏节点调用`performRebuild`，在 3 种场景下被调用：
+ 对于 dirty element，在新一帧绘制过程中由`BuildOwner.buildScope`；
+ 在 element 挂载时，由`Element.mount`调用；
+ 在`update`方法内被调用。

> 上述第 2、3 点仅「Component Element」需要

## performRebuild
Element 基类中该方法是`no-op`。
### ComponentElement
```dart
void performRebuild() {
  Widget built;
  built = build();

  _child = updateChild(_child, built, slot);
}
```
对于组合型 Element，rebuild 过程其实就是调用`build`方法生成「child widget」，再由其更新「child element」。
> StatelessElement.build: `Widget build() => widget.build(this);`
> StatefulElement.build: `Widget build() => state.build(this);`
> ProxyElement.build: `Widget build() => widget.child;`


### RenderObjectElement
```dart
void performRebuild() {
  widget.updateRenderObject(this, renderObject);
  _dirty = false;
}
```
在渲染型 Element 基类中只是用 Widget 更新了对应的「Render Object」。
在相关子类中可以执行更具体的逻辑。

## 生命周期视角
至此，Element 的核心方法基本已介绍完，是不是有点晕乎乎的感觉？`inflateWidget`、`updateChild`、`update`、`mount`、`rebuild`以及`performRebuild`等你中有我、我中有你，再加上不同类型的子类对这些方法的重写。

下面，我们以 Element 生命周期为切入点将这些方法串起来。
对于一个 Element 节点来说在其生命周期内可能会历经几次『重大事件』：
+ 被创建 —— 起源于父节点调用`inflateWidget`，随之被挂载到「Element Tree」上， 此后递归创建子节点；
![](/img/element0to1.jpg)

+ 被更新 —— 由「Element Tree」上祖先节点递归传递下来的更新操作，`parent.updateChild`->`child.update`；
![](/img/element1to1.1.jpg)

+ 被重建 —— 被调用`rebuild`方法(调用场景上面已分析)；
![](/img/element1ton.jpg)

+ 被销毁 —— element 节点所在的子树随着 UI 的变化被移除。
![](/img/element-deactivate.jpg)

# 依赖 (Dependencies)
_________________
在 Element 基类中有这样两个成员：
```dart
Map<Type, InheritedElement> _inheritedWidgets;
Set<InheritedElement> _dependencies;
```
它们是干嘛用的呢？
+ `_inheritedWidgets` —— 用于收集从「Element Tree」根节点到当前节点路径上所有的「Inherited Elements」；
前文提到过在`mount`方法结束处会调用`_updateInheritance`：
以下是 Element 基类的实现，可以看到子节点直接获得父节点的`_inheritedWidgets`：
```dart
void _updateInheritance() {
  _inheritedWidgets = _parent?._inheritedWidgets;
}
```
以下是`InheritedElement`类的实现，其在父节点的基础上将自己加入到`_inheritedWidgets`中，以便其子孙节点的`_inheritedWidgets`包含它 (第 8 行)：
```dart
void _updateInheritance() {
  final Map<Type, InheritedElement> incomingWidgets = _parent?._inheritedWidgets;
  if (incomingWidgets != null)
    _inheritedWidgets = HashMap<Type, InheritedElement>.from(incomingWidgets);
  else
    _inheritedWidgets = HashMap<Type, InheritedElement>();

  _inheritedWidgets[widget.runtimeType] = this;
}
```
+ `_dependencies` —— 用于记录当前节点依赖了哪些「Inherited Elements」，通常我们调用`context.dependOnInheritedWidgetOfExactType<T>`时就会在当前节点与目标 Inherited 节点间形成依赖关系。
> 在 Element 上提供的便利方法`of`，一般殾会调用`dependOnInheritedWidgetOfExactType`。

同时，在`InheritedElement`中还有用于记录所有依赖于它的节点：`final Map<Element, Object> _dependents`。
最终，在「Inherited Element」发生变化，需要通知依赖者时，会利用依赖者的`_dependencies`信息做一下 (debug) check (第 4 行)：
```dart
void notifyClients(InheritedWidget oldWidget) {
  for (Element dependent in _dependents.keys) {
    // check that it really depends on us
    assert(dependent._dependencies.contains(this));
    notifyDependent(oldWidget, dependent);
  }
}
```

# 小结
至此，Element 相关的内容基本已介绍完。总结提炼一下：
+ Element 与 Widget 一一对应，它们间的关系就像 object 与 json；
+ 只有「Render Element」才有对应的「Render Object」；
+ Element 作为 Widget 与 RenderObejct 间协调者，会根据 UI(「Widget Tree」) 的变化对「Element Tree」作出相应的调整，同时对「RenderObject Tree」进行必要的修改；
+ Widget 是不可变的、无状态的，而 Element 是有状态的。

最后，强烈推荐[Keys! What are they good for?](https://medium.com/flutter/keys-what-are-they-good-for-13cb51742e7d)这篇文章，对于理解本文相关的内容有很大的帮助。

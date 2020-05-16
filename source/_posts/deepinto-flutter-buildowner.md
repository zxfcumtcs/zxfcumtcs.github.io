layout: post
title: 深入浅出 Flutter Framework 之 BuildOwner
date: 2020-05-16 15:39:12
tags:
- flutter
- 移动开发
- 跨平台
---
本文是『 深入浅出 Flutter Framework 』系列文章的第二篇，对 BuildOwer 相关内容进行简要地分析介绍，为下一篇文章介绍 Element 作准备 (由于篇幅原因将其单独提出来)。

<!--more-->
©原创文章，转载请注明出处！

本系列文章将深入 Flutter Framework 内部逐步去分析其核心概念和流程，主要包括：
[『 深入浅出 Flutter Framework 之 Widget 』](https://zxfcumtcs.github.io/2020/05/01/deepinto-flutter-widget/)
『 深入浅出 Flutter Framework 之 BuildOwner 』
[『 深入浅出 Flutter Framework 之 Element 』](https://zxfcumtcs.github.io/2020/05/17/deepinto-flutter-element/)
『 深入浅出 Flutter Framework 之 PipelineOwner 』
『 深入浅出 Flutter Framework 之 RenderObejct 』
『 深入浅出 Flutter Framework 之 Layer 』
『 深入浅出 Flutter Framework 之 Binding 』
『 深入浅出 Flutter Framework 之 Rendering Pipeline 』
『 深入浅出 Flutter Framework 之 自定义 Widget 』

# Overview
_________________
`BuildOwer`在 Element 状态管理上起到重要作用：
+ 在 UI 更新过程中跟踪、管理需要 rebuild 的 Element (「dirty elements」);
+ 在有「dirty elements」时，及时通知引擎，以便在下一帧安排上对「dirty elements」的 rebuild，从而去刷新 UI；
+ 管理处于 "inactive" 状态的 Element。

> 这是我们遇到的第一个 Owner，后面还有`PipeOwner`。

整棵「Element Tree」共享同一个`BuildOwer`实例 (全局的)，在 Element 挂载过程中由 parent 传递给 child element。
```dart
@mustCallSuper
void mount(Element parent, dynamic newSlot) {
  _parent = parent;
  _slot = newSlot;
  _depth = _parent != null ? _parent.depth + 1 : 1;
  _active = true;
  if (parent != null) // Only assign ownership if the parent is non-null
    _owner = parent.owner;
}
```
以上是`Element`基类的`mount`方法，第 8 行将 parent.owner 赋给了 child。

> `BuildOwer`实例由`WidgetsBinding`负责创建，并赋值给「Element Tree」的根节点`RenderObjectToWidgetElement`，此后随着「Element Tree」的创建逐级传递给子节点。(具体流程后续文章会详细分析)
一般情况下并不需要我们手动实例化`BuildOwer`，除非需要离屏沉浸 (此时需要构建 off-screen element tree)

`BuildOwer`两个关键成员变量：
```dart
final _InactiveElements _inactiveElements = _InactiveElements();
final List<Element> _dirtyElements = <Element>[];
```
其命名已清晰表达了他们的用途：分别用于存储收集到的「Inactive Elements」、「Dirty Elements」。

# Dirty Elements
_________________
那么`BuildOwer`是如何收集「Dirty Elements」的呢？
对于需要更新的 element，首先会调用`Element.markNeedsBuild`方法，如[前文](https://zxfcumtcs.github.io/2020/05/01/deepinto-flutter-widget/)讲到的`State.setState`方法：
```dart
void setState(VoidCallback fn) {
  final dynamic result = fn() as dynamic;
  _element.markNeedsBuild();
}
```

如下，`Element.markNeedsBuild`调用了`BuildOwer.scheduleBuildFor`方法：
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

`BuildOwer.scheduleBuildFor`方法做了 2 件事：
+ 调用`onBuildScheduled`，该方法(其实是个callback)会通知 Engine 在下一帧需要做更新操作；
+ 将「Dirty Elements」加入到`_dirtyElements`中。
```dart
void scheduleBuildFor(Element element) {
  assert(element.owner == this);
  onBuildScheduled();

  _dirtyElements.add(element);
  element._inDirtyList = true;
}
```

此后，在新一帧绘制到来时，`WidgetsBinding.drawFrame`会调用`BuildOwer.buildScope`方法：
```dart
void buildScope(Element context, [ VoidCallback callback ]) {
  if (callback == null && _dirtyElements.isEmpty)
    return;

  try {
    if (callback != null) {
      callback();
    }

    _dirtyElements.sort(Element._sort);
    int dirtyCount = _dirtyElements.length;
    int index = 0;
    while (index < dirtyCount) {
      _dirtyElements[index].rebuild();
      index += 1;
    }
  } finally {
    for (Element element in _dirtyElements) {
      element._inDirtyList = false;
    }
    _dirtyElements.clear();
  }
}
```
+ 如有回调，先执行回调 (第 7 行)；
+ 对「dirty elements」按在「Element Tree」上的深度排序 (即 parent 排在 child 前面) (第 10 行)；
> 为啥要这样排？确保 parent 先于 child 被 rebuild，以免 child 被重复 rebuild (因为 parent 在 rebuild 时会递归地 update child)。

+ 对`_dirtyElements`中的元素依次调用`rebuild` (第 14 行)；
+ 清理`_dirtyElements` (第 21 行)。


# Inactive Elements
_________________
所谓「Inactive Element」，是指 element 从「Element Tree」上被移除到 dispose 或被重新插入「Element Tree」间的一个中间状态。
**设计 inactive 状态的主要目的是实现『带有「global key」的 element』可以带着『状态』在树上任意移动。**

BuildOwer 负责对「Inactive Element」进行管理，包括添加、删除以及对过期的「Inactive Element」执行 unmount 操作。
关于「Inactive Element」的更多信息将在介绍 Element 时一起介绍。

# 小结
BuildOwner 主要是用于收集那些需要 rebuild 的「Dirty Elements」以及处于 Inactive 状态的 Elements。

结束了！就是这么简单，下篇再见！

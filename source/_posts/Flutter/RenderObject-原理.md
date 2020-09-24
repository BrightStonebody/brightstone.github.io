---
title: RenderObject 原理
date: 2020-09-22 16:28:21
tags:
- Flutter
- RenderObject
cateogries:
- Flutter
---


## ParentData 和各种 Mixin

``` 
@startuml
BoxParentData --|> ParentData
ContainerParentDataMixin --|> ParentData
ContainerBoxParentData --|> BoxParentData
ContainerBoxParentData --|> ContainerParentDataMixin

RenderObject *-- ParentData
RenderObjectWithChildMixin --|> RenderObject
ContainerRenderObjectMixin --|> RenderObject


class ContainerParentDataMixin{
+ previousSlibling
+ nextSibling
}

class ContainerRenderObjectMixin {
+ firstChild *-- ContainerParentDataMixin
+ lastChild
+ childCount
}

class BoxParentData{
+ offset
}

class RenderObject{
+ parentData
}
@enduml
```

* ParentData
在 Flutter 的布局系统中，该类负责存储父节点所需要的子节点的布局信息
    * BoxParentData
    -- offset
    包含有一个 offset 属性，该属性用于存储 child 的布局信息，也就是 child 应该被摆在哪个位置。与 Android 不同，Flutter并不是以父View为坐标基础进行绘制的，所以需要带上 offset 参数，得到一个累加的偏移值。至于偏移的参考点在哪里，可以看下面的绘制阶段的原理解析。
    * ContainerParentDataMixin
    -- previousSibling
    -- nextSibling
    该类使用频率很高，基本上所有父节点的 ParentData 都混入了该类，该类需要与ContainerRenderObjectMixin 共同使用，主要解决了对 child 的管理，previousSibling nextSibling 都是child RenderObject, 从而组成把父节点的所有 child 组成了一个双链表
    * ContainerBoxParentData
    空类，继承了 BoxParentData 并且混入了 ContainerParentDataMixin， 使其拥有二者的能力
* RenderObject
Flutter 中真正实现布局和绘制的类
    * RenderObjectWithChildMixin
    实现对只有一个child的管理
    * ContainerRenderObjectMixin
    -- firstChild
    -- lastChild
    -- childCount
    实现对多child的管理，firstChild、lastChild、childCount，用来获取首末 child，child的个数，配合使用 ContainerParentDataMixin 中的 previousSibling、nextSibling就可以对 child 进行遍历了。
    child 的insert、remove、move等操作都在这个Mixin中实现

除了以上的类，Flutter 还提供了 RenderBoxContainerDefaultsMixin，该类提供了一些 RenderBox 默认的行为方法，如上面绘制 child 的流程调用该类中的 defaultPaint(PaintingContext context, Offset offset) 就可以了，可以简化一些模板代码。

## layout

在 RenderBox 中，控件大小的值为 _size 成员，它只包含宽高两个属性值，我们可以通过该成员的 set 和 get 方法访问或修改它的值。在测量时( layout 方法中)，parent 会传给当前 RenderBox 一个大小的限制，为 BoxConstraints 类型，最后测量得到的 size 必须满足这个限制，在 Flutter 的 debug 模式下对 size 是否满足 constraints 做了 assert 检查，如果检查未通过就会布局失败。所以测量上我们要做的是下面两点：

* 如果没有 child，那么根据自身的属性计算出满足 constraints 的 size.
* 如果有 child，那么综合自身的属性和 child 的测量结果计算出满足 constraints 的 size.

测量和布局都在layout方法中完成

```dart
  void layout(Constraints constraints, { bool parentUsesSize = false }) {
    RenderObject relayoutBoundary;
    if (!parentUsesSize || sizedByParent || constraints.isTight || parent is! RenderObject) {
      relayoutBoundary = this;
    } else {
      final RenderObject parent = this.parent;
      relayoutBoundary = parent._relayoutBoundary;
    }
    if (!_needsLayout && constraints == _constraints && relayoutBoundary == _relayoutBoundary) {
      return;
    }
    _constraints = constraints;
    if (_relayoutBoundary != null && relayoutBoundary != _relayoutBoundary) {
      visitChildren((RenderObject child) {
        child._cleanRelayoutBoundary();
      });
    }
    _relayoutBoundary = relayoutBoundary;
    if (sizedByParent) {
      try {
        performResize();
      } catch (e, stack) {
        _debugReportException('performResize', e, stack);
      }
    }
    try {
      performLayout();
      markNeedsSemanticsUpdate();
    } catch (e, stack) {
      _debugReportException('performLayout', e, stack);
    }
    _needsLayout = false;
    markNeedsPaint();
  }
```

* parentUsesSize
    layout 方法中由父节点传入，表示父节点的 size 依赖当前节点的 size 
* constraints
    layout 方法中从父节点传入，当前节点会从更新到自己的变量中保存
* relayoutBoundary 
    relayoutBoundary 是framework层自动设置的，如果满足 
    `(!parentUsesSize || sizedByParent || constraints.isTight || parent is! RenderObject)` 则该节点是一个 relayoutBoundary, 否则会从父节点寻找。
* sizedByParent & performResize()
    **这两个必须成对配置，performResize() 方法如果被重写，sizedByParent 也必须重写为true，并且你不应该在 perfromLayout() 方法中对 size 重新赋值。** 
    sizedByParent 意为该控件的大小是否能仅通过 parent 赋予它的 constraints 就可以被确定下来了，即该控件的大小与它自身的属性和与它的 child 都无关，比如如果一个控件永远充满 parent 的大小，那么 sizedByParent 就应该返回 true。
    **这个操作不是必须的**
    官方文档中说这是一个优化项，如果完全不care这个，sizedByParent = false，那始终是正确的。 我的理解是，把 size 的计算单独抽出，避免每次 relayout 时对 size 重新计算。
    performResize() 这个方法名字可能会造成误会，认为测量必须放在这里，其实大多是情况下，测量和布局都在 performLayout() 中
* performLayout()
    上面已经说得差不多了

如果是在标脏后，在一帧绘制的回调中刷新布局，不会调用layout方法，而是调用_layoutWithoutSize 

```dart
  void _layoutWithoutResize() {
    try {
      performLayout();
      markNeedsSemanticsUpdate();
    } catch (e, stack) {
      _debugReportException('performLayout', e, stack);
    }
    _needsLayout = false;
    markNeedsPaint();
  }
```

## paint

父节点绘制子节点的入口

**paintContext.paintChild()**

```dart
  void paintChild(RenderObject child, Offset offset) {
    if (child.isRepaintBoundary) {
      stopRecordingIfNeeded();
      _compositeChild(child, offset);
    } else {
      child._paintWithContext(this, offset);
    }
  }


  void _compositeChild(RenderObject child, Offset offset) {
    // Create a layer for our child, and paint the child into it.
    if (child._needsPaint) {
      // 这个方法兜兜转转又会调用 child._paintWithContext 方法
      repaintCompositedChild(child, debugAlsoPaintedParent: true);
    } else {
    }
    final OffsetLayer childOffsetLayer = child._layer;
    childOffsetLayer.offset = offset;
    appendLayer(child._layer);
  }

  @protected
  void appendLayer(Layer layer) {
    layer.remove();
    _containerLayer.append(layer);
  }
```

在 paintChild 方法中，若 child 是一个 repaintBoundary ，会为 child 创建一个 layer。。所以这个layer又是怎样？？？ 
这样可以达成的效果是，flutter引擎在刷新重绘时，可以进行局部刷新。

在drawFrame()中，paint 标脏节点时会直接执行 repaintCompositeChild 

**paint()**

```dart
  void paint(PaintingContext context, Offset offset) { }
```

RenderObject 中的 paint 是一个空方法，交给子类来实现。 以 RenderFlex 为例： 

```dart
  @override
  void paint(PaintingContext context, Offset offset) {
    if (!_hasOverflow) {
      defaultPaint(context, offset);
      return;
    }

    ...
  }

  void defaultPaint(PaintingContext context, Offset offset) {
    ChildType child = firstChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      context.paintChild(child, childParentData.offset + offset);
      child = childParentData.nextSibling;
    }
  }
```

因为Flex没有要绘制的内容，所以调用 RenderBoxContainerDefaultsMixin 提供的默认实现。在 defaultPaint 中遍历child，绘制child时将 childParentData.offset累加到offset中。所以，这个 offset 的基准到底是个啥？？？？

## PaintContext 和 Layer





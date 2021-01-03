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
      // 这个方法会创建layer，然后再调用 child._paintWithContext 方法
      repaintCompositedChild(child, debugAlsoPaintedParent: true);
    } else {
    }
    final OffsetLayer childOffsetLayer = child._layer;
    childOffsetLayer.offset = offset;
    appendLayer(child._layer);
  }

  static void _repaintCompositedChild(
    RenderObject child, {
    bool debugAlsoPaintedParent = false,
    PaintingContext childContext,
  }) {
    OffsetLayer childLayer = child._layer;
    if (childLayer == null) {
      child._layer = childLayer = OffsetLayer();
    } else {
      childLayer.removeAllChildren();
    }
    childContext ??= PaintingContext(child._layer, child.paintBounds);
    child._paintWithContext(childContext, Offset.zero);
    childContext.stopRecordingIfNeeded();
  }

  @protected
  void appendLayer(Layer layer) {
    layer.remove();
    _containerLayer.append(layer);
  }
```

在 paintChild 方法中，若 child 是一个 repaintBoundary ，会为 child 创建一个 layer ，然后再调用 child._paintWithContext(this, offset) ；如果 isRepaintBoundary 为false，则直接调用 child._paintWithContext(this, offset)。。
所以这个layer又是怎样？？？ 

在drawFrame()中，paint 标脏节点刷新时会直接执行 PaintContext.repaintCompositeChild() 

**_paintWithContext(..) & paint()**

```dart
  void _paintWithContext(PaintingContext context, Offset offset) {
    if (_needsLayout)
      return;
    RenderObject debugLastActivePaint;
    _needsPaint = false;
    try {
      paint(context, offset);
    } catch (e, stack) {
      _debugReportException('paint', e, stack);
    }
  }

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

## PaintingContext 和 Layer

### drawFrame()
PaintContext 可以类比于 BuildContext，Layer 和 Element 最终会组成一个 Layer树，最终渲染是在C++的engine层完成的。在flutter的framework层构成了一颗Layer树，传送到engine进行绘制。

回过头来看，在 drawFrame() 中对需要paint的节点是如何处理的。

```dart
  void drawFrame() {
    // BD ADD:
    Boost.resetIdleCallbacks();
    assert(renderView != null);
    pipelineOwner.flushLayout();
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();
    renderView.compositeFrame(); // this sends the bits to the GPU
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
  }
```

先直接来看 pipelineOwner.flushPaint(); 和里面调用到的 PaintContext._repaintCompositedChild 

```dart
  void flushPaint() {
    try {
      final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
      _nodesNeedingPaint = <RenderObject>[];
      // Sort the dirty nodes in reverse order (deepest first).
      for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => b.depth - a.depth)) {
        if (node._needsPaint && node.owner == this) {
          if (node._layer.attached) {
            PaintingContext.repaintCompositedChild(node);
          } else {
            node._skippedPaintingOnLayer();
          }
        }
      }
    } finally {
    }
  }
```

**PaintingContext._repaintCompositedChild(...)**

```dart
  static void _repaintCompositedChild(
    RenderObject child, {
    bool debugAlsoPaintedParent = false,
    PaintingContext childContext,
  }) {
    // 关注这个 assert，证明了最终需要 paint 的都是 RepaintBoundary 节点
    assert(child.isRepaintBoundary);
    OffsetLayer childLayer = child._layer;
    if (childLayer == null) {
      // 为 repaint boundaries 创建layer
      child._layer = childLayer = OffsetLayer();
    } else {
      childLayer.removeAllChildren();
    }
    // 为 RepaintBoundary 创建 PaintContext
    childContext ??= PaintingContext(child._layer, child.paintBounds);
    // 初始化了Offset，说明offset的值是以layer为参考依据的
    child._paintWithContext(childContext, Offset.zero);

    childContext.stopRecordingIfNeeded();
  }
```

再次回顾 PaintingContext._repaintCompositedChild 方法，我们发现，它为 RepaintBoundary ，创建了PaintingContext 和 layer。 说明 RepaintBoundary 、PaintingContext 、layer 是一一对应的， **即 layer 是绘制的基本单位，只有 RepaintBoundary 极其子 RenderObject 共享一个layer， 每次绘制会创建都会为 layer 创建一个 PaintingContext**。而 paint 过程中的offset的参考点，就是layer的左上角的坐标原点。

**renderView.compositeFrame();**

```dart
  void compositeFrame() {
    Timeline.startSync('Compositing', arguments: timelineWhitelistArguments);
    try {
      final ui.SceneBuilder builder = ui.SceneBuilder();
      final ui.Scene scene = layer.buildScene(builder);
      if (automaticSystemUiAdjustment)
        _updateSystemChrome();
      _window.render(scene);
      scene.dispose();
    } finally {
      Timeline.finishSync();
    }
  }
```

renderView 是页面的根RenderObject，renderView.layer 是页面的根layer。 调用 `layer.buildScene(builder);` 生成了一个scene， 调用_window.render(scene) 发送给engine层进行渲染

### Layer的标脏和局部刷新

**layer.buildScene**

```dart
  ui.Scene buildScene(ui.SceneBuilder builder) {
    List<PictureLayer> temporaryLayers;
    updateSubtreeNeedsAddToScene();
    addToScene(builder);
    _needsAddToScene = false;
    final ui.Scene scene = builder.build();
    return scene;
  }

  @override
  void updateSubtreeNeedsAddToScene() {
    super.updateSubtreeNeedsAddToScene();
    Layer child = firstChild;
    while (child != null) {
      child.updateSubtreeNeedsAddToScene();
      _needsAddToScene = _needsAddToScene || child._needsAddToScene;
      // child.nextSibling 是下一个子layer， 这里显然是一个树的 层序遍历
      child = child.nextSibling;
    }
  }

  /**
   * 这是 ContainerLayer 的实现， 没有子Layer的叶子 Layer 实现各不相同
  **/
  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    addChildrenToScene(builder, layerOffset);
  }

  void addChildrenToScene(ui.SceneBuilder builder, [ Offset childOffset = Offset.zero ]) {
    Layer child = firstChild;
    while (child != null) {
      if (childOffset == Offset.zero) {
        child._addToSceneWithRetainedRendering(builder);
      } else {
        child.addToScene(builder, childOffset);
      }
      child = child.nextSibling;
    }
  }
```

renderView 的 layer 是一个 OffsetLayer ，继承自 ContainerLayer 。 
Layer.buildScene(...) 这个方法在framework调用的地方很少，可以认为只有 renderView 的 layer 会调用 buildScene 方法创建创建一个 scene ， 即 scene 包含了页面的完整 layer 树。

首先会调用 updateSubtreeNeedsAddToScene ， 需要update的原因是， 如果子layer标脏， 那显然父layer 也需要标脏

**_addToSceneWithRetainedRendering**

```dart
  void markNeedsAddToScene() {
    // Already marked. Short-circuit.
    if (_needsAddToScene) {
      return;
    }

    _needsAddToScene = true;
  }

  void _addToSceneWithRetainedRendering(ui.SceneBuilder builder) {
    if (!_needsAddToScene && _engineLayer != null) {
      builder.addRetained(_engineLayer);
      return;
    }
    addToScene(builder);
    _needsAddToScene = false;
  }

  
```

markNeedAddToScene() 对 layer 节点进行标脏。在layer属性发生改变时会进行标脏；当子layer添加到父layer时也会进行标脏。

这个方法是唯一用到使用到 _needsAddToScene 标脏flag值的地方。 如果 _needsAddToScene 为false ，调用 builder.addRetained(_engineLayer); 这个 _engineLayer

_eingineLayer 是 layer.addToScene(...) 返回的engine生成的图层（ContainerLayer例外，没有engineLayer）

```dart
  /**
   * 这是 OpacityLayer 的addToScene方法，生成了一个 engineLayer ，
   * 然后调用addChildrenToScene(builder);
  **/
  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    bool enabled = firstChild != null;  // don't add this layer if there's no child
    if (enabled)
      engineLayer = builder.pushOpacity(alpha, offset: offset + layerOffset, oldLayer: _engineLayer);
    else
      engineLayer = null;
    addChildrenToScene(builder);
    if (enabled)
      builder.pop();
  }
```

**SceneBuilder.addRetained()**

```dart
  void addRetained(EngineLayer retainedLayer) {
    final _EngineLayerWrapper wrapper = retainedLayer;
    _addRetained(wrapper._nativeLayer);
  }

  void _addRetained(EngineLayer retainedLayer) native 'SceneBuilder_addRetained';
```

addRetained 直接调用了engine层的代码，对layer进行复用。。。

### 直接使用 layer 进行UI绘制

flutter最终是绘制在layer上的，那样的话，实际上，可以脱离 Widget、Element、RenderObject ，直接操作 layer 进行UI的绘制

```dart
import 'dart:ui';
import 'dart:math';
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';


void main(){
    final OffsetLayer rootLayer = new OffsetLayer();
    final PictureLayer pictureLayer = new PictureLayer(Rect.zero);
    rootLayer.append(pictureLayer);

    PictureRecorder recorder = PictureRecorder();
    Canvas canvas = Canvas(recorder);

    Paint paint = Paint();
    paint.color = Colors.primaries[Random().nextInt(Colors.primaries.length)];

    canvas.drawRect(Rect.fromLTWH(0, 0, 300, 300), paint);
    pictureLayer.picture = recorder.endRecording();
    
    SceneBuilder sceneBuilder = SceneBuilder();
    rootLayer.addToScene(sceneBuilder);

    Scene scene = sceneBuilder.build();
    window.onDrawFrame = (){
      window.render(scene);
      // window.scheduleFrame();
      // 在onDrawFrame回调最后加上scheduleFrame，可以使UI不断刷新，进行动画操作
      // 不是很懂那vsync的作用是什么。。。
    };
    window.scheduleFrame();
}
```

## 参考

在 main.dart 中添加 `debugRepaintRainbowEnabled = true;` ，或者通过 devtools 可以观察到App的layer树

[Flutter Framework 源码解析（ 2 ）—— 图层详解](https://xieguanglei.github.io/blog/post/flutter-code-chapter-02.html)

[从源码看flutter（四）：Layer篇](https://juejin.im/post/6844904133728665614#heading-10)

[Flutter画面渲染全面解析](https://guoshuyu.cn/home/wx/Flutter-21.html)




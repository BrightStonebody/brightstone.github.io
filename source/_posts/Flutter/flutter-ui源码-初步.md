---
title: flutter ui源码 初步
date: 2020-08-14 20:30:42
tags:
- Flutter
---

## Widget, Element, RanderObject 三者之间的关系

Widget纯作为一个配置文件存在，可以理解为一个数据结构。Element作为配置文件的实例化对象，具有生命周期的概念，承载构建上下文数据，且持有RenderObject,系统通过遍历Element来构建RenderObject数据，具体Layout，Paint交给RenderObject来完成。Element更像是一个中间层，隔开了Widget与RenderObject,同时也在正常开发过程中将开发者与RenderObject隔开，开发者大部分情况下只需要关注Widget即可。

## Widget

Widget类和Element类一一对应。Element是通过Widget生成的
Widget是Element的配置数据，保存UI相关参数。Widget十分轻量，可以频繁的销毁和重建。

**Widget.createElement()**

创建Element对象

**Widget.canUpdate(..)**
```dart
static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
}
```

主要用于在Widget树重新build时复用旧的widget，其实具体来说，应该是：是否用新的Widget对象去更新旧UI树上所对应的Element对象的配置；
通过其源码我们可以看到，只要 newWidget 与 oldWidget 的 runtimeType和key 同时相等时就会用newWidget去更新Element对象的配置，否则就会创建新的Element。

### State
一个StatefulWidget类会对应一个State类，State表示与其对应的StatefulWidget要维护的状态，State中的保存UI状态信息.

**createState()**

用于创建和StatefulWidget相关的状态，它在StatefulWidget的生命周期中可能会被多次调用。例如，当一个StatefulWidget同时插入到widget树的多个位置时，Flutter framework就会调用该方法为每一个位置生成一个独立的State实例。createState()会在Element实例化的时候调用，**所以，本质上就是一个StatefulElement对应一个State实例。**

**widget和context**
- widget, 它表示与该State实例关联的widget实例，由Flutter framework动态设置。注意，这种关联并非永久的，因为在应用生命周期中，UI树上的某一个节点的widget实例在重新构建时可能会变化，但**State实例只会在第一次插入到树中时被创建**，当在重新构建时，如果widget被修改了，Flutter framework会动态设置State.widget为新的widget实例。 

- context, StatefulWidget对应的BuildContext，作用同StatelessWidget的BuildContext。Elements继承自BuildContext

**State如何被Widget复用**
只有在Element实例化时会创建新的State。当Widget重新build，Element的updateChild会调用canUpdate(...)方法，若返回true，将child element的widget更新；若返回false，则会重新create新的element。

## Element

最终的UI树其实是由一个个独立的Element节点构成。组件最终的Layout、渲染都是通过RenderObject来完成的，从创建到渲染的大体流程是：根据Widget生成Element，然后创建相应的RenderObject并关联到Element.renderObject属性上，最后再通过RenderObject来完成布局排列和绘制。

下面从Element的 **挂载、更新、卸载**过程来描述Element

### mount

**根Element的mount**

```
- main() 
    - runApp(..)
    - WidgetsFlutterBinding.attachRootWidget(app)
        - RenderObjectToWidgetAdapter<RenderBox>.attachToRenderTree(buildOwner, renderViewElement);
            - createElement()
            - BuildOwner.buildScope(element, () {
                element.mount(null, null);
            })
```

以上是根Element的mount过程，其中rederViewElement是根Elemet

** Element的mount过程

首先来看一下常用的StatelessElement，StatefulElement的继承关系

```
Element -> ComponentElement -> StatelessElement
                            -> StatefulElement
```

```
- mount()
    - Element.updateInheritance()
    - ComponentElement.firstBuild()
        - Element.rebuild()
        - ComponentElement.performRebuild()
            - built = build()
                -Widget.build()
            - Element.updateChild(child, built, _)
```

Element.updateChild方法非常重要

```dart
@protected
Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
    if (newWidget == null) {
      if (child != null)
        deactivateChild(child);
      return null;
    }
    if (child != null) {
      if (child.widget == newWidget) {
        return child;
      }
      if (Widget.canUpdate(child.widget, newWidget)) {
        child.update(newWidget);
        return child;
      }
      deactivateChild(child);
    }
    return inflateWidget(newWidget, newSlot);
}

static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
}
```

根据上面的代码，总结各种case

* child != null && newWidget == null :
    newWidget为空，说明 Widget.build返回了null， child element对应的widget已经不在widget树上，所以也要移除该element
* child == null && newWidget != null : 
    第一次挂载child Element，解析build出的widget，调用 inflateWidget(newWidget, newSlot)
* child != null && newWidget != null :
    更新Element，StatefulWidget 调用setState时会进入这个分支。 会调用 child.canUpdate这个方法，判断是否需要更新Element。如果canUpdate返回false，移除这个element然后重新创建
* child == null && newWidget == null :
    do nothing

在mount过程会进入第二个分支，调用 inflateWidget(newWidget, newSlot)

```dart
@protected
Element inflateWidget(Widget newWidget, dynamic newSlot) {
    ...

    final Element newChild = newWidget.createElement();
    newChild.mount(this, newSlot);
    return newChild;
}
```

```dart
abstract class StatefulWidget extends Widget {
  const StatefulWidget({ Key key }) : super(key: key);

  @override
  StatefulElement createElement() => StatefulElement(this);

  State createState();
}

class StatefulElement extends ComponentElement {

  StatefulElement(StatefulWidget widget)
      : _state = widget.createState(),
        super(widget) {
    _state._element = this;
    _state._widget = widget;
  }

  @override
  Widget build() => state.build(this);
}
```
StatefulWidget，State，StatefulElement的调用关系： 
Widget.createElement() -> Element.build() -> State.build()

### 更新Element

```dart
Widget.setState
    - Element.markNeedsBuild()
        - Element._dirty = true
        - BuildOwner.scheduleBuildFor(this)
            - BuildOwner._dirtyElements.add(element)
```

当调用statState之后，会将element标脏，并加入到_dirtyElements列表中

```dart
- WidgetsBinding.drawFrame()
    - BuildOwner.buildScope(renderViewElement)
        - BuildOwner._dirtyElements[index].rebuild()
```

每次UI刷新时，会回调WidgetBinding的drawFrame(), BuildOwner会将_dirtyElements列表rebuild。
可以翻上去查看挂载阶段rebuild之后的调用链，是完全一样的

### 卸载Element

在Element的updateChild方法中，在 `child != null && newWidget == null` 时，会调用deactivateChild()

```dart
  @protected
  void deactivateChild(Element child) {
    child._parent = null;
    child.detachRenderObject();
    owner._inactiveElements.add(child); // this eventually calls child.deactivate()
  }
```
deactivateChild()方法中，会将child element加入到 BuildOwner的_inactiveElements集合中。

还是WidgetBind.drawFrame()方法，会调用 buildOwner.finalizeTree(); 对不再使用的Element进行unmount

```dart
void finalizeTree() {
    Timeline.startSync('Finalize tree', arguments: timelineWhitelistArguments);
    try {
      lockState(() {
        _inactiveElements._unmountAll(); // this unregisters the GlobalKeys
      });
    }
    ...
}

void _unmountAll() {
    _locked = true;
    final List<Element> elements = _elements.toList()..sort(Element._sort);
    _elements.clear();
    try {
      elements.reversed.forEach(_unmount);
    } finally {
      _locked = false;
    }
}

void _unmount(Element element) {
    element.visitChildren((Element child) {
      _unmount(child);
    });
    element.unmount();
}
```

## RenderObject

### RenderObjectElement

RenderObjectElement使用RenderObjectWidget作为配置文件。在RenderTree中有一个与之对应的RenderObject，用来执行具体的测量，绘制等操作。**不是所有的Element都有对应的RenderObject**

RenderObjectElement有三个常用的子类：

* LeafRenderObjectElement：Leaf render objects, with no children
* SingleChildRenderObjectElement：A single child
* MultiChildRenderObjectElement：A linked list of children.

RenderObjectElement的mount、update、unmount逻辑与ComponentElement大致相同，只不过加上了RenderObject的相关逻辑。

**mmount、update、unmount**

```dart
  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _renderObject = widget.createRenderObject(this);
    attachRenderObject(newSlot);
    _dirty = false;
  }

  @override
  void attachRenderObject(dynamic newSlot) {
    _slot = newSlot;
    _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
    _ancestorRenderObjectElement?.insertChildRenderObject(renderObject, newSlot);
    final ParentDataElement<RenderObjectWidget> parentDataElement = _findAncestorParentDataElement();
    if (parentDataElement != null)
      _updateParentData(parentDataElement.widget);
  }

  @override
  void update(covariant RenderObjectWidget newWidget) {
    super.update(newWidget);
    widget.updateRenderObject(this, renderObject);
    _dirty = false;
  }
```

### RenderObject的更新

我们从render树的insert过程类分析RenderObject的更新
从RenderObjectElement.insertChildRenderObject开始

```dart
  @override
  void insertChildRenderObject(RenderObject child, Element slot) {
    final ContainerRenderObjectMixin<RenderObject, ContainerParentDataMixin<RenderObject>> renderObject = this.renderObject;
    renderObject.insert(child, after: slot?.renderObject);
  }
```

这里调用了ContainerRenderObjectMixin的insert方法，

```dart
  void insert(ChildType child, { ChildType after }) {
    adoptChild(child);
    _insertIntoChildList(child, after: after);
  }

  @override
  void adoptChild(RenderObject child) {
    setupParentData(child);
    markNeedsLayout();
    markNeedsCompositingBitsUpdate();
    markNeedsSemanticsUpdate();
    super.adoptChild(child);
  }

  void markNeedsLayout() {
    if (_relayoutBoundary != this) {
      markParentNeedsLayout();
    } else {
      _needsLayout = true;
      if (owner != null) {
        owner._nodesNeedingLayout.add(this);
        owner.requestVisualUpdate();
      }
    }
  }

  @protected
  void markParentNeedsLayout() {
    _needsLayout = true;
    final RenderObject parent = this.parent;
    if (!_doingThisLayoutWithCallback) {
      parent.markNeedsLayout();
    } else {
      assert(parent._debugDoingThisLayout);
    }
  }
```

和element的build流程不同，RenderObject的标脏会向上标脏，直到找到一个 relayoutBoundary 位置。找到 _relayoutBoundary 节点，触发`owner._nodesNeedingLayout.add( this )`与`owner.requestVisualUpdate()` 其中owner._nodesNeedingLayout.add( this )将当前RenderObject注册进PipelineOwner的待刷新列表中，然后触发“VisualUpdate”。这里的owner是PipelineOwner

PiplelineOwner.requestVisualUpdate会请求engine进行一次刷新。。
(todo 不是标脏吗？ 难道不是等待vsync信号对标脏的node进行重绘？？？)
来看一下PiplelineOwner.requestVisualUpdate

```dart
  void requestVisualUpdate() {
    if (onNeedVisualUpdate != null)
      onNeedVisualUpdate();
  }
```
onNeedVisualUpdate是在PiplelineOwner实例化的时候赋值的，在RendererBinding的initInstances里

```dart
  void initInstances() {
    super.initInstances();
    _instance = this;
    _pipelineOwner = PipelineOwner(
      onNeedVisualUpdate: ensureVisualUpdate,
      onSemanticsOwnerCreated: _handleSemanticsOwnerCreated,
      onSemanticsOwnerDisposed: _handleSemanticsOwnerDisposed,
    );
    window
      ..onMetricsChanged = handleMetricsChanged
      ..onTextScaleFactorChanged = handleTextScaleFactorChanged
      ..onPlatformBrightnessChanged = handlePlatformBrightnessChanged
      ..onSemanticsEnabledChanged = _handleSemanticsEnabledChanged
      ..onSemanticsAction = _handleSemanticsAction;
    initRenderView();
    _handleSemanticsEnabledChanged();
    assert(renderView != null);
    addPersistentFrameCallback(_handlePersistentFrameCallback);
    initMouseTracker();
  }

  void _handlePersistentFrameCallback(Duration timeStamp) {
    drawFrame();
  }
```

ensureVisualUpdate 会调用window.scheduleFrame()请求native层绘制新帧。。。没太搞懂为什么要主动请求绘制


### layout

**RendererBinding.drawFrame**

```dart
  @protected
  void drawFrame() {
    ...
    pipelineOwner.flushLayout();
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();
    renderView.compositeFrame(); // this sends the bits to the GPU
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
  }
```

**pipelineOwner.flushLayout()**
```dart
  void flushLayout() {
    try {
      while (_nodesNeedingLayout.isNotEmpty) {
        final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
        _nodesNeedingLayout = <RenderObject>[];
        for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {
          if (node._needsLayout && node.owner == this)
            node._layoutWithoutResize();
        }
      }
    } finally {
      ...
    }
  }
```

***RendererObject._layoutWithoutResize**

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

这里调用performLayout()进行layout，并标记需要paint

### paint

之前说到过，layout过程中会对paint进行标脏

**markNeedsPaint()**
```dart
  void markNeedsPaint() {
    if (_needsPaint)
      return;
    _needsPaint = true;
    if (isRepaintBoundary) {
      if (owner != null) {
        owner._nodesNeedingPaint.add(this);
        owner.requestVisualUpdate();
      }
    } else if (parent is RenderObject) {
      final RenderObject parent = this.parent;
      parent.markNeedsPaint();
    } else {
      // If we're the root of the render tree (probably a RenderView),
      // then we have to paint ourselves, since nobody else can paint
      // us. We don't add ourselves to _nodesNeedingPaint in this
      // case, because the root is always told to paint regardless.
      if (owner != null)
        owner.requestVisualUpdate();
    }
  }
```

和layout的标脏过程类似，一直向上标脏，直到 isRepaintBoundary 为true。需要说明的是，我们可以 RepaintBoundary 是一个widget，我们可以手动添加，来终止paint的向上标脏过程，来达到局部重绘，提升性能。 layoutBoundary 则是framework层帮我们判断的，我们不需要手动处理。

**pipelineOwner.flushLayout()**

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
      ...
    }
  }
```

与layout不同的是这里的 sort 规则是相反的，最深的节点先重绘。并且，多了判断 `node._layer.attached`, 如果为true，重绘，并且重绘逻辑交给了PaintContext；如果为false，则执行 `node._skippedPaintingOnLayer();`

**node._skippedPaintingOnLayer()**

```dart
  void _skippedPaintingOnLayer() {
    AbstractNode ancestor = parent;
    while (ancestor is RenderObject) {
      final RenderObject node = ancestor;
      if (node.isRepaintBoundary) {
        if (node._layer == null)
          break; // looks like the subtree here has never been painted. let it handle itself.
        if (node._layer.attached)
          break; // it's the one that detached us, so it's the one that will decide to repaint us.
        node._needsPaint = true;
      }
      ancestor = node.parent;
    }
  }
```

我的理解是，该节点的 layer 被 detached 了，需要对该节点及其父节点行重新标脏，保证该节点的 layer 被重新 attach 之后能够重绘。 

**PaintingContext.repaintCompositedChild(node);**

```dart
  static void repaintCompositedChild(RenderObject child, { bool debugAlsoPaintedParent = false }) {
    _repaintCompositedChild(
      child,
      debugAlsoPaintedParent: debugAlsoPaintedParent,
    );
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
```

又是 PaintContext 和 layer。。。先跳过这两个，最终重绘的逻辑在 `child._paintWithContext(childContext, Offset.zero);`中

**child._paintWithContext(childContext, Offset.zero);**

```dart
  void _paintWithContext(PaintingContext context, Offset offset) {
    // 一堆注释解释了为啥在paint的时候 _needsLayout 还可能为true 。。。
    // 英文注释没咋看懂
    if (_needsLayout)
      return;
    RenderObject debugLastActivePaint;
    _needsPaint = false;
    try {
      // 最终的重绘交给具体的 RenderObject 子类
      paint(context, offset);
    } catch (e, stack) {
      _debugReportException('paint', e, stack);
    }
  }
```


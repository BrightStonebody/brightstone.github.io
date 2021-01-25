---
title: redux源码
date: 2021-01-24 21:40:51
tags: 
- Flutter
- Redux
---


Redux是一个Flutter状态管理工具。实现并不复杂，其核心是一个简单的观察者模式。我们从 观察者 和 被观察者 两个角度来分析 Redux

# 被观察者 Store/State

被观察者 明显是 Store 和 State 。 State 是状态的 model类 。 Store 是 State 的持有者。

```dart
class Store<State> {
  Reducer<State> reducer;

  final StreamController<State> _changeController;
  State _state;
  List<NextDispatcher> _dispatchers;

  Store(
    this.reducer, {
    State initialState,
    List<Middleware<State>> middleware = const [],
    bool syncStream = false,
    bool distinct = false,
  }) : _changeController = StreamController.broadcast(sync: syncStream) {
    _state = initialState;
    _dispatchers = _createDispatchers(
      middleware,
      _createReduceAndNotify(distinct),
    );
  }

  NextDispatcher _createReduceAndNotify(bool distinct) {
    return (dynamic action) {
      final state = reducer(_state, action);

      if (distinct && state == _state) return;

      _state = state;
      _changeController.add(state);
    };
  }

  dynamic dispatch(dynamic action) {
    return _dispatchers[0](action);
  }

  Stream<State> get onChange => _changeController.stream;
}
```

当一个 Store 创建时，会实例化一个 _changeController ，并将 middleware 和 reducer 打包合并成 _dispatchers 集合。
注意看 _createReduceAndNotify 方法， 当每一个 action 发送到 Store 时，会交给 reducer 处理，并将新的 state 给 add 到 _changeController 里。这样就完成了状态的更新。。

是不是很简单。。。

# 观察者

Redux 里观察者是 StoreConnector 。
但是得先提一下 StoreProvider ， StoreProvider 是初始化和存储 Store 的 Widget 。

```dart 
class StoreProvider<S> extends InheritedWidget {
  final Store<S> _store;

  /// Create a [StoreProvider] by passing in the required [store] and [child]
  /// parameters.
  const StoreProvider({
    Key key,
    @required Store<S> store,
    @required Widget child,
  })  : assert(store != null),
        assert(child != null),
        _store = store,
        super(key: key, child: child);

  static Store<S> of<S>(BuildContext context, {bool listen = true}) {
    final type = _typeOf<StoreProvider<S>>();
    final provider = (listen
        ? context.inheritFromWidgetOfExactType(type)
        : context
            .ancestorInheritedElementForWidgetOfExactType(type)
            ?.widget) as StoreProvider<S>;

    if (provider == null) throw StoreProviderError(type);

    return provider._store;
  }

  // Workaround to capture generics
  static Type _typeOf<T>() => T;

  @override
  bool updateShouldNotify(StoreProvider<S> oldWidget) =>
      _store != oldWidget._store;
}
```

StoreProvider 也很简单，就是一个典型的 InheritedWidget 。

现在来看 StoreConnector ，

```dart
class StoreConnector<S, ViewModel> extends StatelessWidget {
  const StoreConnector({
    Key key,
    @required this.builder,
    @required this.converter,
    this.distinct = false,
    this.onInit,
    this.onDispose,
    this.rebuildOnChange = true,
    this.ignoreChange,
    this.onWillChange,
    this.onDidChange,
    this.onInitialBuild,
  })  : assert(builder != null),
        assert(converter != null),
        super(key: key);

  @override
  Widget build(BuildContext context) {
    return _StoreStreamListener<S, ViewModel>(
      store: StoreProvider.of<S>(context),
      builder: builder,
      converter: converter,
      distinct: distinct,
      onInit: onInit,
      onDispose: onDispose,
      rebuildOnChange: rebuildOnChange,
      ignoreChange: ignoreChange,
      onWillChange: onWillChange,
      onDidChange: onDidChange,
      onInitialBuild: onInitialBuild,
    );
  }
}
```

可以看出来，StoreConnector 只是一个壳，实际的工作者是 _StoreStreamListener 。 StoreConnect 唯一做的工作就是调用 `StoreProvider.of<S>(context)` 给 _StoreStreamListener 的 store 赋值。

_StoreStreamListener 是一个 StatefulWidget

```dart
class _StoreStreamListenerState<S, ViewModel>
    extends State<_StoreStreamListener<S, ViewModel>> {

  @override
  void initState() {
    ...

    latestValue = widget.converter(widget.store);
    _createStream();

    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    return widget.rebuildOnChange
        ? StreamBuilder<ViewModel>(
            stream: stream,
            builder: (context, snapshot) => widget.builder(
              context,
              latestValue,
            ),
          )
        : widget.builder(context, latestValue);
  }

  void _createStream() {
    stream = widget.store.onChange
        .where(_ignoreChange)
        .map(_mapConverter)
        .where(_whereDistinct)
        .transform(StreamTransformer.fromHandlers(handleData: _handleChange));
  }

  ViewModel _mapConverter(S state) {
    return widget.converter(widget.store);
  }

  bool _whereDistinct(ViewModel vm) {
    if (widget.distinct) {
      return vm != latestValue;
    }

    return true;
  }

  void _handleChange(ViewModel vm, EventSink<ViewModel> sink) {
    ...
    latestValue = vm;
    ...
    sink.add(vm);
  }
}
```

在 build 方法中 widget.rebuildOnChange 默认为 true 。 
StreamBuilder 将 stream 转化为 widget 。
stream 就是对 widget.store.onChange (Store 里的 _changeController) 的监听。

需要说明的是， StoreConnector 只是在 Flutter Framework 基础上添加的额外的监听机制。
之前自己一直有个误区： 当 StoreConnector 的外部 Widget 执行 rebuild 之后， StoreConnector 可以控制 child 不 rebuild ，只要 vm 没有发生改变。
这显然不正确。有空可以看看 StreamBuilder 的源码。

# Action 事件的执行

```dart
class Store {
  dynamic dispatch(dynamic action) {
    return _dispatchers[0](action);
  }
}
```

。。。很简单， Store.dispath 从 _dispatchers 的第一个位置开始分发事件。。





# Flutter Redux 源码分析

[TOC]

## 流程

<img src = 'https://user-gold-cdn.xitu.io/2018/8/19/1655276c750c9fae?imageView2/0/w/1280/h/960/format/webp/ignore-error/1'></img>



## StoreProvider

```dart
/// 对这个 widget 的所有后代提供一个 Redux 中的 [Store]
/// 这个 widget 通常是一个 App 中的根 widget
/// 通过使用[StoreConnector] 或 [StoreBuilder] 来获取 store
class StoreProvider<S> extends InheritedWidget {
  final Store<S> _store;

  const StoreProvider({
    Key key,
    @required Store<S> store,
    @required Widget child,
  })  : assert(store != null),
        assert(child != null),
        _store = store,
        super(key: key, child: child);

  /// 当后代调用 Store.of 时，将自身注册到 dependents 中
  static Store<S> of<S>(BuildContext context) {
    final type = _typeOf<StoreProvider<S>>();
    final provider =
        context.inheritFromWidgetOfExactType(type) as StoreProvider<S>;

    /// 没有找到 Provider ，抛出异常  
    if (provider == null) throw StoreProviderError(type);

    return provider._store;
  }

  /// 类型匹配  
  static Type _typeOf<T>() => T;

  /// 当 store 和之前的 store 不同时，应该通知 dependents    
  @override
  bool updateShouldNotify(StoreProvider<S> oldWidget) =>
      _store != oldWidget._store;
}
```



## 函数原型

```dart
/// 通过 [BuildContext] 和 [ViewModel] 来建造一个widget。
typedef ViewModelBuilder<ViewModel> = Widget Function(
  BuildContext context,
  ViewModel vm,
);

/// 将整个 [Store] 转换为一个 [ViewModel]。
typedef StoreConverter<S, ViewModel> = ViewModel Function(
  Store<S> store,
);

/// 在初始化 [StoreConnector] 时候的回调，可用于初始化视图
typedef OnInitCallback<S> = void Function(
  Store<S> store,
);

/// 当 [StoreConnector] 被移除时候的回调 
typedef OnDisposeCallback<S> = void Function(
  Store<S> store,
);

/// 是否应该在响应　State　变化的时候调用 converter 来重建
typedef IgnoreChangeTest<S> = bool Function(S state);

/// 在 State 改变之前的回调
typedef OnWillChangeCallback<ViewModel> = void Function(ViewModel viewModel);

/// 在 build 阶段之后的回调
typedef OnDidChangeCallback<ViewModel> = void Function(ViewModel viewModel);

/// 初始化完成后的回调
typedef OnInitialBuildCallback<ViewModel> = void Function(ViewModel viewModel);

/// 根据 Action 的类型来产生不同行为，更新 Store
/// 只能返回 新建的 Store 或者原来的Stroe
typedef State Reducer<State>(State state, dynamic action);

/// 可调用类形式
abstract class ReducerClass<State> {
  State call(State state, dynamic action);
}

/// 中间件，用于拦截
typedef void Middleware<State>(
  Store<State> store,
  dynamic action,
  NextDispatcher next,
);

/// 可调用类形式
abstract class MiddlewareClass<State> {
  void call(Store<State> store, dynamic action, NextDispatcher next);
}

/// 可链式调用的函数，派发器
typedef void NextDispatcher(dynamic action);
```



## _StoreStreamListener

```dart
/// 将 Store 转换为小数据，构建UI
class _StoreStreamListener<S, ViewModel> extends StatefulWidget {
  final ViewModelBuilder<ViewModel> builder;
  final StoreConverter<S, ViewModel> converter;
  final Store<S> store;
  final bool rebuildOnChange;
  /// 是否是独一的，即时候允许有相同的VM  
  final bool distinct;
  final OnInitCallback<S> onInit;
  final OnDisposeCallback<S> onDispose;
  final IgnoreChangeTest<S> ignoreChange;
  final OnWillChangeCallback<ViewModel> onWillChange;
  final OnDidChangeCallback<ViewModel> onDidChange;
  final OnInitialBuildCallback<ViewModel> onInitialBuild;

  _StoreStreamListener({
    Key key,
    @required this.builder,
    @required this.store,
    @required this.converter,
    this.distinct = false,
    this.onInit,
    this.onDispose,
    this.rebuildOnChange = true,
    this.ignoreChange,
    this.onWillChange,
    this.onDidChange,
    this.onInitialBuild,
  }) : super(key: key);

  @override
  State<StatefulWidget> createState() {
    return _StoreStreamListenerState<S, ViewModel>();
  }
}

class _StoreStreamListenerState<S, ViewModel>
    extends State<_StoreStreamListener<S, ViewModel>> {
  /// 以 ViewModel 作为元素的　Stream   
  Stream<ViewModel> stream;
  /// 最后一次更新的VM 
  ViewModel latestValue;

  @override
  void initState() {
    _init();
    super.initState();
  }

  @override
  void dispose() {
    if (widget.onDispose != null) {
      widget.onDispose(widget.store);
    }
    super.dispose();
  }

  @override
  void didUpdateWidget(_StoreStreamListener<S, ViewModel> oldWidget) {
    if (widget.store != oldWidget.store) {
      _init();
    }
    super.didUpdateWidget(oldWidget);
  }
  
  void _init() {
    /// 在这里调用了onInit  
    if (widget.onInit != null) {
      widget.onInit(widget.store);
    }

    /// 保存最后上一次更新  
    latestValue = widget.converter(widget.store);

    /// 注册帧尾回调  
    if (widget.onInitialBuild != null) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        widget.onInitialBuild(latestValue);
      });
    }

    /// 获取到 Store 中的流，其中包含　State  
    var _stream = widget.store.onChange;

    /// 忽略无需改变的　state  
    if (widget.ignoreChange != null) {
      _stream = _stream.where((state) => !widget.ignoreChange(state));
    }

    /// 创建VM流 
    /// ！注意！
    /// 由于流的特性，每次当stream（即store.onchange,即store._changeController）发生变化时，
    /// _stream都会随之采取对应的行动
      
    /// 每当stream变化，都将调用convertor来产生VM  
    stream = _stream.map((_) => widget.converter(widget.store));
  
    // Don't use `Stream.distinct` because it cannot capture the initial
    // ViewModel produced by the `converter`.  
    if (widget.distinct) {
      stream = stream.where((vm) { 
        final isDistinct = vm != latestValue;
        return isDistinct;
      });
    }

    /// 对所有的VM进行操作，产生新的 VM stream  
    stream =
     　 stream.transform(StreamTransformer.fromHandlers(
        　　handleData: (vm, sink) {
            　latestValue = vm;	      
            　if (widget.onWillChange != null) {
             　widget.onWillChange(latestValue);

            　if (widget.onDidChange != null) {
                WidgetsBinding.instance.addPostFrameCallback(
                    (_) {widget.onDidChange(latestValue);}
                );
                sink.add(vm);
            }
        　)
     );
  }

  @override
  Widget build(BuildContext context) {
    /// 发生改变时是否要重建
    /// 类似于Provider.of的listen参数  
    return widget.rebuildOnChange
        ? StreamBuilder<ViewModel>(
            stream: stream,
        	/// 每次当VM流发生改变的时候重建
            builder: (context, snapshot) => widget.builder(
                  context,
                  snapshot.hasData ? snapshot.data : latestValue,
                ),
          )
        : widget.builder(context, latestValue);
  }
}
```



## StoreConnector & StoreBuilder

```dart
/// 一下两个类简单封装了_StoreStreamListener
class StoreConnector<S, ViewModel> extends StatelessWidget {
  final ViewModelBuilder<ViewModel> builder;
  final StoreConverter<S, ViewModel> converter;
  final bool distinct;
  final OnInitCallback<S> onInit;
  final OnDisposeCallback<S> onDispose;

  /// 决定当 Store 发射一个 onChange 事件时是否应该重建 widget
  final bool rebuildOnChange;
    
  final IgnoreChangeTest<S> ignoreChange;
  final OnWillChangeCallback<ViewModel> onWillChange;
  final OnDidChangeCallback<ViewModel> onDidChange;
  final OnInitialBuildCallback<ViewModel> onInitialBuild;
    
  StoreConnector({
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

class StoreBuilder<S> extends StatelessWidget {
  static Store<S> _identity<S>(Store<S> store) => store;
    
  final ViewModelBuilder<Store<S>> builder;
  final bool rebuildOnChange;
  final OnInitCallback<S> onInit;
  final OnDisposeCallback<S> onDispose;
  final OnWillChangeCallback<Store<S>> onWillChange;
  final OnDidChangeCallback<Store<S>> onDidChange;
  final OnInitialBuildCallback<Store<S>> onInitialBuild;

  StoreBuilder({
    Key key,
    @required this.builder,
    this.onInit,
    this.onDispose,
    this.rebuildOnChange = true,
    this.onWillChange,
    this.onDidChange,
    this.onInitialBuild,
  })  : assert(builder != null),
        super(key: key);

  @override
  Widget build(BuildContext context) {
    return StoreConnector<S, Store<S>>(
      builder: builder,
      converter: _identity,
      rebuildOnChange: rebuildOnChange,
      onInit: onInit,
      onDispose: onDispose,
      onWillChange: onWillChange,
      onDidChange: onDidChange,
      onInitialBuild: onInitialBuild,
    );
  }
}
```



## Utils

```dart
/// 类型Reducer
/// 根据Action的类型来确定是否调用Reducer
class TypedReducer<State, Action> implements ReducerClass<State> {
  final State Function(State state, Action action) reducer;

  TypedReducer(this.reducer);

  @override
  State call(State state, dynamic action) {
    if (action is Action) {
      return reducer(state, action);
    }

    return state;
  }
}

/// 类型Middleware
/// 在Action到达Store之前，会被中间件拦截
/// 之后（通常）会链式调用NextDispatcher
/// 最后会到达Store
class TypedMiddleware<State, Action> implements MiddlewareClass<State> {
  final void Function(
    Store<State> store,
    Action action,
    NextDispatcher next,
  ) middleware;

  TypedMiddleware(this.middleware);

  @override
  void call(Store<State> store, dynamic action, NextDispatcher next) {
    if (action is Action) {
      middleware(store, action, next);
    } else {
      next(action);
    }
  }
}

/// 函数闭包
/// 根据reducers返回一个能够依次调用其中reducer的函数
/// 这个通常用于Action判断
Reducer<State> combineReducers<State>(Iterable<Reducer<State>> reducers) {
  return (State state, dynamic action) {
    for (final reducer in reducers) {
      state = reducer(state, action);
    }
    return state;
  };
}
```



## Store

```dart
/// 数据储存
class Store<State> {

  /// Reducer  
  Reducer<State> reducer;

  final StreamController<State> _changeController;
  State _state;
  /// 内部根据中间件自动创建  
  List<NextDispatcher> _dispatchers;

  Store(
    this.reducer, {
    /// 初始State    
    State initialState,
    List<Middleware<State>> middleware = const [],
    /// 是否是同步流    
    bool syncStream: false,
    bool distinct: false,
  }) :  _changeController = new StreamController.broadcast(sync: syncStream) {
        _state = initialState;
        _dispatchers = _createDispatchers(
           middleware,
           _createReduceAndNotify(distinct
        ),
    );
  }

  State get state => _state;

  /// 向外暴露流  
  Stream<State> get onChange => _changeController.stream;

  // 最后一个派发件，将action传递给Reducer
  NextDispatcher _createReduceAndNotify(bool distinct) {
    return (dynamic action) {
      /// 通过reducer产生新的state  
      final state = reducer(_state, action);

      if (distinct && state == _state) return;
	  /// 自身持有State	  
      _state = state;
      /// 将State加入流中，触发更新  
      _changeController.add(state);
    };
  }

  /// 根据传入的中间件自动创建 NextDispatchers 
  List<NextDispatcher> _createDispatchers(
    List<Middleware<State>> middleware,
    NextDispatcher reduceAndNotify,
  ) {
    final dispatchers = <NextDispatcher>[]..add(reduceAndNotify);

    /// !注意!
    /// 时间复杂度O(n^2)  
    /// 从 middleware 最后向前生成 dispatcher
    /// 最后一个dispatcher的next是_createDispatchers
    /// _createDispatchers <= 3 <= 2 <= 1 
    for (var nextMiddleware in middleware.reversed) {
      final next = dispatchers.last;

      dispatchers.add(
        /// this 是这个Store，即nextMiddleware的参数Store
        (dynamic action) => nextMiddleware(this, action, next),
      );
    }

    /// 逆置成正确调用顺序  
    return dispatchers.reversed.toList();
  }

  /// 只需调用_dispatchers[0]，
  /// 即可链式传递action  
  void dispatch(dynamic action) {
    _dispatchers[0](action);
  }

  /// dispose  
  Future teardown() async {
    _state = null;
    return _changeController.close();
  }
}
```


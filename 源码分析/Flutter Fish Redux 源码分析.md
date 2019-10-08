# Flutter Fish Redux 源码分析



[TOC]

## FishRedux - github

https://github.com/alibaba/fish-redux


## 总览
### 推荐的项目结构

```dart
sample_page
    -- action.dart  /// Define action types and action creator
    -- page.dart    /// Config a page or component
    -- view.dart    /// Define a function which expresses the presentation of user 		
                    ///    interface
    -- effect.dart  /// Define a function which handles the side-effect
    -- reducer.dart /// Define a function which handles state-change
    -- state.dart   /// Define a state and some connector of substate
    components
        sample_component
        -- action.dart
        -- component.dart
        -- view.dart
        -- effect.dart
        -- reducer.dart
        -- state.dart
```



### Redux 和 FishRedux 的不同

<div align = 'center'>以下内容摘自 FishRedux 的 Doc</div>
#### 1.它们是解决不同层面问题的两个框架



> Redux 是一个专注于状态管理的框架；Fish Redux 是基于 Redux 做状态管理的应用框架。
应用框架不仅仅要解决状态管理的问题，还要解决分治，通信，数据驱动，解耦等等问题。



#### 2.Fish Redux 解决了集中和分治的矛盾



> Redux 通过使用者手动组织代码的形式来完成从小的 Reducer 到主 Reducer 的合并过程；
Fish Redux 通过显式的表达组件之间的依赖关系，由框架自动完成从细力度的 Reducer 到主 Reducer 的合并过程；

[![img](https://camo.githubusercontent.com/e973ad0e5b01057bd6a68ed5e6d303c0b686f894/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f5442316f65584b4a5950704b31526a535a464658586135507058612d313937362d3536382e706e67)](https://camo.githubusercontent.com/e973ad0e5b01057bd6a68ed5e6d303c0b686f894/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f5442316f65584b4a5950704b31526a535a464658586135507058612d313937362d3536382e706e67)

#### 3.Fish Redux 提供了一个简单的组件抽象模型



> 它通过简单的 3 个函数组合而成

[![img](https://camo.githubusercontent.com/1b9897360edb417b55e46dc7dd4c858d7971ca6c/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f544231767142324a3459614b31526a535a466e58586138307058612d3930302d3738302e706e67)](https://camo.githubusercontent.com/1b9897360edb417b55e46dc7dd4c858d7971ca6c/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f544231767142324a3459614b31526a535a466e58586138307058612d3930302d3738302e706e67)

#### 4.Fish Redux 提供了一个 Adapter 的抽象组件模型



> 在基础的组件模型以外，Fish Redux 提供了一个 Adapter 抽象模型，用来解决在 ListView 上大 Cell 的性能问题。
通过上层抽象，我们得到了逻辑上的 ScrollView，性能上的 ListView。



[![img](https://camo.githubusercontent.com/6f1acb86c10851915f71ad6582f795a13c66d3ee/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f544231783531564a37506f4b31526a535a4b6258585831495858612d313835322d3631322e706e67)](https://camo.githubusercontent.com/6f1acb86c10851915f71ad6582f795a13c66d3ee/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f544231783531564a37506f4b31526a535a4b6258585831495858612d313835322d3631322e706e67)



### 流程图

[<img src="https://user-gold-cdn.xitu.io/2019/4/30/16a6d97f0a8365dc?imageslim" alt="img" style="zoom: 200%;" />](https://user-gold-cdn.xitu.io/2019/4/30/16a6d97f0a8365dc?imageslim)




## Redux

### basic

```dart
/// fish_redux/redux/basic

/// 本文件描述了Redux系统下的核心概念及其标准定义。
/// 主要包含:
/// 1. ReduxJS 社区的内容
///    Action          ---- 通过一个对象来定义行为
///    Reducer<T>      ---- 一个修改数据的纯函数
///    Dispatch        ---- 发送行为
///    Middleware<T>   ---- 面型切面（AOP）
///    Store<T>        ---- 状态管理中心
/// 2. 超出 ReduxJS 社区的基本内容的额外抽象概念
///    Connector<S, P> ---- 连接大型 Object<S> 和小型 Object<P>
///    SubReducer<T>   ---- 一个修改部分数据<T>的函数 
///    做出此层抽象的原因：
///    a. 显然，combineReducers 的实现与JS的语法特征是 非耦合 的；
///    b. 更深层次的是可以解决 Redux集中化 与 组件划分 之间的矛盾；


/// Action 的标准定义
/// Action 是一种定义 "intention" 的方式
///     1. 它强调的是意图的明确性，而不是意图的实现；
///     2. 通常，意图的实现是由Effect或reduce完成的；
///     3. type: "intention" 的类型；
///	       payload（负载）: "intention" 负载的初始信息；
///     4. 行动定义和标准，严格遵循Redux社区的行动定义和标准；
class Action {
  const Action(this.type, {this.payload});
  final Object type;
  final dynamic payload;
}

/// Reducer 的标准定义
/// 如果 Reducer 需要相应一个 Action, 它返回一个新 state,否则反复旧 state.
typedef Reducer<T> = T Function(T state, Action action);

/// Dispatch 的标准定义
/// Send an "intention".
typedef Dispatch = dynamic Function(Action action);

/// subscription function 的标准定义
/// 输入一个 subscriber ，产生 anti-subscription function
typedef Subscribe = void Function() Function(void Function() callback);

/// ReplaceReducer 的定义
typedef ReplaceReducer<T> = void Function(Reducer<T> reducer);

/// Observable flow 的定义
typedef Observable<T> = Stream<T> Function();

/// Synthesizable functions（可合成函数）的定义
typedef Composable<T> = T Function(T next);

/// 类获取器的定义（获取 R 实例）
typedef Get<R> = R Function();

/// Middleware 的标准定义
typedef Middleware<T> = Composable<Dispatch> Function({
  Dispatch dispatch,
  Get<T> getState,
});

/// Store 的标准定义
class Store<T> {
  Get<T> getState;
  Dispatch dispatch;
  Subscribe subscribe;
  Observable<T> observable;
  ReplaceReducer<T> replaceReducer;
  Future<dynamic> Function() teardown;
}

/// 创建一个Store
typedef StoreCreator<T> = Store<T> Function(
  T preloadedState,
  Reducer<T> reducer,
);

/// Definition of Enhanced creating a store
typedef StoreEnhancer<T> = StoreCreator<T> Function(StoreCreator<T> creator);

/// SubReducer 的定义
/// [isStateCopied] 与性能有关，确保 T 在整个过程中只会被复制一次
typedef SubReducer<T> = T Function(T state, Action action, bool isStateCopied);

/// 连接 Reducer<S> 和 Reducer<P> 的 Connector 的定义
/// 1. 如何从 S 实例中获取 P 的实例
/// 2. 如何将 P 实例的更改同步到 S 实例
/// 3. 如何复制一个新的S
abstract class AbstractConnector<S, P> {
  P get(S state);

  /// 对于可变的 state, 需要满足以下三个能力
  ///     1. get: (S) => P
  ///     2. set: (S, P) => void
  ///     3. shallow copy: s.clone()
  ///
  /// 对于不可变的 state,需要满足以下两个能力
  ///     1. get: (S) => P
  ///     2. set: (S, P) => S
  ///   
  SubReducer<S> subReducer(Reducer<P> reducer);
}
```



### applyMiddleware
```dart
StoreEnhancer<T> applyMiddleware<T>(List<Middleware<T>> middleware) {
  return middleware == null || middleware.isEmpty
      ? null
      : (StoreCreator<T> creator) => (T initState, Reducer<T> reducer) {
            assert(middleware != null && middleware.isNotEmpty);

            final Store<T> store = creator(initState, reducer);
            final Dispatch initialValue = store.dispatch;
            store.dispatch = (Action action) {
              throw Exception(
                  'Dispatching while constructing your middleware is not allowed. '
                  'Other middleware would not be applied to this dispatch.'
              );
            };
            store.dispatch = middleware
                .map((Middleware<T> middleware) => middleware(
                      dispatch: (Action action) => store.dispatch(action),
                      getState: store.getState,
                    ))
                .fold(
                  initialValue,
                  (Dispatch previousValue,
                          Dispatch Function(Dispatch) element) =>
                      element(previousValue),
                );

            return store;
          };
}
```



### combineReducers

```dart
/// Combine an iterable of SubReducer<T> into one Reducer<T>
Reducer<T> combineSubReducers<T>(Iterable<SubReducer<T>> subReducers) {
  final List<SubReducer<T>> notNullReducers = subReducers
      ?.where((SubReducer<T> e) => e != null)
      ?.toList(growable: false);

  if (notNullReducers == null || notNullReducers.isEmpty) {
    return null;
  }

  if (notNullReducers.length == 1) {
    final SubReducer<T> single = notNullReducers.single;
    return (T state, Action action) => single(state, action, false);
  }

  return (T state, Action action) {
    T copy = state;
    bool hasChanged = false;
    for (SubReducer<T> subReducer in notNullReducers) {
      copy = subReducer(copy, action, hasChanged);
      hasChanged = hasChanged || copy != state;
    }
    assert(copy != null);
    return copy;
  };
}

/// Combine an iterable of Reducer<T> into one Reducer<T>
Reducer<T> combineReducers<T>(Iterable<Reducer<T>> reducers) {
  final List<Reducer<T>> notNullReducers =
      reducers?.where((Reducer<T> r) => r != null)?.toList(growable: false);
  if (notNullReducers == null || notNullReducers.isEmpty) {
    return null;
  }

  if (notNullReducers.length == 1) {
    return notNullReducers.single;
  }

  return (T state, Action action) {
    T nextState = state;
    for (Reducer<T> reducer in notNullReducers) {
      nextState = reducer(nextState, action);
    }
    assert(nextState != null);
    return nextState;
  };
}

/// Convert a super Reducer<Sup> to a sub Reducer<Sub>
Reducer<Sub> castReducer<Sub extends Sup, Sup>(Reducer<Sup> sup) {
  return sup == null
      ? null
      : (Sub state, Action action) {
          final Sub result = sup(state, action);
          return result;
        };
}
```



### connector

```dart
import 'basic.dart';

/// Define a basic connector for immutable state.
///     /// Example:
///     class State {
///       final SubState sub;
///       final String name;
///       const State({this.sub, this.name});
///     }
///
///     class SubState {}
///
///     class Conn<State, SubState> extends ImmutableConn<State, SubState> {
///       SubState get(State state) => state.sub;
///       State set(State state, SubState sub) => State(sub: sub, name: state.name);
///     }
abstract class ImmutableConn<T, P> implements AbstractConnector<T, P> {
  const ImmutableConn();

  T set(T state, P subState);

  @override
  SubReducer<T> subReducer(Reducer<P> reducer) {
    return (T state, Action action, bool isStateCopied) {
      final P props = get(state);
      if (props == null) {
        return state;
      }
      final P newProps = reducer(props, action);
      final bool hasChanged = !identical(newProps, props);
      if (hasChanged) {
        final T result = set(state, newProps);
        assert(result != null, 'Expected to return a non-null value.');
        return result;
      }
      return state;
    };
  }
}

/// Definition of Cloneable
abstract class Cloneable<T extends Cloneable<T>> {
  T clone();
}

/// how to clone an object
dynamic _clone<T>(T state) {
  if (state is Cloneable) {
    return state.clone();
  } else if (state is List) {
    return state.toList();
  } else if (state is Map<String, dynamic>) {
    return <String, dynamic>{}..addAll(state);
  } else if (state == null) {
    return null;
  } else {
    throw ArgumentError(
        'Could not clone this state of type ${state.runtimeType}.');
  }
}

/// Define a basic connector for mutable state.
///     /// Example:
///     class State implments Cloneable<State>{
///       SubState sub;
///       String name;
///       State({this.sub, this.name});
///
///       State clone() => State(sub: sub, name: name);
///     }
///
///     class SubState {}
///
///     class Conn<State, SubState> extends MutableConn<State, SubState> {
///       SubState get(State state) => state.sub;
///       void set(State state, SubState sub) => state.sub = sub;
///     }
abstract class MutableConn<T, P> implements AbstractConnector<T, P> {
  const MutableConn();

  void set(T state, P subState);

  @override
  SubReducer<T> subReducer(Reducer<P> reducer) {
    return (T state, Action action, bool isStateCopied) {
      final P props = get(state);
      if (props == null) {
        return state;
      }
      final P newProps = reducer(props, action);
      final bool hasChanged = newProps != props;
      final T copy = (hasChanged && !isStateCopied) ? _clone<T>(state) : state;
      if (hasChanged) {
        set(copy, newProps);
      }
      return copy;
    };
  }
}
```



### creatStore

```dart
Reducer<T> _noop<T>() => (T state, Action action) => state;

typedef _VoidCallback = void Function();

void _throwIfNot(bool condition, [String message]) {
  if (!condition) {
    throw ArgumentError(message);
  }
}

Store<T> _createStore<T>(final T preloadedState, final Reducer<T> reducer) {
  _throwIfNot(
    preloadedState != null,
    'Expected the preloadedState to be non-null value.',
  );

  final List<_VoidCallback> _listeners = <_VoidCallback>[];
  final StreamController<T> _notifyController =
      StreamController<T>.broadcast(sync: true);

  T _state = preloadedState;
  Reducer<T> _reducer = reducer ?? _noop<T>();
  bool _isDispatching = false;
  bool _isDisposed = false;

  return Store<T>()
    ..getState = (() => _state)
    ..dispatch = (Action action) {
      _throwIfNot(action != null, 'Expected the action to be non-null value.');
      _throwIfNot(action.type != null,
          'Expected the action.type to be non-null value.');
      _throwIfNot(!_isDispatching, 'Reducers may not dispatch actions.');

      if (_isDisposed) {
        return;
      }

      try {
        _isDispatching = true;
        _state = _reducer(_state, action);
      } finally {
        _isDispatching = false;
      }

      final List<_VoidCallback> _notifyListeners = _listeners.toList(
        growable: false,
      );
      for (_VoidCallback listener in _notifyListeners) {
        listener();
      }

      _notifyController.add(_state);
    }
    ..replaceReducer = (Reducer<T> replaceReducer) {
      _reducer = replaceReducer ?? _noop;
    }
    ..subscribe = (_VoidCallback listener) {
      _throwIfNot(
        listener != null,
        'Expected the listener to be non-null value.',
      );
      _throwIfNot(
        !_isDispatching,
        'You may not call store.subscribe() while the reducer is executing.',
      );

      _listeners.add(listener);

      return () {
        _throwIfNot(
          !_isDispatching,
          'You may not unsubscribe from a store listener while the reducer is executing.',
        );
        _listeners.remove(listener);
      };
    }
    ..observable = (() => _notifyController.stream)
    ..teardown = () {
      _isDisposed = true;
      _listeners.clear();
      return _notifyController.close();
    };
}

/// create a store with enhancer
Store<T> createStore<T>(T preloadedState, Reducer<T> reducer,
        [StoreEnhancer<T> enhancer]) =>
    enhancer != null
        ? enhancer(_createStore)(preloadedState, reducer)
        : _createStore(preloadedState, reducer);

StoreEnhancer<T> composeStoreEnhancer<T>(List<StoreEnhancer<T>> enhancers) =>
    enhancers == null || enhancers.isEmpty
        ? null
        : enhancers.reduce((StoreEnhancer<T> previous, StoreEnhancer<T> next) =>
            (StoreCreator<T> creator) => next(previous(creator)));

```

## ReduxComponent

### basic

```dart
/// fish_redux/redux_component/basic

/// Component's view part
/// 1.State is used to decide how to render
/// 2.Dispatch is used to send actions
/// 3.ViewService is used to build sub-components or adapter.
typedef ViewBuilder<T> = Widget Function(
  T state,
  Dispatch dispatch,
  ViewService viewService,
);

/// Define a base ListAdapter which is used for ListView.builder.
/// Many small listAdapters could be merged to a bigger one.
class ListAdapter {
  final int itemCount;
  final IndexedWidgetBuilder itemBuilder;
  const ListAdapter(this.itemBuilder, this.itemCount);
}

/// Adapter's view part
/// 1.State is used to decide how to render
/// 2.Dispatch is used to send actions
/// 3.ViewService is used to build sub-components or adapter.
typedef AdapterBuilder<T> = ListAdapter Function(
  T state,
  Dispatch dispatch,
  ViewService viewService,
);

/// Data driven ui
/// 1. How to render
/// 2. When to update
abstract class ViewUpdater<T> {
  Widget buildWidget();
  void didUpdateWidget();
  void onNotify();
  void forceUpdate();
  void clearCache();
}

/// A little different with Dispatch (with if it is interrupted).
/// bool for sync-functions, interrupted if true
/// Futur<void> for async-functions, should always be interrupted.
// typedef OnAction = Dispatch;

/// Predicate if a component should be updated when the store is changed.
typedef ShouldUpdate<T> = bool Function(T old, T now);

/// Interrupt if not null not false
/// bool for sync-functions, interrupted if true
/// Futur<void> for async-functions, should always be interrupted.
typedef Effect<T> = dynamic Function(Action action, Context<T> ctx);

/// AOP on view
/// usage
/// ViewMiddleware<T> safetyView<T>(
///     {Widget Function(dynamic, StackTrace,
///             {AbstractComponent<dynamic> component, Store<T> store})
///         onError}) {
///   return (AbstractComponent<dynamic> component, Store<T> store) {
///     return (ViewBuilder<dynamic> next) {
///       return isDebug()
///           ? next
///           : (dynamic state, Dispatch dispatch, ViewService viewService) {
///               try {
///                 return next(state, dispatch, viewService);
///               } catch (e, stackTrace) {
///                 return onError?.call(
///                       e,
///                       stackTrace,
///                       component: component,
///                       store: store,
///                     ) ??
///                     Container(width: 0, height: 0);
///               }
///             };
///     };
///   };
/// }
typedef ViewMiddleware<T> = Composable<ViewBuilder<dynamic>> Function(
  AbstractComponent<dynamic>,
  Store<T>,
);

/// AOP on adapter
typedef AdapterMiddleware<T> = Composable<AdapterBuilder<dynamic>> Function(
  AbstractAdapter<dynamic>,
  Store<T>,
);

/// AOP on effect
/// usage
/// EffectMiddleware<T> pageAnalyticsMiddleware<T>() {
///   return (AbstractLogic<dynamic> logic, Store<T> store) {
///     return (Effect<dynamic> effect) {
///       return effect == null ? null : (Action action, Context<dynamic> ctx) {
///         if (logic is Page<dynamic, dynamic>) {
///           print('${logic.runtimeType} ${action.type.toString()} ${ctx.hashCode}');
///         }
///         return effect(action, ctx);
///       };
///     };
///   };
/// }
typedef EffectMiddleware<T> = Composable<Effect<dynamic>> Function(
  AbstractLogic<dynamic>,
  Store<T>,
);

/// AOP in page on store, view, adapter, effect...
abstract class Enhancer<T> {
  ViewBuilder<K> viewEnhance<K>(
    ViewBuilder<K> view,
    AbstractComponent<K> component,
    Store<T> store,
  );

  AdapterBuilder<K> adapterEnhance<K>(
    AdapterBuilder<K> adapterBuilder,
    AbstractAdapter<K> logic,
    Store<T> store,
  );

  Effect<K> effectEnhance<K>(
    Effect<K> effect,
    AbstractLogic<K> logic,
    Store<T> store,
  );

  StoreCreator<T> storeEnhance(StoreCreator<T> creator);

  void unshift({
    List<Middleware<T>> middleware,
    List<ViewMiddleware<T>> viewMiddleware,
    List<EffectMiddleware<T>> effectMiddleware,
    List<AdapterMiddleware<T>> adapterMiddleware,
  });

  void append({
    List<Middleware<T>> middleware,
    List<ViewMiddleware<T>> viewMiddleware,
    List<EffectMiddleware<T>> effectMiddleware,
    List<AdapterMiddleware<T>> adapterMiddleware,
  });
}

/// AOP End

abstract class ExtraData {
  /// Get|Set extra data in context if needed.
  Map<String, Object> get extra;
}

/// Seen in view-part or adapter-part
abstract class ViewService implements ExtraData {
  /// The way to build adapter which is configured in Dependencies.list
  ListAdapter buildAdapter();

  /// The way to build slot component which is configured in Dependencies.slots
  Widget buildComponent(String name);

  /// Get BuildContext from the host-widget
  BuildContext get context;

  /// Broadcast action(the intent) in app (inter-pages)
  void broadcast(Action action);

  /// Broadcast in all component receivers;
  void broadcastEffect(Action action, {bool excluded});
}

///  Seen in effect-part
abstract class Context<T> extends AutoDispose implements ExtraData {
  /// Get the latest state
  T get state;

  /// The way to send action, which will be consumed by self, or by broadcast-module and store.
  dynamic dispatch(Action action);

  /// Get BuildContext from the host-widget
  BuildContext get context;

  /// In general, we should not need this field.
  /// When we have to use this field, it means that we have encountered difficulties.
  /// This is a contradiction between presentation & logical separation, and Flutter's Widgets system.
  ///
  /// How to use ?
  /// For example, we want to use SingleTickerProviderStateMixin
  /// We should
  /// 1. Define a new Component mixin SingleTickerProviderMixin
  ///    class MyComponent<T> extends Component<T> with SingleTickerProviderMixin<T> {}
  /// 2. Get the CustomStfState via context.stfState in Effect.
  ///    /// Through BuildContext -> StatefulElement -> State
  ///    final TickerProvider tickerProvider = context.stfState;
  ///    AnimationController controller = AnimationController(vsync: tickerProvider);
  ///    context.dispatch(ActionCreator.createController(controller));
  State get stfState;

  /// The way to build slot component which is configured in Dependencies.slots
  /// such as custom mask or dialog
  Widget buildComponent(String name);

  /// Broadcast action in app (inter-stores)
  void broadcast(Action action);

  /// Broadcast in all component receivers;
  void broadcastEffect(Action action, {bool excluded});

  /// add observable
  void Function() addObservable(Subscribe observable);

  void forceUpdate();

  /// listen on the changes of some parts of <T>.
  void Function() listen({
    bool Function(T, T) isChanged,
    void Function() onChange,
  });
}

/// Seen in framework-component
abstract class ContextSys<T> extends Context<T> implements ViewService {
  /// Response to lifecycle calls
  void onLifecycle(Action action);

  void bindForceUpdate(void Function() forceUpdate);

  Store<dynamic> get store;

  Enhancer<dynamic> get enhancer;

  DispatchBus get bus;
}

/// Representation of each dependency
abstract class Dependent<T> {
  Get<Object> subGetter(Get<T> getter);

  SubReducer<T> createSubReducer();

  Widget buildComponent(
    Store<Object> store,
    Get<T> getter, {
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  });

  /// P state
  ListAdapter buildAdapter(ContextSys<Object> ctx);

  ContextSys<Object> createContext(
    Store<Object> store,
    BuildContext buildContext,
    Get<T> getState, {
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  });

  bool isComponent();

  bool isAdapter();
}

/// Encapsulation of the logic part of the component
/// The logic is divided into two parts, Reducer & SideEffect.
abstract class AbstractLogic<T> {
  /// To create a reducer<T>
  Reducer<T> get reducer;

  /// To solve Reducer<Object> is neither a subtype nor a supertype of Reducer<T> issue.
  Object onReducer(Object state, Action action);

  /// To create each instance's side-effect-action-handler
  Dispatch createEffectDispatch(ContextSys<T> ctx, Enhancer<Object> enhancer);

  /// To create each instance's side-effect-action-handler
  Dispatch createNextDispatch(ContextSys<T> ctx, Enhancer<Object> enhancer);

  /// To create each instance's dispatch
  /// Dispatch is the most important api for users which is provided by framework
  Dispatch createDispatch(
    Dispatch effectDispatch,
    Dispatch nextDispatch,
    ContextSys<T> ctx,
  );

  /// To create each instance's context
  ContextSys<T> createContext(
    Store<Object> store,
    BuildContext buildContext,
    Get<T> getState, {
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  });

  /// To create each instance's key (for recycle) if needed
  Object key(T state);

  /// Find a dependent by name
  Dependent<T> slot(String name);

  /// Get a adapter-dependent
  Dependent<T> adapterDep();

  Type get propertyType;
}

abstract class AbstractComponent<T> implements AbstractLogic<T> {
  /// How to build component instance
  Widget buildComponent(
    Store<Object> store,
    Get<T> getter, {
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  });
}

abstract class AbstractAdapter<T> implements AbstractLogic<T> {
  ListAdapter buildAdapter(ContextSys<T> ctx);
}

/// Because a main reducer will be very complicated with multiple level's state.
/// When a reducer is slow to handle an action, maybe we should use ReducerFilter to improve the performance.
typedef ReducerFilter<T> = bool Function(T state, Action action);

///
abstract class DispatchBus {
  void attach(DispatchBus parent);

  void detach();

  void dispatch(Action action, {Dispatch excluded});

  void broadcast(Action action, {DispatchBus excluded});

  void Function() registerReceiver(Dispatch dispatch);
}
```



### context

```dart
/// fish_redux/component/context

mixin _ExtraMixin {
  Map<String, Object> _extra;
  Map<String, Object> get extra => _extra ??= <String, Object>{};
}

/// Default Context
abstract class LogicContext<T> extends ContextSys<T> with _ExtraMixin {
  final AbstractLogic<T> logic;

  @override
  final Store<Object> store;
  @override
  final DispatchBus bus;
  @override
  final Enhancer<Object> enhancer;

  final Get<T> getState;

  void Function() _forceUpdate;

  BuildContext _buildContext;
  Dispatch _dispatch;
  Dispatch _effectDispatch;

  LogicContext({
    @required this.logic,
    @required this.store,
    @required BuildContext buildContext,
    @required this.getState,

    /// pageBus
    @required this.bus,
    @required this.enhancer,
  })  : assert(logic != null),
        assert(store != null),
        assert(buildContext != null),
        assert(getState != null),
        assert(bus != null && enhancer != null),
        _buildContext = buildContext {
    ///
    _effectDispatch = logic.createEffectDispatch(this, enhancer);

    /// create Dispatch
    _dispatch = logic.createDispatch(
      _effectDispatch,
      logic.createNextDispatch(
        this,
        enhancer,
      ),
      this,
    );

    /// Register inter-component broadcast
    registerOnDisposed(bus.registerReceiver(_effectDispatch));
  }

  @override
  void bindForceUpdate(void Function() forceUpdate) {
    assert(_forceUpdate == null);
    _forceUpdate = forceUpdate;
  }

  @override
  BuildContext get context => _buildContext;

  @override
  T get state => getState();

  @override
  dynamic dispatch(Action action) => _dispatch(action);

  @override
  Widget buildComponent(String name) {
    assert(name != null, 'The name must be NotNull for buildComponent.');
    final Dependent<T> dependent = logic.slot(name);
    final Widget result = dependent?.buildComponent(store, getState,
        bus: bus, enhancer: enhancer);
    assert(result != null, 'Could not found component by name "$name."');
    return result ?? Container();
  }

  @override
  void onLifecycle(Action action) {
    assert(_throwIfDisposed());
    _dispatch(action);
  }

  @override
  void dispose() {
    super.dispose();
    _buildContext = null;
    _forceUpdate = null;
  }

  bool _throwIfDisposed() {
    if (isDisposed) {
      throw Exception(
          'Ctx has been disposed which could not been used any more.');
    }
    return true;
  }

  @override
  State<StatefulWidget> get stfState {
    assert(_buildContext is StatefulElement);
    if (_buildContext is StatefulElement) {
      final StatefulElement stfElement = _buildContext;
      return stfElement.state;
    }
    return null;
  }

  @override
  void broadcastEffect(Action action, {bool excluded}) =>
      bus.dispatch(action, excluded: excluded == true ? _effectDispatch : null);

  @override
  void broadcast(Action action) => bus.broadcast(action);

  @override
  void Function() addObservable(Subscribe observable) {
    final void Function() unsubscribe = observable(() {
      _forceUpdate?.call();
    });
    registerOnDisposed(unsubscribe);
    return unsubscribe;
  }

  @override
  void forceUpdate() => _forceUpdate?.call();

  @override
  void Function() listen({
    bool Function(T, T) isChanged,
    @required void Function() onChange,
  }) {
    assert(onChange != null);
    T oldState;
    final AutoDispose disposable = registerOnDisposed(
      store.subscribe(
        () => () {
              final T newState = state;
              final bool flag = isChanged == null
                  ? !identical(oldState, newState)
                  : isChanged(oldState, newState);
              oldState = newState;
              if (flag) {
                onChange();
              }
            },
      ),
    );

    return () => disposable?.dispose();
  }
}

class ComponentContext<T> extends LogicContext<T> implements ViewUpdater<T> {
  final ViewBuilder<T> view;
  final ShouldUpdate<T> shouldUpdate;
  final String name;
  final Function() markNeedsBuild;
  final ContextSys<Object> sidecarCtx;

  Widget _widgetCache;
  T _latestState;

  ComponentContext({
    @required AbstractComponent<T> logic,
    @required Store<Object> store,
    @required BuildContext buildContext,
    @required Get<T> getState,
    @required this.view,
    @required this.shouldUpdate,
    @required this.name,
    @required this.markNeedsBuild,
    @required this.sidecarCtx,
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  })  : assert(bus != null && enhancer != null),
        super(
          logic: logic,
          store: store,
          buildContext: buildContext,
          getState: getState,
          bus: bus,
          enhancer: enhancer,
        ) {
    _latestState = state;

    sidecarCtx?.setParent(this);
  }

  @override
  void onLifecycle(Action action) {
    super.onLifecycle(action);
    sidecarCtx?.onLifecycle(LifecycleCreator.reassemble());
  }

  @override
  ListAdapter buildAdapter() {
    assert(sidecarCtx != null);
    return logic.adapterDep()?.buildAdapter(sidecarCtx) ??
        const ListAdapter(null, 0);
  }

  @override
  Widget buildWidget() {
    Widget result = _widgetCache;
    if (result == null) {
      result = _widgetCache = view(state, dispatch, this);

      dispatch(LifecycleCreator.build(name));
    }
    return result;
  }

  @override
  void didUpdateWidget() {
    final T now = state;
    if (shouldUpdate(_latestState, now)) {
      _widgetCache = null;
      _latestState = now;
    }
  }

  @override
  void onNotify() {
    final T now = state;
    if (shouldUpdate(_latestState, now)) {
      _widgetCache = null;

      markNeedsBuild();

      _latestState = now;
    }
  }

  @override
  void clearCache() {
    _widgetCache = null;
  }

  @override
  void forceUpdate() {
    _widgetCache = null;

    try {
      markNeedsBuild();
    } catch (e) {
      /// TODO
      /// should try-catch in force mode which is called from outside
    }
  }
}
```



### logic

```dart
/// fish_redux/redux_component/logic

/// 四个部分
/// 1. Reducer 和 ReducerFilter
/// 2. Effect
/// 3. Dependencies
/// 4. Key

abstract class Logic<T> implements AbstractLogic<T> {
  final Reducer<T> _reducer;
  final ReducerFilter<T> _filter;
  final Effect<T> _effect;
  final Dependencies<T> _dependencies;
  final Object Function(T state) _key;

  /// 用于扩展
  Reducer<T> get protectedReducer => _reducer;
  ReducerFilter<T> get protectedFilter => _filter;
  Effect<T> get protectedEffect => _effect;
  Dependencies<T> get protectedDependencies => _dependencies;
  Object Function(T state) get protectedKey => _key;

  /// 用于提升运行效率的函数缓存
  final Map<String, Object> _resultCache = <String, Object>{};

  Logic({
    Reducer<T> reducer,
    Dependencies<T> dependencies,
    ReducerFilter<T> filter,
    Effect<T> effect,
    Object Function(T state) key,
  })  : _reducer = reducer,
        _filter = filter,
        _effect = effect,
        _dependencies = dependencies,
        _key = key;

  @override
  Type get propertyType => T;

  bool isSuperTypeof<K>() => Tuple0<K>() is Tuple0<T>;

  bool isTypeof<K>() => Tuple0<T>() is Tuple0<K>;

  /// if
  /// _resultCache['key'] = null;
  /// then
  /// _resultCache.containsKey('key') will be true;
  R cache<R>(String key, Get<R> getter) => _resultCache.containsKey(key)
      ? _resultCache[key]
      : (_resultCache[key] = getter());

  @override
  Reducer<T> get reducer => helper.filterReducer(
      combineReducers<T>(
          <Reducer<T>>[protectedReducer, protectedDependencies?.reducer]),
      protectedFilter);

  @override
  Object onReducer(Object state, Action action) =>
      cache<Reducer<T>>('onReducer', () => reducer)?.call(state, action) ??
      state;

  @override
  Dispatch createEffectDispatch(ContextSys<T> ctx, Enhancer<Object> enhancer) {
    return helper.createEffectDispatch<T>(

        /// enhance userEffect
        enhancer.effectEnhance(
          protectedEffect,
          this,
          ctx.store,
        ),
        ctx);
  }

  @override
  Dispatch createNextDispatch(ContextSys<T> ctx, Enhancer<Object> enhancer) =>
      helper.createNextDispatch<T>(ctx);

  @override
  Dispatch createDispatch(
    Dispatch effectDispatch,
    Dispatch nextDispatch,
    Context<T> ctx,
  ) =>
      helper.createDispatch<T>(effectDispatch, nextDispatch, ctx);

  @override
  Object key(T state) => _key?.call(state) ?? ValueKey<Type>(runtimeType);

  @override
  Dependent<T> slot(String type) => protectedDependencies?.slot(type);

  @override
  Dependent<T> adapterDep() => protectedDependencies?.adapter;
}
```



### component

```dart
/// fish_redux/redux_component/component

/// 用于包裹一个widget
/// e.g.
/// WidgetWrapper wrapper =
/// Widget buildRepaintBoundary(Widget child){
///		return RepaintBoundary(child: child);
/// }

typedef WidgetWrapper = Widget Function(Widget child);

@immutable
abstract class Component<T> extends Logic<T> implements AbstractComponent<T> {
  final ViewBuilder<T> _view;
  final ShouldUpdate<T> _shouldUpdate;
  final WidgetWrapper _wrapper;
  final bool _clearOnDependenciesChanged;

  ViewBuilder<T> get protectedView => _view;
  ShouldUpdate<T> get protectedShouldUpdate => _shouldUpdate;
  WidgetWrapper get protectedWrapper => _wrapper;
  bool get protectedClearOnDependenciesChanged => _clearOnDependenciesChanged;

  Component({
    @required ViewBuilder<T> view,
    Reducer<T> reducer,
    ReducerFilter<T> filter,
    Effect<T> effect,
    Dependencies<T> dependencies,
    ShouldUpdate<T> shouldUpdate,
    WidgetWrapper wrapper,
    Key Function(T) key,
    bool clearOnDependenciesChanged = false,
  })  : assert(view != null),
        _view = view,
        _wrapper = wrapper ?? _wrapperByDefault,
        _shouldUpdate = shouldUpdate ?? updateByDefault<T>(),
        _clearOnDependenciesChanged = clearOnDependenciesChanged,
        super(
          reducer: reducer,
          filter: filter,
          effect: effect,
          dependencies: dependencies,
          key: key,
        );

  @override
  Widget buildComponent(
    Store<Object> store,
    Get<Object> getter, {
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  }) {
    // assert(bus != null && enhancer != null);
    return protectedWrapper(
      ComponentWidget<T>(
        component: this,
        getter: _asGetter<T>(getter),
        store: store,
        key: key(getter()),

        ///
        bus: bus ?? DispatchBusDefault(),
        enhancer: enhancer ?? EnhancerDefault<Object>(),
      ),
    );
  }

  @override
  ComponentContext<T> createContext(
    Store<Object> store,
    BuildContext buildContext,
    Get<T> getState, {
    @required void Function() markNeedsBuild,
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  }) {
    assert(bus != null && enhancer != null);
    return ComponentContext<T>(
      logic: this,
      store: store,
      buildContext: buildContext,
      getState: getState,
      view: enhancer.viewEnhance(protectedView, this, store),
      shouldUpdate: protectedShouldUpdate,
      name: name,
      markNeedsBuild: markNeedsBuild,
      sidecarCtx: adapterDep()?.createContext(
        store,
        buildContext,
        getState,
        bus: bus,
        enhancer: enhancer,
      ),
      enhancer: enhancer,
      bus: bus,
    );
  }

  ComponentState<T> createState() => ComponentState<T>();

  String get name => cache<String>('name', () => runtimeType.toString());

  ///
  static ShouldUpdate<K> neverUpdate<K>() => (K _, K __) => false;

  static ShouldUpdate<K> alwaysUpdate<K>() => (K _, K __) => true;

  static ShouldUpdate<K> updateByDefault<K>() =>
      (K _, K __) => !identical(_, __);

  static Widget _wrapperByDefault(Widget child) => child;

  static Get<T> _asGetter<T>(Get<Object> getter) {
    Get<T> runtimeGetter;
    if (getter is Get<T>) {
      runtimeGetter = getter;
    } else {
      runtimeGetter = () {
        final T result = getter();
        return result;
      };
    }
    return runtimeGetter;
  }
}

class ComponentWidget<T> extends StatefulWidget {
  final Component<T> component;
  final Store<Object> store;
  final Get<T> getter;
  final DispatchBus bus;
  final Enhancer<Object> enhancer;

  const ComponentWidget({
    @required this.component,
    @required this.store,
    @required this.getter,
    this.bus,
    this.enhancer,
    Key key,
  })  : assert(component != null),
        assert(store != null),
        assert(getter != null),
        super(key: key);

  @override
  ComponentState<T> createState() => component.createState();
}

class ComponentState<T> extends State<ComponentWidget<T>> {
  ComponentContext<T> _ctx;

  ComponentContext<T> get ctx => _ctx;

  @mustCallSuper
  @override
  Widget build(BuildContext context) => _ctx.buildWidget();

  @override
  @protected
  @mustCallSuper
  void reassemble() {
    super.reassemble();
    _ctx.clearCache();
    _ctx.onLifecycle(LifecycleCreator.reassemble());
  }

  @mustCallSuper
  @override
  void initState() {
    super.initState();

    /// init context
    _ctx = widget.component.createContext(
      widget.store,
      context,
      () => widget.getter(),
      markNeedsBuild: () {
        if (mounted) {
          setState(() {});
        }
      },
      bus: widget.bus,
      enhancer: widget.enhancer,
    );

    /// register store.subscribe
    _ctx.registerOnDisposed(widget.store.subscribe(() => _ctx.onNotify()));

    _ctx.onLifecycle(LifecycleCreator.initState());
  }

  @mustCallSuper
  @override
  void didChangeDependencies() {
    super.didChangeDependencies();

    if (widget.component.protectedClearOnDependenciesChanged != false) {
      _ctx.clearCache();
    }

    _ctx.onLifecycle(LifecycleCreator.didChangeDependencies());
  }

  @mustCallSuper
  @override
  void deactivate() {
    super.deactivate();
    _ctx.onLifecycle(LifecycleCreator.deactivate());
  }

  @mustCallSuper
  @override
  void didUpdateWidget(ComponentWidget<T> oldWidget) {
    super.didUpdateWidget(oldWidget);
    _ctx.didUpdateWidget();
    _ctx.onLifecycle(LifecycleCreator.didUpdateWidget());
  }

  @mustCallSuper
  void disposeCtx() {
    if (!_ctx.isDisposed) {
      _ctx
        ..onLifecycle(LifecycleCreator.dispose())
        ..dispose();
    }
  }

  @mustCallSuper
  @override
  void dispose() {
    disposeCtx();
    super.dispose();

    /// TODO
    // mainCtx
    //   ..onLifecycle(LifecycleCreator.didDisposed())
    //   ..dispose();
  }
}

```



### page

```dart
/// fish_redux/redux_component/page

/// init store's state by route-params
typedef InitState<T, P> = T Function(P params);

typedef StoreUpdater<T> = Store<T> Function(Store<T> store);

@immutable
abstract class Page<T, P> extends Component<T> {
  /// AppBus is a event-bus used to communicate between pages.
  final DispatchBus appBus = DispatchBusDefault.shared;

  final InitState<T, P> _initState;

  final Enhancer<T> enhancer;

  /// connect with other stores
  final List<StoreUpdater<T>> _storeUpdaters = <StoreUpdater<T>>[];

  Page({
    @required InitState<T, P> initState,
    @required ViewBuilder<T> view,
    Reducer<T> reducer,
    ReducerFilter<T> filter,
    Effect<T> effect,
    Dependencies<T> dependencies,
    ShouldUpdate<T> shouldUpdate,
    WidgetWrapper wrapper,
    Key Function(T) key,
    List<Middleware<T>> middleware,
    List<ViewMiddleware<T>> viewMiddleware,
    List<EffectMiddleware<T>> effectMiddleware,
    List<AdapterMiddleware<T>> adapterMiddleware,
  })  : assert(initState != null),
        _initState = initState,
        enhancer = EnhancerDefault<T>(
          middleware: middleware,
          viewMiddleware: viewMiddleware,
          effectMiddleware: effectMiddleware,
          adapterMiddleware: adapterMiddleware,
        ),
        super(
          view: view,
          dependencies: dependencies,
          reducer: reducer,
          filter: filter,
          effect: effect,
          shouldUpdate: shouldUpdate,
          wrapper: wrapper,
          key: key,
        );

  Widget buildPage(P param) => protectedWrapper(_PageWidget<T, P>(
        page: this,
        param: param,
      ));

  Store<T> createStore(P param) => updateStore(createBatchStore<T>(
        _initState(param),
        reducer,
        storeEnhancer: enhancer.storeEnhance,
      ));

  Store<T> updateStore(Store<T> store) => _storeUpdaters.fold(
        store,
        (Store<T> previousValue, StoreUpdater<T> element) =>
            element(previousValue),
      );

  /// page-store connect with app-store
  void connectExtraStore<K>(
    Store<K> extraStore,

    /// To solve Reducer<Object> is neither a subtype nor a supertype of Reducer<T> issue.
    Object Function(Object, K) update,
  ) =>
      _storeUpdaters.add((Store<T> store) => connectStores<Object, K>(
            store,
            extraStore,
            update,
          ));

  DispatchBus createPageBus() => DispatchBusDefault();

  void unshift({
    List<Middleware<T>> middleware,
    List<ViewMiddleware<T>> viewMiddleware,
    List<EffectMiddleware<T>> effectMiddleware,
    List<AdapterMiddleware<T>> adapterMiddleware,
  }) {
    enhancer.unshift(
      middleware: middleware,
      viewMiddleware: viewMiddleware,
      effectMiddleware: effectMiddleware,
      adapterMiddleware: adapterMiddleware,
    );
  }

  void append({
    List<Middleware<T>> middleware,
    List<ViewMiddleware<T>> viewMiddleware,
    List<EffectMiddleware<T>> effectMiddleware,
    List<AdapterMiddleware<T>> adapterMiddleware,
  }) {
    enhancer.append(
      middleware: middleware,
      viewMiddleware: viewMiddleware,
      effectMiddleware: effectMiddleware,
      adapterMiddleware: adapterMiddleware,
    );
  }
}

class _PageWidget<T, P> extends StatefulWidget {
  final Page<T, P> page;
  final P param;

  const _PageWidget({
    Key key,
    @required this.page,
    @required this.param,
  }) : super(key: key);

  @override
  State<StatefulWidget> createState() => _PageState<T, P>();
}

class _PageState<T, P> extends State<_PageWidget<T, P>> {
  Store<T> _store;
  DispatchBus _pageBus;

  final Map<String, Object> extra = <String, Object>{};

  @override
  void initState() {
    super.initState();
    _store = widget.page.createStore(widget.param);
    _pageBus = widget.page.createPageBus();
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();

    /// Register inter-page broadcast
    _pageBus.attach(widget.page.appBus);
  }

  @override
  Widget build(BuildContext context) {
    // return PageProvider(
    //   store: _store,
    //   extra: extra,
    //   child:
    // );

    return widget.page.buildComponent(
      _store,
      _store.getState,
      bus: _pageBus,
      enhancer: widget.page.enhancer,
    );
  }

  @override
  void dispose() {
    _pageBus.detach();
    _store.teardown();
    super.dispose();
  }
}

@deprecated
class PageProvider extends InheritedWidget {
  final Store<Object> store;

  /// Used to store page data if needed
  final Map<String, Object> extra;

  const PageProvider({
    @required this.store,
    @required this.extra,
    @required Widget child,
    Key key,
  })  : assert(store != null),
        assert(child != null),
        super(key: key, child: child);

  static PageProvider tryOf(BuildContext context) {
    final PageProvider provider =
        context.inheritFromWidgetOfExactType(PageProvider);
    return provider;
  }

  @override
  bool updateShouldNotify(PageProvider oldWidget) =>
      store != oldWidget.store && extra != oldWidget.extra;
}
```



## Utils

### collections

```dart
/// /fish_redux/utils/collections

class Collections {
  /// 提前检查list是否 是空 或者 是null，再调用 reduce
  static E reduce<E>(List<E> list, E combine(E e0, E e1)) =>
      (list == null || list.isEmpty) ? null : list.reduce(combine);

  /// 提前检查list是否 是空 或者 是null，再调用 fold
  static T fold<T, E>(T init, List<E> list, T combine(T e0, E e1)) =>
      (list == null || list.isEmpty) ? init : list.fold(init, combine);

  /// 压缩一个 list
  /// 例如:
  /// List<String> a = ['a', 'b', 'c'];
  /// List<String> b = ['1', '2', '3'];
  /// List<List<String>> list = [a, b] // [[a, b, c], [1, 2, 3]]
  /// List<String> listFlatten = Collections.flatten(list) // [a, b, c, 1, 2, 3]
    
  static List<E> flatten<E>(List<List<E>> lists) => reduce(lists, merge);

  /// 合并两个 Iterable，会提前进行 null 判断  
  /// List<String> a = ['a', 'b', 'c'];
  /// List<String> b = ['1', '2', '3'];
  /// List<String> listMerge = Collections.merge(a, b) // [a, b, c, 1, 2, 3]
  static List<T> merge<T>(Iterable<T> a, Iterable<T> b) =>
      <T>[]..addAll(a ?? <T>[])..addAll(b ?? <T>[]);

  /// 列表复制，会提前进行 null 判断
  static List<T> clone<T>(Iterable<T> a) =>
      (a == null || a.isEmpty) ? <T>[] : (<T>[]..addAll(a));

  /// 将 map 转换为 list
  /// 要求提供一个mapper用于从MapEntries中挑选Key或者Value  
  /// Map<String, String> map = {'key0': 'a', 'key1': 'b', 'key2': 'c'};
  /// Function mapFunction = (String value, String key) => value;
  /// List<String> list = Collections
  ///     .castMapToList<String, String, String>(map, mapFunction); // [a, b, c]
    
  static List<T> castMapToList<T, K, V>(Map<K, V> map0, T map(V v, K k)) =>
      map0.entries
          .map((MapEntry<K, V> entry) => map(entry.value, entry.key))
          .toList();

  /// Cast map with a map function
  static Map<K, V1> castMap<K, V0, V1>(Map<K, V0> map0, V1 map(V0 v0, K k)){
      <K, V1>{}..addEntries(
      	castMapToList<MapEntry<K, V1>, K, V0>(
          map0, 
          (V0 v, K k) => MapEntry<K, V1>(k, map(v, k))    					}
      	)
 	  );
  }
      
  /// 删除 list 中的 null 元素
  /// List<String> list = ['1', '2', null, '3', null];
  /// print(list)                       // [1, 2, null, 3, null]
  /// print(Collections.compact(list)); // [1, 2, 3]
  static List<T> compact<T>(Iterable<T> list, {bool growable = true}) =>
      list?.where((T e) => e != null)?.toList(growable: growable);

  /// 检查一个对象是否是空
  static bool isEmpty(Object value) {
    if (value == null) {
      return true;
    } else {
      if (value is String) {
        return value.isEmpty;
      } else if (value is List) {
        return value.isEmpty;
      } else if (value is Map) {
        return value.isEmpty;
      } else if (value is Set) {
        return value.isEmpty;
      } else {
        return false;
      }
    }
  }

  static bool isNotEmpty(Object value) => !isEmpty(value);
}
```



### debug

```dart
/// fish_redux/utils/debug

bool _debugFlag = false;

bool isDebug() {
  /// assert 仅在 debug 模式下有效
  assert(() {
    _debugFlag = true;
    return _debugFlag;
  }());
  return _debugFlag;
}

/// 将 print 包裹在一个返回true的函数里，便于assert调用
bool println(Object object) {
  print(object);
  return true;
}
```


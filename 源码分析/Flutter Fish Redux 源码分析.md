
[TOC]

# 前言

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

<div style = "color: #DE376F">
    以下内容摘自 FishRedux 的 Doc
</div>


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



# Flutter Fish Redux 源码分析

以下部分是源码分析






## Redux

### /basic(  JsRedux基础  )

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
/// 如果 Reducer 需要相应一个 Action, 它返回一个新 state,否则反回旧 state.
typedef Reducer<T> = T Function(T state, Action action);

/// Dispatch 的标准定义
/// Send an "intention"，用于派发一个Action
typedef Dispatch = dynamic Function(Action action);

/// subscription function 的标准定义
/// 输入一个 subscriber ，产生 anti-subscription function
typedef Subscribe = void Function() Function(void Function() callback);

/// ReplaceReducer 的定义
typedef ReplaceReducer<T> = void Function(Reducer<T> reducer);

/// Observable flow 的定义
typedef Observable<T> = Stream<T> Function();

/// Synthesizable functions（可合成函数）的定义
/// 输入一个 T next，返回一个 T 实例
/// 可以被 list.fold 使用，最终产生一个 T
typedef Composable<T> = T Function(T next);

/// 类获取器的定义（获取 R 实例）
/// 通过一个函数来获取 R
typedef Get<R> = R Function();

/// Middleware 的标准定义
/// 输入 Dispatch 和 getState，返回一个 Composable<Dispatch> 函数
/// 产生的 Composable 可以接受一个 Dispatch 并返回一个 Dispatch
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

/// Store 创建器
typedef StoreCreator<T> = Store<T> Function(
  T preloadedState,
  Reducer<T> reducer,
);

/// 修改一个 Store 创建器 
typedef StoreEnhancer<T> = StoreCreator<T> Function(StoreCreator<T> creator);

/// SubReducer 的定义
/// [isStateCopied] 与性能有关，确保 T 在整个过程中只会被复制一次
typedef SubReducer<T> = T Function(T state, Action action, bool isStateCopied);

/// 连接 Reducer<S> 和 Reducer<P> 的 Connector 的定义
/// 1. 如何从 S 实例中获取 P 的实例
/// 2. 如何将 P 实例的更改同步到 S 实例
/// 3. 如何复制一个新的S
abstract class AbstractConnector<S, P> {
  /// 从 State 获取到 P 实例（将大的 State 转化为 小的 State）  
  P get(S state);

  /// 产生一个 SubReducer  
  /// 对于可变的 state, 需要满足以下三个能力
  ///     1. get: (S) => P
  ///     2. set: (S, P) => void
  ///     3. shallow copy: s.clone() // 注意这里是要实现浅拷贝
  ///
  /// 对于不可变的 state,需要满足以下两个能力
  ///     1. get: (S) => P
  ///     2. set: (S, P) => S
  ///   
  SubReducer<S> subReducer(Reducer<P> reducer);
}
```



### /applyMiddleware
```dart
/// 组装所有 middleware，生成一个 StoreEnhancer
/// typedef Middleware<T> = Composable<Dispatch> Function({
///   Dispatch dispatch,
///   Get<T> getState,
/// });
StoreEnhancer<T> applyMiddleware<T>(List<Middleware<T>> middleware) {
  return middleware == null || middleware.isEmpty
      ? null
      : (StoreCreator<T> creator) => (T initState, Reducer<T> reducer) {
          
          final Store<T> store = creator(initState, reducer);
          final Dispatch initialValue = store.dispatch;
          store.dispatch = middleware
              /// map 返回的是 Composable<Dispatch>
              .map((Middleware<T> middleware) => middleware(
                  dispatch: (Action action) => store.dispatch(action),
                  getState: store.getState,
              ))
              /// 折叠所有元素，最终返回一个 Dispatch
              .fold(
                  initialValue,
              	  /// (Dispatch previousValue,Composable<Dispatch> element)
                  (Dispatch previousValue,Dispatch Function(Dispatch) element) =>
                  element(previousValue),
          	  );

          return store;
      }; 
}
```



### /combineReducers

以下两个函数的实现完全一致

#### combineSubReducers

```dart
/// 同 flutter_redux
/// 将多个 subReducer 绑定成为一个大的 subReducer
Reducer<T> combineSubReducers<T>(Iterable<SubReducer<T>> subReducers) {
  final List<SubReducer<T>> notNullReducers = subReducers?.where(
      		(SubReducer<T> e) => e != null)
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
    return copy;
  };
}
```
#### combineReducers
```dart
/// 同 flutter_redux
/// 将多个 reducer 绑定成为一个大的 reducer
Reducer<T> combineReducers<T>(Iterable<Reducer<T>> reducers) {
  /// 取出所有的非 null reducer  
  final List<Reducer<T>> notNullReducers = reducers?.where(
      		(Reducer<T> r) => r != null
  		)?.toList(growable: false);
    
  if (notNullReducers == null || notNullReducers.isEmpty) {
    return null;
  }

  /// 只返回一个，提高效率  
  if (notNullReducers.length == 1) {
    return notNullReducers.single;
  }
    
  /// 返回多个 reducer 的绑定
  /// 将 action 遍历给所有 reducer 处理一边得到最终的 state  
  return (T state, Action action) {
    T nextState = state;
    for (Reducer<T> reducer in notNullReducers) {
      nextState = reducer(nextState, action);
    }
    return nextState;
  };
}
```

#### castReducer
```dart
/// 将一个超 Reducer<Sup> 转换为子 Reducer<Sub>
Reducer<Sub> castReducer<Sub extends Sup, Sup>(Reducer<Sup> sup) {
  return sup == null
      ? null
      : (Sub state, Action action) {
          final Sub result = sup(state, action);
          return result;
        };
}
```



### /connector

connector：将大的 state 转化为小的 state

在“..redux/basic”中有做如下要求（其中T为超State，P为子State）:

For mutable state, there are three abilities needed to be met.
    1. get: (S) => P
    2. set: (S, P) => void
    3. shallow copy: s.clone()

For immutable state, there are two abilities needed to be met.
       1. get: (S) => P
       2. set: (S, P) => S


```dart
/// 深度复制实现
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
```

#### Cloneable

```dart
/// Definition of Cloneable
abstract class Cloneable<T extends Cloneable<T>> {
  T clone();
}
```

#### ImmutableConn

```dart
/// 为不可变的 state 创建一个基础 connnector
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
        return result;
      }
      return state;
    };
  }
}
```

#### MutableConn
```dart
/// 为可变的 state 创建一个基础 connnector
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
      /// 对于State只copy一次以提升性能  
      final T copy = (hasChanged && !isStateCopied) ? _clone<T>(state) : state;
      if (hasChanged) {
        set(copy, newProps);
      }
      return copy;
    };
  }
}
```

### /creatStore

```dart
/// 直接返回 state 的 ruducer
Reducer<T> _noop<T>() => (T state, Action action) => state;

typedef _VoidCallback = void Function();

/// 不满足条件，抛出异常
void _throwIfNot(bool condition, [String message]) {
  if (!condition) {
    throw ArgumentError(message);
  }
}

/// 构建 Store
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
    /// Store.dispatch 的具体实现  
    /// 当收到 Actions 的时候，调用如下函数  
    ..dispatch = (Action action) {
      _throwIfNot(action != null, 'Expected the action to be non-null value.');
      _throwIfNot(action.type != null,
          'Expected the action.type to be non-null value.');
      /// 如果正处于分发阶段，抛出异常
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
      
      /// 通知所有监听者
      for (_VoidCallback listener in _notifyListeners) {
        listener();
      }

      _notifyController.add(_state);
    }
    /// 替换现有的 reducer
    ..replaceReducer = (Reducer<T> replaceReducer) {
      _reducer = replaceReducer ?? _noop;
    }
    /// 为该 store 添加一个 listener
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

      /// 返回一个可以移除这个 listener 的函数，交给调用者自己处理  
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
      /// 这里似乎有内存泄漏
      /// 添加 _listeners = null  
      return _notifyController.close();
    };
}

/// 使用 StoreEnhancer 创建一个 store
Store<T> createStore<T>(
    T preloadedState, 
    Reducer<T> reducer,
    [StoreEnhancer<T> enhancer]
) {
   return enhancer != null
      /// 修改 _createStore，得到新的 createStore 并调用之
      ? enhancer(_createStore)(preloadedState, reducer)
      : _createStore(preloadedState, reducer);  
}
    
/// 压缩多个 StoreEnhancer 形成一个
StoreEnhancer<T> composeStoreEnhancer<T>(List<StoreEnhancer<T>> enhancers){
    enhancers == null || enhancers.isEmpty
        ? null
        : enhancers.reduce((StoreEnhancer<T> previous, StoreEnhancer<T> next) =>
            (StoreCreator<T> creator) => next(previous(creator)));
}
   
```



## Redux_AOP

fish_redux 的面向切面部分

这个部分实现了对已有函数的修改（ApplyLikeEnhancer）

### /common_aop

#### aop

```dart
/// Function.apply 的实现如下
/// external static apply(
///    Function function, 
///    List positionalArguments,
///    [Map<Symbol, dynamic> namedArguments]
/// );
/// 即可以通过提供的 positionalArguments 和 namedArguments 来调用给定函数 function

/// 函数调用器
/// TypedApplyLike，通过指定的参数实现调用不同函数，这个函数返回指定类型结果
typedef TypedApplyLike<R> = R Function(List<dynamic>, [Map<Symbol, dynamic>]);

/// ApplyLike，通过指定的参数实现调用不同函数，这个函数返回指定结果
typedef ApplyLike = dynamic Function(List<dynamic>, [Map<Symbol, dynamic>]);

/// 函数AOP的统一抽象, 输入一个函数产出另一个增强版的函数
typedef ApplyLikeEnhancer = ApplyLike Function(ApplyLike functor);

/// 等价返回调用其
ApplyLike _identity(ApplyLike f) => f;

/// 组合两个 ApplyLikeEnhancer
/// 被 enhances.reduce 使用，用以联合所有的enhancer
ApplyLikeEnhancer _combine(ApplyLikeEnhancer e0, ApplyLikeEnhancer e1) =>
    (ApplyLike f) => (e1 ?? _identity)((e0 ?? _identity)(f));

/// 等价于：
///  return (ApplyLike f){
///    	if(e1 != null){
///            if(e0 != null){
///                return e1(e0(f));
///            }else{
///                return e1(_identity(f));  		// => e1(f)
///            }
///        }else{
///            if(e0 != null){
///                return _identity(e0(f));  		// => e0(f)
///            }else{
///                return _identity(_identity(f));  // => f
///            }
///        }
///	}

const ApplyLikeEnhancer ApplyLikeEnhancerIdentity = _identity;

/// 用局部套用技术实现 AOP。
/// (AOP): https://en.wikipedia.org/wiki/Aspect-oriented_programming
/// (局部套用):https://en.wikipedia.org/wiki/Currying
/// 过程
/// 1.输入用户 [Function]
/// 2.转换为 ApplyLike
/// 3.添加一些修改（通过[ApplyLikeEnhancer])
/// 4.得到新的(ApplyLike)
/// 5.转换为(TypedApplyLike)
/// 6.转换为用户 [Function]

/// 封装面向切面的方法类
class AOP {
  final ApplyLikeEnhancer _enhancer;
 
  AOP(List<ApplyLikeEnhancer> enhances)
      : _enhancer = enhances?.isNotEmpty == true
            /// 聚合的所有 ApplyLikeEnhancer ，最终返回一个 ApplyLikeEnhancer 聚合体
            /// _enhancer(f) => en(...e2(e1(e0(f))))
            ? enhances.reduce(_combine)
            : ApplyLikeEnhancerIdentity;

    
  TypedApplyLike<R> enhance<R>(Function functor) {
    /// 将 functor 转换为 ApplyLike
    final ApplyLike init = (
        	List<dynamic> positionalArguments,
            [Map<Symbol, dynamic> namedArguments]
    ) => Function.apply(functor, positionalArguments, namedArguments);

    /// 修改 ApplyLike
    final ApplyLike enhanced = _enhancer(init);

    /// 如果没有被修改
    if (init == enhanced) {
      return null;
    }

    /// 将 ApplyLike 转换为 TypedApplyLike<R>
    return (List<dynamic> positionalArguments,[Map<Symbol, dynamic> namedArguments]) {
      final R result = enhanced(positionalArguments);
      return result;
    };
  }

  /// 适用于无参数的 functor 
  /// 后面 1~6 依次类推  
  R Function() withZero<R>(R Function() f) {
    final TypedApplyLike<R> enhanced = enhance<R>(f);
    return enhanced != null ? () => enhanced(<dynamic>[]) : f;
  }

  R Function(P) withOne<R, P>(R Function(P) f) {
    final TypedApplyLike<R> enhanced = enhance<R>(f);
    return enhanced != null ? (P p) => enhanced(<dynamic>[p]) : f;
  }

  R Function(P0, P1) withTwo<R, P0, P1>(R Function(P0, P1) f) {
    final R Function(List<dynamic>) enhanced = enhance<R>(f);
    return enhanced != null ? (P0 p0, P1 p1) => enhanced(<dynamic>[p0, p1]) : f;
  }

  R Function(P0, P1, P2) withThree<R, P0, P1, P2>(R Function(P0, P1, P2) f) {
    final TypedApplyLike<R> enhanced = enhance<R>(f);
    return enhanced != null
        ? (P0 p0, P1 p1, P2 p2) => enhanced(<dynamic>[p0, p1, p2])
        : f;
  }

  R Function(P0, P1, P2, P3) withFour<R, P0, P1, P2, P3>(
      R Function(P0, P1, P2, P3) f) {
    final TypedApplyLike<R> enhanced = enhance<R>(f);
    return enhanced != null
        ? (P0 p0, P1 p1, P2 p2, P3 p3) => enhanced(<dynamic>[p0, p1, p2, p3])
        : f;
  }

  R Function(P0, P1, P2, P3, P4) withFive<R, P0, P1, P2, P3, P4>(
      R Function(P0, P1, P2, P3, P4) f) {
    final TypedApplyLike<R> enhanced = enhance<R>(f);
    return enhanced != null
        ? (P0 p0, P1 p1, P2 p2, P3 p3, P4 p4) =>
            enhanced(<dynamic>[p0, p1, p2, p3, p4])
        : f;
  }

  R Function(P0, P1, P2, P3, P4, P5) withSix<R, P0, P1, P2, P3, P4, P5>(
      R Function(P0, P1, P2, P3, P4, P5) f) {
    final TypedApplyLike<R> enhanced = enhance<R>(f);
    return enhanced != null
        ? (P0 p0, P1 p1, P2 p2, P3 p3, P4 p4, P5 p5) =>
            enhanced(<dynamic>[p0, p1, p2, p3, p4, p5])
        : f;
  }
}
```

#### debounce

```dart
/// 将一个 functor 修改为：
/// 延迟 millis 后调用 functor
/// 如果延迟期间 functor 再次被调用则会取消掉先前的调用
ApplyLikeEnhancer debounce(int millis) {
  return (dynamic Function(List<dynamic>) functor) {
    int idGenerator = 0;
    return (List<dynamic> positionalArguments,
        [Map<Symbol, dynamic> namedArguments]) async {
      final int newId = ++idGenerator;
      await Future<void>.delayed(Duration(milliseconds: millis));
      if (newId == idGenerator) {
        return functor(positionalArguments);
      }
    };
  };
}

```

#### debug

```dart
bool _debugFlag = false;

/// app 是否处于 debug mode.
bool isDebug() {
  /// 因为断言只在生产阶段启用，故利用断言来判断是否处于生产环境，把 flag 置为 true
  assert(() {
    _debugFlag = true;
    return _debugFlag;
  }());
  return _debugFlag;
}

```

#### delay

```dart
/// 将一个 functor 修改为 ：
/// functor 会在延迟指定的时间后执行 
ApplyLikeEnhancer delay(int millis) {
 return (dynamic Function(List<dynamic>) functor) {
   return (List<dynamic> positionalArguments,[Map<Symbol, dynamic> namedArguments])async{
      await Future<void>.delayed(Duration(milliseconds: millis));
      return functor(positionalArguments);
   };
 };
}
```

#### log

```dart
/// 用于打印 functor 日志
ApplyLikeEnhancer logAOP(String tag) {
  /// 如果是 debug 模式下才会打印 functor 日志
  /// 默认打印 函数的输入和输出  
  return isDebug()
    ? (dynamic Function(List<dynamic>) functor) {
       return (List<dynamic> positionalArguments,[Map<Symbol, dynamic> namedArguments]) {
         print('$tag input: $positionalArguments');
         final Object result = functor(positionalArguments);
         if (result is Future) {
           result.then((Object r) {
                print('$tag output <Future>: $r');
                return r;
           });
         } else {
           print('$tag output: $result');
       	 }
         return result;
       };
     }
    : ApplyLikeEnhancerIdentity;
}
```

#### menmoize

```dart
/// 用于判断两个 list 是否相等
bool _listEquals<E>(List<E> list1, List<E> list2) {
  if (identical(list1, list2)) {
    return true;
  }
  if (list1 == null || list2 == null) {
    return false;
  }
  final int length = list1.length;
  if (length != list2.length) {
    return false;
  }
  for (int i = 0; i < length; i++) {
    if (list1[i] != list2[i]) {
      return false;
    }
  }
  return true;
}

/// 当调用的参数不发生改变时，记忆上次调用的返回值缓存
ApplyLikeEnhancer memoize() {
  return (dynamic Function(List<dynamic>) functor) {
    List<dynamic> memoizeArguments;
    dynamic memoizeResult;
    bool hasBeenCalled = false;

    return (List<dynamic> positionalArguments, [Map<Symbol, dynamic> namedArguments]) {
      /// 这里用到了 || 运算符的阻断性，当 hasBeenCalled 为 true 时，不计算 !_listEquals 
      if (!hasBeenCalled ||
          !_listEquals<dynamic>(positionalArguments, memoizeArguments)) {
        memoizeResult = functor(positionalArguments);
        memoizeArguments = positionalArguments;
        hasBeenCalled = true;
      }
	  /// 直接返回缓存值，免去调用	
      return memoizeResult;
    };
  };
}
```

#### performance

```dart
/// 返回当前时间距离（970-01-01T00:00:00）的微秒值
int _microSecsSinceEpoch() => DateTime.now().microsecondsSinceEpoch;

/// 计算一个函数消耗的时间
ApplyLikeEnhancer performanceAOP(String tag) {
  return isDebug()
      ? (dynamic Function(List<dynamic>) functor) {
          return (List<dynamic> positionalArguments,
              [Map<Symbol, dynamic> namedArguments]) {
            final int marked = DateTime.now().microsecondsSinceEpoch;
            final Object result = functor(positionalArguments);
            if (result is Future) {
              result.then((Object r) {
                print(
                    '$tag performance <Future>: ${_microSecsSinceEpoch() - marked}');
                return r;
              });
            } else {
              print('$tag performance: ${_microSecsSinceEpoch() - marked}');
            }
            return result;
          };
        }
      : ApplyLikeEnhancerIdentity;
}
```

#### throttle

```dart
int _microSecsSinceEpoch() => DateTime.now().microsecondsSinceEpoch;

/// 如果两次调用 functor 的时间小于 millis，则阻止其调用
ApplyLikeEnhancer throttle(int millis) {
  return (dynamic Function(List<dynamic>) functor) {
    int last = 0;
    return (List<dynamic> positionalArguments,[Map<Symbol, dynamic> namedArguments]) {
      final int now = _microSecsSinceEpoch();
      final int elapsed = now - last;
      if (elapsed >= millis) {
        last = now;
        return functor(positionalArguments);
      }
    };
  };
}
```

#### wait_until

```dart
/// 将一个 functor 修改为：
/// 当这个 functor（返回值为future） 被调用的时候，若再次调用这个 functor 会被取消，
/// 直到上一个 functor完成
ApplyLikeEnhancer waitUntil() {
  return (dynamic Function(List<dynamic>) functor) {
    bool isLocked = false;
    return (List<dynamic> positionalArguments,
        [Map<Symbol, dynamic> namedArguments]) {
      if (isLocked) {
        return null;
      } else {
        final Object result = functor(positionalArguments);
        if (result is Future) {
          isLocked = true;
          return result.whenComplete(() {
            isLocked = false;
          });
        }
        return result;
      }
    };
  };
}
```



## Redux_Component

### /autoDispose

#### _Fields

```dart
class _Fields {
  bool isDisposed = false;
  Set<AutoDispose> children;
  AutoDispose parent;
  void Function() onDisposed;
}
```
#### AutoDispose
```dart
/// 轻量级的生命周期管理系统
/// 当一个对象的dispose被调用
///  1. Dispose 所有 children
///  2. 切断与父级之间的联系
///  3. onDisposed 函数被调用
///  4. 标记 isDisposed = true
class AutoDispose {
  /// 持有一个控制生命周期的 Field  
  final _Fields _fields = _Fields();

  /// 对所有的孩子依次调用visitor   
  void visit(void Function(AutoDispose) visitor) =>  _fields.children?.forEach(visitor);

  /// 是否处于disposed状态  
  bool get isDisposed => _fields.isDisposed;

  /// dispose 方法，用于dispose这个Object和其所有的孩子，切断其与父级的联系  
  void dispose() {
    /// dispose 所有 children
    if (_fields.children != null) {
      /// 从集合到List  
      final List<AutoDispose> copy = _fields.children.toList(growable: false);
      /// 依次dispose  
      for (AutoDispose child in copy) {
        child.dispose();
      }
      /// 置null，释放内存  
      _fields.children = null;
    }

    /// 切断父级联系
    _fields.parent?._fields?.children?.remove(this);
    _fields.parent = null;

    /// 调用dispose回调函数，并释放其占用内存
    _fields.onDisposed?.call();
    _fields.onDisposed = null;

    /// 标记 isDisposed = true.
    _fields.isDisposed = true;
  }

  /// 设置 dispose 时的回调  
  void onDisposed(void Function() onDisposed) {
    /// 只能设置一次  
    assert(_fields.onDisposed == null);
    if (_fields.isDisposed) {
      /// 若已经处于dispose状态，立刻调用  
      onDisposed?.call();
    } else {
      _fields.onDisposed = onDisposed;
    }
  }

  /// 设置父级  
  void setParent(AutoDispose newParent) {
    final AutoDispose oldParent = _fields.parent;
    if (oldParent == newParent || isDisposed) {
      return;
    }

    if (newParent != null && newParent.isDisposed) {
      dispose();
      return;
    }

    if (newParent != null) {
      newParent._fields.children ??= <AutoDispose>{};
      newParent._fields.children.add(this);
    }
    if (oldParent != null) {
      oldParent._fields.children.remove(this);
    }
    _fields.parent = newParent;
  }

  AutoDispose registerOnDisposed(void Function() o
                                 nDisposed) 
  	=> AutoDispose()..setParent(this)..onDisposed(onDisposed);
}
```



### /basic

```dart
/// fish_redux/redux_component/basic

/// Component 的视图部分
/// 1.State 决定了如何渲染界面
/// 2.Dispatch 用于发送 Action
/// 3.ViewService 用于构建子 Component 或 Adapter
typedef ViewBuilder<T> = Widget Function(
  T state,
  Dispatch dispatch,
  ViewService viewService,
);
```
####  ListAdapter 
```dart
/// 被 ListView.builder 使用的 ListAdapter 基类
/// 可以将许多小 listAdapters 合并成一个大的
class ListAdapter {
  final int itemCount;
  final IndexedWidgetBuilder itemBuilder;
  const ListAdapter(this.itemBuilder, this.itemCount);
}

/// 用于提高滚动列表的性能
/// Adapter 的视图部分
/// 1.State 决定了如何渲染界面
/// 2.Dispatch 用于发送 Action
/// 3.ViewService 用于构建子 Component 或 Adapter
typedef AdapterBuilder<T> = ListAdapter Function(
  T state,
  Dispatch dispatch,
  ViewService viewService,
);
```
#### ViewUpdater
```dart
/// 数据驱动 ui
/// 1. 如何渲染
/// 2. 何时渲染
abstract class ViewUpdater<T> {
  Widget buildWidget();
  void didUpdateWidget();
  void onNotify();
  void forceUpdate();
  void clearCache();
}

/// 与 Dispatch 有些许不同(时候被拦截).
/// 同步函数的返回值为 bool, 在返回 true 时被拦截
/// 异步函数的返回值是 Future<void>, 总是被拦截
/// typedef OnAction = Dispatch;

/// 判断 store 被改变时 componenet 是否应该被更新
typedef ShouldUpdate<T> = bool Function(T old, T now);

/// 拦截器
typedef Effect<T> = dynamic Function(Action action, Context<T> ctx);

/// 面向切面的 view
/// 用法：
/// ViewMiddleware<T> safetyView<T>({
///     这里要求传入一个返回Widget的error函数，用于错误处理
///		Widget Function(
///			dynamic, StackTrace,
/// 		{AbstractComponent<dynamic> component, Store<T> store}
///		)onError}
///	) {
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

/// 视图中间件
typedef ViewMiddleware<T> = Composable<ViewBuilder<dynamic>> Function(
  AbstractComponent<dynamic>,
  Store<T>,
);

/// 面向切面的 adapter
typedef AdapterMiddleware<T> = Composable<AdapterBuilder<dynamic>> Function(
  AbstractAdapter<dynamic>,
  Store<T>,
);

/// 面向切面的 effect中间件
/// 使用：
/// EffectMiddleware<T> pageAnalyticsMiddleware<T>() {
///   return (AbstractLogic<dynamic> logic, Store<T> store) {
///     return (Effect<dynamic> effect) {
///       return effect == null 
///       ? null 
///		  : (Action action, Context<dynamic> ctx) {
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
```
#### Enhancer
```dart
/// 面向切面的页面级 store, view, adapter, effect
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

  /// 函数  
  StoreCreator<T> storeEnhance(StoreCreator<T> creator);

  /// 前添加  
  void unshift({
    List<Middleware<T>> middleware,
    List<ViewMiddleware<T>> viewMiddleware,
    List<EffectMiddleware<T>> effectMiddleware,
    List<AdapterMiddleware<T>> adapterMiddleware,
  });

  /// 后添加  
  void append({
    List<Middleware<T>> middleware,
    List<ViewMiddleware<T>> viewMiddleware,
    List<EffectMiddleware<T>> effectMiddleware,
    List<AdapterMiddleware<T>> adapterMiddleware,
  });
}

/// AOP End
```
#### ExtraData
```dart
abstract class ExtraData {
  /// 在 context 设置或获取数据
  Map<String, Object> get extra;
}
```
#### ViewService
```dart
abstract class ViewService implements ExtraData {
  /// 组装在 Dependencies.list 配置的adapter
  ListAdapter buildAdapter();

  /// 组装在 Dependencies.slots 配置的 component
  Widget buildComponent(String name);

  /// 从宿主 widget 中获取 Buildcontext
  BuildContext get context;

  /// Broadcast action(the intent) in app (inter-pages)
  void broadcast(Action action);

  /// Broadcast in all component receivers;
  void broadcastEffect(Action action, {bool excluded});
}
```

#### Context
```dart
/// 这个 context 实现了自动释放内存，并可以附加额外数据，并可以派发、广播 Action 和 Effect
abstract class Context<T> 
    extends AutoDispose 
    implements ExtraData 
{
  /// 最近的一个state
  T get state;

  /// 发送 action 的方法, actions 会被自身或者广播模块或者store接受并使用
  dynamic dispatch(Action action);

  /// 从宿主 widget 中获取 Buildcontext
  BuildContext get context;

  /// In general, we should not need this field.
  /// When we have to use this field, it means that we have encountered difficulties.
  /// 这与 逻辑视图分离思想 和 flutter widget 树相矛盾
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

  /// 组装在 Dependencies.slots 配置的 component
  Widget buildComponent(String name);

  /// 广播 action in app (inter-stores)
  void broadcast(Action action);

  /// Broadcast in all component receivers;
  void broadcastEffect(Action action, {bool excluded});

  /// add observable
  void Function() addObservable(Subscribe observable);

  void forceUpdate();

  /// 监听 <T> 实例的部分变化
  void Function() listen({
    bool Function(T, T) isChanged,
    void Function() onChange,
  });
}
```
#### ContextSys

```dart
/// Seen in framework-component
abstract class ContextSys<T> 
    extends Context<T> 
    implements ViewService 
{
  /// 相应生命周期调用
  void onLifecycle(Action action);

  /// 绑定强制刷新  
  void bindForceUpdate(void Function() forceUpdate);

  Store<dynamic> get store;

  Enhancer<dynamic> get enhancer;

  DispatchBus get bus;
}
```
#### Dependent
```dart
/// 呈现每个 Dependency
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

  /// 构建 ContextSys 
  ContextSys<Object> createContext(
    Store<Object> store,
    BuildContext buildContext,
    Get<T> getState, {
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  });

  /// 是否是 component  
  bool isComponent();

  /// 是否是 adapter  
  bool isAdapter();
}
```
#### AbstractLogic
```dart
/// 组件逻辑部分的封装
/// 逻辑分为两个部分，Reducer 和 Effect。
abstract class AbstractLogic<T> {
  /// 创建一个 educer<T>
  Reducer<T> get reducer;

  /// To solve Reducer<Object> is neither a subtype nor a supertype of Reducer<T> issue.
  Object onReducer(Object state, Action action);

  /// 创建这个实例的副作用处理器
  Dispatch createEffectDispatch(ContextSys<T> ctx, Enhancer<Object> enhancer);

  
  Dispatch createNextDispatch(ContextSys<T> ctx, Enhancer<Object> enhancer);

  /// 创建这个实例的 Dispatch  
  /// Dispatch 是被框架提供给用户的最重要的 api 之一
  Dispatch createDispatch(
    Dispatch effectDispatch,
    Dispatch nextDispatch,
    ContextSys<T> ctx,
  );

  /// 创建每个实例的上下文
  /// 这个文件导入了 'package:flutter/widgets.dart'，包含了 BuildContext  
  ContextSys<T> createContext(
    Store<Object> store,
    BuildContext buildContext,
    Get<T> getState, {
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  });

  /// To create each instance's key (for recycle) if needed
  Object key(T state);

  /// 通过 name 找到一个依赖者
  Dependent<T> slot(String name);

  /// Get a adapter-dependent
  Dependent<T> adapterDep();

  Type get propertyType;
}
```
#### AbstractComponent
```dart
/// component 的抽象
abstract class AbstractComponent<T> implements AbstractLogic<T> {
  /// 只添加了一个构造函数
  Widget buildComponent(
    Store<Object> store,
    Get<T> getter, {
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  });
}
```
#### AbstractAdapter
```dart
/// adpater 的抽象
abstract class AbstractAdapter<T> implements AbstractLogic<T> {
  ListAdapter buildAdapter(ContextSys<T> ctx);
}

/// 因为主 reudecer 会因为多层 state 的创建而变得复杂
/// 使用 ReducerFilter 来提升性能
typedef ReducerFilter<T> = bool Function(T state, Action action);
```

#### DispatchBus
```dart
/// 树状结构的节点？
abstract class DispatchBus {
  void attach(DispatchBus parent);

  void detach();

  void dispatch(Action action, {Dispatch excluded});

  void broadcast(Action action, {DispatchBus excluded});

  void Function() registerReceiver(Dispatch dispatch);
}
```



### /batch_store

#### _BatchNotify

```dart
mixin _BatchNotify<T> on Store<T> {
  final List<void Function()> _listeners = <void Function()>[];
  bool _isBatching = false;
  bool _isSetupBatch = false;
  T _prevState;

  void setupBatch() {
    if (!_isSetupBatch) {
      _isSetupBatch = true;
      super.subscribe(_batch);

      subscribe = (void Function() callback) {
        assert(callback != null);
        _listeners.add(callback);
        return () {
          _listeners.remove(callback);
        };
      };
    }
  }

  bool isInSuitablePhase() {
    return SchedulerBinding.instance != null &&
        SchedulerBinding.instance.schedulerPhase !=
            SchedulerPhase.persistentCallbacks &&
        !(SchedulerBinding.instance.schedulerPhase == SchedulerPhase.idle &&
            WidgetsBinding.instance.renderViewElement == null);
  }

  void _batch() {
    if (!isInSuitablePhase()) {
      if (!_isBatching) {
        _isBatching = true;
        SchedulerBinding.instance.addPostFrameCallback((Duration duration) {
          if (_isBatching) {
            _batch();
          }
        });
      }
    } else {
      final T curState = getState();
      if (!identical(_prevState, curState)) {
        _prevState = curState;

        final List<void Function()> notifyListeners = _listeners.toList(
          growable: false,
        );
        for (void Function() listener in notifyListeners) {
          listener();
        }

        _isBatching = false;
      }
    }
  }
}
```
#### _BatchStore
```dart
class _BatchStore<T> extends Store<T> with _BatchNotify<T> {
  _BatchStore(Store<T> store) : assert(store != null) {
    getState = store.getState;
    subscribe = store.subscribe;
    replaceReducer = store.replaceReducer;
    dispatch = store.dispatch;
    observable = store.observable;
    teardown = store.teardown;

    setupBatch();
  }
}
```
#### createBatchStore
```dart
Store<T> createBatchStore<T>(
  T preloadedState,
  Reducer<T> reducer, {
  StoreEnhancer<T> storeEnhancer,
}) =>
    _BatchStore<T>(
      createStore(
        preloadedState,
        _appendUpdateStateReducer<T>(reducer),
        storeEnhancer,
      ),
    );

/// connect with app-store

```
#### _appendUpdateStateReducer
```dart
enum _UpdateState { Assign }

// replace current state
Reducer<T> _appendUpdateStateReducer<T>(Reducer<T> reducer) =>
    (T state, Action action) => action.type == _UpdateState.Assign
        ? action.payload
        : reducer == null ? state : reducer(state, action);

Store<T> connectStores<T, K>(
  Store<T> mainStore,
  Store<K> extraStore,
  T Function(T, K) update,
) {
  final void Function() subscriber = () {
    final T prevT = mainStore.getState();
    final T nextT = update(prevT, extraStore.getState());
    if (nextT != null && !identical(prevT, nextT)) {
      mainStore.dispatch(Action(_UpdateState.Assign, payload: nextT));
    }
  };

  final void Function() unsubscribe = extraStore.subscribe(subscriber);

  /// should triggle once
  subscriber();

  final Future<dynamic> Function() superMainTD = mainStore.teardown;
  mainStore.teardown = () {
    unsubscribe?.call();
    return superMainTD();
  };

  return mainStore;
}
```



### /component

#### Component

```dart
/// 用于包裹一个 widget
/// e.g.
/// WidgetWrapper wrapper =
/// Widget buildRepaintBoundary(Widget child){
///		return RepaintBoundary(child: child);
/// }
typedef WidgetWrapper = Widget Function(Widget child);

/// Component 的不可变的配置类
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

  static ShouldUpdate<K> updateByDefault<K>() => (K _, K __) => !identical(_, __);

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
```

#### ComponenetWidget

```dart
/// 配合 Route.buildComponent 使用，构建一个Component
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



### /context

#### _ExtraMixin
```dart
mixin _ExtraMixin {
  Map<String, Object> _extra;
  Map<String, Object> get extra => _extra ??= <String, Object>{};
}
```
#### LogicContext
```dart
/// 默认 Context
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

  /// 通过指定的 name 获取到对应的 dependent，调用其 buildComponent 来获取一个 ComponentWidget
  /// 其中 dependent 是 connector + logic,在 dependencies 中注册  
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
```
#### ComponentContext
```dart
class ComponentContext<T> 
    extends LogicContext<T> 
    implements ViewUpdater<T> 
{
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



### /dependencies

#### Dependencies

```dart
class Dependencies<T> {
  /// 用于返回子 componenet 模块的 solots,便于管理 component 并实现复用 
  /// 其中 Dependent = AbstractConnector + AbstractLogic  
  final Map<String, Dependent<T>> slots;
  final Dependent<T> adapter;

  /// 使用 [adapter: NoneConn<T>() + Adapter<T>()] 来替代 [adapter: Adapter<T>()],
  /// 这样可以提高复用率和稳定性
  Dependencies({
    this.slots,
    this.adapter,
  }) : assert(
      	adapter == null || adapter.isAdapter(),
        'The dependent must contains adapter.'
  	   );
  
  /// 调用所有的 Dependent 的 createSubReducer() 来创建一个 SubReducer 列表,合成一个 Ruducer
  /// 即 Depenencies 的 Reducer 是由其 Dependent 的 reducer 组合而成的  
  Reducer<T> get reducer {
    final List<SubReducer<T>> subs = <SubReducer<T>>[];
    if (slots != null && slots.isNotEmpty) {
      subs.addAll(slots.entries.map<SubReducer<T>>(
        (MapEntry<String, Dependent<T>> entry) => entry.value.createSubReducer(),
      ));
    }

    if (adapter != null) {
      subs.add(adapter.createSubReducer());
    }

    return combineReducers(<Reducer<T>>[combineSubReducers(subs)]);
  }

  /// 获取一个 dependent   
  Dependent<T> slot(String type) => slots[type];
}
```



### /dependent

####  _Dependent

```dart
/// Dependent 的实现类，这个类对于用户来说是不可见的
/// Dependent 链接了 Connector 和 AbstractLogic(Componenet 或者 Adapter)
class _Dependent<T, P> implements Dependent<T> {
  final AbstractConnector<T, P> connector;
  final AbstractLogic<P> logic;

  _Dependent({
    @required this.logic,
    @required this.connector,
  })  : assert(logic != null),
        assert(connector != null);

  @override
  SubReducer<T> createSubReducer() {
    final Reducer<P> reducer = logic.reducer;
    return reducer != null ? connector.subReducer(reducer) : null;
  }

  @override
  Widget buildComponent(
    Store<Object> store,
    Get<T> getter, {
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  }) {
    assert(bus != null && enhancer != null);
    assert(isComponent(), 'Unexpected type of ${logic.runtimeType}.');
    final AbstractComponent<P> component = logic;
    return component.buildComponent(
      store,
      () => connector.get(getter()),
      bus: bus,
      enhancer: enhancer,
    );
  }

  @override
  ListAdapter buildAdapter(covariant ContextSys<P> ctx) {
    assert(isAdapter(), 'Unexpected type of ${logic.runtimeType}.');
    final AbstractAdapter<P> adapter = logic;
    return adapter.buildAdapter(ctx);
  }

  @override
  Get<P> subGetter(Get<T> getter) => () => connector.get(getter());

  @override
  ContextSys<P> createContext(
    Store<Object> store,
    BuildContext buildContext,
    Get<T> getState, {
    @required DispatchBus bus,
    @required Enhancer<Object> enhancer,
  }) {
    assert(bus != null && enhancer != null);
    return logic.createContext(
      store,
      buildContext,
      subGetter(getState),
      bus: bus,
      enhancer: enhancer,
    );
  }

  @override
  bool isComponent() => logic is AbstractComponent;

  @override
  bool isAdapter() => logic is AbstractAdapter;
}

/// 构建 Dependet，使用私有类
Dependent<K> createDependent<K, T>(
    AbstractConnector<K, T> connector, 
    AbstractLogic<T> logic
) => logic != null ? _Dependent<K, T>(connector: connector, logic: logic) : null;
```



### /dispatchBus

#### DispatchBusDefault

```dart
/// 默认实现的 DisptachBus
class DispatchBusDefault implements DispatchBus {
  static final DispatchBus shared = DispatchBusDefault();

  final List<Dispatch> _dispatchList = <Dispatch>[];
  DispatchBus parent;
  /// 这个方法是把自身的 dispatch 在父级中注册时获取的回调，可以将自身的 dispatch 在父级中注销  
  void Function() unregister;

  /// 无参数，无方法体的构造函数
  DispatchBusDefault();

  /// 添加父级  
  @override
  void attach(DispatchBus parent) {
    this.parent = parent;  
    unregister?.call();
    unregister = parent?.registerReceiver(dispatch);
  }

  /// 移除  
  @override
  void detach() {
    unregister?.call();
  }

  /// 发送 Action  
  /// 遍历 _dispatchList，找出所有不是 excluded 的 dispatch，组成列表 list
  /// 遍历 list 中的所有 dispatch 发送action  
  @override
  void dispatch(Action action, {Dispatch excluded}) {
    final List<Dispatch> list = _dispatchList
        .where((Dispatch dispatch) => dispatch != excluded)
        .toList(growable: false);

    for (Dispatch dispatch in list) {
      dispatch(action);
    }
  }

  /// 广播，调用父级的 dispatch，不会调用自身的 dispatch  
  @override
  void broadcast(Action action, {DispatchBus excluded}) {
    parent?.dispatch(action, excluded: excluded?.dispatch);
  }

  /// 注册一个 dispatch 
  /// 注意这里的返回值是 void Function
  /// 可以返回 VoidCallBack  
  @override
  void Function() registerReceiver(Dispatch dispatch) {
    assert(!_dispatchList.contains(dispatch),
        'Do not register a dispatch which is already existed');

    if (dispatch != null) {
      _dispatchList.add(dispatch);
      return () {
        _dispatchList.remove(dispatch);
      };
    } else {
      return null;
    }
  }
}
```



### /enhancer

#### EnhancerDefault

```dart
/// 默认实现的 enhancer 集成类
/// 提供 ViewMiddleware EffectMiddleware AdapterMiddleware 的修改默认实现
class EnhancerDefault<T> implements Enhancer<T> {
  StoreEnhancer<T> _storeEnhancer;
  ViewMiddleware<T> _viewEnhancer;
  EffectMiddleware<T> _effectEnhancer;
  AdapterMiddleware<T> _adapterEnhancer;

  final List<Middleware<T>> _middleware = <Middleware<T>>[];
  final List<ViewMiddleware<T>> _viewMiddleware = <ViewMiddleware<T>>[];
  final List<EffectMiddleware<T>> _effectMiddleware = <EffectMiddleware<T>>[];
  final List<AdapterMiddleware<T>> _adapterMiddleware =
      <AdapterMiddleware<T>>[];

  EnhancerDefault({
    List<Middleware<T>> middleware,
    List<ViewMiddleware<T>> viewMiddleware,
    List<EffectMiddleware<T>> effectMiddleware,
    List<AdapterMiddleware<T>> adapterMiddleware,
  }) {
    append(
      middleware: middleware,
      viewMiddleware: viewMiddleware,
      effectMiddleware: effectMiddleware,
      adapterMiddleware: adapterMiddleware,
    );
  }

  @override
  void unshift({
    List<Middleware<T>> middleware,
    List<ViewMiddleware<T>> viewMiddleware,
    List<EffectMiddleware<T>> effectMiddleware,
    List<AdapterMiddleware<T>> adapterMiddleware,
  }) {
    if (middleware != null) {
      _middleware.insertAll(0, middleware);
      _storeEnhancer = applyMiddleware<T>(_middleware);
    }
    if (viewMiddleware != null) {
      _viewMiddleware.insertAll(0, viewMiddleware);
      _viewEnhancer = mergeViewMiddleware<T>(_viewMiddleware);
    }
    if (effectMiddleware != null) {
      _effectMiddleware.insertAll(0, effectMiddleware);
      _effectEnhancer = mergeEffectMiddleware<T>(_effectMiddleware);
    }
    if (adapterMiddleware != null) {
      _adapterMiddleware.insertAll(0, adapterMiddleware);
      _adapterEnhancer = mergeAdapterMiddleware<T>(_adapterMiddleware);
    }
  }

  @override
  void append({
    List<Middleware<T>> middleware,
    List<ViewMiddleware<T>> viewMiddleware,
    List<EffectMiddleware<T>> effectMiddleware,
    List<AdapterMiddleware<T>> adapterMiddleware,
  }) {
    if (middleware != null) {
      _middleware.addAll(middleware);
      _storeEnhancer = applyMiddleware<T>(_middleware);
    }
    if (viewMiddleware != null) {
      _viewMiddleware.addAll(viewMiddleware);
      _viewEnhancer = mergeViewMiddleware<T>(_viewMiddleware);
    }
    if (effectMiddleware != null) {
      _effectMiddleware.addAll(effectMiddleware);
      _effectEnhancer = mergeEffectMiddleware<T>(_effectMiddleware);
    }
    if (adapterMiddleware != null) {
      _adapterMiddleware.addAll(adapterMiddleware);
      _adapterEnhancer = mergeAdapterMiddleware<T>(_adapterMiddleware);
    }
  }

  @override
  ViewBuilder<K> viewEnhance<K>(
    ViewBuilder<K> view,
    AbstractComponent<K> component,
    Store<T> store,
  ) {
    return _viewEnhancer
        ?.call(component, store)
        ?.call(_inverterView<K>(view)) 
        ?? view;
  }
     

  @override
  AdapterBuilder<K> adapterEnhance<K>(
    AdapterBuilder<K> adapterBuilder,
    AbstractAdapter<K> logic,
    Store<T> store,
  ) =>
      _adapterEnhancer
          ?.call(logic, store)
          ?.call(_inverterAdapter<K>(adapterBuilder)) ??
      adapterBuilder;

  @override
  Effect<K> effectEnhance<K>(
    Effect<K> effect,
    AbstractLogic<K> logic,
    Store<T> store,
  ) =>
      _effectEnhancer?.call(logic, store)?.call(_inverterEffect<K>(effect)) ??
      effect;

  @override
  StoreCreator<T> storeEnhance(StoreCreator<T> creator) =>
      _storeEnhancer?.call(creator) ?? creator;

  Effect<dynamic> _inverterEffect<K>(Effect<K> effect) => effect == null
      ? null
      : (Action action, Context<dynamic> ctx) => effect(action, ctx);

  ViewBuilder<dynamic> _inverterView<K>(ViewBuilder<K> view) => view == null
      ? null
      : (dynamic state, Dispatch dispatch, ViewService viewService) =>
          view(state, dispatch, viewService);

  AdapterBuilder<dynamic> _inverterAdapter<K>(AdapterBuilder<K> adapter) =>
      adapter == null
          ? null
          : (dynamic state, Dispatch dispatch, ViewService viewService) =>
              adapter(state, dispatch, viewService);
}
```



### /helper

```dart
AdapterBuilder<T> asAdapter<T>(ViewBuilder<T> view) {
  return (T unstableState, Dispatch dispatch, ViewService service) {
    final ContextSys<T> ctx = service;
    return ListAdapter(
      (BuildContext buildContext, int index) =>
          view(ctx.state, dispatch, service),
      1,
    );
  };
}

Reducer<T> mergeReducers<T extends K, K>(Reducer<K> sup, [Reducer<T> sub]) {
  return (T state, Action action) {
    return sub?.call(sup(state, action), action) ?? sup(state, action);
  };
}

Effect<T> mergeEffects<T extends K, K>(Effect<K> sup, [Effect<T> sub]) {
  return (Action action, Context<T> ctx) {
    return sub?.call(action, ctx) ?? sup.call(action, ctx);
  };
}

/// combine & as
/// for action.type which override it's == operator
Reducer<T> asReducer<T>(Map<Object, Reducer<T>> map) => (map == null ||
        map.isEmpty)
    ? null
    : (T state, Action action) =>
        map.entries
            .firstWhere(
                (MapEntry<Object, Reducer<T>> entry) =>
                    action.type == entry.key,
                orElse: () => null)
            ?.value(state, action) ??
        state;

Reducer<T> filterReducer<T>(Reducer<T> reducer, ReducerFilter<T> filter) {
  return (reducer == null || filter == null)
      ? reducer
      : (T state, Action action) {
          return filter(state, action) ? reducer(state, action) : state;
        };
}

const Object _SUB_EFFECT_RETURN_NULL = Object();

typedef SubEffect<T> = FutureOr<void> Function(Action action, Context<T> ctx);

/// for action.type which override it's == operator
/// return [UserEffecr]
Effect<T> combineEffects<T>(Map<Object, SubEffect<T>> map) =>
    (map == null || map.isEmpty)
        ? null
        : (Action action, Context<T> ctx) {
            final SubEffect<T> subEffect = map.entries
                .firstWhere(
                  (MapEntry<Object, SubEffect<T>> entry) =>
                      action.type == entry.key,
                  orElse: () => null,
                )
                ?.value;

            if (subEffect != null) {
              return subEffect.call(action, ctx) ?? _SUB_EFFECT_RETURN_NULL;
            }

            //skip-lifecycle-actions
            if (action.type is Lifecycle) {
              return _SUB_EFFECT_RETURN_NULL;
            }

            /// no subEffect
            return null;
          };

/// return [EffectDispatch]
Dispatch createEffectDispatch<T>(Effect<T> userEffect, Context<T> ctx) {
  return (Action action) {
    final Object result = userEffect?.call(action, ctx);

    //skip-lifecycle-actions
    if (action.type is Lifecycle && (result == null || result == false)) {
      return _SUB_EFFECT_RETURN_NULL;
    }

    return result;
  };
}

/// return [NextDispatch]
Dispatch createNextDispatch<T>(ContextSys<T> ctx) => (Action action) {
      ctx.broadcastEffect(action);
      ctx.store.dispatch(action);
    };

/// return [Dispatch]
Dispatch createDispatch<T>(Dispatch onEffect, Dispatch next, Context<T> ctx) =>
    (Action action) {
      final Object result = onEffect?.call(action);
      if (result == null || result == false) {
        next(action);
      }

      return result == _SUB_EFFECT_RETURN_NULL ? null : result;
    };

ViewMiddleware<T> mergeViewMiddleware<T>(List<ViewMiddleware<T>> middleware) {
  return Collections.reduce<ViewMiddleware<T>>(middleware,
      (ViewMiddleware<T> first, ViewMiddleware<T> second) {
    return (AbstractComponent<dynamic> component, Store<T> store) {
      final Composable<ViewBuilder<dynamic>> inner = first(component, store);
      final Composable<ViewBuilder<dynamic>> outer = second(component, store);
      return (ViewBuilder<dynamic> view) {
        return outer(inner(view));
      };
    };
  });
}

AdapterMiddleware<T> mergeAdapterMiddleware<T>(
    List<AdapterMiddleware<T>> middleware) {
  return Collections.reduce<AdapterMiddleware<T>>(middleware,
      (AdapterMiddleware<T> first, AdapterMiddleware<T> second) {
    return (AbstractAdapter<dynamic> component, Store<T> store) {
      final Composable<AdapterBuilder<dynamic>> inner = first(component, store);
      final Composable<AdapterBuilder<dynamic>> outer =
          second(component, store);
      return (AdapterBuilder<dynamic> view) {
        return outer(inner(view));
      };
    };
  });
}

EffectMiddleware<T> mergeEffectMiddleware<T>(
    List<EffectMiddleware<T>> middleware) {
  return Collections.reduce<EffectMiddleware<T>>(middleware,
      (EffectMiddleware<T> first, EffectMiddleware<T> second) {
    return (AbstractLogic<dynamic> logic, Store<T> store) {
      final Composable<Effect<dynamic>> inner = first(logic, store);
      final Composable<Effect<dynamic>> outer = second(logic, store);
      return (Effect<dynamic> effect) {
        return outer(inner(effect));
      };
    };
  });
}
```



### /lifecycle

#### Lifecycle
```dart
/// 枚举生命周期
enum Lifecycle {
  /// componenmt(page) 或者 adapter 接受如下 events
  initState,
  didChangeDependencies,
  build,
  reassemble,
  didUpdateWidget,
  deactivate,
  dispose,
  // didDisposed,

  /// 适用于列表的生命周期  
  /// 只有 adapter 的 mixin VisibleChangeMixin 能够接受 appear & disappear events.
  /// class MyAdapter extends Adapter<T> with VisibleChangeMixin<T> {
  ///   MyAdapter():super(
  ///     ///
  ///   );
  /// }
  appear,
  disappear,

  /// 只有 componenmt(page) or adapter mixin WidgetsBindingObserverMixin 会接受 
  /// didChangeAppLifecycleState event.
  /// class MyComponent extends Component<T> with WidgetsBindingObserverMixin<T> {
  ///   MyComponent():super(
  ///     ///
  ///   );
  /// }
  didChangeAppLifecycleState,
}
```
#### LifecycleCreator
```dart
/// 默认实现的生命周期 Action 发生器
class LifecycleCreator {
  static Action initState() => const Action(Lifecycle.initState);

  static Action build(String name) => Action(Lifecycle.build, payload: name);

  static Action reassemble() => const Action(Lifecycle.reassemble);

  static Action dispose() => const Action(Lifecycle.dispose);

  // static Action didDisposed() => const Action(Lifecycle.didDisposed);

  static Action didUpdateWidget() => const Action(Lifecycle.didUpdateWidget);

  static Action didChangeDependencies() =>
      const Action(Lifecycle.didChangeDependencies);

  static Action deactivate() => const Action(Lifecycle.deactivate);

  static Action appear(int index) => Action(Lifecycle.appear, payload: index);

  static Action disappear(int index) =>
      Action(Lifecycle.disappear, payload: index);

  static Action didChangeAppLifecycleState(AppLifecycleState state) =>
      Action(Lifecycle.didChangeAppLifecycleState, payload: state);
}
```



### /logic

#### Logic

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
      combineReducers<T>(<Reducer<T>>[protectedReducer, protectedDependencies?.reducer]),
      protectedFilter
  );

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
        ctx
    );
  }

  @override
  Dispatch createNextDispatch(ContextSys<T> ctx, Enhancer<Object> enhancer) =>
      helper.createNextDispatch<T>(ctx);

  @override
  Dispatch createDispatch(
    Dispatch effectDispatch,
    Dispatch nextDispatch,
    Context<T> ctx,
  ) => helper.createDispatch<T>(effectDispatch, nextDispatch, ctx);

  @override
  Object key(T state) => _key?.call(state) ?? ValueKey<Type>(runtimeType);

  @override
  Dependent<T> slot(String type) => protectedDependencies?.slot(type);

  @override
  Dependent<T> adapterDep() => protectedDependencies?.adapter;
}
```



### /page

```dart
/// init store's state by route-params
typedef InitState<T, P> = T Function(P params);

typedef StoreUpdater<T> = Store<T> Function(Store<T> store);
```
#### Page
```dart
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
```
#### _PageWidget
```dart
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
```
#### _PageState
```dart
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
```



## Redux_Connnector

### /connnector

#### connOP

```dart
/// mixin 提供了 + 运算符重载
class ConnOp<T, P> extends MutableConn<T, P> with ConnOpMixin<T, P> {
  final P Function(T) _getter;
  final void Function(T, P) _setter;

  const ConnOp({
    P Function(T) get,
    void Function(T, P) set,
  })  : _getter = get,
        _setter = set;

  @override
  P get(T state) => _getter(state);

  @override
  void set(T state, P subState) => _setter(state, subState);
}

```



### /generator

#### generator

```dart
String Function() generator() {
  int nextId = 0;
  String prefix = '';
  return () {
    /// fix '0x3FFFFFFFFFFFFFFF' can't be represented exactly in JavaScript.
    if (++nextId >= 0x3FFFFFFF) {
      nextId = 0;
      prefix = '\$' + prefix;
    }
    return prefix + nextId.toString();
  };
}
```



### /map_like

#### MapLike

```dart
/// 可复制的 map 的抽象
abstract class MapLike {
  Map<String, Object> _fieldsMap = <String, Object>{};

  void clear() => _fieldsMap.clear();

  Object operator [](String key) => _fieldsMap[key];

  void operator []=(String key, Object value) => _fieldsMap[key] = value;

  bool containsKey(String key) => _fieldsMap.containsKey(key);

  void copyFrom(MapLike from) =>
      _fieldsMap = <String, Object>{}..addAll(from._fieldsMap);
}
```
#### withMapLike
```dart
/// 通过给定的 key 返回一个 ConnOp<T, P>
ConnOp<T, P> withMapLike<T extends MapLike, P>(String key) => ConnOp<T, P>(
    get: (T state) => state[key],
    set: (T state, P sub) => state[key] = sub,
);
```
#### AutoInitConnector
```dart
/// 自动初始化的 connector
class AutoInitConnector<T extends MapLike, P> extends ConnOp<T, P> {
  /// 每次构建一个 AutoInitConnector<T extends MapLike, P>
  /// 都会调用 generator() 来产生一个 String id  
  static final String Function() _gen = generator();

  final String _key;
  final void Function(T state, P sub) _setHook;
  final P Function(T state) init;

  AutoInitConnector(this.init, {String key, void set(T state, P sub)})
      : assert(init != null),
        _setHook = set,
        _key = key ?? _gen();

  @override
  P get(T state) =>
      state.containsKey(_key) ? state[_key] : (state[_key] = init(state));

  @override
  void set(T state, P subState) {
    state[_key] = subState;
    _setHook?.call(state, subState);
  }
}
```



### /none

#### NoneConn

```dart
/// 不做任何中间处理的 Connector
class NoneConn<T> extends ImmutableConn<T, T> with ConnOpMixin<T, T> {
  @override
  T get(T state) => state;

  @override
  T set(T state, T subState) => subState;
}
```



### /op_mixin

#### ConnOpMixin

```dart
/// 重载运算符操作
/// AbstractConnector + AbstractLogic<P> = Dependent<T> 
mixin ConnOpMixin<T, P> on AbstractConnector<T, P> {
  Dependent<T> operator +(AbstractLogic<P> logic) =>
      createDependent<T, P>(this, logic);
}
```



### /reselector

#### _listEquals

```dart
bool _listEquals(List<dynamic> list1, List<dynamic> list2) {
  if (identical(list1, list2)) {
    return true;
  }
  if (list1 == null || list2 == null) {
    return false;
  }
  final int length = list1.length;
  if (length != list2.length) {
    return false;
  }
  for (int i = 0; i < length; i++) {
    final dynamic e1 = list1[i], e2 = list2[i];
    if (e1 != e2) {
      if (e1 is List && e1.runtimeType == e2?.runtimeType) {
        if (!_listEquals(e1, e2)) {
          return false;
        }
      }
      return false;
    }
  }
  return true;
}
```
#### _BasicReselect
```dart
abstract class _BasicReselect<T, P> 
    extends MutableConn<T, P>
    with ConnOpMixin<T, P> {
  List<dynamic> _subsCache;
  P _pCache;
  bool _hasBeenCalled = false;

  List<dynamic> getSubs(T state);

  P reduceSubs(List<dynamic> list);

  @override
  P get(T state) {
    final List<dynamic> subs = getSubs(state);
    if (!_hasBeenCalled || !_listEquals(subs, _subsCache)) {
      _subsCache = subs;
      _pCache = reduceSubs(_subsCache);
      _hasBeenCalled = true;
    }
    return _pCache;
  }
}
```
#### Reselect1
```dart
abstract class Reselect1<T, P, K0> extends _BasicReselect<T, P> {
  K0 getSub0(T state);
  P computed(K0 state);

  @override
  List<dynamic> getSubs(T state) => <dynamic>[getSub0(state)];

  @override
  P reduceSubs(List<dynamic> list) => Function.apply(computed, list);
}
```
#### Reselect2
```dart
abstract class Reselect2<T, P, K0, K1> extends _BasicReselect<T, P> {
  K0 getSub0(T state);
  K1 getSub1(T state);
  P computed(K0 sub0, K1 sub1);

  @override
  List<dynamic> getSubs(T state) => <dynamic>[getSub0(state), getSub1(state)];

  @override
  P reduceSubs(List<dynamic> list) => Function.apply(computed, list);
}
```
#### Reselect3
```dart
abstract class Reselect3<T, P, K0, K1, K2> extends _BasicReselect<T, P> {
  K0 getSub0(T state);
  K1 getSub1(T state);
  K2 getSub2(T state);
  P computed(K0 sub0, K1 sub1, K2 sub2);

  @override
  List<dynamic> getSubs(T state) => <dynamic>[
        getSub0(state),
        getSub1(state),
        getSub2(state),
      ];

  @override
  P reduceSubs(List<dynamic> list) => Function.apply(computed, list);
}
```
#### ...

#### Reselect
```dart
abstract class Reselect<T, P> extends _BasicReselect<T, P> {
  P computed(List<dynamic> list);

  @override
  P reduceSubs(List<dynamic> list) => Function.apply(computed, list);
}
```



## Reudx_Component_Mixin

### /keep_alive_mixin

#### _KeepAlivesStfState

```dart
/// usage
/// class MyComponent extends Component<T> with KeepAliveMixin<T> {
///   MyComponent():super(
///     ///
///   );
/// }
mixin KeepAliveMixin<T> on Component<T> {
  @override
  ComponentState<T> createState() => _KeepAliveStfState<T>();
}

class _KeepAliveStfState<T> extends ComponentState<T>
    with AutomaticKeepAliveClientMixin {
  @override
  bool get wantKeepAlive => true;

  @override
  Widget build(BuildContext context) {
    super.build(context);
    return ctx.buildWidget();
  }
}

```



### /private_reducer_mixin

#### PrivateReducerMixin
```dart
/// usage
/// class MyComponent extends Component<T> with PrivateReducerMixin<T> {
///   MyComponent():super(
///     ///
///   );
/// }
mixin PrivateReducerMixin<T> on Logic<T> {
  @override
  Reducer<T> get protectedReducer {
    final Reducer<T> superReducer = super.protectedReducer;
    return superReducer != null
        ? (T state, Action action) {
            if (action is PrivateAction && action.target == state) {
              return superReducer(state, action.asAction());
            }
            return state;
          }
        : null;
  }

  @override
  Dispatch createDispatch(Dispatch effect, Dispatch next, Context<T> ctx) {
    final Dispatch superDispatch = super.createDispatch(effect, next, ctx);
    return (Action action) {
      if (action.type is! Lifecycle && action is! PrivateAction) {
        action = PrivateAction(
          action.type,
          payload: action.payload,
          target: ctx.state,
        );
      }
      return superDispatch(action);
    };
  }
}
```
#### PrivateAction
```dart
class PrivateAction extends Action {
  final Object target;
  PrivateAction(Object type, {dynamic payload, this.target})
      : super(type, payload: payload);

  Action asAction() => Action(type, payload: payload);
}
```



### /single_ticker_provider_mixin

#### _SingleTickerProviderStfState

```dart
/// usage
/// class MyComponent extends Component<T> with SingleTickerProviderMixin<T> {
///   MyComponent():super(
///     ///
///   );
/// }
mixin SingleTickerProviderMixin<T> on Component<T> {
  @override
  _SingleTickerProviderStfState<T> createState() =>
      _SingleTickerProviderStfState<T>();
}

class _SingleTickerProviderStfState<T> extends ComponentState<T>
    with SingleTickerProviderStateMixin {
  /// fix SingleTickerProviderStateMixin dispose bug
  @override
  void dispose() {
    disposeCtx();
    super.dispose();
  }
}
```



### /ticker_provider_mixin

#### _TickerProviderStfStat

```dart
/// usage
/// class MyComponent extends Component<T> with TickerProviderMixin<T> {
///   MyComponent():super(
///     ///
///   );
/// }
mixin TickerProviderMixin<T> on Component<T> {
  @override
  _TickerProviderStfState<T> createState() => _TickerProviderStfState<T>();
}

class _TickerProviderStfState<T> extends ComponentState<T>
    with TickerProviderStateMixin {
  /// fix TickerProviderStateMixin dispose bug
  @override
  void dispose() {
    disposeCtx();
    super.dispose();
  }
}
```



### /visible_change_mixin

#### VisibleChangeMixin
```dart
/// usage
/// class MyAdapter extends Adapter<T> with VisibleChangeMixin<T> {
///   MyAdapter():super(
///     ///
///   );
/// }
mixin VisibleChangeMixin<T> on AbstractAdapter<T> {
  @override
  ListAdapter buildAdapter(ContextSys<T> ctx) {
    return _wrapVisibleChange<T>(super.buildAdapter(ctx), ctx);
  }
}
```
####  _VisibleChangeState
```dart
class _VisibleChangeState extends State<_VisibleChangeWidget> {
  @override
  Widget build(BuildContext context) =>
      widget.itemBuilder(context, widget.index);

  @override
  void initState() {
    super.initState();
    widget.dispatch(LifecycleCreator.appear(widget.index));
  }

  @override
  void dispose() {
    widget.dispatch(LifecycleCreator.disappear(widget.index));
    super.dispose();
  }
}
```
####  _VisibleChangeWidget
```dart
class _VisibleChangeWidget extends StatefulWidget {
  final IndexedWidgetBuilder itemBuilder;
  final int index;
  final Dispatch dispatch;

  const _VisibleChangeWidget({
    Key key,
    this.itemBuilder,
    this.index,
    this.dispatch,
  }) : super(key: key);

  @override
  State<StatefulWidget> createState() => _VisibleChangeState();
}

ListAdapter _wrapVisibleChange<T>(
  ListAdapter listAdapter,
  LogicContext<T> ctx,
) {
  final _VisibleChangeDispatch onChange =
      (ctx.extra['\$visible'] ??= _VisibleChangeDispatch(ctx.dispatch));

  return listAdapter == null
      ? null
      : ListAdapter(
          (BuildContext buildContext, int index) => _VisibleChangeWidget(
            itemBuilder: listAdapter.itemBuilder,
            index: index,
            dispatch: onChange.onAction,
            key: ValueKey<Tuple2<Object, int>>(Tuple2<Object, int>(ctx, index)),
          ),
          listAdapter.itemCount,
        );
}
```
#### _VisibleChangeDispatch
```dart
class _VisibleChangeDispatch extends AutoDispose {
  int _appearsCount = 0;
  final Dispatch dispatch;

  _VisibleChangeDispatch(this.dispatch);

  void onAction(Action action) {
    if (action.type == Lifecycle.appear) {
      assert(_appearsCount >= 0);
      if (_appearsCount == 0) {
        if (!isDisposed) {
          dispatch(action);
        }
      }
      _appearsCount++;
    } else if (action.type == Lifecycle.disappear) {
      _appearsCount--;
      assert(_appearsCount >= 0);
      if (_appearsCount == 0) {
        if (!isDisposed) {
          dispatch(action);
        }
      }
    }
  }
}
```



### /widgets_biding_observer

#### _WidgetsBindingObserverStfState

```dart
/// usage
/// class MyComponent extends Component<T> with WidgetsBindingObserverMixin<T> {
///   MyComponent():super(
///     ///
///   );
/// }
mixin WidgetsBindingObserverMixin<T> on Component<T> {
  @override
  _WidgetsBindingObserverStfState<T> createState() =>
      _WidgetsBindingObserverStfState<T>();
}

class _WidgetsBindingObserverStfState<T> 
    extends ComponentState<T>
    with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    super.didChangeAppLifecycleState(state);
    ctx.onLifecycle(LifecycleCreator.didChangeAppLifecycleState(state));
  }
}
```





## Redux_MiddleWare

### /adapter_middleware

#### safetyAdapter

```dart
/// type = {0, 1}
AdapterMiddleware<T> safetyAdapter<T>({
  Widget Function(dynamic, StackTrace,
          {AbstractAdapter<dynamic> adapter, Store<T> store, int type})
      onError,
}) {
  return (AbstractAdapter<dynamic> adapter, Store<T> store) {
    return (AdapterBuilder<dynamic> next) {
      return isDebug()
          ? next
          : (dynamic state, Dispatch dispatch, ViewService viewService) {
              try {
                final ListAdapter result = next(state, dispatch, viewService);
                return ListAdapter((BuildContext buildContext, int index) {
                  try {
                    return result.itemBuilder(buildContext, index);
                  } catch (e, stackTrace) {
                    return onError?.call(
                          e,
                          stackTrace,
                          adapter: adapter,
                          store: store,
                          type: 1,
                        ) ??
                        Container(width: 0, height: 0);
                  }
                }, result.itemCount);
              } catch (e, stackTrace) {
                final Widget errorWidget = onError?.call(
                  e,
                  stackTrace,
                  adapter: adapter,
                  store: store,
                  type: 0,
                );
                return errorWidget == null
                    ? const ListAdapter(null, 0)
                    : ListAdapter(
                        (BuildContext buildContext, int index) => errorWidget,
                        1);
              }
            };
    };
  };
}
```



### /middleware

#### logMiddleware

```dart
/// Middleware for print action dispatch.
/// It works on debug mode.
Middleware<T> logMiddleware<T>({
  String tag = 'redux',
  String Function(T) monitor,
}) {
  return ({Dispatch dispatch, Get<T> getState}) {
    return (Dispatch next) {
      return isDebug()
          ? (Action action) {
              print('---------- [$tag] ----------');
              print('[$tag] ${action.type} ${action.payload}');

              final T prevState = getState();
              if (monitor != null) {
                print('[$tag] prev-state: ${monitor(prevState)}');
              }

              next(action);

              final T nextState = getState();
              if (monitor != null) {
                print('[$tag] next-state: ${monitor(nextState)}');
              }

              // if (prevState == nextState) {
              //   print('[$tag] warning: ${action.type} has not been used.');
              // }

              print('========== [$tag] ================');
            }
          : next;
    };
  };
}
```

#### performanceMiddleware

```dart
/// Middleware for print action dispatch performance by time consuming.
/// It works on debug mode.
Middleware<T> performanceMiddleware<T>({String tag = 'redux'}) {
  return ({Dispatch dispatch, Get<T> getState}) {
    return (Dispatch next) {
      return isDebug()
          ? (Action action) {
              final int markPrev = DateTime.now().microsecondsSinceEpoch;
              next(action);
              final int markNext = DateTime.now().microsecondsSinceEpoch;
              print('$tag performance: ${action.type} ${markNext - markPrev}');
            }
          : next;
    };
  };
}
```



### /view_middleware

```dart
import 'package:flutter/widgets.dart' hide Action;

import '../../redux/redux.dart';
import '../../redux_component/redux_component.dart';
import '../../utils/utils.dart';

ViewMiddleware<T> safetyView<T>(
    {Widget Function(dynamic, StackTrace,
            {AbstractComponent<dynamic> component, Store<T> store})
        onError}) {
  return (AbstractComponent<dynamic> component, Store<T> store) {
    return (ViewBuilder<dynamic> next) {
      return isDebug()
          ? next
          : (dynamic state, Dispatch dispatch, ViewService viewService) {
              try {
                return next(state, dispatch, viewService);
              } catch (e, stackTrace) {
                return onError?.call(
                      e,
                      stackTrace,
                      component: component,
                      store: store,
                    ) ??
                    Container(width: 0, height: 0);
              }
            };
    };
  };
}
```



## Redux_Routes

### /page_routes

```dart
import 'package:flutter/widgets.dart';

import '../redux_component/redux_component.dart';

/// Define a basic behavior of routes.
abstract class AbstractRoutes {
  Widget buildPage(String path, dynamic arguments);
}

/// Each page has a unique store.
@immutable
class PageRoutes implements AbstractRoutes {
  final Map<String, Page<Object, dynamic>> pages;

  PageRoutes({
    @required this.pages,

    /// For common enhance
    void Function(String, Page<Object, dynamic>) visitor,
  }) : assert(pages != null, 'Expected the pages to be non-null value.') {
    if (visitor != null) {
      pages.forEach(visitor);
    }
  }

  @override
  Widget buildPage(String path, dynamic arguments) =>
      pages[path]?.buildPage(arguments);
}
```



## Utils

### /collections

```dart
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



### /debug

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



# 流程阐述 & 框架分析



# 后续




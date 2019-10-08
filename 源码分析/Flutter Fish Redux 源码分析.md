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

以下内容摘自 FishRedux 的 Doc

#### 1.它们是解决不同层面问题的两个框架

> Redux 是一个专注于状态管理的框架；Fish Redux 是基于 Redux 做状态管理的应用框架。
应用框架不仅仅要解决状态管理的问题，还要解决分治，通信，数据驱动，解耦等等问题。

#### 2.Fish Redux 解决了集中和分治的矛盾。

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





## Basic

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

/// 类型获取器的定义（获取 Type R）
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

/// Create a store definition
typedef StoreCreator<T> = Store<T> Function(
  T preloadedState,
  Reducer<T> reducer,
);

/// Definition of Enhanced creating a store
typedef StoreEnhancer<T> = StoreCreator<T> Function(StoreCreator<T> creator);

/// Definition of SubReducer
/// [isStateCopied] is Used to optimize execution performance.
/// Ensure that a T will be cloned at most once during the entire process.
typedef SubReducer<T> = T Function(T state, Action action, bool isStateCopied);

/// Definition of Connector which connects Reducer<S> with Reducer<P>.
/// 1. How to get an instance of type P from an instance of type S.
/// 2. How to synchronize changes of an instance of type P to an instance of type S.
/// 3. How to clone a new S.
abstract class AbstractConnector<S, P> {
  P get(S state);

  /// For mutable state, there are three abilities needed to be met.
  ///     1. get: (S) => P
  ///     2. set: (S, P) => void
  ///     3. shallow copy: s.clone()
  ///
  /// For immutable state, there are two abilities needed to be met.
  ///     1. get: (S) => P
  ///     2. set: (S, P) => S
  ///
  /// See in [connector].
  SubReducer<S> subReducer(Reducer<P> reducer);
}
```




# Flutter Fish Redux 源码分析



[TOC]

## FishRedux - github

https://github.com/alibaba/fish-redux



## 规定项目结构

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



## Redux 和 FishRedux 的不同

以下摘自 FishRedux 的 Doc

#### 它们是解决不同层面问题的两个框架

> Redux 是一个专注于状态管理的框架；Fish Redux 是基于 Redux 做状态管理的应用框架。

> 应用框架不仅仅要解决状态管理的问题，还要解决分治，通信，数据驱动，解耦等等问题。

#### Fish Redux 解决了集中和分治的矛盾。

> Redux 通过使用者手动组织代码的形式来完成从小的 Reducer 到主 Reducer 的合并过程；

> Fish Redux 通过显式的表达组件之间的依赖关系，由框架自动完成从细力度的 Reducer 到主 Reducer 的合并过程；

[![img](https://camo.githubusercontent.com/e973ad0e5b01057bd6a68ed5e6d303c0b686f894/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f5442316f65584b4a5950704b31526a535a464658586135507058612d313937362d3536382e706e67)](https://camo.githubusercontent.com/e973ad0e5b01057bd6a68ed5e6d303c0b686f894/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f5442316f65584b4a5950704b31526a535a464658586135507058612d313937362d3536382e706e67)

#### Fish Redux 提供了一个简单的组件抽象模型

> 它通过简单的 3 个函数组合而成

[![img](https://camo.githubusercontent.com/1b9897360edb417b55e46dc7dd4c858d7971ca6c/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f544231767142324a3459614b31526a535a466e58586138307058612d3930302d3738302e706e67)](https://camo.githubusercontent.com/1b9897360edb417b55e46dc7dd4c858d7971ca6c/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f544231767142324a3459614b31526a535a466e58586138307058612d3930302d3738302e706e67)

#### Fish Redux 提供了一个 Adapter 的抽象组件模型

> 在基础的组件模型以外，Fish Redux 提供了一个 Adapter 抽象模型，用来解决在 ListView 上大 Cell 的性能问题。

> 通过上层抽象，我们得到了逻辑上的 ScrollView，性能上的 ListView。

[![img](https://camo.githubusercontent.com/6f1acb86c10851915f71ad6582f795a13c66d3ee/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f544231783531564a37506f4b31526a535a4b6258585831495858612d313835322d3631322e706e67)](https://camo.githubusercontent.com/6f1acb86c10851915f71ad6582f795a13c66d3ee/68747470733a2f2f696d672e616c6963646e2e636f6d2f7466732f544231783531564a37506f4b31526a535a4b6258585831495858612d313835322d3631322e706e67)



## 流程图

[![img](https://user-gold-cdn.xitu.io/2019/4/30/16a6d97f0a8365dc?imageslim)](https://user-gold-cdn.xitu.io/2019/4/30/16a6d97f0a8365dc?imageslim)




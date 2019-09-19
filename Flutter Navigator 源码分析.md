# Flutter Navigator 源码分析

[TOC]

## RouteSetting

```dart
/// 配置路由
/// 本身为不可变类
@immutable
class RouteSettings {
  const RouteSettings({
    this.name,
    this.isInitialRoute = false,
    this.arguments,
  });

  RouteSettings copyWith({
    String name,
    bool isInitialRoute,
    Object arguments,
  }) {
    return RouteSettings(
      name: name ?? this.name,
      isInitialRoute: isInitialRoute ?? this.isInitialRoute,
      arguments: arguments ?? this.arguments,
    );
  }

  /// 路由名
  final String name;

  /// 初始路由（第一个加入[Navigator]的路由）
  ///
  /// 这个路由会跳过所有的动画效果，用于加速加载
  final bool isInitialRoute;

  /// 参数
  final Object arguments;

  @override
  String toString() => '$runtimeType("$name", $arguments)';
}
```



## Route

```dart
abstract class Route<T> {
  /// 初始化路由
  Route({ RouteSettings settings }) : settings = settings ?? const RouteSettings();

  /// 这个路由所处的 Navigator
  NavigatorState get navigator => _navigator;
  NavigatorState _navigator;

  /// 配置
  final RouteSettings settings;

  /// 路由覆盖层
  List<OverlayEntry> get overlayEntries => const <OverlayEntry>[];

  /// 向 navigator 中插入 route 时调用
  @protected
  @mustCallSuper
  void install(OverlayEntry insertionPoint) { }

  /// 在[install]之后，当向navigator中加入路由时调用
  /// 返回值是Transition的过程（Future）
  /// 在此方法之后，立刻调用[didChangeNext] 和 [didChangePrevious]  
  @protected
  TickerFuture didPush() => TickerFuture.complete();

  /// 在[install]之后，当route替换另一个route时调用
  ///
  /// 在此方法之后，立刻调用[didChangeNext] 和 [didChangePrevious]
  @protected
  @mustCallSuper
  void didReplace(Route<dynamic> oldRoute) { }

  Future<RoutePopDisposition> willPop() async {
    return isFirst ? RoutePopDisposition.bubble : RoutePopDisposition.pop;
  }

  /// Whether calling [didPop] would return false.
  bool get willHandlePopInternally => false;

  T get currentResult => null;

  Future<T> get popped => _popCompleter.future;
  final Completer<T> _popCompleter = Completer<T>();

  /// 请求退出当前路由
  @protected
  @mustCallSuper
  bool didPop(T result) {
    didComplete(result);
    return true;
  }

  /// 完成future
  @protected
  @mustCallSuper
  void didComplete(T result) {
    _popCompleter.complete(result);
  }

  /// The given route, which was above this one, has been popped off the
  /// navigator.
  @protected
  @mustCallSuper
  void didPopNext(Route<dynamic> nextRoute) { }

  /// This route's next route has changed to the given new route. This is called
  /// on a route whenever the next route changes for any reason, so long as it
  /// is in the history, including when a route is first added to a [Navigator]
  /// (e.g. by [Navigator.push]), except for cases when [didPopNext] would be
  /// called. `nextRoute` will be null if there's no next route.
  @protected
  @mustCallSuper
  void didChangeNext(Route<dynamic> nextRoute) { }

  /// This route's previous route has changed to the given new route. This is
  /// called on a route whenever the previous route changes for any reason, so
  /// long as it is in the history. `previousRoute` will be null if there's no
  /// previous route.
  @protected
  @mustCallSuper
  void didChangePrevious(Route<dynamic> previousRoute) { }

  /// Called whenever the internal state of the route has changed.
  ///
  /// This should be called whenever [willHandlePopInternally], [didPop],
  /// [offstage], or other internal state of the route changes value. It is used
  /// by [ModalRoute], for example, to report the new information via its
  /// inherited widget to any children of the route.
  ///
  /// See also:
  ///
  ///  * [changedExternalState], which is called when the [Navigator] rebuilds.
  @protected
  @mustCallSuper
  void changedInternalState() { }

  /// Called whenever the [Navigator] has its widget rebuilt, to indicate that
  /// the route may wish to rebuild as well.
  ///
  /// This is called by the [Navigator] whenever the [NavigatorState]'s
  /// [widget] changes, for example because the [MaterialApp] has been rebuilt.
  /// This ensures that routes that directly refer to the state of the widget
  /// that built the [MaterialApp] will be notified when that widget rebuilds,
  /// since it would otherwise be difficult to notify the routes that state they
  /// depend on may have changed.
  ///
  /// See also:
  ///
  ///  * [changedInternalState], the equivalent but for changes to the internal
  ///    state of the route.
  @protected
  @mustCallSuper
  void changedExternalState() { }

  /// The route should remove its overlays and free any other resources.
  ///
  /// This route is no longer referenced by the navigator.
  @mustCallSuper
  @protected
  void dispose() {
    _navigator = null;
  }

  /// Whether this route is the top-most route on the navigator.
  ///
  /// If this is true, then [isActive] is also true.
  bool get isCurrent {
    return _navigator != null && _navigator._history.last == this;
  }

  /// Whether this route is the bottom-most route on the navigator.
  ///
  /// If this is true, then [Navigator.canPop] will return false if this route's
  /// [willHandlePopInternally] returns false.
  ///
  /// If [isFirst] and [isCurrent] are both true then this is the only route on
  /// the navigator (and [isActive] will also be true).
  bool get isFirst {
    return _navigator != null && _navigator._history.first == this;
  }

  /// Whether this route is on the navigator.
  ///
  /// If the route is not only active, but also the current route (the top-most
  /// route), then [isCurrent] will also be true. If it is the first route (the
  /// bottom-most route), then [isFirst] will also be true.
  ///
  /// If a higher route is entirely opaque, then the route will be active but not
  /// rendered. It is even possible for the route to be active but for the stateful
  /// widgets within the route to not be instantiated. See [ModalRoute.maintainState].
  bool get isActive {
    return _navigator != null && _navigator._history.contains(this);
  }
}
```



## NavigatorObserver

```dart
/// 用于观测Navigator行为的接口
class NavigatorObserver {
  NavigatorState get navigator => _navigator;
  NavigatorState _navigator;

  /// The [Navigator] pushed `route`.
  ///
  /// The route immediately below that one, and thus the previously active
  /// route, is `previousRoute`.
  void didPush(Route<dynamic> route, Route<dynamic> previousRoute) { }

  /// The [Navigator] popped `route`.
  ///
  /// The route immediately below that one, and thus the newly active
  /// route, is `previousRoute`.
  void didPop(Route<dynamic> route, Route<dynamic> previousRoute) { }

  /// The [Navigator] removed `route`.
  ///
  /// If only one route is being removed, then the route immediately below
  /// that one, if any, is `previousRoute`.
  ///
  /// If multiple routes are being removed, then the route below the
  /// bottommost route being removed, if any, is `previousRoute`, and this
  /// method will be called once for each removed route, from the topmost route
  /// to the bottommost route.
  void didRemove(Route<dynamic> route, Route<dynamic> previousRoute) { }

  /// The [Navigator] replaced `oldRoute` with `newRoute`.
  void didReplace({ Route<dynamic> newRoute, Route<dynamic> oldRoute }) { }

  /// The [Navigator]'s route `route` is being moved by a user gesture.
  ///
  /// For example, this is called when an iOS back gesture starts.
  ///
  /// Paired with a call to [didStopUserGesture] when the route is no longer
  /// being manipulated via user gesture.
  ///
  /// If present, the route immediately below `route` is `previousRoute`.
  /// Though the gesture may not necessarily conclude at `previousRoute` if
  /// the gesture is canceled. In that case, [didStopUserGesture] is still
  /// called but a follow-up [didPop] is not.
  void didStartUserGesture(Route<dynamic> route, Route<dynamic> previousRoute) { }

  /// User gesture is no longer controlling the [Navigator].
  ///
  /// Paired with an earlier call to [didStartUserGesture].
  void didStopUserGesture() { }
}
```


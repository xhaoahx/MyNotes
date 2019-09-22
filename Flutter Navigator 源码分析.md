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

  ///  RoutePopDisposition.bubble 延迟给系统导航处理
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

  @mustCallSuper
  @protected
  void dispose() {
    _navigator = null;
  }

  /// 是否是最上层的路由，即_navigator._history的最后一个
  bool get isCurrent {
    return _navigator != null && _navigator._history.last == this;
  }

  /// 是否是最下层的路由，即_navigator._history的第一个
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

  /// [Navigator] 在栈顶插入一个 route
  ///
  /// 这会使得上一个 route 立即变成 previousRoute
  void didPush(Route<dynamic> route, Route<dynamic> previousRoute) { }

  /// [Navigator] 移除栈顶一个 route
  void didPop(Route<dynamic> route, Route<dynamic> previousRoute) { }

  /// [Navigator] 移除栈顶一个 route
  void didRemove(Route<dynamic> route, Route<dynamic> previousRoute) { }

  /// [Navigator] 替换一个 route 
  void didReplace({ Route<dynamic> newRoute, Route<dynamic> oldRoute }) { }

  /// [Navigator] 正在通过手势移除一个 route
  void didStartUserGesture(Route<dynamic> route, Route<dynamic> previousRoute) { }

  /// 手势不再控制 [Navigator].
  void didStopUserGesture() { }
}
```



## Navigator

```dart
class Navigator extends StatefulWidget {
  const Navigator({
    Key key,
    this.initialRoute,
    @required this.onGenerateRoute,
    this.onUnknownRoute,
    this.observers = const <NavigatorObserver>[],
  }) : assert(onGenerateRoute != null),
       super(key: key);

  final String initialRoute;

  final RouteFactory onGenerateRoute;

  final RouteFactory onUnknownRoute;

  /// 这个 navigator 的观测者
  final List<NavigatorObserver> observers;

  static const String defaultRouteName = '/';

  /// 以下方法是调用了 Navigator.of(context) 获取到的 NavigatorState 的方法	
  static Future<T> pushNamed<T extends Object>(
    BuildContext context,
    String routeName, {
    Object arguments,
   }) {
    return Navigator.of(context).pushNamed<T>(routeName, arguments: arguments);
  }

  static Future<T> pushReplacementNamed<T extends Object, TO extends Object>(
    BuildContext context,
    String routeName, {
    TO result,
    Object arguments,
  }) {
    return Navigator.of(context).
        pushReplacementNamed<T, TO>(routeName, arguments: arguments, result: result);
  }

  static Future<T> popAndPushNamed<T extends Object, TO extends Object>(
    BuildContext context,
    String routeName, {
    TO result,
    Object arguments,
   }) {
    return Navigator.of(context).
        popAndPushNamed<T, TO>(routeName, arguments: arguments, result: result);
  }

  static Future<T> pushNamedAndRemoveUntil<T extends Object>(
    BuildContext context,
    String newRouteName,
    RoutePredicate predicate, {
    Object arguments,
  }) {
    return Navigator.of(context).
        pushNamedAndRemoveUntil<T>(newRouteName, predicate, arguments: arguments);
  }

  @optionalTypeArgs
  static Future<T> push<T extends Object>(BuildContext context, Route<T> route) {
    return Navigator.of(context).push(route);
  }

  static Future<T> pushReplacement<T extends Object, TO extends Object>(
      BuildContext context, Route<T> newRoute, { TO result }) 
  {
    return Navigator.of(context).
        pushReplacement<T, TO>(newRoute, result: result);
  }

  @optionalTypeArgs
  static Future<T> pushAndRemoveUntil<T extends Object>(
      BuildContext context, Route<T> newRoute, RoutePredicate predicate) 
  {
    return Navigator.of(context).pushAndRemoveUntil<T>(newRoute, predicate);
  }

 
  static void replace<T extends Object>(BuildContext context, { 
      @required Route<dynamic> oldRoute, 
      @required Route<T> newRoute }) 
  {
    return Navigator.of(context).replace<T>(oldRoute: oldRoute, newRoute: newRoute);
  }

  static void replaceRouteBelow<T extends Object>(
      BuildContext context, { 
          @required Route<dynamic> anchorRoute, 
          Route<T> newRoute }) 
  {
    return Navigator.of(context).
        replaceRouteBelow<T>(anchorRoute: anchorRoute, newRoute: newRoute);
  }


  static bool canPop(BuildContext context) {
    final NavigatorState navigator = Navigator.of(context, nullOk: true);
    return navigator != null && navigator.canPop();
  }

  static Future<bool> maybePop<T extends Object>(
      BuildContext context, [ T result ]) 
  {
    return Navigator.of(context).maybePop<T>(result);
  }

  static bool pop<T extends Object>(BuildContext context, [ T result ]) {
    return Navigator.of(context).pop<T>(result);
  }


  static void popUntil(BuildContext context, RoutePredicate predicate) {
    Navigator.of(context).popUntil(predicate);
  }

  static void removeRoute(BuildContext context, Route<dynamic> route) {
    return Navigator.of(context).removeRoute(route);
  }

  static void removeRouteBelow(BuildContext context, Route<dynamic> anchorRoute) {
    return Navigator.of(context).removeRouteBelow(anchorRoute);
  }

  /// 获取到最近的 NavigatorState  
  static NavigatorState of(
    BuildContext context, {
    bool rootNavigator = false,
    bool nullOk = false,
  }) {
    final NavigatorState navigator = rootNavigator
        ? context.rootAncestorStateOfType(const TypeMatcher<NavigatorState>())
        : context.ancestorStateOfType(const TypeMatcher<NavigatorState>());
    return navigator;
  }

  @override
  NavigatorState createState() => NavigatorState();
}
```



## NavigatorState

```dart
class NavigatorState 
    extends State<Navigator> 
    with TickerProviderStateMixin 
{
  final GlobalKey<OverlayState> _overlayKey = GlobalKey<OverlayState>();
  final List<Route<dynamic>> _history = <Route<dynamic>>[];
  final Set<Route<dynamic>> _poppedRoutes = <Route<dynamic>>{};

  /// The [FocusScopeNode] for the [FocusScope] that encloses the routes.
  final FocusScopeNode focusScopeNode = FocusScopeNode(debugLabel: 'Navigator Scope');

  final List<OverlayEntry> _initialOverlayEntries = <OverlayEntry>[];

  @override
  void initState() {
    super.initState();
    for (NavigatorObserver observer in widget.observers) {
      observer._navigator = this;
    }
    String initialRouteName = widget.initialRoute ?? Navigator.defaultRouteName;
    if (initialRouteName.startsWith('/') && initialRouteName.length > 1) {
      initialRouteName = initialRouteName.substring(1); // strip leading '/'
      final List<Route<dynamic>> plannedInitialRoutes = <Route<dynamic>>[
        _routeNamed<dynamic>(
            Navigator.defaultRouteName, 
            allowNull: true, 
            arguments: null
        ),
      ];
      final List<String> routeParts = initialRouteName.split('/');
      if (initialRouteName.isNotEmpty) {
        String routeName = '';
        for (String part in routeParts) {
          routeName += '/$part';
          plannedInitialRoutes.add(
              _routeNamed<dynamic>(routeName, allowNull: true, arguments: null)
          );
        }
      }
      if (plannedInitialRoutes.last == null) {
        push(_routeNamed<Object>(Navigator.defaultRouteName, arguments: null));
      } else {
        plannedInitialRoutes.where(
            (Route<dynamic> route) => route != null
        ).forEach(push);
      }
    } else {
      Route<Object> route;
      if (initialRouteName != Navigator.defaultRouteName)
        route = _routeNamed<Object>(initialRouteName, allowNull: true, arguments: null);
      route ??= _routeNamed<Object>(Navigator.defaultRouteName, arguments: null);
      push(route);
    }
    for (Route<dynamic> route in _history)
      _initialOverlayEntries.addAll(route.overlayEntries);
  }

  @override
  void didUpdateWidget(Navigator oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (oldWidget.observers != widget.observers) {
      for (NavigatorObserver observer in oldWidget.observers)
        observer._navigator = null;
      for (NavigatorObserver observer in widget.observers) {
        observer._navigator = this;
      }
    }
    for (Route<dynamic> route in _history)
      route.changedExternalState();
  }

  @override
  void dispose() {
    for (NavigatorObserver observer in widget.observers)
      observer._navigator = null;
    final List<Route<dynamic>> doomed = _poppedRoutes.toList()..addAll(_history);
    for (Route<dynamic> route in doomed)
      route.dispose();
    _poppedRoutes.clear();
    _history.clear();
    focusScopeNode.dispose();
    super.dispose();
  }

  /// navigator 用于视觉呈现的 覆盖层
  OverlayState get overlay => _overlayKey.currentState;

  OverlayEntry get _currentOverlayEntry {
    for (Route<dynamic> route in _history.reversed) {
      if (route.overlayEntries.isNotEmpty)
        return route.overlayEntries.last;
    }
    return null;
  }

  bool _debugLocked = false; // used to prevent re-entrant calls to push, pop, and friends

  Route<T> _routeNamed<T>(String name, {
      @required Object arguments, bool allowNull = false })
  {
    final RouteSettings settings = RouteSettings(
      name: name,
      isInitialRoute: _history.isEmpty,
      arguments: arguments,
    );
    Route<T> route = widget.onGenerateRoute(settings);
    if (route == null && !allowNull) {
      route = widget.onUnknownRoute(settings);
    }
    return route;
  }

  /// 推入命名路由
  Future<T> pushNamed<T extends Object>(
    String routeName, {
    Object arguments,
  }) {
    return push<T>(_routeNamed<T>(routeName, arguments: arguments));
  }

  /// 替换命名路由
  Future<T> pushReplacementNamed<T extends Object, TO extends Object>(
    String routeName, {
    TO result,
    Object arguments,
  }) {
    return pushReplacement<T, TO>(
        _routeNamed<T>(routeName, arguments: arguments), result: result);
  }

  /// 
  Future<T> popAndPushNamed<T extends Object, TO extends Object>(
    String routeName, {
    TO result,
    Object arguments,
  }) {
    pop<TO>(result);
    return pushNamed<T>(routeName, arguments: arguments);
  }

  /// Push the route with the given name onto the navigator, and then remove all
  /// the previous routes until the `predicate` returns true.
  ///
  /// {@macro flutter.widgets.navigator.pushNamedAndRemoveUntil}
  ///
  /// {@macro flutter.widgets.navigator.pushNamed.arguments}
  ///
  /// {@tool sample}
  ///
  /// Typical usage is as follows:
  ///
  /// ```dart
  /// void _handleOpenCalendar() {
  ///   navigator.pushNamedAndRemoveUntil('/calendar', ModalRoute.withName('/'));
  /// }
  /// ```
  /// {@end-tool}
  @optionalTypeArgs
  Future<T> pushNamedAndRemoveUntil<T extends Object>(
    String newRouteName,
    RoutePredicate predicate, {
    Object arguments,
  }) {
    return pushAndRemoveUntil<T>(_routeNamed<T>(newRouteName, arguments: arguments), predicate);
  }

  /// 在栈顶推入指定的路由
  @optionalTypeArgs
  Future<T> push<T extends Object>(Route<T> route) {
    final Route<dynamic> oldRoute = _history.isNotEmpty ? _history.last : null;
    route._navigator = this;
    route.install(_currentOverlayEntry);
    _history.add(route);
    route.didPush();
    route.didChangeNext(null);
    if (oldRoute != null) {
      oldRoute.didChangeNext(route);
      route.didChangePrevious(oldRoute);
    }
    for (NavigatorObserver observer in widget.observers)
      observer.didPush(route, oldRoute);
    RouteNotificationMessages.maybeNotifyRouteChange(
        _routePushedMethod, route, oldRoute);
    _afterNavigation(route);
    return route.popped;
  }

  void _afterNavigation<T>(Route<T> route) {
    _cancelActivePointers();
  }

  Future<T> pushReplacement<T extends Object, TO extends Object>(
      Route<T> newRoute, { TO result }) 
  {
    final Route<dynamic> oldRoute = _history.last;
    final int index = _history.length - 1;
    newRoute._navigator = this;
    newRoute.install(_currentOverlayEntry);
    _history[index] = newRoute;
    newRoute.didPush().whenCompleteOrCancel(() {
      if (mounted) {
        oldRoute
          ..didComplete(result ?? oldRoute.currentResult)
          ..dispose();
      }
    });
    newRoute.didChangeNext(null);
    if (index > 0) {
      _history[index - 1].didChangeNext(newRoute);
      newRoute.didChangePrevious(_history[index - 1]);
    }
    for (NavigatorObserver observer in widget.observers)
      observer.didReplace(newRoute: newRoute, oldRoute: oldRoute);
    RouteNotificationMessages.
        maybeNotifyRouteChange(_routeReplacedMethod, newRoute, oldRoute);
    _afterNavigation(newRoute);
    return newRoute.popped;
  }

    
  @optionalTypeArgs
  Future<T> pushAndRemoveUntil<T extends Object>(
      Route<T> newRoute, RoutePredicate predicate) 
  {
    // The route that is being pushed on top of
    final Route<dynamic> precedingRoute = _history.isNotEmpty ? _history.last : null;
    final OverlayEntry precedingRouteOverlay = _currentOverlayEntry;

    // Routes to remove
    final List<Route<dynamic>> removedRoutes = <Route<dynamic>>[];
    while (_history.isNotEmpty && !predicate(_history.last)) {
      final Route<dynamic> removedRoute = _history.removeLast();
      removedRoutes.add(removedRoute);
    }

    // Push new route
    final Route<dynamic> newPrecedingRoute = _history.isNotEmpty ? _history.last : null;
    newRoute._navigator = this;
    newRoute.install(precedingRouteOverlay);
    _history.add(newRoute);

    newRoute.didPush().whenCompleteOrCancel(() {
      if (mounted) {
        for (Route<dynamic> removedRoute in removedRoutes) {
          for (NavigatorObserver observer in widget.observers)
            observer.didRemove(removedRoute, newPrecedingRoute);
          removedRoute.dispose();
        }

        if (newPrecedingRoute != null)
          newPrecedingRoute.didChangeNext(newRoute);
      }
    });

    // Notify for newRoute
    newRoute.didChangeNext(null);
    for (NavigatorObserver observer in widget.observers)
      observer.didPush(newRoute, precedingRoute);

    _afterNavigation(newRoute);
    return newRoute.popped;
  }

  @optionalTypeArgs
  void replace<T extends Object>({ 
      @required Route<dynamic> oldRoute,
      @required Route<T> newRoute }) 
  {
    if (oldRoute == newRoute) return;
      
    final int index = _history.indexOf(oldRoute);
    assert(index >= 0);
    newRoute._navigator = this;
    newRoute.install(oldRoute.overlayEntries.last);
    _history[index] = newRoute;
    newRoute.didReplace(oldRoute);
    if (index + 1 < _history.length) {
      newRoute.didChangeNext(_history[index + 1]);
      _history[index + 1].didChangePrevious(newRoute);
    } else {
      newRoute.didChangeNext(null);
    }
    if (index > 0) {
      _history[index - 1].didChangeNext(newRoute);
      newRoute.didChangePrevious(_history[index - 1]);
    }
    for (NavigatorObserver observer in widget.observers)
      observer.didReplace(newRoute: newRoute, oldRoute: oldRoute);
    RouteNotificationMessages.maybeNotifyRouteChange(_routeReplacedMethod, newRoute, oldRoute);
    oldRoute.dispose();
    assert(() { _debugLocked = false; return true; }());
  }

 
  void replaceRouteBelow<T extends Object>({ @required Route<dynamic> anchorRoute, Route<T> newRoute }) {
    replace<T>(oldRoute: _history[_history.indexOf(anchorRoute) - 1], 
               newRoute: newRoute);
  }

  bool canPop() {
    assert(_history.isNotEmpty);
    return _history.length > 1 || _history[0].willHandlePopInternally;
  }

  /// Tries to pop the current route, while honoring the route's [Route.willPop]
  /// state.
  ///
  /// {@macro flutter.widgets.navigator.maybePop}
  ///
  /// See also:
  ///
  ///  * [Form], which provides an `onWillPop` callback that enables the form
  ///    to veto a [pop] initiated by the app's back button.
  ///  * [ModalRoute], which provides a `scopedWillPopCallback` that can be used
  ///    to define the route's `willPop` method.
  @optionalTypeArgs
  Future<bool> maybePop<T extends Object>([ T result ]) async {
    final Route<T> route = _history.last;
    final RoutePopDisposition disposition = await route.willPop();
    if (disposition != RoutePopDisposition.bubble && mounted) {
      if (disposition == RoutePopDisposition.pop)
        pop(result);
      return true;
    }
    return false;
  }


  bool pop<T extends Object>([ T result ]) {
   
    final Route<dynamic> route = _history.last;
    
    bool debugPredictedWouldPop;
    
    if (route.didPop(result ?? route.currentResult)) {
     
      if (_history.length > 1) {
        _history.removeLast();
        // If route._navigator is null, the route called finalizeRoute from
        // didPop, which means the route has already been disposed and doesn't
        // need to be added to _poppedRoutes for later disposal.
        if (route._navigator != null)
          _poppedRoutes.add(route);
        _history.last.didPopNext(route);
        for (NavigatorObserver observer in widget.observers)
          observer.didPop(route, _history.last);
        RouteNotificationMessages.maybeNotifyRouteChange(
            _routePoppedMethod, route, _history.last);
      } else {
        assert(() { _debugLocked = false; return true; }());
        return false;
      }
    } else {
        
    }
      
    _afterNavigation<dynamic>(route);
    return true;
  }

  void popUntil(RoutePredicate predicate) {
    while (!predicate(_history.last))
      pop();
  }


  void removeRoute(Route<dynamic> route) {
    final int index = _history.indexOf(route);
    assert(index != -1);
    final Route<dynamic> previousRoute = index > 0 ? _history[index - 1] : null;
    final Route<dynamic> nextRoute = 
        (index + 1 < _history.length) ? _history[index + 1] : null;
    _history.removeAt(index);
    previousRoute?.didChangeNext(nextRoute);
    nextRoute?.didChangePrevious(previousRoute);
    for (NavigatorObserver observer in widget.observers)
      observer.didRemove(route, previousRoute);
    route.dispose();
    _afterNavigation<dynamic>(nextRoute);
  }

 
  void removeRouteBelow(Route<dynamic> anchorRoute) {
   
    final int index = _history.indexOf(anchorRoute) - 1;
    final Route<dynamic> targetRoute = _history[index];
  
    _history.removeAt(index);
    final Route<dynamic> nextRoute = index < _history.length ? _history[index] : null;
    final Route<dynamic> previousRoute = index > 0 ? _history[index - 1] : null;
    if (previousRoute != null)
      previousRoute.didChangeNext(nextRoute);
    if (nextRoute != null)
      nextRoute.didChangePrevious(previousRoute);
    targetRoute.dispose();
  }

  /// Complete the lifecycle for a route that has been popped off the navigator.
  ///
  /// When the navigator pops a route, the navigator retains a reference to the
  /// route in order to call [Route.dispose] if the navigator itself is removed
  /// from the tree. When the route is finished with any exit animation, the
  /// route should call this function to complete its lifecycle (e.g., to
  /// receive a call to [Route.dispose]).
  ///
  /// The given `route` must have already received a call to [Route.didPop].
  /// This function may be called directly from [Route.didPop] if [Route.didPop]
  /// will return true.
  void finalizeRoute(Route<dynamic> route) {
    _poppedRoutes.remove(route);
    route.dispose();
  }

  /// Whether a route is currently being manipulated by the user, e.g.
  /// as during an iOS back gesture.
  bool get userGestureInProgress => _userGesturesInProgress > 0;
  int _userGesturesInProgress = 0;

  void didStartUserGesture() {
    _userGesturesInProgress += 1;
    if (_userGesturesInProgress == 1) {
      final Route<dynamic> route = _history.last;
      final Route<dynamic> previousRoute = !route.willHandlePopInternally && _history.length > 1
          ? _history[_history.length - 2]
          : null;
      // Don't operate the _history list since the gesture may be canceled.
      // In case of a back swipe, the gesture controller will call .pop() itself.

      for (NavigatorObserver observer in widget.observers)
        observer.didStartUserGesture(route, previousRoute);
    }
  }

  void didStopUserGesture() {
    assert(_userGesturesInProgress > 0);
    _userGesturesInProgress -= 1;
    if (_userGesturesInProgress == 0) {
      for (NavigatorObserver observer in widget.observers)
        observer.didStopUserGesture();
    }
  }

  final Set<int> _activePointers = <int>{};

  void _handlePointerDown(PointerDownEvent event) {
    _activePointers.add(event.pointer);
  }

  void _handlePointerUpOrCancel(PointerEvent event) {
    _activePointers.remove(event.pointer);
  }

  void _cancelActivePointers() {
    // TODO(abarth): This mechanism is far from perfect. See https://github.com/flutter/flutter/issues/4770
    if (SchedulerBinding.instance.schedulerPhase == SchedulerPhase.idle) {
      // If we're between frames (SchedulerPhase.idle) then absorb any
      // subsequent pointers from this frame. The absorbing flag will be
      // reset in the next frame, see build().
      final RenderAbsorbPointer absorber = _overlayKey.currentContext?.ancestorRenderObjectOfType(const TypeMatcher<RenderAbsorbPointer>());
      setState(() {
        absorber?.absorbing = true;
        // We do this in setState so that we'll reset the absorbing value back
        // to false on the next frame.
      });
    }
    _activePointers.toList().forEach(WidgetsBinding.instance.cancelPointer);
  }

  @override
  Widget build(BuildContext context) {
    return Listener(
      onPointerDown: _handlePointerDown,
      onPointerUp: _handlePointerUpOrCancel,
      onPointerCancel: _handlePointerUpOrCancel,
      child: AbsorbPointer(
        absorbing: false, // it's mutated directly by _cancelActivePointers above
        child: FocusScope(
          node: focusScopeNode,
          autofocus: true,
          child: Overlay(
            key: _overlayKey,
            initialEntries: _initialOverlayEntries,
          ),
        ),
      ),
    );
  }
}
```


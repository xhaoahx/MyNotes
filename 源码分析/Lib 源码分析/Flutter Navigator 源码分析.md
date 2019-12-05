Flutter Navigator 源码分析

[TOC]

## RouteSetting

```dart
/// 路由配置
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

  /// 是否是初始路由（第一个加入[Navigator]的路由）
  ///
  /// 这个路由会跳过所有的动画效果，用于加速加载
  final bool isInitialRoute;

  /// 路由跳转参数
  final Object arguments;

  @override
  String toString() => '$runtimeType("$name", $arguments)';
}
```



## Route

```dart
/// 抽象路由类
abstract class Route<T> {
  /// 初始化路由
  Route({ RouteSettings settings }) : settings = settings ?? const RouteSettings();

  /// 持有这个路由的 Navigator
  NavigatorState get navigator => _navigator;
  NavigatorState _navigator;

  /// 配置
  final RouteSettings settings;

  /// 路由覆盖层（注意在ModelRoute重载的createOverlayEntries，和在OverlayRoute调用到    
  /// _overlayEntries.addAll(createOverlayEntries())，它们产生了页面）  
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

  /// 请求弹出出当前路由
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

  /// 下一个路由被弹出时的调用
  @protected
  @mustCallSuper
  void didPopNext(Route<dynamic> nextRoute) { }

  /// 下一个路由被改变时的回调
  @protected
  @mustCallSuper
  void didChangeNext(Route<dynamic> nextRoute) { }

  /// 路由改变时，对前一个路由的回调
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

  /// 是否是Active  
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

  /// RouteFactory = Route<dynamic> Function(RouteSettings settings) 
  /// 用给定的RouteSetting生成以一个Route
  /// 通常是WidgetApp指定的:
  /// _onGenerateRoute(RouteSettings settings) {
  ///  final String name = settings.name;
  ///  final WidgetBuilder pageContentBuilder = 
  ///   name == Navigator.defaultRouteName && widget.home != null
  ///   ? (BuildContext context) => widget.home
  ///   : widget.routes[name];
  ///  
  ///  if (pageContentBuilder != null) {
  ///    final Route<dynamic> route = widget.pageRouteBuilder<dynamic>(
  ///      settings,
  ///      pageContentBuilder,
  ///    );
  ///     return route;
  ///  }
  ///  if (widget.onGenerateRoute != null)
  ///    return widget.onGenerateRoute(settings);
  ///  return null;
  /// }
  final RouteFactory onGenerateRoute;

  final RouteFactory onUnknownRoute;

  /// 这个 navigator 的观测者,当navigator发生改变时通知观测者
  final List<NavigatorObserver> observers;

  static const String defaultRouteName = '/';

  /// 以下方法都是是调用了 Navigator.of(context) 获取到的 NavigatorState 的方法	
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
    /// 如果initialRouteName类似于 /xxxx/xxxx/xxx 的话，会逐个push对应的route 
    if (initialRouteName.startsWith('/') && initialRouteName.length > 1) {
      initialRouteName = initialRouteName.substring(1); // strip leading '/'
        
      /// 首先将 defaultRouteName 对应的route加入 plannedInitialRoutes中
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
        /// 逐个加入对应的route  
        for (String part in routeParts) {
          routeName += '/$part';
          plannedInitialRoutes.add(
              _routeNamed<dynamic>(routeName, allowNull: true, arguments: null)
          );
        }
      }
      /// plannedInitialRoutes 为空，则push默认route  
      if (plannedInitialRoutes.last == null) {
        push(_routeNamed<Object>(Navigator.defaultRouteName, arguments: null));
      /// 否则 push 所有不为 null 的路由 （可能是因为没有对应的routeNames无法产生route   
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
    /// 在_initialOverlayEntries 中 加入每个 route 的 overlayEntries 
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

  /// 当前覆盖层
  /// 逆转_history之后，返回第一个route（push的最后一个route）的最后一层覆盖层  
  OverlayEntry get _currentOverlayEntry {
    for (Route<dynamic> route in _history.reversed) {
      if (route.overlayEntries.isNotEmpty)
        return route.overlayEntries.last;
    }
    return null;
  }

  bool _debugLocked = false;
      
  /// 配置 RouteSettings ，调用 onGenerateRoute（setting）来产生一个新的 route
  Route<T> _routeNamed<T>(String name, {
      @required Object arguments, 
      bool allowNull = false })
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

  /// 退出一个然后加入一个命名路由
  Future<T> popAndPushNamed<T extends Object, TO extends Object>(
    String routeName, {
    TO result,
    Object arguments,
  }) {
    pop<TO>(result);
    return pushNamed<T>(routeName, arguments: arguments);
  }

  /// 推入一个路由并且移除一些路由
  @optionalTypeArgs
  Future<T> pushNamedAndRemoveUntil<T extends Object>(
    String newRouteName,
    RoutePredicate predicate, {
    Object arguments,
  }) {
    return pushAndRemoveUntil<T>(_routeNamed<T>(newRouteName, arguments: arguments), predicate);
  }

  /// 在栈顶推入指定的路由
  /// 调用顺序：
  /// route.install(_currentOverlayEntry) // 为route创建了overlayEntries
  /// 			|
  /// _history.add(route); 
  ///			|
  /// route.didPush();// TransitionRoute在这里会启用动画 
  ///			|
  /// route.didChangeNext(null);
  /// 			|(oldRoute 为先前的一个路由)
  /// oldRoute.didChangeNext(route);
  /// 			|
  /// route.didChangePrevious(oldRoute); 
  ///			|
  /// for(NavigatorObserver observer in widget.observers) 
  ///      observer.didPush(route, oldRoute);
  /// 			|
  /// _afterNavigation(route);
    
  @optionalTypeArgs
  Future<T> push<T extends Object>(Route<T> route) {
    final Route<dynamic> oldRoute = _history.isNotEmpty ? _history.last : null;
    route._navigator = this;
    /// 插入到第一个route的最后一个overlayEntry后面  
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

  /// 移除栈顶的路由
  /// 调用顺序：
  /// route.didPop(result ?? route.currentResult)
  /// 			|
  /// _history.removeLast();
  ///			|
  /// _poppedRoutes.add(route);
  ///			|
  /// observer.didPop(route, _history.last);
  /// 			|
  /// _afterNavigation<dynamic>(route);
    
  bool pop<T extends Object>([ T result ]) {
   
    final Route<dynamic> route = _history.last;
    
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

  /// 移除一个路由
    
  void removeRoute(Route<dynamic> route) {
    final int index = _history.indexOf(route);
    /// 这个路由必须存在于 _history 列表中
    assert(index != -1);
    final Route<dynamic> previousRoute = 
        index > 0 ? _history[index - 1] : null;
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

  /// 终结一个路由
  /// 在 pop 时，将其加入 _poppedRoutes，日后可以恢复，
  /// finalizeRoute 会完全销毁 route，不可恢复  
  void finalizeRoute(Route<dynamic> route) {
    _poppedRoutes.remove(route);
    route.dispose();
  }

  /// 掠过几个手势控制  
  ...

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
          /// 这里为子树创建了一个最高层 Overlay  
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



### -------------------------- 分割线 ------------------------------



### OverlayRoute

```dart
/// 一个可以创建覆盖层的Route
abstract class OverlayRoute<T> extends Route<T> {
  OverlayRoute({
    RouteSettings settings,
  }) : super(settings: settings);

  Iterable<OverlayEntry> createOverlayEntries();

  @override
  List<OverlayEntry> get overlayEntries => _overlayEntries;
  final List<OverlayEntry> _overlayEntries = <OverlayEntry>[];

  @override
  void install(OverlayEntry insertionPoint) {
    assert(_overlayEntries.isEmpty);
    _overlayEntries.addAll(createOverlayEntries());
    navigator.overlay?.insertAll(_overlayEntries, above: insertionPoint);
    super.install(insertionPoint);
  }

  /// 控制了 [didPop] 是否要调用 [NavigatorState.finalizeRoute].
  ///
  /// 如果为 true ，那么调用 [didPop] 时会移除覆盖层
  /// 子类可以重载这个这个值。但是必须要调用 [finishedWhenPopped]
  @protected
  bool get finishedWhenPopped => true;

  @override
  bool didPop(T result) {
    final bool returnValue = super.didPop(result);
    if (finishedWhenPopped)
      navigator.finalizeRoute(this);
    return returnValue;
  }

  @override
  void dispose() {
    for (OverlayEntry entry in _overlayEntries)
      entry.remove();
    _overlayEntries.clear();
    super.dispose();
  }
}
```



### TransitionRoute

```dart
abstract class TransitionRoute<T> extends OverlayRoute<T> {
  /// 当 Pop 或 Push 时，有动画效果过渡
  TransitionRoute({
    RouteSettings settings,
  }) : super(settings: settings);

  /// 在 push 或 pop 各完成一次
  Future<T> get completed => _transitionCompleter.future;
  final Completer<T> _transitionCompleter = Completer<T>();

  Duration get transitionDuration;

  /// 如果设置为 true ，那么当 Transition 完成时，这个 route 之前的 route 就不会被建造，以节省资源
  bool get opaque;

  @override
  bool get finishedWhenPopped => _controller.status == AnimationStatus.dismissed;

  Animation<double> get animation => _animation;
  Animation<double> _animation;

  @protected
  AnimationController get controller => _controller;
  AnimationController _controller;

  Animation<double> get secondaryAnimation => _secondaryAnimation;
  final ProxyAnimation _secondaryAnimation = ProxyAnimation(kAlwaysDismissedAnimation);

  AnimationController createAnimationController() {
    final Duration duration = transitionDuration;
    return AnimationController(
      duration: duration,
      debugLabel: debugLabel,
      /// 注意：
      /// navigator mixin 了一个 TickerProvider  
      vsync: navigator,
    );
  }

  Animation<double> createAnimation() {
    /// Animation<double> get view => this; 
    /// 将 _controller 自身向外暴露  
    return _controller.view;
  }

  T _result;

  /// 但动画状态改变时，触发回调  
  void _handleStatusChanged(AnimationStatus status) {
    switch (status) {
      /// 如果当动画完成时，把overlayEntries的第一个OverlayEntry的opaque设置为true
      /// 这样会使得先前路由被遮蔽，在 OverlayState 会把先前的路由从_onstage里移除
      /// 先前的route不再会被build      
      case AnimationStatus.completed:
        if (overlayEntries.isNotEmpty)
          overlayEntries.first.opaque = opaque;
        break;
      case AnimationStatus.forward:
      case AnimationStatus.reverse:
        if (overlayEntries.isNotEmpty)
          overlayEntries.first.opaque = false;
        break;
      case AnimationStatus.dismissed:
        if (!isActive) {
          navigator.finalizeRoute(this);
        }
        break;
    }
    /// 这个方法在ModelRoute里被重写
    /// 调用了_modalBarrier.markNeedsBuild();  
    changedInternalState();
  }

  /// 创建了AnimationController   
  @override
  void install(OverlayEntry insertionPoint) {
    _controller = createAnimationController();
    _animation = createAnimation();
    super.install(insertionPoint);
  }

  /// 开始动画  
  @override
  TickerFuture didPush() {
    _animation.addStatusListener(_handleStatusChanged);
    return _controller.forward();
  }

  @override
  void didReplace(Route<dynamic> oldRoute) {
    if (oldRoute is TransitionRoute)
      _controller.value = oldRoute._controller.value;
    _animation.addStatusListener(_handleStatusChanged);
    super.didReplace(oldRoute);
  }

  @override
  bool didPop(T result) {
    _result = result;
    _controller.reverse();
    return super.didPop(result);
  }

  @override
  void didPopNext(Route<dynamic> nextRoute) {
    _updateSecondaryAnimation(nextRoute);
    super.didPopNext(nextRoute);
  }

  @override
  void didChangeNext(Route<dynamic> nextRoute) {
    _updateSecondaryAnimation(nextRoute);
    super.didChangeNext(nextRoute);
  }

  void _updateSecondaryAnimation(Route<dynamic> nextRoute) {
    if (nextRoute is TransitionRoute<dynamic> 
        && canTransitionTo(nextRoute) 
        && nextRoute.canTransitionFrom(this)) 
    {
      final Animation<double> current = _secondaryAnimation.parent;
      if (current != null) {
        if (current is TrainHoppingAnimation) {
          TrainHoppingAnimation newAnimation;
          newAnimation = TrainHoppingAnimation(
            current.currentTrain,
            nextRoute._animation,
            onSwitchedTrain: () {
              assert(_secondaryAnimation.parent == newAnimation);
              assert(newAnimation.currentTrain == nextRoute._animation);
              _secondaryAnimation.parent = newAnimation.currentTrain;
              newAnimation.dispose();
            },
          );
          _secondaryAnimation.parent = newAnimation;
          current.dispose();
        } else {
          _secondaryAnimation.parent = TrainHoppingAnimation(
              current, nextRoute._animation);
        }
      } else {
        _secondaryAnimation.parent = nextRoute._animation;
      }
    } else {
      _secondaryAnimation.parent = kAlwaysDismissedAnimation;
    }
  }
    
  bool canTransitionTo(TransitionRoute<dynamic> nextRoute) => true;

  bool canTransitionFrom(TransitionRoute<dynamic> previousRoute) => true;

  @override
  void dispose() {
    _controller?.dispose();
    _transitionCompleter.complete(_result);
    super.dispose();
  }

}
```



## LocalHistoryEntry

```dart
class LocalHistoryEntry {
  /// Creates an entry in the history of a [LocalHistoryRoute].
  LocalHistoryEntry({ this.onRemove });

  /// 被移除时调用
  final VoidCallback onRemove;

  LocalHistoryRoute<dynamic> _owner;

  /// Remove this entry from the history of its associated [LocalHistoryRoute].
  void remove() {
    _owner.removeLocalHistoryEntry(this);
    assert(_owner == null);
  }

  void _notifyRemoved() {
    if (onRemove != null)
      onRemove();
  }
}
```



### LocalHistoryRoute

```dart
mixin LocalHistoryRoute<T> on Route<T> {
  List<LocalHistoryEntry> _localHistory;

  void addLocalHistoryEntry(LocalHistoryEntry entry) {
    entry._owner = this;
    _localHistory ??= <LocalHistoryEntry>[];
    final bool wasEmpty = _localHistory.isEmpty;
    _localHistory.add(entry);
    if (wasEmpty)
      changedInternalState();
  }

  /// 同步地移除一个当前 route 的历史 route
  void removeLocalHistoryEntry(LocalHistoryEntry entry) {
    _localHistory.remove(entry);
    entry._owner = null;
    entry._notifyRemoved();
    if (_localHistory.isEmpty)
      changedInternalState();
  }

  @override
  Future<RoutePopDisposition> willPop() async {
    if (willHandlePopInternally)
      return RoutePopDisposition.pop;
    return await super.willPop();
  }

  @override
  bool didPop(T result) {
    if (_localHistory != null && _localHistory.isNotEmpty) {
      final LocalHistoryEntry entry = _localHistory.removeLast();
      assert(entry._owner == this);
      entry._owner = null;
      entry._notifyRemoved();
      if (_localHistory.isEmpty)
        changedInternalState();
      return false;
    }
    return super.didPop(result);
  }

  @override
  bool get willHandlePopInternally {
    return _localHistory != null && _localHistory.isNotEmpty;
  }
}
```



### ModalRoute 

```dart
/// 一个可以在执行动画时阻碍与先前路由交互的 Route
abstract class ModalRoute<T> 
    extends TransitionRoute<T> 
    with LocalHistoryRoute<T> 
{
  ModalRoute({
    RouteSettings settings,
  }) : super(settings: settings);

  @optionalTypeArgs
  static ModalRoute<T> of<T extends Object>(BuildContext context) {
    final _ModalScopeStatus widget = 
        context.inheritFromWidgetOfExactType(_ModalScopeStatus);
    return widget?.route;
  }

  @protected
  void setState(VoidCallback fn) {
    if (_scopeKey.currentState != null) {
      _scopeKey.currentState._routeSetState(fn);
    } else {
      // route 当前处于不可见状态，因此不需要setState
      fn();
    }
  }

  static RoutePredicate withName(String name) {
    return (Route<dynamic> route) {
      return !route.willHandlePopInternally
          && route is ModalRoute
          && route.settings.name == name;
    };
  }

  Widget buildPage(
      BuildContext context, 
      Animation<double> animation,
      Animation<double> secondaryAnimation
  );


  Widget buildTransitions(
    BuildContext context,
    Animation<double> animation,
    Animation<double> secondaryAnimation,
    Widget child,
  ) {
    return child;
  }

  @override
  void install(OverlayEntry insertionPoint) {
    super.install(insertionPoint);
    _animationProxy = ProxyAnimation(super.animation);
    _secondaryAnimationProxy = ProxyAnimation(super.secondaryAnimation);
  }

  @override
  TickerFuture didPush() {
    if (_scopeKey.currentState != null) {
      navigator.focusScopeNode.setFirstFocus(_scopeKey.currentState.focusScopeNode);
    }
    return super.didPush();
  }

  /// 是否点击背景 pop 当前路由
  /// 例如：弹出 dialog 时，点击背景 pop dialog  
  bool get barrierDismissible;

  bool get semanticsDismissible => true;

  /// 背景版颜色  
  Color get barrierColor;

  String get barrierLabel;

  /// 当 inactive 时是否要保存状态
  bool get maintainState;

  /// 是否可见（OffStage通过设置最大Size为Size（0,0）来组织布局渲染  
  bool get offstage => _offstage;
  bool _offstage = false;
  set offstage(bool value) {
    if (_offstage == value)
      return;
    setState(() {
      _offstage = value;
    });
    _animationProxy.parent = 
        _offstage ? kAlwaysCompleteAnimation : super.animation;
    _secondaryAnimationProxy.parent = 
        _offstage ? kAlwaysDismissedAnimation : super.secondaryAnimation;
  }

  BuildContext get subtreeContext => _subtreeKey.currentContext;

  @override
  Animation<double> get animation => _animationProxy;
  ProxyAnimation _animationProxy;

  @override
  Animation<double> get secondaryAnimation => _secondaryAnimationProxy;
  ProxyAnimation _secondaryAnimationProxy;

  final List<WillPopCallback> _willPopCallbacks = <WillPopCallback>[];

  @override
  Future<RoutePopDisposition> willPop() async {
    final _ModalScopeState<T> scope = _scopeKey.currentState;
    for (WillPopCallback callback in List<WillPopCallback>.from(_willPopCallbacks)) {
      if (!await callback())
        return RoutePopDisposition.doNotPop;
    }
    return await super.willPop();
  }

  void addScopedWillPopCallback(WillPopCallback callback) {
    _willPopCallbacks.add(callback);
  }

  void removeScopedWillPopCallback(WillPopCallback callback) {
    _willPopCallbacks.remove(callback);
  }

  @protected
  bool get hasScopedWillPopCallback {
    return _willPopCallbacks.isNotEmpty;
  }

  @override
  void didChangePrevious(Route<dynamic> previousRoute) {
    super.didChangePrevious(previousRoute);
    changedInternalState();
  }

  @override
  void changedInternalState() {
    super.changedInternalState();
    _modalBarrier.markNeedsBuild();
  }

  @override
  void changedExternalState() {
    super.changedExternalState();
    if (_scopeKey.currentState != null)
      _scopeKey.currentState._forceRebuildPage();
  }

  bool get canPop => !isFirst || willHandlePopInternally;

  // Internals

  final GlobalKey<_ModalScopeState<T>> _scopeKey = GlobalKey<_ModalScopeState<T>>();
  final GlobalKey _subtreeKey = GlobalKey();
  final PageStorageBucket _storageBucket = PageStorageBucket();

  static final Animatable<double> _easeCurveTween = CurveTween(curve: Curves.ease);

  // one of the builders
  OverlayEntry _modalBarrier;
  
  /// 建造一个遮罩层  
  Widget _buildModalBarrier(BuildContext context) {
    Widget barrier;
    /// 创建遮罩层实体
    if (barrierColor != null && !offstage) { 
        /// 从透明色到指定颜色的渐变
        ColorTween(
          begin: _kTransparent,
          end: barrierColor, // changedInternalState is called if this updates
        ).chain(_easeCurveTween),
      );
      barrier = AnimatedModalBarrier(
        color: color,
        dismissible: barrierDismissible, 
        semanticsLabel: barrierLabel, 
        barrierSemanticsDismissible: semanticsDismissible,
      );
    } else {
      barrier = ModalBarrier(
        dismissible: barrierDismissible, 
        semanticsLabel: barrierLabel, 
        barrierSemanticsDismissible: semanticsDismissible,
      );
    }
    /// 当动画处于开始阶段或者反转阶段的时候，忽视点击
    return IgnorePointer(
      ignoring: animation.status == AnimationStatus.reverse 
        || animation.status == AnimationStatus.dismissed, 
      child: barrier,
    );
  }

  // We cache the part of the modal scope that doesn't change from frame to
  // frame so that we minimize the amount of building that happens.
  Widget _modalScopeCache;

  // 建立页面实体
  Widget _buildModalScope(BuildContext context) {
    return _modalScopeCache ??= _ModalScope<T>(
      key: _scopeKey,
      route: this,
    );
  }

  @override
  Iterable<OverlayEntry> createOverlayEntries() sync* {
    yield _modalBarrier = OverlayEntry(builder: _buildModalBarrier);
    yield OverlayEntry(builder: _buildModalScope, maintainState: maintainState);
  }

}

```



### PageRoute

```dart
/// 替换整个屏幕的 Route
abstract class PageRoute<T> extends ModalRoute<T> {
  PageRoute({
    RouteSettings settings,
    this.fullscreenDialog = false,
  }) : super(settings: settings);

  /// 是否是一个全屏对话框
  /// 在 android 上，返回按钮会变成一个关闭按钮
  /// 在 ios 上，会取消手势控制  
  final bool fullscreenDialog;

  @override
  bool get opaque => true;

  @override
  bool get barrierDismissible => false;

  @override
  bool canTransitionTo(TransitionRoute<dynamic> nextRoute) 
      => nextRoute is PageRoute;

  @override
  bool canTransitionFrom(TransitionRoute<dynamic> previousRoute) 
      => previousRoute is PageRoute;

  @override
  AnimationController createAnimationController() {
    final AnimationController controller = super.createAnimationController();
    /// 如果是第一个页面，设置立刻完成，以提升加载速度  
    if (settings.isInitialRoute)
      controller.value = 1.0;
    return controller;
  }
}
```



### MaterialPageRoute

```dart
/// 平台自适应的 page 切换效果
class MaterialPageRoute<T> extends PageRoute<T> {

  MaterialPageRoute({
    @required this.builder,
    RouteSettings settings,
    /// 默认maintainState为true
    /// 这会使得push另一个页面时，保持上一个页面状态，但也会消耗一定资源
    /// 当页面层数多，且上一页状态无需被储存时，设置false提升性能  
    this.maintainState = true,
    bool fullscreenDialog = false,
  }) : assert(builder != null),
       assert(maintainState != null),
       assert(fullscreenDialog != null),
       assert(opaque),
       super(settings: settings, fullscreenDialog: fullscreenDialog);

  final WidgetBuilder builder;

  @override
  final bool maintainState;

  @override
  Duration get transitionDuration => const Duration(milliseconds: 300);

  @override
  Color get barrierColor => null;

  @override
  String get barrierLabel => null;

  @override
  bool canTransitionFrom(TransitionRoute<dynamic> previousRoute) {
    return previousRoute is MaterialPageRoute || previousRoute is CupertinoPageRoute;
  }

  @override
  bool canTransitionTo(TransitionRoute<dynamic> nextRoute) {
    // Don't perform outgoing animation if the next route is a fullscreen dialog.
    return (nextRoute is MaterialPageRoute && !nextRoute.fullscreenDialog)
        || (nextRoute is CupertinoPageRoute && !nextRoute.fullscreenDialog);
  }

  @override
  Widget buildPage(
    BuildContext context,
    Animation<double> animation,
    Animation<double> secondaryAnimation,
  ) {
    final Widget result = builder(context);
    return Semantics(
      scopesRoute: true,
      explicitChildNodes: true,
      child: result,
    );
  }

  @override
  Widget buildTransitions(
      BuildContext context, 
      Animation<double> animation,
      Animation<double> secondaryAnimation, 
      Widget child) 
  {
    final PageTransitionsTheme theme = Theme.of(context).pageTransitionsTheme;
    return theme.buildTransitions<T>(
        this, 
        context, 
        animation, 
        secondaryAnimation, 
        child
    );
  }

  @override
  String get debugLabel => '${super.debugLabel}(${settings.name})';
}
```



### _ModelScopeStatus

```dart
class _ModalScopeStatus extends InheritedWidget {
  const _ModalScopeStatus({
    Key key,
    @required this.isCurrent,
    @required this.canPop,
    @required this.route,
    @required Widget child,
  }) : assert(isCurrent != null),
       assert(canPop != null),
       assert(route != null),
       assert(child != null),
       super(key: key, child: child);

  /// 是否是current route  
  final bool isCurrent;
  /// 能否弹出  
  final bool canPop;
  /// 持有的route  
  final Route<dynamic> route;

  @override
  bool updateShouldNotify(_ModalScopeStatus old) {
    return isCurrent != old.isCurrent ||
           canPop != old.canPop ||
           route != old.route;
  }
}
```



### _ModelScope

```dart
class _ModalScope<T> extends StatefulWidget {
  const _ModalScope({
    Key key,
    this.route,
  }) : super(key: key);

  final ModalRoute<T> route;

  @override
  _ModalScopeState<T> createState() => _ModalScopeState<T>();
}
```



### _ModelScopeState

```dart
class _ModalScopeState<T> extends State<_ModalScope<T>> {
  Widget _page;

  /// 这个listenable合并了一个route的两个animation（animation 和 secondaryAnimation）
  Listenable _listenable;

  /// The node this scope will use for its root [FocusScope] widget.
  final FocusScopeNode focusScopeNode = 
      FocusScopeNode(debugLabel: '$_ModalScopeState Focus Scope');

  @override
  void initState() {
    super.initState();
    final List<Listenable> animations = <Listenable>[];
    if (widget.route.animation != null)
      animations.add(widget.route.animation);
    if (widget.route.secondaryAnimation != null)
      animations.add(widget.route.secondaryAnimation);
    _listenable = Listenable.merge(animations);
    if (widget.route.isCurrent) {
      widget.route.navigator.focusScopeNode.setFirstFocus(focusScopeNode);
    }
  }

  @override
  void didUpdateWidget(_ModalScope<T> oldWidget) {
    super.didUpdateWidget(oldWidget);
    assert(widget.route == oldWidget.route);
    if (widget.route.isCurrent) {
      widget.route.navigator.focusScopeNode.setFirstFocus(focusScopeNode);
    }
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    _page = null;
  }

  /// 强制重建页面，这回清除已有的 _page 缓存  
  void _forceRebuildPage() {
    setState(() {
      _page = null;
    });
  }

  @override
  void dispose() {
    focusScopeNode.dispose();
    super.dispose();
  }

  // This should be called to wrap any changes to route.isCurrent, route.canPop,
  // and route.offstage.
  void _routeSetState(VoidCallback fn) {
    setState(fn);
  }

  @override
  Widget build(BuildContext context) {
    return _ModalScopeStatus(
      route: widget.route,
      isCurrent: widget.route.isCurrent, // _routeSetState is called if this updates
      canPop: widget.route.canPop, // _routeSetState is called if this updates
      child: Offstage(
        offstage: widget.route.offstage, // _routeSetState is called if this updates
        child: PageStorage(
          bucket: widget.route._storageBucket, // immutable
          child: FocusScope(
            node: focusScopeNode, // immutable
            child: RepaintBoundary(
              child: AnimatedBuilder(
                animation: _listenable, // 当animation改变时，调用builder
                builder: (BuildContext context, Widget child) {
                  /// buildTransitions在这里被调用，用于构建过渡
                  /// buildPage产生的widget在这里作为child传入  
                  return widget.route.buildTransitions(
                    context,
                    widget.route.animation,
                    widget.route.secondaryAnimation,
                    IgnorePointer(
                      ignoring: widget.route.navigator.userGestureInProgress
                        || widget.route.animation?.status == AnimationStatus.reverse,
                      child: child,
                    ),
                  );
                },
                /// 缓存_page  
                child: _page ??= RepaintBoundary(
                  key: widget.route._subtreeKey, // immutable
                  child: Builder(
                    builder: (BuildContext context) {
                      /// buildPage在这里被调用，用于建造页面  
                      return widget.route.buildPage(
                        context,
                        widget.route.animation,
                        widget.route.secondaryAnimation,
                      );
                    },
                  ),
                ),
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```




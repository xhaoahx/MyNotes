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

  /// 是否是初始路由（第一个加入 [Navigator] 的路由）
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
/// 其中 T 是路由退出时的结果类型
abstract class Route<T> {
  /// 初始化路由
  Route({ RouteSettings settings }) : settings = settings ?? const RouteSettings();

  /// 持有这个路由的 Navigator
  NavigatorState get navigator => _navigator;
  NavigatorState _navigator;

  /// 配置
  final RouteSettings settings;

  /// 路由覆盖层（注意在 ModelRoute 重载的 createOverlayEntries，和在 OverlayRoute 调用到    
  /// _overlayEntries.addAll(createOverlayEntries())，它们产生了页面）  
  List<OverlayEntry> get overlayEntries => const <OverlayEntry>[];

  /// 向 navigator 中插入 route 时调用，插入在 insertionPoint（插入点）之后
  @protected
  @mustCallSuper
  void install(OverlayEntry insertionPoint) { }

  /// 在 [install] 之后，即当向 navigator 中加入路由之后调用
  /// 返回值是 Transition的过程（Future）
  /// 在此方法之后，立刻调用 [didChangeNext] 和 [didChangePrevious]  
  @protected
  TickerFuture didPush() => TickerFuture.complete();

  /// 在 [install] 之后，当使用一个新 route 替换另一个 route （oldRoute）时调用
  ///
  /// 在此方法之后，立刻调用 [didChangeNext] 和 [didChangePrevious]
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

  /// 以下是用于路由退出处理结果 FutureCompleter
  Future<T> get popped => _popCompleter.future;
  final Completer<T> _popCompleter = Completer<T>();

  /// 在 Pop 方法调用之后立刻调用此方法。默认实现调用了 didComplete，即完成 Pop 结果返回的 Completer
  @protected
  @mustCallSuper
  bool didPop(T result) {
    didComplete(result);
    return true;
  }

  /// 完成结果 future
  @protected
  @mustCallSuper
  void didComplete(T result) {
    _popCompleter.complete(result);
  }

  /// 下一个路由被弹出之后调用
  @protected
  @mustCallSuper
  void didPopNext(Route<dynamic> nextRoute) { }

  /// 下一个路由被改变时之后调用
  @protected
  @mustCallSuper
  void didChangeNext(Route<dynamic> nextRoute) { }

  /// 路由改变之后调用，对上一个路由的回调
  @protected
  @mustCallSuper
  void didChangePrevious(Route<dynamic> previousRoute) { }

  /// Called whenever the internal state of the route has changed.
  ///
  /// 当路由的[willhandlepopinner]、[didPop]、[offstage] 或其他内部状态改变时，应该调用这个函数。
  /// 例如，[ModalRoute]使用它通过 inheritedWidget 来向路由的任何子节点反馈更新信息。
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

  /// 是否是当前（最上层）的路由，即 _navigator._history 列表的最后一个
  bool get isCurrent {
    return _navigator != null && _navigator._history.last == this;
  }

  /// 是否是最下层的路由，即 _navigator._history 的第一个
  bool get isFirst {
    return _navigator != null && _navigator._history.first == this;
  }

  /// 是否是 Active 状态，即处于  _navigator._history 列表中
  bool get isActive {
    return _navigator != null && _navigator._history.contains(this);
  }
}
```



## NavigatorObserver

```dart
/// 用于观测 Navigator 行为的接口
class NavigatorObserver {
  NavigatorState get navigator => _navigator;
  NavigatorState _navigator;

  /// [Navigator] 在栈顶插入一个 route 之后调用
  ///
  /// 这会使得上一个 route 立即变成 previousRoute
  void didPush(Route<dynamic> route, Route<dynamic> previousRoute) { }

  /// [Navigator] 移除栈顶一个 route 之后调用
  void didPop(Route<dynamic> route, Route<dynamic> previousRoute) { }

  /// [Navigator] 移除栈中一个 route 之后调用
  void didRemove(Route<dynamic> route, Route<dynamic> previousRoute) { }

  /// [Navigator] 替换一个 route 之后调用
  void didReplace({ Route<dynamic> newRoute, Route<dynamic> oldRoute }) { }

  /// [Navigator] 正在通过手势控制一个 route
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
  /// 用给定的 RouteSetting 生成以一个 Route
  /// 通常是 WidgetApp 指定的:
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

  /// 这个 navigator 的观测者,当 navigator 发生改变时通知观测者
  final List<NavigatorObserver> observers;

  static const String defaultRouteName = '/';

  /// 以下方法都是是调用了 Navigator.of(context) 获取到的 NavigatorState，并调用其对应的方法	
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

  /// 当前路由是否可以退出
  static bool canPop(BuildContext context) {
    final NavigatorState navigator = Navigator.of(context, nullOk: true);
    return navigator != null && navigator.canPop();
  }

  /// 如果当前路由可以退出的话，那么退出
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
  /// 如果 rootNavigator = true，那么会查找根 NavigatorState
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
/// 注意，这里 mixin 了一个 TickerProviderStateMixin ，那么 NavigatorState 是可以作为 vsync 的
class NavigatorState 
    extends State<Navigator> 
    with TickerProviderStateMixin 
{
  ///此 globalKey 用于获取子 Overlay 的 State
  final GlobalKey<OverlayState> _overlayKey = GlobalKey<OverlayState>();
  final List<Route<dynamic>> _history = <Route<dynamic>>[];
  /// 已经被 pop 的路由
  final Set<Route<dynamic>> _poppedRoutes = <Route<dynamic>>{};

  /// The [FocusScopeNode] for the [FocusScope] that encloses the routes.
  final FocusScopeNode focusScopeNode = FocusScopeNode(debugLabel: 'Navigator Scope');

  final List<OverlayEntry> _initialOverlayEntries = <OverlayEntry>[];

  @override
  void initState() {
    super.initState();
    /// 使 NavigatorObserver 中每一个观测者都持有对自身的引用
    for (NavigatorObserver observer in widget.observers) {
      observer._navigator = this;
    }
    String initialRouteName = widget.initialRoute ?? Navigator.defaultRouteName;
    /// 如果initialRouteName类似于 /xxxx/xxxx/xxx 的话，会逐个 push 对应的 route 
    if (initialRouteName.startsWith('/') && initialRouteName.length > 1) {
      initialRouteName = initialRouteName.substring(1); // strip leading '/'
        
      /// 首先将 defaultRouteName 对应的 route 加入 plannedInitialRoutes 中
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
        
      /// plannedInitialRoutes 最后一项为 null ，则 push 默认 route  
      if (plannedInitialRoutes.last == null) {
        push(_routeNamed<Object>(Navigator.defaultRouteName, arguments: null));
      /// 否则 push 所有不为 null 的路由 （可能是因为没有对应的 routeNames 无法产生 route）   
      } else {
        plannedInitialRoutes.where(
            (Route<dynamic> route) => route != null
        ).forEach(push);
      }  
    /// 否则，如果 initialRouteName 不是以 ''/' 开头并且长度大与一的话，
    /// push 默认的路由
    } else {
      Route<Object> route;
      /// 如果自定义了初始路由名，push 对应的路由
      if (initialRouteName != Navigator.defaultRouteName)
        route = _routeNamed<Object>(initialRouteName, allowNull: true, arguments: null);
      /// 如果没有则使用默认的初始路由名，即'/'
      route ??= _routeNamed<Object>(Navigator.defaultRouteName, arguments: null);
      push(route);
    }
    /// 在_initialOverlayEntries 中 加入每个 route 的 overlayEntries 
    for (Route<dynamic> route in _history)
      _initialOverlayEntries.addAll(route.overlayEntries);
  }

  /// 更新 widget 的时候一并更新观测者
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
    /// 通知外部状态改变
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

  /// 当前覆盖层（即最上层可视的覆盖层）
  /// 逆转 _history 之后，返回第一个 overlayEntries 不为空 route 的最后一层覆盖层 
  OverlayEntry get _currentOverlayEntry {
    for (Route<dynamic> route in _history.reversed) {
      if (route.overlayEntries.isNotEmpty)
        return route.overlayEntries.last;
    }
    return null;
  }

  bool _debugLocked = false;
      
  /// 使用给定 name 和参数来配置 RouteSettings ，并调用 onGenerateRoute（setting）来产生一个新的 route
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

  /// 推入一个路由并且移除一些路由（当  RoutePredicate 返回 true 的时候停止移除）
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
  /// route.install(_currentOverlayEntry) // 为 route 创建了 overlayEntries
  /// 			|
  /// _history.add(route); 
  ///			|
  /// route.didPush();// TransitionRoute 在这里会启用动画 
  ///			|
  /// route.didChangeNext(null);
  /// 			|(oldRoute 为先前的一个路由)
  /// oldRoute.didChangeNext(route);
  /// 			|
  /// route.didChangePrevious(oldRoute); 
  ///			|
  /// for(NavigatorObserver observer in widget.observers) // 通知所有观测者 
  ///      observer.didPush(route, oldRoute);
  /// 			|
  /// _afterNavigation(route);// 使得页面 不可被点击
    
  /// 入栈一个给定的 route
  @optionalTypeArgs
  Future<T> push<T extends Object>(Route<T> route) {
    final Route<dynamic> oldRoute = _history.isNotEmpty ? _history.last : null;
    route._navigator = this;
    /// 插入到第一个 route 的最后一个 overlayEntry 后面  
    route.install(_currentOverlayEntry);
    /// 加入路由历史
    _history.add(route);
    route.didPush();
    route.didChangeNext(null);
    if (oldRoute != null) {
      oldRoute.didChangeNext(route);
      route.didChangePrevious(oldRoute);
    }
    /// 通知观测者
    for (NavigatorObserver observer in widget.observers)
      observer.didPush(route, oldRoute);
    /// 派发通知
    RouteNotificationMessages.maybeNotifyRouteChange(
        _routePushedMethod, route, oldRoute);
    /// 使页面不可被点击
    _afterNavigation(route);
    /// 返回一个 Future<T> 作为结果
    /// 注意到 poped 是 completer.future，故只需在 route.pop 的时候，调用 completer.complete 即可以完成 
    /// future
    return route.popped;
  }

  void _afterNavigation<T>(Route<T> route) {
    _cancelActivePointers();
  }

  /// 入栈一个给定的路由用于替换当前的的路由
  Future<T> pushReplacement<T extends Object, TO extends Object>(
      Route<T> newRoute, { TO result }) 
  {
    final Route<dynamic> oldRoute = _history.last;
    /// index 即最后一个路由的位置
    final int index = _history.length - 1;
    newRoute._navigator = this;
    newRoute.install(_currentOverlayEntry);
    /// 直接用新路由来替换旧路由
    _history[index] = newRoute;
    /// 在新路由完成推入的时候，把之前的路由 dispose
    newRoute.didPush().whenCompleteOrCancel(() {
      if (mounted) {
        oldRoute
          ..didComplete(result ?? oldRoute.currentResult)
          ..dispose();
      }
    });
    newRoute.didChangeNext(null);
    /// 旧路由的前一个路由，即倒数第二个路由
    if (index > 0) {
      _history[index - 1].didChangeNext(newRoute);
      newRoute.didChangePrevious(_history[index - 1]);
    }
    /// 通知观测者
    for (NavigatorObserver observer in widget.observers)
      observer.didReplace(newRoute: newRoute, oldRoute: oldRoute);
    RouteNotificationMessages.
        maybeNotifyRouteChange(_routeReplacedMethod, newRoute, oldRoute);
    _afterNavigation(newRoute);
    return newRoute.popped;
  }

  /// 推入一个路由并且不断移除直到 RoutePredicate 返回 true，过程和 pushReplacement 类似
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

  /// 用给定的新路由来替换一个指定的路由（不一定是最后一个，可以是 _history 中的任一个）
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

 
  /// 用给定的新路由替换给定的锚点路由下层的一个路由
  void replaceRouteBelow<T extends Object>({ @required Route<dynamic> anchorRoute, Route<T> newRoute }) {
    replace<T>(oldRoute: _history[_history.indexOf(anchorRoute) - 1], 
               newRoute: newRoute);
  }
    
  /// 是否可以退出当前路由
  bool canPop() {
    assert(_history.isNotEmpty);
    return _history.length > 1 || _history[0].willHandlePopInternally;
  }

  /// 尝试退出当前的路由 [Route.willPop]，能否退出会考虑 route.willPop 调用
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
        // 如果 route._navigator 是 null, 这说明在 didPop 时已经调用了 finalizeRoute，即这个路由已经被
        // 而无需把它加入到 _poppedRoutes 随后 dispose
        if (route._navigator != null)
          _poppedRoutes.add(route);
        /// 调用 didPopNext 回调
        _history.last.didPopNext(route);
        /// 通知观测者
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

  /// 移除一个指定的路由
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

  /// 移除一个给定锚点下层所有的路由
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
  /// 在 pop 时，将其加入 _poppedRoutes，之后可以恢复，
  /// finalizeRoute 会完全销毁 route，不可恢复  
  void finalizeRoute(Route<dynamic> route) {
    _poppedRoutes.remove(route);
    route.dispose();
  }

  /// 用户手势控制进程
  int get _userGesturesInProgress => _userGesturesInProgressCount;
  int _userGesturesInProgressCount = 0;
  set _userGesturesInProgress(int value) {
    _userGesturesInProgressCount = value;
    userGestureInProgressNotifier.value = _userGesturesInProgress > 0;
  }

  /// 路由是否当前正在被用户的手势操控，例如在 IOS 系统上的返回手势
  bool get userGestureInProgress => userGestureInProgressNotifier.value;

  final ValueNotifier<bool> userGestureInProgressNotifier = ValueNotifier<bool>(false);


  /// navigator 正在被用户的手势控制
  void didStartUserGesture() {
    _userGesturesInProgress += 1;
    if (_userGesturesInProgress == 1) {
      final Route<dynamic> route = _history.last;
      final Route<dynamic> previousRoute = !route.willHandlePopInternally && _history.length > 1
          ? _history[_history.length - 2]
          : null;
      for (NavigatorObserver observer in widget.observers)
        observer.didStartUserGesture(route, previousRoute);
    }
  }

  /// 用户手势完成
  ///
  /// Notifies the navigator that a gesture regarding which the navigator was
  /// previously notified with [didStartUserGesture] has completed.
  void didStopUserGesture() {
    assert(_userGesturesInProgress > 0);
    _userGesturesInProgress -= 1;
    if (_userGesturesInProgress == 0) {
      for (NavigatorObserver observer in widget.observers)
        observer.didStopUserGesture();
    }
  }

  /// 保存了指针 id 的集合
  final Set<int> _activePointers = <int>{};

  /// 处理指针落下事件
  void _handlePointerDown(PointerDownEvent event) {
    _activePointers.add(event.pointer);
  }

  /// 处理指针离开或者取消事件
  void _handlePointerUpOrCancel(PointerEvent event) {
    _activePointers.remove(event.pointer);
  }

  /// 操作 Navigator 是否能被点击
  void _cancelActivePointers() {
    // TODO(abarth): 这一机制还不够完善
    if (SchedulerBinding.instance.schedulerPhase == SchedulerPhase.idle) {
      // 向下查找到 RenderAbsorbPointer，将其 absorbing 设置为 true，即吸收所有的指针事件
      // 之后 markNeedsBuild 触发刷新
      // （这是对 RenderObject 的直接操作，似乎打破了框架的运作规律）
      final RenderAbsorbPointer absorber = 
          _overlayKey.currentContext?.findAncestorRenderObjectOfType<RenderAbsorbPointer>();
      setState(() {
        absorber?.absorbing = true;
      });
    }
    _activePointers.toList().forEach(WidgetsBinding.instance.cancelPointer);
  }

  /// 构建
  @override
  Widget build(BuildContext context) {
    /// 最外层有一个指针监听器，用于监听路由页面的指针事件，
    return Listener(
      onPointerDown: _handlePointerDown,
      onPointerUp: _handlePointerUpOrCancel,
      onPointerCancel: _handlePointerUpOrCancel,
      /// 是否允许被点击，被 _cancelActivePointers 直接操控
      child: AbsorbPointer(
        absorbing: false, // 被 _cancelActivePointers 直接操控
        child: FocusScope(
          node: focusScopeNode,
          autofocus: true,
          /// 这里为子树创建了一个顶层 Overlay ，用于显示页面内容  
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



## Route 子类



### 1.OverlayRoute

```dart
/// 一个可以创建覆盖层的 Route
abstract class OverlayRoute<T> extends Route<T> {
  OverlayRoute({
    RouteSettings settings,
  }) : super(settings: settings);

  /// 创建覆盖层列表，这个方法被子类重载
  /// ModalRoute 重载了此方法返回了两个 Overlay，分别包含了 Barrier 和 Page
  Iterable<OverlayEntry> createOverlayEntries();

  @override
  List<OverlayEntry> get overlayEntries => _overlayEntries;
  final List<OverlayEntry> _overlayEntries = <OverlayEntry>[];

  @override
  void install(OverlayEntry insertionPoint) {
    assert(_overlayEntries.isEmpty);
    _overlayEntries.addAll(createOverlayEntries());
    /// 调用 navigator.overlay.intsertAll（详见 [Navigator.overlay] 和 [Overlay]）
    /// 把创建的所有覆盖层插入到插入点之后
    /// 通常，Navigator 调用 Push 方法的时候，传入 insertionPoint 总是最上层的 OverlayEntry，即当前视图对
    /// 应的 OverlayEntry
    /// 故，Push 方法总是会添加新的 Overlay 到最上层
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



### 2.TransitionRoute

```dart
/// 当 Pop 或 Push 时，有动画效果过渡
abstract class TransitionRoute<T> extends OverlayRoute<T> {
  TransitionRoute({
    RouteSettings settings,
  }) : super(settings: settings);

  /// 过渡动画时候完成
  Future<T> get completed => _transitionCompleter.future;
  final Completer<T> _transitionCompleter = Completer<T>();

  Duration get transitionDuration;

  /// 页面透明度
  /// 如果设置为 true （也就是说这个页面是不透明的），那么当 Transition 完成时，这个 route 之前（或者说这个
  /// 页面之下的所有页面）的 route 就不会被建造，以节省资源
  bool get opaque;

  /// 返回动画完成的时候销毁路由
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
      /// navigator mixin 了一个 TickerProvider，这里直接把 Navigator 当作 vsync
      vsync: navigator,
    );
  }

  Animation<double> createAnimation() {
    /// Animation<double> get view => this; 
    /// 显然，_controller 自身也是一个 Animation<double> 
    return _controller.view;
  }

  /// 路由返回的结果
  T _result;

  /// 但动画状态改变时，触发回调  
  void _handleStatusChanged(AnimationStatus status) {
    switch (status) {
      /// 如果当动画完成时，把 overlayEntries 的第一个 OverlayEntry.opaque （即最下层的覆盖层的透明度）设置
      /// 为给定的 opaque
      /// 若给定 opaque 是 true 的话，会使得先前路由被遮蔽，在 OverlayState 中非把先前的路由从
      /// _onstage 列表中里移除 
      case AnimationStatus.completed:
        if (overlayEntries.isNotEmpty)
          overlayEntries.first.opaque = opaque;
        break;
      /// 显然，在动画还没有完成的时候，需要展示下层的覆盖层，必须把本覆盖层的 opaque 设置成 false
      case AnimationStatus.forward:
      case AnimationStatus.reverse:
        if (overlayEntries.isNotEmpty)
          overlayEntries.first.opaque = false;
        break;
      /// 如果 AnimationStatus == dismissed，这说明这个路由被 pop 了，因此需要被销毁
      case AnimationStatus.dismissed:
        if (!isActive) {
          navigator.finalizeRoute(this);
        }
        break;
    }
      
    /// 这个方法在 ModalRoute 里被重写
    /// 调用了_modalBarrier.markNeedsBuild();  
    changedInternalState();
  }

  /// 创建了 AnimationController   
  @override
  void install(OverlayEntry insertionPoint) {
    _controller = createAnimationController();
    /// 这里的 animation 实际就是 _controller
    _animation = createAnimation();
    super.install(insertionPoint);
  }

  /// 在 install 之后通常会调用 didPush,开始动画  
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

  /// 时候能够过渡到另一个 route
  /// 当返回 true 时，当一个新的路由被 push 或者 pop 到这个路由的上层的时候，这个路由的 secondaryAnimation 
  /// 会开始运行
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



#### LocalHistoryEntry

```dart
/// 信息类
class LocalHistoryEntry {
  LocalHistoryEntry({ this.onRemove });

  /// 被移除时的回调
  final VoidCallback onRemove;

  LocalHistoryRoute<dynamic> _owner;

  /// 把这个 Entry 从其关联的 [LocalHistoryRoute] 的 history 中移除
  void remove() {
    _owner.removeLocalHistoryEntry(this);
  }

  /// 通知这个 Entry 已经被移除了，调用给定的 移除回调
  void _notifyRemoved() {
    if (onRemove != null)
      onRemove();
  }
}
```



### 3.LocalHistoryRoute

```dart
/// 一个能够保存在当前页面上相加新视图的路由
/// 用例如下：
///
/// ```dart
/// void main() => runApp(App());
/// class App extends StatelessWidget {
///   @override
///   Widget build(BuildContext context) {
///     return MaterialApp(
///       initialRoute: '/',
///       routes: {
///         '/': (BuildContext context) => HomePage(),
///         '/second_page': (BuildContext context) => SecondPage(),
///       },
///     );
///   }
/// }
///
/// class HomePage extends StatefulWidget {
///   HomePage();
///
///   @override
///   _HomePageState createState() => _HomePageState();
/// }
///
/// class _HomePageState extends State<HomePage> {
///   @override
///   Widget build(BuildContext context) {
///     return Scaffold(
///       body: Center(
///         child: Column(
///           mainAxisSize: MainAxisSize.min,
///           children: <Widget>[
///             Text('HomePage'),
///             // Press this button to open the SecondPage.
///             RaisedButton(
///               child: Text('Second Page >'),
///               onPressed: () {
///                 Navigator.pushNamed(context, '/second_page');
///               },
///             ),
///           ],
///         ),
///       ),
///     );
///   }
/// }
///
/// class SecondPage extends StatefulWidget {
///   @override
///   _SecondPageState createState() => _SecondPageState();
/// }
///
/// class _SecondPageState extends State<SecondPage> {
///
///   bool _showRectangle = false;
///
///   void _navigateLocallyToShowRectangle() {
///     setState(() => _showRectangle = true);
///     ModalRoute.of(context).addLocalHistoryEntry(
///         LocalHistoryEntry(
///             onRemove: () {
///               // Hide the red rectangle.
///               setState(() => _showRectangle = false);
///             }
///         )
///     );
///   }
///
///   @override
///   Widget build(BuildContext context) {
///     final localNavContent = _showRectangle
///       ? Container(
///           width: 100.0,
///           height: 100.0,
///           color: Colors.red,
///         )
///       : RaisedButton(
///           child: Text('Show Rectangle'),
///           onPressed: _navigateLocallyToShowRectangle,
///         );
///
///     return Scaffold(
///       body: Center(
///         child: Column(
///           mainAxisAlignment: MainAxisAlignment.center,
///           children: <Widget>[
///             localNavContent,
///             RaisedButton(
///               child: Text('< Back'),
///               onPressed: () {
///                 // 当红色矩形存在时，会使其消失；否则，直接退出路由，返回 Home 页
///                 Navigator.of(context).pop();
///               },
///             ),
///           ],
///         ),
///       ),
///     );
///   }
/// }
/// ```
/// {@end-tool}
mixin LocalHistoryRoute<T> on Route<T> {
  List<LocalHistoryEntry> _localHistory;

  /// 添加一个 HistoryEntry 到列表中
  void addLocalHistoryEntry(LocalHistoryEntry entry) {
    entry._owner = this;
    _localHistory ??= <LocalHistoryEntry>[];
    final bool wasEmpty = _localHistory.isEmpty;
    _localHistory.add(entry);
    if (wasEmpty)
      changedInternalState();
  }

  /// 移除一个 historyEntry，并调用其回调
  void removeLocalHistoryEntry(LocalHistoryEntry entry) {
    _localHistory.remove(entry);
    entry._owner = null;
    entry._notifyRemoved();
    if (_localHistory.isEmpty)
      changedInternalState();
  }

  /// 重载了 willPop 方法，如果
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

  /// 如果 _localHistory 列表的长度不为 0 的话，那么这次 Pop 将被视作页面内部的 Pop，也就是说，这次 Pop 不会
  /// 将当前页面 Pop 掉，而是移除一层 Histroy
  @override
  bool get willHandlePopInternally {
    return _localHistory != null && _localHistory.isNotEmpty;
  }
}
```



#### _ModalScopeStatus

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

  /// 是否是 current route  
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



#### _ModalScope

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



#### _ModalScopeState

```dart
class _ModalScopeState<T> extends State<_ModalScope<T>> {
  Widget _page;

  /// 这个 listenable 合并了一个 route 的两个 animation（animation 和 secondaryAnimation）
  /// 附：Listenable.merge 构造函数实际返回了一个 _MergeListenable ，当向其中添加一个 listenr 时，它会
  /// 在其 _children 中的每一个 listenable 中注册这个 listener
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
    /// 返回一个 _ModalScopeStatus
    return _ModalScopeStatus(
      route: widget.route,
      isCurrent: widget.route.isCurrent, // _routeSetState is called if this updates
      canPop: widget.route.canPop, // _routeSetState is called if this updates
      /// 当 offstage 为真时，不会进行子树的绘制
      child: Offstage(
        offstage: widget.route.offstage, // 当 route.offstage 改变的时候，setState 会被调用
        /// PageStorage 用于储存滚动状态等信息
        child: PageStorage(
          bucket: widget.route._storageBucket,
          /// 焦点根组件
          child: FocusScope(
            node: focusScopeNode,
            /// 重绘边界，用于提高性能
            /// 显然，当子树（页面）重绘的时候，上层无需重绘
            child: RepaintBoundary(
              child: AnimatedBuilder(
                animation: _listenable,
                builder: (BuildContext context, Widget child) {
                  /// buildTransitions 在这里被调用，用于构建过渡
                  /// buildPage 产生的 widget 在这里作为child传入  
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
                /// 缓存 _page  
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

### 3.ModalRoute 

```dart
/// 一个可以在执行动画时阻碍与先前路由交互的 Route
abstract class ModalRoute<T> 
    extends TransitionRoute<T>
    /// 注意这里 mixin 了 LocalHistoryRoute<T> 
    with LocalHistoryRoute<T> 
{
  ModalRoute({
    RouteSettings settings,
  }) : super(settings: settings);

  /// 在一个 Widget 中调用 ModalRoute.of(context) 以获取到它所在路由（ModalRoute）
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
      // route 当前处于不可见状态，因此不需要 setState
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
    /// super.animation 即 TransitionRoute 中的 animation
    _animationProxy = ProxyAnimation(super.animation);
    /// super.secondaryAnimation 即 TransitionRoute 中的 secondaryAnimation
    _secondaryAnimationProxy = ProxyAnimation(super.secondaryAnimation);
  }

  /// push 之后，为推送的页面请求一次焦点，这可能是 TextField 的 autoFocus 属性的用处
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

  /// 是否可见（offStage 为 true 时，会阻止子树的渲染（并不会阻止布局，通常来说，布局是一定会进行的））  
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

  /// 子树的 Buildcontext
  BuildContext get subtreeContext => _subtreeKey.currentContext;

  /// 向外提供 animation，被 buildPage 和 buildTransition 方法使用
  @override
  Animation<double> get animation => _animationProxy;
  ProxyAnimation _animationProxy;
  /// 同上
  @override
  Animation<double> get secondaryAnimation => _secondaryAnimationProxy;
  ProxyAnimation _secondaryAnimationProxy;
  
  final List<WillPopCallback> _willPopCallbacks = <WillPopCallback>[];

  @override
  Future<RoutePopDisposition> willPop() async {
    final _ModalScopeState<T> scope = _scopeKey.currentState;
    for (WillPopCallback callback in List<WillPopCallback>.from(_willPopCallbacks)) {
      /// 如果有一个回调返回了 false 的话，那么不能退出
      if (!await callback())
        return RoutePopDisposition.doNotPop;
    }
    return await super.willPop();
  }

  /// 添加一个退出时回调
  void addScopedWillPopCallback(WillPopCallback callback) {
    _willPopCallbacks.add(callback);
  }

  /// 移除一个退出时回调
  void removeScopedWillPopCallback(WillPopCallback callback) {
    _willPopCallbacks.remove(callback);
  }

  /// 是否有退出时回调
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

  /// 外部状态改变，这会清除 _ModalScopeState 中的 page 缓存
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
  
  /// 建造一个背景板  
  Widget _buildModalBarrier(BuildContext context) {
    Widget barrier;
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

  /// 这里重载了 createOverlayEntries，返回了两个 OverlayEntry，
  /// 其一包含了背景版，（把 _buildModalBarrier 当作 builder）
  /// 其二包含了页面实体，（把 _buildModalScope 当作 builder。详见[_ModalScope]）
  /// 通常，在 insert 的时候把这两个 OverlayEntry 插入到了 Overlay 中 _entries 的尾端，
  /// 即当前视图的最上层。
  @override
  Iterable<OverlayEntry> createOverlayEntries() sync* {
    yield _modalBarrier = OverlayEntry(builder: _buildModalBarrier);
    yield OverlayEntry(builder: _buildModalScope, maintainState: maintainState);
  }

}

```



### 4.PageRoute

```dart
/// 其实是对 ModalRoute 的简单封装
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



### 5.MaterialPageRoute

```dart
/// Material 设计风格适应的页面路由
class MaterialPageRoute<T> extends PageRoute<T> {

  MaterialPageRoute({
    @required this.builder,
    RouteSettings settings,
    /// 默认 maintainState 为 true
    /// 这会使得 push 另一个页面时，保持上一个页面状态，但也会消耗一定资源
    /// 当页面层数多，且上一页状态无需被储存时，设置 false 提升性能  
    this.maintainState = true,
    bool fullscreenDialog = false,
  }) : super(settings: settings, fullscreenDialog: fullscreenDialog);

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



### 6.CupertinoPageRoute

```dart
class CupertinoPageRoute<T> extends PageRoute<T> {
  /// Creates a page route for use in an iOS designed app.
  ///
  /// The [builder], [maintainState], and [fullscreenDialog] arguments must not
  /// be null.
  CupertinoPageRoute({
    @required this.builder,
    this.title,
    RouteSettings settings,
    this.maintainState = true,
    bool fullscreenDialog = false,
  }) : super(settings: settings, fullscreenDialog: fullscreenDialog);

  /// Builds the primary contents of the route.
  final WidgetBuilder builder;

  /// A title string for this route.
  ///
  /// Used to auto-populate [CupertinoNavigationBar] and
  /// [CupertinoSliverNavigationBar]'s `middle`/`largeTitle` widgets when
  /// one is not manually supplied.
  final String title;

  ValueNotifier<String> _previousTitle;

  /// The title string of the previous [CupertinoPageRoute].
  ///
  /// The [ValueListenable]'s value is readable after the route is installed
  /// onto a [Navigator]. The [ValueListenable] will also notify its listeners
  /// if the value changes (such as by replacing the previous route).
  ///
  /// The [ValueListenable] itself will be null before the route is installed.
  /// Its content value will be null if the previous route has no title or
  /// is not a [CupertinoPageRoute].
  ///
  /// See also:
  ///
  ///  * [ValueListenableBuilder], which can be used to listen and rebuild
  ///    widgets based on a ValueListenable.
  ValueListenable<String> get previousTitle {
    assert(
      _previousTitle != null,
      'Cannot read the previousTitle for a route that has not yet been installed',
    );
    return _previousTitle;
  }

  @override
  void didChangePrevious(Route<dynamic> previousRoute) {
    final String previousTitleString = previousRoute is CupertinoPageRoute
        ? previousRoute.title
        : null;
    if (_previousTitle == null) {
      _previousTitle = ValueNotifier<String>(previousTitleString);
    } else {
      _previousTitle.value = previousTitleString;
    }
    super.didChangePrevious(previousRoute);
  }

  @override
  final bool maintainState;

  @override
  // A relatively rigorous eyeball estimation.
  Duration get transitionDuration => const Duration(milliseconds: 400);

  @override
  Color get barrierColor => null;

  @override
  String get barrierLabel => null;

  @override
  bool canTransitionFrom(TransitionRoute<dynamic> previousRoute) {
    return previousRoute is CupertinoPageRoute;
  }

  @override
  bool canTransitionTo(TransitionRoute<dynamic> nextRoute) {
    // Don't perform outgoing animation if the next route is a fullscreen dialog.
    return nextRoute is CupertinoPageRoute && !nextRoute.fullscreenDialog;
  }

  /// True if an iOS-style back swipe pop gesture is currently underway for [route].
  ///
  /// This just check the route's [NavigatorState.userGestureInProgress].
  ///
  /// See also:
  ///
  ///  * [popGestureEnabled], which returns true if a user-triggered pop gesture
  ///    would be allowed.
  static bool isPopGestureInProgress(PageRoute<dynamic> route) {
    return route.navigator.userGestureInProgress;
  }

  /// True if an iOS-style back swipe pop gesture is currently underway for this route.
  ///
  /// See also:
  ///
  ///  * [isPopGestureInProgress], which returns true if a Cupertino pop gesture
  ///    is currently underway for specific route.
  ///  * [popGestureEnabled], which returns true if a user-triggered pop gesture
  ///    would be allowed.
  bool get popGestureInProgress => isPopGestureInProgress(this);

  /// Whether a pop gesture can be started by the user.
  ///
  /// Returns true if the user can edge-swipe to a previous route.
  ///
  /// Returns false once [isPopGestureInProgress] is true, but
  /// [isPopGestureInProgress] can only become true if [popGestureEnabled] was
  /// true first.
  ///
  /// This should only be used between frames, not during build.
  bool get popGestureEnabled => _isPopGestureEnabled(this);

  static bool _isPopGestureEnabled<T>(PageRoute<T> route) {
    // If there's nothing to go back to, then obviously we don't support
    // the back gesture.
    if (route.isFirst)
      return false;
    // If the route wouldn't actually pop if we popped it, then the gesture
    // would be really confusing (or would skip internal routes), so disallow it.
    if (route.willHandlePopInternally)
      return false;
    // If attempts to dismiss this route might be vetoed such as in a page
    // with forms, then do not allow the user to dismiss the route with a swipe.
    if (route.hasScopedWillPopCallback)
      return false;
    // Fullscreen dialogs aren't dismissible by back swipe.
    if (route.fullscreenDialog)
      return false;
    // If we're in an animation already, we cannot be manually swiped.
    if (route.animation.status != AnimationStatus.completed)
      return false;
    // If we're being popped into, we also cannot be swiped until the pop above
    // it completes. This translates to our secondary animation being
    // dismissed.
    if (route.secondaryAnimation.status != AnimationStatus.dismissed)
      return false;
    // If we're in a gesture already, we cannot start another.
    if (isPopGestureInProgress(route))
      return false;

    // Looks like a back gesture would be welcome!
    return true;
  }

  @override
  Widget buildPage(
      BuildContext context, 
      Animation<double> animation, 
      Animation<double> secondaryAnimation
  ) {
    final Widget child = builder(context);
    final Widget result = Semantics(
      scopesRoute: true,
      explicitChildNodes: true,
      child: child,
    );
    return result;
  }

  // Called by _CupertinoBackGestureDetector when a pop ("back") drag start
  // gesture is detected. The returned controller handles all of the subsequent
  // drag events.
  static _CupertinoBackGestureController<T> _startPopGesture<T>(PageRoute<T> route) {
    assert(_isPopGestureEnabled(route));

    return _CupertinoBackGestureController<T>(
      navigator: route.navigator,
      controller: route.controller, // protected access
    );
  }

  /// Returns a [CupertinoFullscreenDialogTransition] if [route] is a full
  /// screen dialog, otherwise a [CupertinoPageTransition] is returned.
  ///
  /// Used by [CupertinoPageRoute.buildTransitions].
  ///
  /// This method can be applied to any [PageRoute], not just
  /// [CupertinoPageRoute]. It's typically used to provide a Cupertino style
  /// horizontal transition for material widgets when the target platform
  /// is [TargetPlatform.iOS].
  ///
  /// See also:
  ///
  ///  * [CupertinoPageTransitionsBuilder], which uses this method to define a
  ///    [PageTransitionsBuilder] for the [PageTransitionsTheme].
  static Widget buildPageTransitions<T>(
    PageRoute<T> route,
    BuildContext context,
    Animation<double> animation,
    Animation<double> secondaryAnimation,
    Widget child,
  ) {
    if (route.fullscreenDialog) {
      return CupertinoFullscreenDialogTransition(
        animation: animation,
        child: child,
      );
    } else {
      return CupertinoPageTransition(
        primaryRouteAnimation: animation,
        secondaryRouteAnimation: secondaryAnimation,
        // Check if the route has an animation that's currently participating
        // in a back swipe gesture.
        //
        // In the middle of a back gesture drag, let the transition be linear to
        // match finger motions.
        linearTransition: isPopGestureInProgress(route),
        child: _CupertinoBackGestureDetector<T>(
          enabledCallback: () => _isPopGestureEnabled<T>(route),
          onStartPopGesture: () => _startPopGesture<T>(route),
          child: child,
        ),
      );
    }
  }

  @override
  Widget buildTransitions(BuildContext context, Animation<double> animation, Animation<double> secondaryAnimation, Widget child) {
    return buildPageTransitions<T>(this, context, animation, secondaryAnimation, child);
  }

  @override
  String get debugLabel => '${super.debugLabel}(${settings.name})';
}

```


# Flutter WidgetsApp 源码分析



## 函数原型

```dart
typedef RouteFactory = Route<dynamic> Function(RouteSettings settings);

typedef PageRouteFactory = PageRoute<T> Function<T>(RouteSettings settings, WidgetBuilder builder);

typedef GenerateAppTitle = String Function(BuildContext context);
```



## WidgetsApp
```dart
/// 一个方便的小部件，它包装了许多应用程序通常需要的小部件。
///
/// [WidgetsApp]提供的主要角色之一是将系统后退按钮绑定到弹出[Navigator]或退出应用程序。
/// 
/// 它被 [MaterialApp] 和 [CupertinoApp] 用来实现应用程序的基本功能。
class WidgetsApp extends StatefulWidget {
  // 大多数调用者希望使用 [home] 或 [route] ，或者两者都使用。[home] 参数是为了方便以下 [route] 映射:
  ///
  /// ```dart
  /// <String, WidgetBuilder>{ '/': (BuildContext context) => myWidget }
  /// ```
  ///
  /// 可用同时指定 [home] and [routes], 但仅当 [route] 不包含'/'的条目时。相反，如果 [home] 被省略，
  /// [routes],必须包含一个'/'项。
  ///
  /// 如果 [home] 或者 [routes] 都不为 null, 路由实现则必须知道如何正确地构建 [PageRoutes]. This can be achieved by supplying the
  /// [pageRouteBuilder] parameter.  The [pageRouteBuilder] is used by [MaterialApp]
  /// and [CupertinoApp] to create [MaterialPageRoute]s and [CupertinoPageRoute],
  /// respectively.
  ///
  /// The [builder] parameter is designed to provide the ability to wrap the visible
  /// content of the app in some other widget. It is recommended that you use [home]
  /// rather than [builder] if you intend to only display a single route in your app.
  ///
  /// [WidgetsApp] is also possible to provide a custom implementation of routing via the
  /// [onGeneratedRoute] and [onUnknownRoute] parameters. These parameters correspond
  /// to [Navigator.onGenerateRoute] and [Navigator.onUnknownRoute]. If [home], [routes],
  /// and [builder] are null, or if they fail to create a requested route,
  /// [onGeneratedRoute] will be invoked.  If that fails, [onUnknownRoute] will be invoked.
  ///
  /// The [pageRouteBuilder] will create a [PageRoute] that wraps newly built routes.
  /// If the [builder] is non-null and the [onGenerateRoute] argument is null, then the
  /// [builder] will not be provided only with the context and the child widget, whereas
  /// the [pageRouteBuilder] will be provided with [RouteSettings]. If [onGenerateRoute]
  /// is not provided, [navigatorKey], [onUnknownRoute], [navigatorObservers], and
  /// [initialRoute] must have their default values, as they will have no effect.
  ///
  /// The `supportedLocales` argument must be a list of one or more elements.
  /// By default supportedLocales is `[const Locale('en', 'US')]`.
  WidgetsApp({ // can't be const because the asserts use methods on Iterable :-(
    Key key,
    this.navigatorKey,
    this.onGenerateRoute,
    this.onUnknownRoute,
    this.navigatorObservers = const <NavigatorObserver>[],
    this.initialRoute,
    this.pageRouteBuilder,
    this.home,
    this.routes = const <String, WidgetBuilder>{},
    this.builder,
    this.title = '',
    this.onGenerateTitle,
    this.textStyle,
    @required this.color,
    this.locale,
    this.localizationsDelegates,
    this.localeListResolutionCallback,
    this.localeResolutionCallback,
    this.supportedLocales = const <Locale>[Locale('en', 'US')],
    this.showPerformanceOverlay = false,
    this.checkerboardRasterCacheImages = false,
    this.checkerboardOffscreenLayers = false,
    this.showSemanticsDebugger = false,
    this.debugShowWidgetInspector = false,
    this.debugShowCheckedModeBanner = true,
    this.inspectorSelectButtonBuilder,
    this.shortcuts,
    this.actions,
  }) : super(key: key);

  final GlobalKey<NavigatorState> navigatorKey;

  /// app 导航到新路由时的路由产生回调
  ///
  /// If this returns null when building the routes to handle the specified
  /// [initialRoute], then all the routes are discarded and
  /// [Navigator.defaultRouteName] is used instead (`/`). See [initialRoute].
  ///
  /// 通常，在 app 运行期间，[onGenerateRoute] 只会被命名路由使用，因此这个方法不应返回 null
  ///
  /// This is used if [routes] does not contain the requested route.
  ///
  /// The [Navigator] is only built if routes are provided (either via [home],
  /// [routes], [onGenerateRoute], or [onUnknownRoute]); if they are not,
  /// [builder] must not be null.
  /// {@endtemplate}
  ///
  /// If this property is not set, either the [routes] or [home] properties must
  /// be set, and the [pageRouteBuilder] must also be set so that the
  /// default handler will know what routes and [PageRoute]s to build.
  final RouteFactory onGenerateRoute;

  /// 当应用程序导航到指定路由时使用的 [PageRoute] 生成器回调。
  ///
  /// 例如，可以使用这个回调来指定应该使用 [MaterialPageRoute] 或 [CupertinoPageRoute] 来构建页面转换。
  final PageRouteFactory pageRouteBuilder;
    
  final Widget home;

  /// 应用的顶层路由表
  ///
  /// 当用 [Navigator] 推送指定的命名路由时，在这映射表上查找名字。如果存在名称，则使用关联的 
  /// [WidgetBuilder] 来构造一个 [PageRoute]，该 [PageRoute] 由 [pageRouteBuilder] 指定，以执行适当的转换
  /// 动画(包括 [Hero] 动画)到新路由。
  final Map<String, WidgetBuilder> routes;
    
  final RouteFactory onUnknownRoute;
  final String initialRoute;

  /// 导航的观测者
  final List<NavigatorObserver> navigatorObservers;
  final TransitionBuilder builder;

  /// 用于表示 app 的名字
  final String title;
  final GenerateAppTitle onGenerateTitle;

  /// 此应用的默认 [Text] 文字样式
  final TextStyle textStyle;

  /// app 在操作系统界面的颜色
  final Color color;

  /// 本地化，如果没有显示提供，则使用系统默认值
  final Locale locale;

  /// 用于配置语言
  final Iterable<LocalizationsDelegate<dynamic>> localizationsDelegates;
  final LocaleListResolutionCallback localeListResolutionCallback;
  final LocaleResolutionCallback localeResolutionCallback;

  /// 支持的语言
  /// ```dart
  /// // Full Chinese support for CN, TW, and HK
  /// supportedLocales: [
  ///   const Locale.fromSubtags(languageCode: 'zh'), // generic Chinese 'zh'
  ///   const Locale.fromSubtags(languageCode: 'zh', scriptCode: 'Hans'), 
  ///   const Locale.fromSubtags(languageCode: 'zh', scriptCode: 'Hant'), 
  ///   const Locale.fromSubtags(languageCode: 'zh', scriptCode: 'Hans', countryCode: 'CN'), 
  ///   const Locale.fromSubtags(languageCode: 'zh', scriptCode: 'Hant', countryCode: 'TW'), 
  ///   const Locale.fromSubtags(languageCode: 'zh', scriptCode: 'Hant', countryCode: 'HK'), 
  /// ],
  /// ```
  ///
  final Iterable<Locale> supportedLocales;

  /// 性能展示图层
  final bool showPerformanceOverlay;

  /// 棋盘格光栅缓存图像。
  final bool checkerboardRasterCacheImages;

  /// 棋盘格层渲染到屏幕外的位图。
  final bool checkerboardOffscreenLayers;

  /// 可访问性相关
  final bool showSemanticsDebugger;
  final bool debugShowWidgetInspector;

  /// Builds the widget the [WidgetInspector] uses to switch between view and
  /// inspect modes.
  ///
  /// This lets [MaterialApp] to use a material button to toggle the inspector
  /// select mode without requiring [WidgetInspector] to depend on the
  /// material package.
  final InspectorSelectButtonBuilder inspectorSelectButtonBuilder;

  /// 是否显示右上角的 debug benner
  final bool debugShowCheckedModeBanner;

  /// 键盘快捷方式
  final Map<LogicalKeySet, Intent> shortcuts;
  
  final Map<LocalKey, ActionFactory> actions;

  static bool showPerformanceOverlayOverride = false;

  /// 如果为 true，则强制使 Inspector 可见
  ///
  /// Used by the `debugShowWidgetInspector` debugging extension.
  ///
  /// Inspector 允许在设备或模拟器上选择一个位置，并查看与之关联的组件和渲染对象。
  /// 所选组的概要和一些摘要信息显示在设备上，更详细的信息显示在 IDE 或 Observatory 中。
  static bool debugShowWidgetInspectorOverride = false;
  static bool debugAllowBannerOverride = true;

  /// 以下定义了键盘的快捷键的默认行为：
    
  /// android & fuchsia
  static final Map<LogicalKeySet, Intent> _defaultShortcuts = <LogicalKeySet, Intent>{
    // Activation
    LogicalKeySet(LogicalKeyboardKey.enter): 
      const Intent(ActivateAction.key),
    LogicalKeySet(LogicalKeyboardKey.space): 
      const Intent(ActivateAction.key),

    // Keyboard traversal.
    LogicalKeySet(LogicalKeyboardKey.tab): 
      const Intent(NextFocusAction.key),
    LogicalKeySet(LogicalKeyboardKey.shift, LogicalKeyboardKey.tab): 
      const Intent(PreviousFocusAction.key),
    LogicalKeySet(LogicalKeyboardKey.arrowLeft): 
      const DirectionalFocusIntent(TraversalDirection.left),
    LogicalKeySet(LogicalKeyboardKey.arrowRight): 
      const DirectionalFocusIntent(TraversalDirection.right),
    LogicalKeySet(LogicalKeyboardKey.arrowDown): 
      const DirectionalFocusIntent(TraversalDirection.down),
    LogicalKeySet(LogicalKeyboardKey.arrowUp): 
      const DirectionalFocusIntent(TraversalDirection.up),

    // scrolling
    LogicalKeySet(LogicalKeyboardKey.control, LogicalKeyboardKey.arrowUp): 
      const ScrollIntent(direction: AxisDirection.up),
    LogicalKeySet(LogicalKeyboardKey.control, LogicalKeyboardKey.arrowDown): 
      const ScrollIntent(direction: AxisDirection.down),
    LogicalKeySet(LogicalKeyboardKey.control, LogicalKeyboardKey.arrowLeft): 
      const ScrollIntent(direction: AxisDirection.left),
    LogicalKeySet(LogicalKeyboardKey.control, LogicalKeyboardKey.arrowRight): 
      const ScrollIntent(direction: AxisDirection.right),
    LogicalKeySet(LogicalKeyboardKey.pageUp): 
      const ScrollIntent(direction: AxisDirection.up, type: ScrollIncrementType.page),
    LogicalKeySet(LogicalKeyboardKey.pageDown): 
      const ScrollIntent(direction: AxisDirection.down, type: ScrollIncrementType.page),
  };

  /// web
  static final Map<LogicalKeySet, Intent> _defaultWebShortcuts = <LogicalKeySet, Intent>{
    // Activation
    LogicalKeySet(LogicalKeyboardKey.space): 
      const Intent(ActivateAction.key),

    // Keyboard traversal.
    LogicalKeySet(LogicalKeyboardKey.tab): 
      const Intent(NextFocusAction.key),
    LogicalKeySet(LogicalKeyboardKey.shift, LogicalKeyboardKey.tab): 
      const Intent(PreviousFocusAction.key),

    // Scrolling
    LogicalKeySet(LogicalKeyboardKey.arrowUp): 
      const ScrollIntent(direction: AxisDirection.up),
    LogicalKeySet(LogicalKeyboardKey.arrowDown): 
      const ScrollIntent(direction: AxisDirection.down),
    LogicalKeySet(LogicalKeyboardKey.arrowLeft): 
      const ScrollIntent(direction: AxisDirection.left),
    LogicalKeySet(LogicalKeyboardKey.arrowRight): 
      const ScrollIntent(direction: AxisDirection.right),
    LogicalKeySet(LogicalKeyboardKey.pageUp): 
      const ScrollIntent(direction: AxisDirection.up, type: ScrollIncrementType.page),
    LogicalKeySet(LogicalKeyboardKey.pageDown): 
      const ScrollIntent(direction: AxisDirection.down, type: ScrollIncrementType.page),
  };

  // macOS
  static final Map<LogicalKeySet, Intent> _defaultMacOsShortcuts = <LogicalKeySet, Intent>{
    // Activation
    LogicalKeySet(LogicalKeyboardKey.enter): 
      const Intent(ActivateAction.key),
    LogicalKeySet(LogicalKeyboardKey.space): 
      const Intent(ActivateAction.key),

    // Keyboard traversal
    LogicalKeySet(LogicalKeyboardKey.tab): const Intent(NextFocusAction.key),
    LogicalKeySet(LogicalKeyboardKey.shift, LogicalKeyboardKey.tab): 
      const Intent(PreviousFocusAction.key),
    LogicalKeySet(LogicalKeyboardKey.arrowLeft): 
      const DirectionalFocusIntent(TraversalDirection.left),
    LogicalKeySet(LogicalKeyboardKey.arrowRight): 
      const DirectionalFocusIntent(TraversalDirection.right),
    LogicalKeySet(LogicalKeyboardKey.arrowDown): 
      const DirectionalFocusIntent(TraversalDirection.down),
    LogicalKeySet(LogicalKeyboardKey.arrowUp): 
      const DirectionalFocusIntent(TraversalDirection.up),

    // Scrolling
    LogicalKeySet(LogicalKeyboardKey.meta, LogicalKeyboardKey.arrowUp): 
      const ScrollIntent(direction: AxisDirection.up),
    LogicalKeySet(LogicalKeyboardKey.meta, LogicalKeyboardKey.arrowDown): 
      const ScrollIntent(direction: AxisDirection.down),
    LogicalKeySet(LogicalKeyboardKey.meta, LogicalKeyboardKey.arrowLeft): 
      const ScrollIntent(direction: AxisDirection.left),
    LogicalKeySet(LogicalKeyboardKey.meta, LogicalKeyboardKey.arrowRight): 
      const ScrollIntent(direction: AxisDirection.right),
    LogicalKeySet(LogicalKeyboardKey.pageUp): 
      const ScrollIntent(direction: AxisDirection.up, type: ScrollIncrementType.page),
    LogicalKeySet(LogicalKeyboardKey.pageDown): 
      const ScrollIntent(direction: AxisDirection.down, type: ScrollIncrementType.page),
  };

  /// 根据平台来获取对应的快捷方式
  static Map<LogicalKeySet, Intent> get defaultShortcuts {
    if (kIsWeb) {
      return _defaultWebShortcuts;
    }

    switch (defaultTargetPlatform) {
      case TargetPlatform.android:
      case TargetPlatform.fuchsia:
        return _defaultShortcuts;
      case TargetPlatform.macOS:
        return _defaultMacOsShortcuts;
      case TargetPlatform.iOS:
        break;
    }
    return <LogicalKeySet, Intent>{};
  }

  /// The default value of [WidgetsApp.actions].
  static final Map<LocalKey, ActionFactory> defaultActions = <LocalKey, ActionFactory>{
    DoNothingAction.key: () => const DoNothingAction(),
    RequestFocusAction.key: () => RequestFocusAction(),
    NextFocusAction.key: () => NextFocusAction(),
    PreviousFocusAction.key: () => PreviousFocusAction(),
    DirectionalFocusAction.key: () => DirectionalFocusAction(),
    ScrollAction.key: () => ScrollAction(),
  };

  @override
  _WidgetsAppState createState() => _WidgetsAppState();
}
```



## _WidgetsAppState

```dart
/// 注意：
/// 这里混入了 WidgetsBindingObserver，因此可以实现对各种 app 事件的观测
class _WidgetsAppState 
    extends State<WidgetsApp> 
    with WidgetsBindingObserver 
{
  // STATE LIFECYCLE

  @override
  void initState() {
    super.initState();
    _updateNavigator();
    _locale = _resolveLocales(WidgetsBinding.instance.window.locales, widget.supportedLocales);
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void didUpdateWidget(WidgetsApp oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.navigatorKey != oldWidget.navigatorKey)
      _updateNavigator();
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  // 导航
    
  // Navigator 持有的 GlobalKey，用于获取 Navigator 的 State
  GlobalKey<NavigatorState> _navigator;

  void _updateNavigator() {
   /// 如果 widget 中指定了一个 Globalkey，则使用它，
   /// 否则，新建一个 GlobalObjectKey 
    _navigator = widget.navigatorKey ?? GlobalObjectKey<NavigatorState>(this);
  }

  /// 
  Route<dynamic> _onGenerateRoute(RouteSettings settings) {
    final String name = settings.name;
      
    final WidgetBuilder pageContentBuilder = 
        (name == Navigator.defaultRouteName && widget.home != null)
        /// 如果是初始路由，并且给定了 home widget
        ? (BuildContext context) => widget.home
        /// 否则直接在路由表里查找
        : widget.routes[name];
    
    /// 如果获取到 WidgetBuilder
    /// 则会调用 pageRouteBuilder (PageRouteFactory) 来产生一个路由
    if (pageContentBuilder != null) {
      final Route<dynamic> route = widget.pageRouteBuilder<dynamic>(
        settings,
        pageContentBuilder,
      );
      return route;
    }
      
    /// 否则，调用 onGenerateRoute 来产生新路由
    if (widget.onGenerateRoute != null)
      return widget.onGenerateRoute(settings);
    
    /// 否则返回 null，即未找到 setting 对应的路由
    return null;
  }

  Route<dynamic> _onUnknownRoute(RouteSettings settings) {
    final Route<dynamic> result = widget.onUnknownRoute(settings);
    return result;
  }

  /// app 退出路由
  @override
  Future<bool> didPopRoute() async {
    assert(mounted);
    final NavigatorState navigator = _navigator?.currentState;
    if (navigator == null)
      return false;
    return await navigator.maybePop();
  }

  /// app 插入路由
  @override
  Future<bool> didPushRoute(String route) async {
    assert(mounted);
    final NavigatorState navigator = _navigator?.currentState;
    if (navigator == null)
      return false;
    navigator.pushNamed(route);
    return true;
  }

  // 本地化略
  
  // build 方法
  @override
  Widget build(BuildContext context) {
    Widget navigator;
    
    if (_navigator != null) {
      navigator = Navigator(
        key: _navigator,
        // If window.defaultRouteName isn't '/', we should assume it was set
        // intentionally via `setInitialRoute`, and should override whatever
        // is in [widget.initialRoute].
        initialRoute: 
            (WidgetsBinding.instance.window.defaultRouteName != Navigator.defaultRouteName)
            ? WidgetsBinding.instance.window.defaultRouteName
            : widget.initialRoute ?? WidgetsBinding.instance.window.defaultRouteName,
        /// 每当要插入一个新路由的时候，navigator 会给出 RouteSetting 来回调 _onGenerateRoute，产生一个 
        /// Route
        onGenerateRoute: _onGenerateRoute,
        onUnknownRoute: _onUnknownRoute,
        observers: widget.navigatorObservers,
      );
    }

    Widget result;
    if (widget.builder != null) {
      /// 如果提供了 builder，则套一层 Builder 确保 context 的正确性
      result = Builder(
        builder: (BuildContext context) {
          return widget.builder(context, navigator);
        },
      );
    } else {
      result = navigator;
    }

    /// 如果提供了默认 TextStyle，这里套一层 DefaultTextStyle
    if (widget.textStyle != null) {
      result = DefaultTextStyle(
        style: widget.textStyle,
        child: result,
      );
    }

    PerformanceOverlay performanceOverlay;
    // 性能图层
    if (widget.showPerformanceOverlay || WidgetsApp.showPerformanceOverlayOverride) {
      performanceOverlay = PerformanceOverlay.allEnabled(
        checkerboardRasterCacheImages: widget.checkerboardRasterCacheImages,
        checkerboardOffscreenLayers: widget.checkerboardOffscreenLayers,
      );
    } else if (widget.checkerboardRasterCacheImages || widget.checkerboardOffscreenLayers) {
      performanceOverlay = PerformanceOverlay(
        checkerboardRasterCacheImages: widget.checkerboardRasterCacheImages,
        checkerboardOffscreenLayers: widget.checkerboardOffscreenLayers,
      );
    }
    if (performanceOverlay != null) {
      /// 套 Stack 用于展示性能图层，由 Stack 的特性可知，总是显示在最上层
      result = Stack(
        children: <Widget>[
          result,
          Positioned(top: 0.0, left: 0.0, right: 0.0, child: performanceOverlay),
        ],
      );
    }

    if (widget.showSemanticsDebugger) {
      result = SemanticsDebugger(
        child: result,
      );
    }

    Widget title;
    if (widget.onGenerateTitle != null) {
      title = Builder(
        // This Builder exists to provide a context below the Localizations widget.
        // The onGenerateTitle callback can refer to Localizations via its context
        // parameter.
        builder: (BuildContext context) {
          final String title = widget.onGenerateTitle(context);
          assert(title != null, 'onGenerateTitle must return a non-null String');
          return Title(
            title: title,
            color: widget.color,
            child: result,
          );
        },
      );
    } else {
      title = Title(
        title: widget.title,
        color: widget.color,
        child: result,
      );
    }

    final Locale appLocale = widget.locale != null
      ? _resolveLocales(<Locale>[widget.locale], widget.supportedLocales)
      : _locale;

    assert(_debugCheckLocalizations(appLocale));
    return Shortcuts(
      shortcuts: widget.shortcuts ?? WidgetsApp.defaultShortcuts,
      child: Actions(
        actions: widget.actions ?? WidgetsApp.defaultActions,
        child: DefaultFocusTraversal(
          policy: ReadingOrderTraversalPolicy(),
          /// 注意：
          /// 这里套用了一层 _MediaQueryFromWindow，用于向子树提供 [MediaQuery]
          child: _MediaQueryFromWindow(
            child: Localizations(
              locale: appLocale,
              delegates: _localizationsDelegates.toList(),
              child: title,
            ),
          ),
        ),
      ),
    );
  }
}
```



## _MediaQueryFromWindow

```dart
/// Builds [MediaQuery] from `window` by listening to [WidgetsBinding].
///
/// It is performed in a standalone widget to rebuild **only** [MediaQuery] and
/// its dependents when `window` changes, instead of rebuilding the entire widget tree.

class _MediaQueryFromWindow extends StatefulWidget {
  const _MediaQueryFromWindow({Key key, this.child}) : super(key: key);

  final Widget child;

  @override
  _MediaQueryFromWindowsState createState() => _MediaQueryFromWindowsState();
}
```



## _MediaQueryFromWindowsState

```dart
class _MediaQueryFromWindowsState 
    extends State<_MediaQueryFromWindow> 
    with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

    
  /// 以下方法，当各种 app 事件（例如屏幕大小发生比变化）到来的时候，需要通知子树对事件做出相应
  /// 对应的方法实现都只是调用一次 setState，重新构建子树
  /// 由于 MediaQuery 的存在，所有依赖于 MediaQuery 的组件都会被重建
    
  // ACCESSIBILITY

  @override
  void didChangeAccessibilityFeatures() {
    setState(() {
    });
  }

  // METRICS

  @override
  void didChangeMetrics() {
    setState(() {
    });
  }

  @override
  void didChangeTextScaleFactor() {
    setState(() {
    });
  }

  // RENDERING
  @override
  void didChangePlatformBrightness() {
    setState(() {
    });
  }

  @override
  Widget build(BuildContext context) {
    /// 这里提供了 MediaQuery，每次重建时，都会向 window 实例获取一次数据，随后通知子树更新
    return MediaQuery(
      data: MediaQueryData.fromWindow(WidgetsBinding.instance.window),
      child: widget.child,
    );
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }
}
```
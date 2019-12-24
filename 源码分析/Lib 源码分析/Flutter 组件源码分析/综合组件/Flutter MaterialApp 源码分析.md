# Flutter MaterialApp 源码分析



## MaterialApp

```dart
/// 使用 Material Design 的 app
///
/// 一个常用的上层的用于提供 material design 配置的组件
/// 它基于 [WidgetsApp] 构建，并且添加了 material-design 特定的功能，如 [AnimatedTheme] 和 [GridPaper]
///
/// [MaterialApp] 配置了顶层的 [Navigator]，并以以下顺序，搜索路由
///
///  1.如果不为 null 则使用
///
///  2.否则, 如果 中有任何的路由，则使用它
///
///  3.否则, 如果提供了 [onGenerateRoute]，它将会被调用。 对于任何没有被 [home] 或者 [routes] 处理的路由，
///    它应该返回一个非空值
///
///  4.否则，[onUnknownRoute] 被调用
///
/// 这个组件也配置了顶层 [Navigator] 的观测者，用于运行 [hero] 动画
///
/// 如果 [home], [routes], [onGenerateRoute] 以及 [onUnknownRoute] 都是 null,
/// 并且 [builder] 是非 null 的，那么 [Navigator] 不会被创建
class MaterialApp extends StatefulWidget {
  const MaterialApp({
    Key key,
    this.navigatorKey,
    this.home,
    this.routes = const <String, WidgetBuilder>{},
    this.initialRoute,
    this.onGenerateRoute,
    this.onUnknownRoute,
    this.navigatorObservers = const <NavigatorObserver>[],
    this.builder,
    this.title = '',
    this.onGenerateTitle,
    this.color,
    this.theme,
    this.darkTheme,
    this.themeMode = ThemeMode.system,
    this.locale,
    this.localizationsDelegates,
    this.localeListResolutionCallback,
    this.localeResolutionCallback,
    this.supportedLocales = const <Locale>[Locale('en', 'US')],
    this.debugShowMaterialGrid = false,
    this.showPerformanceOverlay = false,
    this.checkerboardRasterCacheImages = false,
    this.checkerboardOffscreenLayers = false,
    this.showSemanticsDebugger = false,
    this.debugShowCheckedModeBanner = true,
    this.shortcuts,
    this.actions,
  }) : super(key: key);

  /// 提供给 Navigator 的 GlobalKey。如果是 null，则自动创建一个
  /// 外界可以主动提供这个 key，用于操作 navigator。例如，可以实现无 context 导航
  final GlobalKey<NavigatorState> navigatorKey;

  /// 首页
  final Widget home;

  ///  应用的顶层路由表
  ///
  /// 当使用 [Navigator.pushNamed] 推送一个命名路由入栈的时候, 在此表中查找路由名.
  /// 如果存在此路由名，相关联的 [WidgetBuilder] 将会被用于构建 [MaterialPageRoute]，它将会呈现合适的页面过
  /// 渡效果，包括 [hero]动画
  final Map<String, WidgetBuilder> routes;

  /// 初始路由名
  final String initialRoute;

  /// 路由工厂
  /// typedef RouteFactory = Route<dynamic> Function(RouteSettings settings);
  /// 即，通过给定的路由配置来展示一个路由
  final RouteFactory onGenerateRoute;

  /// 处理位置路由
  final RouteFactory onUnknownRoute;

  /// 导航观测者，通常会自动包含一个 HeroController
  final List<NavigatorObserver> navigatorObservers;

  /// Material Design 中特有的功能函数，比如 [showDialog] 和 [showMenu], 或者组件，比如 [Tooltip] 和
  /// [PopupMenuButton] 都需要一个 [Navigator] 来正确的运行
  final TransitionBuilder builder;

  /// 这个值直接传递给 [WidgetsApp.title].
  final String title;

  /// 逐个值直接传递给 [WidgetsApp.onGenerateTitle].
  /// typedef GenerateAppTitle = String Function(BuildContext context);
  final GenerateAppTitle onGenerateTitle;

  /// 对于 material widgets 来说，默认的视觉属性，例如颜色，字体，形状
  ///
  /// 其次能够指定一个 [darkTheme] [ThemeData] 值, 即一个黑夜版本的用户界面。
  ///
  /// 此值的默认值时 [ThemeData.light()].
  final ThemeData theme;
  final ThemeData darkTheme;

  /// Determines which theme will be used by the application if both [theme]
  /// and [darkTheme] are provided.
  ///
  /// If set to [ThemeMode.system], the choice of which theme to use will
  /// be based on the user's system preferences. If the [MediaQuery.platformBrightnessOf]
  /// is [Brightness.light], [theme] will be used. If it is [Brightness.dark],
  /// [darkTheme] will be used (unless it is [null], in which case [theme]
  /// will be used.
  ///
  /// If set to [ThemeMode.light] the [theme] will always be used,
  /// regardless of the user's system preference.
  ///
  /// If set to [ThemeMode.dark] the [darkTheme] will be used
  /// regardless of the user's system preference. If [darkTheme] is [null]
  /// then it will fallback to using [theme].
  ///
  /// The default value is [ThemeMode.system].
  final ThemeMode themeMode;

  /// {@macro flutter.widgets.widgetsApp.color}
  final Color color;

  /// 语言本地化
  final Locale locale;

  /// {@macro flutter.widgets.widgetsApp.localizationsDelegates}
  ///
  /// Internationalized apps that require translations for one of the locales
  /// listed in [GlobalMaterialLocalizations] should specify this parameter
  /// and list the [supportedLocales] that the application can handle.
  ///
  /// ```dart
  /// import 'package:flutter_localizations/flutter_localizations.dart';
  /// MaterialApp(
  ///   localizationsDelegates: [
  ///     // ... app-specific localization delegate[s] here
  ///     GlobalMaterialLocalizations.delegate,
  ///     GlobalWidgetsLocalizations.delegate,
  ///   ],
  ///   supportedLocales: [
  ///     const Locale('en', 'US'), // English
  ///     const Locale('he', 'IL'), // Hebrew
  ///     // ... other locales the app supports
  ///   ],
  ///   // ...
  /// )
  /// ```
  ///
  /// ## Adding localizations for a new locale
  ///
  /// The information that follows applies to the unusual case of an app
  /// adding translations for a language not already supported by
  /// [GlobalMaterialLocalizations].
  ///
  /// Delegates that produce [WidgetsLocalizations] and [MaterialLocalizations]
  /// are included automatically. Apps can provide their own versions of these
  /// localizations by creating implementations of
  /// [LocalizationsDelegate<WidgetsLocalizations>] or
  /// [LocalizationsDelegate<MaterialLocalizations>] whose load methods return
  /// custom versions of [WidgetsLocalizations] or [MaterialLocalizations].
  ///
  /// For example: to add support to [MaterialLocalizations] for a
  /// locale it doesn't already support, say `const Locale('foo', 'BR')`,
  /// one could just extend [DefaultMaterialLocalizations]:
  ///
  /// ```dart
  /// class FooLocalizations extends DefaultMaterialLocalizations {
  ///   FooLocalizations(Locale locale) : super(locale);
  ///   @override
  ///   String get okButtonLabel {
  ///     if (locale == const Locale('foo', 'BR'))
  ///       return 'foo';
  ///     return super.okButtonLabel;
  ///   }
  /// }
  ///
  /// ```
  ///
  /// A `FooLocalizationsDelegate` is essentially just a method that constructs
  /// a `FooLocalizations` object. We return a [SynchronousFuture] here because
  /// no asynchronous work takes place upon "loading" the localizations object.
  ///
  /// ```dart
  /// class FooLocalizationsDelegate extends LocalizationsDelegate<MaterialLocalizations> {
  ///   const FooLocalizationsDelegate();
  ///   @override
  ///   Future<FooLocalizations> load(Locale locale) {
  ///     return SynchronousFuture(FooLocalizations(locale));
  ///   }
  ///   @override
  ///   bool shouldReload(FooLocalizationsDelegate old) => false;
  /// }
  /// ```
  ///
  /// Constructing a [MaterialApp] with a `FooLocalizationsDelegate` overrides
  /// the automatically included delegate for [MaterialLocalizations] because
  /// only the first delegate of each [LocalizationsDelegate.type] is used and
  /// the automatically included delegates are added to the end of the app's
  /// [localizationsDelegates] list.
  ///
  /// ```dart
  /// MaterialApp(
  ///   localizationsDelegates: [
  ///     const FooLocalizationsDelegate(),
  ///   ],
  ///   // ...
  /// )
  /// ```
  /// See also:
  ///
  ///  * [supportedLocales], which must be specified along with
  ///    [localizationsDelegates].
  ///  * [GlobalMaterialLocalizations], a [localizationsDelegates] value
  ///    which provides material localizations for many languages.
  ///  * The Flutter Internationalization Tutorial,
  ///    <https://flutter.dev/tutorials/internationalization/>.
  final Iterable<LocalizationsDelegate<dynamic>> localizationsDelegates;

  /// {@macro flutter.widgets.widgetsApp.localeListResolutionCallback}
  ///
  /// This callback is passed along to the [WidgetsApp] built by this widget.
  final LocaleListResolutionCallback localeListResolutionCallback;

  /// {@macro flutter.widgets.widgetsApp.localeResolutionCallback}
  ///
  /// This callback is passed along to the [WidgetsApp] built by this widget.
  final LocaleResolutionCallback localeResolutionCallback;

  /// {@macro flutter.widgets.widgetsApp.supportedLocales}
  ///
  /// It is passed along unmodified to the [WidgetsApp] built by this widget.
  ///
  /// See also:
  ///
  ///  * [localizationsDelegates], which must be specified for localized
  ///    applications.
  ///  * [GlobalMaterialLocalizations], a [localizationsDelegates] value
  ///    which provides material localizations for many languages.
  ///  * The Flutter Internationalization Tutorial,
  ///    <https://flutter.dev/tutorials/internationalization/>.
  final Iterable<Locale> supportedLocales;

  /// Turns on a performance overlay.
  ///
  /// See also:
  ///
  ///  * <https://flutter.dev/debugging/#performanceoverlay>
  final bool showPerformanceOverlay;

  /// Turns on checkerboarding of raster cache images.
  final bool checkerboardRasterCacheImages;

  /// Turns on checkerboarding of layers rendered to offscreen bitmaps.
  final bool checkerboardOffscreenLayers;

  /// Turns on an overlay that shows the accessibility information
  /// reported by the framework.
  final bool showSemanticsDebugger;

  /// {@macro flutter.widgets.widgetsApp.debugShowCheckedModeBanner}
  final bool debugShowCheckedModeBanner;

  /// {@macro flutter.widgets.widgetsApp.shortcuts}
  /// {@tool sample}
  /// This example shows how to add a single shortcut for
  /// [LogicalKeyboardKey.select] to the default shortcuts without needing to
  /// add your own [Shortcuts] widget.
  ///
  /// Alternatively, you could insert a [Shortcuts] widget with just the mapping
  /// you want to add between the [WidgetsApp] and its child and get the same
  /// effect.
  ///
  /// ```dart
  /// Widget build(BuildContext context) {
  ///   return WidgetsApp(
  ///     shortcuts: <LogicalKeySet, Intent>{
  ///       ... WidgetsApp.defaultShortcuts,
  ///       LogicalKeySet(LogicalKeyboardKey.select): const Intent(ActivateAction.key),
  ///     },
  ///     color: const Color(0xFFFF0000),
  ///     builder: (BuildContext context, Widget child) {
  ///       return const Placeholder();
  ///     },
  ///   );
  /// }
  /// ```
  /// {@end-tool}
  /// {@macro flutter.widgets.widgetsApp.shortcuts.seeAlso}
  final Map<LogicalKeySet, Intent> shortcuts;

  /// {@macro flutter.widgets.widgetsApp.actions}
  /// {@tool sample}
  /// This example shows how to add a single action handling an
  /// [ActivateAction] to the default actions without needing to
  /// add your own [Actions] widget.
  ///
  /// Alternatively, you could insert a [Actions] widget with just the mapping
  /// you want to add between the [WidgetsApp] and its child and get the same
  /// effect.
  ///
  /// ```dart
  /// Widget build(BuildContext context) {
  ///   return WidgetsApp(
  ///     actions: <LocalKey, ActionFactory>{
  ///       ... WidgetsApp.defaultActions,
  ///       ActivateAction.key: () => CallbackAction(
  ///         ActivateAction.key,
  ///         onInvoke: (FocusNode focusNode, Intent intent) {
  ///           // Do something here...
  ///         },
  ///       ),
  ///     },
  ///     color: const Color(0xFFFF0000),
  ///     builder: (BuildContext context, Widget child) {
  ///       return const Placeholder();
  ///     },
  ///   );
  /// }
  /// ```
  /// {@end-tool}
  /// {@macro flutter.widgets.widgetsApp.actions.seeAlso}
  final Map<LocalKey, ActionFactory> actions;

  /// 启用 [GridPaper] 浮层 绘制了基线网格
  ///
  /// 只在测试模式下有效
  final bool debugShowMaterialGrid;

  @override
  _MaterialAppState createState() => _MaterialAppState();
}
```



## _MaterialScrollBehavior

```dart
class _MaterialScrollBehavior extends ScrollBehavior {
  @override
  TargetPlatform getPlatform(BuildContext context) {
    return Theme.of(context).platform;
  }

  @override
  Widget buildViewportChrome(BuildContext context, Widget child, AxisDirection axisDirection) {
    /// 根据平台来选择滑动行为
    switch (getPlatform(context)) {
      /// 默认的弹簧滑动
      case TargetPlatform.iOS:
      case TargetPlatform.macOS:
        return child;
      /// android 和 fuchsia 采用水波纹效果 
      case TargetPlatform.android:
      case TargetPlatform.fuchsia:
        return GlowingOverscrollIndicator(
          child: child,
          axisDirection: axisDirection,
          color: Theme.of(context).accentColor,
        );
    }
    return null;
  }
}
```



## _MaterialAppState

```dart
class _MaterialAppState extends State<MaterialApp> {
  HeroController _heroController;

  @override
  void initState() {
    super.initState();
    _heroController = HeroController(createRectTween: _createRectTween);
    _updateNavigator();
  }

  @override
  void didUpdateWidget(MaterialApp oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.navigatorKey != oldWidget.navigatorKey) {
      // 如果 navigator 发生改变, 我们必须创建一个新的观测者, 因为旧的 navigator 不会被销毁（因此也不会注销
      // 它的观测者），直到新的 navigator 被创建完毕（因为 navigator 持有 globalKey）。
      // 这里更新了 heroController
      _heroController = HeroController(createRectTween: _createRectTween);
    }
    _updateNavigator();
  }

  List<NavigatorObserver> _navigatorObservers;

  void _updateNavigator() {
    /// 如果以下属性都不为 null 的话，会自动创建一个 navigator
    if (widget.home != null ||
        widget.routes.isNotEmpty ||
        widget.onGenerateRoute != null ||
        widget.onUnknownRoute != null) {
      _navigatorObservers = List<NavigatorObserver>.from(widget.navigatorObservers)
        ..add(_heroController);
    /// 否则，将观测者清空（因为此时不会自动创建一个 navigator）
    } else {
      _navigatorObservers = const <NavigatorObserver>[];
    }
  }

  /// 被 Hero 使用，用于创建过渡动画的轨迹
  RectTween _createRectTween(Rect begin, Rect end) {
    return MaterialRectArcTween(begin: begin, end: end);
  }

  // Combine the Localizations for Material with the ones contributed
  // by the localizationsDelegates parameter, if any. Only the first delegate
  // of a particular LocalizationsDelegate.type is loaded so the
  // localizationsDelegate parameter can be used to override
  // _MaterialLocalizationsDelegate.
  Iterable<LocalizationsDelegate<dynamic>> get _localizationsDelegates sync* {
    if (widget.localizationsDelegates != null)
      yield* widget.localizationsDelegates;
    yield DefaultMaterialLocalizations.delegate;
    yield DefaultCupertinoLocalizations.delegate;
  }

  @override
  Widget build(BuildContext context) {
    Widget result = WidgetsApp(
      key: GlobalObjectKey(this),
      navigatorKey: widget.navigatorKey,
      navigatorObservers: _navigatorObservers,
      pageRouteBuilder: <T>(RouteSettings settings, WidgetBuilder builder) {
        return MaterialPageRoute<T>(settings: settings, builder: builder);
      },
      home: widget.home,
      routes: widget.routes,
      initialRoute: widget.initialRoute,
      onGenerateRoute: widget.onGenerateRoute,
      onUnknownRoute: widget.onUnknownRoute,
      builder: (BuildContext context, Widget child) {
        // Use a light theme, dark theme, or fallback theme.
        final ThemeMode mode = widget.themeMode ?? ThemeMode.system;
        ThemeData theme;
        if (widget.darkTheme != null) {
          final ui.Brightness platformBrightness = MediaQuery.platformBrightnessOf(context);
          if (mode == ThemeMode.dark ||
              (mode == ThemeMode.system && platformBrightness == ui.Brightness.dark)) {
            theme = widget.darkTheme;
          }
        }
        theme ??= widget.theme ?? ThemeData.fallback();

        return AnimatedTheme(
          data: theme,
          isMaterialAppTheme: true,
          child: widget.builder != null
              ? Builder(
                  builder: (BuildContext context) {
                    // Why are we surrounding a builder with a builder?
                    //
                    // The widget.builder may contain code that invokes
                    // Theme.of(), which should return the theme we selected
                    // above in AnimatedTheme. However, if we invoke
                    // widget.builder() directly as the child of AnimatedTheme
                    // then there is no Context separating them, and the
                    // widget.builder() will not find the theme. Therefore, we
                    // surround widget.builder with yet another builder so that
                    // a context separates them and Theme.of() correctly
                    // resolves to the theme we passed to AnimatedTheme.
                    return widget.builder(context, child);
                  },
                )
              : child,
        );
      },
      title: widget.title,
      onGenerateTitle: widget.onGenerateTitle,
      textStyle: _errorTextStyle,
      // The color property is always pulled from the light theme, even if dark
      // mode is activated. This was done to simplify the technical details
      // of switching themes and it was deemed acceptable because this color
      // property is only used on old Android OSes to color the app bar in
      // Android's switcher UI.
      //
      // blue is the primary color of the default theme
      color: widget.color ?? widget.theme?.primaryColor ?? Colors.blue,
      locale: widget.locale,
      localizationsDelegates: _localizationsDelegates,
      localeResolutionCallback: widget.localeResolutionCallback,
      localeListResolutionCallback: widget.localeListResolutionCallback,
      supportedLocales: widget.supportedLocales,
      showPerformanceOverlay: widget.showPerformanceOverlay,
      checkerboardRasterCacheImages: widget.checkerboardRasterCacheImages,
      checkerboardOffscreenLayers: widget.checkerboardOffscreenLayers,
      showSemanticsDebugger: widget.showSemanticsDebugger,
      debugShowCheckedModeBanner: widget.debugShowCheckedModeBanner,
      inspectorSelectButtonBuilder: (BuildContext context, VoidCallback onPressed) {
        return FloatingActionButton(
          child: const Icon(Icons.search),
          onPressed: onPressed,
          mini: true,
        );
      },
      shortcuts: widget.shortcuts,
      actions: widget.actions,
    );

    return ScrollConfiguration(
      behavior: _MaterialScrollBehavior(),
      child: result,
    );
  }
}

```


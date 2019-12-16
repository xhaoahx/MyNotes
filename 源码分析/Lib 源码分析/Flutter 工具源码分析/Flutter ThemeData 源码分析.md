# Flutter ThemeData 源码分析
```dart
  /// APP的全局亮度。被button之类的widget使用来决定颜色的使用（不使用原始或强调色时）。
  ///
  /// 当[Brightness]为dark时，画布、卡片和原色都是暗色。
  /// 当[Brightness]是light，画布和卡片的颜色都是亮色。
  /// 原色的暗度根据 primaryColorBrightness所描述的不同而不同
  /// 当亮度较暗时，卡片和画布的颜色，原色和底色的对比度不太好
  /// 当亮度为深色，使用颜色。白色或对比色的强调色。

  /// enum Brightness {
  /// 当亮度为dark时，文字颜色会变亮来达到 可阅读性对比。
  /// 例如，颜色变成灰色，字体变成白色。
  ///    dark,

  /// 当亮度为light时，文字颜色会变暗来达到 可阅读性对比。
  /// 例如，颜色为白色时，字体变成黑色。
  ///    light,
  /// }	
  final Brightness brightness;
  
  /// App主要部分的背景颜色（工具栏、按钮栏等等）。
  /// 
  /// 主题的[colorScheme]属性包含[ColorScheme.primary]),
  /// 以及一种与其形成鲜明对比的颜色[ColorScheme.onPrimary]。
  /// 依据[colorScheme]来配置App的视觉效果可能更加简便。

  final Color primaryColor;

  /// [primaryColor]的亮度。被用于决定放置在primary上的字体和文字的颜色（例如 toolbar text）。
  final Brightness primaryColorBrightness;

  /// 比[primaryColor]更加明亮的颜色版本。
  final Color primaryColorLight;

  /// 比[primaryColor]更加暗的颜色版本。
  final Color primaryColorDark;

  /// [MaterialType.canvas] [Material]的默认颜色.
  final Color canvasColor;

  /// widgets 的前景色 (knobs, text, overscroll edge effect（滑动水波颜色）等等)。
  ///
  /// 强调色就是二级颜色
  ///
  /// 主题的[colorScheme]属性包含[ColorScheme.secondary]),
  /// 以及一种与其色形成鲜明对比的颜色[ColorScheme.onSecondary]。
  /// 依据[colorScheme]来配置App的视觉效果可能更加简便。
  final Color accentColor;

  /// [accentColor]的亮度。被用于决定放置在accent color之上的字体和图标的颜色。
  /// 例如浮动按钮的图标颜色
  final Brightness accentColorBrightness;

  /// [Scaffold]下的[Material]的颜色。
  /// material app 或者 App 中的某一页的背景色。
  final Color scaffoldBackgroundColor;

  /// [BottomAppBar]的默认颜色.
  ///
  /// 这个能被重载[BottomAppBar.color]改变。
  final Color bottomAppBarColor;

  /// [Card]的[Material]的颜色。（卡片背景色）
  final Color cardColor;

  /// [Divider] 和 [PopupMenuDivider] 的颜色, 也被用于
  /// [ListTile]之间, [DataTable]的行之间的分割线颜色。
  ///
  /// 为了创建一个使用此颜色的[BorderSide], 考虑使用[Divider.createBorderSide]。
  final Color dividerColor;

  /// 一个被于显示组件获取到了输入焦点的的颜色。（例如输入框下划线颜色）
  final Color focusColor;

  /// 当指针悬浮到组件上，显示的颜色（？？似乎更适用于网页）。
  final Color hoverColor;

  /// 在inkWell动画中使用的高亮颜色，表示在菜单中选择一个项目。
  final Color highlightColor;

  /// splashes的颜（水波扩散颜色）. See [InkWell].
  /// 以下为InkResponse中的注释：
  /// The splash color of the ink response. If this property is null then the
  /// splash color of the theme, [ThemeData.splashColor], will be used.
  final Color splashColor;

  /// 定义了 [InkWell]和[InkResponse]产生的splashes的外观。
  final InteractiveInkFeatureFactory splashFactory;

  /// 高亮选中行的颜色.
  final Color selectedRowColor;

  /// widgets在未激活（但是启用状态）的颜色。例如：没有选中的选择框。
  /// 通常与 [accentColor]形成对照
  final Color unselectedWidgetColor;

  /// widgets在未启用的颜色。例如：未启用的选择框（无论是否选中）。
  final Color disabledColor;

  /// 按钮主题，例如[RaisedButton]和[FlatButton]的默认主题。
  final ButtonThemeData buttonTheme;

  /// [RaisedButton] 中的 [Material]的默认颜色。
  final Color buttonColor;

  /// [PaginatedDataTable]被选中时的标题的颜色。
  final Color secondaryHeaderColor;

  /// text fields被选中的文字的颜色, 例如[TextField]。
  final Color textSelectionColor;

  /// Material-style text fields的光标颜色, 例如[TextField]。
  final Color cursorColor;

  /// 文字选择器的颜色（长按文字之后，控制复制范围的光标的颜色）。
  final Color textSelectionHandleColor;

  /// 与[primaryColor]形成对照的颜色, progress bar的剩余部分的颜色。
  final Color backgroundColor;

  /// 对话框[Dialog]的背景。
  final Color dialogBackgroundColor;

  /// tab bar 选中的 tab 的指示器颜色。
  final Color indicatorColor;

  /// text fields中提示文字和占位文字的颜色。例如[TextField]。
  final Color hintColor;

  /// input validation errors的颜色（例如密码长度不够）, 例如[TextField]。
  final Color errorColor;

  /// 用于高亮可触发控件（例如[Switch], [Radio], and [Checkbox]）的激活状态的颜色。
  final Color toggleableActiveColor;

  /// 用于和卡片、画布形成的对照的文字颜色。
  final TextTheme textTheme;

  /// 与原始色形成对照的文字颜色。
  final TextTheme primaryTextTheme;

  /// 与强调色形成对照的文字颜色。
  final TextTheme accentTextTheme;

  /// [InputDecorator]的默认的[InputDecoration]。
  /// [TextField] 和 [TextFormField] 都基于这个主题。
  final InputDecorationTheme inputDecorationTheme;

  /// 用于和卡片、画布形成的对照的图标颜色。
  final IconThemeData iconTheme;

  /// 与原始色形成对照的图标颜色。
  final IconThemeData primaryIconTheme;

  /// 与强调色形成对照的图标颜色。
  final IconThemeData accentIconTheme;

  /// 用于渲染[Slider]的颜色和形状。
  ///
  /// 用[SliderTheme.of]获取这个值。
  final SliderThemeData sliderTheme;

  /// 一个自定义tab bar indicator的颜色大小形状的主题。
  final TabBarTheme tabBarTheme;

  /// 用于渲染[Card]的性质和颜色。
  ///
  /// 用[CardTheme.of]获取这个值。
  final CardTheme cardTheme;

  /// 用于渲染[Chip]的颜色形状。
  ///
  /// 用[ChipTheme.of]获取这个值。
  final ChipThemeData chipTheme;

  /// material widgets 应该适配的目标平台。
  ///
  /// 默认为当前平台。UI元素会符合平台传统。
  /// 
  /// [Platform.defaultTargetPlatform]应该直接使用，而不是只在在极少数依赖于平台的情况下。
  /// [dart.io.Platform.environment]应该在需要知道当前平台的关键时刻使用。而不应该进行任何重载。
  ///（例如，当系统API即将被调用时）。
  final TargetPlatform platform;

  /// 配置某个Material widget的点击测试范围。
  final MaterialTapTargetSize materialTapTargetSize;
  /// enum MaterialTapTargetSize {
  /// 向外扩展48px
  ///
  /// 这是[ThemeData]的默认值。materialHitTestSize)和建议尺寸。
  /// 符合Android易访问性扫描仪建议。
  ///    padded,

  /// 紧缩
  ///    shrinkWrap,
  /// }

  /// 默认转页面换动画主题。依据平台。
  final PageTransitionsTheme pageTransitionsTheme;

  /// 用于自定义[AppBar]的颜色、阴影、亮度、图标主题、文字主题。
  final AppBarTheme appBarTheme;

  /// 用于自定义[BottomAppBar]的颜色、阴影、形状。
  final BottomAppBarTheme bottomAppBarTheme;

  /// 使用一个十三种颜色的集合来配置绝大多数的组件的颜色。
  /// 
  /// 这个属性是在主题的high集合之后添加的
  /// 特定的颜色，如[cardColor]， [buttonColor]， [canvasColor]等。
  /// 可以根据[colorScheme]定义新的组件。
  /// 现有的组件将在一定程度上逐渐迁移到它
  /// 在没有明显的向后兼容性中断的情况下，这是可能的
  final ColorScheme colorScheme;

  /// 用于自定义[SnackBar]颜色、阴影、形状、行为。
  final SnackBarThemeData snackBarTheme;

  /// 自定义dialog的形状。
  final DialogTheme dialogTheme;

  /// 用于自定义[FloatingActionButton]的形状、阴影、颜色。
  final FloatingActionButtonThemeData floatingActionButtonTheme;

  /// [TextTheme] 的几何、颜色值，被用于配置[textTheme]、[primaryTextTheme], 和     
  /// [accentTextTheme]。
  final Typography typography;

  /// 组件的[CupertinoThemeData]自适应[ThemeData]
  ///
  /// 默认情况下，[cupertinoOverrideTheme]是null。并且Cupertino widgets
  /// 后代[MaterialTheme]将保持同一个[CupertinoTheme]（源自[MeteialData]）
  /// 例如[ThemeData]和[ColorTheme]。
  /// 还将通知[CupertinoThemeData]的‘primary color’等。
  ///
  /// 对于[CupertinoThemeData]的单个属性的级联，
  /// 可以使用这个[cupertinoOverrideTheme]的属性重写。
  final CupertinoThemeData cupertinoOverrideTheme;

  /// 用于自定义 bottom sheet 的颜色、阴影和形状。
  final BottomSheetThemeData bottomSheetTheme;
```


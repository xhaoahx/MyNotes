# Flutter Scaffold 源码分析

[TOC]

## ScaffoldPrelayoutGeometry 

```dart
class ScaffoldPrelayoutGeometry {
  /// Abstract const constructor. This constructor enables subclasses to provide
  /// const constructors so that they can be used in const expressions.
  const ScaffoldPrelayoutGeometry({
    @required this.bottomSheetSize,
    @required this.contentBottom,
    @required this.contentTop,
    @required this.floatingActionButtonSize,
    @required this.minInsets,
    @required this.scaffoldSize,
    @required this.snackBarSize,
    @required this.textDirection,
  });

  /// The [Size] of [Scaffold.floatingActionButton].
  ///
  /// If [Scaffold.floatingActionButton] is null, this will be [Size.zero].
  final Size floatingActionButtonSize;

  /// The [Size] of the [Scaffold]'s [BottomSheet].
  ///
  /// If the [Scaffold] is not currently showing a [BottomSheet],
  /// this will be [Size.zero].
  final Size bottomSheetSize;

  /// The vertical distance from the Scaffold's origin to the bottom of
  /// [Scaffold.body].
  ///
  /// This is useful in a [FloatingActionButtonLocation] designed to
  /// place the [FloatingActionButton] at the bottom of the screen, while
  /// keeping it above the [BottomSheet], the [Scaffold.bottomNavigationBar],
  /// or the keyboard.
  ///
  /// The [Scaffold.body] is laid out with respect to [minInsets] already. This
  /// means that a [FloatingActionButtonLocation] does not need to factor in
  /// [minInsets.bottom] when aligning a [FloatingActionButton] to
  /// [contentBottom].
  final double contentBottom;

  /// The vertical distance from the [Scaffold]'s origin to the top of
  /// [Scaffold.body].
  ///
  /// This is useful in a [FloatingActionButtonLocation] designed to
  /// place the [FloatingActionButton] at the top of the screen, while
  /// keeping it below the [Scaffold.appBar].
  ///
  /// The [Scaffold.body] is laid out with respect to [minInsets] already. This
  /// means that a [FloatingActionButtonLocation] does not need to factor in
  /// [minInsets.top] when aligning a [FloatingActionButton] to [contentTop].
  final double contentTop;

  /// The minimum padding to inset the [FloatingActionButton] by for it
  /// to remain visible.
  ///
  /// This value is the result of calling [MediaQuery.padding] in the
  /// [Scaffold]'s [BuildContext],
  /// and is useful for insetting the [FloatingActionButton] to avoid features like
  /// the system status bar or the keyboard.
  ///
  /// If [Scaffold.resizeToAvoidBottomInset] is set to false, [minInsets.bottom]
  /// will be 0.0.
  final EdgeInsets minInsets;

  /// The [Size] of the whole [Scaffold].
  ///
  /// If the [Size] of the [Scaffold]'s contents is modified by values such as
  /// [Scaffold.resizeToAvoidBottomInset] or the keyboard opening, then the
  /// [scaffoldSize] will not reflect those changes.
  ///
  /// This means that [FloatingActionButtonLocation]s designed to reposition
  /// the [FloatingActionButton] based on events such as the keyboard popping
  /// up should use [minInsets] to make sure that the [FloatingActionButton] is
  /// inset by enough to remain visible.
  ///
  /// See [minInsets] and [MediaQuery.padding] for more information on the appropriate
  /// insets to apply.
  final Size scaffoldSize;

  /// The [Size] of the [Scaffold]'s [SnackBar].
  ///
  /// If the [Scaffold] is not showing a [SnackBar], this will be [Size.zero].
  final Size snackBarSize;

  /// The [TextDirection] of the [Scaffold]'s [BuildContext].
  final TextDirection textDirection;
}
```



## ScaffoldGeometry

```dart
class ScaffoldGeometry {
  /// Create an object that describes the geometry of a [Scaffold].
  const ScaffoldGeometry({
    this.bottomNavigationBarTop,
    this.floatingActionButtonArea,
  });

  /// The distance from the [Scaffold]'s top edge to the top edge of the
  /// rectangle in which the [Scaffold.bottomNavigationBar] bar is laid out.
  ///
  /// Null if [Scaffold.bottomNavigationBar] is null.
  final double bottomNavigationBarTop;

  /// The [Scaffold.floatingActionButton]'s bounding rectangle.
  ///
  /// This is null when there is no floating action button showing.
  final Rect floatingActionButtonArea;

  ScaffoldGeometry _scaleFloatingActionButton(double scaleFactor) {
    if (scaleFactor == 1.0)
      return this;

    if (scaleFactor == 0.0) {
      return ScaffoldGeometry(
        bottomNavigationBarTop: bottomNavigationBarTop,
      );
    }

    final Rect scaledButton = Rect.lerp(
      floatingActionButtonArea.center & Size.zero,
      floatingActionButtonArea,
      scaleFactor,
    );
    return copyWith(floatingActionButtonArea: scaledButton);
  }

  /// Creates a copy of this [ScaffoldGeometry] but with the given fields replaced with
  /// the new values.
  ScaffoldGeometry copyWith({
    double bottomNavigationBarTop,
    Rect floatingActionButtonArea,
  }) {
    return ScaffoldGeometry(
      bottomNavigationBarTop: bottomNavigationBarTop ?? this.bottomNavigationBarTop,
      floatingActionButtonArea: floatingActionButtonArea ?? this.floatingActionButtonArea,
    );
  }
}
```



## _BodyBoxConstraints

```dart
/// 拓展了BoxConstraint，要求提供底部Widget高度和appbar高度
class _BodyBoxConstraints extends BoxConstraints {
  const _BodyBoxConstraints({
    double minWidth = 0.0,
    double maxWidth = double.infinity,
    double minHeight = 0.0,
    double maxHeight = double.infinity,
    @required this.bottomWidgetsHeight,
    @required this.appBarHeight,
  }) : assert(bottomWidgetsHeight != null),
       assert(bottomWidgetsHeight >= 0),
       assert(appBarHeight != null),
       assert(appBarHeight >= 0),
       super(minWidth: minWidth, maxWidth: maxWidth, minHeight: minHeight, maxHeight: maxHeight);

  final double bottomWidgetsHeight;
  final double appBarHeight;
  ...  
}
```



## _ScaffoldLayout

```dart
/// 布局代理，整个Scaffold的布局算法
class _ScaffoldLayout extends MultiChildLayoutDelegate {
  _ScaffoldLayout({
    @required this.minInsets,
    @required this.textDirection,
    @required this.geometryNotifier,
    // for floating action button
    @required this.previousFloatingActionButtonLocation,
    @required this.currentFloatingActionButtonLocation,
    @required this.floatingActionButtonMoveAnimationProgress,
    @required this.floatingActionButtonMotionAnimator,
    @required this.isSnackBarFloating,
    @required this.extendBody,
    @required this.extendBodyBehindAppBar,
  });

  final bool extendBody;
  final bool extendBodyBehindAppBar;
  final EdgeInsets minInsets;
  final TextDirection textDirection;
  final _ScaffoldGeometryNotifier geometryNotifier;

  final FloatingActionButtonLocation previousFloatingActionButtonLocation;
  final FloatingActionButtonLocation currentFloatingActionButtonLocation;
  final double floatingActionButtonMoveAnimationProgress;
  final FloatingActionButtonAnimator floatingActionButtonMotionAnimator;

  final bool isSnackBarFloating;

  @override
  void performLayout(Size size) {
    /// 将约束宽松  
    final BoxConstraints looseConstraints = BoxConstraints.loose(size);
    final BoxConstraints fullWidthConstraints = 
        looseConstraints.tighten(width: size.width);
        
    final double bottom = size.height;
    double contentTop = 0.0;
    double bottomWidgetsHeight = 0.0;
    double appBarHeight = 0.0;

    if (hasChild(_ScaffoldSlot.appBar)) {
      appBarHeight = layoutChild(_ScaffoldSlot.appBar, fullWidthConstraints).height;
      contentTop = extendBodyBehindAppBar ? 0.0 : appBarHeight;
      positionChild(_ScaffoldSlot.appBar, Offset.zero);
    }

    double bottomNavigationBarTop;
    if (hasChild(_ScaffoldSlot.bottomNavigationBar)) {
      final double bottomNavigationBarHeight = 
          layoutChild(_ScaffoldSlot.bottomNavigationBar, fullWidthConstraints).height;
      bottomWidgetsHeight += bottomNavigationBarHeight;
      bottomNavigationBarTop = math.max(0.0, bottom - bottomWidgetsHeight);
      positionChild(_ScaffoldSlot.bottomNavigationBar, 
                    Offset(0.0, bottomNavigationBarTop)
                   );
    }

    if (hasChild(_ScaffoldSlot.persistentFooter)) {
      final BoxConstraints footerConstraints = BoxConstraints(
        maxWidth: fullWidthConstraints.maxWidth,
        maxHeight: math.max(0.0, bottom - bottomWidgetsHeight - contentTop),
      );
      final double persistentFooterHeight = 
          layoutChild(_ScaffoldSlot.persistentFooter, footerConstraints).height;
      bottomWidgetsHeight += persistentFooterHeight;
      positionChild(_ScaffoldSlot.persistentFooter, 
                    Offset(0.0, math.max(0.0, bottom - bottomWidgetsHeight))
                   );
    }

    // Set the content bottom to account for the greater of the height of any
    // bottom-anchored material widgets or of the keyboard or other
    // bottom-anchored system UI.
    final double contentBottom = 
        math.max(0.0, bottom - math.max(minInsets.bottom, bottomWidgetsHeight));

    if (hasChild(_ScaffoldSlot.body)) {
      double bodyMaxHeight = math.max(0.0, contentBottom - contentTop);

      if (extendBody) {
        bodyMaxHeight += bottomWidgetsHeight;
        bodyMaxHeight = bodyMaxHeight.clamp(
            0.0, looseConstraints.maxHeight - contentTop
        ).toDouble();
        assert(bodyMaxHeight <= math.max(0.0, looseConstraints.maxHeight - contentTop));
      }

      final BoxConstraints bodyConstraints = _BodyBoxConstraints(
        maxWidth: fullWidthConstraints.maxWidth,
        maxHeight: bodyMaxHeight,
        bottomWidgetsHeight: extendBody ? bottomWidgetsHeight : 0.0,
        appBarHeight: appBarHeight,
      );
      layoutChild(_ScaffoldSlot.body, bodyConstraints);
      positionChild(_ScaffoldSlot.body, Offset(0.0, contentTop));
    }

    // The BottomSheet and the SnackBar are anchored to the bottom of the parent,
    // they're as wide as the parent and are given their intrinsic height. The
    // only difference is that SnackBar appears on the top side of the
    // BottomNavigationBar while the BottomSheet is stacked on top of it.
    //
    // If all three elements are present then either the center of the FAB straddles
    // the top edge of the BottomSheet or the bottom of the FAB is
    // kFloatingActionButtonMargin above the SnackBar, whichever puts the FAB
    // the farthest above the bottom of the parent. If only the FAB is has a
    // non-zero height then it's inset from the parent's right and bottom edges
    // by kFloatingActionButtonMargin.

    Size bottomSheetSize = Size.zero;
    Size snackBarSize = Size.zero;
    if (hasChild(_ScaffoldSlot.bodyScrim)) {
      final BoxConstraints bottomSheetScrimConstraints = BoxConstraints(
        maxWidth: fullWidthConstraints.maxWidth,
        maxHeight: contentBottom,
      );
      layoutChild(_ScaffoldSlot.bodyScrim, bottomSheetScrimConstraints);
      positionChild(_ScaffoldSlot.bodyScrim, Offset.zero);
    }

    // Set the size of the SnackBar early if the behavior is fixed so
    // the FAB can be positioned correctly.
    if (hasChild(_ScaffoldSlot.snackBar) && !isSnackBarFloating) {
      snackBarSize = layoutChild(_ScaffoldSlot.snackBar, fullWidthConstraints);
    }

    if (hasChild(_ScaffoldSlot.bottomSheet)) {
      final BoxConstraints bottomSheetConstraints = BoxConstraints(
        maxWidth: fullWidthConstraints.maxWidth,
        maxHeight: math.max(0.0, contentBottom - contentTop),
      );
      bottomSheetSize = layoutChild(_ScaffoldSlot.bottomSheet, bottomSheetConstraints);
      positionChild(_ScaffoldSlot.bottomSheet, Offset((size.width - bottomSheetSize.width) / 2.0, contentBottom - bottomSheetSize.height));
    }

    Rect floatingActionButtonRect;
    if (hasChild(_ScaffoldSlot.floatingActionButton)) {
      final Size fabSize = layoutChild(_ScaffoldSlot.floatingActionButton, looseConstraints);

      // To account for the FAB position being changed, we'll animate between
      // the old and new positions.
      final ScaffoldPrelayoutGeometry currentGeometry = ScaffoldPrelayoutGeometry(
        bottomSheetSize: bottomSheetSize,
        contentBottom: contentBottom,
        contentTop: contentTop,
        floatingActionButtonSize: fabSize,
        minInsets: minInsets,
        scaffoldSize: size,
        snackBarSize: snackBarSize,
        textDirection: textDirection,
      );
      final Offset currentFabOffset = currentFloatingActionButtonLocation.getOffset(currentGeometry);
      final Offset previousFabOffset = previousFloatingActionButtonLocation.getOffset(currentGeometry);
      final Offset fabOffset = floatingActionButtonMotionAnimator.getOffset(
        begin: previousFabOffset,
        end: currentFabOffset,
        progress: floatingActionButtonMoveAnimationProgress,
      );
      positionChild(_ScaffoldSlot.floatingActionButton, fabOffset);
      floatingActionButtonRect = fabOffset & fabSize;
    }

    if (hasChild(_ScaffoldSlot.snackBar)) {
      if (snackBarSize == Size.zero) {
        snackBarSize = layoutChild(_ScaffoldSlot.snackBar, fullWidthConstraints);
      }
      final double snackBarYOffsetBase = floatingActionButtonRect != null && isSnackBarFloating
        ? floatingActionButtonRect.top
        : contentBottom;
      positionChild(_ScaffoldSlot.snackBar, Offset(0.0, snackBarYOffsetBase - snackBarSize.height));
    }

    if (hasChild(_ScaffoldSlot.statusBar)) {
      layoutChild(_ScaffoldSlot.statusBar, fullWidthConstraints.tighten(height: minInsets.top));
      positionChild(_ScaffoldSlot.statusBar, Offset.zero);
    }

    if (hasChild(_ScaffoldSlot.drawer)) {
      layoutChild(_ScaffoldSlot.drawer, BoxConstraints.tight(size));
      positionChild(_ScaffoldSlot.drawer, Offset.zero);
    }

    if (hasChild(_ScaffoldSlot.endDrawer)) {
      layoutChild(_ScaffoldSlot.endDrawer, BoxConstraints.tight(size));
      positionChild(_ScaffoldSlot.endDrawer, Offset.zero);
    }

    geometryNotifier._updateWith(
      bottomNavigationBarTop: bottomNavigationBarTop,
      floatingActionButtonArea: floatingActionButtonRect,
    );
  }

  @override
  bool shouldRelayout(_ScaffoldLayout oldDelegate) {
    return oldDelegate.minInsets != minInsets
        || oldDelegate.textDirection != textDirection
        || oldDelegate.floatingActionButtonMoveAnimationProgress 
           != floatingActionButtonMoveAnimationProgress
        || oldDelegate.previousFloatingActionButtonLocation 
           != previousFloatingActionButtonLocation
        || oldDelegate.currentFloatingActionButtonLocation 
           != currentFloatingActionButtonLocation
        || oldDelegate.extendBody != extendBody
        || oldDelegate.extendBodyBehindAppBar != extendBodyBehindAppBar;
  }
}
```



## Scaffold

```dart
class Scaffold extends StatefulWidget {
  /// Creates a visual scaffold for material design widgets.
  const Scaffold({
    Key key,
    this.appBar,
    this.body,
    this.floatingActionButton,
    this.floatingActionButtonLocation,
    this.floatingActionButtonAnimator,
    this.persistentFooterButtons,
    this.drawer,
    this.endDrawer,
    this.bottomNavigationBar,
    this.bottomSheet,
    this.backgroundColor,
    this.resizeToAvoidBottomPadding,
    this.resizeToAvoidBottomInset,
    this.primary = true,
    this.drawerDragStartBehavior = DragStartBehavior.start,
    this.extendBody = false,
    this.extendBodyBehindAppBar = false,
    this.drawerScrimColor,
    this.drawerEdgeDragWidth,
  }) : assert(primary != null),
       assert(extendBody != null),
       assert(extendBodyBehindAppBar != null),
       assert(drawerDragStartBehavior != null),
       super(key: key);

  /// 以下两个属性可以模拟出ios的半透明导航栏效果  
  /// 是否将body拓展到navigationbar顶部以下
  final bool extendBody;
  /// 是否将body拓展到appbar底部以上
  final bool extendBodyBehindAppBar;

  /// appbar
  final PreferredSizeWidget appBar;

  final Widget body;

  final Widget floatingActionButton;

  /// 浮动按钮的位置
  final FloatingActionButtonLocation floatingActionButtonLocation;

  /// 将 [floatingActionButton] 移动到一个新的 [floatingActionButtonLocation].
  ///
  /// 默认使用[FloatingActionButtonAnimator.scaling].
  final FloatingActionButtonAnimator floatingActionButtonAnimator;

  /// 底部按钮
  final List<Widget> persistentFooterButtons;

  /// Drawer
  final Widget drawer;

  /// 反向Drawer
  final Widget endDrawer;

  /// 用于遮蔽body的颜色，默认为[Colors.black54]
  final Color drawerScrimColor;

  /// 背景颜色
  final Color backgroundColor;

  /// 底部导航栏
  final Widget bottomNavigationBar;

  /// 永久底部面板
  final Widget bottomSheet;

  /// 当屏幕高度改变时，是否改变自身大小
  final bool resizeToAvoidBottomInset;

  /// 是否在状态栏之下
  final bool primary;

  /// {@macro flutter.material.drawer.dragStartBehavior}
  final DragStartBehavior drawerDragStartBehavior;

  /// 打开抽屉需要的水平滑动的距离
  final double drawerEdgeDragWidth;

  static ScaffoldState of(BuildContext context, { bool nullOk = false }) {
    final ScaffoldState result = 
        context.ancestorStateOfType(const TypeMatcher<ScaffoldState>());
    if (nullOk || result != null)
      return result;
  }

  /// Returns a [ValueListenable] for the [ScaffoldGeometry] for the closest
  /// [Scaffold] ancestor of the given context.
  ///
  /// The [ValueListenable.value] is only available at paint time.
  ///
  /// Notifications are guaranteed to be sent before the first paint pass
  /// with the new geometry, but there is no guarantee whether a build or
  /// layout passes are going to happen between the notification and the next
  /// paint pass.
  ///
  /// The closest [Scaffold] ancestor for the context might change, e.g when
  /// an element is moved from one scaffold to another. For [StatefulWidget]s
  /// using this listenable, a change of the [Scaffold] ancestor will
  /// trigger a [State.didChangeDependencies].
  ///
  /// A typical pattern for listening to the scaffold geometry would be to
  /// call [Scaffold.geometryOf] in [State.didChangeDependencies], compare the
  /// return value with the previous listenable, if it has changed, unregister
  /// the listener, and register a listener to the new [ScaffoldGeometry]
  /// listenable.
  static ValueListenable<ScaffoldGeometry> geometryOf(BuildContext context) {
    final _ScaffoldScope scaffoldScope = 
        context.inheritFromWidgetOfExactType(_ScaffoldScope);
    return scaffoldScope.geometryNotifier;
  }

  /// 最近的scaffold是否有Drawer
  static bool hasDrawer(BuildContext context, { bool registerForUpdates = true }) {
    if (registerForUpdates) {
      final _ScaffoldScope scaffold = 
          context.inheritFromWidgetOfExactType(_ScaffoldScope);
      return scaffold?.hasDrawer ?? false;
    } else {
      final ScaffoldState scaffold = context.ancestorStateOfType(
          const TypeMatcher<ScaffoldState>());
      return scaffold?.hasDrawer ?? false;
    }
  }

  @override
  ScaffoldState createState() => ScaffoldState();
}
```



## ScaffoldState

```dart
class ScaffoldState extends State<Scaffold> with TickerProviderStateMixin {

  // DRAWER API

  /// 两个globalkey用于控制左右两个drawer  
  final GlobalKey<DrawerControllerState> _drawerKey = 
      GlobalKey<DrawerControllerState>();
  final GlobalKey<DrawerControllerState> _endDrawerKey = 
      GlobalKey<DrawerControllerState>();

  /// 是否具有指定的组件  
  bool get hasAppBar => widget.appBar != null;
  bool get hasDrawer => widget.drawer != null;
  bool get hasEndDrawer => widget.endDrawer != null;
  bool get hasFloatingActionButton => widget.floatingActionButton != null;

  double _appBarMaxHeight;
  /// appbar的最高（总）高度
  double get appBarMaxHeight => _appBarMaxHeight;
    
  /// 两个drawer是否处于打开状态  
  bool _drawerOpened = false;
  bool _endDrawerOpened = false;

  bool get isDrawerOpen => _drawerOpened;
  bool get isEndDrawerOpen => _endDrawerOpened;
  
  void _drawerOpenedCallback(bool isOpened) {
    setState(() {
      _drawerOpened = isOpened;
    });
  }

  void _endDrawerOpenedCallback(bool isOpened) {
    setState(() {
      _endDrawerOpened = isOpened;
    });
  }

  /// 打开抽屉
  void openDrawer() {
    if (_endDrawerKey.currentState != null && _endDrawerOpened)
      _endDrawerKey.currentState.close();
    _drawerKey.currentState?.open();
  }
    
  void openEndDrawer() {
    if (_drawerKey.currentState != null && _drawerOpened)
      _drawerKey.currentState.close();
    _endDrawerKey.currentState?.open();
  }

  // SNACKBAR API

  final Queue<ScaffoldFeatureController<SnackBar, SnackBarClosedReason>> _snackBars = 
      Queue<ScaffoldFeatureController<SnackBar, SnackBarClosedReason>>();
  AnimationController _snackBarController;
  Timer _snackBarTimer;
  bool _accessibleNavigation;

  /// Shows a [SnackBar] at the bottom of the scaffold.
  ///
  /// A scaffold can show at most one snack bar at a time. If this function is
  /// called while another snack bar is already visible, the given snack bar
  /// will be added to a queue and displayed after the earlier snack bars have
  /// closed.
  ///
  /// To control how long a [SnackBar] remains visible, use [SnackBar.duration].
  ///
  /// To remove the [SnackBar] with an exit animation, use [hideCurrentSnackBar]
  /// or call [ScaffoldFeatureController.close] on the returned
  /// [ScaffoldFeatureController]. To remove a [SnackBar] suddenly (without an
  /// animation), use [removeCurrentSnackBar].
  ///
  /// See [Scaffold.of] for information about how to obtain the [ScaffoldState].
  ScaffoldFeatureController<SnackBar, SnackBarClosedReason> showSnackBar(SnackBar snackbar) {
    _snackBarController ??= SnackBar.createAnimationController(vsync: this)
      ..addStatusListener(_handleSnackBarStatusChange);
    if (_snackBars.isEmpty) {
      assert(_snackBarController.isDismissed);
      _snackBarController.forward();
    }
    ScaffoldFeatureController<SnackBar, SnackBarClosedReason> controller;
    controller = ScaffoldFeatureController<SnackBar, SnackBarClosedReason>._(
      // We provide a fallback key so that if back-to-back snackbars happen to
      // match in structure, material ink splashes and highlights don't survive
      // from one to the next.
      snackbar.withAnimation(_snackBarController, fallbackKey: UniqueKey()),
      Completer<SnackBarClosedReason>(),
      () {
        assert(_snackBars.first == controller);
        hideCurrentSnackBar(reason: SnackBarClosedReason.hide);
      },
      null, // SnackBar doesn't use a builder function so setState() wouldn't rebuild it
    );
    setState(() {
      _snackBars.addLast(controller);
    });
    return controller;
  }

  void _handleSnackBarStatusChange(AnimationStatus status) {
    switch (status) {
      case AnimationStatus.dismissed:
        assert(_snackBars.isNotEmpty);
        setState(() {
          _snackBars.removeFirst();
        });
        if (_snackBars.isNotEmpty)
          _snackBarController.forward();
        break;
      case AnimationStatus.completed:
        setState(() {
          assert(_snackBarTimer == null);
          // build will create a new timer if necessary to dismiss the snack bar
        });
        break;
      case AnimationStatus.forward:
      case AnimationStatus.reverse:
        break;
    }
  }

  /// Removes the current [SnackBar] (if any) immediately.
  ///
  /// The removed snack bar does not run its normal exit animation. If there are
  /// any queued snack bars, they begin their entrance animation immediately.
  void removeCurrentSnackBar({ SnackBarClosedReason reason = SnackBarClosedReason.remove }) {
    assert(reason != null);
    if (_snackBars.isEmpty)
      return;
    final Completer<SnackBarClosedReason> completer = _snackBars.first._completer;
    if (!completer.isCompleted)
      completer.complete(reason);
    _snackBarTimer?.cancel();
    _snackBarTimer = null;
    _snackBarController.value = 0.0;
  }

  /// Removes the current [SnackBar] by running its normal exit animation.
  ///
  /// The closed completer is called after the animation is complete.
  void hideCurrentSnackBar({ SnackBarClosedReason reason = SnackBarClosedReason.hide }) {
    assert(reason != null);
    if (_snackBars.isEmpty || _snackBarController.status == AnimationStatus.dismissed)
      return;
    final MediaQueryData mediaQuery = MediaQuery.of(context);
    final Completer<SnackBarClosedReason> completer = _snackBars.first._completer;
    if (mediaQuery.accessibleNavigation) {
      _snackBarController.value = 0.0;
      completer.complete(reason);
    } else {
      _snackBarController.reverse().then<void>((void value) {
        assert(mounted);
        if (!completer.isCompleted)
          completer.complete(reason);
      });
    }
    _snackBarTimer?.cancel();
    _snackBarTimer = null;
  }


  // PERSISTENT BOTTOM SHEET API

  // Contains bottom sheets that may still be animating out of view.
  // Important if the app/user takes an action that could repeatedly show a
  // bottom sheet.
  final List<_StandardBottomSheet> _dismissedBottomSheets = <_StandardBottomSheet>[];
  PersistentBottomSheetController<dynamic> _currentBottomSheet;

  void _maybeBuildPersistentBottomSheet() {
    if (widget.bottomSheet != null && _currentBottomSheet == null) {
      // The new _currentBottomSheet is not a local history entry so a "back" button
      // will not be added to the Scaffold's appbar and the bottom sheet will not
      // support drag or swipe to dismiss.
      final AnimationController animationController = BottomSheet.createAnimationController(this)..value = 1.0;
      LocalHistoryEntry _persistentSheetHistoryEntry;
      bool _persistentBottomSheetExtentChanged(DraggableScrollableNotification notification) {
        if (notification.extent > notification.initialExtent) {
          if (_persistentSheetHistoryEntry == null) {
            _persistentSheetHistoryEntry = LocalHistoryEntry(onRemove: () {
              if (notification.extent > notification.initialExtent) {
                DraggableScrollableActuator.reset(notification.context);
              }
              showBodyScrim(false, 0.0);
              _floatingActionButtonVisibilityValue = 1.0;
              _persistentSheetHistoryEntry = null;
            });
            ModalRoute.of(context).addLocalHistoryEntry(_persistentSheetHistoryEntry);
          }
        } else if (_persistentSheetHistoryEntry != null) {
          ModalRoute.of(context).removeLocalHistoryEntry(_persistentSheetHistoryEntry);
        }
        return false;
      }

      _currentBottomSheet = _buildBottomSheet<void>(
        (BuildContext context) {
          return NotificationListener<DraggableScrollableNotification>(
            onNotification: _persistentBottomSheetExtentChanged,
            child: DraggableScrollableActuator(
              child: widget.bottomSheet,
            ),
          );
        },
        true,
        animationController: animationController,
      );
    }
  }

  void _closeCurrentBottomSheet() {
    if (_currentBottomSheet != null) {
      if (!_currentBottomSheet._isLocalHistoryEntry) {
        _currentBottomSheet.close();
      }
      assert(() {
        _currentBottomSheet?._completer?.future?.whenComplete(() {
          assert(_currentBottomSheet == null);
        });
        return true;
      }());
    }
  }

  PersistentBottomSheetController<T> _buildBottomSheet<T>(
    WidgetBuilder builder,
    bool isPersistent, {
    AnimationController animationController,
    Color backgroundColor,
    double elevation,
    ShapeBorder shape,
    Clip clipBehavior,
  }) {
    assert(() {
      if (widget.bottomSheet != null && isPersistent && _currentBottomSheet != null) {
        throw FlutterError(
          'Scaffold.bottomSheet cannot be specified while a bottom sheet displayed '
          'with showBottomSheet() is still visible.\n Rebuild the Scaffold with a null '
          'bottomSheet before calling showBottomSheet().'
        );
      }
      return true;
    }());

    final Completer<T> completer = Completer<T>();
    final GlobalKey<_StandardBottomSheetState> bottomSheetKey = GlobalKey<_StandardBottomSheetState>();
    _StandardBottomSheet bottomSheet;

    bool removedEntry = false;
    void _removeCurrentBottomSheet() {
      removedEntry = true;
      if (_currentBottomSheet == null) {
        return;
      }
      assert(_currentBottomSheet._widget == bottomSheet);
      assert(bottomSheetKey.currentState != null);
      _showFloatingActionButton();

      void _closed(void value) {
        setState(() {
          _currentBottomSheet = null;
        });

        if (animationController.status != AnimationStatus.dismissed) {
          _dismissedBottomSheets.add(bottomSheet);
        }
        completer.complete();
      }

      final Future<void> closing = bottomSheetKey.currentState.close();
      if (closing != null) {
        closing.then(_closed);
      } else {
        _closed(null);
      }
    }

    final LocalHistoryEntry entry = isPersistent
      ? null
      : LocalHistoryEntry(onRemove: () {
        if (!removedEntry) {
          _removeCurrentBottomSheet();
        }
      });

    bottomSheet = _StandardBottomSheet(
      key: bottomSheetKey,
      animationController: animationController,
      enableDrag: !isPersistent,
      onClosing: () {
        if (_currentBottomSheet == null) {
          return;
        }
        assert(_currentBottomSheet._widget == bottomSheet);
        if (!isPersistent && !removedEntry) {
          assert(entry != null);
          entry.remove();
          removedEntry = true;
        }
      },
      onDismissed: () {
        if (_dismissedBottomSheets.contains(bottomSheet)) {
          setState(() {
            _dismissedBottomSheets.remove(bottomSheet);
          });
        }
      },
      builder: builder,
      isPersistent: isPersistent,
      backgroundColor: backgroundColor,
      elevation: elevation,
      shape: shape,
      clipBehavior: clipBehavior,
    );

    if (!isPersistent)
      ModalRoute.of(context).addLocalHistoryEntry(entry);

    return PersistentBottomSheetController<T>._(
      bottomSheet,
      completer,
      entry != null
        ? entry.remove
        : _removeCurrentBottomSheet,
      (VoidCallback fn) { bottomSheetKey.currentState?.setState(fn); },
      !isPersistent,
    );
  }

  /// Shows a material design bottom sheet in the nearest [Scaffold]. To show
  /// a persistent bottom sheet, use the [Scaffold.bottomSheet].
  ///
  /// Returns a controller that can be used to close and otherwise manipulate the
  /// bottom sheet.
  ///
  /// To rebuild the bottom sheet (e.g. if it is stateful), call
  /// [PersistentBottomSheetController.setState] on the controller returned by
  /// this method.
  ///
  /// The new bottom sheet becomes a [LocalHistoryEntry] for the enclosing
  /// [ModalRoute] and a back button is added to the appbar of the [Scaffold]
  /// that closes the bottom sheet.
  ///
  /// To create a persistent bottom sheet that is not a [LocalHistoryEntry] and
  /// does not add a back button to the enclosing Scaffold's appbar, use the
  /// [Scaffold.bottomSheet] constructor parameter.
  ///
  /// A persistent bottom sheet shows information that supplements the primary
  /// content of the app. A persistent bottom sheet remains visible even when
  /// the user interacts with other parts of the app.
  ///
  /// A closely related widget is a modal bottom sheet, which is an alternative
  /// to a menu or a dialog and prevents the user from interacting with the rest
  /// of the app. Modal bottom sheets can be created and displayed with the
  /// [showModalBottomSheet] function.
  ///
  /// See also:
  ///
  ///  * [BottomSheet], which becomes the parent of the widget returned by the
  ///    `builder`.
  ///  * [showBottomSheet], which calls this method given a [BuildContext].
  ///  * [showModalBottomSheet], which can be used to display a modal bottom
  ///    sheet.
  ///  * [Scaffold.of], for information about how to obtain the [ScaffoldState].
  ///  * <https://material.io/design/components/sheets-bottom.html#standard-bottom-sheet>
  PersistentBottomSheetController<T> showBottomSheet<T>(
    WidgetBuilder builder, {
    Color backgroundColor,
    double elevation,
    ShapeBorder shape,
    Clip clipBehavior,
  }) {
    assert(() {
      if (widget.bottomSheet != null) {
        throw FlutterError(
          'Scaffold.bottomSheet cannot be specified while a bottom sheet displayed '
          'with showBottomSheet() is still visible.\n Rebuild the Scaffold with a null '
          'bottomSheet before calling showBottomSheet().'
        );
      }
      return true;
    }());
    assert(debugCheckHasMediaQuery(context));

    _closeCurrentBottomSheet();
    final AnimationController controller = BottomSheet.createAnimationController(this)..forward();
    setState(() {
      _currentBottomSheet = _buildBottomSheet<T>(
        builder,
        false,
        animationController: controller,
        backgroundColor: backgroundColor,
        elevation: elevation,
        shape: shape,
        clipBehavior: clipBehavior,
      );
    });
    return _currentBottomSheet;
  }

  // Floating Action Button API
  AnimationController _floatingActionButtonMoveController;
  FloatingActionButtonAnimator _floatingActionButtonAnimator;
  FloatingActionButtonLocation _previousFloatingActionButtonLocation;
  FloatingActionButtonLocation _floatingActionButtonLocation;

  AnimationController _floatingActionButtonVisibilityController;

  /// Gets the current value of the visibility animation for the
  /// [Scaffold.floatingActionButton].
  double get _floatingActionButtonVisibilityValue => _floatingActionButtonVisibilityController.value;

  /// Sets the current value of the visibility animation for the
  /// [Scaffold.floatingActionButton].  This value must not be null.
  set _floatingActionButtonVisibilityValue(double newValue) {
    assert(newValue != null);
    _floatingActionButtonVisibilityController.value = newValue.clamp(
      _floatingActionButtonVisibilityController.lowerBound,
      _floatingActionButtonVisibilityController.upperBound,
    );
  }

  /// Shows the [Scaffold.floatingActionButton].
  TickerFuture _showFloatingActionButton() {
    return _floatingActionButtonVisibilityController.forward();
  }

  // Moves the Floating Action Button to the new Floating Action Button Location.
  void _moveFloatingActionButton(final FloatingActionButtonLocation newLocation) {
    FloatingActionButtonLocation previousLocation = _floatingActionButtonLocation;
    double restartAnimationFrom = 0.0;
    // If the Floating Action Button is moving right now, we need to start from a snapshot of the current transition.
    if (_floatingActionButtonMoveController.isAnimating) {
      previousLocation = _TransitionSnapshotFabLocation(_previousFloatingActionButtonLocation, _floatingActionButtonLocation, _floatingActionButtonAnimator, _floatingActionButtonMoveController.value);
      restartAnimationFrom = _floatingActionButtonAnimator.getAnimationRestart(_floatingActionButtonMoveController.value);
    }

    setState(() {
      _previousFloatingActionButtonLocation = previousLocation;
      _floatingActionButtonLocation = newLocation;
    });

    // Animate the motion even when the fab is null so that if the exit animation is running,
    // the old fab will start the motion transition while it exits instead of jumping to the
    // new position.
    _floatingActionButtonMoveController.forward(from: restartAnimationFrom);
  }

  // iOS FEATURES - status bar tap, back gesture

  // On iOS, tapping the status bar scrolls the app's primary scrollable to the
  // top. We implement this by providing a primary scroll controller and
  // scrolling it to the top when tapped.

  final ScrollController _primaryScrollController = ScrollController();

  void _handleStatusBarTap() {
    if (_primaryScrollController.hasClients) {
      _primaryScrollController.animateTo(
        0.0,
        duration: const Duration(milliseconds: 300),
        curve: Curves.linear, // TODO(ianh): Use a more appropriate curve.
      );
    }
  }

  // INTERNALS

  _ScaffoldGeometryNotifier _geometryNotifier;

  // Backwards compatibility for deprecated resizeToAvoidBottomPadding property
  bool get _resizeToAvoidBottomInset {
    // ignore: deprecated_member_use_from_same_package
    return widget.resizeToAvoidBottomInset ?? widget.resizeToAvoidBottomPadding ?? true;
  }

  @override
  void initState() {
    super.initState();
    _geometryNotifier = _ScaffoldGeometryNotifier(const ScaffoldGeometry(), context);
    _floatingActionButtonLocation = widget.floatingActionButtonLocation ?? _kDefaultFloatingActionButtonLocation;
    _floatingActionButtonAnimator = widget.floatingActionButtonAnimator ?? _kDefaultFloatingActionButtonAnimator;
    _previousFloatingActionButtonLocation = _floatingActionButtonLocation;
    _floatingActionButtonMoveController = AnimationController(
      vsync: this,
      lowerBound: 0.0,
      upperBound: 1.0,
      value: 1.0,
      duration: kFloatingActionButtonSegue * 2,
    );

    _floatingActionButtonVisibilityController = AnimationController(
      duration: kFloatingActionButtonSegue,
      vsync: this,
    );
  }

  @override
  void didUpdateWidget(Scaffold oldWidget) {
    // Update the Floating Action Button Animator, and then schedule the Floating Action Button for repositioning.
    if (widget.floatingActionButtonAnimator != oldWidget.floatingActionButtonAnimator) {
      _floatingActionButtonAnimator = widget.floatingActionButtonAnimator ?? _kDefaultFloatingActionButtonAnimator;
    }
    if (widget.floatingActionButtonLocation != oldWidget.floatingActionButtonLocation) {
      _moveFloatingActionButton(widget.floatingActionButtonLocation ?? _kDefaultFloatingActionButtonLocation);
    }
    if (widget.bottomSheet != oldWidget.bottomSheet) {
      assert(() {
        if (widget.bottomSheet != null && _currentBottomSheet?._isLocalHistoryEntry == true) {
          throw FlutterError(
            'Scaffold.bottomSheet cannot be specified while a bottom sheet displayed '
            'with showBottomSheet() is still visible.\n Use the PersistentBottomSheetController '
            'returned by showBottomSheet() to close the old bottom sheet before creating '
            'a Scaffold with a (non null) bottomSheet.'
          );
        }
        return true;
      }());
      _closeCurrentBottomSheet();
      _maybeBuildPersistentBottomSheet();
    }
    super.didUpdateWidget(oldWidget);
  }

  @override
  void didChangeDependencies() {
    final MediaQueryData mediaQuery = MediaQuery.of(context);
    // If we transition from accessible navigation to non-accessible navigation
    // and there is a SnackBar that would have timed out that has already
    // completed its timer, dismiss that SnackBar. If the timer hasn't finished
    // yet, let it timeout as normal.
    if (_accessibleNavigation == true
      && !mediaQuery.accessibleNavigation
      && _snackBarTimer != null
      && !_snackBarTimer.isActive) {
      hideCurrentSnackBar(reason: SnackBarClosedReason.timeout);
    }
    _accessibleNavigation = mediaQuery.accessibleNavigation;
    _maybeBuildPersistentBottomSheet();
    super.didChangeDependencies();
  }

  @override
  void dispose() {
    _snackBarController?.dispose();
    _snackBarTimer?.cancel();
    _snackBarTimer = null;
    _geometryNotifier.dispose();
    for (_StandardBottomSheet bottomSheet in _dismissedBottomSheets) {
      bottomSheet.animationController?.dispose();
    }
    if (_currentBottomSheet != null) {
      _currentBottomSheet._widget.animationController?.dispose();
    }
    _floatingActionButtonMoveController.dispose();
    _floatingActionButtonVisibilityController.dispose();
    super.dispose();
  }

  void _addIfNonNull(
    List<LayoutId> children,
    Widget child,
    Object childId, {
    @required bool removeLeftPadding,
    @required bool removeTopPadding,
    @required bool removeRightPadding,
    @required bool removeBottomPadding,
    bool removeBottomInset = false,
    bool maintainBottomViewPadding = false,
  }) {
    MediaQueryData data = MediaQuery.of(context).removePadding(
      removeLeft: removeLeftPadding,
      removeTop: removeTopPadding,
      removeRight: removeRightPadding,
      removeBottom: removeBottomPadding,
    );
    if (removeBottomInset)
      data = data.removeViewInsets(removeBottom: true);

    if (maintainBottomViewPadding && data.viewInsets.bottom != 0.0) {
      data = data.copyWith(
        padding: data.padding.copyWith(bottom: data.viewPadding.bottom)
      );
    }

    if (child != null) {
      children.add(
        LayoutId(
          id: childId,
          child: MediaQuery(data: data, child: child),
        ),
      );
    }
  }

  void _buildEndDrawer(List<LayoutId> children, TextDirection textDirection) {
    if (widget.endDrawer != null) {
      assert(hasEndDrawer);
      _addIfNonNull(
        children,
        DrawerController(
          key: _endDrawerKey,
          alignment: DrawerAlignment.end,
          child: widget.endDrawer,
          drawerCallback: _endDrawerOpenedCallback,
          dragStartBehavior: widget.drawerDragStartBehavior,
          scrimColor: widget.drawerScrimColor,
          edgeDragWidth: widget.drawerEdgeDragWidth,
        ),
        _ScaffoldSlot.endDrawer,
        // remove the side padding from the side we're not touching
        removeLeftPadding: textDirection == TextDirection.ltr,
        removeTopPadding: false,
        removeRightPadding: textDirection == TextDirection.rtl,
        removeBottomPadding: false,
      );
    }
  }

  void _buildDrawer(List<LayoutId> children, TextDirection textDirection) {
    if (widget.drawer != null) {
      assert(hasDrawer);
      _addIfNonNull(
        children,
        DrawerController(
          key: _drawerKey,
          alignment: DrawerAlignment.start,
          child: widget.drawer,
          drawerCallback: _drawerOpenedCallback,
          dragStartBehavior: widget.drawerDragStartBehavior,
          scrimColor: widget.drawerScrimColor,
          edgeDragWidth: widget.drawerEdgeDragWidth,
        ),
        _ScaffoldSlot.drawer,
        // remove the side padding from the side we're not touching
        removeLeftPadding: textDirection == TextDirection.rtl,
        removeTopPadding: false,
        removeRightPadding: textDirection == TextDirection.ltr,
        removeBottomPadding: false,
      );
    }
  }

  bool _showBodyScrim = false;
  Color _bodyScrimColor = Colors.black;

  /// Whether to show a [ModalBarrier] over the body of the scaffold.
  ///
  /// The `value` parameter must not be null.
  void showBodyScrim(bool value, double opacity) {
    assert(value != null);
    if (_showBodyScrim == value && _bodyScrimColor.opacity == opacity) {
      return;
    }
    setState(() {
      _showBodyScrim = value;
      _bodyScrimColor = Colors.black.withOpacity(opacity);
    });
  }

  @override
  Widget build(BuildContext context) {
    assert(debugCheckHasMediaQuery(context));
    assert(debugCheckHasDirectionality(context));
    final MediaQueryData mediaQuery = MediaQuery.of(context);
    final ThemeData themeData = Theme.of(context);
    final TextDirection textDirection = Directionality.of(context);
    _accessibleNavigation = mediaQuery.accessibleNavigation;

    if (_snackBars.isNotEmpty) {
      final ModalRoute<dynamic> route = ModalRoute.of(context);
      if (route == null || route.isCurrent) {
        if (_snackBarController.isCompleted && _snackBarTimer == null) {
          final SnackBar snackBar = _snackBars.first._widget;
          _snackBarTimer = Timer(snackBar.duration, () {
            assert(_snackBarController.status == AnimationStatus.forward ||
                   _snackBarController.status == AnimationStatus.completed);
            // Look up MediaQuery again in case the setting changed.
            final MediaQueryData mediaQuery = MediaQuery.of(context);
            if (mediaQuery.accessibleNavigation && snackBar.action != null)
              return;
            hideCurrentSnackBar(reason: SnackBarClosedReason.timeout);
          });
        }
      } else {
        _snackBarTimer?.cancel();
        _snackBarTimer = null;
      }
    }

    final List<LayoutId> children = <LayoutId>[];
    _addIfNonNull(
      children,
      widget.body == null ? null : _BodyBuilder(
        extendBody: widget.extendBody,
        extendBodyBehindAppBar: widget.extendBodyBehindAppBar,
        body: widget.body,
      ),
      _ScaffoldSlot.body,
      removeLeftPadding: false,
      removeTopPadding: widget.appBar != null,
      removeRightPadding: false,
      removeBottomPadding: widget.bottomNavigationBar != null || widget.persistentFooterButtons != null,
      removeBottomInset: _resizeToAvoidBottomInset,
    );
    if (_showBodyScrim) {
      _addIfNonNull(
        children,
        ModalBarrier(
          dismissible: false,
          color: _bodyScrimColor,
        ),
        _ScaffoldSlot.bodyScrim,
        removeLeftPadding: true,
        removeTopPadding: true,
        removeRightPadding: true,
        removeBottomPadding: true,
      );
    }

    if (widget.appBar != null) {
      final double topPadding = widget.primary ? mediaQuery.padding.top : 0.0;
      _appBarMaxHeight = widget.appBar.preferredSize.height + topPadding;
      assert(_appBarMaxHeight >= 0.0 && _appBarMaxHeight.isFinite);
      _addIfNonNull(
        children,
        ConstrainedBox(
          constraints: BoxConstraints(maxHeight: _appBarMaxHeight),
          child: FlexibleSpaceBar.createSettings(
            currentExtent: _appBarMaxHeight,
            child: widget.appBar,
          ),
        ),
        _ScaffoldSlot.appBar,
        removeLeftPadding: false,
        removeTopPadding: false,
        removeRightPadding: false,
        removeBottomPadding: true,
      );
    }

    bool isSnackBarFloating = false;
    if (_snackBars.isNotEmpty) {
      final SnackBarBehavior snackBarBehavior = _snackBars.first._widget.behavior
        ?? themeData.snackBarTheme.behavior
        ?? SnackBarBehavior.fixed;
      isSnackBarFloating = snackBarBehavior == SnackBarBehavior.floating;

      _addIfNonNull(
        children,
        _snackBars.first._widget,
        _ScaffoldSlot.snackBar,
        removeLeftPadding: false,
        removeTopPadding: true,
        removeRightPadding: false,
        removeBottomPadding: widget.bottomNavigationBar != null || widget.persistentFooterButtons != null,
        maintainBottomViewPadding: !_resizeToAvoidBottomInset,
      );
    }

    if (widget.persistentFooterButtons != null) {
      _addIfNonNull(
        children,
        Container(
          decoration: BoxDecoration(
            border: Border(
              top: Divider.createBorderSide(context, width: 1.0),
            ),
          ),
          child: SafeArea(
            top: false,
            child: ButtonBar(
              children: widget.persistentFooterButtons,
            ),
          ),
        ),
        _ScaffoldSlot.persistentFooter,
        removeLeftPadding: false,
        removeTopPadding: true,
        removeRightPadding: false,
        removeBottomPadding: false,
        maintainBottomViewPadding: !_resizeToAvoidBottomInset,
      );
    }

    if (widget.bottomNavigationBar != null) {
      _addIfNonNull(
        children,
        widget.bottomNavigationBar,
        _ScaffoldSlot.bottomNavigationBar,
        removeLeftPadding: false,
        removeTopPadding: true,
        removeRightPadding: false,
        removeBottomPadding: false,
        maintainBottomViewPadding: !_resizeToAvoidBottomInset,
      );
    }

    if (_currentBottomSheet != null || _dismissedBottomSheets.isNotEmpty) {
      final Widget stack = Stack(
        alignment: Alignment.bottomCenter,
        children: <Widget>[
          ..._dismissedBottomSheets,
          if (_currentBottomSheet != null) _currentBottomSheet._widget,
        ],
      );
      _addIfNonNull(
        children,
        stack,
        _ScaffoldSlot.bottomSheet,
        removeLeftPadding: false,
        removeTopPadding: true,
        removeRightPadding: false,
        removeBottomPadding: _resizeToAvoidBottomInset,
      );
    }

    _addIfNonNull(
      children,
      _FloatingActionButtonTransition(
        child: widget.floatingActionButton,
        fabMoveAnimation: _floatingActionButtonMoveController,
        fabMotionAnimator: _floatingActionButtonAnimator,
        geometryNotifier: _geometryNotifier,
        currentController: _floatingActionButtonVisibilityController,
      ),
      _ScaffoldSlot.floatingActionButton,
      removeLeftPadding: true,
      removeTopPadding: true,
      removeRightPadding: true,
      removeBottomPadding: true,
    );

    switch (themeData.platform) {
      case TargetPlatform.iOS:
        _addIfNonNull(
          children,
          GestureDetector(
            behavior: HitTestBehavior.opaque,
            onTap: _handleStatusBarTap,
            // iOS accessibility automatically adds scroll-to-top to the clock in the status bar
            excludeFromSemantics: true,
          ),
          _ScaffoldSlot.statusBar,
          removeLeftPadding: false,
          removeTopPadding: true,
          removeRightPadding: false,
          removeBottomPadding: true,
        );
        break;
      case TargetPlatform.android:
      case TargetPlatform.fuchsia:
        break;
    }

    if (_endDrawerOpened) {
      _buildDrawer(children, textDirection);
      _buildEndDrawer(children, textDirection);
    } else {
      _buildEndDrawer(children, textDirection);
      _buildDrawer(children, textDirection);
    }

    // The minimum insets for contents of the Scaffold to keep visible.
    final EdgeInsets minInsets = mediaQuery.padding.copyWith(
      bottom: _resizeToAvoidBottomInset ? mediaQuery.viewInsets.bottom : 0.0,
    );

    // extendBody locked when keyboard is open
    final bool _extendBody = minInsets.bottom <= 0 && widget.extendBody;

    return _ScaffoldScope(
      hasDrawer: hasDrawer,
      geometryNotifier: _geometryNotifier,
      child: PrimaryScrollController(
        controller: _primaryScrollController,
        child: Material(
          color: widget.backgroundColor ?? themeData.scaffoldBackgroundColor,
          child: AnimatedBuilder(animation: _floatingActionButtonMoveController, builder: (BuildContext context, Widget child) {
            return CustomMultiChildLayout(
              children: children,
              delegate: _ScaffoldLayout(
                extendBody: _extendBody,
                extendBodyBehindAppBar: widget.extendBodyBehindAppBar,
                minInsets: minInsets,
                currentFloatingActionButtonLocation: _floatingActionButtonLocation,
                floatingActionButtonMoveAnimationProgress: _floatingActionButtonMoveController.value,
                floatingActionButtonMotionAnimator: _floatingActionButtonAnimator,
                geometryNotifier: _geometryNotifier,
                previousFloatingActionButtonLocation: _previousFloatingActionButtonLocation,
                textDirection: textDirection,
                isSnackBarFloating: isSnackBarFloating,
              ),
            );
          }),
        ),
      ),
    );
  }
}
```



## ScaffoldFeatureController

```dart
class ScaffoldFeatureController<T extends Widget, U> {
  const ScaffoldFeatureController._(this._widget, this._completer, this.close, this.setState);
  final T _widget;
  final Completer<U> _completer;

  /// Completes when the feature controlled by this object is no longer visible.
  Future<U> get closed => _completer.future;

  /// Remove the feature (e.g., bottom sheet or snack bar) from the scaffold.
  final VoidCallback close;

  /// Mark the feature (e.g., bottom sheet or snack bar) as needing to rebuild.
  final StateSetter setState;
}
```



## _ScaffoldScope

```dart
class _ScaffoldScope extends InheritedWidget {
  const _ScaffoldScope({
    @required this.hasDrawer,
    @required this.geometryNotifier,
    @required Widget child,
  }) : assert(hasDrawer != null),
       super(child: child);

  final bool hasDrawer;
  final _ScaffoldGeometryNotifier geometryNotifier;

  @override
  bool updateShouldNotify(_ScaffoldScope oldWidget) {
    return hasDrawer != oldWidget.hasDrawer;
  }
}
```


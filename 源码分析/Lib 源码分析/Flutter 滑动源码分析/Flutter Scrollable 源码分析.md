# Flutter Scrollable 源码分析

## Scrollable

```dart
/// 一个可以滚动的组件
///
/// [Scrollable] 实现了可滚动组建的交互模型，包括手势识别等，但是，它不了解视窗如何展示它的孩子（简言之，就是
/// [Scrollable] 只处理了用户手势对 scorllOffset 的影响，并不关心 scorllOffset 如何影响孩子的布局。其相关的
/// 工作由 [Viewport] 完成）
///
/// 静态方法 [Scrollable.of] 和 [Scrollable.ensureVisible] 通常被用于与 [ListView] 或者 [GridView].内的
/// [Scrollable] 组件进行交互 
///
/// 遵循如下规则可以进一步自定义 [Scrollable]:
///
/// 1. 提供 [viewportBuilder] 来自定义孩子模型。例如，[SingleChildScrollView] 使用了单孩子视窗，而 
///    [CustomScrollView] 使用 [Viewport] 或者 [ShrinkWrappingViewport] 来展示多个孩子。
///
/// 2. 提供 [ScrollController]，它可以给出自定义的 [ScrollPosition] 子类。例如，[PageView] 使用了
///    [PageController], 它能够固定页面的滚动
class Scrollable extends StatefulWidget {
  const Scrollable({
    Key key,
    this.axisDirection = AxisDirection.down,
    this.controller,
    this.physics,
    @required this.viewportBuilder,
    this.incrementCalculator,
    this.excludeFromSemantics = false,
    this.semanticChildCount,
    this.dragStartBehavior = DragStartBehavior.start,
  }) : super (key: key);


  final AxisDirection axisDirection;

  /// 可用于控制此组件对应的 scroll position 的对象。
  ///
  /// [ScrollController] 有以下几点目标：
  /// 1.它能够被用于控制I初始滚动位置
  /// 2.它能够被用于控制滚动视图时候应该自动保存或者恢复它的 scroll position
  /// 3.它能够被用于获取当前 scroll position，或者改变它
  final ScrollController controller;

  /// 此组件如何相应用户的手势输入
  final ScrollPhysics physics;

  /// 视窗构建器
  final ViewportBuilder viewportBuilder;

  /// 一个可选的函数，当使用 [ScrollAction] 要求 scrollable 通过键盘滚动时，将调用该函数来计算要滚动的距离。
  final ScrollIncrementCalculator incrementCalculator;

  
  final bool excludeFromSemantics;
  final int semanticChildCount;

  /// 拖拽开始行为
  final DragStartBehavior dragStartBehavior;

  /// 轴方向
  Axis get axis => axisDirectionToAxis(axisDirection);

  @override
  ScrollableState createState() => ScrollableState();

  /// 获取到最近的 ScrollableState
  ///
  /// 使用如下：
  ///
  /// ```dart
  /// ScrollableState scrollable = Scrollable.of(context);
  /// ```
  static ScrollableState of(BuildContext context) {
    final _ScrollableScope widget = context.dependOnInheritedWidgetOfExactType<_ScrollableScope>();
    return widget?.scrollable;
  }

  /// 是给定的 context 对应的 widget 被展示到视窗里
  static Future<void> ensureVisible(
    BuildContext context, {
    double alignment = 0.0,
    Duration duration = Duration.zero,
    Curve curve = Curves.ease,
    ScrollPositionAlignmentPolicy alignmentPolicy = ScrollPositionAlignmentPolicy.explicit,
  }) {
    final List<Future<void>> futures = <Future<void>>[];

    ScrollableState scrollable = Scrollable.of(context);
    /// 注意，这里会逐级地向上查找 scrollable（其实是 ScrollableState），并在找到的每一个的 
    /// ScrollableState.position 上调用 [ensureVisible]
    /// 详见 [ScrollPosition.ensureVisible]
    while (scrollable != null) {
      futures.add(scrollable.position.ensureVisible(
        context.findRenderObject(),
        alignment: alignment,
        duration: duration,
        curve: curve,
        alignmentPolicy: alignmentPolicy,
      ));
      context = scrollable.context;
      scrollable = Scrollable.of(context);
    }

    if (futures.isEmpty || duration == Duration.zero)
      return Future<void>.value();
    if (futures.length == 1)
      return futures.single;
    /// 找到了不止一个 ？？
    return Future.wait<void>(futures).then<void>((List<void> _) => null);
  }
}
```



## ScrollContext

```dart
/// [Scrollable] 为了使用 [ScrollPosition] 而需要实现的接口
abstract class ScrollContext {
  /// 当需要派发 [ScrollNotification]s 时用到的 [BuildContext]
  ///
  /// This context is typically different that the context of the scrollable
  /// widget itself. For example, [Scrollable] uses a context outside the
  /// [Viewport] but inside the widgets created by
  /// [ScrollBehavior.buildViewportChrome].
  BuildContext get notificationContext;

  /// 用于搜索 [PageStorage] 的 [BuildContext]
  ///
  /// This context is typically the context of the scrollable widget itself. In
  /// particular, it should involve any [GlobalKey]s that are dynamically
  /// created as part of creating the scrolling widget, since those would be
  /// different each time the widget is created.
  BuildContext get storageContext;

  TickerProvider get vsync;

  AxisDirection get axisDirection;

  /// 此组件的内容是否应该忽略 [PointerEvent]，也就是是否应该相应指针事件
  ///
  /// 将此值设置为 true 将阻止使用指针事件此组件的内容进行交互。但是此组件本身仍然是可交互的。
  ///
  /// 例如，如果 scroll position 正在被动画驱动，它可能会将此字段设置为 true 以阻止用户意外地与内容交互（例
  /// 如，阻止用户点击内容），但同时，它自身是可交互的，用户可以滑动来停止动画驱动 
  void setIgnorePointer(bool value);

  /// 用户是否可以拖动此组件
  void setCanDrag(bool value);

  void setSemanticsActions(Set<SemanticsAction> actions);
}
```



## ScrollableState

```dart
/// 要操作 [Scrollable] 组件的滚动位置，请使用从 [position] 属性获得的对象。
///
/// 为了获取到 [Scrollable] 组件的滚动事件, 使用 [NotificationListener] 来监听 [ScrollNotification] 
///
/// 通常来说，若要自定义滚动，无需继承此类，而是提供自定义 [ScrollPhysics]
///
/// 注意：此类实现了 ScrollContext
class ScrollableState 
    extends State<Scrollable> 
    with TickerProviderStateMixin
    implements ScrollContext {
  
  /// 管理滚动位置
  ///
  /// 为了控制 [Scrollable] 使用何种类型的 [ScrollPosition]，提供一个自定义的 [ScrollController]。 
  /// [ScrollController] 可以利用其 [ScrollController.createScrollPosition] 方法提供恰当的 
  /// [ScrollPosition] 
  ScrollPosition get position => _position;
  ScrollPosition _position;

  @override
  AxisDirection get axisDirection => widget.axisDirection;

  ScrollBehavior _configuration;
  ScrollPhysics _physics;

  // 这个方法只应该需要重建的时候调用
  void _updatePosition() {
    _configuration = ScrollConfiguration.of(context);
    _physics = _configuration.getScrollPhysics(context);
    if (widget.physics != null)
      _physics = widget.physics.applyTo(_physics);
    final ScrollController controller = widget.controller;
    final ScrollPosition oldPosition = position;
    if (oldPosition != null) {
      controller?.detach(oldPosition);
      // It's important that we not dispose the old position until after the
      // viewport has had a chance to unregister its listeners from the old
      // position. So, schedule a microtask to do it.
      scheduleMicrotask(oldPosition.dispose);
    }

    _position = controller?.createScrollPosition(
        _physics, 
        this, 
        oldPosition
    ) ?? ScrollPositionWithSingleContext(
        physics: _physics, 
        context: this, 
        oldPosition: oldPosition
    );

    controller?.attach(position);
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    _updatePosition();
  }

  bool _shouldUpdatePosition(Scrollable oldWidget) {
    ScrollPhysics newPhysics = widget.physics;
    ScrollPhysics oldPhysics = oldWidget.physics;
    do {
      if (newPhysics?.runtimeType != oldPhysics?.runtimeType)
        return true;
      newPhysics = newPhysics?.parent;
      oldPhysics = oldPhysics?.parent;
    } while (newPhysics != null || oldPhysics != null);

    return widget.controller?.runtimeType != oldWidget.controller?.runtimeType;
  }

  @override
  void didUpdateWidget(Scrollable oldWidget) {
    super.didUpdateWidget(oldWidget);

    if (widget.controller != oldWidget.controller) {
      oldWidget.controller?.detach(position);
      widget.controller?.attach(position);
    }

    if (_shouldUpdatePosition(oldWidget))
      _updatePosition();
  }

  @override
  void dispose() {
    widget.controller?.detach(position);
    position.dispose();
    super.dispose();
  }


  // SEMANTICS

  final GlobalKey _scrollSemanticsKey = GlobalKey();

  @override
  @protected
  void setSemanticsActions(Set<SemanticsAction> actions) {
    if (_gestureDetectorKey.currentState != null)
      _gestureDetectorKey.currentState.replaceSemanticsActions(actions);
  }


  // GESTURE RECOGNITION AND POINTER IGNORING

  final GlobalKey<RawGestureDetectorState> _gestureDetectorKey = GlobalKey<RawGestureDetectorState>();
  final GlobalKey _ignorePointerKey = GlobalKey();

  // This field is set during layout, and then reused until the next time it is set.
  Map<Type, GestureRecognizerFactory> _gestureRecognizers = const <Type, GestureRecognizerFactory>{};
  bool _shouldIgnorePointer = false;

  bool _lastCanDrag;
  Axis _lastAxisDirection;

  @override
  @protected
  void setCanDrag(bool canDrag) {
    if (canDrag == _lastCanDrag && (!canDrag || widget.axis == _lastAxisDirection))
      return;
    if (!canDrag) {
      _gestureRecognizers = const <Type, GestureRecognizerFactory>{};
    } else {
      switch (widget.axis) {
        case Axis.vertical:
          _gestureRecognizers = <Type, GestureRecognizerFactory>{
            VerticalDragGestureRecognizer: GestureRecognizerFactoryWithHandlers<VerticalDragGestureRecognizer>(
              () => VerticalDragGestureRecognizer(),
              (VerticalDragGestureRecognizer instance) {
                instance
                  ..onDown = _handleDragDown
                  ..onStart = _handleDragStart
                  ..onUpdate = _handleDragUpdate
                  ..onEnd = _handleDragEnd
                  ..onCancel = _handleDragCancel
                  ..minFlingDistance = _physics?.minFlingDistance
                  ..minFlingVelocity = _physics?.minFlingVelocity
                  ..maxFlingVelocity = _physics?.maxFlingVelocity
                  ..dragStartBehavior = widget.dragStartBehavior;
              },
            ),
          };
          break;
        case Axis.horizontal:
          _gestureRecognizers = <Type, GestureRecognizerFactory>{
            HorizontalDragGestureRecognizer: GestureRecognizerFactoryWithHandlers<HorizontalDragGestureRecognizer>(
              () => HorizontalDragGestureRecognizer(),
              (HorizontalDragGestureRecognizer instance) {
                instance
                  ..onDown = _handleDragDown
                  ..onStart = _handleDragStart
                  ..onUpdate = _handleDragUpdate
                  ..onEnd = _handleDragEnd
                  ..onCancel = _handleDragCancel
                  ..minFlingDistance = _physics?.minFlingDistance
                  ..minFlingVelocity = _physics?.minFlingVelocity
                  ..maxFlingVelocity = _physics?.maxFlingVelocity
                  ..dragStartBehavior = widget.dragStartBehavior;
              },
            ),
          };
          break;
      }
    }
    _lastCanDrag = canDrag;
    _lastAxisDirection = widget.axis;
    if (_gestureDetectorKey.currentState != null)
      _gestureDetectorKey.currentState.replaceGestureRecognizers(_gestureRecognizers);
  }

  @override
  TickerProvider get vsync => this;

  @override
  @protected
  void setIgnorePointer(bool value) {
    if (_shouldIgnorePointer == value)
      return;
    _shouldIgnorePointer = value;
    if (_ignorePointerKey.currentContext != null) {
      final RenderIgnorePointer renderBox = _ignorePointerKey.currentContext.findRenderObject() as RenderIgnorePointer;
      renderBox.ignoring = _shouldIgnorePointer;
    }
  }

  @override
  BuildContext get notificationContext => _gestureDetectorKey.currentContext;

  @override
  BuildContext get storageContext => context;

  // TOUCH HANDLERS

  Drag _drag;
  ScrollHoldController _hold;

  void _handleDragDown(DragDownDetails details) {
    assert(_drag == null);
    assert(_hold == null);
    _hold = position.hold(_disposeHold);
  }

  void _handleDragStart(DragStartDetails details) {
    // It's possible for _hold to become null between _handleDragDown and
    // _handleDragStart, for example if some user code calls jumpTo or otherwise
    // triggers a new activity to begin.
    assert(_drag == null);
    _drag = position.drag(details, _disposeDrag);
    assert(_drag != null);
    assert(_hold == null);
  }

  void _handleDragUpdate(DragUpdateDetails details) {
    // _drag might be null if the drag activity ended and called _disposeDrag.
    assert(_hold == null || _drag == null);
    _drag?.update(details);
  }

  void _handleDragEnd(DragEndDetails details) {
    // _drag might be null if the drag activity ended and called _disposeDrag.
    assert(_hold == null || _drag == null);
    _drag?.end(details);
    assert(_drag == null);
  }

  void _handleDragCancel() {
    // _hold might be null if the drag started.
    // _drag might be null if the drag activity ended and called _disposeDrag.
    assert(_hold == null || _drag == null);
    _hold?.cancel();
    _drag?.cancel();
    assert(_hold == null);
    assert(_drag == null);
  }

  void _disposeHold() {
    _hold = null;
  }

  void _disposeDrag() {
    _drag = null;
  }

  // SCROLL WHEEL

  // Returns the offset that should result from applying [event] to the current
  // position, taking min/max scroll extent into account.
  double _targetScrollOffsetForPointerScroll(PointerScrollEvent event) {
    double delta = widget.axis == Axis.horizontal
        ? event.scrollDelta.dx
        : event.scrollDelta.dy;

    if (axisDirectionIsReversed(widget.axisDirection)) {
      delta *= -1;
    }

    return math.min(math.max(position.pixels + delta, position.minScrollExtent),
        position.maxScrollExtent);
  }

  void _receivedPointerSignal(PointerSignalEvent event) {
    if (event is PointerScrollEvent && position != null) {
      final double targetScrollOffset = _targetScrollOffsetForPointerScroll(event);
      // Only express interest in the event if it would actually result in a scroll.
      if (targetScrollOffset != position.pixels) {
        GestureBinding.instance.pointerSignalResolver.register(event, _handlePointerScroll);
      }
    }
  }

  void _handlePointerScroll(PointerEvent event) {
    assert(event is PointerScrollEvent);
    if (_physics != null && !_physics.shouldAcceptUserOffset(position)) {
      return;
    }
    final double targetScrollOffset = _targetScrollOffsetForPointerScroll(event as PointerScrollEvent);
    if (targetScrollOffset != position.pixels) {
      position.jumpTo(targetScrollOffset);
    }
  }

  // DESCRIPTION

  @override
  Widget build(BuildContext context) {
    assert(position != null);
    // _ScrollableScope must be placed above the BuildContext returned by notificationContext
    // so that we can get this ScrollableState by doing the following:
    //
    // ScrollNotification notification;
    // Scrollable.of(notification.context)
    //
    // Since notificationContext is pointing to _gestureDetectorKey.context, _ScrollableScope
    // must be placed above the widget using it: RawGestureDetector
    Widget result = _ScrollableScope(
      scrollable: this,
      position: position,
      // TODO(ianh): Having all these global keys is sad.
      child: Listener(
        onPointerSignal: _receivedPointerSignal,
        child: RawGestureDetector(
          key: _gestureDetectorKey,
          gestures: _gestureRecognizers,
          behavior: HitTestBehavior.opaque,
          excludeFromSemantics: widget.excludeFromSemantics,
          child: Semantics(
            explicitChildNodes: !widget.excludeFromSemantics,
            child: IgnorePointer(
              key: _ignorePointerKey,
              ignoring: _shouldIgnorePointer,
              ignoringSemantics: false,
              child: widget.viewportBuilder(context, position),
            ),
          ),
        ),
      ),
    );

    if (!widget.excludeFromSemantics) {
      result = _ScrollSemantics(
        key: _scrollSemanticsKey,
        child: result,
        position: position,
        allowImplicitScrolling: widget?.physics?.allowImplicitScrolling ?? _physics.allowImplicitScrolling,
        semanticChildCount: widget.semanticChildCount,
      );
    }

    return _configuration.buildViewportChrome(context, result, widget.axisDirection);
  }
}
```



##  _ScrollableScope

```dart
/// 一个用于向子树提供 ScrollableState 和 ScrollPosition 的 InheritedWidget
class _ScrollableScope extends InheritedWidget {
  const _ScrollableScope({
    Key key,
    @required this.scrollable,
    @required this.position,
    @required Widget child,
  }) : super(key: key, child: child);

  final ScrollableState scrollable;
  final ScrollPosition position;

  @override
  bool updateShouldNotify(_ScrollableScope old) {
    return position != old.position;
  }
}
```


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

  /// 设置此组件的内容是否应该忽略 [PointerEvent]，也就是是否应该相应指针事件
  ///
  /// 将此值设置为 true 将阻止使用指针事件此组件的内容进行交互。但是此组件本身仍然是可交互的。
  ///
  /// 例如，如果 scroll position 正在被动画驱动，它可能会将此字段设置为 true 以阻止用户意外地与内容交互（例
  /// 如，阻止用户点击内容），但同时，它自身是可交互的，用户可以滑动来停止动画驱动 
  /// 
  /// 此方法通常被 scrollPosition 的实现类调用
  void setIgnorePointer(bool value);

  /// 设置用户是否可以拖动此组件
  ///
  /// 此方法通常被 applyNewDimensions 调用
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

  // 这个方法只应该需要重建的时候调用，创建（或者更新）position 字段
  void _updatePosition() {
    /// 依赖配置。可以在顶层的依赖里为子树设置默认的滚动效果
    _configuration = ScrollConfiguration.of(context);
    _physics = _configuration.getScrollPhysics(context);
    /// 如果 widget 中提供了额外的 _physics，将会把此 _physics 和父 _physics 合并
    if (widget.physics != null)
      _physics = widget.physics.applyTo(_physics);
      
    final ScrollController controller = widget.controller;
    final ScrollPosition oldPosition = position;
    /// 如果 oldPosition 不为 null，也就是说不是首次调用此方法的话，需要把 oldPosition detach
    if (oldPosition != null) {
      controller?.detach(oldPosition);
      // 在 viewport 有机会从 oldPosition 中取消其侦听器注册之前，我们不能销毁 oldPosition。
      // 所以，安排一个微任务来完成它。
      scheduleMicrotask(oldPosition.dispose);
    }

    /// 调用 controller.createScrollPosition 的方法来创建 position
    _position = controller?.createScrollPosition(
        _physics, 
        this, 
        oldPosition
    ) ?? ScrollPositionWithSingleContext(
        physics: _physics, 
        context: this, 
        oldPosition: oldPosition
    );

    /// 把 position 添加到 controller 的 position 列表中，并使得 controller 监听 position 的变化
    controller?.attach(position);
  }

  /// 依赖发生改变则更新配置（在 updatePosition 中建立了对上层 ScrollConfiguration 的依赖）
  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    _updatePosition();
  }

  /// 是否应该更新 Position
  bool _shouldUpdatePosition(Scrollable oldWidget) {
    ScrollPhysics newPhysics = widget.physics;
    ScrollPhysics oldPhysics = oldWidget.physics;
    /// 沿着 Physics.parent 向上查找，如果找到的 newPhysics 对应的运行类型不同于 oldPhysics，则认为需要更新 
    /// Position
    do {
      if (newPhysics?.runtimeType != oldPhysics?.runtimeType)
        return true;
      newPhysics = newPhysics?.parent;
      oldPhysics = oldPhysics?.parent;
    } while (newPhysics != null || oldPhysics != null);

    /// 如果 controller 的运行类型发生了改变，也需要更新 position
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


  // 语义

  final GlobalKey _scrollSemanticsKey = GlobalKey();

  @override
  @protected
  void setSemanticsActions(Set<SemanticsAction> actions) {
    if (_gestureDetectorKey.currentState != null)
      _gestureDetectorKey.currentState.replaceSemanticsActions(actions);
  }


  // 手势识别和指针忽略

  final GlobalKey<RawGestureDetectorState> _gestureDetectorKey = 
        GlobalKey<RawGestureDetectorState>();
  final GlobalKey _ignorePointerKey = GlobalKey();

  // 这一字段在布局的时候被设置，直到下一次被设置之前被重用
  Map<Type, GestureRecognizerFactory> _gestureRecognizers = const <Type, GestureRecognizerFactory>{};
  bool _shouldIgnorePointer = false;

  bool _lastCanDrag;
  Axis _lastAxisDirection;

  @override
  @protected
  void setCanDrag(bool canDrag) {
    if (canDrag == _lastCanDrag && (!canDrag || widget.axis == _lastAxisDirection))
      return;
    /// 如果不能滑动，清空手势识别器
    if (!canDrag) {
      _gestureRecognizers = const <Type, GestureRecognizerFactory>{};
    /// 否则，根据轴方向来决定 _gestureRecognizers
    } else {
      switch (widget.axis) {
        /// 如果是垂直滑动，添加垂直手势识别器
        case Axis.vertical:
          _gestureRecognizers = <Type, GestureRecognizerFactory>{
            VerticalDragGestureRecognizer: 
            GestureRecognizerFactoryWithHandlers<VerticalDragGestureRecognizer>(
              () => VerticalDragGestureRecognizer(),
              (VerticalDragGestureRecognizer instance) {
                instance
                  /// 为 VerticalDragGestureRecognizer 实例设置了回调方法，下同
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
        /// 如果是水平滑动，添加水平手势识别器
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
    
    /// 置换新的 _gestureRecognizers
    if (_gestureDetectorKey.currentState != null)
      _gestureDetectorKey.currentState.replaceGestureRecognizers(_gestureRecognizers);
  }

  @override
  TickerProvider get vsync => this;

  @override
  @protected
  /// 设置需要忽略指针
  void setIgnorePointer(bool value) {
    if (_shouldIgnorePointer == value)
      return;
    _shouldIgnorePointer = value;
    if (_ignorePointerKey.currentContext != null) {
      final RenderIgnorePointer renderBox = 
          _ignorePointerKey.currentContext.findRenderObject() as RenderIgnorePointer;
      renderBox.ignoring = _shouldIgnorePointer;
    }
  }

  /// 通知上下文就是 gestureDetectorKey.currentContext
  @override
  BuildContext get notificationContext => _gestureDetectorKey.currentContext;

  /// 存储上下文就是 此 state 对应的 context
  @override
  BuildContext get storageContext => context;

  // 点击处理
  
  /// 拖动抽象
  Drag _drag;
  /// 滚动保持控制器
  ScrollHoldController _hold;

    
  /// 以下方法绑定给 GestureRecognizer，在对应的情况下触发
    
  /// 当用户接触 scrollView （但是还没有开始拖动）的时候，这时候视作 hold 状态，产生一个 
  /// ScrollHoldController，并且，接下来可能发生拖动，或者取消事件
  void _handleDragDown(DragDownDetails details) {
    _hold = position.hold(_disposeHold);
  }

  /// 用户开始拖动，产生一个 ScrollDragController，并且，接下来可以发生拖动更新，结束，取消事件
  void _handleDragStart(DragStartDetails details) {
    _drag = position.drag(details, _disposeDrag);
  }

  /// 用户正在拖动，这时候需要更新 scrollOffset，并通知 viewport 进行重新布局和重绘
  void _handleDragUpdate(DragUpdateDetails details) {
    _drag?.update(details);
  }

  /// 拖动结束
  void _handleDragEnd(DragEndDetails details) {
    _drag?.end(details);
  }

  /// 拖动取消
  /// 有两种情况：在 hold 状态下的取消和在 drag 状态下的取消
  void _handleDragCancel() {
    _hold?.cancel();
    _drag?.cancel();
  }

  /// 销毁 hold 或者 drag：直接置 null 回收内存
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
    final double targetScrollOffset = 
        _targetScrollOffsetForPointerScroll(event as PointerScrollEvent);
    if (targetScrollOffset != position.pixels) {
      position.jumpTo(targetScrollOffset);
    }
  }

  // DESCRIPTION

  @override
  Widget build(BuildContext context) {
    // _ScrollableScope 必须放置在 notificationContext 的“上方”，因此，我们可以用以下的方式来获取
    // ScrollableState
    //
    // ScrollNotification notification;
    // Scrollable.of(notification.context)
    //
    // 因为 notificationContext 被指定为 _gestureDetectorKey.context, _ScrollableScope
    // 必须放置在 RawGestureDetector 的“上方”
    Widget result = _ScrollableScope(
      scrollable: this,
      position: position,
      // TODO(ianh): Having all these global keys is sad.（不应该全部使用 GlobalKey）
      child: Listener(
        onPointerSignal: _receivedPointerSignal,
        /// 用于识别用户手势，并在对应时候触发回调
        child: RawGestureDetector(
          key: _gestureDetectorKey,
          gestures: _gestureRecognizers,
          behavior: HitTestBehavior.opaque,
          excludeFromSemantics: widget.excludeFromSemantics,
          child: Semantics(
            explicitChildNodes: !widget.excludeFromSemantics,
            /// 用于吸收指针。
            /// 当发生滚动事件时，可能需要吸收指针以阻止点击交互，
            /// 注意：
            /// 此组件的位置位于 GestureRecognizer 的下层，也就是说，此组件并不会阻止用户与 scroll widget 
            /// 本身的交互（可以进行拖动操作），但是可以阻止用户与 scroll widget 的子树进行交互（阻止用户在
            /// 滑动的时候点击滑块）
            child: IgnorePointer(
              key: _ignorePointerKey,
              ignoring: _shouldIgnorePointer,
              ignoringSemantics: false,
              /// 最下层是视窗及其子树，视窗承担了根据给定  scorllOffset 来布局绘制的工作
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
        allowImplicitScrolling: 
          widget?.physics?.allowImplicitScrolling ?? _physics.allowImplicitScrolling,
        semanticChildCount: widget.semanticChildCount,
      );
    }

    /// _configuration.buildViewportChrome 方法如下所示：
    /// Widget buildViewportChrome(BuildContext context, Widget child, AxisDirection axisDirection) {
    /// 	switch (getPlatform(context)) {
    /// 	  case TargetPlatform.iOS:
    /// 	  case TargetPlatform.macOS:
    /// 	    return child;
    /// 	  case TargetPlatform.android:
    /// 	  case TargetPlatform.fuchsia:
    /// 	    return GlowingOverscrollIndicator(
    /// 	      child: child,
    /// 	      axisDirection: axisDirection,
    /// 	      color: _kDefaultGlowColor,
    /// 	    );
    /// 	}
    /// 	return null;
    /// }
    /// 若平台是 android 或者 fuchsia，则采用 GlowingOverscrollIndicator，也就是过渡滚动时的波纹效果
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


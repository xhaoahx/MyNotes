# Flutter ViewportOffset 源码分析



## ViewportOffset

```dart
/// 这个类决定了视窗内部的那一部分内容需要被展示
///
/// 当 pixels 发生改变的时候，通知所有的监听者
abstract class ViewportOffset extends ChangeNotifier {
  ViewportOffset();

  /// 工厂方法，返回 _FixedViewportOffset，其 [pixels] 值不会被改变，除非视窗发起了一次更正
  factory ViewportOffset.fixed(double value) = _FixedViewportOffset;
  factory ViewportOffset.zero() = _FixedViewportOffset.zero;

  /// 用于将孩子向 axis direction 反方向偏离的像素值
  ///
  /// 例如，axis direction 是 down, pixels 代表了在屏幕中是向上移动孩子的像素值。类似的，如果 axis 
  /// direction 是 left， 那么 pixels 代表了在屏幕中是向右移动孩子的像素值
  /// 当这个值发生变化的时候，通知所有监听者（除非这个变化是由 [correctBy] 调用引起的）
  double get pixels;

  /// 当视窗被建立的时候调用
  ///
  /// 参数是 [RenderViewport] 的主轴大小（例如，垂直视窗的高度）
  ///
  /// This may be called redundantly, with the same value, each frame. This is
  /// called during layout for the [RenderViewport]. If the viewport is
  /// configured to shrink-wrap its contents, it may be called several times,
  /// since the layout is repeated each time the scroll offset is corrected.
  ///
  /// If this is called, it is called before [applyContentDimensions]. If this
  /// is called, [applyContentDimensions] will be called soon afterwards in the
  /// same layout phase. If the viewport is not configured to shrink-wrap its
  /// contents, then this will only be called when the viewport recomputes its
  /// size (i.e. when its parent lays out), and not during normal scrolling.
  ///
  /// If applying the viewport dimensions changes the scroll offset, return
  /// false. Otherwise, return true. If you return false, the [RenderViewport]
  /// will be laid out again with the new scroll offset. This is expensive. (The
  /// return value is answering the question "did you accept these viewport
  /// dimensions unconditionally?"; if the new dimensions change the
  /// [ViewportOffset]'s actual [pixels] value, then the viewport will need to
  /// be laid out again.)
  bool applyViewportDimension(double viewportDimension);

  /// Called when the viewport's content extents are established.
  ///
  /// The arguments are the minimum and maximum scroll extents respectively. The
  /// minimum will be equal to or less than the maximum. In the case of slivers,
  /// the minimum will be equal to or less than zero, the maximum will be equal
  /// to or greater than zero.
  ///
  /// The maximum scroll extent has the viewport dimension subtracted from it.
  /// For instance, if there is 100.0 pixels of scrollable content, and the
  /// viewport is 80.0 pixels high, then the minimum scroll extent will
  /// typically be 0.0 and the maximum scroll extent will typically be 20.0,
  /// because there's only 20.0 pixels of actual scroll slack.
  ///
  /// If applying the content dimensions changes the scroll offset, return
  /// false. Otherwise, return true. If you return false, the [RenderViewport]
  /// will be laid out again with the new scroll offset. This is expensive. (The
  /// return value is answering the question "did you accept these content
  /// dimensions unconditionally?"; if the new dimensions change the
  /// [ViewportOffset]'s actual [pixels] value, then the viewport will need to
  /// be laid out again.)
  ///
  /// This is called at least once each time the [RenderViewport] is laid out,
  /// even if the values have not changed. It may be called many times if the
  /// scroll offset is corrected (if this returns false). This is always called
  /// after [applyViewportDimension], if that method is called.
  bool applyContentDimensions(double minScrollExtent, double maxScrollExtent);

  /// 调用一次布局时对 scrollOffset 的修正
  void correctBy(double correction);

  /// 立即将 [pixels] 改成指定的值（如果是有效的），并通知所有的监听者
  void jumpTo(double pixels);

  /// 将 [pixels] 从当前值向给定的值过渡
  Future<void> animateTo(
    double to, {
    @required Duration duration,
    @required Curve curve,
  });

  /// 如果给定的时间为 null 或者零，那么调用 jumpTo，否则调用 animateTo
  Future<void> moveTo(
    double to, {
    Duration duration,
    Curve curve,
    bool clamp,
  }) {
    assert(to != null);
    if (duration == null || duration == Duration.zero) {
      jumpTo(to);
      return Future<void>.value();
    } else {
      return animateTo(to, duration: duration, curve: curve ?? Curves.ease);
    }
  }

  /// 用户视图改变 [pixels] 的方向, 相对于[RenderViewport.axisDirection].
  ScrollDirection get userScrollDirection;

  /// 是否允许间接地改变 [pixels] 来响应 [RenderObject.showOnScreen] 的调用
  bool get allowImplicitScrolling;
}
```



## ScrollMetrics

```dart
/// 对 [Scrollable] 内容的描述
abstract class ScrollMetrics {
  ScrollMetrics copyWith({
    double minScrollExtent,
    double maxScrollExtent,
    double pixels,
    double viewportDimension,
    AxisDirection axisDirection,
  }) {
    return FixedScrollMetrics(
      minScrollExtent: minScrollExtent ?? this.minScrollExtent,
      maxScrollExtent: maxScrollExtent ?? this.maxScrollExtent,
      pixels: pixels ?? this.pixels,
      viewportDimension: viewportDimension ?? this.viewportDimension,
      axisDirection: axisDirection ?? this.axisDirection,
    );
  }

  /// 关于 minScrollExtent 和 maxScrollExtent
  /// 
  /// 考虑 ScrollView 中 center 不为第一个 child 的情况：
  /// 如果把 anchor 设置为 0.0，且 aixsDirection 设置为 down，那么，center 将会显示到视窗的顶部并且与边缘对
  /// 其。此时，scrollOffset（也就是此 [pixels]）显然是 0.0，但是视窗“上方”还有一部分内容可以显示。因此，可
  /// 以向下滑动视窗（泛化地说，向 aixsDirection 的反方向滚动）。这会导致 scrollOffset（也就是此 [pixels]）
  /// 小于 0.0（向 aixsDirection 的正方向滚动时，scrollOffset 是增加的）。由于 center 之前的内容可能是有限
  /// 的，因此会对应有 scrollOffset（也就是此 [pixels] 的最小；相反，有最大值。
  
  /// 最小的被视作在范围内的 [pixels].
  ///
  /// [pixels] 的实际值可能处于 [outOfRange]（也就是过渡滚动的时候）
  double get minScrollExtent;

  /// 最大的被视作在范围内 [pixels].
  ///
  /// [pixels] 的实际值可能处于 [outOfRange]（也就是过渡滚动的时候）
  double get maxScrollExtent;

  /// 当 [axisDirection] 正方向的滚动, [axisDirection] 上的逻辑像素值
  double get pixels;

  /// 沿着 [axisDirection] 上的视窗的大小
  double get viewportDimension;

  /// scrollview 的方向
  AxisDirection get axisDirection;

  /// 轴方向
  /// 即：如果 axisDirection 是 down 或者 up,那么轴方向是 vertical，否则是 horizental
  Axis get axis => axisDirectionToAxis(axisDirection);

  /// 是否 [pixels] 的值处于 [minScrollExtent] 之间 [maxScrollExtent]，即是否出界，出界值将被视为过渡滚动
  bool get outOfRange => pixels < minScrollExtent || pixels > maxScrollExtent;

  /// 是否 [pixels] 的值等于 [minScrollExtent] 或者 [maxScrollExtent].
  bool get atEdge => pixels == minScrollExtent || pixels == maxScrollExtent;

  /// 概念上“高于”可滚动视图的大小。即 [extentInside] 描述的内容上方的内容。
  /// 
  /// 参考对 minScrollExtent 的描述，我们假设 center “上方” 的内容的高度是 100.0
  /// 因此，minScrollExtent 是 -100.0（即最多可以向 axisDirection 的反方向滚动 100.0），并且，此时的 
  /// scrollOffset 是 50.0（即我们向 axisDirection 的正方向滚动了 50.0）。这种情况下，我们知道，在视窗“上
  /// 方”的内容高度为 pixels - minScrollExtent = 100 + 50 = 150。（由于可以过渡滚动，即 pixels < 
  /// minScrollExtent，因此取最小值为 0.0）
  double get extentBefore => math.max(pixels - minScrollExtent, 0.0);

  /// 概念上“处于”可滚动视图内的大小。
  ///
  /// 当 [outOfRange] 是 false 时，这个值通常是是视窗的高度
  /// 这个值可能会小于视窗的大小，例如当过度滚动的时候
  ///
  /// 这个值总是非负的，并且小于等于 [viewportDimension].
  double get extentInside {
    /// 首先，我们假设视窗的高度是 500
    /// 然后我们假设有一个垂直的列表，它的长度是 1000，axisDirection 是 down，center 是第一个 child
    /// 显然，其 maxScrollExtent = 1000，minScrollExtent = 0.0
    /// 假设 pixels 是 1100
    /// 即我们向上滚动了 1100 像素，此时发生了过度滚动。即 outOfRange == true
    /// 此时，下方过渡滚动的值是 100
    /// 因此，视窗内的实际内容高度只有 400，即 extentInside = 400.0
    /// 此时 extentAfter = 0.0，extentBefore = 1100
    return viewportDimension
      // “上方” 过度滚动的值
      - (minScrollExtent - pixels).clamp(0, viewportDimension)
      // “下方” 过度滚动的值
      - (pixels - maxScrollExtent).clamp(0, viewportDimension);
  }

  /// 类比于 extentBefore 的概念可知
  double get extentAfter => math.max(maxScrollExtent - pixels, 0.0);
}
```



## ScrollPostion

```dart
/// 决定了 scrollview 应该展示那一部分了内容
///
/// [pixels] 值决定滚动视图用来选择要显示内容的哪一部分。
/// 当用户滚动视图端口时，这个值会改变，从而改变显示的内容。
///
/// [ScrollPosition] 运用 [physics] 进行滚动，并且储存 [minScrollExtent] 和 [maxScrollExtent].
///
/// 滚动被当前的 [activity] 所控制, [activity] 是被 [beginActivity] 设置的
/// [ScrollPosition] 本身并不开始任何滚动活动，其子类 [ScrollPositionWithSingleContext],
/// 通常会开始一个活动来响应用户输入，或者 [ScrollController] 的指令
///
/// ## 继承 ScrollPosition
///
/// 随着时间变化, a [Scrollable] 可能会持有许多的 [ScrollPosition] 对象
/// 例如，假如 [Scrollable.physics] 改变了类型, [Scrollable] 会创建一个新的 [ScrollPosition] 
/// 为了将旧的 physics 向新的 physics 变换，子类必须调用 [absorb]
///
/// 无论何时当 [userScrollDirection] 发生改变的时候，子类必须调用 [didUpdateScrollDirection] 
abstract class ScrollPosition extends ViewportOffset with ScrollMetrics {
  ScrollPosition({
    @required this.physics,
    @required this.context,
    this.keepScrollOffset = true,
    ScrollPosition oldPosition,
    this.debugLabel,
  }) {
    if (oldPosition != null)
      absorb(oldPosition);
    if (keepScrollOffset)
      restoreScrollOffset();
  }

  /// scroll position 如何响应用户输入
  final ScrollPhysics physics;

  /// 滚动发生的位置
  ///
  /// 通常情况下是一个 [ScrollableState].
  final ScrollContext context;

  /// 时候要保存 scroll offset 到 [PageStorage] 中
  final bool keepScrollOffset;

  final String debugLabel;

  @override
  double get minScrollExtent => _minScrollExtent;
  double _minScrollExtent;

  @override
  double get maxScrollExtent => _maxScrollExtent;
  double _maxScrollExtent;

  @override
  double get pixels => _pixels;
  double _pixels;

  @override
  double get viewportDimension => _viewportDimension;
  double _viewportDimension;

  /// 是否 [viewportDimension], [minScrollExtent], [maxScrollExtent],
  /// [outOfRange], and [atEdge] 能够被使用
  ///
  /// 当 [applyNewDimensions] 首次被调用的时候，设置为 true
  bool get haveDimensions => _haveDimensions;
  bool _haveDimensions = false;

  /// 从给定 [ScrollPosition] 获取到所有可运用的状态
  ///
  /// 如果给定了一个 `oldPosition` 的话，这个方法被构造函数调用
  /// `other` 可以不与本对象具有相同的 [runtimeType]，即可以改变类型
  ///
  /// This method can be destructive to the other [ScrollPosition]. The other
  /// object must be disposed immediately after this call (in the same call
  /// stack, before microtask resolution, by whomever called this object's
  /// constructor).
  ///
  /// If the old [ScrollPosition] object is a different [runtimeType] than this
  /// one, the [ScrollActivity.resetActivity] method is invoked on the newly
  /// adopted [ScrollActivity].
  @protected
  @mustCallSuper
  void absorb(ScrollPosition other) {
    _minScrollExtent = other.minScrollExtent;
    _maxScrollExtent = other.maxScrollExtent;
    _pixels = other._pixels;
    _viewportDimension = other.viewportDimension;
    _activity = other.activity;
    other._activity = null;
    if (other.runtimeType != runtimeType)
      activity.resetActivity();
    context.setIgnorePointer(activity.shouldIgnorePointer);
    isScrollingNotifier.value = activity.isScrolling;
  }

  /// 更新这个 scrollPostion 的 scrollOffset ([pixels]) 到一个给定的值
  ///
  /// 只能被当前的 [ScrollActivity] 在 transient callback phase 或者响应用户输入的时候调用
  ///
  /// 如果有过度滚动的话，返回它
  /// If the return value is 0.0, that means
  /// that [pixels] now returns the given `value`. If the return value is
  /// positive, then [pixels] is less than the requested `value` by the given
  /// amount (overscroll past the max extent), and if it is negative, it is
  /// greater than the requested `value` by the given amount (underscroll past
  /// the min extent).
  ///
  /// 过渡滚动的像素值被 [applyBoundaryConditions] 所计算
  ///
  /// The amount of the change that is applied is reported using [didUpdateScrollPositionBy].
  /// If there is any overscroll, it is reported using [didOverscrollBy].
  double setPixels(double newPixels) {
    if (newPixels != pixels) {
      final double overscroll = applyBoundaryConditions(newPixels);
      final double oldPixels = _pixels;
      _pixels = newPixels - overscroll;
      if (_pixels != oldPixels) {
        notifyListeners();
        didUpdateScrollPositionBy(_pixels - oldPixels);
      }
      if (overscroll != 0.0) {
        didOverscrollBy(overscroll);
        return overscroll;
      }
    }
    return 0.0;
  }

  /// 这个方法改变 [pixels] 的值，但是不通知任何的监听者
  ///
  /// 这个方法在布局的时候用来调节此 Positon. 这个方法被调用来响应 [applyViewportDimension] 或
  /// [applyContentDimensions] (在两种情况下，如果这个函数被调用了，那么它们必须返回 false 来表示 Positon 
  /// 已经被调节了).
  void correctPixels(double value) {
    _pixels = value;
  }

  /// 调用一次布局时对 scroll offset 的修正
  ///
  /// 这个方法改变 [pixels] 的值，但是不通知任何的监听者
  /// 他在布局阶段被 [RenderViewport] 调用，在 [applyContentDimensions] 被调用之前
  /// 当这个方法被调用之后，Layout 方法将会重新被调用
  @override
  void correctBy(double correction) {
    _pixels += correction;
    _didChangeViewportDimensionOrReceiveCorrection = true;
  }

  /// Change the value of [pixels] to the new value, and notify any customers,
  /// but without honoring normal conventions for changing the scroll offset.
  ///
  /// This is used to implement [jumpTo]. It can also be used adjust the
  /// position when the dimensions of the viewport change. It should only be
  /// used when manually implementing the logic for honoring the relevant
  /// conventions of the class. For example, [ScrollPositionWithSingleContext]
  /// introduces [ScrollActivity] objects and uses [forcePixels] in conjunction
  /// with adjusting the activity, e.g. by calling
  /// [ScrollPositionWithSingleContext.goIdle], so that the activity does
  /// not immediately set the value back. (Consider, for instance, a case where
  /// one is using a [DrivenScrollActivity]. That object will ignore any calls
  /// to [forcePixels], which would result in the rendering stuttering: changing
  /// in response to [forcePixels], and then changing back to the next value
  /// derived from the animation.)
  ///
  /// To cause the position to jump or animate to a new value, consider [jumpTo]
  /// or [animateTo].
  ///
  /// This should not be called during layout (e.g. when setting the initial
  /// scroll offset). Consider [correctPixels] if you find you need to adjust
  /// the position during layout.
  @protected
  void forcePixels(double value) {
    assert(pixels != null);
    _pixels = value;
    notifyListeners();
  }

  /// 滚动结束的时候，在 PageStorage 中储存 scroll offset
  /// PageStorage 通常被 navigator 构建
  @protected
  void saveScrollOffset() {
    PageStorage.of(context.storageContext)?.writeState(context.storageContext, pixels);
  }

  /// 恢复存储的 Scrolloffset 的值
  @protected
  void restoreScrollOffset() {
    if (pixels == null) {
      final double value = 
          PageStorage.of(context.storageContext)?.readState(context.storageContext) as double;
      if (value != null)
        correctPixels(value);
    }
  }

  /// 运用边界条件，返回过渡滚动的值
  ///
  /// 如果给定的值在边界以内，返回 0.0. 否则返回边界条件下不能运用到 [pixels] 的值的和
  /// 如果 [physics] 允许出界滚动，这个方法总是返回 0.0
  ///
  /// 默认实现延迟给 [physics] 对象的 [ScrollPhysics.applyBoundaryConditions].
  @protected
  double applyBoundaryConditions(double value) {
    final double result = physics.applyBoundaryConditions(this, value);
    return result;
  }

  bool _didChangeViewportDimensionOrReceiveCorrection = true;

  @override
  bool applyViewportDimension(double viewportDimension) {
    if (_viewportDimension != viewportDimension) {
      _viewportDimension = viewportDimension;
      _didChangeViewportDimensionOrReceiveCorrection = true;
    }
    return true;
  }

  Set<SemanticsAction> _semanticActions;
    
  void _updateSemanticActions() {
    SemanticsAction forward;
    SemanticsAction backward;
    switch (axis) {
      case Axis.vertical:
        forward = SemanticsAction.scrollUp;
        backward = SemanticsAction.scrollDown;
        break;
      case Axis.horizontal:
        forward = SemanticsAction.scrollLeft;
        backward = SemanticsAction.scrollRight;
        break;
    }

    final Set<SemanticsAction> actions = <SemanticsAction>{};
    if (pixels > minScrollExtent)
      actions.add(backward);
    if (pixels < maxScrollExtent)
      actions.add(forward);

    if (setEquals<SemanticsAction>(actions, _semanticActions))
      return;

    _semanticActions = actions;
    context.setSemanticsActions(_semanticActions);
  }

  /// 
  @override
  bool applyContentDimensions(double minScrollExtent, double maxScrollExtent) {
    /// 如果给定的 minScrollExtent 或 maxScrollExtent 不与之前的值近似相等的话
    /// 或者当视窗的大小发生了改变，或者接收到了以一次更正
    /// 那么会更新  _minScrollExtent 、_maxScrollExtent 的值，并调用 applyNewDimensions
    if (   !nearEqual(_minScrollExtent, minScrollExtent, Tolerance.defaultTolerance.distance) 
        || !nearEqual(_maxScrollExtent, maxScrollExtent, Tolerance.defaultTolerance.distance) 
        || _didChangeViewportDimensionOrReceiveCorrection
    ) {
      _minScrollExtent = minScrollExtent;
      _maxScrollExtent = maxScrollExtent;
      _haveDimensions = true;
      applyNewDimensions();
      _didChangeViewportDimensionOrReceiveCorrection = false;
    }
    return true;
  }

  /// 通知 activity 当前视窗的大小维度或者内容发生了改变
  ///
  /// 当 [applyViewportDimension] 或者 [applyContentDimensions] 改变了 [minScrollExtent]，或者 
  /// [maxScrollExtent] 或者 [viewportDimension] 之后调用
  /// 并且，这个方法必须在任何的修正被 [correctPixels], 运用到 [pixels] 之后再调用
  ///
  /// 默认实现使用 [ScrollActivity.applyNewDimensions] 来通知 [activity] 新的维度的变化
  @protected
  @mustCallSuper
  void applyNewDimensions() {
    assert(pixels != null);
    activity.applyNewDimensions();
    _updateSemanticActions(); 
  }

  /// 使指定的 renderObject 尽可能地被展示
  /// 其中，duration 是动画间隔，
  /// alignment 和 alignmentPolicy 指定了排列方式
  Future<void> ensureVisible(
    RenderObject object, {
    double alignment = 0.0,
    Duration duration = Duration.zero,
    Curve curve = Curves.ease,
    ScrollPositionAlignmentPolicy alignmentPolicy = ScrollPositionAlignmentPolicy.explicit,
  }) {
    final RenderAbstractViewport viewport = RenderAbstractViewport.of(object);
    double target;
    switch (alignmentPolicy) {
      case ScrollPositionAlignmentPolicy.explicit:
        target = 
            viewport.getOffsetToReveal(object, alignment)
            .offset.clamp(minScrollExtent, maxScrollExtent) as double;
        break;
      case ScrollPositionAlignmentPolicy.keepVisibleAtEnd:
        target = 
            viewport.getOffsetToReveal(object, 1.0).
            offset.clamp(minScrollExtent, maxScrollExtent) as double;
        if (target < pixels) {
          target = pixels;
        }
        break;
      case ScrollPositionAlignmentPolicy.keepVisibleAtStart:
        target = 
            viewport.getOffsetToReveal(object, 0.0)
            .offset.clamp(minScrollExtent, maxScrollExtent) as double;
        if (target > pixels) {
          target = pixels;
        }
        break;
    }

    if (target == pixels)
      return Future<void>.value();

    if (duration == Duration.zero) {
      jumpTo(target);
      return Future<void>.value();
    }

    return animateTo(target, duration: duration, curve: curve);
  }

  /// 是否有滚动正在进行
  final ValueNotifier<bool> isScrollingNotifier = ValueNotifier<bool>(false);

  @override
  Future<void> animateTo(
    double to, {
    @required Duration duration,
    @required Curve curve,
  });

  @override
  void jumpTo(double value);

  @override
  Future<void> moveTo(
    double to, {
    Duration duration,
    Curve curve,
    bool clamp = true,
  }) {
    if (clamp)
      to = to.clamp(minScrollExtent, maxScrollExtent) as double;
    return super.moveTo(to, duration: duration, curve: curve);
  }

  /// 是否允许间接地改变 [pixels] 来响应 [RenderObject.showOnScreen] 的调用
  /// 默认实现延迟给 physics
  @override
  bool get allowImplicitScrolling => physics.allowImplicitScrolling;

  /// 停止当前的 activity 并且开始一个 [HoldScrollActivity]，返回一个 ScrollHoldController
  ScrollHoldController hold(VoidCallback holdCancelCallback);

  /// 开始一个对应于给定 [DragStartDetails] 的 activity，子类实现通常返回一个 ScrollDragController 
  Drag drag(DragStartDetails details, VoidCallback dragCancelCallback);

  /// 当前运行的 [ScrollActivity].
  ///
  /// 没有滚动发生时，activity 为 [IdleScrollActivity]
  ///
  /// 调用 [beginActivity] 来改变当前的 activity
  @protected
  @visibleForTesting
  ScrollActivity get activity => _activity;
  ScrollActivity _activity;

  /// 改变当前的 [activity], 销毁之前的 activity 并且在必要的时候发送 scroll notification
  void beginActivity(ScrollActivity newActivity) {
    if (newActivity == null)
      return;
    bool wasScrolling, oldIgnorePointer;
    if (_activity != null) {
      oldIgnorePointer = _activity.shouldIgnorePointer;
      wasScrolling = _activity.isScrolling;
      /// 如果当前 activity 是滚动并且新的 activity 不是滚动的话
      /// 发送终止滚动通知，并且保存一次 scrollOffset
      if (wasScrolling && !newActivity.isScrolling)
        didEndScroll(); 
      _activity.dispose();
    } else {
      oldIgnorePointer = false;
      wasScrolling = false;
    }
    _activity = newActivity;
    /// oldIgnorePointer 发生变化的时候
    if (oldIgnorePointer != activity.shouldIgnorePointer)
      context.setIgnorePointer(activity.shouldIgnorePointer);
    isScrollingNotifier.value = activity.isScrolling;
    /// 如果当前 activity 不是滚动并且新的 activity 是滚动的话
    /// 发送开始滚动通知
    if (!wasScrolling && _activity.isScrolling)
      didStartScroll();
  }


  // NOTIFICATION DISPATCH

  /// 被 [beginActivity] 调用来报告滚动开始
  void didStartScroll() {
    activity.dispatchScrollStartNotification(copyWith(), context.notificationContext);
  }

  /// 被 [setPixels] 调用来报告 [pixels] 发生了改变，即滚动正在进行中
  void didUpdateScrollPositionBy(double delta) {
    activity.dispatchScrollUpdateNotification(copyWith(), context.notificationContext, delta);
  }

  
  /// 被 [beginActivity] 调用来报告滚动结束
  /// 这个方法调用 [saveScrollOffset] 来保存 scrollOffset
  void didEndScroll() {
    activity.dispatchScrollEndNotification(copyWith(), context.notificationContext);
    if (keepScrollOffset)
      saveScrollOffset();
  }

  /// 被 [setPixels] 调用来报告 试图对 [pixels] 进行改变的时候，发生了过度滚动
  /// 过度滚动的值是没有被运用到 [pixels] 上的值的和
  void didOverscrollBy(double value) {
    activity.dispatchOverscrollNotification(copyWith(), context.notificationContext, value);
  }

  /// 派发通知一个报告 [userScrollDirection] 发生改变的通知
  ///
  /// 子类应当在 [userScrollDirection] 发生改变的时候调用此方法
  void didUpdateScrollDirection(ScrollDirection direction) {
    UserScrollNotification(
        metrics: copyWith(), 
        context: context.notificationContext, 
        direction: direction
    ).dispatch(context.notificationContext);
  }

  @override
  void dispose() {
    activity?.dispose(); // it will be null if it got absorbed by another ScrollPosition
    _activity = null;
    super.dispose();
  }

  @override
  void notifyListeners() {
    _updateSemanticActions(); // will potentially request a semantics update.
    super.notifyListeners();
  }
}

```



## ScrollPositionWithSingleContext

```dart
/// scroll position 管理单个 [ScrollContext] 的活动
///
/// 此类是 [ScrollPosition] 的具体处理单个 [ScrollContext] 逻辑的子类，例如 [Scrollable]
/// 此类的实例管理了 [ScrollActivity] 实例，[ScrollActivity] 能够改变视窗中的那一部分可见
class ScrollPositionWithSingleContext 
    extends ScrollPosition 
    implements ScrollActivityDelegate 
{
  /// Create a [ScrollPosition] object that manages its behavior using
  /// [ScrollActivity] objects.
  ///
  /// The `initialPixels` argument can be null, but in that case it is
  /// imperative that the value be set, using [correctPixels], as soon as
  /// [applyNewDimensions] is invoked, before calling the inherited
  /// implementation of that method.
  ///
  /// 如果 [keepScrollOffset] 为 true (the default), 那么当前的 scroll offset 将会被存储到
  /// [PageStorage] 中，并且在 scrollable 被重建的时候恢复 scoroll offset
  ScrollPositionWithSingleContext({
    @required ScrollPhysics physics,
    @required ScrollContext context,
    double initialPixels = 0.0,
    bool keepScrollOffset = true,
    ScrollPosition oldPosition,
    String debugLabel,
  }) : super(
         physics: physics,
         context: context,
         keepScrollOffset: keepScrollOffset,
         oldPosition: oldPosition,
         debugLabel: debugLabel,
  ) {
    // If oldPosition is not null, the superclass will first call absorb(),
    // which may set _pixels and _activity.
    if (pixels == null && initialPixels != null)
      correctPixels(initialPixels);
    if (activity == null)
      goIdle();
    assert(activity != null);
  }

  /// Velocity from a previous activity temporarily held by [hold] to potentially
  /// transfer to a next activity.
  double _heldPreviousVelocity = 0.0;

  @override
  AxisDirection get axisDirection => context.axisDirection;

  @override
  double setPixels(double newPixels) {
    assert(activity.isScrolling);
    return super.setPixels(newPixels);
  }

  @override
  void absorb(ScrollPosition other) {
    super.absorb(other);
    if (other is! ScrollPositionWithSingleContext) {
      goIdle();
      return;
    }
    activity.updateDelegate(this);
    final ScrollPositionWithSingleContext typedOther = other as ScrollPositionWithSingleContext;
    _userScrollDirection = typedOther._userScrollDirection;
    if (typedOther._currentDrag != null) {
      _currentDrag = typedOther._currentDrag;
      _currentDrag.updateDelegate(this);
      typedOther._currentDrag = null;
    }
  }

  @override
  void applyNewDimensions() {
    super.applyNewDimensions();
    context.setCanDrag(physics.shouldAcceptUserOffset(this));
  }

    
  /// 开始给定 activity
  @override
  void beginActivity(ScrollActivity newActivity) {
    _heldPreviousVelocity = 0.0;
    if (newActivity == null)
      return;
    super.beginActivity(newActivity);
    _currentDrag?.dispose();
    _currentDrag = null;
    if (!activity.isScrolling)
      updateUserScrollDirection(ScrollDirection.idle);
  }

  /// 当用户手指滑动时回调，将 delta（用户滑动举例）作用在 scroll offset 上
  /// 具体实现更新了用户滚动方向且更新了 Pixels
  @override
  void applyUserOffset(double delta) {
    updateUserScrollDirection(delta > 0.0 ? ScrollDirection.forward : ScrollDirection.reverse);
    setPixels(pixels - physics.applyPhysicsToUserOffset(this, delta));
  }

  /// 开始 IdleScrollAcitivity
  @override
  void goIdle() {
    beginActivity(IdleScrollActivity(this));
  }

  /// 启动一个物理驱动的模拟，确定 [pixels] 位置，以特定的速度开始。
  ///
  /// 这个方法延的实现延迟给 [ScrollPhysics.createBallisticSimulation], 它通常在 position 超出边界的时候
  /// 提供了弹性模拟 和 在 position 在边界以内且滑动速度不为 0.0 时的摩擦力模拟
  /// 
  /// 单位为 逻辑像素/秒
  @override
  void goBallistic(double velocity) {
    final Simulation simulation = physics.createBallisticSimulation(this, velocity);
    if (simulation != null) {
      /// 如果有 simulation 的话，开始弹性滚动活动
      beginActivity(BallisticScrollActivity(this, simulation, context.vsync));
    } else {
      /// 否则开始 IdelActivity 
      goIdle();
    }
  }

  @override
  ScrollDirection get userScrollDirection => _userScrollDirection;
  ScrollDirection _userScrollDirection = ScrollDirection.idle;

  /// Set [userScrollDirection] to the given value.
  ///
  /// If this changes the value, then a [UserScrollNotification] is dispatched.
  @protected
  @visibleForTesting
  void updateUserScrollDirection(ScrollDirection value) {
    assert(value != null);
    if (userScrollDirection == value)
      return;
    _userScrollDirection = value;
    didUpdateScrollDirection(value);
  }

    
  /// 以下两个方法用于间接修改 scrollOffset
  @override
  Future<void> animateTo(
    double to, {
    @required Duration duration,
    @required Curve curve,
  }) {
    if (nearEqual(to, pixels, physics.tolerance.distance)) {
      // Skip the animation, go straight to the position as we are already close.
      jumpTo(to);
      return Future<void>.value();
    }

    final DrivenScrollActivity activity = DrivenScrollActivity(
      this,
      from: pixels,
      to: to,
      duration: duration,
      curve: curve,
      vsync: context.vsync,
    );
    beginActivity(activity);
    return activity.done;
  }

  @override
  void jumpTo(double value) {
    goIdle();
    if (pixels != value) {
      final double oldPixels = pixels;
      forcePixels(value);
      notifyListeners();
      didStartScroll();
      didUpdateScrollPositionBy(pixels - oldPixels);
      didEndScroll();
    }
    goBallistic(0.0);
  }

  /// 对应于 handleDragDown 和 handleDragStart 回调
  @override
  ScrollHoldController hold(VoidCallback holdCancelCallback) {
    /// 当前滚动事件的速度
    final double previousVelocity = activity.velocity;
    /// 对应于 hold 状态的 Activity
    final HoldScrollActivity holdActivity = HoldScrollActivity(
      delegate: this,
      onHoldCanceled: holdCancelCallback,
    );
    /// 开始 holdAcitity
    beginActivity(holdActivity);
    _heldPreviousVelocity = previousVelocity;
    /// 由于 holdActivity 实现了 ScrollHoldController，直接返回
    return holdActivity;
  }

  ScrollDragController _currentDrag;

  @override
  Drag drag(DragStartDetails details, VoidCallback dragCancelCallback) {
    final ScrollDragController drag = ScrollDragController(
      delegate: this,
      details: details,
      onDragCanceled: dragCancelCallback,
      carriedVelocity: physics.carriedMomentum(_heldPreviousVelocity),
      motionStartDistanceThreshold: physics.dragStartDistanceMotionThreshold,
    );
    beginActivity(DragScrollActivity(this, drag));
    _currentDrag = drag;
    return drag;
  }

  @override
  void dispose() {
    _currentDrag?.dispose();
    _currentDrag = null;
    super.dispose();
  }
}

```




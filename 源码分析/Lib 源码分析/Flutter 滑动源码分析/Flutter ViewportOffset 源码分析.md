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

  /// 停止当前的 activity 并且开始一个 [HoldScrollActivity].
  ScrollHoldController hold(VoidCallback holdCancelCallback);

  /// 开始一个对应于给定 [DragStartDetails] 的 activity
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
    UserScrollNotification(metrics: copyWith(), context: context.notificationContext, direction: direction).dispatch(context.notificationContext);
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



## ScrollActivity 及其子类相关

### ScrollActivityDelegate 

```dart
/// 子类是 ScrollPositionWithSingleContext
abstract class ScrollActivityDelegate {
  AxisDirection get axisDirection;
  double setPixels(double pixels);

  /// 更新 sroll positon
  /// 通常应用 [ScrollPhysics.applyPhysicsToUserOffset] 或者其他时候适合用户发起的滚动
  void applyUserOffset(double delta);

  /// Terminate the current activity and start an idle activity.
  void goIdle();

  /// Terminate the current activity and start a ballistic activity with the
  /// given velocity.
  void goBallistic(double velocity);
}
```



### ScrollActivity

```dart
/// scrolling activities，例如拖动或者抛投，的基类
abstract class ScrollActivity {
  ScrollActivity(this._delegate);

  ScrollActivityDelegate get delegate => _delegate;
  ScrollActivityDelegate _delegate;

  void updateDelegate(ScrollActivityDelegate value) {
    _delegate = value;
  }

  void resetActivity() { }

  /// 用给定的 ScrollMetrics ,派发 [ScrollStartNotification]
  void dispatchScrollStartNotification(ScrollMetrics metrics, BuildContext context) {
    ScrollStartNotification(metrics: metrics, context: context)
        .dispatch(context);
  }

  /// 用给定的 ScrollMetrics 和 delta,派发 [ScrollUpdateNotification]
  void dispatchScrollUpdateNotification(
      ScrollMetrics metrics, 
      BuildContext context, 
      double scrollDelta
  ) {
    ScrollUpdateNotification(metrics: metrics, context: context, scrollDelta: scrollDelta)
        .dispatch(context);
  }

  /// 用给定的 ScrollMetrics 和 overscroll,派发 [OverscrollNotification]
  void dispatchOverscrollNotification(
      ScrollMetrics metrics, 
      BuildContext context, 
      double overscroll
  ) {
    OverscrollNotification(metrics: metrics, context: context, overscroll: overscroll)
        .dispatch(context);
  }

   /// 用给定的 ScrollMetrics ,派发 [crollEndNotification]
  void dispatchScrollEndNotification(ScrollMetrics metrics, BuildContext context) {
    ScrollEndNotification(metrics: metrics, context: context)
        .dispatch(context);
  }

  /// Called when the scroll view that is performing this activity changes its metrics.
  void applyNewDimensions() { }

  /// 当这个 activity 进行的时候，时候应该忽略指针
  bool get shouldIgnorePointer;

  /// 这个 activity 是否是持续滚动
  bool get isScrolling;

  /// 如果适用，这个值代表当前滚动偏移的速度，以逻辑像素/秒为单位独立变化(即没有外部刺激，如拖动手势)。
  double get velocity;

  @mustCallSuper
  void dispose() {
    _delegate = null;
  }
}
```
### IdleScrollActivity
```dart
/// 什么也不做的活动
class IdleScrollActivity extends ScrollActivity {
  /// Creates a scroll activity that does nothing.
  IdleScrollActivity(ScrollActivityDelegate delegate) : super(delegate);

  @override
  void applyNewDimensions() {
    delegate.goBallistic(0.0);
  }

  @override
  bool get shouldIgnorePointer => false;

  @override
  bool get isScrolling => false;

  @override
  double get velocity => 0.0;
}
```
### ScrollHoldController
```dart
/// 保持 [Scrollable] 静止不动
///
/// [ScrollPosition.hold] 返回了一个实现此接口的对象. I它保持了这个 [Scrollable] 静止不动，直到一个新的 
/// acitivity 被发起，或者 [cancel] 方法被调用
abstract class ScrollHoldController {
  void cancel();
}
```
### HoldScrollActivity
```dart
/// 用户接触 scrollView 但是保持它静止不动（即还没有开始滑动的时候）
class HoldScrollActivity extends ScrollActivity implements ScrollHoldController {
  HoldScrollActivity({
    @required ScrollActivityDelegate delegate,
    this.onHoldCanceled,
  }) : super(delegate);

  /// 当 [dispose] 调用的时候的回调
  final VoidCallback onHoldCanceled;

  @override
  bool get shouldIgnorePointer => false;

  @override
  bool get isScrolling => false;

  @override
  double get velocity => 0.0;

  @override
  void cancel() {
    delegate.goBallistic(0.0);
  }

  @override
  void dispose() {
    if (onHoldCanceled != null)
      onHoldCanceled();
    super.dispose();
  }
}
```

### Drag
```dart
abstract class Drag {
  /// 指针开始移动
  void update(DragUpdateDetails details) { }

  /// 指针不再与屏幕解除
  void end(DragEndDetails details) { }

  /// 取消拖动
  void cancel() { }
}

```

### ScrollDragController

```dart
/// 当用户划过屏幕的时候滚动 scroll view
class ScrollDragController implements Drag {
  ScrollDragController({
    @required ScrollActivityDelegate delegate,
    @required DragStartDetails details,
    this.onDragCanceled,
    this.carriedVelocity,
    this.motionStartDistanceThreshold,
  }) : delegate = delegate,
       _lastDetails = details,
       _retainMomentum = carriedVelocity != null && carriedVelocity != 0.0,
       _lastNonStationaryTimestamp = details.sourceTimeStamp,
       _offsetSinceLastStop = motionStartDistanceThreshold == null ? null : 0.0;

  /// The object that will actuate the scroll view as the user drags.
  ScrollActivityDelegate get delegate => _delegate;
  ScrollActivityDelegate _delegate;

  /// Called when [dispose] is called.
  final VoidCallback onDragCanceled;

  /// Velocity that was present from a previous [ScrollActivity] when this drag
  /// began.
  final double carriedVelocity;

  /// Amount of pixels in either direction the drag has to move by to start
  /// scroll movement again after each time scrolling came to a stop.
  final double motionStartDistanceThreshold;

  Duration _lastNonStationaryTimestamp;
  bool _retainMomentum;
  /// Null if already in motion or has no [motionStartDistanceThreshold].
  double _offsetSinceLastStop;

  /// Maximum amount of time interval the drag can have consecutive stationary
  /// pointer update events before losing the momentum carried from a previous
  /// scroll activity.
  static const Duration momentumRetainStationaryDurationThreshold =
      Duration(milliseconds: 20);

  /// Maximum amount of time interval the drag can have consecutive stationary
  /// pointer update events before needing to break the
  /// [motionStartDistanceThreshold] to start motion again.
  static const Duration motionStoppedDurationThreshold =
      Duration(milliseconds: 50);

  /// The drag distance past which, a [motionStartDistanceThreshold] breaking
  /// drag is considered a deliberate fling.
  static const double _bigThresholdBreakDistance = 24.0;

  bool get _reversed => axisDirectionIsReversed(delegate.axisDirection);

  /// Updates the controller's link to the [ScrollActivityDelegate].
  ///
  /// This should only be called when a controller is being moved from a defunct
  /// (or about-to-be defunct) [ScrollActivityDelegate] object to a new one.
  void updateDelegate(ScrollActivityDelegate value) {
    assert(_delegate != value);
    _delegate = value;
  }

  /// Determines whether to lose the existing incoming velocity when starting
  /// the drag.
  void _maybeLoseMomentum(double offset, Duration timestamp) {
    if (_retainMomentum &&
        offset == 0.0 &&
        (timestamp == null || // If drag event has no timestamp, we lose momentum.
         timestamp - _lastNonStationaryTimestamp > momentumRetainStationaryDurationThreshold)) {
      // If pointer is stationary for too long, we lose momentum.
      _retainMomentum = false;
    }
  }

  /// If a motion start threshold exists, determine whether the threshold needs
  /// to be broken to scroll. Also possibly apply an offset adjustment when
  /// threshold is first broken.
  ///
  /// Returns `0.0` when stationary or within threshold. Returns `offset`
  /// transparently when already in motion.
  double _adjustForScrollStartThreshold(double offset, Duration timestamp) {
    if (timestamp == null) {
      // If we can't track time, we can't apply thresholds.
      // May be null for proxied drags like via accessibility.
      return offset;
    }
    if (offset == 0.0) {
      if (motionStartDistanceThreshold != null &&
          _offsetSinceLastStop == null &&
          timestamp - _lastNonStationaryTimestamp > motionStoppedDurationThreshold) {
        // Enforce a new threshold.
        _offsetSinceLastStop = 0.0;
      }
      // Not moving can't break threshold.
      return 0.0;
    } else {
      if (_offsetSinceLastStop == null) {
        // Already in motion or no threshold behavior configured such as for
        // Android. Allow transparent offset transmission.
        return offset;
      } else {
        _offsetSinceLastStop += offset;
        if (_offsetSinceLastStop.abs() > motionStartDistanceThreshold) {
          // Threshold broken.
          _offsetSinceLastStop = null;
          if (offset.abs() > _bigThresholdBreakDistance) {
            // This is heuristically a very deliberate fling. Leave the motion
            // unaffected.
            return offset;
          } else {
            // This is a normal speed threshold break.
            return math.min(
              // Ease into the motion when the threshold is initially broken
              // to avoid a visible jump.
              motionStartDistanceThreshold / 3.0,
              offset.abs(),
            ) * offset.sign;
          }
        } else {
          return 0.0;
        }
      }
    }
  }

  @override
  void update(DragUpdateDetails details) {
    assert(details.primaryDelta != null);
    _lastDetails = details;
    double offset = details.primaryDelta;
    if (offset != 0.0) {
      _lastNonStationaryTimestamp = details.sourceTimeStamp;
    }
    // By default, iOS platforms carries momentum and has a start threshold
    // (configured in [BouncingScrollPhysics]). The 2 operations below are
    // no-ops on Android.
    _maybeLoseMomentum(offset, details.sourceTimeStamp);
    offset = _adjustForScrollStartThreshold(offset, details.sourceTimeStamp);
    if (offset == 0.0) {
      return;
    }
    if (_reversed) // e.g. an AxisDirection.up scrollable
      offset = -offset;
    delegate.applyUserOffset(offset);
  }

  @override
  void end(DragEndDetails details) {
    assert(details.primaryVelocity != null);
    // We negate the velocity here because if the touch is moving downwards,
    // the scroll has to move upwards. It's the same reason that update()
    // above negates the delta before applying it to the scroll offset.
    double velocity = -details.primaryVelocity;
    if (_reversed) // e.g. an AxisDirection.up scrollable
      velocity = -velocity;
    _lastDetails = details;

    // Build momentum only if dragging in the same direction.
    if (_retainMomentum && velocity.sign == carriedVelocity.sign)
      velocity += carriedVelocity;
    delegate.goBallistic(velocity);
  }

  @override
  void cancel() {
    delegate.goBallistic(0.0);
  }

  /// Called by the delegate when it is no longer sending events to this object.
  @mustCallSuper
  void dispose() {
    _lastDetails = null;
    if (onDragCanceled != null)
      onDragCanceled();
  }

  /// The most recently observed [DragStartDetails], [DragUpdateDetails], or
  /// [DragEndDetails] object.
  dynamic get lastDetails => _lastDetails;
  dynamic _lastDetails;

  @override
  String toString() => describeIdentity(this);
}
```
### DragScrollActivity
```dart
/// The activity a scroll view performs when a the user drags their finger
/// across the screen.
///
/// See also:
///
///  * [ScrollDragController], which listens to the [Drag] and actually scrolls
///    the scroll view.
class DragScrollActivity extends ScrollActivity {
  /// Creates an activity for when the user drags their finger across the
  /// screen.
  DragScrollActivity(
    ScrollActivityDelegate delegate,
    ScrollDragController controller,
  ) : _controller = controller,
      super(delegate);

  ScrollDragController _controller;

  @override
  void dispatchScrollStartNotification(ScrollMetrics metrics, BuildContext context) {
    final dynamic lastDetails = _controller.lastDetails;
    assert(lastDetails is DragStartDetails);
    ScrollStartNotification(metrics: metrics, context: context, dragDetails: lastDetails as DragStartDetails).dispatch(context);
  }

  @override
  void dispatchScrollUpdateNotification(ScrollMetrics metrics, BuildContext context, double scrollDelta) {
    final dynamic lastDetails = _controller.lastDetails;
    assert(lastDetails is DragUpdateDetails);
    ScrollUpdateNotification(metrics: metrics, context: context, scrollDelta: scrollDelta, dragDetails: lastDetails as DragUpdateDetails).dispatch(context);
  }

  @override
  void dispatchOverscrollNotification(ScrollMetrics metrics, BuildContext context, double overscroll) {
    final dynamic lastDetails = _controller.lastDetails;
    assert(lastDetails is DragUpdateDetails);
    OverscrollNotification(metrics: metrics, context: context, overscroll: overscroll, dragDetails: lastDetails as DragUpdateDetails).dispatch(context);
  }

  @override
  void dispatchScrollEndNotification(ScrollMetrics metrics, BuildContext context) {
    // We might not have DragEndDetails yet if we're being called from beginActivity.
    final dynamic lastDetails = _controller.lastDetails;
    ScrollEndNotification(
      metrics: metrics,
      context: context,
      dragDetails: lastDetails is DragEndDetails ? lastDetails : null,
    ).dispatch(context);
  }

  @override
  bool get shouldIgnorePointer => true;

  @override
  bool get isScrolling => true;

  // DragScrollActivity is not independently changing velocity yet
  // until the drag is ended.
  @override
  double get velocity => 0.0;

  @override
  void dispose() {
    _controller = null;
    super.dispose();
  }

  @override
  String toString() {
    return '${describeIdentity(this)}($_controller)';
  }
}
```
### BallisticScrollActivity
```dart
/// An activity that animates a scroll view based on a physics [Simulation].
///
/// A [BallisticScrollActivity] is typically used when the user lifts their
/// finger off the screen to continue the scrolling gesture with the current velocity.
///
/// [BallisticScrollActivity] is also used to restore a scroll view to a valid
/// scroll offset when the geometry of the scroll view changes. In these
/// situations, the [Simulation] typically starts with a zero velocity.
///
/// See also:
///
///  * [DrivenScrollActivity], which animates a scroll view based on a set of
///    animation parameters.
class BallisticScrollActivity extends ScrollActivity {
  /// Creates an activity that animates a scroll view based on a [simulation].
  ///
  /// The [delegate], [simulation], and [vsync] arguments must not be null.
  BallisticScrollActivity(
    ScrollActivityDelegate delegate,
    Simulation simulation,
    TickerProvider vsync,
  ) : super(delegate) {
    _controller = AnimationController.unbounded(
      debugLabel: kDebugMode ? '$runtimeType' : null,
      vsync: vsync,
    )
      ..addListener(_tick)
      ..animateWith(simulation)
       .whenComplete(_end); // won't trigger if we dispose _controller first
  }

  @override
  double get velocity => _controller.velocity;

  AnimationController _controller;

  @override
  void resetActivity() {
    delegate.goBallistic(velocity);
  }

  @override
  void applyNewDimensions() {
    delegate.goBallistic(velocity);
  }

  void _tick() {
    if (!applyMoveTo(_controller.value))
      delegate.goIdle();
  }

  /// Move the position to the given location.
  ///
  /// If the new position was fully applied, returns true. If there was any
  /// overflow, returns false.
  ///
  /// The default implementation calls [ScrollActivityDelegate.setPixels]
  /// and returns true if the overflow was zero.
  @protected
  bool applyMoveTo(double value) {
    return delegate.setPixels(value) == 0.0;
  }

  void _end() {
    delegate?.goBallistic(0.0);
  }

  @override
  void dispatchOverscrollNotification(ScrollMetrics metrics, BuildContext context, double overscroll) {
    OverscrollNotification(metrics: metrics, context: context, overscroll: overscroll, velocity: velocity).dispatch(context);
  }

  @override
  bool get shouldIgnorePointer => true;

  @override
  bool get isScrolling => true;

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  String toString() {
    return '${describeIdentity(this)}($_controller)';
  }
}
```
### DrivenScrollActivity
```dart
/// An activity that animates a scroll view based on animation parameters.
///
/// For example, a [DrivenScrollActivity] is used to implement
/// [ScrollController.animateTo].
///
/// See also:
///
///  * [BallisticScrollActivity], which animates a scroll view based on a
///    physics [Simulation].
class DrivenScrollActivity extends ScrollActivity {
  /// Creates an activity that animates a scroll view based on animation
  /// parameters.
  ///
  /// All of the parameters must be non-null.
  DrivenScrollActivity(
    ScrollActivityDelegate delegate, {
    @required double from,
    @required double to,
    @required Duration duration,
    @required Curve curve,
    @required TickerProvider vsync,
  }) : super(delegate) {
    _completer = Completer<void>();
    _controller = AnimationController.unbounded(
      value: from,
      debugLabel: '$runtimeType',
      vsync: vsync,
    )
      ..addListener(_tick)
      ..animateTo(to, duration: duration, curve: curve)
       .whenComplete(_end); // won't trigger if we dispose _controller first
  }

  Completer<void> _completer;
  AnimationController _controller;

  /// A [Future] that completes when the activity stops.
  ///
  /// For example, this [Future] will complete if the animation reaches the end
  /// or if the user interacts with the scroll view in way that causes the
  /// animation to stop before it reaches the end.
  Future<void> get done => _completer.future;

  @override
  double get velocity => _controller.velocity;

  void _tick() {
    if (delegate.setPixels(_controller.value) != 0.0)
      delegate.goIdle();
  }

  void _end() {
    delegate?.goBallistic(velocity);
  }

  @override
  void dispatchOverscrollNotification(ScrollMetrics metrics, BuildContext context, double overscroll) {
    OverscrollNotification(metrics: metrics, context: context, overscroll: overscroll, velocity: velocity).dispatch(context);
  }

  @override
  bool get shouldIgnorePointer => true;

  @override
  bool get isScrolling => true;

  @override
  void dispose() {
    _completer.complete();
    _controller.dispose();
    super.dispose();
  }

  @override
  String toString() {
    return '${describeIdentity(this)}($_controller)';
  }
}

```


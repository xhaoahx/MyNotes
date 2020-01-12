

# Flutter ScrollActivity 源码分析

## ScrollActivityDelegate 

```dart
/// 子类是 ScrollPositionWithSingleContext
abstract class ScrollActivityDelegate {
  AxisDirection get axisDirection;
  double setPixels(double pixels);

  /// 更新 sroll positon
  /// 通常应用 [ScrollPhysics.applyPhysicsToUserOffset] 或者其他时候适合用户发起的滚动
  void applyUserOffset(double delta);

  /// 终止当前的活动并且开始 idle activity.
  void goIdle();

  /// 终止当前的活动并且按照给定的 velocity 开始 ballistic activity
  void goBallistic(double velocity);
}
```



## ScrollActivity

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

  /// 事件派发
  /// 派发滚动中的各种事件，例如滚动开始、结束等
    
  /// 用给定的 ScrollMetrics,派发 [ScrollStartNotification]
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

  /// 当执行此活动的滚动视图更改其维度时调用。
  void applyNewDimensions() { }

  /// 当这个 activity 进行的时候，是否应该忽略指针
  bool get shouldIgnorePointer;

  /// 这个 activity 是否是持续滚动
  bool get isScrolling;

  /// 如果可用，这个值代表当前滚动偏移的速度，以逻辑像素/秒为单位独立变化(即没有外部刺激，如拖动手势)。
  double get velocity;

  @mustCallSuper
  void dispose() {
    _delegate = null;
  }
}
```

## IdleScrollActivity

```dart
/// 什么也不做的活动
class IdleScrollActivity extends ScrollActivity {
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

## ScrollHoldController

```dart
/// 保持 [Scrollable] 静止不动
///
/// [ScrollPosition.hold] 返回了一个实现此接口的对象. 它保持了这个 [Scrollable] 静止不动，直到一个新的 
/// acitivity 被发起，或者 [cancel] 方法被调用
abstract class ScrollHoldController {
  void cancel();
}
```

## HoldScrollActivity

```dart
/// 用户接触 scrollView 但是保持它静止不动（即还没有开始滑动的时候）
class HoldScrollActivity 
    extends ScrollActivity 
    implements ScrollHoldController 
{
    
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







## Drag

```dart
abstract class Drag {
  /// 指针开始移动
  void update(DragUpdateDetails details) { }

  /// 指针不再与屏幕接触
  void end(DragEndDetails details) { }

  /// 取消拖动
  void cancel() { }
}

```

## ScrollDragController

```dart
/// 当用户划过屏幕的时候滚动 scroll view
class ScrollDragController implements Drag {
  ScrollDragController({
    @required ScrollActivityDelegate delegate,
    
    @required DragStartDetails details,
    /// 滑动取消后，调用此回调。
    /// scrollable 的具体实现是将 scrollable 的 darg 字段置 null
    this.onDragCanceled,
    /// 上一个 activity 对应的速度
    this.carriedVelocity,
    this.motionStartDistanceThreshold,
  }) : delegate = delegate,
       _lastDetails = details,
       _retainMomentum = carriedVelocity != null && carriedVelocity != 0.0,
       _lastNonStationaryTimestamp = details.sourceTimeStamp,
       _offsetSinceLastStop = motionStartDistanceThreshold == null ? null : 0.0;

  /// 在用户拖动时触发滚动视图的对象。
  /// 其实现类之一是 ScrollPositionWithSingleContext 
  ScrollActivityDelegate get delegate => _delegate;
  ScrollActivityDelegate _delegate;

  /// 注销回调
  final VoidCallback onDragCanceled;

  /// 当这个 activity 开始的时候，上一个 [ScrollActivity] 对应的速度
  final double carriedVelocity;

  /// 在滚动停止后，要移动多少像素才能重新开始滚动
  final double motionStartDistanceThreshold;

  Duration _lastNonStationaryTimestamp;
  bool _retainMomentum;
  /// Null if already in motion or has no [motionStartDistanceThreshold].
  double _offsetSinceLastStop;

  /// 在失去上一个 scroll activity 的滚动动量之前，drag 能够持有连续静态指针更新事件的最大时间间隔
  static const Duration momentumRetainStationaryDurationThreshold =
      Duration(milliseconds: 20);

  /// Maximum amount of time interval the drag can have consecutive stationary
  /// pointer update events before needing to break the
  /// [motionStartDistanceThreshold] to start motion again.
  static const Duration motionStoppedDurationThreshold =
      Duration(milliseconds: 50);

  /// 超过这个距离，一个打破 [motionStartDistanceThreshold] 的拖拽被认为是故意抛出的。
  static const double _bigThresholdBreakDistance = 24.0;

  bool get _reversed => axisDirectionIsReversed(delegate.axisDirection);

  /// 更新持有的 [ScrollActivityDelegate].
  void updateDelegate(ScrollActivityDelegate value) {
    assert(_delegate != value);
    _delegate = value;
  }

  /// 确定开始拖动时是否丢失现有的传入速度.
  void _maybeLoseMomentum(double offset, Duration timestamp) {
    if (_retainMomentum 
     && offset == 0.0 
     && (timestamp == null // If drag event has no timestamp, we lose momentum.
      || timestamp - _lastNonStationaryTimestamp > momentumRetainStationaryDurationThreshold)
   ) {
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

    
  /// 以下方法实现对应于 handleDragUpdate 和 handleDragEnd 回调
  @override
  void update(DragUpdateDetails details) {
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
    /// 调用 delegate.applyUserOffset 更新 scrollOffset
    delegate.applyUserOffset(offset);
  }

  @override
  void end(DragEndDetails details) {
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
    /// 进行惯性运动
    delegate.goBallistic(velocity);
  }

  @override
  void cancel() {
    /// 做速度为 0.0 的惯性运动，也就是禁止
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

## DragScrollActivity

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
    ScrollStartNotification(
        metrics: metrics, 
        context: context, 
        dragDetails: lastDetails as DragStartDetails
    ).dispatch(context);
  }

  @override
  void dispatchScrollUpdateNotification(
      ScrollMetrics metrics, 
      BuildContext context, 
      double scrollDelta
  ) {
    final dynamic lastDetails = _controller.lastDetails;
    assert(lastDetails is DragUpdateDetails);
    ScrollUpdateNotification(
        metrics: metrics, 
        context: context, 
        scrollDelta: scrollDelta, 
        dragDetails: lastDetails as DragUpdateDetails
    ).dispatch(context);
  }

  @override
  void dispatchOverscrollNotification(ScrollMetrics metrics, BuildContext context, double overscroll) {
    final dynamic lastDetails = _controller.lastDetails;
    assert(lastDetails is DragUpdateDetails);
    OverscrollNotification(
        metrics: metrics, 
        context: context, 
        overscroll: overscroll, 
        dragDetails: lastDetails as DragUpdateDetails
    ).dispatch(context);
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
}
```

## BallisticScrollActivity

```dart
/// 一个基于 [Simulation] 使 scroll view 滚动的 activity
///
/// [BallisticScrollActivity] 通常被用于用户抬起手指，scroll view 依照当前（速度）惯性滚动
///
/// [BallisticScrollActivity] 也通常被用于：但 scroll view 的维度发生了改变的时候，将 scroll view 恢复到有
/// 效，这种情况下 [Simulation] 通常以 0 速度开始
class BallisticScrollActivity extends ScrollActivity {
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

## DrivenScrollActivity

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
}

```


# Flutter 滚动



[TOC]

## ScrollContext

```dart
abstract class ScrollContext {
  /// 用于发送[ScrollNotification]s的BuildContext
  BuildContext get notificationContext;
    
  BuildContext get storageContext;

  /// A [TickerProvider] to use when animating the scroll position.
  TickerProvider get vsync;

  /// The direction in which the widget scrolls.
  AxisDirection get axisDirection;

  void setIgnorePointer(bool value);

  void setCanDrag(bool value);

  ...
}

```



## PageStorage

```dart
/// 为子树建立一个PageStorage,可以用于储存数据
class PageStorage extends StatelessWidget {
  const PageStorage({
    Key key,
    @required this.bucket,
    @required this.child,
  }) : assert(bucket != null),
       super(key: key);
    
  final Widget child;
  final PageStorageBucket bucket;

  /// 返回最近的 PageStorage.bucket 实例
  static PageStorageBucket of(BuildContext context) {
    final PageStorage widget = context.ancestorWidgetOfExactType(PageStorage);
    return widget?.bucket;
  }

  @override
  Widget build(BuildContext context) => child;
}

/// 简单封装ValueKey
class PageStorageKey<T> extends ValueKey<T> {
  const PageStorageKey(T value) : super(value);
}

/// 用于判断两组 keys 是否完全相等
/// 其中 ValueKey 的 == 方法实现如下
/// @override
///   bool operator ==(dynamic other) {
///     if (other.runtimeType != runtimeType)
///       return false;
///     final ValueKey<T> typedOther = other;
///     return value == typedOther.value;
///   }
class _StorageEntryIdentifier {
  _StorageEntryIdentifier(this.keys)
    : assert(keys != null);

  final List<PageStorageKey<dynamic>> keys;

  bool get isNotEmpty => keys.isNotEmpty;

  @override
  bool operator ==(dynamic other) {
    if (other.runtimeType != runtimeType)
      return false;
    final _StorageEntryIdentifier typedOther = other;
    for (int index = 0; index < keys.length; index += 1) {
      if (keys[index] != typedOther.keys[index])
        return false;
    }
    return true;
  }

  @override
  int get hashCode => hashList(keys);
}

class PageStorageBucket {
  static bool _maybeAddKey(
      BuildContext context, 
      List<PageStorageKey<dynamic>> keys) 
  {
    final Widget widget = context.widget;
    final Key key = widget.key;
    /// 只用当 Key 是PageStorageKey 时，才能加入到 keys 中
    if (key is PageStorageKey)
      keys.add(key);
    return widget is! PageStorage;
  }

  /// 遍历先祖，获取所有 PageStorageKey
  List<PageStorageKey<dynamic>> _allKeys(BuildContext context) {
    final List<PageStorageKey<dynamic>> keys = <PageStorageKey<dynamic>>[];
    if (_maybeAddKey(context, keys)) {
      context.visitAncestorElements((Element element) {
        return _maybeAddKey(element, keys);
      });
    }
    return keys;
  }

  _StorageEntryIdentifier _computeIdentifier(BuildContext context) {
    return _StorageEntryIdentifier(_allKeys(context));
  }

  Map<Object, dynamic> _storage;

  /// 将 identifier 作为 Key 值写入给定的数据
  /// 在 ScrollPostion 里有如下调用： 
  /// PageStorage.of(context.storageContext)?.writeState(context.storageContext, pixels);
  /// 写入了 pixels  
  void writeState(BuildContext context, dynamic data, { Object identifier }) {
    _storage ??= <Object, dynamic>{};
    if (identifier != null) {
      _storage[identifier] = data;
    } else {
      final _StorageEntryIdentifier contextIdentifier = _computeIdentifier(context);
      if (contextIdentifier.isNotEmpty)
        _storage[contextIdentifier] = data;
    }
  }

  /// 读取数据
  dynamic readState(BuildContext context, { Object identifier }) {
    if (_storage == null)
      return null;
    if (identifier != null)
      return _storage[identifier];
    final _StorageEntryIdentifier contextIdentifier = _computeIdentifier(context);
    return contextIdentifier.isNotEmpty ? _storage[contextIdentifier] : null;
  }
}
```



## ViewportOffset

```dart
/// 当[pixels]发生改变时，通知 listener
abstract class ViewportOffset extends ChangeNotifier {
  ViewportOffset();

  /// 调用 _FixedViewportOffset 的构造方法
  /// 创建一个[pixels]不可变的对象（除非使用 correctPixels 来修正pixels）  
  factory ViewportOffset.fixed(double value) = _FixedViewportOffset;

  factory ViewportOffset.zero() = _FixedViewportOffset.zero;

  /// 偏移子轴方向相反方向的像素值
  ///
  /// 例如，如果轴方向是向下的，那么pixels表示子元素向上移出屏幕的逻辑像素
  /// 类似地，如果轴方向是向左的，则pixels表示子元素向右移出屏幕的逻辑像素
  ///
  double get pixels;

  /// Called when the viewport's extents are established.
  ///
  /// The argument is the dimension of the [RenderViewport] in the main axis
  /// (e.g. the height, for a vertical viewport).
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

  /// 在布局时更新偏移量
  void correctBy(double correction);

  void jumpTo(double pixels);

  Future<void> animateTo(
    double to, {
    @required Duration duration,
    @required Curve curve,
  });

  /// 当 duration == null 或者 duration == Duration.zero 时，调用jumpTo
  /// 否则调用 animateTo 
  Future<void> moveTo(
    double to, {
    Duration duration,
    Curve curve,
    bool clamp,
  }) {
    if (duration == null || duration == Duration.zero) {
      jumpTo(to);
      return Future<void>.value();
    } else {
      return animateTo(to, duration: duration, curve: curve ?? Curves.ease);
    }
  }

  /// 使用者的滑动方向
  ScrollDirection get userScrollDirection;

  /// Whether a viewport is allowed to change [pixels] implicitly to respond to
  /// a call to [RenderObject.showOnScreen].
  ///
  /// [RenderObject.showOnScreen] is for example used to bring a text field
  /// fully on screen after it has received focus. This property controls
  /// whether the viewport associated with this offset is allowed to change the
  /// offset's [pixels] value to fulfill such a request.
  bool get allowImplicitScrolling;
 
  ...  
}
```



## ScrollMetrics

```dart
/// 描述[Scrollable]的内容
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

  /// 最小的“在边界里”[pixels]值
  ///
  /// 实际上，[pixels]可能会出界
  ///
  /// 这个值应该非null并且小于[maxScrollExtent]
  /// 当滚动没有边界时，这个值可以是无穷小
  double get minScrollExtent;

  /// 最大的“在边界里”[pixels]值
  ///
  /// 实际上，[pixels]可能会出界
  ///
  /// 这个值应该非null并且大于[maxScrollExtent]
  /// 当滚动没有边界时，这个值可以是无穷大
  double get maxScrollExtent;

  /// 当前滚动位置，沿[axisDirection]，以逻辑像素表示
  double get pixels;

  /// viewport 沿着 [axisDirection] 的大小
  double get viewportDimension;

  /// 轴方向
  AxisDirection get axisDirection;

  /// 滚动view的方向
  Axis get axis => axisDirectionToAxis(axisDirection);

  /// [pixels] 是否处于 [minScrollExtent] 和 [maxScrollExtent] 之间
  bool get outOfRange => pixels < minScrollExtent || pixels > maxScrollExtent;

  /// 是否处于滚动边界
  bool get atEdge => pixels == minScrollExtent || pixels == maxScrollExtent;

  /// /// 在视窗上方 scrollable 的内容的长度
  double get extentBefore => math.max(pixels - minScrollExtent, 0.0);

  /// 视窗之内的 scrollable 的长度
  /// 当 outOfRange == false 时，这个值就等于  viewportDimension
  /// 当 overscroll 发生时，这个值会变小，因此，显示的内容也会变少
 
  double get extentInside {
    return viewportDimension
      // 上方 overscroll 的值（向下拉动越界）
      - (minScrollExtent - pixels).clamp(0, viewportDimension)
      // 下方 overscroll 的值（向上拉动越界）
      - (pixels - maxScrollExtent).clamp(0, viewportDimension);
  }

  /// 在视窗下方 scrollable 的内容的长度
  double get extentAfter => math.max(maxScrollExtent - pixels, 0.0);
}
```



## ScrollPosition

```dart
/// 决定了 content 在 scroll view 中的显示比例
///
/// [pixels] 决定了 view 的偏移量。滑动时，这个值会改变。content的显示也随之改变
abstract class ScrollPosition extends ViewportOffset with ScrollMetrics {
  ScrollPosition({
    @required this.physics,
    @required this.context,
    this.keepScrollOffset = true,
    ScrollPosition oldPosition,
    this.debugLabel,
  }) : assert(physics != null),
       assert(context != null),
       assert(context.vsync != null),
       assert(keepScrollOffset != null) {
    if (oldPosition != null)
      absorb(oldPosition);
    if (keepScrollOffset)
      restoreScrollOffset();
  }

  /// 滚动规律
  final ScrollPhysics physics;
    
  final ScrollContext context;

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

  bool get haveDimensions => _haveDimensions;
  bool _haveDimensions = false;

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

  /// 修正 _pixels ，不会通知 listener
  void correctPixels(double value) {
    _pixels = value;
  }

  /// 在布局时修正 _pixels
  @override
  void correctBy(double correction) {
    _pixels += correction;
    _didChangeViewportDimensionOrReceiveCorrection = true;
  }

  /// 修改 Pixels ，并通知 listeners。这个通常被用于实现 jumpTo
  @protected
  void forcePixels(double value) {
    _pixels = value;
    notifyListeners();
  }

  /// 无论何时滚动结束，调用这个方法  
  @protected
  void saveScrollOffset() {
    PageStorage.of(context.storageContext)?.writeState(context.storageContext, pixels);
  }

  /// 当可滚动对象重建时，调用这个方法
  @protected
  void restoreScrollOffset() {
    if (pixels == null) {
      final double value = 
          PageStorage.of(context.storageContext)?.readState(context.storageContext);
      if (value != null)
        correctPixels(value);
    }
  }

  /// 根据边界条件返回 overscroll
  ///
  /// 如果给定的值在边界内，返回 0.0。否则，返回无法附加给pixels的总和值
  ///
  /// 由[ScrollPhysics.applyBoundaryConditions]具体实现
  @protected
  double applyBoundaryConditions(double value) {
    final double result = physics.applyBoundaryConditions(this, value);
    return result;
  }

  /// 标记，是否需要从重新改变ViewportDimension大小  
  bool _didChangeViewportDimensionOrReceiveCorrection = true;

  @override
  bool applyViewportDimension(double viewportDimension) {
    if (_viewportDimension != viewportDimension) {
      _viewportDimension = viewportDimension;  
      _didChangeViewportDimensionOrReceiveCorrection = true;
    }
    return true;
  }

  ...

  @override
  bool applyContentDimensions(double minScrollExtent, double maxScrollExtent) {
    if (!nearEqual(
            _minScrollExtent, 
            minScrollExtent, 
            Tolerance.defaultTolerance.distance) ||
        !nearEqual(
            _maxScrollExtent, 
            maxScrollExtent, 
            Tolerance.defaultTolerance.distance) ||
        _didChangeViewportDimensionOrReceiveCorrection) 
    {
      _minScrollExtent = minScrollExtent;
      _maxScrollExtent = maxScrollExtent;
      _haveDimensions = true;
      applyNewDimensions();
      _didChangeViewportDimensionOrReceiveCorrection = false;
    }
    return true;
  }

  /// 通知 activity 底层 viewport 或者 content 发生了改变
  ///
  @protected
  @mustCallSuper
  void applyNewDimensions() {
    activity.applyNewDimensions();
    /// 语义更新，略过  
    _updateSemanticActions(); // will potentially request a semantics update.
  }

  Future<void> ensureVisible(
    RenderObject object, {
    double alignment = 0.0,
    Duration duration = Duration.zero,
    Curve curve = Curves.ease,
  }) {
    final RenderAbstractViewport viewport = RenderAbstractViewport.of(object);
    final double target = 
        viewport.getOffsetToReveal(object, alignment).offset.
        clamp(minScrollExtent, maxScrollExtent);

    if (target == pixels)
      return Future<void>.value();

    if (duration == Duration.zero) {
      jumpTo(target);
      return Future<void>.value();
    }

    return animateTo(target, duration: duration, curve: curve);
  }

  /// 每当 isScrolling 发生改变时，发出通知 
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
      to = to.clamp(minScrollExtent, maxScrollExtent);
    return super.moveTo(to, duration: duration, curve: curve);
  }

  @override
  bool get allowImplicitScrolling => physics.allowImplicitScrolling;

  ...

  ScrollHoldController hold(VoidCallback holdCancelCallback);

  Drag drag(DragStartDetails details, VoidCallback dragCancelCallback);

  /// 当前操作的 [ScrollActivity].
  ///
  /// 如果 scrollPosition 不再运行任何 activity, activity 将会变成 [IdleScrollActivity]。
  /// 调用 [beginActivity] 来开始一个 activity
  @protected
  @visibleForTesting
  ScrollActivity get activity => _activity;
  ScrollActivity _activity;

  /// 改变当前 [activity], 销毁旧的 activity
  void beginActivity(ScrollActivity newActivity) {
    if (newActivity == null)
      return;
    bool wasScrolling, oldIgnorePointer;
    if (_activity != null) {
      oldIgnorePointer = _activity.shouldIgnorePointer;
      wasScrolling = _activity.isScrolling;
      if (wasScrolling && !newActivity.isScrolling)
        didEndScroll(); // notifies and then saves the scroll offset
      _activity.dispose();
    } else {
      oldIgnorePointer = false;
      wasScrolling = false;
    }
    _activity = newActivity;
    if (oldIgnorePointer != activity.shouldIgnorePointer)
      context.setIgnorePointer(activity.shouldIgnorePointer);
    isScrollingNotifier.value = activity.isScrolling;
    if (!wasScrolling && _activity.isScrolling)
      didStartScroll();
  }


  // 发送 NOTIFICATION

  /// 被 [beginActivity] 调用，向上报告 activity 已经开始
  void didStartScroll() {
    activity.dispatchScrollStartNotification(copyWith(), context.notificationContext);
  }

  /// 被 [setPixels] 调用，向上报告 [pixels] 发生了改变
  void didUpdateScrollPositionBy(double delta) {
    activity.dispatchScrollUpdateNotification(copyWith(), context.notificationContext, delta);
  }

  /// 被 [beginActivity] 调用，向上报告 activity 已经结束
  ///
  /// 这个方法调用 [saveScrollOffset] 来保存滚动结果
  void didEndScroll() {
    activity.dispatchScrollEndNotification(copyWith(), context.notificationContext);
    if (keepScrollOffset)
      saveScrollOffset();
  }

  /// 被 [setPixels] 调用，向上报告 overscroll 试图修改 pixels
  void didOverscrollBy(double value) {
    assert(activity.isScrolling);
    activity.dispatchOverscrollNotification(copyWith(), context.notificationContext, value);
  }

  /// 报告 [userScrollDirection] 发生了改变
    
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



## ScrollController

```dart
class ScrollController extends ChangeNotifier {
  ScrollController({
    double initialScrollOffset = 0.0,
    this.keepScrollOffset = true,
    this.debugLabel,
  }) : assert(initialScrollOffset != null),
       assert(keepScrollOffset != null),
       _initialScrollOffset = initialScrollOffset;

  /// 最初的偏移量，即列表开始的位置
  double get initialScrollOffset => _initialScrollOffset;
  final double _initialScrollOffset;

  /// 每次滚动结束时，保存当前的滚动偏移量。当可滚动对象重现创建时，恢复滚动偏移
  ///
  /// 是否要储存偏移量
  final bool keepScrollOffset;
    
  final String debugLabel;

  /// 不应该直接修改这个列表。使用[attach] 和 [detach] 来添加或者移除 [ScrollPosition]
  @protected
  Iterable<ScrollPosition> get positions => _positions;
  final List<ScrollPosition> _positions = <ScrollPosition>[];

  /// 是否有 附着的 scrollPostion
  bool get hasClients => _positions.isNotEmpty;

  /// 返回一个附着的[ScrollPosition]，可以获取 SrcollView 的实际 offset
  ///
  /// 只有在单个附着的ScroolPostion时调用才有效
  ScrollPosition get position {
    return _positions.single;
  }

  /// 同上
  double get offset => position.pixels;

  Future<void> animateTo(
    double offset, {
    @required Duration duration,
    @required Curve curve,
  }) {
    final List<Future<void>> animations = List<Future<void>>(_positions.length);
    for (int i = 0; i < _positions.length; i += 1)
      animations[i] = 
        _positions[i].animateTo(
        	offset, 
        	duration: duration, 
        	curve: curve
   		);
    return Future.wait<void>(animations).then<void>((List<void> _) => null);
  }

  void jumpTo(double value) {
    for (ScrollPosition position in List<ScrollPosition>.from(_positions))
      position.jumpTo(value);
  }

  /// 为 controller 注册一个 postion
  /// 注意，ScrollController 继承自 ChangeNotifier
  /// 这里的 notifyListeners 是 ChangeNotifier 的方法  
  void attach(ScrollPosition position) {
    _positions.add(position);
    position.addListener(notifyListeners);
  }

  /// 注销一个 postion
  void detach(ScrollPosition position) {
    position.removeListener(notifyListeners);
    _positions.remove(position);
  }

  @override
  void dispose() {
    for (ScrollPosition position in _positions)
      position.removeListener(notifyListeners);
    super.dispose();
  }

  /// 创建一个可以被 [Scrollable] widget 使用的[ScrollPosition]
  ScrollPosition createScrollPosition(
    ScrollPhysics physics,
    ScrollContext context,
    ScrollPosition oldPosition,
  ) {
    return ScrollPositionWithSingleContext(
      physics: physics,
      context: context,
      initialPixels: initialScrollOffset,
      keepScrollOffset: keepScrollOffset,
      oldPosition: oldPosition,
      debugLabel: debugLabel,
    );
  }

}
```



## ScrollBehavior

```dart
/// 描述 [Scrollable] widgets 的行为
@immutable
class ScrollBehavior {
  /// Creates a description of how [Scrollable] widgets should behave.
  const ScrollBehavior();

  TargetPlatform getPlatform(BuildContext context) => defaultTargetPlatform;

  Widget buildViewportChrome(
      BuildContext context, 
      Widget child, 
      AxisDirection axisDirection) 
  {
    switch (getPlatform(context)) {
      case TargetPlatform.iOS:
        return child;
      case TargetPlatform.android:
      case TargetPlatform.fuchsia:
        return GlowingOverscrollIndicator(
          child: child,
          axisDirection: axisDirection,
          color: _kDefaultGlowColor,
        );
    }
    return null;
  }

  /// ios: 弹性，anfroid/fuchsia: 滚动
  ScrollPhysics getScrollPhysics(BuildContext context) {
    switch (getPlatform(context)) {
      case TargetPlatform.iOS:
        return const BouncingScrollPhysics();
      case TargetPlatform.android:
      case TargetPlatform.fuchsia:
        return const ClampingScrollPhysics();
    }
    return null;
  }

  bool shouldNotify(covariant ScrollBehavior oldDelegate) => false;
}

```



## ScrollConfiguration

```dart
/// 控制子树的 [Scrollable] 的行为 
class ScrollConfiguration extends InheritedWidget {

  const ScrollConfiguration({
    Key key,
    @required this.behavior,
    @required Widget child,
  }) : super(key: key, child: child);

  final ScrollBehavior behavior;

  static ScrollBehavior of(BuildContext context) {
    final ScrollConfiguration configuration = 
        context.inheritFromWidgetOfExactType(ScrollConfiguration);
    return configuration?.behavior ?? const ScrollBehavior();
  }

  @override
  bool updateShouldNotify(ScrollConfiguration oldWidget) {
    return 
        behavior.runtimeType != oldWidget.behavior.runtimeType || 
        (behavior != oldWidget.behavior && behavior.shouldNotify(oldWidget.behavior));
  }

}
```





## Scrollable

```dart
typedef ViewportBuilder = Widget Function(BuildContext context, ViewportOffset position);

class Scrollable extends StatefulWidget {
  const Scrollable({
    Key key,
    this.axisDirection = AxisDirection.down,
    this.controller,
    this.physics,
    @required this.viewportBuilder,
    this.excludeFromSemantics = false,
    this.semanticChildCount,
    this.dragStartBehavior = DragStartBehavior.start,
  }) : assert(axisDirection != null),
       assert(dragStartBehavior != null),
       assert(viewportBuilder != null),
       assert(excludeFromSemantics != null),
       super (key: key);

  final AxisDirection axisDirection;

  final ScrollController controller;

  final ScrollPhysics physics;

  final ViewportBuilder viewportBuilder;

  final DragStartBehavior dragStartBehavior;

  /// 由轴方向返回滚动方向：
  /// 上下 => 垂直，左右 => 水平  
  Axis get axis => axisDirectionToAxis(axisDirection);

  @override
  ScrollableState createState() => ScrollableState();

  static ScrollableState of(BuildContext context) {
    final _ScrollableScope widget = 
        context.inheritFromWidgetOfExactType(_ScrollableScope);
    return widget?.scrollable;
  }

  /// Scrolls the scrollables that enclose the given context so as to make the
  /// given context visible.
  static Future<void> ensureVisible(
    BuildContext context, {
    double alignment = 0.0,
    Duration duration = Duration.zero,
    Curve curve = Curves.ease,
  }) {
    final List<Future<void>> futures = <Future<void>>[];

    ScrollableState scrollable = Scrollable.of(context);
    while (scrollable != null) {
      futures.add(scrollable.position.ensureVisible(
        context.findRenderObject(),
        alignment: alignment,
        duration: duration,
        curve: curve,
      ));
      context = scrollable.context;
      scrollable = Scrollable.of(context);
    }

    if (futures.isEmpty || duration == Duration.zero)
      return Future<void>.value();
    if (futures.length == 1)
      return futures.single;
    return Future.wait<void>(futures).then<void>((List<void> _) => null);
  }
}
```



##  _ScrollableScope

```dart
class _ScrollableScope extends InheritedWidget {
  const _ScrollableScope({
    Key key,
    @required this.scrollable,
    @required this.position,
    @required Widget child,
  }) : assert(scrollable != null),
       assert(child != null),
       super(key: key, child: child);

  final ScrollableState scrollable;
  final ScrollPosition position;

  @override
  bool updateShouldNotify(_ScrollableScope old) {
    return position != old.position;
  }
}
```



## ScrollableState

```dart
class ScrollableState 
    extends State<Scrollable> 
    with TickerProviderStateMixin
    /// 注意，ScrollableState 实现了 ScrollContext 
    implements ScrollContext 
{

  ScrollPosition get position => _position;
  ScrollPosition _position;

  @override
  AxisDirection get axisDirection => widget.axisDirection;

  ScrollBehavior _configuration;
  ScrollPhysics _physics;

  // 必须在会触发重建的地方调用这个方法
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

    _position = 
        controller?.createScrollPosition(_physics, this, oldPosition) ?? 			
        ScrollPositionWithSingleContext(
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


  ...


  // 手势识别

  final GlobalKey<RawGestureDetectorState> _gestureDetectorKey = 
      GlobalKey<RawGestureDetectorState>();
  final GlobalKey _ignorePointerKey = GlobalKey();

  // This field is set during layout, and then reused until the next time it is set.
  Map<Type, GestureRecognizerFactory> _gestureRecognizers = 
      const <Type, GestureRecognizerFactory>{};
    
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
            VerticalDragGestureRecognizer: 
              GestureRecognizerFactoryWithHandlers<VerticalDragGestureRecognizer>(
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
            HorizontalDragGestureRecognizer: 
              GestureRecognizerFactoryWithHandlers<HorizontalDragGestureRecognizer>(
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
      final RenderIgnorePointer renderBox = _ignorePointerKey.currentContext.findRenderObject();
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
    _hold = position.hold(_disposeHold);
  }

  void _handleDragStart(DragStartDetails details) {
    _drag = position.drag(details, _disposeDrag);
  }

  void _handleDragUpdate(DragUpdateDetails details) {
    _drag?.update(details);
  }

  void _handleDragEnd(DragEndDetails details) {
    _drag?.end(details);
  }

  void _handleDragCancel() {
    _hold?.cancel();
    _drag?.cancel();
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
        GestureBinding.instance.
        pointerSignalResolver.register(event, _handlePointerScroll);
      }
    }
  }

  void _handlePointerScroll(PointerEvent event) {
    if (_physics != null && !_physics.shouldAcceptUserOffset(position)) {
      return;
    }
    final double targetScrollOffset = _targetScrollOffsetForPointerScroll(event);
    if (targetScrollOffset != position.pixels) {
      position.jumpTo(targetScrollOffset);
    }
  }

  // DESCRIPTION

  @override
  Widget build(BuildContext context) {
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
        allowImplicitScrolling: widget?.physics?.
        allowImplicitScrolling ?? _physics.allowImplicitScrolling,
          
        semanticChildCount: widget.semanticChildCount,
      );
    }

    return _configuration.buildViewportChrome(context, result, widget.axisDirection);
  }
    
}
```



## ScrollView

```dart
abstract class ScrollView extends StatelessWidget {
  const ScrollView({
    Key key,
    this.scrollDirection = Axis.vertical,
    this.reverse = false,
    this.controller,
    bool primary,
    ScrollPhysics physics,
    this.shrinkWrap = false,
    this.center,
    this.anchor = 0.0,
    this.cacheExtent,
    this.semanticChildCount,
    this.dragStartBehavior = DragStartBehavior.start,
  }) : primary = primary ?? 
       		controller == null && identical(scrollDirection, Axis.vertical),
       physics = physics ?? 
            (  primary == true || 
               (  primary == null && 
                  controller == null && 
                  identical(scrollDirection, Axis.vertical)
               )
            ) ? 
            const AlwaysScrollableScrollPhysics() : 
            null,
       super(key: key);

  
  final Axis scrollDirection;

  /// 是否反转
  final bool reverse;

  final ScrollController controller;

  /// Whether this is the primary scroll view associated with the parent
  /// [PrimaryScrollController].
  ///
  /// When this is true, the scroll view is scrollable even if it does not have
  /// sufficient content to actually scroll. Otherwise, by default the user can
  /// only scroll the view if it has sufficient content. See [physics].
  ///
  /// On iOS, this also identifies the scroll view that will scroll to top in
  /// response to a tap in the status bar.
  ///
  /// Defaults to true when [scrollDirection] is [Axis.vertical] and
  /// [controller] is null.
  final bool primary;

  final ScrollPhysics physics;

  /// 设为 false 的话，那么这个 scrollView 会试图拓展到最大主轴长度
  final bool shrinkWrap;

  final Key center;

  /// 滚动零点的相对位置
  final double anchor;

  /// 缓存的滚动区域大小（超出此区域的widget将会dispose）
  final double cacheExtent;

  final DragStartBehavior dragStartBehavior;
    
  @protected
  AxisDirection getDirection(BuildContext context) {
    return getAxisDirectionFromAxisReverseAndDirectionality(
        context, 
        scrollDirection, 
        reverse
    );
  }

  /// 建造视窗内的widget
  @protected
  List<Widget> buildSlivers(BuildContext context);

  /// 建造视窗
  @protected
  Widget buildViewport(
    BuildContext context,
    ViewportOffset offset,
    AxisDirection axisDirection,
    List<Widget> slivers,
  ) {
    if (shrinkWrap) {
      return ShrinkWrappingViewport(
        axisDirection: axisDirection,
        offset: offset,
        slivers: slivers,
      );
    }
    return Viewport(
      axisDirection: axisDirection,
      offset: offset,
      slivers: slivers,
      cacheExtent: cacheExtent,
      center: center,
      anchor: anchor,
    );
  }

  @override
  Widget build(BuildContext context) {
    final List<Widget> slivers = buildSlivers(context);
    final AxisDirection axisDirection = getDirection(context);

    final ScrollController scrollController = primary
      ? PrimaryScrollController.of(context)
      : controller;
    final Scrollable scrollable = Scrollable(
      dragStartBehavior: dragStartBehavior,
      axisDirection: axisDirection,
      controller: scrollController,
      physics: physics,
      semanticChildCount: semanticChildCount,
      viewportBuilder: (BuildContext context, ViewportOffset offset) {
        return buildViewport(context, offset, axisDirection, slivers);
      },
    );
    return primary && scrollController != null
      ? PrimaryScrollController.none(child: scrollable)
      : scrollable;
  }
}
```


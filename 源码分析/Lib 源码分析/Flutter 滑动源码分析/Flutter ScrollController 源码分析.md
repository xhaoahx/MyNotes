

## ScrollController

```dart
/// 控制 scrollable widget.
///
/// Scroll Controlller 通常在 [State] 储存为成员变量，并在每一次 State.build 方法间被重复使用。一个 scroll 
/// controller 可以被用于控制多个 scrollable widget，但是默写操作，例如获取 [offset]，要求此 controller 只
/// 被用于单个 scrollable widget
///
/// scrollController 对象可以创建 [ScrollPosition] 来管理独立 [Scrollable] widget 的特定状态。为了自定义. [ScrollPosition] ，可以继承 [ScrollController] 并且重载其 [createScrollPosition] 方法。 
///
/// [ScrollController] 实际是 [Listenable]。无论何时， [ScrollController] 在 [ScrollPosition] 通知它们的
/// 监听者的时候，通知它自身的监听者（例如，[ScrollPosition] 中的任一者发生了滚动。通常情况下，我们可以通过 
/// ScrollController.addListener(VoidCallBack) 方法来添加滚动监听）。但是，在 ScrollPosition 发生改变的时
/// 候，它不会通知监听者
///
/// 通常被 [ListView], [GridView], [CustomScrollView] 使用
class ScrollController extends ChangeNotifier {
  ScrollController({
    double initialScrollOffset = 0.0,
    this.keepScrollOffset = true,
    this.debugLabel,
  }) : _initialScrollOffset = initialScrollOffset;

  /// The initial value to use for [offset].
  ///
  /// New [ScrollPosition] objects that are created and attached to this
  /// controller will have their offset initialized to this value
  /// if [keepScrollOffset] is false or a scroll offset hasn't been saved yet.
  ///
  /// Defaults to 0.0.
  double get initialScrollOffset => _initialScrollOffset;
  final double _initialScrollOffset;

  /// 是否要保存 scroll Offset
  final bool keepScrollOffset;

  final String debugLabel;

  /// 当前 attach 的 position
  @protected
  Iterable<ScrollPosition> get positions => _positions;
  final List<ScrollPosition> _positions = <ScrollPosition>[];

  /// 是否监听了任何的 position
  bool get hasClients => _positions.isNotEmpty;

  /// 返回附加的[ScrollPosition]，可以从中获得[ScrollView]的实际滚动偏移量。
  ///
  /// 只在 position 列表只有一个（也就是此 scrollController 只对应了一个 position 的时候）有效
  ScrollPosition get position {
    assert(_positions.isNotEmpty, 'ScrollController not attached to any scroll views.');
    assert(_positions.length == 1, 'ScrollController attached to multiple scroll views.');
    return _positions.single;
  }

  /// 当前（唯一） position 的 offset
  double get offset => position.pixels;

  /// 将 scorll offset 过渡到为给定的值
  /// 注意： 
  /// 这个方法并不要求只注册了一个 position
  Future<void> animateTo(
    double offset, {
    @required Duration duration,
    @required Curve curve,
  }) {
    assert(_positions.isNotEmpty, 'ScrollController not attached to any scroll views.');
    final List<Future<void>> animations = List<Future<void>>(_positions.length);
    for (int i = 0; i < _positions.length; i += 1)
      animations[i] = _positions[i].animateTo(offset, duration: duration, curve: curve);
    return Future.wait<void>(animations).then<void>((List<void> _) => null);
  }

  /// 将 scorll offset 设置为给定的值
  /// 注意： 
  /// 这个方法并不要求只注册了一个 position
  void jumpTo(double value) {
    assert(_positions.isNotEmpty, 'ScrollController not attached to any scroll views.');
    for (ScrollPosition position in List<ScrollPosition>.from(_positions))
      position.jumpTo(value);
  }

  /// 注册一个 position
  void attach(ScrollPosition position) {
    assert(!_positions.contains(position));
    _positions.add(position);
    position.addListener(notifyListeners);
  }

  /// 注销一个已注册的 position
  void detach(ScrollPosition position) {
    assert(_positions.contains(position));
    position.removeListener(notifyListeners);
    _positions.remove(position);
  }

  @override
  void dispose() {
    for (ScrollPosition position in _positions)
      position.removeListener(notifyListeners);
    super.dispose();
  }

  /// 默认返回一个 [ScrollPositionWithSingleContext].
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


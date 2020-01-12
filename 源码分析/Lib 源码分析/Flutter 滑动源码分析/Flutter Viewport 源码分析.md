# Flutter Viewport 源码分析

## RevealedOffset

```dart
/// 它指出了在一个视口中显示一个元素所需的 [Offset]，以及该元素在该视口中的 [rect] 位置。
class RevealedOffset {
  const RevealedOffset({
    @required this.offset,
    @required this.rect,
  });

  /// 展示给定元素需要的偏移量
  final double offset;

  /// 将一个 RenderObject 'target' 作为参数传递给 RenderAbstractView.getOffsetToReveal,会得到一个
  /// RevealedOffset 返回值。
  /// 其中 offset 意味着当 scrollOffset == offset 的时候，target 将会被展示
  ///
  /// rect 代表了 target 在外部坐标系中的位置
  ///
  /// 一个视窗通常具有两种坐标系，这个类起到一个在两者之间的适配器的作用
  ///
  /// 内部坐标系的原点处于在视窗内部滚动的元素的左上角
  /// 当视窗的 scroll offset 发生变化时，内坐标系原点相对于视窗的边缘进行移动
  ///
  /// 外部坐标系的原点处于视窗的左上角。这个坐标系原点与 scroll offset 无关
  ///
  /// 换句话说: [rect] 描述了被展示的元素在外部坐标系中的位置
  final Rect rect;
  ...
}
```



## _SingleChildViewport

```dart
/// 专门提供给 SingleChildScrolldView 的视窗组件
/// 简单封装了 SingleChildRenderObject，创建了 _RenderSingleChildViewport 渲染对象
class _SingleChildViewport extends SingleChildRenderObjectWidget {
  const _SingleChildViewport({
    Key key,
    this.axisDirection = AxisDirection.down,
    this.offset,
    Widget child,
  }) : uper(key: key, child: child);

  final AxisDirection axisDirection;
  final ViewportOffset offset;

  @override
  _RenderSingleChildViewport createRenderObject(BuildContext context) {
    return _RenderSingleChildViewport(
      axisDirection: axisDirection,
      offset: offset,
    );
  }

  @override
  void updateRenderObject(BuildContext context, _RenderSingleChildViewport renderObject) {
    renderObject
      ..axisDirection = axisDirection
      ..offset = offset;
  }
}

```



## Viewport

```dart
/// A widget that is bigger on the inside.
///
/// [Viewport] is the visual workhorse of the scrolling machinery. It displays a
/// subset of its children according to its own dimensions and the given
/// [offset]. As the offset varies, different children are visible through
/// the viewport.
///
/// [Viewport] hosts a bidirectional list of slivers, anchored on a [center]
/// sliver, which is placed at the zero scroll offset. The center widget is
/// displayed in the viewport according to the [anchor] property.
///
/// Slivers that are earlier in the child list than [center] are displayed in
/// reverse order in the reverse [axisDirection] starting from the [center]. For
/// example, if the [axisDirection] is [AxisDirection.down], the first sliver
/// before [center] is placed above the [center]. The slivers that are later in
/// the child list than [center] are placed in order in the [axisDirection]. For
/// example, in the preceding scenario, the first sliver after [center] is
/// placed below the [center].
///
/// [Viewport] cannot contain box children directly. Instead, use a
/// [SliverList], [SliverFixedExtentList], [SliverGrid], or a
/// [SliverToBoxAdapter], for example.
///
/// See also:
///
///  * [ListView], [PageView], [GridView], and [CustomScrollView], which combine
///    [Scrollable] and [Viewport] into widgets that are easier to use.
///  * [SliverToBoxAdapter], which allows a box widget to be placed inside a
///    sliver context (the opposite of this widget).
///  * [ShrinkWrappingViewport], a variant of [Viewport] that shrink-wraps its
///    contents along the main axis.
class Viewport extends MultiChildRenderObjectWidget {
  /// Creates a widget that is bigger on the inside.
  ///
  /// The viewport listens to the [offset], which means you do not need to
  /// rebuild this widget when the [offset] changes.
  ///
  /// The [offset] argument must not be null.
  Viewport({
    Key key,
    this.axisDirection = AxisDirection.down,
    this.crossAxisDirection,
    this.anchor = 0.0,
    @required this.offset,
    this.center,
    this.cacheExtent,
    this.cacheExtentStyle = CacheExtentStyle.pixel,
    List<Widget> slivers = const <Widget>[],
  }) : assert(offset != null),
       assert(slivers != null),
       assert(center == null || slivers.where((Widget child) => child.key == center).length == 1),
       assert(cacheExtentStyle != null),
       assert(cacheExtentStyle != CacheExtentStyle.viewport || cacheExtent != null),
       super(key: key, children: slivers);

  /// The direction in which the [offset]'s [ViewportOffset.pixels] increases.
  ///
  /// For example, if the [axisDirection] is [AxisDirection.down], a scroll
  /// offset of zero is at the top of the viewport and increases towards the
  /// bottom of the viewport.
  final AxisDirection axisDirection;

  /// The direction in which child should be laid out in the cross axis.
  ///
  /// If the [axisDirection] is [AxisDirection.down] or [AxisDirection.up], this
  /// property defaults to [AxisDirection.left] if the ambient [Directionality]
  /// is [TextDirection.rtl] and [AxisDirection.right] if the ambient
  /// [Directionality] is [TextDirection.ltr].
  ///
  /// If the [axisDirection] is [AxisDirection.left] or [AxisDirection.right],
  /// this property defaults to [AxisDirection.down].
  final AxisDirection crossAxisDirection;

  /// The relative position of the zero scroll offset.
  ///
  /// For example, if [anchor] is 0.5 and the [axisDirection] is
  /// [AxisDirection.down] or [AxisDirection.up], then the zero scroll offset is
  /// vertically centered within the viewport. If the [anchor] is 1.0, and the
  /// [axisDirection] is [AxisDirection.right], then the zero scroll offset is
  /// on the left edge of the viewport.
  final double anchor;

  /// Which part of the content inside the viewport should be visible.
  ///
  /// The [ViewportOffset.pixels] value determines the scroll offset that the
  /// viewport uses to select which part of its content to display. As the user
  /// scrolls the viewport, this value changes, which changes the content that
  /// is displayed.
  ///
  /// Typically a [ScrollPosition].
  final ViewportOffset offset;

  /// The first child in the [GrowthDirection.forward] growth direction.
  ///
  /// Children after [center] will be placed in the [axisDirection] relative to
  /// the [center]. Children before [center] will be placed in the opposite of
  /// the [axisDirection] relative to the [center].
  ///
  /// The [center] must be the key of a child of the viewport.
  final Key center;

  /// {@macro flutter.rendering.viewport.cacheExtent}
  final double cacheExtent;

  /// {@macro flutter.rendering.viewport.cacheExtentStyle}
  final CacheExtentStyle cacheExtentStyle;

  /// Given a [BuildContext] and an [AxisDirection], determine the correct cross
  /// axis direction.
  ///
  /// This depends on the [Directionality] if the `axisDirection` is vertical;
  /// otherwise, the default cross axis direction is downwards.
  static AxisDirection getDefaultCrossAxisDirection(BuildContext context, AxisDirection axisDirection) {
    assert(axisDirection != null);
    switch (axisDirection) {
      case AxisDirection.up:
        return textDirectionToAxisDirection(Directionality.of(context));
      case AxisDirection.right:
        return AxisDirection.down;
      case AxisDirection.down:
        return textDirectionToAxisDirection(Directionality.of(context));
      case AxisDirection.left:
        return AxisDirection.down;
    }
    return null;
  }

  @override
  RenderViewport createRenderObject(BuildContext context) {
    return RenderViewport(
      axisDirection: axisDirection,
      crossAxisDirection: crossAxisDirection ?? Viewport.getDefaultCrossAxisDirection(context, axisDirection),
      anchor: anchor,
      offset: offset,
      cacheExtent: cacheExtent,
      cacheExtentStyle: cacheExtentStyle,
    );
  }

  @override
  void updateRenderObject(BuildContext context, RenderViewport renderObject) {
    renderObject
      ..axisDirection = axisDirection
      ..crossAxisDirection = crossAxisDirection ?? Viewport.getDefaultCrossAxisDirection(context, axisDirection)
      ..anchor = anchor
      ..offset = offset
      ..cacheExtent = cacheExtent
      ..cacheExtentStyle = cacheExtentStyle;
  }

  @override
  _ViewportElement createElement() => _ViewportElement(this);

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(EnumProperty<AxisDirection>('axisDirection', axisDirection));
    properties.add(EnumProperty<AxisDirection>('crossAxisDirection', crossAxisDirection, defaultValue: null));
    properties.add(DoubleProperty('anchor', anchor));
    properties.add(DiagnosticsProperty<ViewportOffset>('offset', offset));
    if (center != null) {
      properties.add(DiagnosticsProperty<Key>('center', center));
    } else if (children.isNotEmpty && children.first.key != null) {
      properties.add(DiagnosticsProperty<Key>('center', children.first.key, tooltip: 'implicit'));
    }
    properties.add(DiagnosticsProperty<double>('cacheExtent', cacheExtent));
    properties.add(DiagnosticsProperty<CacheExtentStyle>('cacheExtentStyle', cacheExtentStyle));
  }
}
```


## _ViewportElement

```dart
class _ViewportElement extends MultiChildRenderObjectElement {
  /// Creates an element that uses the given widget as its configuration.
  _ViewportElement(Viewport widget) : super(widget);

  @override
  Viewport get widget => super.widget as Viewport;

  @override
  RenderViewport get renderObject => super.renderObject as RenderViewport;

  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _updateCenter();
  }

  @override
  void update(MultiChildRenderObjectWidget newWidget) {
    super.update(newWidget);
    _updateCenter();
  }

  void _updateCenter() {
    // TODO(ianh): cache the keys to make this faster
    if (widget.center != null) {
      renderObject.center = children.singleWhere(
        (Element element) => element.widget.key == widget.center
      ).renderObject as RenderSliver;
    } else if (children.isNotEmpty) {
      renderObject.center = children.first.renderObject as RenderSliver;
    } else {
      renderObject.center = null;
    }
  }

  @override
  void debugVisitOnstageChildren(ElementVisitor visitor) {
    children.where((Element e) {
      final RenderSliver renderSliver = e.renderObject as RenderSliver;
      return renderSliver.geometry.visible;
    }).forEach(visitor);
  }
}
```



## RenderAbstactViewport

```dart
/// 一个渲染视窗的接口
///
/// 一些渲染对象，例如 [RenderViewport], 呈现了它们的一部分内容，并且能够被 [ViewportOffset] 控制
/// 这个接口能够让框架识别这些渲染对象
abstract class RenderAbstractViewport extends RenderObject {
  // 这个类应该被作为一个接口
  factory RenderAbstractViewport._() => null;

  /// 返回距离给定 RenderObject 最近的 [RenderAbstractViewport]
  static RenderAbstractViewport of(RenderObject object) {
    while (object != null) {
      if (object is RenderAbstractViewport)
        return object;
      object = object.parent;
    }
    return null;
  }

  /// 返回展示 `target` [RenderObject]，所需要的偏移量，和其在外部视窗中的位置
  ///
  /// 可选参数 `rect` 描述了 `target` object 的哪一部分需要在视窗内被展示。
  /// 如果 rect == null，那么整个（即 [RenderObject.paintBounds]）`target` object 将会被展示
  /// 如果 rect 被指定了，那么它必须是 `target` object 坐标系内的 rect
  ///
  /// `alignment`描述了 target 放置的位置
  ///
  /// 这个方法假设视窗的移动是线性的，即，当 scroll offset 变化了 x 的时候，target 也移动了 x 
  /// 
  /// 返回值详见 RevealedOffset
  RevealedOffset getOffsetToReveal(RenderObject target, double alignment, { Rect rect });

  /// 默认缓存距离
  ///
  @protected
  static const double defaultCacheExtent = 250.0;
}
```



## _RenderSingleChildViewport

```dart
/// 专门提供给 SingleChildScrollView 的视窗组件的渲染对象
/// 注意，这个对象继承了 RenderBox，表明这个渲染对象是遵顼笛卡尔坐标系的
/// 混入了 RenderObjectWithChildMixin，提供了默认的单子的渲染对象的管理模式
/// 实现了 RenderAbstractViewport 
class _RenderSingleChildViewport
    extends RenderBox 
    with RenderObjectWithChildMixin<RenderBox> 
    implements RenderAbstractViewport 
{
  _RenderSingleChildViewport({
    AxisDirection axisDirection = AxisDirection.down,
    @required ViewportOffset offset,
    double cacheExtent = RenderAbstractViewport.defaultCacheExtent,
    RenderBox child,
  }) : _axisDirection = axisDirection,
       _offset = offset,
       _cacheExtent = cacheExtent {
    this.child = child;
  }

  /// 轴方向
  AxisDirection get axisDirection => _axisDirection;
  AxisDirection _axisDirection;
  set axisDirection(AxisDirection value) {
    if (value == _axisDirection)
      return;
    _axisDirection = value;
    markNeedsLayout();
  }

  Axis get axis => axisDirectionToAxis(axisDirection);

  /// 详见 scrollPositon
  ViewportOffset get offset => _offset;
  ViewportOffset _offset;
  set offset(ViewportOffset value) {
    if (value == _offset)
      return;
    /// 注意：
    /// 这里向 offset 中注册的监听是 _hasScrolled，而不是 markNeedsLayout
    if (attached)
      _offset.removeListener(_hasScrolled);
    _offset = value;
    if (attached)
      _offset.addListener(_hasScrolled);
    markNeedsLayout();
  }

  /// 缓存区域大小
  double get cacheExtent => _cacheExtent;
  double _cacheExtent;
  set cacheExtent(double value) {
    assert(value != null);
    if (value == _cacheExtent)
      return;
    _cacheExtent = value;
    markNeedsLayout();
  }

  /// 不需要重新布局，只需要重绘并进行语义更新
  void _hasScrolled() {
    markNeedsPaint();
    markNeedsSemanticsUpdate();
  }

  @override
  void setupParentData(RenderObject child) {
    // We don't actually use the offset argument in BoxParentData, so let's
    // avoid allocating it at all.
    if (child.parentData is! ParentData)
      child.parentData = ParentData();
  }

  @override
  void attach(PipelineOwner owner) {
    super.attach(owner);
    _offset.addListener(_hasScrolled);
  }

  @override
  void detach() {
    _offset.removeListener(_hasScrolled);
    super.detach();
  }

  /// 显然，一个视窗是一个重绘边界
  @override
  bool get isRepaintBoundary => true;

  /// 根据轴来决定视窗的大小
  double get _viewportExtent {
    switch (axis) {
      case Axis.horizontal:
        return size.width;
      case Axis.vertical:
        return size.height;
    }
    return null;
  }

  /// 最小滚动距离
  /// 这个视窗只有一个孩子，因此这个视窗是不能够向轴方向的反方向进行滚动的。（多个孩子的视窗可以将 center 
  /// 设置为非首孩子，即，其滚动偏移零点上方可能还会有其他的孩子。若需要呈现那些孩子，反向滚动是必须的，这意味
  /// 其 minScrollExtent 是可以小于 0.0 的。） 
  double get _minScrollExtent {
    return 0.0;
  }

  /// 显然，最大滚动距离即是孩子在轴方向上的大小
  double get _maxScrollExtent {
    if (child == null)
      return 0.0;
    switch (axis) {
      case Axis.horizontal:
        return math.max(0.0, child.size.width - size.width);
      case Axis.vertical:
        return math.max(0.0, child.size.height - size.height);
    }
    return null;
  }

  /// 根据轴方向给定孩子的约束，只约束孩子宽高中的一者。
  BoxConstraints _getInnerConstraints(BoxConstraints constraints) {
    switch (axis) {
      case Axis.horizontal:
        /// 若轴方向是水平的，只约束孩子的宽度
        return constraints.heightConstraints();
      case Axis.vertical:
        /// 若轴方向是垂直的，只约束孩子的宽度
        return constraints.widthConstraints();
    }
    return null;
  }

  /// 计算固有最大最小宽高
  /// 以下实现均是返回孩子的最大最小宽高
  @override
  double computeMinIntrinsicWidth(double height) {
    if (child != null)
      return child.getMinIntrinsicWidth(height);
    return 0.0;
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    if (child != null)
      return child.getMaxIntrinsicWidth(height);
    return 0.0;
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    if (child != null)
      return child.getMinIntrinsicHeight(width);
    return 0.0;
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    if (child != null)
      return child.getMaxIntrinsicHeight(width);
    return 0.0;
  }

  // We don't override computeDistanceToActualBaseline(), because we
  // want the default behavior (returning null). Otherwise, as you
  // scroll, it would shift in its parent if the parent was baseline-aligned,
  // which makes no sense.

  @override
  void performLayout() {
    /// 如果没有孩子，将自身的大小设定为给定的约束的最小值
    if (child == null) {
      size = constraints.smallest;
    /// 否则，使用 _getInnerConstraints 获取到的单一宽高约束孩子进行布局，
    /// 并且根据孩子的大小和自身的约束来确定自身的大小
    } else {
      child.layout(_getInnerConstraints(constraints), parentUsesSize: true);
      size = constraints.constrain(child.size);
    }
	/// 通知 offset 运用新的 _viewportExtent、_minScrollExtent 和 _maxScrollExtent
    offset.applyViewportDimension(_viewportExtent);
    offset.applyContentDimensions(_minScrollExtent, _maxScrollExtent);
  }

  Offset get _paintOffset => _paintOffsetForPosition(offset.pixels);

  /// 根据轴方向和给定的 postion 来确定最终的绘制偏移，其中 position 通常是 offset.pixels
  Offset _paintOffsetForPosition(double position) {
    switch (axisDirection) {
      /// 注意在AxisDirection == up 或者 left 时候的计算，
      /// 考虑以下情况：
      /// 我们设置 AxisDirection == up，并且设置视窗的高度为 100.0，其孩子的高度为 200.0
      /// 假定 scrollOffset == 0.0（这时候 scrollOffset 表示的是孩子的顶部（这个顶部其实是相对的，即孩子在 
      /// down 情况下的底部）离开视窗顶端（这个顶端也是相对的，即 down 情况下的底部）的距离。简言之，这时候孩
      /// 子应该对其视窗的顶部（用相对的情况来说，即渲染了孩子下方的 100 像素，上方的 100 像素在视窗之外）。
      /// paintOffset = position - child.size.height + size.height
      ///             = 0.0 - 200.0 + 100.0
      ///             = -100.0
      /// 这与我们预测的结果一致，即孩子的正确绘制的起点（这里可以得到多子视窗布局绘制的启发）
      case AxisDirection.up:
        return Offset(0.0, position - child.size.height + size.height);
      case AxisDirection.down:
        return Offset(0.0, -position);
      case AxisDirection.left:
        return Offset(position - child.size.width + size.width, 0.0);
      case AxisDirection.right:
        return Offset(-position, 0.0);
    }
    return null;
  }

  /// 在给定的 paintOffset 情况下，是否应该裁剪视窗
  bool _shouldClipAtPaintOffset(Offset paintOffset) {
    /// 若绘制偏移小于 0.0，也就是绘制是从视窗之外开始的
    /// 或者孩子的最右下角的点不包含在视窗的大小之内（也就是说孩子的大小大于视窗）
    /// 那么需要进行裁剪
    return paintOffset < Offset.zero 
        || !(Offset.zero & size).contains((paintOffset & child.size).bottomRight);
  }

  /// 绘制方法
  /// 由于 _paintOffset =  _paintOffsetForPosition(offset.pixels)
  @override
  void paint(PaintingContext context, Offset offset) {
    if (child != null) {
      final Offset paintOffset = _paintOffset;
	  /// 绘制回调，方便使用 context.pushClipRect
      /// 这个回调直接使用计算好的 paintoffset 来绘制孩子
      void paintContents(PaintingContext context, Offset offset) {
        context.paintChild(child, offset + paintOffset);
      }
	  /// 需要裁剪的情况
      if (_shouldClipAtPaintOffset(paintOffset)) {
        context.pushClipRect(needsCompositing, offset, Offset.zero & size, paintContents);
      } else {
        paintContents(context, offset);
      }
    }
  }

  /// 将绘制变化应用到给定的矩阵上  
  @override
  void applyPaintTransform(RenderBox child, Matrix4 transform) {
    final Offset paintOffset = _paintOffset;
    transform.translate(paintOffset.dx, paintOffset.dy);
  }

  /// 描述合适的裁剪范围（默认是自身的大小）
  @override
  Rect describeApproximatePaintClip(RenderObject child) {
    if (child != null && _shouldClipAtPaintOffset(_paintOffset))
      return Offset.zero & size;
    return null;
  }

  /// 对孩子进行点击测试
  @override
  bool hitTestChildren(BoxHitTestResult result, { Offset position }) {
    if (child != null) {
      return result.addWithPaintOffset(
        offset: _paintOffset,
        position: position,
        hitTest: (BoxHitTestResult result, Offset transformed) {
          assert(transformed == position + -_paintOffset);
          return child.hitTest(result, position: transformed);
        },
      );
    }
    return false;
  }

  /// 重载 RendeerAbstactViewport 中的方法，返回展示给定的 RenderObject 所需的 offset
  @override
  RevealedOffset getOffsetToReveal(RenderObject target, double alignment, { Rect rect }) {
    rect ??= target.paintBounds;
    /// 如果给定的目标不是一个渲染盒子
    if (target is! RenderBox)
      return RevealedOffset(offset: offset.pixels, rect: rect);

    final RenderBox targetBox = target as RenderBox;
    final Matrix4 transform = targetBox.getTransformTo(child);
    final Rect bounds = MatrixUtils.transformRect(transform, rect);
    final Size contentSize = child.size;

    double leadingScrollOffset;
    double targetMainAxisExtent;
    double mainAxisExtent;

    assert(axisDirection != null);
    switch (axisDirection) {
      case AxisDirection.up:
        mainAxisExtent = size.height;
        leadingScrollOffset = contentSize.height - bounds.bottom;
        targetMainAxisExtent = bounds.height;
        break;
      case AxisDirection.right:
        mainAxisExtent = size.width;
        leadingScrollOffset = bounds.left;
        targetMainAxisExtent = bounds.width;
        break;
      case AxisDirection.down:
        mainAxisExtent = size.height;
        leadingScrollOffset = bounds.top;
        targetMainAxisExtent = bounds.height;
        break;
      case AxisDirection.left:
        mainAxisExtent = size.width;
        leadingScrollOffset = contentSize.width - bounds.right;
        targetMainAxisExtent = bounds.width;
        break;
    }

    final double targetOffset = 
        leadingScrollOffset - (mainAxisExtent - targetMainAxisExtent) * alignment;
    final Rect targetRect = bounds.shift(_paintOffsetForPosition(targetOffset));
    return RevealedOffset(offset: targetOffset, rect: targetRect);
  }

  @override
  void showOnScreen({
    RenderObject descendant,
    Rect rect,
    Duration duration = Duration.zero,
    Curve curve = Curves.ease,
  }) {
    if (!offset.allowImplicitScrolling) {
      return super.showOnScreen(
        descendant: descendant,
        rect: rect,
        duration: duration,
        curve: curve,
      );
    }

    final Rect newRect = RenderViewportBase.showInViewport(
      descendant: descendant,
      viewport: this,
      offset: offset,
      rect: rect,
      duration: duration,
      curve: curve,
    );
    super.showOnScreen(
      rect: newRect,
      duration: duration,
      curve: curve,
    );
  }
   
  /// ... sematics
}
```



## RenderViewportBase

```dart
/// viewport 监听了 [offset], 它决定了 [SliverConstraints.scrollOffset] 
///
/// 子类通常需要重载 [performLayout] 并且调用 [layoutChildSequence]（可能许多次）
/// 注意以下类定义：
/// ParaentDataClass 是泛型，要求提供一个继承（混入）自 ContainerParentDataMixin 的 ParentData
/// 参见[SliverLogicalContainerParentData]
/// 混入了 ContainerRenderObjectMixin，这个混入通常与 ContainerParentDataMixin 搭配使用，两者共同提供了
/// 链表管理兄弟节点的方式
abstract class RenderViewportBase<
    	ParentDataClass 
    	extends ContainerParentDataMixin<RenderSliver>>
    extends RenderBox 
    with ContainerRenderObjectMixin<RenderSliver, ParentDataClass>
    implements RenderAbstractViewport
{
  /// Initializes fields for subclasses.
  RenderViewportBase({
    AxisDirection axisDirection = AxisDirection.down,
    @required AxisDirection crossAxisDirection,
    @required ViewportOffset offset,
    double cacheExtent,
  }) : _axisDirection = axisDirection,
       _crossAxisDirection = crossAxisDirection,
       /// 在构造函数里对 offset 进行修改，触发 offset 的 setter，将 markNeedsbuild 方法加入 offset 监听者
       /// 中
       _offset = offset,
       _cacheExtent = cacheExtent ?? RenderAbstractViewport.defaultCacheExtent;

  @override
  void describeSemanticsConfiguration(SemanticsConfiguration config) {
    super.describeSemanticsConfiguration(config);
    config.addTagForChildren(RenderViewport.useTwoPaneSemantics);
  }

  @override
  void visitChildrenForSemantics(RenderObjectVisitor visitor) {
    childrenInPaintOrder
        .where((RenderSliver sliver) => sliver.geometry.visible || sliver.geometry.cacheExtent > 0.0)
        .forEach(visitor);
  }

  /// [SliverConstraints.scrollOffset] 增加的方向，即滑动开始的方向
  AxisDirection get axisDirection => _axisDirection;
  AxisDirection _axisDirection;
  /// 发生改变的时侯需要重新布局
  set axisDirection(AxisDirection value) {
    if (value == _axisDirection)
      return;
    _axisDirection = value;
    markNeedsLayout();
  }

  /// 交叉轴的方向
  AxisDirection get crossAxisDirection => _crossAxisDirection;
  AxisDirection _crossAxisDirection;
  set crossAxisDirection(AxisDirection value) {
    assert(value != null);
    if (value == _crossAxisDirection)
      return;
    _crossAxisDirection = value;
    markNeedsLayout();
  }

  /// 视窗轴方向
  Axis get axis => axisDirectionToAxis(axisDirection);

  /// 视窗内部的哪一部分内容应该可见
  ///
  /// [ViewportOffset.pixels] 决定了 scroll offset，其影响了视窗展示的部分
  /// 当用户滚动视窗的时候，ViewportOffset.pixels 发生了改变，于是改变了正在展示的内容
  ViewportOffset get offset => _offset;
  ViewportOffset _offset;
  set offset(ViewportOffset value) {
    if (value == _offset)
      return;
    /// 当设定 ViewportOffset 时候，向其中加入了 markNeedsLayout，
    /// 即每当其值变化的时候，renderViewport 都会重新布局
    /// 若已有一个 offset，需要注销监听
    if (attached)
      _offset.removeListener(markNeedsLayout);
    _offset = value;
    /// 监听新的 offset
    if (attached)
      _offset.addListener(markNeedsLayout);
    markNeedsLayout();
  }

  /// 缓存距离。
  double get cacheExtent => _cacheExtent;
  double _cacheExtent;
  set cacheExtent(double value) {
    value = value ?? RenderAbstractViewport.defaultCacheExtent;
    if (value == _cacheExtent)
      return;
    _cacheExtent = value;
    markNeedsLayout();
  }

  /// 在 attach 的时候，会向 viewportOffset 中加入 markNeedsLayout 回调
  /// 这要求 viewportOffset 不为 null
  @override
  void attach(PipelineOwner owner) {
    super.attach(owner);
    _offset.addListener(markNeedsLayout);
  }

  @override
  void detach() {
    _offset.removeListener(markNeedsLayout);
    super.detach();
  }

  /// 计算固有长宽
  @override
  double computeMinIntrinsicWidth(double height) {
    return 0.0;
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    return 0.0;
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    return 0.0;
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    return 0.0;
  }

  /// 显然，每一个视窗是一个重绘边界
  @override
  bool get isRepaintBoundary => true;

  /// 决定了孩子在视窗内的大小和位置
  ///
  /// 布局从给定的 `child`, 并不断调用 `advance` 回调来布局其他的孩子
  /// 直到 `advance` 返回 null 
  ///
  ///  * scrollOffset 是传递给第一个孩子的 [SliverConstraints.scrollOffset]
  ///    scrollOffset 被 [SliverGeometry.scrollExtent] 为接下来的孩子更新.
  ///
  ///  * overlap 是传递给第一个孩子的 [SliverConstraints.overlap]
  ///    overlap 被 [SliverGeometry.paintOrigin] 为接下来的孩子调节
  ///
  ///  * layoutOffset 是放置第一个孩子的布局偏移量.
  ///    layoutOffset 被 [SliverGeometry.layoutExtent] 为接下来的孩子更新
  ///
  ///  * remainingPaintExtent 是传递给第一个孩子的 [SliverConstraints.remainingPaintExtent]
  ///    remainingPaintExtent 被 [SliverGeometry.layoutExtent] 为接下来的孩子更新
  ///
  ///  * mainAxisExtent 是传递给每一个孩子的 [SliverConstraints.viewportMainAxisExtent]
  ///
  ///  * crossAxisExtent` 是传递给每一个孩子的 [SliverConstraints.crossAxisExtent]
  /// 
  ///  * growthDirection` 是传递给每一个孩子的 [SliverConstraints.growthDirection] 
  ///
  /// 返回第一个遇到的非零的 [SliverGeometry.scrollOffsetCorrection]，如果有的话，否则返回 0.0。
  /// 当返回值不为 0.0 的时候，布局会重新进行
  @protected
  double layoutChildSequence({
    @required RenderSliver child,
    @required double scrollOffset,
    @required double overlap,
    @required double layoutOffset,
    @required double remainingPaintExtent,
    @required double mainAxisExtent,
    @required double crossAxisExtent,
    @required GrowthDirection growthDirection,
    @required RenderSliver advance(RenderSliver child),
    @required double remainingCacheExtent,
    @required double cacheOrigin,
  }) {
    /// 给定的一个孩子的布局偏移
    final double initialLayoutOffset = layoutOffset;
    
    /// 如果增长方向是 forward 的话，那么 adjustedUserScrollDirection = userScrollDirection
    /// 否则 那么 adjustedUserScrollDirection 是 userScrollDirection 的反向
    final ScrollDirection adjustedUserScrollDirection =
        applyGrowthDirectionToScrollDirection(offset.userScrollDirection, growthDirection);
    
    double maxPaintOffset = layoutOffset + overlap;
    /// 这个孩子之前的所有孩子（在主轴上）占用的像素大小
    double precedingScrollExtent = 0.0;

    while (child != null) {
      final double sliverScrollOffset = scrollOffset <= 0.0 ? 0.0 : scrollOffset;
      // If the scrollOffset is too small we adjust the paddedOrigin because it
      // doesn't make sense to ask a sliver for content before its scroll
      // offset.
      final double correctedCacheOrigin = math.max(cacheOrigin, -sliverScrollOffset);
      final double cacheExtentCorrection = cacheOrigin - correctedCacheOrigin;

      /// 用提供的信息构成一个滑块约束
	  final SliverConstraints constraints = SliverConstraints(
        axisDirection: axisDirection,
        growthDirection: growthDirection,
        userScrollDirection: adjustedUserScrollDirection,
        scrollOffset: sliverScrollOffset,
        precedingScrollExtent: precedingScrollExtent,
        /// 很奇怪，maxPaintOffset = overlap + layoutOffset
        overlap: maxPaintOffset - layoutOffset,
        remainingPaintExtent: 
          math.max(0.0, remainingPaintExtent - layoutOffset + initialLayoutOffset),
        crossAxisExtent: crossAxisExtent,
        crossAxisDirection: crossAxisDirection,
        viewportMainAxisExtent: mainAxisExtent,
        remainingCacheExtent: math.max(0.0, remainingCacheExtent + cacheExtentCorrection),
        cacheOrigin: correctedCacheOrigin,
      );
      
      /// 把约束传给当前孩子，命令其布局
      child.layout(constraints, parentUsesSize: true);
	  /// 布局之后孩子会提供一个 geometry（类似于 RenderBox，其布局之后会提供一个 Size）
      final SliverGeometry childLayoutGeometry = child.geometry;

      // 如果不需要任何地修正，那么无需重新布局，否则返回一个修正值
      if (childLayoutGeometry.scrollOffsetCorrection != null)
        return childLayoutGeometry.scrollOffsetCorrection;

      // 我们在我们的坐标系里使用孩子的绘制原点作为其布局偏移，并将储存在其 parent data 里
      final double effectiveLayoutOffset = layoutOffset + childLayoutGeometry.paintOrigin;

      // `effectiveLayoutOffset` becomes meaningless once we moved past the trailing edge
      // because `childLayoutGeometry.layoutExtent` is zero. Using the still increasing
      // 'scrollOffset` to roughly position these invisible slivers in the right order.
      if (childLayoutGeometry.visible || scrollOffset > 0) {
        /// 直接修改孩子的 parentData 中的 paintOffset
        updateChildLayoutOffset(child, effectiveLayoutOffset, growthDirection);
      } else {
        updateChildLayoutOffset(child, -scrollOffset + initialLayoutOffset, growthDirection);
      }

      /// 更新最大绘制距离
      maxPaintOffset = 
          math.max(effectiveLayoutOffset + childLayoutGeometry.paintExtent, maxPaintOffset);
      scrollOffset -= childLayoutGeometry.scrollExtent;
      /// 更新孩子占有距离
      precedingScrollExtent += childLayoutGeometry.scrollExtent;
      /// 更新布局偏移
      layoutOffset += childLayoutGeometry.layoutExtent;
      /// 更新缓存
      if (childLayoutGeometry.cacheExtent != 0.0) {
        remainingCacheExtent -= childLayoutGeometry.cacheExtent - cacheExtentCorrection;
        cacheOrigin = math.min(correctedCacheOrigin + childLayoutGeometry.cacheExtent, 0.0);
      }

      updateOutOfBandData(growthDirection, childLayoutGeometry);

      // 进行下一个孩子的布局
      child = advance(child);
    }

    /// 运行到此处的时候，说明当完成了所有孩子的布局，并且没有需要修正的地方，返回 0.0
    return 0.0;
  }

  @override
  Rect describeApproximatePaintClip(RenderSliver child) {
    final Rect viewportClip = Offset.zero & size;
    if (child.constraints.overlap == 0) {
      return viewportClip;
    }

    // Adjust the clip rect for this sliver by the overlap from the previous sliver.
    double left = viewportClip.left;
    double right = viewportClip.right;
    double top = viewportClip.top;
    double bottom = viewportClip.bottom;
    final double startOfOverlap = 
        child.constraints.viewportMainAxisExtent - child.constraints.remainingPaintExtent;
    final double overlapCorrection = startOfOverlap + child.constraints.overlap;
    switch (applyGrowthDirectionToAxisDirection(axisDirection, child.constraints.growthDirection)) {
      case AxisDirection.down:
        top += overlapCorrection;
        break;
      case AxisDirection.up:
        bottom -= overlapCorrection;
        break;
      case AxisDirection.right:
        left += overlapCorrection;
        break;
      case AxisDirection.left:
        right -= overlapCorrection;
        break;
    }
    return Rect.fromLTRB(left, top, right, bottom);
  }

  @override
  Rect describeSemanticsClip(RenderSliver child) {
    assert (axis != null);
    switch (axis) {
      case Axis.vertical:
        return Rect.fromLTRB(
          semanticBounds.left,
          semanticBounds.top - cacheExtent,
          semanticBounds.right,
          semanticBounds.bottom + cacheExtent,
        );
      case Axis.horizontal:
        return Rect.fromLTRB(
          semanticBounds.left - cacheExtent,
          semanticBounds.top,
          semanticBounds.right + cacheExtent,
          semanticBounds.bottom,
        );
    }
    return null;
  }

  /// 绘制方法：
  /// 如果有视觉溢出，那么对当前视窗进行裁剪
  /// 否则，直接绘制所有的孩子
  @override
  void paint(PaintingContext context, Offset offset) {
    if (firstChild == null)
      return;
    if (hasVisualOverflow) {
      context.pushClipRect(needsCompositing, offset, Offset.zero & size, _paintContents);
    } else {
      _paintContents(context, offset);
    }
  }

  /// 如果孩子 geometry 表示这个孩子是可见的话，那么绘制它
  /// 注意孩子的绘制顺序
  void _paintContents(PaintingContext context, Offset offset) {
    for (RenderSliver child in childrenInPaintOrder) {
      if (child.geometry.visible)
        context.paintChild(child, offset + paintOffsetOf(child));
    }
  }
    
  /// 对于孩子的点击测试
  /// 注意孩子的点击测试的顺序
  @override
  bool hitTestChildren(BoxHitTestResult result, { Offset position }) {
    double mainAxisPosition, crossAxisPosition;
    /// 以下部分要求 postion 不为 null
    switch (axis) {
      case Axis.vertical:
        mainAxisPosition = position.dy;
        crossAxisPosition = position.dx;
        break;
      case Axis.horizontal:
        mainAxisPosition = position.dx;
        crossAxisPosition = position.dy;
        break;
    }

    final SliverHitTestResult sliverResult = SliverHitTestResult.wrap(result);
    for (RenderSliver child in childrenInHitTestOrder) {
      if (!child.geometry.visible) {
        continue;
      }
      final Matrix4 transform = Matrix4.identity();
      applyPaintTransform(child, transform);
      final bool isHit = result.addWithPaintTransform(
        transform: transform,
        position: null, // Manually adapting from box to sliver position below.
        hitTest: (BoxHitTestResult result, Offset _) {
          return child.hitTest(
            sliverResult,
            mainAxisPosition: computeChildMainAxisPosition(child, mainAxisPosition),
            crossAxisPosition: crossAxisPosition,
          );
        },
      );
      if (isHit) {
        return true;
      }
    }
    return false;
  }

  @override
  RevealedOffset getOffsetToReveal(RenderObject target, double alignment, { Rect rect }) {
    double leadingScrollOffset = 0.0;
    double targetMainAxisExtent;
    rect ??= target.paintBounds;

    // Starting at `target` and walking towards the root:
    //  - `child` will be the last object before we reach this viewport, and
    //  - `pivot` will be the last RenderBox before we reach this viewport.
    RenderObject child = target;
    RenderBox pivot;
    bool onlySlivers = target is RenderSliver; // ... between viewport and `target` (`target` included).
    while (child.parent != this) {
      
      if (child is RenderBox) {
        pivot = child;
      }
      if (child.parent is RenderSliver) {
        final RenderSliver parent = child.parent;
        leadingScrollOffset += parent.childScrollOffset(child);
      } else {
        onlySlivers = false;
        leadingScrollOffset = 0.0;
      }
      child = child.parent;
    }

    if (pivot != null) {
      final RenderSliver pivotParent = pivot.parent;

      final Matrix4 transform = target.getTransformTo(pivot);
      final Rect bounds = MatrixUtils.transformRect(transform, rect);

      final GrowthDirection growthDirection = pivotParent.constraints.growthDirection;
      switch (applyGrowthDirectionToAxisDirection(axisDirection, growthDirection)) {
        case AxisDirection.up:
          double offset;
          switch (growthDirection) {
            case GrowthDirection.forward:
              offset = bounds.bottom;
              break;
            case GrowthDirection.reverse:
              offset = bounds.top;
              break;
          }
          leadingScrollOffset += pivot.size.height - offset;
          targetMainAxisExtent = bounds.height;
          break;
        case AxisDirection.right:
          double offset;
          switch (growthDirection) {
            case GrowthDirection.forward:
              offset = bounds.left;
              break;
            case GrowthDirection.reverse:
              offset = bounds.right;
              break;
          }
          leadingScrollOffset += offset;
          targetMainAxisExtent = bounds.width;
          break;
        case AxisDirection.down:
          double offset;
          switch (growthDirection) {
            case GrowthDirection.forward:
              offset = bounds.top;
              break;
            case GrowthDirection.reverse:
              offset = bounds.bottom;
              break;
          }
          leadingScrollOffset += offset;
          targetMainAxisExtent = bounds.height;
          break;
        case AxisDirection.left:
          double offset;
          switch (growthDirection) {
            case GrowthDirection.forward:
              offset = bounds.right;
              break;
            case GrowthDirection.reverse:
              offset = bounds.left;
              break;
          }
          leadingScrollOffset += pivot.size.width - offset;
          targetMainAxisExtent = bounds.width;
          break;
      }
    } else if (onlySlivers) {
      final RenderSliver targetSliver = target;
      targetMainAxisExtent = targetSliver.geometry.scrollExtent;
    } else {
      return RevealedOffset(offset: offset.pixels, rect: rect);
    }

    assert(child.parent == this);
    assert(child is RenderSliver);
    final RenderSliver sliver = child;
    final double extentOfPinnedSlivers = maxScrollObstructionExtentBefore(sliver);
    leadingScrollOffset = scrollOffsetOf(sliver, leadingScrollOffset);
    switch (sliver.constraints.growthDirection) {
      case GrowthDirection.forward:
        leadingScrollOffset -= extentOfPinnedSlivers;
        break;
      case GrowthDirection.reverse:
        // Nothing to do.
        break;
    }

    double mainAxisExtent;
    switch (axis) {
      case Axis.horizontal:
        mainAxisExtent = size.width - extentOfPinnedSlivers;
        break;
      case Axis.vertical:
        mainAxisExtent = size.height - extentOfPinnedSlivers;
        break;
    }

    final double targetOffset = 
        leadingScrollOffset - (mainAxisExtent - targetMainAxisExtent) * alignment;
    final double offsetDifference = offset.pixels - targetOffset;

    final Matrix4 transform = target.getTransformTo(this);
    applyPaintTransform(child, transform);
    Rect targetRect = MatrixUtils.transformRect(transform, rect);

    switch (axisDirection) {
      case AxisDirection.down:
        targetRect = targetRect.translate(0.0, offsetDifference);
        break;
      case AxisDirection.right:
        targetRect = targetRect.translate(offsetDifference, 0.0);
        break;
      case AxisDirection.up:
        targetRect = targetRect.translate(0.0, -offsetDifference);
        break;
      case AxisDirection.left:
        targetRect = targetRect.translate(-offsetDifference, 0.0);
        break;
    }

    return RevealedOffset(offset: targetOffset, rect: targetRect);
  }

  /// 给定的 `child` 应该在何处被绘制
  ///
  /// 返回的偏移量是从视窗内部左上角到给定孩子的坐标系的左上角（即坐标原点）的偏移
  ///
  /// 这个方法被 [paintOffsetOf] 在布局时为孩子计算偏移使用
  @protected
  Offset computeAbsolutePaintOffset(
      RenderSliver child, 
      double layoutOffset, 
      GrowthDirection growthDirection
  ) {
    switch (applyGrowthDirectionToAxisDirection(axisDirection, growthDirection)) {
      /// ？？ 当 AxisDirection == up 的时候，似乎是以视窗底部为布局原点
      /// 也就是说，返回的偏移是相对与底部的
      case AxisDirection.up:
        return Offset(0.0, size.height - (layoutOffset + child.geometry.paintExtent));
      case AxisDirection.right:
        return Offset(layoutOffset, 0.0);
      /// 否则，直接返回给定 layoutOffset 作为偏移 
      case AxisDirection.down:
        return Offset(0.0, layoutOffset);
      case AxisDirection.left:
        return Offset(size.width - (layoutOffset + child.geometry.paintExtent), 0.0);
    }
    return null;
  }

  // 以下是子类应该实现的 API

  // setupParentData

  // performLayout (and optionally sizedByParent and performResize)

  /// 是否有视觉溢出（即是否可以绘制在视窗的外部）
  @protected
  bool get hasVisualOverflow;

  /// Called during [layoutChildSequence] for each child.
  ///
  /// 通常被用于更新外部的数据,例如最大滚动距离
  @protected
  void updateOutOfBandData(GrowthDirection growthDirection, SliverGeometry childLayoutGeometry);

  /// Called during [layoutChildSequence] to store the layout offset for the
  /// given child.
  ///
  /// Different subclasses using different representations for their children's
  /// layout offset (e.g., logical or physical coordinates). This function lets
  /// subclasses transform the child's layout offset before storing it in the
  /// child's parent data.
  @protected
  void updateChildLayoutOffset(
      RenderSliver child, 
      double layoutOffset, 
      GrowthDirection growthDirection
  );

  /// The offset at which the given `child` should be painted.
  ///
  /// The returned offset is from the top left corner of the inside of the
  /// viewport to the top left corner of the paint coordinate system of the
  /// `child`.
  ///
  /// See also [computeAbsolutePaintOffset], which computes the paint offset
  /// from an explicit layout offset and growth direction instead of using the
  /// values computed for the child during layout.
  @protected
  Offset paintOffsetOf(RenderSliver child);

  /// Returns the scroll offset within the viewport for the given
  /// `scrollOffsetWithinChild` within the given `child`.
  ///
  /// The returned value is an estimate that assumes the slivers within the
  /// viewport do not change the layout extent in response to changes in their
  /// scroll offset.
  @protected
  double scrollOffsetOf(RenderSliver child, double scrollOffsetWithinChild);

  /// Returns the total scroll obstruction extent of all slivers in the viewport
  /// before [child].
  ///
  /// This is the extent by which the actual area in which content can scroll
  /// is reduced. For example, an app bar that is pinned at the top will reduce
  /// the area in which content can actually scroll by the height of the app bar.
  @protected
  double maxScrollObstructionExtentBefore(RenderSliver child);

  /// Converts the `parentMainAxisPosition` into the child's coordinate system.
  ///
  /// The `parentMainAxisPosition` is a distance from the top edge (for vertical
  /// viewports) or left edge (for horizontal viewports) of the viewport bounds.
  /// This describes a line, perpendicular to the viewport's main axis, heretofor
  /// known as the target line.
  ///
  /// The child's coordinate system's origin in the main axis is at the leading
  /// edge of the given child, as given by the child's
  /// [SliverConstraints.axisDirection] and [SliverConstraints.growthDirection].
  ///
  /// This method returns the distance from the leading edge of the given child to
  /// the target line described above.
  ///
  /// (The `parentMainAxisPosition` is not from the leading edge of the
  /// viewport, it's always the top or left edge.)
  @protected
  double computeChildMainAxisPosition(RenderSliver child, double parentMainAxisPosition);

  /// 第一个孩子的下标
  @protected
  int get indexOfFirstChild;

  @protected
  String labelForChild(int index);

  /// 孩子的绘制顺序
  /// 与 [childrenInHitTestOrder] 的顺序是相反的
  @protected
  Iterable<RenderSliver> get childrenInPaintOrder;

  /// 孩子的点击测试顺序
  /// 与 [childrenInPaintOrder] 的顺序是相反的
  /// 注意：顺序问题
  /// 考虑如下一个列表：
  /// 孩子是层叠的，当用户向上滑动列表的时候，上层列表被移出视窗，从而展示出下层孩子
  /// 此时，孩子的绘制顺序应该是从最后一个孩子（即最底层的孩子）开始，向前（向高层）绘制
  /// 点击测试的顺序应该是从第一个孩子（即最上层的孩子）开始，向后（向底层）绘制
  @protected
  Iterable<RenderSliver> get childrenInHitTestOrder;

  /// 把给定的孩子展示在屏幕上
  @override
  void showOnScreen({
    RenderObject descendant,
    Rect rect,
    Duration duration = Duration.zero,
    Curve curve = Curves.ease,
  }) {
    if (!offset.allowImplicitScrolling) {
      return super.showOnScreen(
        descendant: descendant,
        rect: rect,
        duration: duration,
        curve: curve,
      );
    }

    final Rect newRect = RenderViewportBase.showInViewport(
      descendant: descendant,
      viewport: this,
      offset: offset,
      rect: rect,
      duration: duration,
      curve: curve,
    );
    super.showOnScreen(
      rect: newRect,
      duration: duration,
      curve: curve,
    );
  }

  /// Make (a portion of) the given `descendant` of the given `viewport` fully
  /// visible in the `viewport` by manipulating the provided [ViewportOffset]
  /// `offset`.
  ///
  /// The optional `rect` parameter describes which area of the `descendant`
  /// should be shown in the viewport. If `rect` is null, the entire
  /// `descendant` will be revealed. The `rect` parameter is interpreted
  /// relative to the coordinate system of `descendant`.
  ///
  /// The returned [Rect] describes the new location of `descendant` or `rect`
  /// in the viewport after it has been revealed. See [RevealedOffset.rect]
  /// for a full definition of this [Rect].
  ///
  /// The parameters `viewport` and `offset` are required and cannot be null.
  /// If `descendant` is null, this is a no-op and `rect` is returned.
  ///
  /// If both `descendant` and `rect` are null, null is returned because there is
  /// nothing to be shown in the viewport.
  ///
  /// The `duration` parameter can be set to a non-zero value to animate the
  /// target object into the viewport with an animation defined by `curve`.
  static Rect showInViewport({
    RenderObject descendant,
    Rect rect,
    @required RenderAbstractViewport viewport,
    @required ViewportOffset offset,
    Duration duration = Duration.zero,
    Curve curve = Curves.ease,
  }) {
    assert(viewport != null);
    assert(offset != null);
    if (descendant == null) {
      return rect;
    }
    final RevealedOffset leadingEdgeOffset = 
        viewport.getOffsetToReveal(descendant, 0.0, rect: rect);
    final RevealedOffset trailingEdgeOffset = 
        viewport.getOffsetToReveal(descendant, 1.0, rect: rect);
    final double currentOffset = offset.pixels;

    //           scrollOffset
    //                       0 +---------+
    //                         |         |
    //                       _ |         |
    //    viewport position |  |         |
    // with `descendant` at |  |         | _
    //        trailing edge |_ | xxxxxxx |  | viewport position
    //                         |         |  | with `descendant` at
    //                         |         | _| leading edge
    //                         |         |
    //                     800 +---------+
    //
    // `trailingEdgeOffset`: Distance from scrollOffset 0 to the start of the
    //                       viewport on the left in image above.
    // `leadingEdgeOffset`: Distance from scrollOffset 0 to the start of the
    //                      viewport on the right in image above.
    //
    // The viewport position on the left is achieved by setting `offset.pixels`
    // to `trailingEdgeOffset`, the one on the right by setting it to
    // `leadingEdgeOffset`.

    RevealedOffset targetOffset;
    if (leadingEdgeOffset.offset < trailingEdgeOffset.offset) {
      // `descendant` is too big to be visible on screen in its entirety. Let's
      // align it with the edge that requires the least amount of scrolling.
      final double leadingEdgeDiff = (offset.pixels - leadingEdgeOffset.offset).abs();
      final double trailingEdgeDiff = (offset.pixels - trailingEdgeOffset.offset).abs();
      targetOffset = leadingEdgeDiff < trailingEdgeDiff ? leadingEdgeOffset : trailingEdgeOffset;
    } else if (currentOffset > leadingEdgeOffset.offset) {
      // `descendant` currently starts above the leading edge and can be shown
      // fully on screen by scrolling down (which means: moving viewport up).
      targetOffset = leadingEdgeOffset;
    } else if (currentOffset < trailingEdgeOffset.offset) {
      // `descendant currently ends below the trailing edge and can be shown
      // fully on screen by scrolling up (which means: moving viewport down)
      targetOffset = trailingEdgeOffset;
    } else {
      // `descendant` is between leading and trailing edge and hence already
      //  fully shown on screen. No action necessary.
      final Matrix4 transform = descendant.getTransformTo(viewport.parent);
      return MatrixUtils.transformRect(transform, rect ?? descendant.paintBounds);
    }

    assert(targetOffset != null);

    offset.moveTo(targetOffset.offset, duration: duration, curve: curve);
    return targetOffset.rect;
  }
}
```

## RenderViewport

```dart
/// [RenderViewport] 是滚动机器的视觉加工厂. 他根据自身维度和给定的 [offset] 来决定需要展示视窗内容的哪一部
/// 分，当 offset 变化的时候，其展示的内容也会发生改变
///
/// [RenderViewport] 持有两个方向的滑块列表,锚定在一个 [center] 滑块的位置， [center] 被放置在滚动偏移零点
///
/// 在 center 之前的滑块列表将以相反的顺序展示，之后的列表将以正序展示（这里的展示是绘制的意思）
///
/// [RenderViewport] 不能直接包含 [RenderBox] 孩子。相反，应该 使用 [RenderSliverList], 
/// 或者 [RenderSliverFixedExtentList] 或者 [RenderSliverGrid] 或者
/// [RenderSliverToBoxAdapter]
class RenderViewport extends RenderViewportBase<SliverPhysicalContainerParentData> {
  RenderViewport({
    AxisDirection axisDirection = AxisDirection.down,
    @required AxisDirection crossAxisDirection,
    @required ViewportOffset offset,
    double anchor = 0.0,
    List<RenderSliver> children,
    RenderSliver center,
    double cacheExtent,
  }) : assert(anchor != null),
       assert(anchor >= 0.0 && anchor <= 1.0),
       _anchor = anchor,
       _center = center,
       super(
           axisDirection: axisDirection, 
           crossAxisDirection: crossAxisDirection, 
           offset: offset, 
           cacheExtent: cacheExtent
       ) {
    /// 这个渲染对象本身是持有一些孩子的（即构建组件时给定的 children，即一些 renderSliver）
    /// 如果给定一些新的孩子，会将其附加在已经存在的孩子列表（通过 RenderObjectContainerParentData 混入类来
    /// 管理的链表）的尾端
    addAll(children);
    if (center == null && firstChild != null)
      _center = firstChild;
  }

  static const SemanticsTag useTwoPaneSemantics = SemanticsTag('RenderViewport.twoPane');
  static const SemanticsTag excludeFromScrolling = 
      SemanticsTag('RenderViewport.excludeFromScrolling');

  /// 为孩子设置 SliverPhysicalContainerParentData 
  @override
  void setupParentData(RenderObject child) {
    if (child.parentData is! SliverPhysicalContainerParentData)
      child.parentData = SliverPhysicalContainerParentData();
  }

  /// 锚点
  double get anchor => _anchor;
  double _anchor;
  set anchor(double value) {
    if (value == _anchor)
      return;
    _anchor = value;
    markNeedsLayout();
  }

  /// 正序孩子列表的第一个孩子
  ///
  /// 当 [offset.pixels] 是 0.0 的时候，center 渲染对象将会处于滚动偏移零点
  RenderSliver get center => _center;
  RenderSliver _center;
  set center(RenderSliver value) {
    if (value == _center)
      return;
    _center = value;
    markNeedsLayout();
  }

  /// 一个视窗的大小总是由父级给定的约束决定的
  @override
  bool get sizedByParent => true;

  @override
  void performResize() {
    size = constraints.biggest;
    /// 总是使用约束的最大值来作为大小
    /// 每次 size 的大小改变之后，回调 offset.applyViewportDimension，应用新的视窗维度
    switch (axis) {
      case Axis.vertical:
        offset.applyViewportDimension(size.height);
        break;
      case Axis.horizontal:
        offset.applyViewportDimension(size.width);
        break;
    }
  }

  /// 最大布局次数，即最大修正布局的次数
  /// 超出此次数会抛出异常
  static const int _maxLayoutCycles = 10;

  // Out-of-band data computed during layout.
  double _minScrollExtent;
  double _maxScrollExtent;
  bool _hasVisualOverflow = false;

  @override
  void performLayout() {
    /// 如果 center == null ，将这个视窗看作一个空视窗，其大小为 0.0
    if (center == null) {
      _minScrollExtent = 0.0;
      _maxScrollExtent = 0.0;
      _hasVisualOverflow = false;
      offset.applyContentDimensions(0.0, 0.0);
      return;
    }
      
	/// 主轴和交叉轴的长度
    double mainAxisExtent;
    double crossAxisExtent;
    switch (axis) {
      case Axis.vertical:
        mainAxisExtent = size.height;
        crossAxisExtent = size.width;
        break;
      case Axis.horizontal:
        mainAxisExtent = size.width;
        crossAxisExtent = size.height;
        break;
    }

    final double centerOffsetAdjustment = center.centerOffsetAdjustment;

    double correction;
    int count = 0;
    do {
      correction = 
          /// 尝试一次布局
          // 其中（当视窗轴是垂直的时候）：
          // mainAxisExtent 是视窗的高度，crossAxisExtent 是视窗的宽度
          // correctedOffset = offset.pixels + centerOffsetAdjustment 是更正后的中心偏移量
          _attemptLayout(mainAxisExtent, crossAxisExtent, offset.pixels + centerOffsetAdjustment);
      /// 如果返回了一个修正值，那么重新布局，并且用给定的修正来更正 offset
      if (correction != 0.0) {
        offset.correctBy(correction);
      /// 否则，应用新的内容维度
      } else {
        if (offset.applyContentDimensions(
              math.min(0.0, _minScrollExtent + mainAxisExtent * anchor),
              math.max(0.0, _maxScrollExtent - mainAxisExtent * (1.0 - anchor)),
           ))
          break;
      }
      count += 1;
    } while (count < _maxLayoutCycles);
  }

  double _attemptLayout(double mainAxisExtent, double crossAxisExtent, double correctedOffset) {
    _minScrollExtent = 0.0;
    _maxScrollExtent = 0.0;
    _hasVisualOverflow = false;

    /// centerOffset 是从视窗顶点到滚动偏移零点(划分正序滑块和逆序滑块的线)的偏移量
    /// 其中 correctedOffset = offset.pixels + centerOffsetAdjustment
    // 注意：
    // 在 RenderSliver 中对 ceterOffsetAdjustment 的描述如下：
    // “例如，如果一个满足 [AxisDirection.down] 和 [RenderViewport.anchor = 0.5] 的视窗，其唯一一个滑块的
    // 高度为 100.0，其 [centerOffsetAdjustment] 是 50.0，当滚动偏移量为 0.0 时，滑块将位于视窗的中心”
    // 在上例中，我们假设 [centerOffsetAdjustment] = 0.0，那么当滚动偏移量为 0.0 时，滑块将位于视窗距离顶端
    // 50（即 mainAxisExtent * anchor，也就是说，center 的布局其实是将其顶端放置在滚动偏移零点的位置）的位
    // 置，这说明，centerOffsetAdjustment 是用来调整 center 的位置的
    // 考虑 correctedOffset 的计算
    // 即 centerOffset = mainAxisExtent * anchor - offset.pixels - centerOffsetAdjustment：
    // 显然 mainAxisExtent * anchor - offset.pixels 的值即是滚动偏移零点的位置，（即分割线是可以移动的）
    // 即 center 的顶端的位置，再减去 centerOffsetAdjustment ，即是 center 的被调整之后位置
    // （在例中，若 滚动偏移量为 50.0 时，滑块将处于视窗顶端） 
    final double centerOffset = mainAxisExtent * anchor - correctedOffset;
    // 逆序剩余绘制长度，从 0.0 到 mainAxisExtent 之间的一个值，即在当前视窗内部，center（线）之前还有多少距
    // 离可以用来绘制
    final double reverseDirectionRemainingPaintExtent = centerOffset.clamp(0.0, mainAxisExtent);
    // 正序剩余绘制长度，从 0.0 到 mainAxisExtent 之间的一个值，即在当前视窗内部，center（线）之后还有多少距
    // 离可以用来绘制
    final double forwardDirectionRemainingPaintExtent = 
        (mainAxisExtent - centerOffset).clamp(0.0, mainAxisExtent);
	// 最大缓存距离，即 mainAxisExtent 加上两端缓存的距离
    final double fullCacheExtent = mainAxisExtent + 2 * cacheExtent;
    // 中心缓存偏移
    final double centerCacheOffset = centerOffset + cacheExtent;
    // 逆序缓存长度
    final double reverseDirectionRemainingCacheExtent = 
        centerCacheOffset.clamp(0.0, fullCacheExtent);
    // 正序缓存长度
    final double forwardDirectionRemainingCacheExtent = 
        (fullCacheExtent - centerCacheOffset).clamp(0.0, fullCacheExtent);

    // 第一个逆序的孩子，即 center 之前的第一个孩子
    final RenderSliver leadingNegativeChild = childBefore(center);
    // 首先对逆序的孩子列表进行布局
    if (leadingNegativeChild != null) {
      // 以下给定的参数都是对于第一个孩子设置的
      // 在连续的布局进程中，给定的值会不断地进行更新
      final double result = layoutChildSequence(
        /// 逆序列表的第一个孩子
        child: leadingNegativeChild,
        scrollOffset: math.max(mainAxisExtent, centerOffset) - mainAxisExtent,
        overlap: 0.0,
        layoutOffset: forwardDirectionRemainingPaintExtent,
        remainingPaintExtent: reverseDirectionRemainingPaintExtent,
        mainAxisExtent: mainAxisExtent,
        crossAxisExtent: crossAxisExtent,
        growthDirection: GrowthDirection.reverse,
        advance: childBefore,
        remainingCacheExtent: reverseDirectionRemainingCacheExtent,
        cacheOrigin: (mainAxisExtent - centerOffset).clamp(-cacheExtent, 0.0),
      );
      if (result != 0.0)
        return -result;
    }

    // 对正向孩子进行布局
    return layoutChildSequence(
      child: center,
      /// 显然，-centerOffset 就是 [center] 的 scrolloffset
      scrollOffset: math.max(0.0, -centerOffset),
      overlap: leadingNegativeChild == null ? math.min(0.0, -centerOffset) : 0.0,
      /// 如果 centerOffset >= mainAxisExtent，这意味着 center 可能在视窗的外部（例如，当 axisDirection = 
      /// down 的时候在视窗底部以下），这时即采用 centerOffset 作为布局起点
      layoutOffset: centerOffset >= mainAxisExtent 
        ? centerOffset
        : reverseDirectionRemainingPaintExtent,
      /// 还未绘制第一个孩子时剩余的绘制距离
      remainingPaintExtent: forwardDirectionRemainingPaintExtent,
      mainAxisExtent: mainAxisExtent,
      crossAxisExtent: crossAxisExtent,
      growthDirection: GrowthDirection.forward,
      advance: childAfter,
      remainingCacheExtent: forwardDirectionRemainingCacheExtent,
      cacheOrigin: centerOffset.clamp(-cacheExtent, 0.0),
    );
  }

  @override
  bool get hasVisualOverflow => _hasVisualOverflow;
 
  /// 根据增长方向来计算最大最小 scrollExtent
  @override
  void updateOutOfBandData(GrowthDirection growthDirection, SliverGeometry childLayoutGeometry) {
    switch (growthDirection) {
      case GrowthDirection.forward:
        _maxScrollExtent += childLayoutGeometry.scrollExtent;
        break;
      case GrowthDirection.reverse:
        _minScrollExtent -= childLayoutGeometry.scrollExtent;
        break;
    }
    /// 如果给定的 childLayoutGeometry 提示有视觉溢出，那么应该使用裁剪
    if (childLayoutGeometry.hasVisualOverflow)
      _hasVisualOverflow = true;
  }

  /// 使用给定的 layoutOffset 更新给定孩子的偏移
  @override
  void updateChildLayoutOffset(
      RenderSliver child, 
      double layoutOffset, 
      GrowthDirection growthDirection
  ) {
    final SliverPhysicalParentData childParentData = child.parentData;
    childParentData.paintOffset = computeAbsolutePaintOffset(child, layoutOffset, growthDirection);
  }

  /// 给定孩子的绘制偏移
  @override
  Offset paintOffsetOf(RenderSliver child) {
    final SliverPhysicalParentData childParentData = child.parentData;
    return childParentData.paintOffset;
  }

  /// 给定孩子的滚动偏移
  @override
  double scrollOffsetOf(RenderSliver child, double scrollOffsetWithinChild) {
    assert(child.parent == this);
    final GrowthDirection growthDirection = child.constraints.growthDirection;
    switch (growthDirection) {
      case GrowthDirection.forward:
        double scrollOffsetToChild = 0.0;
        RenderSliver current = center;
        while (current != child) {
          scrollOffsetToChild += current.geometry.scrollExtent;
          current = childAfter(current);
        }
        return scrollOffsetToChild + scrollOffsetWithinChild;
      case GrowthDirection.reverse:
        double scrollOffsetToChild = 0.0;
        RenderSliver current = childBefore(center);
        while (current != child) {
          scrollOffsetToChild -= current.geometry.scrollExtent;
          current = childBefore(current);
        }
        return scrollOffsetToChild - scrollOffsetWithinChild;
    }
    return null;
  }

  @override
  double maxScrollObstructionExtentBefore(RenderSliver child) {
    assert(child.parent == this);
    final GrowthDirection growthDirection = child.constraints.growthDirection;
    assert(growthDirection != null);
    switch (growthDirection) {
      case GrowthDirection.forward:
        double pinnedExtent = 0.0;
        RenderSliver current = center;
        while (current != child) {
          pinnedExtent += current.geometry.maxScrollObstructionExtent;
          current = childAfter(current);
        }
        return pinnedExtent;
      case GrowthDirection.reverse:
        double pinnedExtent = 0.0;
        RenderSliver current = childBefore(center);
        while (current != child) {
          pinnedExtent += current.geometry.maxScrollObstructionExtent;
          current = childBefore(current);
        }
        return pinnedExtent;
    }
    return null;
  }

  /// 对给定孩子使用给定的绘制矩阵变化（例如旋转，平移）
  @override
  void applyPaintTransform(RenderObject child, Matrix4 transform) {
    assert(child != null);
    final SliverPhysicalParentData childParentData = child.parentData;
    childParentData.applyPaintTransform(transform);
  }

  /// 计算给定的孩子在主轴上的偏移）
  @override
  double computeChildMainAxisPosition(RenderSliver child, double parentMainAxisPosition) {
    final SliverPhysicalParentData childParentData = child.parentData;
    switch (applyGrowthDirectionToAxisDirection(
             child.constraints.axisDirection, 
        	child.constraints.growthDirection
    )) {
      case AxisDirection.down:
        return parentMainAxisPosition - childParentData.paintOffset.dy;
      case AxisDirection.right:
        return parentMainAxisPosition - childParentData.paintOffset.dx;
      case AxisDirection.up:
        return child.geometry.paintExtent 
            - (parentMainAxisPosition - childParentData.paintOffset.dy);
      case AxisDirection.left:
        return child.geometry.paintExtent 
            - (parentMainAxisPosition - childParentData.paintOffset.dx);
    }
    return 0.0;
  }

  /// 给定孩子的下标
  @override
  int get indexOfFirstChild {
    int count = 0;
    RenderSliver child = center;
    while (child != firstChild) {
      count -= 1;
      child = childBefore(child);
    }
    return count;
  }

  @override
  String labelForChild(int index) {
    if (index == 0)
      return 'center child';
    return 'child $index';
  }

  /// 孩子的绘制顺序
  @override
  Iterable<RenderSliver> get childrenInPaintOrder sync* {
    if (firstChild == null)
      return;
    RenderSliver child = firstChild;
    while (child != center) {
      yield child;
      child = childAfter(child);
    }
    child = lastChild;
    while (true) {
      yield child;
      if (child == center)
        return;
      child = childBefore(child);
    }
  }

  /// 孩子的点击测试顺序
  @override
  Iterable<RenderSliver> get childrenInHitTestOrder sync* {
    if (firstChild == null)
      return;
    RenderSliver child = center;
    while (child != null) {
      yield child;
      child = childAfter(child);
    }
    child = childBefore(center);
    while (child != null) {
      yield child;
      child = childBefore(child);
    }
  }
}
```


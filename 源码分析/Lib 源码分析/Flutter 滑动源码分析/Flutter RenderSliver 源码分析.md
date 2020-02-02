# Flutter 滑动源码分析



## SliverConstraints

```dart
/// 轴方向（各值即滑动的起始方向，例如 down，从下向上滑动）
enum AxisDirection {
  /// 0 在底部，正值在上方，即自上向下滑动（这个通常不会被用到）
  up,
  /// 0 在左侧，正值在右方，即自右向左滑动
  right,
  /// 0 在顶部，正值在下方，即自下向上滑动（这通常是 ListView 的默认滚动方向）
  down,
  /// 0 在右侧，正值在左方，即自左向右滑动
  left,
}

/// 增长方向
enum GrowthDirection {
  /// 排列顺序与 [AxisDirection] 相同
  forward,
  /// 排列顺序与 [AxisDirection] 相反
  reverse,
}

/// 轴的滚动方向，相对于[AxisDirection]和[GrowthDirection]给出的滚动轴的正偏移轴。
enum ScrollDirection {
  /// 空闲状态，即没有滚动行为
  idle,

  /// 正沿着正的滚动偏移方向滚动
  ///
  /// 例如，对于使用 [AxisDirection.down]、[GrowthDirection.forward] 的垂直列表来说
  /// 意味着内容正向上移动，暴露了之后的内容
  forward,

  /// 正沿着负的滚动偏移方向滚动
  ///
  /// 例如，对于使用 [AxisDirection.down]、[GrowthDirection.forward] 的垂直列表来说
  /// 意味着内容正向下移动，暴露了先前的内容
  reverse,
}


class SliverConstraints extends Constraints {
  const SliverConstraints({
    @required this.axisDirection,
    @required this.growthDirection,
    @required this.userScrollDirection,
    @required this.scrollOffset,
    @required this.precedingScrollExtent,
    @required this.overlap,
    @required this.remainingPaintExtent,
    @required this.crossAxisExtent,
    @required this.crossAxisDirection,
    @required this.viewportMainAxisExtent,
    @required this.remainingCacheExtent,
    @required this.cacheOrigin,
  });

  /// 部分属性拷贝 
  SliverConstraints copyWith({...}) {...}

  /// [scrollOffset] 和 [remainingPaintExtent] 增加的方向，即滑动方向
  final AxisDirection axisDirection;

  /// 子组件相对于 [AxisDirection] 排列的方向
  /// 当布局 center 前方的孩子时是 reverse
  /// 当布局 center 后方的孩子时时 forward
  final GrowthDirection growthDirection;

  /// 相对于 [axisDirection] and [growthDirection]，用户正在滑动的方向
  final ScrollDirection userScrollDirection;

  /// The scroll offset, in this sliver's coordinate system, that corresponds to
  /// the earliest visible part of this sliver in the [AxisDirection] if
  /// [growthDirection] is [GrowthDirection.forward] or in the opposite
  /// [AxisDirection] direction if [growthDirection] is [GrowthDirection.reverse].
  ///
  /// 在滑块坐标系中，
  /// 如果 [growthDirection] 是 [growthDirection.forward]，scroll offset 对应于滑块在 [AxisDirection] 
  /// 上最早可见的部分
  /// 如果 [growthDirection] 是 [growthDirection.forward]，scroll offset 对应于滑块在 [AxisDirection] 
  /// 反方向上最早可见的部分（即正方向最晚可见的部分）
  /// 
  /// 例如 AxisDirection == AxisDirection.down 并且 
  /// growthDirection == GrowthDirection.forward，这时 scroll offset 是 sliver 的顶端离开（离开指移出视
  /// 窗顶端）视窗顶端部的距离 
  ///
  /// 此值通常用于计算此滑块是否仍然在视窗之内，通过 [SliverGeometry.paintExtent] 和 
  /// [SliverGeometry.layoutExtent] 来考虑这个滑块的顶端离开视窗的顶端有多远
  ///
  /// 当 AxisDirection == AxisDirection.down 且 growthDirection == GrowthDirection.forward ，一个
  /// slivers 还没有滑过 viewport 的顶部的时候, 其 [scrollOffset] 是 `0` 
  /// 带有[scrollOffset] == 0 的 sliver 子集包括所有在视窗底下的所有 sliver。
  ///
  /// [SliverConstraints.remainingPaintExtent] 通常用于完成相同的目标，即计算滚动的滑块是否仍然从视口
  /// 底部有“部分突出”
  ///
  /// 对应于滑块的顶部或者底部取决于 [growthDirection].
  final double scrollOffset;

  /// 在此之前的所有 [Sliver] 所消耗的滚动距离。
  final double precedingScrollExtent;

  /// The number of pixels from where the pixels corresponding to the
  /// [scrollOffset] will be painted up to the first pixel that has not yet been
  /// painted on by an earlier sliver, in the [axisDirection].
  ///
  /// For example, if the previous sliver had a [SliverGeometry.paintExtent] of
  /// 100.0 pixels but a [SliverGeometry.layoutExtent] of only 50.0 pixels,
  /// then the [overlap] of this sliver will be 50.0.
  ///
  /// 这一属性通常被忽略，除非一个滑块希望它自身被固定或者浮动并且它不希望它之前的滑块有同样的效果
  /// 考虑以下情况：
  /// 有一个垂直列表 ABCDEFG，其中 D 元素在滚动一定距离后固定在原地
  /// 并且它希望只有它被固定，它之前的所有hua'kuai
  final double overlap;

  /// 滑块应该考虑提供的内容像素值。
  /// (提供多于此值的像素值通常是低效率的)
  ///
  /// 具体的值应该在 [RenderSliver.geometry] as [SliverGeometry.paintExtent] 中被指定
  ///
  /// This value may be infinite, for example if the viewport is an
  /// unconstrained [RenderShrinkWrappingViewport].
  ///
  /// 这个值可能是 0.0，例如当一个滑块滚动到垂直视窗的低端之下
  final double remainingPaintExtent;

  /// 交叉轴的大小
  /// 对于垂直的列表，这个值是滑块的宽度
  final double crossAxisExtent;

  /// 交叉轴方向
  final AxisDirection crossAxisDirection;

  /// 视窗主轴的最大距离
  ///
  /// 对于垂直列表，这是视窗的高度
  final double viewportMainAxisExtent;

  /// 相对于 [scrollOffset] 的缓存起点
  ///
  /// 滑块在离开视窗顶端或者底部的时候进入到了缓存区域，在这个时候它们还应该被渲染
  ///
  /// [cacheOrigin] 描述了 [remainingCacheExtent] 相对于 [scrollOffset] 的开始位置
  /// 缓存起点为 0.0 意味着滑块不必体统提供任何的在 [scrollOffset] 之前的内容
  /// [cacheOrigin] == -250.0 意味着尽管滑块的第一处可见部分位于给定的 [scrollOffset] 处，但是滑块人需要渲
  /// 染在 250 像素之前的部分  means that even though the first visible part of
  ///
  /// [cacheOrigin] 总是非正值，并且不会超过 -[scrollOffset]
  /// 换句话说,滑块不会在 [scrollOffset] == 0 之前提供内容
  final double cacheOrigin;


  /// 描述从 [cacheOrigin] 开始滑块应该提供多少内容
  ///
  /// 并不是 [remainingCacheExtent] 其中所有的内容都是可见的，因为其中的一部分可能落入缓存区域
  ///
  /// 每一个滑块都应该在 [cacheOrigin] 开始布局，并在 [remainingCacheExtent] 允许的情况下提供尽可能多内容
  ///
  /// [remainingCacheExtent] 总是大于等于 [remainingPaintExtent].
  /// 落入 [remainingCacheExtent] 之内而在 [remainingPaintExtent] 之外的内容当前不可见
  final double remainingCacheExtent;

  /// 获取轴方向
  Axis get axis => axisDirectionToAxis(axisDirection);

  /// 返回相对于 axisDirection 的增长方向
  /// 如果 axisDirection 是 down 或者 right，那么返回 growthDirection
  /// 否则返回 growthDirection 的反向
  GrowthDirection get normalizedGrowthDirection {
    switch (axisDirection) {
      case AxisDirection.down:
      case AxisDirection.right:
        return growthDirection;
      case AxisDirection.up:
      case AxisDirection.left:
        switch (growthDirection) {
          case GrowthDirection.forward:
            return GrowthDirection.reverse;
          case GrowthDirection.reverse:
            return GrowthDirection.forward;
        }
        return null;
    }
    return null;
  }

  @override
  bool get isTight => false;

  @override
  bool get isNormalized {
    return scrollOffset >= 0.0
        && crossAxisExtent >= 0.0
        && axisDirectionToAxis(axisDirection) != axisDirectionToAxis(crossAxisDirection)
        && viewportMainAxisExtent >= 0.0
        && remainingPaintExtent >= 0.0;
  }

  /// 返回一个与此 SliverConsratraints 对应的 [BoxConstraints]
  /// 注意，给定的交叉轴的盒约束的最大长度总是 crossAxisExtent
  BoxConstraints asBoxConstraints({
    double minExtent = 0.0,
    double maxExtent = double.infinity,
    double crossAxisExtent,
  }) {
    crossAxisExtent ??= this.crossAxisExtent;
    /// 根据轴方向来决定最大最小宽高
    switch (axis) {
      case Axis.horizontal:
        return BoxConstraints(
          /// 注意：若是水平滑动，这里给出高度的约束是 Tight
          minHeight: crossAxisExtent,
          maxHeight: crossAxisExtent,
          minWidth: minExtent,
          maxWidth: maxExtent,
        );
      case Axis.vertical:
        return BoxConstraints(
          /// 注意：若是水平滑动，这里给出宽度的约束是 Tight  
          minWidth: crossAxisExtent,
          maxWidth: crossAxisExtent,
          minHeight: minExtent,
          maxHeight: maxExtent,
        );
    }
    return null;
  }

  ...
}
```



## SliverGeometry

```dart
/// 描述一个滑块所占据的控件
/// 一个滑块可以以不同的形式来占据空间，这是为什么这个类包含了许多值的原因
/// 这个类可以类比于滑块在 盒子中对应的 size
class SliverGeometry extends Diagnosticable {
  const SliverGeometry({
    this.scrollExtent = 0.0,
    this.paintExtent = 0.0,
    this.paintOrigin = 0.0,
    double layoutExtent,
    this.maxPaintExtent = 0.0,
    this.maxScrollObstructionExtent = 0.0,
    double hitTestExtent,
    bool visible,
    this.hasVisualOverflow = false,
    this.scrollOffsetCorrection,
    double cacheExtent,
  }) : layoutExtent = layoutExtent ?? paintExtent,
       hitTestExtent = hitTestExtent ?? paintExtent,
       cacheExtent = cacheExtent ?? layoutExtent ?? paintExtent,
       visible = visible ?? paintExtent > 0.0;

  static const SliverGeometry zero = SliverGeometry();

  /// 预测的这个滑块占用的总大小，即滑块的顶部到滑块的底部的距离之和
  ///
  /// 这个值被用于计算所有 scrollable 内部滑块的 [SliverConstraints.scrollOffset]，因此，无论滑块在不在视
  /// 窗之内，这个值都应该被提供
  ///
  /// 通常情况下，[scrollExtent] 对于滑块来说是一个常量。
  ///
  /// 如果 [paintExtent] 小于 [SliverConstraints.remainingPaintExtent]，那么它必须在布局的时候精确地提供
  final double scrollExtent;

  /// 第一个可视的位置，相对于布局位置
  ///
  /// 例如，如果一个滑块希望在其布局的位置之前绘制，那么 [paintOrigin] 的值是负数
  /// 用于此滑块绘制坐标系统与 [paintOrigin] 相对。换句话说，[RenderSliver.paint] 被调用的时候，
  /// （0，0）位置位于 [paintOrigin].
  /// 
  /// 用于 [paintOrigin] 本身的坐标系统是相对于这个滑块的布局位置的开始，而不是相对于它在视窗上的当前位置。
  /// 换句话说，通常情况下，[paintOrigin] 保持 0.0 不变，而不是当滑块滚动经过视口时，从 0.0 变化到
  /// [SliverConstraints.viewportMainAxisExtent]
  /// 
  /// 这个值不影响后续滑块的布局。下一个滑块仍然放置在该条子的布局位置之后的 [layoutExtent] 的位置。
  /// 这个值会影响，当计算下一个滑块的 [SliverConstraints.overlap] 时 [paintExtent] 从何处开始算起，
  final double paintOrigin;

  /// The amount of currently visible visual space that was taken by the sliver
  /// to render the subset of the sliver that covers all or part of the
  /// [SliverConstraints.remainingPaintExtent] in the current viewport.
  ///
  /// 这个值不影响下一个滑块放置的位置. 换句话说，这个值是 100.0，并且 [layoutExtent] 是 0.0
  /// 
  /// words, if this value was 100 and [layoutExtent] was 0, typical slivers
  /// placed after it would end up drawing in the same 100 pixel space while
  /// painting.
  ///
  /// 这个值必须处于 0 到 [SliverConstraints.remainingPaintExtent] 之间
  ///
  /// 这个值当滑块处于视窗之外时为 0，并且当滑块划过视窗的时候从 0 开始变化
  /// This value is typically 0 when outside of the viewport and grows or
  /// shrinks from 0 or to 0 as the sliver is being scrolled into and out of the
  /// viewport unless the sliver wants to achieve a special effect and paint
  /// even when scrolled away.
  ///
  /// This contributes to the calculation for the next sliver's
  /// [SliverConstraints.overlap].
  final double paintExtent;

  /// 这个滑块的首个个可见的部分到下一个滑块首个可见部分的距离（假设下个滑块的  
  /// [SliverConstraints.scrollOffset] == 0）
  ///
  /// 这个值必须处于 [paintExtent]. 默认等于 [paintExtent].
  ///
  /// 当滑块在视窗之外的时候，这个值是 0，并且当滑块划过视窗的时候，这个值不断变化
  final double layoutExtent;

  /// 滑块能够提供的最大绘制距离
  final double maxPaintExtent;

  /// 如果将滑块固定在边缘，则此滑块可以减少内容滚动的最大范围。
  ///
  /// 不会被固定的滑块，这个值永远是 0.0
  ///
  /// A pinned app bar is an example for a sliver that would use this setting:
  /// When the app bar is pinned to the top, the area in which content can
  /// actually scroll is reduced by the height of the app bar.
  final double maxScrollObstructionExtent;

  /// 滑块接受点击的距离
  ///
  /// 这个值必须在 0 到 [paintExtent] 之间，默认值是 [paintExtent]
  final double hitTestExtent;

  /// 这个滑块是否应该被绘制
  ///
  /// 默认情况下，如果 [paintExtent] 大于 0，那么为 true，否则，为 false
  final bool visible;

  /// 是否有视觉溢出
  ///
  /// 这个值默认为 false，意味着视窗无需裁剪它的孩子。如果有任何的滑块发生了视觉溢出，视窗将会对其孩子运用
  /// 一个裁剪
  final bool hasVisualOverflow;

  /// 如果这个值在 [RenderSliver.performLayout] 返回之后处于非 0 状态，那么，父级将会调整这个滑块的 
  /// Scroll Offset，并且整个父级的 Layout 会被重新运行
  ///
  /// When the value is non-zero, the [RenderSliver] does not need to compute
  /// the rest of the values when constructing the [SliverGeometry] or call
  /// [RenderObject.layout] on its children since [RenderSliver.performLayout]
  /// will be called again on this sliver in the same frame after the
  /// [SliverConstraints.scrollOffset] correction has been applied, when the
  /// proper [SliverGeometry] and layout of its children can be computed.
  ///
  /// If the parent is also a [RenderSliver], it must propagate this value
  /// in its own [RenderSliver.geometry] property until a viewport which adjusts
  /// its offset based on this value.
  final double scrollOffsetCorrection;

  /// 这个滑块在 [SliverConstraints.remainingCacheExtent] 总消耗了多少像素
  ///
  /// 这个值必须等于或者大于 [layoutExtent] 因为它总是至少在 [SliverConstraints.remainingCacheExtent] 占
  /// 用 [layoutExtent] 的像素，甚至更多
  final double cacheExtent;

  /// Asserts
  ...
}

```



## RenderSliver

```dart
abstract class RenderSliver extends RenderObject {
  // 约束输入
  @override
  SliverConstraints get constraints => super.constraints;

  /// 描述此滑块所占据的空间
  ///
  /// 这个值在 layout 的时候是落后的，因此，在 [performLayout] 时不要读取这个内容
  /// 除非，在 调用孩子的 layout 的时候将 parentUsesSize 设置为 true
  /// 
  /// 这个值只应该在 [performLayout] or [performResize] 运行时设置
  /// 如果里希望在这些函数之外修改 geometry ，调用 [markNeedsLayout]
  SliverGeometry get geometry => _geometry;
  SliverGeometry _geometry;
  set geometry(SliverGeometry value) {
    _geometry = value;
  }

  @override
  Rect get semanticBounds => paintBounds;

  /// 获取绘制边界
  @override
  Rect get paintBounds {
    switch (constraints.axis) {
      case Axis.horizontal:
        return Rect.fromLTWH(
          0.0, 0.0,
          geometry.paintExtent,
          constraints.crossAxisExtent,
        );
      case Axis.vertical:
        return Rect.fromLTWH(
          0.0, 0.0,
          constraints.crossAxisExtent,
          geometry.paintExtent,
        );
    }
    return null;
  }

  @override
  void performResize() {
    assert(false);
  }

  /// 对于一个 center 滑块, 这个值等于他在绝对滚动偏移零点之前的覆盖距离
  /// 这个值通常是正值
  double get centerOffsetAdjustment => 0.0;

  /// Determines the set of render objects located at the given position.
  ///
  /// Returns true if the given point is contained in this render object or one
  /// of its descendants. Adds any render objects that contain the point to the
  /// given hit test result.
  ///
  /// The caller is responsible for providing the position in the local
  /// coordinate space of the callee. The callee is responsible for checking
  /// whether the given position is within its bounds.
  ///
  /// Hit testing requires layout to be up-to-date but does not require painting
  /// to be up-to-date. That means a render object can rely upon [performLayout]
  /// having been called in [hitTest] but cannot rely upon [paint] having been
  /// called. For example, a render object might be a child of a [RenderOpacity]
  /// object, which calls [hitTest] on its children when its opacity is zero
  /// even through it does not [paint] its children.
  ///
  /// ## Coordinates for RenderSliver objects
  ///
  /// The `mainAxisPosition` is the distance in the [AxisDirection] (after
  /// applying the [GrowthDirection]) from the edge of the sliver's painted
  /// area. This can be an unusual direction, for example in the
  /// [AxisDirection.up] case this is a distance from the _bottom_ of the
  /// sliver's painted area.
  ///
  /// The `crossAxisPosition` is the distance in the other axis. If the cross
  /// axis is horizontal (i.e. the [SliverConstraints.axisDirection] is either
  /// [AxisDirection.down] or [AxisDirection.up]), then the `crossAxisPosition`
  /// is a distance from the left edge of the sliver. If the cross axis is
  /// vertical (i.e. the [SliverConstraints.axisDirection] is either
  /// [AxisDirection.right] or [AxisDirection.left]), then the
  /// `crossAxisPosition` is a distance from the top edge of the sliver.
  ///
  /// ## Implementing hit testing for slivers
  ///
  /// The most straight-forward way to implement hit testing for a new sliver
  /// render object is to override its [hitTestSelf] and [hitTestChildren]
  /// methods.
  bool hitTest(SliverHitTestResult result, { 
      @required double mainAxisPosition, 
      @required double crossAxisPosition 
  }) {
    if (mainAxisPosition >= 0.0 && mainAxisPosition < geometry.hitTestExtent &&
        crossAxisPosition >= 0.0 && crossAxisPosition < constraints.crossAxisExtent) {
      if (hitTestChildren(
          result, 
          mainAxisPosition: mainAxisPosition, 
          crossAxisPosition: 
          crossAxisPosition
      ) || hitTestSelf(
          mainAxisPosition: mainAxisPosition, 
          crossAxisPosition: crossAxisPosition
      )) {
        result.add(SliverHitTestEntry(
          this,
          mainAxisPosition: mainAxisPosition,
          crossAxisPosition: crossAxisPosition,
        ));
        return true;
      }
    }
    return false;
  }

  /// Override this method if this render object can be hit even if its
  /// children were not hit.
  ///
  /// Used by [hitTest]. If you override [hitTest] and do not call this
  /// function, then you don't need to implement this function.
  ///
  /// For a discussion of the semantics of the arguments, see [hitTest].
  @protected
  bool hitTestSelf({ 
      @required double mainAxisPosition, 
      @required double crossAxisPosition
  }) => false;

  /// Override this method to check whether any children are located at the
  /// given position.
  ///
  /// Typically children should be hit-tested in reverse paint order so that
  /// hit tests at locations where children overlap hit the child that is
  /// visually "on top" (i.e., paints later).
  ///
  /// Used by [hitTest]. If you override [hitTest] and do not call this
  /// function, then you don't need to implement this function.
  ///
  /// For a discussion of the semantics of the arguments, see [hitTest].
  @protected
  bool hitTestChildren(SliverHitTestResult result, { 
      @required double mainAxisPosition, 
      @required double crossAxisPosition 
  }) => false;

  /// 计算从 from 到 to 可见区域的比例，假设只有 [SliverConstraints.scrollOffset] （也就是 
  /// [SliverConstraints.remainingPaintExtent] 的高度）是可见的。并且，假设滚动偏移和绘制偏移的关系也是现行
  /// 的 
  ///
  /// 例如，假设 constraints 的滚动偏移是 100.0 并且剩余绘制距离也是 100.0，并且传入此方法的参数表述的区域是 
  /// 50 ~ 150，那么，返回值将会是 50（也就是从滚动偏移 100 到滚动偏移 150）
  ///
  /// 这个方法
  /// For example, if the constraints have a scroll offset of 100 and a
  /// remaining paint extent of 100, and the arguments to this method describe
  /// the region 50..150, then the returned value would be 50 (from scroll
  /// offset 100 to scroll offset 150).
  ///
  /// 如果消耗的滚动偏移量与消耗的绘制范围之间没有 1:1 的线性关系，则此方法无效。
  /// 例如，如果滑块总是绘制相同的数量，但是消耗了与 [SliverConstraints] 成比例的滚动偏移量。，则此函数的结果
  /// 将不一致。
  double calculatePaintOffset(
      SliverConstraints constraints, {
      @required double from, 
      @required double to 
  }) {
    final double a = constraints.scrollOffset;
    final double b = constraints.scrollOffset + constraints.remainingPaintExtent;
    // the clamp on the next line is to avoid floating point rounding errors
    return (to.clamp(a, b) - from.clamp(a, b)).clamp(0.0, constraints.remainingPaintExtent);
  }

  /// Computes the portion of the region from `from` to `to` that is within
  /// the cache extent of the viewport, assuming that only the region from the
  /// [SliverConstraints.cacheOrigin] that is
  /// [SliverConstraints.remainingCacheExtent] high is visible, and that
  /// the relationship between scroll offsets and paint offsets is linear.
  ///
  /// This method is not useful if there is not a 1:1 relationship between
  /// consumed scroll offset and consumed cache extent.
  double calculateCacheOffset(
      SliverConstraints constraints, { 
      @required double from,
      @required double to 
  }) {
    final double a = constraints.scrollOffset + constraints.cacheOrigin;
    final double b = constraints.scrollOffset + constraints.remainingCacheExtent;
    // the clamp on the next line is to avoid floating point rounding errors
    return (to.clamp(a, b) - from.clamp(a, b)).clamp(0.0, constraints.remainingCacheExtent);
  }

  /// Returns the distance from the leading _visible_ edge of the sliver to the
  /// side of the given child closest to that edge.
  ///
  /// For example, if the [constraints] describe this sliver as having an axis
  /// direction of [AxisDirection.down], then this is the distance from the top
  /// of the visible portion of the sliver to the top of the child. On the other
  /// hand, if the [constraints] describe this sliver as having an axis
  /// direction of [AxisDirection.up], then this is the distance from the bottom
  /// of the visible portion of the sliver to the bottom of the child. In both
  /// cases, this is the direction of increasing
  /// [SliverConstraints.scrollOffset] and
  /// [SliverLogicalParentData.layoutOffset].
  ///
  /// For children that are [RenderSliver]s, the leading edge of the _child_
  /// will be the leading _visible_ edge of the child, not the part of the child
  /// that would locally be a scroll offset 0.0. For children that are not
  /// [RenderSliver]s, for example a [RenderBox] child, it's the actual distance
  /// to the edge of the box, since those boxes do not know how to handle being
  /// scrolled.
  ///
  /// This method differs from [childScrollOffset] in that
  /// [childMainAxisPosition] gives the distance from the leading _visible_ edge
  /// of the sliver whereas [childScrollOffset] gives the distance from the
  /// sliver's zero scroll offset.
  ///
  /// Calling this for a child that is not visible is not valid.
  @protected
  double childMainAxisPosition(covariant RenderObject child) {
    return 0.0;
  }

  /// Returns the distance along the cross axis from the zero of the cross axis
  /// in this sliver's [paint] coordinate space to the nearest side of the given
  /// child.
  ///
  /// For example, if the [constraints] describe this sliver as having an axis
  /// direction of [AxisDirection.down], then this is the distance from the left
  /// of the sliver to the left of the child. Similarly, if the [constraints]
  /// describe this sliver as having an axis direction of [AxisDirection.up],
  /// then this is value is the same. If the axis direction is
  /// [AxisDirection.left] or [AxisDirection.right], then it is the distance
  /// from the top of the sliver to the top of the child.
  ///
  /// Calling this for a child that is not visible is not valid.
  @protected
  double childCrossAxisPosition(covariant RenderObject child) => 0.0;

  /// Returns the scroll offset for the leading edge of the given child.
  ///
  /// The `child` must be a child of this sliver.
  ///
  /// This method differs from [childMainAxisPosition] in that
  /// [childMainAxisPosition] gives the distance from the leading _visible_ edge
  /// of the sliver whereas [childScrollOffset] gives the distance from sliver's
  /// zero scroll offset.
  double childScrollOffset(covariant RenderObject child) {
    assert(child.parent == this);
    return 0.0;
  }

  @override
  void applyPaintTransform(RenderObject child, Matrix4 transform) {
    assert(() {
      throw FlutterError('$runtimeType does not implement applyPaintTransform.');
    }());
  }

  /// This returns a [Size] with dimensions relative to the leading edge of the
  /// sliver, specifically the same offset that is given to the [paint] method.
  /// This means that the dimensions may be negative.
  ///
  /// This is only valid after [layout] has completed.
  ///
  /// See also:
  ///
  ///  * [getAbsoluteSize], which returns absolute size.
  @protected
  Size getAbsoluteSizeRelativeToOrigin() {
    assert(geometry != null);
    assert(!debugNeedsLayout);
    switch (applyGrowthDirectionToAxisDirection(constraints.axisDirection, constraints.growthDirection)) {
      case AxisDirection.up:
        return Size(constraints.crossAxisExtent, -geometry.paintExtent);
      case AxisDirection.right:
        return Size(geometry.paintExtent, constraints.crossAxisExtent);
      case AxisDirection.down:
        return Size(constraints.crossAxisExtent, geometry.paintExtent);
      case AxisDirection.left:
        return Size(-geometry.paintExtent, constraints.crossAxisExtent);
    }
    return null;
  }

  /// This returns the absolute [Size] of the sliver.
  ///
  /// The dimensions are always positive and calling this is only valid after
  /// [layout] has completed.
  ///
  /// See also:
  ///
  ///  * [getAbsoluteSizeRelativeToOrigin], which returns the size relative to
  ///    the leading edge of the sliver.
  @protected
  Size getAbsoluteSize() {
    assert(geometry != null);
    assert(!debugNeedsLayout);
    switch (constraints.axisDirection) {
      case AxisDirection.up:
      case AxisDirection.down:
        return Size(constraints.crossAxisExtent, geometry.paintExtent);
      case AxisDirection.right:
      case AxisDirection.left:
        return Size(geometry.paintExtent, constraints.crossAxisExtent);
    }
    return null;
  }


  // This override exists only to change the type of the second argument.
  @override
  void handleEvent(PointerEvent event, SliverHitTestEntry entry) { }

}
```



## RenderSliverHelpers

```dart
/// Mixin for [RenderSliver] subclasses that provides some utility functions.
abstract class RenderSliverHelpers implements RenderSliver {

  bool _getRightWayUp(SliverConstraints constraints) {
    assert(constraints != null);
    assert(constraints.axisDirection != null);
    bool rightWayUp;
    switch (constraints.axisDirection) {
      case AxisDirection.up:
      case AxisDirection.left:
        rightWayUp = false;
        break;
      case AxisDirection.down:
      case AxisDirection.right:
        rightWayUp = true;
        break;
    }
    assert(constraints.growthDirection != null);
    switch (constraints.growthDirection) {
      case GrowthDirection.forward:
        break;
      case GrowthDirection.reverse:
        rightWayUp = !rightWayUp;
        break;
    }
    assert(rightWayUp != null);
    return rightWayUp;
  }

  /// Utility function for [hitTestChildren] for use when the children are
  /// [RenderBox] widgets.
  ///
  /// This function takes care of converting the position from the sliver
  /// coordinate system to the Cartesian coordinate system used by [RenderBox].
  ///
  /// This function relies on [childMainAxisPosition] to determine the position of
  /// child in question.
  ///
  /// Calling this for a child that is not visible is not valid.
  @protected
  bool hitTestBoxChild(BoxHitTestResult result, RenderBox child, { @required double mainAxisPosition, @required double crossAxisPosition }) {
    final bool rightWayUp = _getRightWayUp(constraints);
    double delta = childMainAxisPosition(child);
    final double crossAxisDelta = childCrossAxisPosition(child);
    double absolutePosition = mainAxisPosition - delta;
    final double absoluteCrossAxisPosition = crossAxisPosition - crossAxisDelta;
    Offset paintOffset, transformedPosition;
    assert(constraints.axis != null);
    switch (constraints.axis) {
      case Axis.horizontal:
        if (!rightWayUp) {
          absolutePosition = child.size.width - absolutePosition;
          delta = geometry.paintExtent - child.size.width - delta;
        }
        paintOffset = Offset(delta, crossAxisDelta);
        transformedPosition = Offset(absolutePosition, absoluteCrossAxisPosition);
        break;
      case Axis.vertical:
        if (!rightWayUp) {
          absolutePosition = child.size.height - absolutePosition;
          delta = geometry.paintExtent - child.size.height - delta;
        }
        paintOffset = Offset(crossAxisDelta, delta);
        transformedPosition = Offset(absoluteCrossAxisPosition, absolutePosition);
        break;
    }
    assert(paintOffset != null);
    assert(transformedPosition != null);
    return result.addWithPaintOffset(
      offset: paintOffset,
      position: null, // Manually adapting from sliver to box position above.
      hitTest: (BoxHitTestResult result, Offset _) {
        return child.hitTest(result, position: transformedPosition);
      },
    );
  }

  /// Utility function for [applyPaintTransform] for use when the children are
  /// [RenderBox] widgets.
  ///
  /// This function turns the value returned by [childMainAxisPosition] and
  /// [childCrossAxisPosition]for the child in question into a translation that
  /// it then applies to the given matrix.
  ///
  /// Calling this for a child that is not visible is not valid.
  @protected
  void applyPaintTransformForBoxChild(RenderBox child, Matrix4 transform) {
    final bool rightWayUp = _getRightWayUp(constraints);
    double delta = childMainAxisPosition(child);
    final double crossAxisDelta = childCrossAxisPosition(child);
    assert(constraints.axis != null);
    switch (constraints.axis) {
      case Axis.horizontal:
        if (!rightWayUp)
          delta = geometry.paintExtent - child.size.width - delta;
        transform.translate(delta, crossAxisDelta);
        break;
      case Axis.vertical:
        if (!rightWayUp)
          delta = geometry.paintExtent - child.size.height - delta;
        transform.translate(crossAxisDelta, delta);
        break;
    }
  }
}
```


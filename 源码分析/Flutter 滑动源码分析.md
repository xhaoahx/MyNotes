# Flutter 滑动源码分析



## SliverConstraints

```dart
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
  SliverConstraints copyWith({
    ...
  }) {
    ...
  }

  /// [scrollOffset] 和 [remainingPaintExtent] 增加的方向
  final AxisDirection axisDirection;

  /// 子组件相对于 [AxisDirection] 排列的方向
  final GrowthDirection growthDirection;

  /// 相对于 [axisDirection] and [growthDirection]，用户试图滑动的方向
  final ScrollDirection userScrollDirection;

  /// The scroll offset, in this sliver's coordinate system, that corresponds to
  /// the earliest visible part of this sliver in the [AxisDirection] if
  /// [growthDirection] is [GrowthDirection.forward] or in the opposite
  /// [AxisDirection] direction if [growthDirection] is [GrowthDirection.reverse].
  ///
  /// 例如 AxisDirection == AxisDirection.down 并且 
  /// growthDirection == GrowthDirection.forward，这时 scroll offset 是 sliver 的顶端离开视
  /// 窗顶端部的距离 
  ///
  /// 这个值通常被用于计算 sliver 是否突出 viewport 通过 [SliverGeometry.paintExtent] 
  /// 和 [SliverGeometry.layoutExtent] 或者某个 sliver 顶端离开视窗顶端部的距离 
  ///
  /// 当 slivers 还没有滑过 viewport 的顶部的时候, [scrollOffset] 是 `0` 
  /// (AxisDirection == AxisDirection.down 且 growthDirection == GrowthDirection.forward)
  /// 带有[scrollOffset] == 0 的 sliver 子集包括所有在 viewport 底下的 sliver。
  ///
  /// [SliverConstraints.remainingPaintExtent] is typically used to accomplish
  /// the same goal of computing whether scrolled out slivers should still
  /// partially 'protrude in' from the bottom of the viewport.
  ///
  /// Whether this corresponds to the beginning or the end of the sliver's
  /// contents depends on the [growthDirection].
  final double scrollOffset;

  /// The scroll distance that has been consumed by all [Sliver]s that came
  /// before this [Sliver].
  ///
  /// # Edge Cases
  ///
  /// [Sliver]s often lazily create their internal content as layout occurs,
  /// e.g., [SliverList]. In this case, when [Sliver]s exceed the viewport,
  /// their children are built lazily, and the [Sliver] does not have enough
  /// information to estimate its total extent, [precedingScrollExtent] will be
  /// [double.infinity] for all [Sliver]s that appear after the lazily
  /// constructed child. This is because a total [scrollExtent] cannot be
  /// calculated unless all inner children have been created and sized, or the
  /// number of children and estimated extents are provided. The infinite
  /// [scrollExtent] will become finite as soon as enough information is
  /// available to estimate the overall extent of all children within the given
  /// [Sliver].
  ///
  /// [Sliver]s may legitimately be infinite, meaning that they can scroll
  /// content forever without reaching the end. For any [Sliver]s that appear
  /// after the infinite [Sliver], the [precedingScrollExtent] will be
  /// [double.infinity].
  final double precedingScrollExtent;

  /// The number of pixels from where the pixels corresponding to the
  /// [scrollOffset] will be painted up to the first pixel that has not yet been
  /// painted on by an earlier sliver, in the [axisDirection].
  ///
  /// For example, if the previous sliver had a [SliverGeometry.paintExtent] of
  /// 100.0 pixels but a [SliverGeometry.layoutExtent] of only 50.0 pixels,
  /// then the [overlap] of this sliver will be 50.0.
  ///
  /// This is typically ignored unless the sliver is itself going to be pinned
  /// or floating and wants to avoid doing so under the previous sliver.
  final double overlap;

  /// The number of pixels of content that the sliver should consider providing.
  /// (Providing more pixels than this is inefficient.)
  ///
  /// The actual number of pixels provided should be specified in the
  /// [RenderSliver.geometry] as [SliverGeometry.paintExtent].
  ///
  /// This value may be infinite, for example if the viewport is an
  /// unconstrained [RenderShrinkWrappingViewport].
  ///
  /// This value may be 0.0, for example if the sliver is scrolled off the
  /// bottom of a downwards vertical viewport.
  final double remainingPaintExtent;

  /// The number of pixels in the cross-axis.
  ///
  /// For a vertical list, this is the width of the sliver.
  final double crossAxisExtent;

  /// The direction in which children should be placed in the cross axis.
  ///
  /// Typically used in vertical lists to describe whether the ambient
  /// [TextDirection] is [TextDirection.rtl] or [TextDirection.ltr].
  final AxisDirection crossAxisDirection;

  /// The number of pixels the viewport can display in the main axis.
  ///
  /// For a vertical list, this is the height of the viewport.
  final double viewportMainAxisExtent;

  /// Where the cache area starts relative to the [scrollOffset].
  ///
  /// Slivers that fall into the cache area located before the leading edge and
  /// after the trailing edge of the viewport should still render content
  /// because they are about to become visible when the user scrolls.
  ///
  /// The [cacheOrigin] describes where the [remainingCacheExtent] starts relative
  /// to the [scrollOffset]. A cache origin of 0 means that the sliver does not
  /// have to provide any content before the current [scrollOffset]. A
  /// [cacheOrigin] of -250.0 means that even though the first visible part of
  /// the sliver will be at the provided [scrollOffset], the sliver should
  /// render content starting 250.0 before the [scrollOffset] to fill the
  /// cache area of the viewport.
  ///
  /// The [cacheOrigin] is always negative or zero and will never exceed
  /// -[scrollOffset]. In other words, a sliver is never asked to provide
  /// content before its zero [scrollOffset].
  ///
  /// See also:
  ///
  ///  * [RenderViewport.cacheExtent] for a description of a viewport's cache area.
  final double cacheOrigin;


  /// Describes how much content the sliver should provide starting from the
  /// [cacheOrigin].
  ///
  /// Not all content in the [remainingCacheExtent] will be visible as some
  /// of it might fall into the cache area of the viewport.
  ///
  /// Each sliver should start laying out content at the [cacheOrigin] and
  /// try to provide as much content as the [remainingCacheExtent] allows.
  ///
  /// The [remainingCacheExtent] is always larger or equal to the
  /// [remainingPaintExtent]. Content, that falls in the [remainingCacheExtent],
  /// but is outside of the [remainingPaintExtent] is currently not visible
  /// in the viewport.
  ///
  /// See also:
  ///
  ///  * [RenderViewport.cacheExtent] for a description of a viewport's cache area.
  final double remainingCacheExtent;

  /// The axis along which the [scrollOffset] and [remainingPaintExtent] are measured.
  Axis get axis => axisDirectionToAxis(axisDirection);

  /// Return what the [growthDirection] would be if the [axisDirection] was
  /// either [AxisDirection.down] or [AxisDirection.right].
  ///
  /// This is the same as [growthDirection] unless the [axisDirection] is either
  /// [AxisDirection.up] or [AxisDirection.left], in which case it is the
  /// opposite growth direction.
  ///
  /// This can be useful in combination with [axis] to view the [axisDirection]
  /// and [growthDirection] in different terms.
  GrowthDirection get normalizedGrowthDirection {
    assert(axisDirection != null);
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

  /// 返回一个对应的 [BoxConstraints]
  /// 根据轴方向来确定最大最小宽高约束
  BoxConstraints asBoxConstraints({
    double minExtent = 0.0,
    double maxExtent = double.infinity,
    double crossAxisExtent,
  }) {
    crossAxisExtent ??= this.crossAxisExtent;
    switch (axis) {
      case Axis.horizontal:
        return BoxConstraints(
          minHeight: crossAxisExtent,
          maxHeight: crossAxisExtent,
          minWidth: minExtent,
          maxWidth: maxExtent,
        );
      case Axis.vertical:
        return BoxConstraints(
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
class SliverGeometry extends Diagnosticable {
  /// Creates an object that describes the amount of space occupied by a sliver.
  ///
  /// If the [layoutExtent] argument is null, [layoutExtent] defaults to the
  /// [paintExtent]. If the [hitTestExtent] argument is null, [hitTestExtent]
  /// defaults to the [paintExtent]. If [visible] is null, [visible] defaults to
  /// whether [paintExtent] is greater than zero.
  ///
  /// The other arguments must not be null.
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
  }) : assert(scrollExtent != null),
       assert(paintExtent != null),
       assert(paintOrigin != null),
       assert(maxPaintExtent != null),
       assert(hasVisualOverflow != null),
       assert(scrollOffsetCorrection != 0.0),
       layoutExtent = layoutExtent ?? paintExtent,
       hitTestExtent = hitTestExtent ?? paintExtent,
       cacheExtent = cacheExtent ?? layoutExtent ?? paintExtent,
       visible = visible ?? paintExtent > 0.0;

  /// A sliver that occupies no space at all.
  static const SliverGeometry zero = SliverGeometry();

  /// The (estimated) total scrollable extent that this sliver has content for.
  ///
  /// This is the amount of scrolling the user needs to do to get from the
  /// beginning of this sliver to the end of this sliver.
  ///
  /// The value is used to calculate the [SliverConstraints.scrollOffset] of
  /// all slivers in the scrollable and thus should be provided whether the
  /// sliver is currently in the viewport or not.
  ///
  /// In a typical scrolling scenario, the [scrollExtent] is constant for a
  /// sliver throughout the scrolling while [paintExtent] and [layoutExtent]
  /// will progress from `0` when offscreen to between `0` and [scrollExtent]
  /// as the sliver scrolls partially into and out of the screen and is
  /// equal to [scrollExtent] while the sliver is entirely on screen. However,
  /// these relationships can be customized to achieve more special effects.
  ///
  /// This value must be accurate if the [paintExtent] is less than the
  /// [SliverConstraints.remainingPaintExtent] provided during layout.
  final double scrollExtent;

  /// The visual location of the first visible part of this sliver relative to
  /// its layout position.
  ///
  /// For example, if the sliver wishes to paint visually before its layout
  /// position, the [paintOrigin] is negative. The coordinate system this sliver
  /// uses for painting is relative to this [paintOrigin]. In other words,
  /// when [RenderSliver.paint] is called, the (0, 0) position of the [Offset]
  /// given to it is at this [paintOrigin].
  ///
  /// The coordinate system used for the [paintOrigin] itself is relative
  /// to the start of this sliver's layout position rather than relative to
  /// its current position on the viewport. In other words, in a typical
  /// scrolling scenario, [paintOrigin] remains constant at 0.0 rather than
  /// tracking from 0.0 to [SliverConstraints.viewportMainAxisExtent] as the
  /// sliver scrolls past the viewport.
  ///
  /// This value does not affect the layout of subsequent slivers. The next
  /// sliver is still placed at [layoutExtent] after this sliver's layout
  /// position. This value does affect where the [paintExtent] extent is
  /// measured from when computing the [SliverConstraints.overlap] for the next
  /// sliver.
  ///
  /// Defaults to 0.0, which means slivers start painting at their layout
  /// position by default.
  final double paintOrigin;

  /// The amount of currently visible visual space that was taken by the sliver
  /// to render the subset of the sliver that covers all or part of the
  /// [SliverConstraints.remainingPaintExtent] in the current viewport.
  ///
  /// This value does not affect how the next sliver is positioned. In other
  /// words, if this value was 100 and [layoutExtent] was 0, typical slivers
  /// placed after it would end up drawing in the same 100 pixel space while
  /// painting.
  ///
  /// This must be between zero and [SliverConstraints.remainingPaintExtent].
  ///
  /// This value is typically 0 when outside of the viewport and grows or
  /// shrinks from 0 or to 0 as the sliver is being scrolled into and out of the
  /// viewport unless the sliver wants to achieve a special effect and paint
  /// even when scrolled away.
  ///
  /// This contributes to the calculation for the next sliver's
  /// [SliverConstraints.overlap].
  final double paintExtent;

  /// The distance from the first visible part of this sliver to the first
  /// visible part of the next sliver, assuming the next sliver's
  /// [SliverConstraints.scrollOffset] is zero.
  ///
  /// This must be between zero and [paintExtent]. It defaults to [paintExtent].
  ///
  /// This value is typically 0 when outside of the viewport and grows or
  /// shrinks from 0 or to 0 as the sliver is being scrolled into and out of the
  /// viewport unless then sliver wants to achieve a special effect and push
  /// down the layout start position of subsequent slivers before the sliver is
  /// even scrolled into the viewport.
  final double layoutExtent;

  /// The (estimated) total paint extent that this sliver would be able to
  /// provide if the [SliverConstraints.remainingPaintExtent] was infinite.
  ///
  /// This is used by viewports that implement shrink-wrapping.
  ///
  /// By definition, this cannot be less than [paintExtent].
  final double maxPaintExtent;

  /// The maximum extent by which this sliver can reduce the area in which
  /// content can scroll if the sliver were pinned at the edge.
  ///
  /// Slivers that never get pinned at the edge, should return zero.
  ///
  /// A pinned app bar is an example for a sliver that would use this setting:
  /// When the app bar is pinned to the top, the area in which content can
  /// actually scroll is reduced by the height of the app bar.
  final double maxScrollObstructionExtent;

  /// The distance from where this sliver started painting to the bottom of
  /// where it should accept hits.
  ///
  /// This must be between zero and [paintExtent]. It defaults to [paintExtent].
  final double hitTestExtent;

  /// Whether this sliver should be painted.
  ///
  /// By default, this is true if [paintExtent] is greater than zero, and
  /// false if [paintExtent] is zero.
  final bool visible;

  /// Whether this sliver has visual overflow.
  ///
  /// By default, this is false, which means the viewport does not need to clip
  /// its children. If any slivers have visual overflow, the viewport will apply
  /// a clip to its children.
  final bool hasVisualOverflow;

  /// If this is non-zero after [RenderSliver.performLayout] returns, the scroll
  /// offset will be adjusted by the parent and then the entire layout of the
  /// parent will be rerun.
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

  /// How many pixels the sliver has consumed in the
  /// [SliverConstraints.remainingCacheExtent].
  ///
  /// This value should be equal to or larger than the [layoutExtent] because
  /// the sliver always consumes at least the [layoutExtent] from the
  /// [SliverConstraints.remainingCacheExtent] and possibly more if it falls
  /// into the cache area of the viewport.
  ///
  /// See also:
  ///
  ///  * [RenderViewport.cacheExtent] for a description of a viewport's cache area.
  final double cacheExtent;

  /// Asserts that this geometry is internally consistent.
  ///
  /// Does nothing if asserts are disabled. Always returns true.
  bool debugAssertIsValid({
    InformationCollector informationCollector,
  }) {
    assert(() {
      void verify(bool check, String summary, {List<DiagnosticsNode> details}) {
        if (check)
          return;
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('$runtimeType is not valid: $summary'),
          ...?details,
          if (informationCollector != null)
            ...informationCollector(),
        ]);
      }

      verify(scrollExtent != null, 'The "scrollExtent" is null.');
      verify(scrollExtent >= 0.0, 'The "scrollExtent" is negative.');
      verify(paintExtent != null, 'The "paintExtent" is null.');
      verify(paintExtent >= 0.0, 'The "paintExtent" is negative.');
      verify(paintOrigin != null, 'The "paintOrigin" is null.');
      verify(layoutExtent != null, 'The "layoutExtent" is null.');
      verify(layoutExtent >= 0.0, 'The "layoutExtent" is negative.');
      verify(cacheExtent >= 0.0, 'The "cacheExtent" is negative.');
      if (layoutExtent > paintExtent) {
        verify(false,
          'The "layoutExtent" exceeds the "paintExtent".',
          details: _debugCompareFloats('paintExtent', paintExtent, 'layoutExtent', layoutExtent),
        );
      }
      verify(maxPaintExtent != null, 'The "maxPaintExtent" is null.');
      // If the paintExtent is slightly more than the maxPaintExtent, but the difference is still less
      // than precisionErrorTolerance, we will not throw the assert below.
      if (paintExtent - maxPaintExtent > precisionErrorTolerance) {
        verify(false,
          'The "maxPaintExtent" is less than the "paintExtent".',
          details:
            _debugCompareFloats('maxPaintExtent', maxPaintExtent, 'paintExtent', paintExtent)
              ..add(ErrorDescription('By definition, a sliver can\'t paint more than the maximum that it can paint!')),
        );
      }
      verify(hitTestExtent != null, 'The "hitTestExtent" is null.');
      verify(hitTestExtent >= 0.0, 'The "hitTestExtent" is negative.');
      verify(visible != null, 'The "visible" property is null.');
      verify(hasVisualOverflow != null, 'The "hasVisualOverflow" is null.');
      verify(scrollOffsetCorrection != 0.0, 'The "scrollOffsetCorrection" is zero.');
      return true;
    }());
    return true;
  }

  @override
  String toStringShort() => '$runtimeType';

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(DoubleProperty('scrollExtent', scrollExtent));
    if (paintExtent > 0.0) {
      properties.add(DoubleProperty('paintExtent', paintExtent, unit : visible ? null : ' but not painting'));
    } else if (paintExtent == 0.0) {
      if (visible) {
        properties.add(DoubleProperty('paintExtent', paintExtent, unit: visible ? null : ' but visible'));
      }
      properties.add(FlagProperty('visible', value: visible, ifFalse: 'hidden'));
    } else {
      // Negative paintExtent!
      properties.add(DoubleProperty('paintExtent', paintExtent, tooltip: '!'));
    }
    properties.add(DoubleProperty('paintOrigin', paintOrigin, defaultValue: 0.0));
    properties.add(DoubleProperty('layoutExtent', layoutExtent, defaultValue: paintExtent));
    properties.add(DoubleProperty('maxPaintExtent', maxPaintExtent));
    properties.add(DoubleProperty('hitTestExtent', hitTestExtent, defaultValue: paintExtent));
    properties.add(DiagnosticsProperty<bool>('hasVisualOverflow', hasVisualOverflow, defaultValue: false));
    properties.add(DoubleProperty('scrollOffsetCorrection', scrollOffsetCorrection, defaultValue: null));
    properties.add(DoubleProperty('cacheExtent', cacheExtent, defaultValue: 0.0));
  }
}

```



## RenderSliver

```dart
abstract class RenderSliver extends RenderObject {
  // layout input
  @override
  SliverConstraints get constraints => super.constraints;

  /// The amount of space this sliver occupies.
  ///
  /// This value is stale whenever this object is marked as needing layout.
  /// During [performLayout], do not read the [geometry] of a child unless you
  /// pass true for parentUsesSize when calling the child's [layout] function.
  ///
  /// The geometry of a sliver should be set only during the sliver's
  /// [performLayout] or [performResize] functions. If you wish to change the
  /// geometry of a sliver outside of those functions, call [markNeedsLayout]
  /// instead to schedule a layout of the sliver.
  SliverGeometry get geometry => _geometry;
  SliverGeometry _geometry;
  set geometry(SliverGeometry value) {
    assert(!(debugDoingThisResize && debugDoingThisLayout));
    assert(sizedByParent || !debugDoingThisResize);
    assert(() {
      if ((sizedByParent && debugDoingThisResize) ||
          (!sizedByParent && debugDoingThisLayout))
        return true;
      assert(!debugDoingThisResize);
      DiagnosticsNode contract, violation, hint;
      if (debugDoingThisLayout) {
        assert(sizedByParent);
        violation = ErrorDescription('It appears that the geometry setter was called from performLayout().');
      } else {
        violation = ErrorDescription('The geometry setter was called from outside layout (neither performResize() nor performLayout() were being run for this object).');
        if (owner != null && owner.debugDoingLayout)
          hint = ErrorDescription('Only the object itself can set its geometry. It is a contract violation for other objects to set it.');
      }
      if (sizedByParent)
        contract = ErrorDescription('Because this RenderSliver has sizedByParent set to true, it must set its geometry in performResize().');
      else
        contract = ErrorDescription('Because this RenderSliver has sizedByParent set to false, it must set its geometry in performLayout().');

      final List<DiagnosticsNode> information = <DiagnosticsNode>[
        ErrorSummary('RenderSliver geometry setter called incorrectly.'),
        violation,
        if (hint != null) hint,
        contract,
        describeForError('The RenderSliver in question is'),
      ];
      throw FlutterError.fromParts(information);
    }());
    _geometry = value;
  }

  @override
  Rect get semanticBounds => paintBounds;

  @override
  Rect get paintBounds {
    assert(constraints.axis != null);
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
  void debugResetSize() { }

  @override
  void debugAssertDoesMeetConstraints() {
    assert(geometry.debugAssertIsValid(
      informationCollector: () sync* {
        yield describeForError('The RenderSliver that returned the offending geometry was');
      }
    ));
    assert(() {
      if (geometry.paintExtent > constraints.remainingPaintExtent) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('SliverGeometry has a paintOffset that exceeds the remainingPaintExtent from the constraints.'),
          describeForError('The render object whose geometry violates the constraints is the following'),
          ..._debugCompareFloats(
            'remainingPaintExtent', constraints.remainingPaintExtent,
            'paintExtent', geometry.paintExtent,
          ),
          ErrorDescription(
            'The paintExtent must cause the child sliver to paint within the viewport, and so '
            'cannot exceed the remainingPaintExtent.',
          ),
        ]);
      }
      return true;
    }());
  }

  @override
  void performResize() {
    assert(false);
  }

  /// For a center sliver, the distance before the absolute zero scroll offset
  /// that this sliver can cover.
  ///
  /// For example, if an [AxisDirection.down] viewport with an
  /// [RenderViewport.anchor] of 0.5 has a single sliver with a height of 100.0
  /// and its [centerOffsetAdjustment] returns 50.0, then the sliver will be
  /// centered in the viewport when the scroll offset is 0.0.
  ///
  /// The distance here is in the opposite direction of the
  /// [RenderViewport.axisDirection], so values will typically be positive.
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
  bool hitTest(SliverHitTestResult result, { @required double mainAxisPosition, @required double crossAxisPosition }) {
    if (mainAxisPosition >= 0.0 && mainAxisPosition < geometry.hitTestExtent &&
        crossAxisPosition >= 0.0 && crossAxisPosition < constraints.crossAxisExtent) {
      if (hitTestChildren(result, mainAxisPosition: mainAxisPosition, crossAxisPosition: crossAxisPosition) ||
          hitTestSelf(mainAxisPosition: mainAxisPosition, crossAxisPosition: crossAxisPosition)) {
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
  bool hitTestSelf({ @required double mainAxisPosition, @required double crossAxisPosition }) => false;

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
  bool hitTestChildren(SliverHitTestResult result, { @required double mainAxisPosition, @required double crossAxisPosition }) => false;

  /// Computes the portion of the region from `from` to `to` that is visible,
  /// assuming that only the region from the [SliverConstraints.scrollOffset]
  /// that is [SliverConstraints.remainingPaintExtent] high is visible, and that
  /// the relationship between scroll offsets and paint offsets is linear.
  ///
  /// For example, if the constraints have a scroll offset of 100 and a
  /// remaining paint extent of 100, and the arguments to this method describe
  /// the region 50..150, then the returned value would be 50 (from scroll
  /// offset 100 to scroll offset 150).
  ///
  /// This method is not useful if there is not a 1:1 relationship between
  /// consumed scroll offset and consumed paint extent. For example, if the
  /// sliver always paints the same amount but consumes a scroll offset extent
  /// that is proportional to the [SliverConstraints.scrollOffset], then this
  /// function's results will not be consistent.
  // This could be a static method but isn't, because it would be less convenient
  // to call it from subclasses if it was.
  double calculatePaintOffset(SliverConstraints constraints, { @required double from, @required double to }) {
    assert(from <= to);
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
  double calculateCacheOffset(SliverConstraints constraints, { @required double from, @required double to }) {
    assert(from <= to);
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
    assert(() {
      throw FlutterError('$runtimeType does not implement childPosition.');
    }());
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

  void _debugDrawArrow(Canvas canvas, Paint paint, Offset p0, Offset p1, GrowthDirection direction) {
    assert(() {
      if (p0 == p1)
        return true;
      assert(p0.dx == p1.dx || p0.dy == p1.dy); // must be axis-aligned
      final double d = (p1 - p0).distance * 0.2;
      Offset temp;
      double dx1, dx2, dy1, dy2;
      switch (direction) {
        case GrowthDirection.forward:
          dx1 = dx2 = dy1 = dy2 = d;
          break;
        case GrowthDirection.reverse:
          temp = p0;
          p0 = p1;
          p1 = temp;
          dx1 = dx2 = dy1 = dy2 = -d;
          break;
      }
      if (p0.dx == p1.dx) {
        dx2 = -dx2;
      } else {
        dy2 = -dy2;
      }
      canvas.drawPath(
        Path()
          ..moveTo(p0.dx, p0.dy)
          ..lineTo(p1.dx, p1.dy)
          ..moveTo(p1.dx - dx1, p1.dy - dy1)
          ..lineTo(p1.dx, p1.dy)
          ..lineTo(p1.dx - dx2, p1.dy - dy2),
        paint,
      );
      return true;
    }());
  }

  @override
  void debugPaint(PaintingContext context, Offset offset) {
    assert(() {
      if (debugPaintSizeEnabled) {
        final double strokeWidth = math.min(4.0, geometry.paintExtent / 30.0);
        final Paint paint = Paint()
          ..color = const Color(0xFF33CC33)
          ..strokeWidth = strokeWidth
          ..style = PaintingStyle.stroke
          ..maskFilter = MaskFilter.blur(BlurStyle.solid, strokeWidth);
        final double arrowExtent = geometry.paintExtent;
        final double padding = math.max(2.0, strokeWidth);
        final Canvas canvas = context.canvas;
        canvas.drawCircle(
          offset.translate(padding, padding),
          padding * 0.5,
          paint,
        );
        switch (constraints.axis) {
          case Axis.vertical:
            canvas.drawLine(
              offset,
              offset.translate(constraints.crossAxisExtent, 0.0),
              paint,
            );
            _debugDrawArrow(
              canvas,
              paint,
              offset.translate(constraints.crossAxisExtent * 1.0 / 4.0, padding),
              offset.translate(constraints.crossAxisExtent * 1.0 / 4.0, arrowExtent - padding),
              constraints.normalizedGrowthDirection,
            );
            _debugDrawArrow(
              canvas,
              paint,
              offset.translate(constraints.crossAxisExtent * 3.0 / 4.0, padding),
              offset.translate(constraints.crossAxisExtent * 3.0 / 4.0, arrowExtent - padding),
              constraints.normalizedGrowthDirection,
            );
            break;
          case Axis.horizontal:
            canvas.drawLine(
              offset,
              offset.translate(0.0, constraints.crossAxisExtent),
              paint,
            );
            _debugDrawArrow(
              canvas,
              paint,
              offset.translate(padding, constraints.crossAxisExtent * 1.0 / 4.0),
              offset.translate(arrowExtent - padding, constraints.crossAxisExtent * 1.0 / 4.0),
              constraints.normalizedGrowthDirection,
            );
            _debugDrawArrow(
              canvas,
              paint,
              offset.translate(padding, constraints.crossAxisExtent * 3.0 / 4.0),
              offset.translate(arrowExtent - padding, constraints.crossAxisExtent * 3.0 / 4.0),
              constraints.normalizedGrowthDirection,
            );
            break;
        }
      }
      return true;
    }());
  }

  // This override exists only to change the type of the second argument.
  @override
  void handleEvent(PointerEvent event, SliverHitTestEntry entry) { }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(DiagnosticsProperty<SliverGeometry>('geometry', geometry));
  }
}
```



## RenderAbstactViewport

```dart
abstract class RenderAbstractViewport extends RenderObject {
  // This class is intended to be used as an interface with the implements
  // keyword, and should not be extended directly.
  factory RenderAbstractViewport._() => null;

  /// Returns the [RenderAbstractViewport] that most tightly encloses the given
  /// render object.
  ///
  /// If the object does not have a [RenderAbstractViewport] as an ancestor,
  /// this function returns null.
  static RenderAbstractViewport of(RenderObject object) {
    while (object != null) {
      if (object is RenderAbstractViewport)
        return object;
      object = object.parent;
    }
    return null;
  }

  /// Returns the offset that would be needed to reveal the `target`
  /// [RenderObject].
  ///
  /// The optional `rect` parameter describes which area of that `target` object
  /// should be revealed in the viewport. If `rect` is null, the entire
  /// `target` [RenderObject] (as defined by its [RenderObject.paintBounds])
  /// will be revealed. If `rect` is provided it has to be given in the
  /// coordinate system of the `target` object.
  ///
  /// The `alignment` argument describes where the target should be positioned
  /// after applying the returned offset. If `alignment` is 0.0, the child must
  /// be positioned as close to the leading edge of the viewport as possible. If
  /// `alignment` is 1.0, the child must be positioned as close to the trailing
  /// edge of the viewport as possible. If `alignment` is 0.5, the child must be
  /// positioned as close to the center of the viewport as possible.
  ///
  /// The `target` might not be a direct child of this viewport but it must be a
  /// descendant of the viewport. Other viewports in between this viewport and
  /// the `target` will not be adjusted.
  ///
  /// This method assumes that the content of the viewport moves linearly, i.e.
  /// when the offset of the viewport is changed by x then `target` also moves
  /// by x within the viewport.
  ///
  /// See also:
  ///
  ///  * [RevealedOffset], which describes the return value of this method.
  RevealedOffset getOffsetToReveal(RenderObject target, double alignment, { Rect rect });

  /// The default value for the cache extent of the viewport.
  ///
  /// See also:
  ///
  ///  * [RenderViewportBase.cacheExtent] for a definition of the cache extent.
  @protected
  static const double defaultCacheExtent = 250.0;
}
```



## RenderViewportBase

```dart
abstract class RenderViewportBase<ParentDataClass extends ContainerParentDataMixin<RenderSliver>>
    extends RenderBox with ContainerRenderObjectMixin<RenderSliver, ParentDataClass>
    implements RenderAbstractViewport {
  /// Initializes fields for subclasses.
  RenderViewportBase({
    AxisDirection axisDirection = AxisDirection.down,
    @required AxisDirection crossAxisDirection,
    @required ViewportOffset offset,
    double cacheExtent,
  }) : assert(axisDirection != null),
       assert(crossAxisDirection != null),
       assert(offset != null),
       assert(axisDirectionToAxis(axisDirection) != axisDirectionToAxis(crossAxisDirection)),
       _axisDirection = axisDirection,
       _crossAxisDirection = crossAxisDirection,
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

  /// The direction in which the [SliverConstraints.scrollOffset] increases.
  ///
  /// For example, if the [axisDirection] is [AxisDirection.down], a scroll
  /// offset of zero is at the top of the viewport and increases towards the
  /// bottom of the viewport.
  AxisDirection get axisDirection => _axisDirection;
  AxisDirection _axisDirection;
  set axisDirection(AxisDirection value) {
    assert(value != null);
    if (value == _axisDirection)
      return;
    _axisDirection = value;
    markNeedsLayout();
  }

  /// The direction in which child should be laid out in the cross axis.
  ///
  /// For example, if the [axisDirection] is [AxisDirection.down], this property
  /// is typically [AxisDirection.left] if the ambient [TextDirection] is
  /// [TextDirection.rtl] and [AxisDirection.right] if the ambient
  /// [TextDirection] is [TextDirection.ltr].
  AxisDirection get crossAxisDirection => _crossAxisDirection;
  AxisDirection _crossAxisDirection;
  set crossAxisDirection(AxisDirection value) {
    assert(value != null);
    if (value == _crossAxisDirection)
      return;
    _crossAxisDirection = value;
    markNeedsLayout();
  }

  /// The axis along which the viewport scrolls.
  ///
  /// For example, if the [axisDirection] is [AxisDirection.down], then the
  /// [axis] is [Axis.vertical] and the viewport scrolls vertically.
  Axis get axis => axisDirectionToAxis(axisDirection);

  /// Which part of the content inside the viewport should be visible.
  ///
  /// The [ViewportOffset.pixels] value determines the scroll offset that the
  /// viewport uses to select which part of its content to display. As the user
  /// scrolls the viewport, this value changes, which changes the content that
  /// is displayed.
  ViewportOffset get offset => _offset;
  ViewportOffset _offset;
  set offset(ViewportOffset value) {
    assert(value != null);
    if (value == _offset)
      return;
    if (attached)
      _offset.removeListener(markNeedsLayout);
    _offset = value;
    if (attached)
      _offset.addListener(markNeedsLayout);
    // We need to go through layout even if the new offset has the same pixels
    // value as the old offset so that we will apply our viewport and content
    // dimensions.
    markNeedsLayout();
  }

  /// {@template flutter.rendering.viewport.cacheExtent}
  /// The viewport has an area before and after the visible area to cache items
  /// that are about to become visible when the user scrolls.
  ///
  /// Items that fall in this cache area are laid out even though they are not
  /// (yet) visible on screen. The [cacheExtent] describes how many pixels
  /// the cache area extends before the leading edge and after the trailing edge
  /// of the viewport.
  ///
  /// The total extent, which the viewport will try to cover with children, is
  /// [cacheExtent] before the leading edge + extent of the main axis +
  /// [cacheExtent] after the trailing edge.
  ///
  /// The cache area is also used to implement implicit accessibility scrolling
  /// on iOS: When the accessibility focus moves from an item in the visible
  /// viewport to an invisible item in the cache area, the framework will bring
  /// that item into view with an (implicit) scroll action.
  /// {@endtemplate}
  double get cacheExtent => _cacheExtent;
  double _cacheExtent;
  set cacheExtent(double value) {
    value = value ?? RenderAbstractViewport.defaultCacheExtent;
    assert(value != null);
    if (value == _cacheExtent)
      return;
    _cacheExtent = value;
    markNeedsLayout();
  }

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

  /// Throws an exception saying that the object does not support returning
  /// intrinsic dimensions if, in checked mode, we are not in the
  /// [RenderObject.debugCheckingIntrinsics] mode.
  ///
  /// This is used by [computeMinIntrinsicWidth] et al because viewports do not
  /// generally support returning intrinsic dimensions. See the discussion at
  /// [computeMinIntrinsicWidth].
  @protected
  bool debugThrowIfNotCheckingIntrinsics() {
    assert(() {
      if (!RenderObject.debugCheckingIntrinsics) {
        assert(this is! RenderShrinkWrappingViewport); // it has its own message
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('$runtimeType does not support returning intrinsic dimensions.'),
          ErrorDescription(
            'Calculating the intrinsic dimensions would require instantiating every child of '
            'the viewport, which defeats the point of viewports being lazy.',
          ),
          ErrorHint(
            'If you are merely trying to shrink-wrap the viewport in the main axis direction, '
            'consider a RenderShrinkWrappingViewport render object (ShrinkWrappingViewport widget), '
            'which achieves that effect without implementing the intrinsic dimension API.'
          ),
        ]);
      }
      return true;
    }());
    return true;
  }

  @override
  double computeMinIntrinsicWidth(double height) {
    assert(debugThrowIfNotCheckingIntrinsics());
    return 0.0;
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    assert(debugThrowIfNotCheckingIntrinsics());
    return 0.0;
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    assert(debugThrowIfNotCheckingIntrinsics());
    return 0.0;
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    assert(debugThrowIfNotCheckingIntrinsics());
    return 0.0;
  }

  @override
  bool get isRepaintBoundary => true;

  /// Determines the size and position of some of the children of the viewport.
  ///
  /// This function is the workhorse of `performLayout` implementations in
  /// subclasses.
  ///
  /// Layout starts with `child`, proceeds according to the `advance` callback,
  /// and stops once `advance` returns null.
  ///
  ///  * `scrollOffset` is the [SliverConstraints.scrollOffset] to pass the
  ///    first child. The scroll offset is adjusted by
  ///    [SliverGeometry.scrollExtent] for subsequent children.
  ///  * `overlap` is the [SliverConstraints.overlap] to pass the first child.
  ///    The overlay is adjusted by the [SliverGeometry.paintOrigin] and
  ///    [SliverGeometry.paintExtent] for subsequent children.
  ///  * `layoutOffset` is the layout offset at which to place the first child.
  ///    The layout offset is updated by the [SliverGeometry.layoutExtent] for
  ///    subsequent children.
  ///  * `remainingPaintExtent` is [SliverConstraints.remainingPaintExtent] to
  ///    pass the first child. The remaining paint extent is updated by the
  ///    [SliverGeometry.layoutExtent] for subsequent children.
  ///  * `mainAxisExtent` is the [SliverConstraints.viewportMainAxisExtent] to
  ///    pass to each child.
  ///  * `crossAxisExtent` is the [SliverConstraints.crossAxisExtent] to pass to
  ///    each child.
  ///  * `growthDirection` is the [SliverConstraints.growthDirection] to pass to
  ///    each child.
  ///
  /// Returns the first non-zero [SliverGeometry.scrollOffsetCorrection]
  /// encountered, if any. Otherwise returns 0.0. Typical callers will call this
  /// function repeatedly until it returns 0.0.
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
    assert(scrollOffset.isFinite);
    assert(scrollOffset >= 0.0);
    final double initialLayoutOffset = layoutOffset;
    final ScrollDirection adjustedUserScrollDirection =
        applyGrowthDirectionToScrollDirection(offset.userScrollDirection, growthDirection);
    assert(adjustedUserScrollDirection != null);
    double maxPaintOffset = layoutOffset + overlap;
    double precedingScrollExtent = 0.0;

    while (child != null) {
      final double sliverScrollOffset = scrollOffset <= 0.0 ? 0.0 : scrollOffset;
      // If the scrollOffset is too small we adjust the paddedOrigin because it
      // doesn't make sense to ask a sliver for content before its scroll
      // offset.
      final double correctedCacheOrigin = math.max(cacheOrigin, -sliverScrollOffset);
      final double cacheExtentCorrection = cacheOrigin - correctedCacheOrigin;

      assert(sliverScrollOffset >= correctedCacheOrigin.abs());
      assert(correctedCacheOrigin <= 0.0);
      assert(sliverScrollOffset >= 0.0);
      assert(cacheExtentCorrection <= 0.0);

      child.layout(SliverConstraints(
        axisDirection: axisDirection,
        growthDirection: growthDirection,
        userScrollDirection: adjustedUserScrollDirection,
        scrollOffset: sliverScrollOffset,
        precedingScrollExtent: precedingScrollExtent,
        overlap: maxPaintOffset - layoutOffset,
        remainingPaintExtent: math.max(0.0, remainingPaintExtent - layoutOffset + initialLayoutOffset),
        crossAxisExtent: crossAxisExtent,
        crossAxisDirection: crossAxisDirection,
        viewportMainAxisExtent: mainAxisExtent,
        remainingCacheExtent: math.max(0.0, remainingCacheExtent + cacheExtentCorrection),
        cacheOrigin: correctedCacheOrigin,
      ), parentUsesSize: true);

      final SliverGeometry childLayoutGeometry = child.geometry;
      assert(childLayoutGeometry.debugAssertIsValid());

      // If there is a correction to apply, we'll have to start over.
      if (childLayoutGeometry.scrollOffsetCorrection != null)
        return childLayoutGeometry.scrollOffsetCorrection;

      // We use the child's paint origin in our coordinate system as the
      // layoutOffset we store in the child's parent data.
      final double effectiveLayoutOffset = layoutOffset + childLayoutGeometry.paintOrigin;

      // `effectiveLayoutOffset` becomes meaningless once we moved past the trailing edge
      // because `childLayoutGeometry.layoutExtent` is zero. Using the still increasing
      // 'scrollOffset` to roughly position these invisible slivers in the right order.
      if (childLayoutGeometry.visible || scrollOffset > 0) {
        updateChildLayoutOffset(child, effectiveLayoutOffset, growthDirection);
      } else {
        updateChildLayoutOffset(child, -scrollOffset + initialLayoutOffset, growthDirection);
      }

      maxPaintOffset = math.max(effectiveLayoutOffset + childLayoutGeometry.paintExtent, maxPaintOffset);
      scrollOffset -= childLayoutGeometry.scrollExtent;
      precedingScrollExtent += childLayoutGeometry.scrollExtent;
      layoutOffset += childLayoutGeometry.layoutExtent;
      if (childLayoutGeometry.cacheExtent != 0.0) {
        remainingCacheExtent -= childLayoutGeometry.cacheExtent - cacheExtentCorrection;
        cacheOrigin = math.min(correctedCacheOrigin + childLayoutGeometry.cacheExtent, 0.0);
      }

      updateOutOfBandData(growthDirection, childLayoutGeometry);

      // move on to the next child
      child = advance(child);
    }

    // we made it without a correction, whee!
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
    final double startOfOverlap = child.constraints.viewportMainAxisExtent - child.constraints.remainingPaintExtent;
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

  void _paintContents(PaintingContext context, Offset offset) {
    for (RenderSliver child in childrenInPaintOrder) {
      if (child.geometry.visible)
        context.paintChild(child, offset + paintOffsetOf(child));
    }
  }

  @override
  void debugPaintSize(PaintingContext context, Offset offset) {
    assert(() {
      super.debugPaintSize(context, offset);
      final Paint paint = Paint()
        ..style = PaintingStyle.stroke
        ..strokeWidth = 1.0
        ..color = const Color(0xFF00FF00);
      final Canvas canvas = context.canvas;
      RenderSliver child = firstChild;
      while (child != null) {
        Size size;
        switch (axis) {
          case Axis.vertical:
            size = Size(child.constraints.crossAxisExtent, child.geometry.layoutExtent);
            break;
          case Axis.horizontal:
            size = Size(child.geometry.layoutExtent, child.constraints.crossAxisExtent);
            break;
        }
        assert(size != null);
        canvas.drawRect(((offset + paintOffsetOf(child)) & size).deflate(0.5), paint);
        child = childAfter(child);
      }
      return true;
    }());
  }

  @override
  bool hitTestChildren(BoxHitTestResult result, { Offset position }) {
    double mainAxisPosition, crossAxisPosition;
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
    assert(mainAxisPosition != null);
    assert(crossAxisPosition != null);
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
      assert(child.parent != null, '$target must be a descendant of $this');
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
      assert(pivot.parent != null);
      assert(pivot.parent != this);
      assert(pivot != this);
      assert(pivot.parent is RenderSliver);  // TODO(abarth): Support other kinds of render objects besides slivers.
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

    final double targetOffset = leadingScrollOffset - (mainAxisExtent - targetMainAxisExtent) * alignment;
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

  /// The offset at which the given `child` should be painted.
  ///
  /// The returned offset is from the top left corner of the inside of the
  /// viewport to the top left corner of the paint coordinate system of the
  /// `child`.
  ///
  /// See also [paintOffsetOf], which uses the layout offset and growth
  /// direction computed for the child during layout.
  @protected
  Offset computeAbsolutePaintOffset(RenderSliver child, double layoutOffset, GrowthDirection growthDirection) {
    assert(hasSize); // this is only usable once we have a size
    assert(axisDirection != null);
    assert(growthDirection != null);
    assert(child != null);
    assert(child.geometry != null);
    switch (applyGrowthDirectionToAxisDirection(axisDirection, growthDirection)) {
      case AxisDirection.up:
        return Offset(0.0, size.height - (layoutOffset + child.geometry.paintExtent));
      case AxisDirection.right:
        return Offset(layoutOffset, 0.0);
      case AxisDirection.down:
        return Offset(0.0, layoutOffset);
      case AxisDirection.left:
        return Offset(size.width - (layoutOffset + child.geometry.paintExtent), 0.0);
    }
    return null;
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(EnumProperty<AxisDirection>('axisDirection', axisDirection));
    properties.add(EnumProperty<AxisDirection>('crossAxisDirection', crossAxisDirection));
    properties.add(DiagnosticsProperty<ViewportOffset>('offset', offset));
  }

  @override
  List<DiagnosticsNode> debugDescribeChildren() {
    final List<DiagnosticsNode> children = <DiagnosticsNode>[];
    RenderSliver child = firstChild;
    if (child == null)
      return children;

    int count = indexOfFirstChild;
    while (true) {
      children.add(child.toDiagnosticsNode(name: labelForChild(count)));
      if (child == lastChild)
        break;
      count += 1;
      child = childAfter(child);
    }
    return children;
  }

  // API TO BE IMPLEMENTED BY SUBCLASSES

  // setupParentData

  // performLayout (and optionally sizedByParent and performResize)

  /// Whether the contents of this viewport would paint outside the bounds of
  /// the viewport if [paint] did not clip.
  ///
  /// This property enables an optimization whereby [paint] can skip apply a
  /// clip of the contents of the viewport are known to paint entirely within
  /// the bounds of the viewport.
  @protected
  bool get hasVisualOverflow;

  /// Called during [layoutChildSequence] for each child.
  ///
  /// Typically used by subclasses to update any out-of-band data, such as the
  /// max scroll extent, for each child.
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
  void updateChildLayoutOffset(RenderSliver child, double layoutOffset, GrowthDirection growthDirection);

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

  /// The index of the first child of the viewport relative to the center child.
  ///
  /// For example, the center child has index zero and the first child in the
  /// reverse growth direction has index -1.
  @protected
  int get indexOfFirstChild;

  /// A short string to identify the child with the given index.
  ///
  /// Used by [debugDescribeChildren] to label the children.
  @protected
  String labelForChild(int index);

  /// Provides an iterable that walks the children of the viewport, in the order
  /// that they should be painted.
  ///
  /// This should be the reverse order of [childrenInHitTestOrder].
  @protected
  Iterable<RenderSliver> get childrenInPaintOrder;

  /// Provides an iterable that walks the children of the viewport, in the order
  /// that hit-testing should use.
  ///
  /// This should be the reverse order of [childrenInPaintOrder].
  @protected
  Iterable<RenderSliver> get childrenInHitTestOrder;

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
    final RevealedOffset leadingEdgeOffset = viewport.getOffsetToReveal(descendant, 0.0, rect: rect);
    final RevealedOffset trailingEdgeOffset = viewport.getOffsetToReveal(descendant, 1.0, rect: rect);
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
class RenderViewport extends RenderViewportBase<SliverPhysicalContainerParentData> {
  /// Creates a viewport for [RenderSliver] objects.
  ///
  /// If the [center] is not specified, then the first child in the `children`
  /// list, if any, is used.
  ///
  /// The [offset] must be specified. For testing purposes, consider passing a
  /// [new ViewportOffset.zero] or [new ViewportOffset.fixed].
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
       super(axisDirection: axisDirection, crossAxisDirection: crossAxisDirection, offset: offset, cacheExtent: cacheExtent) {
    addAll(children);
    if (center == null && firstChild != null)
      _center = firstChild;
  }

  /// If a [RenderAbstractViewport] overrides
  /// [RenderObject.describeSemanticsConfiguration] to add the [SemanticsTag]
  /// [useTwoPaneSemantics] to its [SemanticsConfiguration], two semantics nodes
  /// will be used to represent the viewport with its associated scrolling
  /// actions in the semantics tree.
  ///
  /// Two semantics nodes (an inner and an outer node) are necessary to exclude
  /// certain child nodes (via the [excludeFromScrolling] tag) from the
  /// scrollable area for semantic purposes: The [SemanticsNode]s of children
  /// that should be excluded from scrolling will be attached to the outer node.
  /// The semantic scrolling actions and the [SemanticsNode]s of scrollable
  /// children will be attached to the inner node, which itself is a child of
  /// the outer node.
  static const SemanticsTag useTwoPaneSemantics = SemanticsTag('RenderViewport.twoPane');

  /// When a top-level [SemanticsNode] below a [RenderAbstractViewport] is
  /// tagged with [excludeFromScrolling] it will not be part of the scrolling
  /// area for semantic purposes.
  ///
  /// This behavior is only active if the [RenderAbstractViewport]
  /// tagged its [SemanticsConfiguration] with [useTwoPaneSemantics].
  /// Otherwise, the [excludeFromScrolling] tag is ignored.
  ///
  /// As an example, a [RenderSliver] that stays on the screen within a
  /// [Scrollable] even though the user has scrolled past it (e.g. a pinned app
  /// bar) can tag its [SemanticsNode] with [excludeFromScrolling] to indicate
  /// that it should no longer be considered for semantic actions related to
  /// scrolling.
  static const SemanticsTag excludeFromScrolling = SemanticsTag('RenderViewport.excludeFromScrolling');

  @override
  void setupParentData(RenderObject child) {
    if (child.parentData is! SliverPhysicalContainerParentData)
      child.parentData = SliverPhysicalContainerParentData();
  }

  /// The relative position of the zero scroll offset.
  ///
  /// For example, if [anchor] is 0.5 and the [axisDirection] is
  /// [AxisDirection.down] or [AxisDirection.up], then the zero scroll offset is
  /// vertically centered within the viewport. If the [anchor] is 1.0, and the
  /// [axisDirection] is [AxisDirection.right], then the zero scroll offset is
  /// on the left edge of the viewport.
  double get anchor => _anchor;
  double _anchor;
  set anchor(double value) {
    assert(value != null);
    assert(value >= 0.0 && value <= 1.0);
    if (value == _anchor)
      return;
    _anchor = value;
    markNeedsLayout();
  }

  /// The first child in the [GrowthDirection.forward] growth direction.
  ///
  /// This child that will be at the position defined by [anchor] when the
  /// [offset.pixels] is `0`.
  ///
  /// Children after [center] will be placed in the [axisDirection] relative to
  /// the [center]. Children before [center] will be placed in the opposite of
  /// the [axisDirection] relative to the [center].
  ///
  /// The [center] must be a child of the viewport.
  RenderSliver get center => _center;
  RenderSliver _center;
  set center(RenderSliver value) {
    if (value == _center)
      return;
    _center = value;
    markNeedsLayout();
  }

  @override
  bool get sizedByParent => true;

  @override
  void performResize() {
    assert(() {
      if (!constraints.hasBoundedHeight || !constraints.hasBoundedWidth) {
        switch (axis) {
          case Axis.vertical:
            if (!constraints.hasBoundedHeight) {
              throw FlutterError.fromParts(<DiagnosticsNode>[
                ErrorSummary('Vertical viewport was given unbounded height.'),
                ErrorDescription(
                  'Viewports expand in the scrolling direction to fill their container. '
                  'In this case, a vertical viewport was given an unlimited amount of '
                  'vertical space in which to expand. This situation typically happens '
                  'when a scrollable widget is nested inside another scrollable widget.'
                ),
                ErrorHint(
                  'If this widget is always nested in a scrollable widget there '
                  'is no need to use a viewport because there will always be enough '
                  'vertical space for the children. In this case, consider using a '
                  'Column instead. Otherwise, consider using the "shrinkWrap" property '
                  '(or a ShrinkWrappingViewport) to size the height of the viewport '
                  'to the sum of the heights of its children.'
                )
              ]);
            }
            if (!constraints.hasBoundedWidth) {
              throw FlutterError(
                'Vertical viewport was given unbounded width.\n'
                'Viewports expand in the cross axis to fill their container and '
                'constrain their children to match their extent in the cross axis. '
                'In this case, a vertical viewport was given an unlimited amount of '
                'horizontal space in which to expand.'
              );
            }
            break;
          case Axis.horizontal:
            if (!constraints.hasBoundedWidth) {
              throw FlutterError.fromParts(<DiagnosticsNode>[
                ErrorSummary('Horizontal viewport was given unbounded width.'),
                ErrorDescription(
                  'Viewports expand in the scrolling direction to fill their container.'
                  'In this case, a horizontal viewport was given an unlimited amount of '
                  'horizontal space in which to expand. This situation typically happens '
                  'when a scrollable widget is nested inside another scrollable widget.'
                ),
                ErrorHint(
                  'If this widget is always nested in a scrollable widget there '
                  'is no need to use a viewport because there will always be enough '
                  'horizontal space for the children. In this case, consider using a '
                  'Row instead. Otherwise, consider using the "shrinkWrap" property '
                  '(or a ShrinkWrappingViewport) to size the width of the viewport '
                  'to the sum of the widths of its children.'
                )
              ]);
            }
            if (!constraints.hasBoundedHeight) {
              throw FlutterError(
                'Horizontal viewport was given unbounded height.\n'
                'Viewports expand in the cross axis to fill their container and '
                'constrain their children to match their extent in the cross axis. '
                'In this case, a horizontal viewport was given an unlimited amount of '
                'vertical space in which to expand.'
              );
            }
            break;
        }
      }
      return true;
    }());
    size = constraints.biggest;
    // We ignore the return value of applyViewportDimension below because we are
    // going to go through performLayout next regardless.
    switch (axis) {
      case Axis.vertical:
        offset.applyViewportDimension(size.height);
        break;
      case Axis.horizontal:
        offset.applyViewportDimension(size.width);
        break;
    }
  }

  static const int _maxLayoutCycles = 10;

  // Out-of-band data computed during layout.
  double _minScrollExtent;
  double _maxScrollExtent;
  bool _hasVisualOverflow = false;

  @override
  void performLayout() {
    if (center == null) {
      assert(firstChild == null);
      _minScrollExtent = 0.0;
      _maxScrollExtent = 0.0;
      _hasVisualOverflow = false;
      offset.applyContentDimensions(0.0, 0.0);
      return;
    }
    assert(center.parent == this);

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
      assert(offset.pixels != null);
      correction = _attemptLayout(mainAxisExtent, crossAxisExtent, offset.pixels + centerOffsetAdjustment);
      if (correction != 0.0) {
        offset.correctBy(correction);
      } else {
        if (offset.applyContentDimensions(
              math.min(0.0, _minScrollExtent + mainAxisExtent * anchor),
              math.max(0.0, _maxScrollExtent - mainAxisExtent * (1.0 - anchor)),
           ))
          break;
      }
      count += 1;
    } while (count < _maxLayoutCycles);
    assert(() {
      if (count >= _maxLayoutCycles) {
        assert(count != 1);
        throw FlutterError(
          'A RenderViewport exceeded its maximum number of layout cycles.\n'
          'RenderViewport render objects, during layout, can retry if either their '
          'slivers or their ViewportOffset decide that the offset should be corrected '
          'to take into account information collected during that layout.\n'
          'In the case of this RenderViewport object, however, this happened $count '
          'times and still there was no consensus on the scroll offset. This usually '
          'indicates a bug. Specifically, it means that one of the following three '
          'problems is being experienced by the RenderViewport object:\n'
          ' * One of the RenderSliver children or the ViewportOffset have a bug such'
          ' that they always think that they need to correct the offset regardless.\n'
          ' * Some combination of the RenderSliver children and the ViewportOffset'
          ' have a bad interaction such that one applies a correction then another'
          ' applies a reverse correction, leading to an infinite loop of corrections.\n'
          ' * There is a pathological case that would eventually resolve, but it is'
          ' so complicated that it cannot be resolved in any reasonable number of'
          ' layout passes.'
        );
      }
      return true;
    }());
  }

  double _attemptLayout(double mainAxisExtent, double crossAxisExtent, double correctedOffset) {
    assert(!mainAxisExtent.isNaN);
    assert(mainAxisExtent >= 0.0);
    assert(crossAxisExtent.isFinite);
    assert(crossAxisExtent >= 0.0);
    assert(correctedOffset.isFinite);
    _minScrollExtent = 0.0;
    _maxScrollExtent = 0.0;
    _hasVisualOverflow = false;

    // centerOffset is the offset from the leading edge of the RenderViewport
    // to the zero scroll offset (the line between the forward slivers and the
    // reverse slivers).
    final double centerOffset = mainAxisExtent * anchor - correctedOffset;
    final double reverseDirectionRemainingPaintExtent = centerOffset.clamp(0.0, mainAxisExtent);
    final double forwardDirectionRemainingPaintExtent = (mainAxisExtent - centerOffset).clamp(0.0, mainAxisExtent);

    final double fullCacheExtent = mainAxisExtent + 2 * cacheExtent;
    final double centerCacheOffset = centerOffset + cacheExtent;
    final double reverseDirectionRemainingCacheExtent = centerCacheOffset.clamp(0.0, fullCacheExtent);
    final double forwardDirectionRemainingCacheExtent = (fullCacheExtent - centerCacheOffset).clamp(0.0, fullCacheExtent);

    final RenderSliver leadingNegativeChild = childBefore(center);

    if (leadingNegativeChild != null) {
      // negative scroll offsets
      final double result = layoutChildSequence(
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

    // positive scroll offsets
    return layoutChildSequence(
      child: center,
      scrollOffset: math.max(0.0, -centerOffset),
      overlap: leadingNegativeChild == null ? math.min(0.0, -centerOffset) : 0.0,
      layoutOffset: centerOffset >= mainAxisExtent ? centerOffset: reverseDirectionRemainingPaintExtent,
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
    if (childLayoutGeometry.hasVisualOverflow)
      _hasVisualOverflow = true;
  }

  @override
  void updateChildLayoutOffset(RenderSliver child, double layoutOffset, GrowthDirection growthDirection) {
    final SliverPhysicalParentData childParentData = child.parentData;
    childParentData.paintOffset = computeAbsolutePaintOffset(child, layoutOffset, growthDirection);
  }

  @override
  Offset paintOffsetOf(RenderSliver child) {
    final SliverPhysicalParentData childParentData = child.parentData;
    return childParentData.paintOffset;
  }

  @override
  double scrollOffsetOf(RenderSliver child, double scrollOffsetWithinChild) {
    assert(child.parent == this);
    final GrowthDirection growthDirection = child.constraints.growthDirection;
    assert(growthDirection != null);
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

  @override
  void applyPaintTransform(RenderObject child, Matrix4 transform) {
    assert(child != null);
    final SliverPhysicalParentData childParentData = child.parentData;
    childParentData.applyPaintTransform(transform);
  }

  @override
  double computeChildMainAxisPosition(RenderSliver child, double parentMainAxisPosition) {
    assert(child != null);
    assert(child.constraints != null);
    final SliverPhysicalParentData childParentData = child.parentData;
    switch (applyGrowthDirectionToAxisDirection(child.constraints.axisDirection, child.constraints.growthDirection)) {
      case AxisDirection.down:
        return parentMainAxisPosition - childParentData.paintOffset.dy;
      case AxisDirection.right:
        return parentMainAxisPosition - childParentData.paintOffset.dx;
      case AxisDirection.up:
        return child.geometry.paintExtent - (parentMainAxisPosition - childParentData.paintOffset.dy);
      case AxisDirection.left:
        return child.geometry.paintExtent - (parentMainAxisPosition - childParentData.paintOffset.dx);
    }
    return 0.0;
  }

  @override
  int get indexOfFirstChild {
    assert(center != null);
    assert(center.parent == this);
    assert(firstChild != null);
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

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(DoubleProperty('anchor', anchor));
  }
}
```


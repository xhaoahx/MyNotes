# Flutter RenderBox 源码分析

## BoxParentData

```dart
class BoxParentData extends ParentData {
  /// The offset at which to paint the child in the parent's coordinate system.
  Offset offset = Offset.zero;

  @override
  String toString() => 'offset=$offset';
}
```



## RenderBox

```dart
/// 位于 2D 笛卡尔坐标系中的渲染对象
///
/// 每个框的[尺寸]表示为一个宽度和一个高度。每个盒子都有自己的坐标系，其左上角在(0,0)处，因此盒子的右下角
/// 在(宽，高)处。该框包含所有点，包括左上角和延伸到右下角，但不包括右下角。
///
/// 通过向树下传递一个[BoxConstraints]对象来执行盒子布局。
/// 盒约束为子节点的宽度和高度确定了一个最小值和最大值。在确定其大小时，子接待了必须遵守父节点给它的约束。
///
/// 此协议足以表示许多公共盒布局数据流。例如，要实现宽度加高的数据流，请使用一组具有窄宽度值的框约束调用子
/// [layout]函数(并为parentUsesSize传递true)。在孩子确定它的高度之后，使用孩子的高度来确定自身的大小。
///
/// ## Writing a RenderBox subclass
///
/// 可以实现一个新的[RenderBox]子类来描述一个新的布局模型，新的绘制模型，新的点击测试模型，或者新的语义
/// 模型，同时保持在由[RenderBox]协议定义的笛卡尔坐标系
///
/// 为了创建一个新的协议，继承[RenderObject]
///
/// ### [RenderBox]子类的构造函数和属性
///
/// 每个属性都有如下的 getter / setter格式：
///
/// ```dart
/// AxisDirection get axis => _axis;
/// AxisDirection _axis;
/// set axis(AxisDirection value) {
///   if (value == _axis)
///     return;
///   _axis = value;
///   markNeedsLayout();
/// }
/// ```
///
/// 如果布局使用了这个属性，setter通常会以调用[markNeedsLayout]结束，或者[markNeedsPaint]，如果只有
/// 绘制使用了这个属性。(不需要两者都调用，[markNeedsLayout]意味着[markNeedsPaint]。)
///
/// 考虑到布局和绘制是相对昂贵的;对于调用[markNeedsLayout]或[markNeedsPaint]要保守。只有在布局(或绘
/// 制)实际发生变化时才应该调用它们。
///
/// ### 子节点
///
/// 如果一个渲染对象是一个叶子，也就是说，它不能有任何子对象，那么忽略这个部分。(叶子渲染对象的例子是
/// [RenderImage]和[RenderParagraph]。)
///
/// 对于带有子对象的渲染对象，有四种可能的场景:
///
/// 1.只有一个单独的[RenderBox]子元素。在这个场景中，可以考虑从[RenderProxyBox](这个渲染通过改变它自
///   身的大小来适配孩子节点)或[RenderShiftedBox](如果子节点比盒子小，并且盒子内部会对齐子节点)继承。
///
/// 2.只有一个单一的子节点，但不是[RenderBox]。使用[RenderObjectWithChildMixin] mixin.
///
/// 3.一组孩子列表，使用[ContainerRenderObjectMixin] mixin.
///
/// 4.更加复杂的模型
///
/// #### 使用 RenderProxyBox
///
/// 默认情况下，[RenderProxyBox]渲染对象的改变自身的大小来适配它的孩子，或者在没有子节点的时候尽可能小;
/// 它将所有的点击测试和绘制工作委任传递给子节点，类似地固有维度和基线的测量方式也同样委任给孩子。
///
/// [RenderProxyBox]的子类只需要重载[RenderBox]协议中重要的部分。
/// 例如，[render]只是重载了 paint 方法和[alwaysNeedsCompositing]，并添
/// 加了一个[renderOpacity.opacity]域)。
///
/// [RenderProxyBox] 假设直接当放置在 0,0 的位置，否则使用 [RenderShiftedBox]
///
/// #### 使用 RenderShiftedBox
///
/// 默认情况下，[RenderShiftedBox]的行为与[RenderProxyBox]很相似，但是没有假设子节点的位置是0,0(而
/// 是使用子节点的[parentData]字段中记录的实际偏移位置)，也没有提供默认的布局算法。
///
/// #### 多个子节点
///
/// 一个[RenderBox]的子节点不一定为[RenderBox]，可以将[RenderObject]的子类作为[RenderBox]的孩子
///
/// 子节点可以拥有被父节点持有的额外数据，这届数据必须存储在子节点的[parentData]域上。
/// 用于储存该数据的类必须继承自[ParentData]。[setupParentData]方法用于在挂载子节点时初始化该子节点的
/// 的[parentData]字段。
///
/// 按照惯例，具有[RenderBox]子节点的[RenderBox]让其孩子使用[BoxParentData]类，该类有一个
/// [BoxParentData.offset]字段来存储子元素相对于父元素的位置。([RenderProxyBox]不需要这个偏移量，因
/// 此是这个规则的一个例外。)
///
/// #### 更复杂的模型
///
/// 渲染对象可以有更复杂的模型，例如以enum为key值的子节点的映射，或者高效随机访问的子节点的2D网格，或者多
/// 个子节点列表，等等。如果一个渲染对象的模型不能被上面的mixin处理，它必须实现[RenderObject]的协议，如
/// 下所示:
///
/// 1.任何时候移除一个孩子, 调用 [dropChild]
///
/// 2.任何时候添加一个孩子，调用 [adoptChild]
///
/// 3.实现在每一个子节点上调用的 [attach] 方法
///
/// 4.实现在每一个子节点上调用的 [unattach] 方法
///
/// 5.实现[redepthChildren] 方法调用每一个孩子的 [redepthChild]
///
/// 6.实现 [visitChildren] 方法它的每一个孩子, 通常以绘制的顺序：(从后到前).
///
/// 实现这这些要点实际上就是前面提到的两个mixin所做的全部工作。
///
/// ### 布局
///
/// [RenderBox] 类实现了布局算法。它们有一组提供给它的约束，它根据这些约束和它们可能拥有的任何其他输入
/// (例如，它们的子节点或属性)来调整自己的大小。
///
/// 在实现[RenderBox]子类时，必须做出选择：它只根据约束来调整自身的大小，或者使用其他信息来调整自身的大
/// 小。纯基于约束的大小调整的一个例子是扩展以适应父级。
///
/// 纯基于约束的大小调整允许系统进行一些重要的优化。使用这种方法的类应该重载[sizedByParent]来返回 
/// true，然后重载[performResize]来设置[size]，只使用约束，例如:
///
/// ```dart
/// @override
/// bool get sizedByParent => true;
///
/// @override
/// void performResize() {
///   size = constraints.smallest;
/// }
/// ```
///
/// 否则，其大小在[performLayout]里设定
///
/// [performLayout]函数是盒子用于决定，假如它们不是[sizedByParent]，那么它们的大小应该如何，并且决定
/// 它的子节点应该放置在哪里
///
/// ### Painting
///
/// 要描述渲染框如何绘制，请实现[paint]方法。它有一个[PaintingContext]对象和一个[Offset]。
/// [PaintingContext]提 供了能够影响图层树的方法，比如[PaintingContext.canvas],可用于添加绘图命
/// 令。canvas对象不应该在调用 [PaintingContext]的方法时被缓存;每次调用[PaintingContext]上的方法
/// 时，画布都有可能改变标识。偏移 量指定了框的左上角在[PaintingContext.canvas]的坐标系统中的位置。
///
/// 使用[TextPainter]来绘制文字
///
/// 使用[paintImage]来绘制图片
///
/// 一个[RenderBox]在[PaintingContext]上使用引入新层的方法时应该重载[alwaysNeedsCompositing] 
/// getter并将它设置为true。
/// 如果对象有时做，有时不做，那么在某些情况下，它可以让getter返回true，而在另一些情况下返回false。在这
/// 种情况下，只要返回值发生变化，就调用[markNeedsCompositingBitsUpdate]。(这会在添加或删除子元素时
/// 自动完成，因此如果[alwaysNeedsCompositing] getter只根据子元素的存在或不存在来更改值，则不必显式
/// 地调用它。
///
/// 任何时候，如果有任何变化会导致[paint]方法绘制一些不同的东西(但不会导致布局改变),应该调用
/// [markNeedsPaint]方法
///
/// #### Painting children
///
/// The [paint] method's `context` argument has a [PaintingContext.paintChild]
/// method, which should be called for each child that is to be painted. It
/// should be given a reference to the child, and an [Offset] giving the
/// position of the child relative to the parent.
///
/// 如果[paint]方法在绘制子元素之前对[PaintingContext]应用变换(或者通常应用一个额外的的偏移量，除了作
/// 为参数给出的偏移量)，那么[applyPaintTransform]方法也应该被覆盖。该方法必须调整其给定的矩阵，其方式
/// 与在绘制给定子元素之前转换[PaintingContext]和偏移量的方式相同。
/// 这个方法是被[globalToLocal]和[localToGlobal]方法使用的。
///
/// #### Hit Tests
///
/// 渲染盒子的点击测试是通过[hitTest]方法实现的。此方法的默认实现被推迟给于[hitTestSelf]和
/// [hitTestChildren]。在实现点击测试时，可以重载后两个方法，或者忽略它们，直接重载[hitTest]。
///
/// [hitTest]方法本身有一个[偏移量]，如果对象或它的一个子对象已经吸收了点击(防止下面的对象被点击)，
/// 那么它必须返回true;如果点击可以继续作用于下面的其他对象，则返回false。
///
/// 对于每个子[RenderBox]，应该使用相同的[HitTestResult]参数调用的[hitTest]方法，并将该点转换为子坐
/// 标空间(与[applyPaintTransform]方法相同)。默认的实现延迟给[hitTestChildren]。
/// [RenderBoxContainerDefaultsMixin]提供了一个
/// [RenderBoxContainerDefaultsMixin.defaulthittestchildren]方法，该方法假设子元素是轴向对齐
/// 的，没有被变换的，并且是根据[parentData]字段中[BoxParentData.offset]定位的。更复杂的盒子可以相应
/// 地重载[hitTestChildren]。
///
/// 如果对象被命中，那么它也应该使用[HitTestResult.add]将自己添加到[HitTestResult]对象中。该对象是
/// [hitTest]方法的一个参数。
/// 默认实现依赖于[hitTestSelf]来确定是否击中了该框。如果对象在子对象添加自己之前添加自己，那么就它就处
/// 于子对象之上。如果它加在子元素之后添加自己，那么它就子对象之下。
/// 添加到[HitTestResult]对象的entry应该使用[BoxHitTestEntry]类。然后，系统按照它们被添加的顺序遍历
/// 这些条目，并为每个条目调用目标的[handleEvent]方法，传入[HitTestEntry]对象。
///
///
/// ### Intrinsics and Baselines
///
/// 布局、绘图、命中测试和语义协议对所有渲染对象都是通用的。对象必须实现两个附加协议:固有大小和基线测量。
///
/// 有四个方法来计算固有大小，通过它们来计算最大最小的固有宽高：
/// [computeMinIntrinsicWidth], [computeMaxIntrinsicWidth],
/// [computeMinIntrinsicHeight], [computeMaxIntrinsicHeight].
///
/// 此外，如果一个盒子有任何的子对象，那么它必须实现[computeDistanceToActualBaseline]. 
/// [RenderProxyBox] 它提供了一个简单的实现，可以转发给子节点;
/// [RenderShiftedBox]提供了一个实现，它通过子元素相对于父元素的位置来抵消子元素的基线信息。
/// 如果里没有继承这两个类，则必须自己实现算法
abstract class RenderBox extends RenderObject {
  /// 为孩子建立 BoxParentData  
  @override
  void setupParentData(covariant RenderObject child) {
    if (child.parentData is! BoxParentData)
      child.parentData = BoxParentData();
  }

  /// 固有宽高计算结果的缓存
  Map<_IntrinsicDimensionsCacheEntry, double> _cachedIntrinsicDimensions;

  double _computeIntrinsicDimension(
      _IntrinsicDimension dimension, 
      double argument, 
      double computer(double argument)
  ) {
    bool shouldCache = true;
    if (shouldCache) {
      _cachedIntrinsicDimensions ??= <_IntrinsicDimensionsCacheEntry, double>{};
      return _cachedIntrinsicDimensions.putIfAbsent(
        _IntrinsicDimensionsCacheEntry(dimension, argument),
        () => computer(argument),
      );
    }
    return computer(argument);
  }

  /// 返回这个盒子在不剪裁，能够成功绘制内容的最小宽度
  ///
  /// height参数可以给出一个特定的高度。给定的高度可以是无限的，这意味着在无约束的环境中要求固有宽度。
  /// 给定的高度不应该是负数或null。
  ///
  /// 这个函数只能在自己的孩子身上调用。调用这个函数将子节点和父节点连接起来，这样当子节点的布局改变时，父
  /// 节点就会收到通知(通过[markNeedsLayout])。
  ///
  /// 这个函数的事件复杂度是 O(N^2)
  ///
  /// 不要重载这个函数，而是实现[computeMinIntrinsicWidth].
  @mustCallSuper
  double getMinIntrinsicWidth(double height) {
    return _computeIntrinsicDimension(
        _IntrinsicDimension.minWidth, 
        height, 
        computeMinIntrinsicWidth
    );
  }

  /// 计算[getMinIntrinsicWidth]的返回值
  ///
  /// 在实现了[performLayout]的子类里面重载
  /// 这个方法应该返回这个盒子在不剪裁，能够成功绘制内容的最小宽度
  ///
  /// 如果布局算法是严格的高入宽出，或者是宽入高出，当宽度是无约束的，那么高度参数是使用的高度。
  ///
  /// If this algorithm depends on the intrinsic dimensions of a child, the
  /// intrinsic dimensions of that child should be obtained using the functions
  /// whose names start with `get`, not `compute`.
  ///
  /// This function should never return a negative or infinite value.
  ///
  /// ## Examples
  ///
  /// ### Text
  ///
  /// Text is the canonical example of a width-in-height-out algorithm. The
  /// `height` argument is therefore ignored.
  ///
  /// Consider the string "Hello World" The _maximum_ intrinsic width (as
  /// returned from [computeMaxIntrinsicWidth]) would be the width of the string
  /// with no line breaks.
  ///
  /// The minimum intrinsic width would be the width of the widest word, "Hello"
  /// or "World". If the text is rendered in an even narrower width, however, it
  /// might still not overflow. For example, maybe the rendering would put a
  /// line-break half-way through the words, as in "Hel⁞lo⁞Wor⁞ld". However,
  /// this wouldn't be a _correct_ rendering, and [computeMinIntrinsicWidth] is
  /// supposed to render the minimum width that the box could be without failing
  /// to _correctly_ paint the contents within itself.
  ///
  /// The minimum intrinsic _height_ for a given width smaller than the minimum
  /// intrinsic width could therefore be greater than the minimum intrinsic
  /// height for the minimum intrinsic width.
  ///
  /// ### Viewports (e.g. scrolling lists)
  ///
  /// Some render boxes are intended to clip their children. For example, the
  /// render box for a scrolling list might always size itself to its parents'
  /// size (or rather, to the maximum incoming constraints), regardless of the
  /// children's sizes, and then clip the children and position them based on
  /// the current scroll offset.
  ///
  /// The intrinsic dimensions in these cases still depend on the children, even
  /// though the layout algorithm sizes the box in a way independent of the
  /// children. It is the size that is needed to paint the box's contents (in
  /// this case, the children) _without clipping_ that matters.
  ///
  /// ### When the intrinsic dimensions cannot be known
  ///
  /// There are cases where render objects do not have an efficient way to
  /// compute their intrinsic dimensions. For example, it may be prohibitively
  /// expensive to reify and measure every child of a lazy viewport (viewports
  /// generally only instantiate the actually visible children), or the
  /// dimensions may be computed by a callback about which the render object
  /// cannot reason.
  ///
  /// In such cases, it may be impossible (or at least impractical) to actually
  /// return a valid answer. In such cases, the intrinsic functions should throw
  /// when [RenderObject.debugCheckingIntrinsics] is false and asserts are
  /// enabled, and return 0.0 otherwise.
  ///
  /// See the implementations of [LayoutBuilder] or [RenderViewportBase] for
  /// examples (in particular,
  /// [RenderViewportBase.debugThrowIfNotCheckingIntrinsics]).
  ///
  /// ### Aspect-ratio-driven boxes
  ///
  /// Some boxes always return a fixed size based on the constraints. For these
  /// boxes, the intrinsic functions should return the appropriate size when the
  /// incoming `height` or `width` argument is finite, treating that as a tight
  /// constraint in the respective direction and treating the other direction's
  /// constraints as unbounded. This is because the definitions of
  /// [computeMinIntrinsicWidth] and [computeMinIntrinsicHeight] are in terms of
  /// what the dimensions _could be_, and such boxes can only be one size in
  /// such cases.
  ///
  /// When the incoming argument is not finite, then they should return the
  /// actual intrinsic dimensions based on the contents, as any other box would.
  @protected
  double computeMinIntrinsicWidth(double height) {
    return 0.0;
  }

  /// 返回一个最小宽度，超过此最小宽度时，增加宽度不会降低首选高度。
  /// 首选的高度是[getminininsicheight]为该宽度返回的值。
  @mustCallSuper
  double getMaxIntrinsicWidth(double height) {
    return _computeIntrinsicDimension(
        _IntrinsicDimension.maxWidth, 
        height, 
        computeMaxIntrinsicWidth
    );
  }

  @protected
  double computeMaxIntrinsicWidth(double height) {
    return 0.0;
  }

  @mustCallSuper
  double getMinIntrinsicHeight(double width) {
    return _computeIntrinsicDimension(
        _IntrinsicDimension.minHeight, 
        width, 
        computeMinIntrinsicHeight
    );
  }

  @protected
  double computeMinIntrinsicHeight(double width) {
    return 0.0;
  }

  @mustCallSuper
  double getMaxIntrinsicHeight(double width) {
    return _computeIntrinsicDimension(
        _IntrinsicDimension.maxHeight, 
        width, 
        computeMaxIntrinsicHeight
    );
  }

  @protected
  double computeMaxIntrinsicHeight(double width) {
    return 0.0;
  }

  /// 这个盒子是否已经布局，并且拥有[size].
  bool get hasSize => _size != null;

  Size get size {
    return _size;
  }
  Size _size;
    
  @protected
  set size(Size value) {
    _size = value;
  }

  @override
  Rect get semanticBounds => Offset.zero & size;

  Map<TextBaseline, double> _cachedBaselines;

  /// Returns the distance from the y-coordinate of the position of the box to
  /// the y-coordinate of the first given baseline in the box's contents.
  ///
  /// Used by certain layout models to align adjacent boxes on a common
  /// baseline, regardless of padding, font size differences, etc. If there is
  /// no baseline, this function returns the distance from the y-coordinate of
  /// the position of the box to the y-coordinate of the bottom of the box
  /// (i.e., the height of the box) unless the caller passes true
  /// for `onlyReal`, in which case the function returns null.
  ///
  /// Only call this function after calling [layout] on this box. You
  /// are only allowed to call this from the parent of this box during
  /// that parent's [performLayout] or [paint] functions.
  ///
  /// When implementing a [RenderBox] subclass, to override the baseline
  /// computation, override [computeDistanceToActualBaseline].
  double getDistanceToBaseline(TextBaseline baseline, { bool onlyReal = false }) {
    final double result = getDistanceToActualBaseline(baseline);
    if (result == null && !onlyReal)
      return size.height;
    return result;
  }

  /// Calls [computeDistanceToActualBaseline] and caches the result.
  ///
  /// This function must only be called from [getDistanceToBaseline] and
  /// [computeDistanceToActualBaseline]. Do not call this function directly from
  /// outside those two methods.
  @protected
  @mustCallSuper
  double getDistanceToActualBaseline(TextBaseline baseline) {
    _cachedBaselines ??= <TextBaseline, double>{};
    _cachedBaselines.putIfAbsent(
        baseline, 
        () => computeDistanceToActualBaseline(baseline)
    );
    return _cachedBaselines[baseline];
  }

  /// Returns the distance from the y-coordinate of the position of the box to
  /// the y-coordinate of the first given baseline in the box's contents, if
  /// any, or null otherwise.
  ///
  /// Do not call this function directly. If you need to know the baseline of a
  /// child from an invocation of [performLayout] or [paint], call
  /// [getDistanceToBaseline].
  ///
  /// Subclasses should override this method to supply the distances to their
  /// baselines. When implementing this method, there are generally three
  /// strategies:
  ///
  ///  * For classes that use the [ContainerRenderObjectMixin] child model,
  ///    consider mixing in the [RenderBoxContainerDefaultsMixin] class and
  ///    using
  ///    [RenderBoxContainerDefaultsMixin.defaultComputeDistanceToFirstActualBaseline].
  ///
  ///  * For classes that define a particular baseline themselves, return that
  ///    value directly.
  ///
  ///  * For classes that have a child to which they wish to defer the
  ///    computation, call [getDistanceToActualBaseline] on the child (not
  ///    [computeDistanceToActualBaseline], the internal implementation, and not
  ///    [getDistanceToBaseline], the public entry point for this API).
  @protected
  double computeDistanceToActualBaseline(TextBaseline baseline) {
    return null;
  }

  /// The box constraints most recently received from the parent.
  @override
  BoxConstraints get constraints => super.constraints;

  @override
  void markNeedsLayout() {
    if ((_cachedBaselines != null && _cachedBaselines.isNotEmpty) ||
        (_cachedIntrinsicDimensions != null && _cachedIntrinsicDimensions.isNotEmpty)) {
      _cachedBaselines?.clear();
      _cachedIntrinsicDimensions?.clear();
      if (parent is RenderObject) {
        markParentNeedsLayout();
        return;
      }
    }
    super.markNeedsLayout();
  }

  @override
  void performResize() {
    // default behavior for subclasses that have sizedByParent = true
    size = constraints.smallest;
    assert(size.isFinite);
  }

  @override
  void performLayout() {
  }

  /// Determines the set of render objects located at the given position.
  ///
  /// Returns true, and adds any render objects that contain the point to the
  /// given hit test result, if this render object or one of its descendants
  /// absorbs the hit (preventing objects below this one from being hit).
  /// Returns false if the hit can continue to other objects below this one.
  ///
  /// The caller is responsible for transforming [position] from global
  /// coordinates to its location relative to the origin of this [RenderBox].
  /// This [RenderBox] is responsible for checking whether the given position is
  /// within its bounds.
  ///
  /// If transforming is necessary, [BoxHitTestResult.addWithPaintTransform],
  /// [BoxHitTestResult.addWithPaintOffset], or
  /// [BoxHitTestResult.addWithRawTransform] need to be invoked by the caller
  /// to record the required transform operations in the [HitTestResult]. These
  /// methods will also help with applying the transform to `position`.
  ///
  /// Hit testing requires layout to be up-to-date but does not require painting
  /// to be up-to-date. That means a render object can rely upon [performLayout]
  /// having been called in [hitTest] but cannot rely upon [paint] having been
  /// called. For example, a render object might be a child of a [RenderOpacity]
  /// object, which calls [hitTest] on its children when its opacity is zero
  /// even through it does not [paint] its children.
  bool hitTest(BoxHitTestResult result, { @required Offset position }) {
    if (_size.contains(position)) {
      if (hitTestChildren(result, position: position) || hitTestSelf(position)) {
        result.add(BoxHitTestEntry(this, position));
        return true;
      }
    }
    return false;
  }

  /// Override this method if this render object can be hit even if its
  /// children were not hit.
  ///
  /// The caller is responsible for transforming [position] from global
  /// coordinates to its location relative to the origin of this [RenderBox].
  /// This [RenderBox] is responsible for checking whether the given position is
  /// within its bounds.
  ///
  /// Used by [hitTest]. If you override [hitTest] and do not call this
  /// function, then you don't need to implement this function.
  @protected
  bool hitTestSelf(Offset position) => false;

  /// Override this method to check whether any children are located at the
  /// given position.
  ///
  /// Typically children should be hit-tested in reverse paint order so that
  /// hit tests at locations where children overlap hit the child that is
  /// visually "on top" (i.e., paints later).
  ///
  /// The caller is responsible for transforming [position] from global
  /// coordinates to its location relative to the origin of this [RenderBox].
  /// This [RenderBox] is responsible for checking whether the given position is
  /// within its bounds.
  ///
  /// If transforming is necessary, [HitTestResult.addWithPaintTransform],
  /// [HitTestResult.addWithPaintOffset], or [HitTestResult.addWithRawTransform] need
  /// to be invoked by the caller to record the required transform operations
  /// in the [HitTestResult]. These methods will also help with applying the
  /// transform to `position`.
  ///
  /// Used by [hitTest]. If you override [hitTest] and do not call this
  /// function, then you don't need to implement this function.
  @protected
  bool hitTestChildren(BoxHitTestResult result, { Offset position }) => false;

  /// Multiply the transform from the parent's coordinate system to this box's
  /// coordinate system into the given transform.
  ///
  /// This function is used to convert coordinate systems between boxes.
  /// Subclasses that apply transforms during painting should override this
  /// function to factor those transforms into the calculation.
  ///
  /// The [RenderBox] implementation takes care of adjusting the matrix for the
  /// position of the given child as determined during layout and stored on the
  /// child's [parentData] in the [BoxParentData.offset] field.
  @override
  void applyPaintTransform(RenderObject child, Matrix4 transform) {
    final BoxParentData childParentData = child.parentData;
    final Offset offset = childParentData.offset;
    transform.translate(offset.dx, offset.dy);
  }

  /// Convert the given point from the global coordinate system in logical pixels
  /// to the local coordinate system for this box.
  ///
  /// This method will un-project the point from the screen onto the widget,
  /// which makes it different from [MatrixUtils.transformPoint].
  ///
  /// If the transform from global coordinates to local coordinates is
  /// degenerate, this function returns [Offset.zero].
  ///
  /// If `ancestor` is non-null, this function converts the given point from the
  /// coordinate system of `ancestor` (which must be an ancestor of this render
  /// object) instead of from the global coordinate system.
  ///
  /// This method is implemented in terms of [getTransformTo].
  Offset globalToLocal(Offset point, { RenderObject ancestor }) {
    // We want to find point (p) that corresponds to a given point on the
    // screen (s), but that also physically resides on the local render plane,
    // so that it is useful for visually accurate gesture processing in the
    // local space. For that, we can't simply transform 2D screen point to
    // the 3D local space since the screen space lacks the depth component |z|,
    // and so there are many 3D points that correspond to the screen point.
    // We must first unproject the screen point onto the render plane to find
    // the true 3D point that corresponds to the screen point.
    // We do orthogonal unprojection after undoing perspective, in local space.
    // The render plane is specified by renderBox offset (o) and Z axis (n).
    // Unprojection is done by finding the intersection of the view vector (d)
    // with the local X-Y plane: (o-s).dot(n) == (p-s).dot(n), (p-s) == |z|*d.
    final Matrix4 transform = getTransformTo(ancestor);
    final double det = transform.invert();
    if (det == 0.0)
      return Offset.zero;
    final Vector3 n = Vector3(0.0, 0.0, 1.0);
    final Vector3 i = transform.perspectiveTransform(Vector3(0.0, 0.0, 0.0));
    final Vector3 d = transform.perspectiveTransform(Vector3(0.0, 0.0, 1.0)) - i;
    final Vector3 s = transform.perspectiveTransform(Vector3(point.dx, point.dy, 0.0));
    final Vector3 p = s - d * (n.dot(s) / n.dot(d));
    return Offset(p.x, p.y);
  }

  /// Convert the given point from the local coordinate system for this box to
  /// the global coordinate system in logical pixels.
  ///
  /// If `ancestor` is non-null, this function converts the given point to the
  /// coordinate system of `ancestor` (which must be an ancestor of this render
  /// object) instead of to the global coordinate system.
  ///
  /// This method is implemented in terms of [getTransformTo].
  Offset localToGlobal(Offset point, { RenderObject ancestor }) {
    return MatrixUtils.transformPoint(getTransformTo(ancestor), point);
  }

  /// Returns a rectangle that contains all the pixels painted by this box.
  ///
  /// The paint bounds can be larger or smaller than [size], which is the amount
  /// of space this box takes up during layout. For example, if this box casts a
  /// shadow, that shadow might extend beyond the space allocated to this box
  /// during layout.
  ///
  /// The paint bounds are used to size the buffers into which this box paints.
  /// If the box attempts to paints outside its paint bounds, there might not be
  /// enough memory allocated to represent the box's visual appearance, which
  /// can lead to undefined behavior.
  ///
  /// The returned paint bounds are in the local coordinate system of this box.
  @override
  Rect get paintBounds => Offset.zero & size;

  /// Override this method to handle pointer events that hit this render object.
  ///
  /// For [RenderBox] objects, the `entry` argument is a [BoxHitTestEntry]. From this
  /// object you can determine the [PointerDownEvent]'s position in local coordinates.
  /// (This is useful because [PointerEvent.position] is in global coordinates.)
  ///
  /// If you override this, consider calling [debugHandleEvent] as follows, so
  /// that you can support [debugPaintPointersEnabled]:
  ///
  /// ```dart
  /// @override
  /// void handleEvent(PointerEvent event, HitTestEntry entry) {
  ///   assert(debugHandleEvent(event, entry));
  ///   // ... handle the event ...
  /// }
  /// ```
  @override
  void handleEvent(PointerEvent event, BoxHitTestEntry entry) {
    super.handleEvent(event, entry);
  }
}

```



## RenderBoxContainerDefaultsMixin

```dart
/// 为包含由[ContainerRenderObjectMixin] mixin管理的子元素的盒子提供有用的默认行为的 mixin。
/// 按照惯例，这个类不覆盖超类的任何成员。相反，它提供了有用的函数，子类可以适当地调用这些函数。
mixin RenderBoxContainerDefaultsMixin
  <ChildType extends RenderBox, 
   ParentDataType extends ContainerBoxParentData<ChildType>> 
  implements ContainerRenderObjectMixin<ChildType, ParentDataType> {
  /// 返回带有基线的第一个孩子的基线。
  /// 当子元素以它们在子列表中出现的顺序垂直显示时非常有用。
  double defaultComputeDistanceToFirstActualBaseline(TextBaseline baseline) {
    ChildType child = firstChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      final double result = child.getDistanceToActualBaseline(baseline);
      if (result != null)
        return result + childParentData.offset.dy;
      child = childParentData.nextSibling;
    }
    return null;
  }

  /// 返回所有孩子的最小基线
  ///
  /// 当子列表的垂直位置不是由子列表中的顺序决定时，此选项非常有用。
  double defaultComputeDistanceToHighestActualBaseline(TextBaseline baseline) {
    double result;
    ChildType child = firstChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      double candidate = child.getDistanceToActualBaseline(baseline);
      if (candidate != null) {
        candidate += childParentData.offset.dy;
        if (result != null)
          result = math.min(result, candidate);
        else
          result = candidate;
      }
      child = childParentData.nextSibling;
    }
    return result;
  }

  /// Performs a hit test on each child by walking the child list backwards.
  ///
  /// Stops walking once after the first child reports that it contains the
  /// given point. Returns whether any children contain the given point.
  ///
  /// See also:
  ///
  ///  * [defaultPaint], which paints the children appropriate for this
  ///    hit-testing strategy.
  bool defaultHitTestChildren(BoxHitTestResult result, { Offset position }) {
    // the x, y parameters have the top left of the node's box as the origin
    ChildType child = lastChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      final bool isHit = result.addWithPaintOffset(
        offset: childParentData.offset,
        position: position,
        hitTest: (BoxHitTestResult result, Offset transformed) {
          return child.hitTest(result, position: transformed);
        },
      );
      if (isHit)
        return true;
      child = childParentData.previousSibling;
    }
    return false;
  }

  /// 默认绘制方法
  /// 以给定的 offset 来绘制每一个孩子     
  void defaultPaint(PaintingContext context, Offset offset) {
    ChildType child = firstChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      context.paintChild(child, childParentData.offset + offset);
      child = childParentData.nextSibling;
    }
  }

  /// 获取孩子列表
  List<ChildType> getChildrenAsList() {
    final List<ChildType> result = <ChildType>[];
    RenderBox child = firstChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      result.add(child);
      child = childParentData.nextSibling;
    }
    return result;
  }
}
```


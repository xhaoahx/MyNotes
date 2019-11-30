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
/// #### Using RenderObjectWithChildMixin
///
/// 如果一个渲染对象有一个单独的子节点，但是它不是一个[RenderBox]，那么可以使用
/// [RenderObjectWithChildMixin]，它是用于管理子对象的模板
///
/// It's a generic class with one type argument, the type of the child. For
/// example, if you are building a `RenderFoo` class which takes a single
/// `RenderBar` child, you would use the mixin as follows:
///
/// ```dart
/// class RenderFoo extends RenderBox
///   with RenderObjectWithChildMixin<RenderBar> {
///   // ...
/// }
/// ```
///
/// Since the `RenderFoo` class itself is still a [RenderBox] in this case, you
/// still have to implement the [RenderBox] layout algorithm, as well as
/// features like intrinsics and baselines, painting, and hit testing.
///
/// #### Using ContainerRenderObjectMixin
///
/// If a render box can have multiple children, then the
/// [ContainerRenderObjectMixin] mixin can be used to handle the boilerplate. It
/// uses a linked list to model the children in a manner that is easy to mutate
/// dynamically and that can be walked efficiently. Random access is not
/// efficient in this model; if you need random access to the children consider
/// the next section on more complicated child models.
///
/// The [ContainerRenderObjectMixin] class has two type arguments. The first is
/// the type of the child objects. The second is the type for their
/// [parentData]. The class used for [parentData] must itself have the
/// [ContainerParentDataMixin] class mixed into it; this is where
/// [ContainerRenderObjectMixin] stores the linked list. A [ParentData] class
/// can extend [ContainerBoxParentData]; this is essentially
/// [BoxParentData] mixed with [ContainerParentDataMixin]. For example, if a
/// `RenderFoo` class wanted to have a linked list of [RenderBox] children, one
/// might create a `FooParentData` class as follows:
///
/// ```dart
/// class FooParentData extends ContainerBoxParentData<RenderBox> {
///   // (any fields you might need for these children)
/// }
/// ```
///
/// When using [ContainerRenderObjectMixin] in a [RenderBox], consider mixing in
/// [RenderBoxContainerDefaultsMixin], which provides a collection of utility
/// methods that implement common parts of the [RenderBox] protocol (such as
/// painting the children).
///
/// The declaration of the `RenderFoo` class itself would thus look like this:
///
/// ```dart
/// class RenderFoo extends RenderBox with
///   ContainerRenderObjectMixin<RenderBox, FooParentData>,
///   RenderBoxContainerDefaultsMixin<RenderBox, FooParentData> {
///   // ...
/// }
/// ```
///
/// When walking the children (e.g. during layout), the following pattern is
/// commonly used (in this case assuming that the children are all [RenderBox]
/// objects and that this render object uses `FooParentData` objects for its
/// children's [parentData] fields):
///
/// ```dart
/// RenderBox child = firstChild;
/// while (child != null) {
///   final FooParentData childParentData = child.parentData;
///   // ...operate on child and childParentData...
///   assert(child.parentData == childParentData);
///   child = childParentData.nextSibling;
/// }
/// ```
///
/// #### More complicated child models
///
/// Render objects can have more complicated models, for example a map of
/// children keyed on an enum, or a 2D grid of efficiently randomly-accessible
/// children, or multiple lists of children, etc. If a render object has a model
/// that can't be handled by the mixins above, it must implement the
/// [RenderObject] child protocol, as follows:
///
/// * Any time a child is removed, call [dropChild] with the child.
///
/// * Any time a child is added, call [adoptChild] with the child.
///
/// * Implement the [attach] method such that it calls [attach] on each child.
///
/// * Implement the [detach] method such that it calls [detach] on each child.
///
/// * Implement the [redepthChildren] method such that it calls [redepthChild]
///   on each child.
///
/// * Implement the [visitChildren] method such that it calls its argument for
///   each child, typically in paint order (back-most to front-most).
///
/// * Implement [debugDescribeChildren] such that it outputs a [DiagnosticsNode]
///   for each child.
///
/// Implementing these seven bullet points is essentially all that the two
/// aforementioned mixins do.
///
/// ### Layout
///
/// [RenderBox] classes implement a layout algorithm. They have a set of
/// constraints provided to them, and they size themselves based on those
/// constraints and whatever other inputs they may have (for example, their
/// children or properties).
///
/// When implementing a [RenderBox] subclass, one must make a choice. Does it
/// size itself exclusively based on the constraints, or does it use any other
/// information in sizing itself? An example of sizing purely based on the
/// constraints would be growing to fit the parent.
///
/// Sizing purely based on the constraints allows the system to make some
/// significant optimizations. Classes that use this approach should override
/// [sizedByParent] to return true, and then override [performResize] to set the
/// [size] using nothing but the constraints, e.g.:
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
/// Otherwise, the size is set in the [performLayout] function.
///
/// The [performLayout] function is where render boxes decide, if they are not
/// [sizedByParent], what [size] they should be, and also where they decide
/// where their children should be.
///
/// #### Layout of RenderBox children
///
/// The [performLayout] function should call the [layout] function of each (box)
/// child, passing it a [BoxConstraints] object describing the constraints
/// within which the child can render. Passing tight constraints (see
/// [BoxConstraints.isTight]) to the child will allow the rendering library to
/// apply some optimizations, as it knows that if the constraints are tight, the
/// child's dimensions cannot change even if the layout of the child itself
/// changes.
///
/// If the [performLayout] function will use the child's size to affect other
/// aspects of the layout, for example if the render box sizes itself around the
/// child, or positions several children based on the size of those children,
/// then it must specify the `parentUsesSize` argument to the child's [layout]
/// function, setting it to true.
///
/// This flag turns off some optimizations; algorithms that do not rely on the
/// children's sizes will be more efficient. (In particular, relying on the
/// child's [size] means that if the child is marked dirty for layout, the
/// parent will probably also be marked dirty for layout, unless the
/// [constraints] given by the parent to the child were tight constraints.)
///
/// For [RenderBox] classes that do not inherit from [RenderProxyBox], once they
/// have laid out their children, they should also position them, by setting the
/// [BoxParentData.offset] field of each child's [parentData] object.
///
/// #### Layout of non-RenderBox children
///
/// The children of a [RenderBox] do not have to be [RenderBox]es themselves. If
/// they use another protocol (as discussed at [RenderObject]), then instead of
/// [BoxConstraints], the parent would pass in the appropriate [Constraints]
/// subclass, and instead of reading the child's size, the parent would read
/// whatever the output of [layout] is for that layout protocol. The
/// `parentUsesSize` flag is still used to indicate whether the parent is going
/// to read that output, and optimizations still kick in if the child has tight
/// constraints (as defined by [Constraints.isTight]).
///
/// ### Painting
///
/// To describe how a render box paints, implement the [paint] method. It is
/// given a [PaintingContext] object and an [Offset]. The painting context
/// provides methods to affect the layer tree as well as a
/// [PaintingContext.canvas] which can be used to add drawing commands. The
/// canvas object should not be cached across calls to the [PaintingContext]'s
/// methods; every time a method on [PaintingContext] is called, there is a
/// chance that the canvas will change identity. The offset specifies the
/// position of the top left corner of the box in the coordinate system of the
/// [PaintingContext.canvas].
///
/// To draw text on a canvas, use a [TextPainter].
///
/// To draw an image to a canvas, use the [paintImage] method.
///
/// A [RenderBox] that uses methods on [PaintingContext] that introduce new
/// layers should override the [alwaysNeedsCompositing] getter and set it to
/// true. If the object sometimes does and sometimes does not, it can have that
/// getter return true in some cases and false in others. In that case, whenever
/// the return value would change, call [markNeedsCompositingBitsUpdate]. (This
/// is done automatically when a child is added or removed, so you don't have to
/// call it explicitly if the [alwaysNeedsCompositing] getter only changes value
/// based on the presence or absence of children.)
///
/// Anytime anything changes on the object that would cause the [paint] method
/// to paint something different (but would not cause the layout to change),
/// the object should call [markNeedsPaint].
///
/// #### Painting children
///
/// The [paint] method's `context` argument has a [PaintingContext.paintChild]
/// method, which should be called for each child that is to be painted. It
/// should be given a reference to the child, and an [Offset] giving the
/// position of the child relative to the parent.
///
/// If the [paint] method applies a transform to the painting context before
/// painting children (or generally applies an additional offset beyond the
/// offset it was itself given as an argument), then the [applyPaintTransform]
/// method should also be overridden. That method must adjust the matrix that it
/// is given in the same manner as it transformed the painting context and
/// offset before painting the given child. This is used by the [globalToLocal]
/// and [localToGlobal] methods.
///
/// #### Hit Tests
///
/// Hit testing for render boxes is implemented by the [hitTest] method. The
/// default implementation of this method defers to [hitTestSelf] and
/// [hitTestChildren]. When implementing hit testing, you can either override
/// these latter two methods, or ignore them and just override [hitTest].
///
/// The [hitTest] method itself is given an [Offset], and must return true if the
/// object or one of its children has absorbed the hit (preventing objects below
/// this one from being hit), or false if the hit can continue to other objects
/// below this one.
///
/// For each child [RenderBox], the [hitTest] method on the child should be
/// called with the same [HitTestResult] argument and with the point transformed
/// into the child's coordinate space (in the same manner that the
/// [applyPaintTransform] method would). The default implementation defers to
/// [hitTestChildren] to call the children. [RenderBoxContainerDefaultsMixin]
/// provides a [RenderBoxContainerDefaultsMixin.defaultHitTestChildren] method
/// that does this assuming that the children are axis-aligned, not transformed,
/// and positioned according to the [BoxParentData.offset] field of the
/// [parentData]; more elaborate boxes can override [hitTestChildren]
/// accordingly.
///
/// If the object is hit, then it should also add itself to the [HitTestResult]
/// object that is given as an argument to the [hitTest] method, using
/// [HitTestResult.add]. The default implementation defers to [hitTestSelf] to
/// determine if the box is hit. If the object adds itself before the children
/// can add themselves, then it will be as if the object was above the children.
/// If it adds itself after the children, then it will be as if it was below the
/// children. Entries added to the [HitTestResult] object should use the
/// [BoxHitTestEntry] class. The entries are subsequently walked by the system
/// in the order they were added, and for each entry, the target's [handleEvent]
/// method is called, passing in the [HitTestEntry] object.
///
/// Hit testing cannot rely on painting having happened.
///
///
/// ### Intrinsics and Baselines
///
/// The layout, painting, hit testing, and semantics protocols are common to all
/// render objects. [RenderBox] objects must implement two additional protocols:
/// intrinsic sizing and baseline measurements.
///
/// There are four methods to implement for intrinsic sizing, to compute the
/// minimum and maximum intrinsic width and height of the box. The documentation
/// for these methods discusses the protocol in detail:
/// [computeMinIntrinsicWidth], [computeMaxIntrinsicWidth],
/// [computeMinIntrinsicHeight], [computeMaxIntrinsicHeight].
///
/// In addition, if the box has any children, it must implement
/// [computeDistanceToActualBaseline]. [RenderProxyBox] provides a simple
/// implementation that forwards to the child; [RenderShiftedBox] provides an
/// implementation that offsets the child's baseline information by the position
/// of the child relative to the parent. If you do not inherited from either of
/// these classes, however, you must implement the algorithm yourself.
abstract class RenderBox extends RenderObject {
  @override
  void setupParentData(covariant RenderObject child) {
    if (child.parentData is! BoxParentData)
      child.parentData = BoxParentData();
  }

  Map<_IntrinsicDimensionsCacheEntry, double> _cachedIntrinsicDimensions;

  double _computeIntrinsicDimension(_IntrinsicDimension dimension, double argument, double computer(double argument)) {
    assert(RenderObject.debugCheckingIntrinsics || !debugDoingThisResize); // performResize should not depend on anything except the incoming constraints
    bool shouldCache = true;
    assert(() {
      // we don't want the checked-mode intrinsic tests to affect
      // who gets marked dirty, etc.
      if (RenderObject.debugCheckingIntrinsics)
        shouldCache = false;
      return true;
    }());
    if (shouldCache) {
      _cachedIntrinsicDimensions ??= <_IntrinsicDimensionsCacheEntry, double>{};
      return _cachedIntrinsicDimensions.putIfAbsent(
        _IntrinsicDimensionsCacheEntry(dimension, argument),
        () => computer(argument),
      );
    }
    return computer(argument);
  }

  /// Returns the minimum width that this box could be without failing to
  /// correctly paint its contents within itself, without clipping.
  ///
  /// The height argument may give a specific height to assume. The given height
  /// can be infinite, meaning that the intrinsic width in an unconstrained
  /// environment is being requested. The given height should never be negative
  /// or null.
  ///
  /// This function should only be called on one's children. Calling this
  /// function couples the child with the parent so that when the child's layout
  /// changes, the parent is notified (via [markNeedsLayout]).
  ///
  /// Calling this function is expensive and as it can result in O(N^2)
  /// behavior.
  ///
  /// Do not override this method. Instead, implement [computeMinIntrinsicWidth].
  @mustCallSuper
  double getMinIntrinsicWidth(double height) {
    assert(() {
      if (height == null) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('The height argument to getMinIntrinsicWidth was null.'),
          ErrorDescription('The argument to getMinIntrinsicWidth must not be negative or null.'),
          ErrorHint('If you do not have a specific height in mind, then pass double.infinity instead.'),
        ]);
      }
      if (height < 0.0) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('The height argument to getMinIntrinsicWidth was negative.'),
          ErrorDescription('The argument to getMinIntrinsicWidth must not be negative or null.'),
          ErrorHint(
            'If you perform computations on another height before passing it to '
            'getMinIntrinsicWidth, consider using math.max() or double.clamp() '
            'to force the value into the valid range.'
          ),
        ]);
      }
      return true;
    }());
    return _computeIntrinsicDimension(_IntrinsicDimension.minWidth, height, computeMinIntrinsicWidth);
  }

  /// Computes the value returned by [getMinIntrinsicWidth]. Do not call this
  /// function directly, instead, call [getMinIntrinsicWidth].
  ///
  /// Override in subclasses that implement [performLayout]. This method should
  /// return the minimum width that this box could be without failing to
  /// correctly paint its contents within itself, without clipping.
  ///
  /// If the layout algorithm is independent of the context (e.g. it always
  /// tries to be a particular size), or if the layout algorithm is
  /// width-in-height-out, or if the layout algorithm uses both the incoming
  /// width and height constraints (e.g. it always sizes itself to
  /// [BoxConstraints.biggest]), then the `height` argument should be ignored.
  ///
  /// If the layout algorithm is strictly height-in-width-out, or is
  /// height-in-width-out when the width is unconstrained, then the height
  /// argument is the height to use.
  ///
  /// The `height` argument will never be negative or null. It may be infinite.
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

  /// Returns the smallest width beyond which increasing the width never
  /// decreases the preferred height. The preferred height is the value that
  /// would be returned by [getMinIntrinsicHeight] for that width.
  ///
  /// The height argument may give a specific height to assume. The given height
  /// can be infinite, meaning that the intrinsic width in an unconstrained
  /// environment is being requested. The given height should never be negative
  /// or null.
  ///
  /// This function should only be called on one's children. Calling this
  /// function couples the child with the parent so that when the child's layout
  /// changes, the parent is notified (via [markNeedsLayout]).
  ///
  /// Calling this function is expensive and as it can result in O(N^2)
  /// behavior.
  ///
  /// Do not override this method. Instead, implement
  /// [computeMaxIntrinsicWidth].
  @mustCallSuper
  double getMaxIntrinsicWidth(double height) {
    assert(() {
      if (height == null) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('The height argument to getMaxIntrinsicWidth was null.'),
          ErrorDescription('The argument to getMaxIntrinsicWidth must not be negative or null.'),
          ErrorHint('If you do not have a specific height in mind, then pass double.infinity instead.'),
        ]);
      }
      if (height < 0.0) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('The height argument to getMaxIntrinsicWidth was negative.'),
          ErrorDescription('The argument to getMaxIntrinsicWidth must not be negative or null.'),
          ErrorHint(
            'If you perform computations on another height before passing it to '
            'getMaxIntrinsicWidth, consider using math.max() or double.clamp() '
            'to force the value into the valid range.'
          ),
        ]);
      }
      return true;
    }());
    return _computeIntrinsicDimension(_IntrinsicDimension.maxWidth, height, computeMaxIntrinsicWidth);
  }

  /// Computes the value returned by [getMaxIntrinsicWidth]. Do not call this
  /// function directly, instead, call [getMaxIntrinsicWidth].
  ///
  /// Override in subclasses that implement [performLayout]. This should return
  /// the smallest width beyond which increasing the width never decreases the
  /// preferred height. The preferred height is the value that would be returned
  /// by [computeMinIntrinsicHeight] for that width.
  ///
  /// If the layout algorithm is strictly height-in-width-out, or is
  /// height-in-width-out when the width is unconstrained, then this should
  /// return the same value as [computeMinIntrinsicWidth] for the same height.
  ///
  /// Otherwise, the height argument should be ignored, and the returned value
  /// should be equal to or bigger than the value returned by
  /// [computeMinIntrinsicWidth].
  ///
  /// The `height` argument will never be negative or null. It may be infinite.
  ///
  /// The value returned by this method might not match the size that the object
  /// would actually take. For example, a [RenderBox] subclass that always
  /// exactly sizes itself using [BoxConstraints.biggest] might well size itself
  /// bigger than its max intrinsic size.
  ///
  /// If this algorithm depends on the intrinsic dimensions of a child, the
  /// intrinsic dimensions of that child should be obtained using the functions
  /// whose names start with `get`, not `compute`.
  ///
  /// This function should never return a negative or infinite value.
  ///
  /// See also examples in the definition of [computeMinIntrinsicWidth].
  @protected
  double computeMaxIntrinsicWidth(double height) {
    return 0.0;
  }

  /// Returns the minimum height that this box could be without failing to
  /// correctly paint its contents within itself, without clipping.
  ///
  /// The width argument may give a specific width to assume. The given width
  /// can be infinite, meaning that the intrinsic height in an unconstrained
  /// environment is being requested. The given width should never be negative
  /// or null.
  ///
  /// This function should only be called on one's children. Calling this
  /// function couples the child with the parent so that when the child's layout
  /// changes, the parent is notified (via [markNeedsLayout]).
  ///
  /// Calling this function is expensive and as it can result in O(N^2)
  /// behavior.
  ///
  /// Do not override this method. Instead, implement
  /// [computeMinIntrinsicHeight].
  @mustCallSuper
  double getMinIntrinsicHeight(double width) {
    assert(() {
      if (width == null) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('The width argument to getMinIntrinsicHeight was null.'),
          ErrorDescription('The argument to getMinIntrinsicHeight must not be negative or null.'),
          ErrorHint('If you do not have a specific width in mind, then pass double.infinity instead.'),
        ]);
      }
      if (width < 0.0) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('The width argument to getMinIntrinsicHeight was negative.'),
          ErrorDescription('The argument to getMinIntrinsicHeight must not be negative or null.'),
          ErrorHint(
            'If you perform computations on another width before passing it to '
            'getMinIntrinsicHeight, consider using math.max() or double.clamp() '
            'to force the value into the valid range.'
          ),
        ]);
      }
      return true;
    }());
    return _computeIntrinsicDimension(_IntrinsicDimension.minHeight, width, computeMinIntrinsicHeight);
  }

  /// Computes the value returned by [getMinIntrinsicHeight]. Do not call this
  /// function directly, instead, call [getMinIntrinsicHeight].
  ///
  /// Override in subclasses that implement [performLayout]. Should return the
  /// minimum height that this box could be without failing to correctly paint
  /// its contents within itself, without clipping.
  ///
  /// If the layout algorithm is independent of the context (e.g. it always
  /// tries to be a particular size), or if the layout algorithm is
  /// height-in-width-out, or if the layout algorithm uses both the incoming
  /// height and width constraints (e.g. it always sizes itself to
  /// [BoxConstraints.biggest]), then the `width` argument should be ignored.
  ///
  /// If the layout algorithm is strictly width-in-height-out, or is
  /// width-in-height-out when the height is unconstrained, then the width
  /// argument is the width to use.
  ///
  /// The `width` argument will never be negative or null. It may be infinite.
  ///
  /// If this algorithm depends on the intrinsic dimensions of a child, the
  /// intrinsic dimensions of that child should be obtained using the functions
  /// whose names start with `get`, not `compute`.
  ///
  /// This function should never return a negative or infinite value.
  ///
  /// See also examples in the definition of [computeMinIntrinsicWidth].
  @protected
  double computeMinIntrinsicHeight(double width) {
    return 0.0;
  }

  /// Returns the smallest height beyond which increasing the height never
  /// decreases the preferred width. The preferred width is the value that
  /// would be returned by [getMinIntrinsicWidth] for that height.
  ///
  /// The width argument may give a specific width to assume. The given width
  /// can be infinite, meaning that the intrinsic height in an unconstrained
  /// environment is being requested. The given width should never be negative
  /// or null.
  ///
  /// This function should only be called on one's children. Calling this
  /// function couples the child with the parent so that when the child's layout
  /// changes, the parent is notified (via [markNeedsLayout]).
  ///
  /// Calling this function is expensive and as it can result in O(N^2)
  /// behavior.
  ///
  /// Do not override this method. Instead, implement
  /// [computeMaxIntrinsicHeight].
  @mustCallSuper
  double getMaxIntrinsicHeight(double width) {
    assert(() {
      if (width == null) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('The width argument to getMaxIntrinsicHeight was null.'),
          ErrorDescription('The argument to getMaxIntrinsicHeight must not be negative or null.'),
          ErrorHint('If you do not have a specific width in mind, then pass double.infinity instead.'),
        ]);
      }
      if (width < 0.0) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('The width argument to getMaxIntrinsicHeight was negative.'),
          ErrorDescription('The argument to getMaxIntrinsicHeight must not be negative or null.'),
          ErrorHint(
            'If you perform computations on another width before passing it to '
            'getMaxIntrinsicHeight, consider using math.max() or double.clamp() '
            'to force the value into the valid range.'
          ),
        ]);
      }
      return true;
    }());
    return _computeIntrinsicDimension(_IntrinsicDimension.maxHeight, width, computeMaxIntrinsicHeight);
  }

  /// Computes the value returned by [getMaxIntrinsicHeight]. Do not call this
  /// function directly, instead, call [getMaxIntrinsicHeight].
  ///
  /// Override in subclasses that implement [performLayout]. Should return the
  /// smallest height beyond which increasing the height never decreases the
  /// preferred width. The preferred width is the value that would be returned
  /// by [computeMinIntrinsicWidth] for that height.
  ///
  /// If the layout algorithm is strictly width-in-height-out, or is
  /// width-in-height-out when the height is unconstrained, then this should
  /// return the same value as [computeMinIntrinsicHeight] for the same width.
  ///
  /// Otherwise, the width argument should be ignored, and the returned value
  /// should be equal to or bigger than the value returned by
  /// [computeMinIntrinsicHeight].
  ///
  /// The `width` argument will never be negative or null. It may be infinite.
  ///
  /// The value returned by this method might not match the size that the object
  /// would actually take. For example, a [RenderBox] subclass that always
  /// exactly sizes itself using [BoxConstraints.biggest] might well size itself
  /// bigger than its max intrinsic size.
  ///
  /// If this algorithm depends on the intrinsic dimensions of a child, the
  /// intrinsic dimensions of that child should be obtained using the functions
  /// whose names start with `get`, not `compute`.
  ///
  /// This function should never return a negative or infinite value.
  ///
  /// See also examples in the definition of [computeMinIntrinsicWidth].
  @protected
  double computeMaxIntrinsicHeight(double width) {
    return 0.0;
  }

  /// Whether this render object has undergone layout and has a [size].
  bool get hasSize => _size != null;

  /// The size of this render box computed during layout.
  ///
  /// This value is stale whenever this object is marked as needing layout.
  /// During [performLayout], do not read the size of a child unless you pass
  /// true for parentUsesSize when calling the child's [layout] function.
  ///
  /// The size of a box should be set only during the box's [performLayout] or
  /// [performResize] functions. If you wish to change the size of a box outside
  /// of those functions, call [markNeedsLayout] instead to schedule a layout of
  /// the box.
  Size get size {
    assert(hasSize, 'RenderBox was not laid out: ${toString()}');
    assert(() {
      if (_size is _DebugSize) {
        final _DebugSize _size = this._size;
        assert(_size._owner == this);
        if (RenderObject.debugActiveLayout != null) {
          // We are always allowed to access our own size (for print debugging
          // and asserts if nothing else). Other than us, the only object that's
          // allowed to read our size is our parent, if they've said they will.
          // If you hit this assert trying to access a child's size, pass
          // "parentUsesSize: true" to that child's layout().
          assert(debugDoingThisResize || debugDoingThisLayout ||
                 (RenderObject.debugActiveLayout == parent && _size._canBeUsedByParent));
        }
        assert(_size == this._size);
      }
      return true;
    }());
    return _size;
  }
  Size _size;
  /// Setting the size, in checked mode, triggers some analysis of the render box,
  /// as implemented by [debugAssertDoesMeetConstraints], including calling the intrinsic
  /// sizing methods and checking that they meet certain invariants.
  @protected
  set size(Size value) {
    assert(!(debugDoingThisResize && debugDoingThisLayout));
    assert(sizedByParent || !debugDoingThisResize);
    assert(() {
      if ((sizedByParent && debugDoingThisResize) ||
          (!sizedByParent && debugDoingThisLayout))
        return true;
      assert(!debugDoingThisResize);
      final List<DiagnosticsNode> information = <DiagnosticsNode>[
        ErrorSummary('RenderBox size setter called incorrectly.'),
      ];
      if (debugDoingThisLayout) {
        assert(sizedByParent);
        information.add(ErrorDescription('It appears that the size setter was called from performLayout().'));
      } else {
        information.add(ErrorDescription(
          'The size setter was called from outside layout (neither performResize() nor performLayout() were being run for this object).'
        ));
        if (owner != null && owner.debugDoingLayout)
          information.add(ErrorDescription('Only the object itself can set its size. It is a contract violation for other objects to set it.'));
      }
      if (sizedByParent)
        information.add(ErrorDescription('Because this RenderBox has sizedByParent set to true, it must set its size in performResize().'));
      else
        information.add(ErrorDescription('Because this RenderBox has sizedByParent set to false, it must set its size in performLayout().'));
      throw FlutterError.fromParts(information);
    }());
    assert(() {
      value = debugAdoptSize(value);
      return true;
    }());
    _size = value;
    assert(() {
      debugAssertDoesMeetConstraints();
      return true;
    }());
  }

  @override
  Rect get semanticBounds => Offset.zero & size;

  Map<TextBaseline, double> _cachedBaselines;
  static bool _debugDoingBaseline = false;
  static bool _debugSetDoingBaseline(bool value) {
    _debugDoingBaseline = value;
    return true;
  }

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
    assert(!_debugDoingBaseline, 'Please see the documentation for computeDistanceToActualBaseline for the required calling conventions of this method.');
    assert(!debugNeedsLayout);
    assert(() {
      final RenderObject parent = this.parent;
      if (owner.debugDoingLayout)
        return (RenderObject.debugActiveLayout == parent) && parent.debugDoingThisLayout;
      if (owner.debugDoingPaint)
        return ((RenderObject.debugActivePaint == parent) && parent.debugDoingThisPaint) ||
               ((RenderObject.debugActivePaint == this) && debugDoingThisPaint);
      assert(parent == this.parent);
      return false;
    }());
    assert(_debugSetDoingBaseline(true));
    final double result = getDistanceToActualBaseline(baseline);
    assert(_debugSetDoingBaseline(false));
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
    assert(_debugDoingBaseline, 'Please see the documentation for computeDistanceToActualBaseline for the required calling conventions of this method.');
    _cachedBaselines ??= <TextBaseline, double>{};
    _cachedBaselines.putIfAbsent(baseline, () => computeDistanceToActualBaseline(baseline));
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
    assert(_debugDoingBaseline, 'Please see the documentation for computeDistanceToActualBaseline for the required calling conventions of this method.');
    return null;
  }

  /// The box constraints most recently received from the parent.
  @override
  BoxConstraints get constraints => super.constraints;

  @override
  void markNeedsLayout() {
    if ((_cachedBaselines != null && _cachedBaselines.isNotEmpty) ||
        (_cachedIntrinsicDimensions != null && _cachedIntrinsicDimensions.isNotEmpty)) {
      // If we have cached data, then someone must have used our data.
      // Since the parent will shortly be marked dirty, we can forget that they
      // used the baseline and/or intrinsic dimensions. If they use them again,
      // then we'll fill the cache again, and if we get dirty again, we'll
      // notify them again.
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
mixin RenderBoxContainerDefaultsMixin
  <ChildType extends RenderBox, 
   ParentDataType extends ContainerBoxParentData<ChildType>> 
  implements ContainerRenderObjectMixin<ChildType, ParentDataType> {
  /// Returns the baseline of the first child with a baseline.
  ///
  /// Useful when the children are displayed vertically in the same order they
  /// appear in the child list.
  double defaultComputeDistanceToFirstActualBaseline(TextBaseline baseline) {
    assert(!debugNeedsLayout);
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

  /// Returns the minimum baseline value among every child.
  ///
  /// Useful when the vertical position of the children isn't determined by the
  /// order in the child list.
  double defaultComputeDistanceToHighestActualBaseline(TextBaseline baseline) {
    assert(!debugNeedsLayout);
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
          assert(transformed == position - childParentData.offset);
          return child.hitTest(result, position: transformed);
        },
      );
      if (isHit)
        return true;
      child = childParentData.previousSibling;
    }
    return false;
  }

  /// Paints each child by walking the child list forwards.
  ///
  /// See also:
  ///
  ///  * [defaultHitTestChildren], which implements hit-testing of the children
  ///    in a manner appropriate for this painting strategy.
  void defaultPaint(PaintingContext context, Offset offset) {
    ChildType child = firstChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      context.paintChild(child, childParentData.offset + offset);
      child = childParentData.nextSibling;
    }
  }

  /// Returns a list containing the children of this render object.
  ///
  /// This function is useful when you need random-access to the children of
  /// this render object. If you're accessing the children in order, consider
  /// walking the child list directly.
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


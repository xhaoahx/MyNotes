# Flutter RenderObject 源码分析

[TOC]

## AbstractNode

```dart
/// 抽象节点
class AbstractNode {
  /// 深度
  int get depth => _depth;
  int _depth = 0;

  /// 重新分配孩子深度
  @protected
  void redepthChild(AbstractNode child) {
    if (child._depth <= _depth) {
      child._depth = _depth + 1;
      child.redepthChildren();
    }
  }

  void redepthChildren() { }

  /// 这个节点的持有者（如果没有attach，则为null）
  /// 整棵树只有一个持有者
  Object get owner => _owner;
  Object _owner;

  /// 是否atatched
  bool get attached => _owner != null;

  /// 为这个节点 attach 持有者
  @mustCallSuper
  void attach(covariant Object owner) {
    assert(owner != null);
    assert(_owner == null);
    _owner = owner;
  }

  /// detach 这个节点的持有者
  @mustCallSuper
  void detach() {
    assert(_owner != null);
    _owner = null;
    assert(parent == null || attached == parent.attached);
  }

  /// 父节点
  AbstractNode get parent => _parent;
  AbstractNode _parent;

  /// 把给定的节点作为自己的孩子
  @protected
  @mustCallSuper
  void adoptChild(covariant AbstractNode child) {
    child._parent = this;
    if (attached)
      child.attach(_owner);
    redepthChild(child);
  }

  /// 移除一个已有的孩子
  @protected
  @mustCallSuper
  void dropChild(covariant AbstractNode child) {
    child._parent = null;
    if (attached)
      child.detach();
  }
}
```

## PaintContext

```dart
/// 一个用于绘制的上下文
///
/// [RenderObject]的绘制使用一个[PaintingContext]，而不是直接持有[Canvas]。
/// PaintingContext 有一个[Canvas]，它接收单独的绘制操作，还具有绘画子渲染对象的函数。
///
/// 绘制子渲染对象时，由绘制上下文所保持的画布可能会改变。
/// 因为在绘制子渲染对象之前和之后发出的绘制操作可能被记录在独立（不同的）的合成层中。因此，不要在绘制子呈
/// 现对象的操作之间持有对画布的引用。
///
/// 使用[PaintingContext.repaintCompositedChild]和[pushLayer]时会自动创建新的
/// [PaintingContext]对象。
class PaintingContext extends ClipContext {

  @protected
  PaintingContext(this._containerLayer, this.estimatedBounds);

  final ContainerLayer _containerLayer;

  /// 一个预测的 Rect，[canvas]将会在这个范围内记录绘制操作
  ///
  /// 可以在画布之外进行操作
  ///
  /// [estimatedBounds] 处于 [canvas] 的坐标系内
  final Rect estimatedBounds;

  /// 重绘一个给定的渲染对象
  ///
  /// 给定的渲染对象必须挂载了一个[PipelineOwner]，必须有一个合成层，并且必须需要绘制。
  /// 渲染对象的层(如果有的话)和子树中不需要重新绘制的任何层一起被重用。
  ///
  static void repaintCompositedChild(
      RenderObject child, {
      bool debugAlsoPaintedParent = false 
  }) {
    _repaintCompositedChild(
      child,
      debugAlsoPaintedParent: debugAlsoPaintedParent,
    );
  }

  static void _repaintCompositedChild(
    RenderObject child, {
    bool debugAlsoPaintedParent = false,
    PaintingContext childContext,
  }) {
    OffsetLayer childLayer = child._layer;
    if (childLayer == null) {
      child._layer = childLayer = OffsetLayer();
    } else {
      childLayer.removeAllChildren();
    }
    /// 如果 childContext 为 null，则会新建一个 PaintingContext  
    childContext ??= PaintingContext(child._layer, child.paintBounds);
    child._paintWithContext(childContext, Offset.zero);
    childContext.stopRecordingIfNeeded();
  }

  /// Paint a child [RenderObject].
  ///
  /// 如果子节点有自己的复合层，则子节点将被复合到与该关联的层子树中。
  /// 否则，child 将被绘制到当前[PaintContext]的 PictureLayer中。
  void paintChild(RenderObject child, Offset offset) {

    if (child.isRepaintBoundary) {
      // 停止绘制操作的录制  
      stopRecordingIfNeeded();
      // 开始合成  
      _compositeChild(child, offset);
    } else {
      child._paintWithContext(this, offset);
    }
  }

  void _compositeChild(RenderObject child, Offset offset) {

    // Create a layer for our child, and paint the child into it.
    if (child._needsPaint) {
      repaintCompositedChild(child, debugAlsoPaintedParent: true);
    } else {
      assert(...); 
    }
    final OffsetLayer childOffsetLayer = child._layer;
    childOffsetLayer.offset = offset;
    appendLayer(child._layer);
  }

  /// Adds a layer to the recording requiring that the recording is already
  /// stopped.
  ///
  /// Do not call this function directly: call [addLayer] or [pushLayer]
  /// instead. This function is called internally when all layers not
  /// generated from the [canvas] are added.
  ///
  /// Subclasses that need to customize how layers are added should override
  /// this method.
  @protected
  void appendLayer(Layer layer) {
    assert(!_isRecording);
    layer.remove();
    _containerLayer.append(layer);
  }

  bool get _isRecording {
    final bool hasCanvas = _canvas != null;
    return hasCanvas;
  }

  // 绘制操作的录制状态
  PictureLayer _currentLayer;
  ui.PictureRecorder _recorder;
  Canvas _canvas;

  /// 用于绘制的 canvas
  ///
  /// 当使用这个[PaintContext]绘制一个孩子时，当前画布可能更改，这意味着保存此getter返回的 Canvas 的
  /// 引用是很脆弱的。
  @override
  Canvas get canvas {
    if (_canvas == null)
      _startRecording();
    return _canvas;
  }

  void _startRecording() {
    _currentLayer = PictureLayer(estimatedBounds);
    _recorder = ui.PictureRecorder();
    _canvas = Canvas(_recorder);
    _containerLayer.append(_currentLayer);
  }

  /// Stop recording to a canvas if recording has started.
  ///
  /// Do not call this function directly: functions in this class will call
  /// this method as needed. This function is called internally to ensure that
  /// recording is stopped before adding layers or finalizing the results of a
  /// paint.
  ///
  /// Subclasses that need to customize how recording to a canvas is performed
  /// should override this method to save the results of the custom canvas
  /// recordings.
  @protected
  @mustCallSuper
  void stopRecordingIfNeeded() {
    if (!_isRecording)
      return;
    _currentLayer.picture = _recorder.endRecording();
    _currentLayer = null;
    _recorder = null;
    _canvas = null;
  }

  /// Hints that the painting in the current layer is complex and would benefit
  /// from caching.
  ///
  /// If this hint is not set, the compositor will apply its own heuristics to
  /// decide whether the current layer is complex enough to benefit from
  /// caching.
  void setIsComplexHint() {
    _currentLayer?.isComplexHint = true;
  }

  /// Hints that the painting in the current layer is likely to change next frame.
  ///
  /// This hint tells the compositor not to cache the current layer because the
  /// cache will not be used in the future. If this hint is not set, the
  /// compositor will apply its own heuristics to decide whether the current
  /// layer is likely to be reused in the future.
  void setWillChangeHint() {
    _currentLayer?.willChangeHint = true;
  }

  /// Adds a composited leaf layer to the recording.
  ///
  /// After calling this function, the [canvas] property will change to refer to
  /// a new [Canvas] that draws on top of the given layer.
  ///
  /// A [RenderObject] that uses this function is very likely to require its
  /// [RenderObject.alwaysNeedsCompositing] property to return true. That informs
  /// ancestor render objects that this render object will include a composited
  /// layer, which, for example, causes them to use composited clips.
  ///
  /// See also:
  ///
  ///  * [pushLayer], for adding a layer and using its canvas to paint with that
  ///    layer.
  void addLayer(Layer layer) {
    stopRecordingIfNeeded();
    appendLayer(layer);
  }

  /// Appends the given layer to the recording, and calls the `painter` callback
  /// with that layer, providing the `childPaintBounds` as the estimated paint
  /// bounds of the child. The `childPaintBounds` can be used for debugging but
  /// have no effect on painting.
  ///
  /// The given layer must be an unattached orphan. (Providing a newly created
  /// object, rather than reusing an existing layer, satisfies that
  /// requirement.)
  ///
  /// The `offset` is the offset to pass to the `painter`.
  ///
  /// If the `childPaintBounds` are not specified then the current layer's paint
  /// bounds are used. This is appropriate if the child layer does not apply any
  /// transformation or clipping to its contents. The `childPaintBounds`, if
  /// specified, must be in the coordinate system of the new layer, and should
  /// not go outside the current layer's paint bounds.
  ///
  /// See also:
  ///
  ///  * [addLayer], for pushing a leaf layer whose canvas is not used.
  void pushLayer(ContainerLayer childLayer, PaintingContextCallback painter, Offset offset, { Rect childPaintBounds }) {
    assert(painter != null);
    // If a layer is being reused, it may already contain children. We remove
    // them so that `painter` can add children that are relevant for this frame.
    if (childLayer.hasChildren) {
      childLayer.removeAllChildren();
    }
    stopRecordingIfNeeded();
    appendLayer(childLayer);
    final PaintingContext childContext = createChildContext(childLayer, childPaintBounds ?? estimatedBounds);
    painter(childContext, offset);
    childContext.stopRecordingIfNeeded();
  }

  /// Creates a compatible painting context to paint onto [childLayer].
  @protected
  PaintingContext createChildContext(ContainerLayer childLayer, Rect bounds) {
    return PaintingContext(childLayer, bounds);
  }

  /// Clip further painting using a rectangle.
  ///
  /// {@template flutter.rendering.object.needsCompositing}
  /// * `needsCompositing` is whether the child needs compositing. Typically
  ///   matches the value of [RenderObject.needsCompositing] for the caller. If
  ///   false, this method returns null, indicating that a layer is no longer
  ///   necessary. If a render object calling this method stores the `oldLayer`
  ///   in its [RenderObject.layer] field, it should set that field to null.
  /// {@end template}
  /// * `offset` is the offset from the origin of the canvas' coordinate system
  ///   to the origin of the caller's coordinate system.
  /// * `clipRect` is rectangle (in the caller's coordinate system) to use to
  ///   clip the painting done by [painter].
  /// * `painter` is a callback that will paint with the [clipRect] applied. This
  ///   function calls the [painter] synchronously.
  /// * `clipBehavior` controls how the rectangle is clipped.
  /// {@template flutter.rendering.object.oldLayer}
  /// * `oldLayer` is the layer created in the previous frame. Specifying the
  ///   old layer gives the engine more information for performance
  ///   optimizations. Typically this is the value of [RenderObject.layer] that
  ///   a render object creates once, then reuses for all subsequent frames
  ///   until a layer is no longer needed (e.g. the render object no longer
  ///   needs compositing) or until the render object changes the type of the
  ///   layer (e.g. from opacity layer to a clip rect layer).
  /// {@end template}
  ClipRectLayer pushClipRect(bool needsCompositing, Offset offset, Rect clipRect, PaintingContextCallback painter, { Clip clipBehavior = Clip.hardEdge, ClipRectLayer oldLayer }) {
    final Rect offsetClipRect = clipRect.shift(offset);
    if (needsCompositing) {
      final ClipRectLayer layer = oldLayer ?? ClipRectLayer();
      layer
        ..clipRect = offsetClipRect
        ..clipBehavior = clipBehavior;
      pushLayer(layer, painter, offset, childPaintBounds: offsetClipRect);
      return layer;
    } else {
      clipRectAndPaint(offsetClipRect, clipBehavior, offsetClipRect, () => painter(this, offset));
      return null;
    }
  }

  /// Clip further painting using a rounded rectangle.
  ///
  /// {@macro flutter.rendering.object.needsCompositing}
  /// * `offset` is the offset from the origin of the canvas' coordinate system
  ///   to the origin of the caller's coordinate system.
  /// * `bounds` is the region of the canvas (in the caller's coordinate system)
  ///   into which `painter` will paint in.
  /// * `clipRRect` is the rounded-rectangle (in the caller's coordinate system)
  ///   to use to clip the painting done by `painter`.
  /// * `painter` is a callback that will paint with the `clipRRect` applied. This
  ///   function calls the `painter` synchronously.
  /// * `clipBehavior` controls how the path is clipped.
  /// {@macro flutter.rendering.object.oldLayer}
  ClipRRectLayer pushClipRRect(bool needsCompositing, Offset offset, Rect bounds, RRect clipRRect, PaintingContextCallback painter, { Clip clipBehavior = Clip.antiAlias, ClipRRectLayer oldLayer }) {
    assert(clipBehavior != null);
    final Rect offsetBounds = bounds.shift(offset);
    final RRect offsetClipRRect = clipRRect.shift(offset);
    if (needsCompositing) {
      final ClipRRectLayer layer = oldLayer ?? ClipRRectLayer();
      layer
        ..clipRRect = offsetClipRRect
        ..clipBehavior = clipBehavior;
      pushLayer(layer, painter, offset, childPaintBounds: offsetBounds);
      return layer;
    } else {
      clipRRectAndPaint(offsetClipRRect, clipBehavior, offsetBounds, () => painter(this, offset));
      return null;
    }
  }

  /// Clip further painting using a path.
  ///
  /// {@macro flutter.rendering.object.needsCompositing}
  /// * `offset` is the offset from the origin of the canvas' coordinate system
  ///   to the origin of the caller's coordinate system.
  /// * `bounds` is the region of the canvas (in the caller's coordinate system)
  ///   into which `painter` will paint in.
  /// * `clipPath` is the path (in the coordinate system of the caller) to use to
  ///   clip the painting done by `painter`.
  /// * `painter` is a callback that will paint with the `clipPath` applied. This
  ///   function calls the `painter` synchronously.
  /// * `clipBehavior` controls how the rounded rectangle is clipped.
  /// {@macro flutter.rendering.object.oldLayer}
  ClipPathLayer pushClipPath(bool needsCompositing, Offset offset, Rect bounds, Path clipPath, PaintingContextCallback painter, { Clip clipBehavior = Clip.antiAlias, ClipPathLayer oldLayer }) {
    assert(clipBehavior != null);
    final Rect offsetBounds = bounds.shift(offset);
    final Path offsetClipPath = clipPath.shift(offset);
    if (needsCompositing) {
      final ClipPathLayer layer = oldLayer ?? ClipPathLayer();
      layer
        ..clipPath = offsetClipPath
        ..clipBehavior = clipBehavior;
      pushLayer(layer, painter, offset, childPaintBounds: offsetBounds);
      return layer;
    } else {
      clipPathAndPaint(offsetClipPath, clipBehavior, offsetBounds, () => painter(this, offset));
      return null;
    }
  }

  /// Blend further painting with a color filter.
  ///
  /// * `offset` is the offset from the origin of the canvas' coordinate system
  ///   to the origin of the caller's coordinate system.
  /// * `colorFilter` is the [ColorFilter] value to use when blending the
  ///   painting done by `painter`.
  /// * `painter` is a callback that will paint with the `colorFilter` applied.
  ///   This function calls the `painter` synchronously.
  /// {@macro flutter.rendering.object.oldLayer}
  ///
  /// A [RenderObject] that uses this function is very likely to require its
  /// [RenderObject.alwaysNeedsCompositing] property to return true. That informs
  /// ancestor render objects that this render object will include a composited
  /// layer, which, for example, causes them to use composited clips.
  ColorFilterLayer pushColorFilter(Offset offset, ColorFilter colorFilter, PaintingContextCallback painter, { ColorFilterLayer oldLayer }) {
    assert(colorFilter != null);
    final ColorFilterLayer layer = oldLayer ?? ColorFilterLayer();
    layer.colorFilter = colorFilter;
    pushLayer(layer, painter, offset);
    return layer;
  }

  /// Transform further painting using a matrix.
  ///
  /// {@macro flutter.rendering.object.needsCompositing}
  /// * `offset` is the offset from the origin of the canvas' coordinate system
  ///   to the origin of the caller's coordinate system.
  /// * `transform` is the matrix to apply to the painting done by `painter`.
  /// * `painter` is a callback that will paint with the `transform` applied. This
  ///   function calls the `painter` synchronously.
  /// {@macro flutter.rendering.object.oldLayer}
  TransformLayer pushTransform(bool needsCompositing, Offset offset, Matrix4 transform, PaintingContextCallback painter, { TransformLayer oldLayer }) {
    final Matrix4 effectiveTransform = Matrix4.translationValues(offset.dx, offset.dy, 0.0)
      ..multiply(transform)..translate(-offset.dx, -offset.dy);
    if (needsCompositing) {
      final TransformLayer layer = oldLayer ?? TransformLayer();
      layer.transform = effectiveTransform;
      pushLayer(
        layer,
        painter,
        offset,
        childPaintBounds: MatrixUtils.inverseTransformRect(effectiveTransform, estimatedBounds),
      );
      return layer;
    } else {
      canvas
        ..save()
        ..transform(effectiveTransform.storage);
      painter(this, offset);
      canvas
        ..restore();
      return null;
    }
  }

  /// Blend further painting with an alpha value.
  ///
  /// * `offset` is the offset from the origin of the canvas' coordinate system
  ///   to the origin of the caller's coordinate system.
  /// * `alpha` is the alpha value to use when blending the painting done by
  ///   `painter`. An alpha value of 0 means the painting is fully transparent
  ///   and an alpha value of 255 means the painting is fully opaque.
  /// * `painter` is a callback that will paint with the `alpha` applied. This
  ///   function calls the `painter` synchronously.
  /// {@macro flutter.rendering.object.oldLayer}
  ///
  /// A [RenderObject] that uses this function is very likely to require its
  /// [RenderObject.alwaysNeedsCompositing] property to return true. That informs
  /// ancestor render objects that this render object will include a composited
  /// layer, which, for example, causes them to use composited clips.
  OpacityLayer pushOpacity(Offset offset, int alpha, PaintingContextCallback painter, { OpacityLayer oldLayer }) {
    final OpacityLayer layer = oldLayer ?? OpacityLayer();
    layer
      ..alpha = alpha
      ..offset = offset;
    pushLayer(layer, painter, Offset.zero);
    return layer;
  }
}
```



## RenderObject

```dart
abstract class RenderObject 
    extends AbstractNode 
    with DiagnosticableTreeMixin 
    implements HitTestTarget 
{
  /// 初始化域
  RenderObject() {
    _needsCompositing = isRepaintBoundary || alwaysNeedsCompositing;
  }

 /// 热重载的调用方法（这个方法十分昂贵，只在开发时调用）
  void reassemble() {
    markNeedsLayout();
    markNeedsCompositingBitsUpdate();
    markNeedsPaint();
    markNeedsSemanticsUpdate();
    visitChildren((RenderObject child) {
      child.reassemble();
    });
  }
```

### Layout
```dart
  // LAYOUT 布局

  /// 为孩子设置有用的信息，仅通过父节点的 setupParentData 方法设置，而不是直接修改
  ParentData parentData;

  /// 可以在子 RenderObject 还未加入到子元素列表时，为它附加 parentData
  void setupParentData(covariant RenderObject child) {
    if (child.parentData is! ParentData)
      child.parentData = ParentData();
  }

  /// 将一个 RenderObejct 设置为孩子，会触发重新布局
  @override
  void adoptChild(RenderObject child) {
    setupParentData(child);
    /// 需要重新布局  
    markNeedsLayout();
    /// 需要合成   
    markNeedsCompositingBitsUpdate();
    markNeedsSemanticsUpdate();
    super.adoptChild(child);
  }

  /// 移除一个孩子，会触发重新布局
  @override
  void dropChild(RenderObject child) {
    child._cleanRelayoutBoundary();
    child.parentData.detach();
    child.parentData = null;
    super.dropChild(child);
    markNeedsLayout();
    markNeedsCompositingBitsUpdate();
    markNeedsSemanticsUpdate();
  }

  /// 访问孩子
  void visitChildren(RenderObjectVisitor visitor) { }
	
  ...  

  @override
  PipelineOwner get owner => super.owner;

  /// 分配 PipelineOwner
  @override
  void attach(PipelineOwner owner) {
    super.attach(owner);
    // If the node was dirtied in some way while unattached, make sure to add
    // it to the appropriate dirty list now that an owner is available
    if (_needsLayout && _relayoutBoundary != null) {
      // Don't enter this block if we've never laid out at all;
      // scheduleInitialLayout() will handle it
      _needsLayout = false;
      markNeedsLayout();
    }
    if (_needsCompositingBitsUpdate) {
      _needsCompositingBitsUpdate = false;
      markNeedsCompositingBitsUpdate();
    }
    if (_needsPaint && _layer != null) {
      // Don't enter this block if we've never painted at all;
      // scheduleInitialPaint() will handle it
      _needsPaint = false;
      markNeedsPaint();
    }
    if (_needsSemanticsUpdate && _semanticsConfiguration.isSemanticBoundary) {
      // Don't enter this block if we've never updated semantics at all;
      // scheduleInitialSemantics() will handle it
      _needsSemanticsUpdate = false;
      markNeedsSemanticsUpdate();
    }
  }

  /// 是否需要重新布局（在实例化 RenderObject 的时候初始化为 true，这意味着每个RenderObject attach
  /// 的时候都需要重新布局）  
  bool _needsLayout = true;
  /// 这个 RenderObject 的重绘边界
  /// 注意 _relayoutBoundary 是一个 Render Object，子树具有相同的 _relayoutBoundary
  RenderObject _relayoutBoundary;
  bool _doingThisLayoutWithCallback = false;

  /// 父节点给子节点的约束
  @protected
  Constraints get constraints => _constraints;
  Constraints _constraints;
    
  /// 将这个渲染对象标记为 _needsLayout，并在 PipelineOwner 中注册（或延迟给父节点），
  /// 这取决于这个节点是否是 RelayoutBoundary
  ///
  /// 我们没有急切地响应写入 渲染对象 的更新布局信息，而是将布局信息标记为dirty，这将调度一个视觉更新。
  /// 作为视觉更新的一部分，PipelineOwner 将会更新渲染对象的布局信息
  /// 
  /// 该机制批量处理布局工作，以便将多个相续写入的信息合并在一起，消除冗余计算。
  /// （这是一种延迟更新机制，在调用 markNeedsLayout，PipelineOwner 不会立即更新需要布局的渲染对象，
  /// 而是等到下一个 vsync 信号到来的时候，engine 会调用 drawFrame 方法，进而调用到 
  /// PipelineOwner.flushLayout 方法完成布局或者视觉更新。在此期间，可能会有更多的渲染对象标记
  /// dirty，他们会一并被更新，而不是独立更新。）

  /// 如果一个渲染对象的父节点在计算它的布局信息时表明它依赖于([parentUseSize] == true)它的一个子节点
  /// 的大小，那么当调用这个子对象的[markNeedsLayout]时，这个函数也将标记父对象需要布局。
  /// 在这种情况下，由于父节点和子节点都需要重新计算它们的布局，所以 PipelineOwner 只会通知父节点需要重
  /// 新被布局.当父节点布局时，它的[performLayout]将调用子节点的[layout]方法，因此子节点也将被布局。

  /// [RenderObject]的一些子类，特别是[RenderBox]，在其他情况下，如果子节点标记了 dirty，需要通知父 
  /// 父节点(例如，如果子元素的固有维度或基线发生了变化)。
  /// 这样的子类重载了 markNeedsLayout，在正常情况下调用'super.markNeedsLayout()'，或者在需要
  /// 父节点和子节点同时需要布局的情况下调用[markParentNeedsLayout]。
    
  /// 如果[sizedByParent]发生改变，调用
  /// [markNeedsLayoutForSizedByParentChange]而不是[markNeedsLayout].
    
  void markNeedsLayout() {
    /// 如果已经标记，返回
    if (_needsLayout) {
      return;
    }
    /// 如果自身不是 _relayoutBoundary，则调用 markParentNeedsLayout()，
    /// 其中调用了parent.markNeedsLayout()  
    if (_relayoutBoundary != this) {
      markParentNeedsLayout();
    } else {
    /// 否则，在 PipelineOwne 注册，在 _nodesNeedingLayout 加入自己
    /// 并请求一次视觉更新，之后 engine 会在下一个 vsync 到来时回调 dramFrame,完成布局   
      _needsLayout = true;
      if (owner != null) {
        owner._nodesNeedingLayout.add(this);
        owner.requestVisualUpdate();
      }
    }
  }

  /// 标记父节点需要布局
  /// 这个函数只能从子类的[markNeedsLayout]或[markNeedsLayoutForSizedByParentChange]实现中调
  /// 用，这些实现引入了更多的原因来延迟对脏布局的处理给父节点
  /// 
  /// 标记自身 _needsLayout = true，并调用父节点的 markNeedsLayout
  /// 当父节点的 performLayout被调用的时候，会调用到子节点的 layout
  @protected
  void markParentNeedsLayout() {
    _needsLayout = true;
    final RenderObject parent = this.parent;
    if (!_doingThisLayoutWithCallback) {
      parent.markNeedsLayout();
    } else {
      assert(parent._debugDoingThisLayout);
    }
    assert(parent == this.parent);
  }

  /// 在任何 [sizedByParent] 发生改变的时候，要调用这个方法
  /// 将父节点和自身斗标记为需要重新布局  
  void markNeedsLayoutForSizedByParentChange() {
    markNeedsLayout();
    markParentNeedsLayout();
  }

  /// _relayoutBoundary 可能发生了变化，需要清除 RelayoutBoundary
  /// 这个操作会调用孩子的 _cleanRelayoutBoundary
  /// 之后会重新为自身和子树设置 _relayoutBoundary 
  void _cleanRelayoutBoundary() {
    if (_relayoutBoundary != this) {
      _relayoutBoundary = null;
      _needsLayout = true;
      visitChildren((RenderObject child) {
        child._cleanRelayoutBoundary();
      });
    }
  }

  /// 通过计划第一次布局来启动渲染流水线
  /// 仅被根节点调用  
  void scheduleInitialLayout() {
    /// 将自身（this）设为_relayoutBoundary  
    _relayoutBoundary = this;
    owner._nodesNeedingLayout.add(this);
  }

  /// PipelineOwner 通过调用其 _nodesNeedsLayout 列表内的每个渲染对象的 _layoutWithoutResize
  /// 来触发布局。通常 _nodesNeedsLayout 列表内的每个渲染对象都是 _relayoutBoundary
  void _layoutWithoutResize() {
    RenderObject debugPreviousActiveLayout;
    try {
      performLayout();
      markNeedsSemanticsUpdate();
    } catch (e, stack) {
      _debugReportException('performLayout', e, stack);
    }
    _needsLayout = false;
    markNeedsPaint();
  }

  /// 为本节点计算布局
  ///
  /// 这个方法是父节点请求孩子更新布局信息的主要入口。父节点传递一个constraints对象，它通知子节点哪些布
  /// 局是允许的。孩子被要求遵守给定的约束。
  ///
  /// 如果父节点读取子节点布局期间计算的信息，则父节点设置' parentUsesSize ' == true。
  /// 在这种情况下，无论何时子节点被标记为需要布局，父节点都会被标记为需要布局，因为父节点的布局信息依赖于
  /// 子节点的布局信息。
  /// 如果父节点使用' parentUsesSize '的默认值(false)，则子节点可以在不通知父类的情况下更改其布局信息
  /// (受给定约束)。
  ///
  /// 子类不应该直接重载[layout] 而是重载 [performResize]、[performLayout]
  ///
  /// 父节点的 [performLayout] 方法应该无条件地调用其所有子节点的 [layout]。如果子节点不需要做任何工
  /// 作来更新它的布局信息，[layout]方法的职责(在这里实现)是尽早返回。
  void layout(Constraints constraints, { bool parentUsesSize = false }) {
    /// 需要确定的 relayoutBoundary 
    RenderObject relayoutBoundary;
    // 如果满足以下要求之一，那么自身是一个 relayoutBoundary  
    if (
        !parentUsesSize            // 父级不需要使用这个节点的布局信息
        || sizedByParent           // 大小只受到父节点的约束，而不是由自身布局决定     
        || constraints.isTight     // 大小约束是固定的
        || parent is! RenderObject // 根节点
    ) {
      relayoutBoundary = this;
    } else {
    // 否则设置 _relayoutBoundary 成父节点的 relayoutBoundary  
      final RenderObject parent = this.parent;
      relayoutBoundary = parent._relayoutBoundary;
    }
    /// 满足以下所有条件时，返回。这样可以避免不必要的布局和绘制，提高性能  
    if (
        !_needsLayout                             // 无需重新布局
        && constraints == _constraints            // 约束没有改变
        && relayoutBoundary == _relayoutBoundary  // _relayoutBoundary 没有变化
    ) {
      return;
    }
    /// 更新 constraint  
    _constraints = constraints;
    /// 如果 _relayoutBoundary 发生了变化，需要清空孩子节点的 _relayoutBoundary，并标记它们需要重
    /// 新布局，之后它们的[layout]会设置自身的 _relayoutBoundary 
    if (_relayoutBoundary != null && relayoutBoundary != _relayoutBoundary) {
      visitChildren((RenderObject child) {
        child._cleanRelayoutBoundary();
      });
    }
    _relayoutBoundary = relayoutBoundary;
    /// 自身大小由父级决定  
    if (sizedByParent) {
      try {
        performResize();
      } catch (e, stack) {
        _debugReportException('performResize', e, stack);
      }
    }
    try {
      /// 调用到 performLayout 进行实际的布局操作
      performLayout();
      markNeedsSemanticsUpdate();
    } catch (e, stack) {
      _debugReportException('performLayout', e, stack);
    }
    _needsLayout = false;
    /// 标记需要绘制
    markNeedsPaint();
  }

  /// 大小约束是否是size算法的惟一参数
  /// 返回false总是正确的，但是返回true可以使得计算render object的size更有效率。因为在约束没有发生改
  /// 变时，不必重新计算大小
  /// 无论何时，当这个值发生改变时，调用[markNeedsLayoutForSizedByParentChange]  
  @protected
  bool get sizedByParent => false;

  /// 只用 constraint 来更新渲染对象
  ///
  /// 子类设置 [sizedByParent] 为 true，一定重载这个方法来计算他们的大小
  ///
  /// 只在 [sizedByParent] 为 true 时调用这个方法
    
  @protected
  void performResize();

  /// 完成这个节点的布局。
  /// 如果[sizedByParent]为 true，那么这个函数不应该改变这个渲染对象的尺寸。相反，这些工作应该由
  /// [performResize]来完成。
  /// 如果[sizedByParent]是 false 的，那么这个函数应该改变渲染对象的尺寸，并命令它的子对象进行布局。
  @protected
  void performLayout();

 /// 允许对该对象的子列表(以及任何后代)以及渲染树中任何其他的 被与该相同的[PipelineOwner]所持有的 拥有 
 /// 标记为 dirty 的节点进行更改。“callback” 函数是同步调用的，并且只允许在执行回调期间进行更改。
 /// 
 /// 它的存在是为了允许在布局过程中按需构建子列表(例如，基于对象的大小)，并允许节点在树中移动(例如，处理
 /// [GlobalKey]重排序)，同时仍然确保任何特定节点在每一帧中只布局一次。
 ///
 /// 调用这个方法会禁用一些 debug 流程，因此，通常不建议调用这个方法
 /// 这个方法只能在布局的时候被调用 
 @protected
  void invokeLayoutCallback<T extends Constraints>(LayoutCallback<T> callback) {
    _doingThisLayoutWithCallback = true;
    try {
      owner._enableMutationsToDirtySubtrees(() { callback(constraints); });
    } finally {
      _doingThisLayoutWithCallback = false;
    }
  }
  ... /// 未实现的渲染对象旋转
```

### Paint
```dart
  // PAINTING 绘制

  /// render object 是否与其父节点分开重绘
  ///
  /// 在子类中重载它，用来说明子类的实例应该独立重新绘制。
  /// 例如，频繁需要重新绘制的渲染对象可能希望自己重新绘制时不请求父节点的重新绘制。
  /// 
  /// 如果这个getter返回true， [paintBounds]将应用于这个对象和所有的后代。框架自动创建一个
  /// [OffsetLayer] 并将其赋值给[layer]字段。
  /// 将自身声明为重绘边界的渲染对象绝对不能替换框架创建的[OffsetLayer]，即不能修改[layer]。
  bool get isRepaintBoundary => false;

  /// 这个渲染对象是否总是需要合成。
  ///
  /// 在子类中重写它，以表示 paint 函数始终创建至少一个合成层。
  /// 例如，如果使用硬件解码器，应该返回true。
  ///
  /// 如果这个getter的值被修改，必须调用[markNeedsCompositingBitsUpdate]
  @protected
  bool get alwaysNeedsCompositing => false;

  /// 这个渲染对象由于重绘的合成层
  ///
  /// 如果此呈现对象不是重绘边界，则[paint]方法负责填充此字段。
  /// 
  /// 如果[needsCompositing]为 true，那么这个字段可能由渲染对象的实现使用的最靠近树的根的层来填充。重
  /// 新绘制时，渲染对象可能会更新存储在该字段中的层，而不是创建新层，以获得更好的性能。将该字段保留为 
  /// null 并在每次重绘时创建一个新层也是可行的，但是没有性能上的收益。
  /// 如果[needsCompositing]为 false，则必须将该字段设置为 null。方法是永远不填充该字段，或者在
  /// [needsCompositing]的值从 true 变为 false 时将其设置为 null。
  ///
  /// 如果这个渲染对象不是一个重绘边界，那么框架会自动创建一个[OffsetLayer]并在调用[paint]方法之前填充
  /// 这个字段。paint方法不能替换该字段的值。
  @protected
  ContainerLayer get layer {
    assert(!isRepaintBoundary || (_layer == null || _layer is OffsetLayer));
    return _layer;
  }

  @protected
  set layer(ContainerLayer newLayer) {
    assert(
      !isRepaintBoundary,
      'Attempted to set a layer to a repaint boundary render object.\n'
      'The framework creates and assigns an OffsetLayer to a repaint '
      'boundary automatically.',
    );
    _layer = newLayer;
  }
  /// 只能是 ContainerLayer
  ContainerLayer _layer;

  bool _needsCompositingBitsUpdate = false; // 当一个孩子被添加的时候设置为 true

  /// 标记这个渲染对象的合成状态为 dirty
  ///
  /// 调用它是为了指示需要在下一个[needsCompositing]需要在下一次
  /// [PipelineOwner.flushCompositingBits]期间重新计算。
  ///
  /// 当子树发生变化时，我们需要重新计算[needsCompositing]位，一些祖先也需要做同样的事情(以防变化会改
  /// 变他们的状态)。为此，[adoptChild]和[dropChild]会调用这个方法。
  /// 必要时，这个方法调用父节点的这个方法，等等，遍历树以标记所有需要更新的节点。
  ///
  /// 这个方法不调度渲染帧，因为它不能只改变合成位，其他的实现会为调度一个帧。
  void markNeedsCompositingBitsUpdate() {
    if (_needsCompositingBitsUpdate)
      return;
    _needsCompositingBitsUpdate = true;
    if (parent is RenderObject) {
      final RenderObject parent = this.parent;
      if (parent._needsCompositingBitsUpdate)
        return;
      // 如果自身不是重绘边界或者父节点不是重绘边界，则需要调用父节点的 
      // markNeedsCompositingBitsUpdate
      if (!isRepaintBoundary && !parent.isRepaintBoundary) {
        parent.markNeedsCompositingBitsUpdate();
        return;
      }
    }
    // 把自身加入 owner 的 _nodesNeedingCompositingBitsUpdate 中
    if (owner != null)
      owner._nodesNeedingCompositingBitsUpdate.add(this);
  }

  // 构造函数中 _needsCompositing = isRepaintBoundary || alwaysNeedsCompositing;
  // 即当自身是重绘边界或者总是需要合成（至少创建一个合成层）的时候，在首次构建即为true
  bool _needsCompositing; 
  /// Whether we or one of our descendants has a compositing layer.
  ///
  /// If this node needs compositing as indicated by this bit, then all ancestor
  /// nodes will also need compositing.
  ///
  /// 只在 [PipelineOwner.flushLayout] 和 [PipelineOwner.flushCompositingBits] 被调用后有效
  bool get needsCompositing {
    return _needsCompositing;
  }

  /// PipelineOwner 通过调用其 _nodesneedsCompositingBitsUpdate 列表内的每个渲染对象的
  /// _updateCompositingBits 来检查那个渲染对象和它的子树是否需要合成。当需要合成的时候，会调用渲染对
  /// 象的 markNeedsPaint
  void _updateCompositingBits() {
    if (!_needsCompositingBitsUpdate)
      return;
    final bool oldNeedsCompositing = _needsCompositing;
    _needsCompositing = false;
    visitChildren((RenderObject child) {
      child._updateCompositingBits();
      if (child.needsCompositing)
        _needsCompositing = true;
    });
    if (isRepaintBoundary || alwaysNeedsCompositing)
      _needsCompositing = true;
    if (oldNeedsCompositing != _needsCompositing)
      markNeedsPaint();
    _needsCompositingBitsUpdate = false;
  }
  
  /// 初始标记为 true  
  bool _needsPaint = true;

  /// 标记这个渲染对象改变了他的视觉外观
  ///
  /// 同 [markNeedsLayout]，这个方法是一种懒作用的。
  void markNeedsPaint() {
    if (_needsPaint)
      return;
    _needsPaint = true;
    // 如果自身是重绘边界的话，把自身加入到 PipelineOwner 的 _nodesNeedingPaint 列表中
    // 并请求一次视觉更新  
    if (isRepaintBoundary) {
      if (owner != null) {
        owner._nodesNeedingPaint.add(this);
        owner.requestVisualUpdate();
      }
    // 否则，如果父级是渲染对象（即不是根节点的话），调用父节点 markNeedsPaint() 
    } else if (parent is RenderObject) {
      // 这个渲染对象没有自己的图层;它的的一个祖先会负责更新这个渲染对象所在的层
      // 当他们这样做时，这个渲染对象会调用 paint 方法。
      assert(_layer == null);
      final RenderObject parent = this.parent;
      parent.markNeedsPaint();
      assert(parent == this.parent);
    } else {
      // 这里是渲染对象树的根节点(可能是一个 renderView)，因此这个渲染对象的paint总是会被调用
      // 把自身加入到 PipelineOwner 的 _nodesNeedingPaint 列表中  
      if (owner != null)
        owner.requestVisualUpdate();
    }
  }

  /// 调用时，flushPaint 试图绘制这个节点，但图层还没有被装载。
  /// 为了确保我们的子树在最后重新连接时被重新绘制，即使在某些祖先层本身没有被标记为dirty的情况下，我们也
  /// 必须将整个分离的子树标记为脏的，并且需要重新绘制。那样的话，我们最终会被重新绘制
  void _skippedPaintingOnLayer() {
    AbstractNode ancestor = parent;
    while (ancestor is RenderObject) {
      final RenderObject node = ancestor;
      if (node.isRepaintBoundary) {
        if (node._layer == null)
          break; 
        if (node._layer.attached)
          break; 
        /// 将绘制边界的 _needsPaint 设置为 true，则其子树会被重绘   
        node._needsPaint = true;
      }
      ancestor = node.parent;
    }
  }

  /// 通过调度第一次绘制来引导渲染流水线。
  /// 需要这个渲染对象满足： 1.已经挂载 2.是渲染树的根 3.并且有一个合成层。
  void scheduleInitialPaint(ContainerLayer rootLayer) {
    _layer = rootLayer;
    owner._nodesNeedingPaint.add(this);
  }

  /// 替换合成层，
  ///
  /// 可能在设备像素比发生变化的时候调用
  void replaceRootLayer(OffsetLayer rootLayer) {
    _layer.detach();
    _layer = rootLayer;
    markNeedsPaint();
  }

  /// 这个方法会调用paint方法
  void _paintWithContext(PaintingContext context, Offset offset) {
    // If we still need layout, then that means that we were skipped in the
    // layout phase and therefore don't need painting. We might not know that
    // yet (that is, our layer might not have been detached yet), because the
    // same node that skipped us in layout is above us in the tree (obviously)
    // and therefore may not have had a chance to paint yet (since the tree
    // paints in reverse order). In particular this will happen if they have
    // a different layer, because there's a repaint boundary between us.
    if (_needsLayout)
      return;
    RenderObject debugLastActivePaint;
    _needsPaint = false;
    try {
      paint(context, offset);
    } catch (e, stack) {
      _debugReportException('paint', e, stack);
    }
  }

  /// An estimate of the bounds within which this render object will paint.
  /// Useful for debugging flags such as [debugPaintLayerBordersEnabled].
  ///
  /// These are also the bounds used by [showOnScreen] to make a [RenderObject]
  /// visible on screen.
  Rect get paintBounds;

  /// Override this method to paint debugging information.
  void debugPaint(PaintingContext context, Offset offset) { }

  /// 以给定的偏移量将此渲染对象绘制到给定的上下文中。
  ///
  /// 子类应该重载这个方法来展现它们的视觉外观
  ///
  /// 这个方法不应该被直接调用
  /// 如果需要绘制自身，调用 [markNeedsPaint]
  /// 如果需要绘制孩子，调用 [PaintingContext.paintChild]
  ///
  /// 当绘制一个孩子(通过给定上下文中的paint child函数)时，当前由上下文中持有的画布可能会改变。
  /// 因为绘制前后的绘制操作可能需要在单独的合成层上记录
  void paint(PaintingContext context, Offset offset) { }

  /// 应用将在将给定子元素绘制到给定矩阵时应用的转换。
  ///
  /// 坐标转换函数用于将一个渲染对象的本地坐标转换为另一个渲染对象的本地坐标。
  void applyPaintTransform(covariant RenderObject child, Matrix4 transform) {
    assert(child.parent == this);
  }

  /// 获取从这个渲染对象到给定的RenderObject（先祖）的矩阵变换
  ///
  /// 返回一个矩阵，该矩阵将本地绘制坐标系统映射到“祖先”的坐标系统。
  ///
  /// 如果‘祖先’为空，则此方法返回一个矩阵，该矩阵将从本地绘制坐标系统映射到[PipelineOwner.rootNode] 
  /// （渲染对象树的根节点）的坐标系统。
  /// 对于被[RendererBinding]所持有的渲染树，(即对于设备上显示的主要渲染树)，该方法映射到全局逻辑像素
  /// 坐标系统
  /// 要获取物理像素，从[RenderView]中使用[applyPaintTransform]来进一步转换坐标。
  Matrix4 getTransformTo(RenderObject ancestor) {
    /// 如果 ancestor == null，则默认为根节点
    if (ancestor == null) {
      final AbstractNode rootNode = owner.rootNode;
      if (rootNode is RenderObject)
        ancestor = rootNode;
    }
    final List<RenderObject> renderers = <RenderObject>[];
    /// 向上遍历每一个渲染对象直到根节点，添加到列表中 
    for (
        RenderObject renderer = this; 
        renderer != ancestor; 
        renderer = renderer.parent
    ) {
      renderers.add(renderer);
    }
    /// 单位矩阵  
    final Matrix4 transform = Matrix4.identity();
    /// 对列表中的每一个渲染对象，调用其.applyPaintTransform，这会修改 transform
    /// 当遍历完成之后，就得到了从这个渲染对象到目标对象的矩阵变换  
    for (int index = renderers.length - 1; index > 0; index -= 1) {
      renderers[index].applyPaintTransform(renderers[index - 1], transform);
    }
    return transform;
  }


  /// Returns a rect in this object's coordinate system that describes
  /// the approximate bounding box of the clip rect that would be
  /// applied to the given child during the paint phase, if any.
  ///
  /// Returns null if the child would not be clipped.
  ///
  /// This is used in the semantics phase to avoid including children
  /// that are not physically visible.
  Rect describeApproximatePaintClip(covariant RenderObject child) => null;

  /// Returns a rect in this object's coordinate system that describes
  /// which [SemanticsNode]s produced by the `child` should be included in the
  /// semantics tree. [SemanticsNode]s from the `child` that are positioned
  /// outside of this rect will be dropped. Child [SemanticsNode]s that are
  /// positioned inside this rect, but outside of [describeApproximatePaintClip]
  /// will be included in the tree marked as hidden. Child [SemanticsNode]s
  /// that are inside of both rect will be included in the tree as regular
  /// nodes.
  ///
  /// This method only returns a non-null value if the semantics clip rect
  /// is different from the rect returned by [describeApproximatePaintClip].
  /// If the semantics clip rect and the paint clip rect are the same, this
  /// method returns null.
  ///
  /// A viewport would typically implement this method to include semantic nodes
  /// in the semantics tree that are currently hidden just before the leading
  /// or just after the trailing edge. These nodes have to be included in the
  /// semantics tree to implement implicit accessibility scrolling on iOS where
  /// the viewport scrolls implicitly when moving the accessibility focus from
  /// a the last visible node in the viewport to the first hidden one.
  Rect describeSemanticsClip(covariant RenderObject child) => null;
```
### Semantics
```dart
  // SEMANTICS

  /// Bootstrap the semantics reporting mechanism by marking this node
  /// as needing a semantics update.
  ///
  /// Requires that this render object is attached, and is the root of
  /// the render tree.
  ///
  /// See [RendererBinding] for an example of how this function is used.
  void scheduleInitialSemantics() {
    assert(attached);
    assert(parent is! RenderObject);
    assert(!owner._debugDoingSemantics);
    assert(_semantics == null);
    assert(_needsSemanticsUpdate);
    assert(owner._semanticsOwner != null);
    owner._nodesNeedingSemantics.add(this);
    owner.requestVisualUpdate();
  }

  /// Report the semantics of this node, for example for accessibility purposes.
  ///
  /// This method should be overridden by subclasses that have interesting
  /// semantic information.
  ///
  /// The given [SemanticsConfiguration] object is mutable and should be
  /// annotated in a manner that describes the current state. No reference
  /// should be kept to that object; mutating it outside of the context of the
  /// [describeSemanticsConfiguration] call (for example as a result of
  /// asynchronous computation) will at best have no useful effect and at worse
  /// will cause crashes as the data will be in an inconsistent state.
  ///
  /// {@tool sample}
  ///
  /// The following snippet will describe the node as a button that responds to
  /// tap actions.
  ///
  /// ```dart
  /// abstract class SemanticButtonRenderObject extends RenderObject {
  ///   @override
  ///   void describeSemanticsConfiguration(SemanticsConfiguration config) {
  ///     super.describeSemanticsConfiguration(config);
  ///     config
  ///       ..onTap = _handleTap
  ///       ..label = 'I am a button'
  ///       ..isButton = true;
  ///   }
  ///
  ///   void _handleTap() {
  ///     // Do something.
  ///   }
  /// }
  /// ```
  /// {@end-tool}
  @protected
  void describeSemanticsConfiguration(SemanticsConfiguration config) {
    // Nothing to do by default.
  }

  /// Sends a [SemanticsEvent] associated with this render object's [SemanticsNode].
  ///
  /// If this render object has no semantics information, the first parent
  /// render object with a non-null semantic node is used.
  ///
  /// If semantics are disabled, no events are dispatched.
  ///
  /// See [SemanticsNode.sendEvent] for a full description of the behavior.
  void sendSemanticsEvent(SemanticsEvent semanticsEvent) {
    if (owner.semanticsOwner == null)
      return;
    if (_semantics != null && !_semantics.isMergedIntoParent) {
      _semantics.sendEvent(semanticsEvent);
    } else if (parent != null) {
      final RenderObject renderParent = parent;
      renderParent.sendSemanticsEvent(semanticsEvent);
    }
  }

  // Use [_semanticsConfiguration] to access.
  SemanticsConfiguration _cachedSemanticsConfiguration;

  SemanticsConfiguration get _semanticsConfiguration {
    if (_cachedSemanticsConfiguration == null) {
      _cachedSemanticsConfiguration = SemanticsConfiguration();
      describeSemanticsConfiguration(_cachedSemanticsConfiguration);
    }
    return _cachedSemanticsConfiguration;
  }

  /// The bounding box, in the local coordinate system, of this
  /// object, for accessibility purposes.
  Rect get semanticBounds;

  bool _needsSemanticsUpdate = true;
  SemanticsNode _semantics;

  /// The semantics of this render object.
  ///
  /// Exposed only for testing and debugging. To learn about the semantics of
  /// render objects in production, obtain a [SemanticsHandle] from
  /// [PipelineOwner.ensureSemantics].
  ///
  /// Only valid when asserts are enabled. In release builds, always returns
  /// null.
  SemanticsNode get debugSemantics {
    SemanticsNode result;
    assert(() {
      result = _semantics;
      return true;
    }());
    return result;
  }

  /// Removes all semantics from this render object and its descendants.
  ///
  /// Should only be called on objects whose [parent] is not a [RenderObject].
  ///
  /// Override this method if you instantiate new [SemanticsNode]s in an
  /// overridden [assembleSemanticsNode] method, to dispose of those nodes.
  @mustCallSuper
  void clearSemantics() {
    _needsSemanticsUpdate = true;
    _semantics = null;
    visitChildren((RenderObject child) {
      child.clearSemantics();
    });
  }

  /// Mark this node as needing an update to its semantics description.
  ///
  /// This must be called whenever the semantics configuration of this
  /// [RenderObject] as annotated by [describeSemanticsConfiguration] changes in
  /// any way to update the semantics tree.
  void markNeedsSemanticsUpdate() {
    assert(!attached || !owner._debugDoingSemantics);
    if (!attached || owner._semanticsOwner == null) {
      _cachedSemanticsConfiguration = null;
      return;
    }

    // Dirty the semantics tree starting at `this` until we have reached a
    // RenderObject that is a semantics boundary. All semantics past this
    // RenderObject are still up-to date. Therefore, we will later only rebuild
    // the semantics subtree starting at the identified semantics boundary.

    final bool wasSemanticsBoundary = _semantics != null && _cachedSemanticsConfiguration?.isSemanticBoundary == true;
    _cachedSemanticsConfiguration = null;
    bool isEffectiveSemanticsBoundary = _semanticsConfiguration.isSemanticBoundary && wasSemanticsBoundary;
    RenderObject node = this;

    while (!isEffectiveSemanticsBoundary && node.parent is RenderObject) {
      if (node != this && node._needsSemanticsUpdate)
        break;
      node._needsSemanticsUpdate = true;

      node = node.parent;
      isEffectiveSemanticsBoundary = node._semanticsConfiguration.isSemanticBoundary;
      if (isEffectiveSemanticsBoundary && node._semantics == null) {
        // We have reached a semantics boundary that doesn't own a semantics node.
        // That means the semantics of this branch are currently blocked and will
        // not appear in the semantics tree. We can abort the walk here.
        return;
      }
    }
    if (node != this && _semantics != null && _needsSemanticsUpdate) {
      // If `this` node has already been added to [owner._nodesNeedingSemantics]
      // remove it as it is no longer guaranteed that its semantics
      // node will continue to be in the tree. If it still is in the tree, the
      // ancestor `node` added to [owner._nodesNeedingSemantics] at the end of
      // this block will ensure that the semantics of `this` node actually gets
      // updated.
      // (See semantics_10_test.dart for an example why this is required).
      owner._nodesNeedingSemantics.remove(this);
    }
    if (!node._needsSemanticsUpdate) {
      node._needsSemanticsUpdate = true;
      if (owner != null) {
        assert(node._semanticsConfiguration.isSemanticBoundary || node.parent is! RenderObject);
        owner._nodesNeedingSemantics.add(node);
        owner.requestVisualUpdate();
      }
    }
  }

  /// Updates the semantic information of the render object.
  void _updateSemantics() {
    assert(_semanticsConfiguration.isSemanticBoundary || parent is! RenderObject);
    if (_needsLayout) {
      // There's not enough information in this subtree to compute semantics.
      // The subtree is probably being kept alive by a viewport but not laid out.
      return;
    }
    final _SemanticsFragment fragment = _getSemanticsForParent(
      mergeIntoParent: _semantics?.parent?.isPartOfNodeMerging ?? false,
    );
    assert(fragment is _InterestingSemanticsFragment);
    final _InterestingSemanticsFragment interestingFragment = fragment;
    final SemanticsNode node = interestingFragment.compileChildren(
      parentSemanticsClipRect: _semantics?.parentSemanticsClipRect,
      parentPaintClipRect: _semantics?.parentPaintClipRect,
      elevationAdjustment: _semantics?.elevationAdjustment ?? 0.0,
    ).single;
    // Fragment only wants to add this node's SemanticsNode to the parent.
    assert(interestingFragment.config == null && node == _semantics);
  }

  /// Returns the semantics that this node would like to add to its parent.
  _SemanticsFragment _getSemanticsForParent({
    @required bool mergeIntoParent,
  }) {
    assert(mergeIntoParent != null);
    assert(!_needsLayout, 'Updated layout information required for $this to calculate semantics.');

    final SemanticsConfiguration config = _semanticsConfiguration;
    bool dropSemanticsOfPreviousSiblings = config.isBlockingSemanticsOfPreviouslyPaintedNodes;

    final bool producesForkingFragment = !config.hasBeenAnnotated && !config.isSemanticBoundary;
    final List<_InterestingSemanticsFragment> fragments = <_InterestingSemanticsFragment>[];
    final Set<_InterestingSemanticsFragment> toBeMarkedExplicit = <_InterestingSemanticsFragment>{};
    final bool childrenMergeIntoParent = mergeIntoParent || config.isMergingSemanticsOfDescendants;

    // When set to true there's currently not enough information in this subtree
    // to compute semantics. In this case the walk needs to be aborted and no
    // SemanticsNodes in the subtree should be updated.
    // This will be true for subtrees that are currently kept alive by a
    // viewport but not laid out.
    bool abortWalk = false;

    visitChildrenForSemantics((RenderObject renderChild) {
      if (abortWalk || _needsLayout) {
        abortWalk = true;
        return;
      }
      final _SemanticsFragment parentFragment = renderChild._getSemanticsForParent(
        mergeIntoParent: childrenMergeIntoParent,
      );
      if (parentFragment.abortsWalk) {
        abortWalk = true;
        return;
      }
      if (parentFragment.dropsSemanticsOfPreviousSiblings) {
        fragments.clear();
        toBeMarkedExplicit.clear();
        if (!config.isSemanticBoundary)
          dropSemanticsOfPreviousSiblings = true;
      }
      // Figure out which child fragments are to be made explicit.
      for (_InterestingSemanticsFragment fragment in parentFragment.interestingFragments) {
        fragments.add(fragment);
        fragment.addAncestor(this);
        fragment.addTags(config.tagsForChildren);
        if (config.explicitChildNodes || parent is! RenderObject) {
          fragment.markAsExplicit();
          continue;
        }
        if (!fragment.hasConfigForParent || producesForkingFragment)
          continue;
        if (!config.isCompatibleWith(fragment.config))
          toBeMarkedExplicit.add(fragment);
        for (_InterestingSemanticsFragment siblingFragment in fragments.sublist(0, fragments.length - 1)) {
          if (!fragment.config.isCompatibleWith(siblingFragment.config)) {
            toBeMarkedExplicit.add(fragment);
            toBeMarkedExplicit.add(siblingFragment);
          }
        }
      }
    });

    if (abortWalk) {
      return _AbortingSemanticsFragment(owner: this);
    }

    for (_InterestingSemanticsFragment fragment in toBeMarkedExplicit)
      fragment.markAsExplicit();

    _needsSemanticsUpdate = false;

    _SemanticsFragment result;
    if (parent is! RenderObject) {
      assert(!config.hasBeenAnnotated);
      assert(!mergeIntoParent);
      result = _RootSemanticsFragment(
        owner: this,
        dropsSemanticsOfPreviousSiblings: dropSemanticsOfPreviousSiblings,
      );
    } else if (producesForkingFragment) {
      result = _ContainerSemanticsFragment(
        dropsSemanticsOfPreviousSiblings: dropSemanticsOfPreviousSiblings,
      );
    } else {
      result = _SwitchableSemanticsFragment(
        config: config,
        mergeIntoParent: mergeIntoParent,
        owner: this,
        dropsSemanticsOfPreviousSiblings: dropSemanticsOfPreviousSiblings,
      );
      if (config.isSemanticBoundary) {
        final _SwitchableSemanticsFragment fragment = result;
        fragment.markAsExplicit();
      }
    }

    result.addAll(fragments);

    return result;
  }

  /// Called when collecting the semantics of this node.
  ///
  /// The implementation has to return the children in paint order skipping all
  /// children that are not semantically relevant (e.g. because they are
  /// invisible).
  ///
  /// The default implementation mirrors the behavior of
  /// [visitChildren()] (which is supposed to walk all the children).
  void visitChildrenForSemantics(RenderObjectVisitor visitor) {
    visitChildren(visitor);
  }

  /// Assemble the [SemanticsNode] for this [RenderObject].
  ///
  /// If [isSemanticBoundary] is true, this method is called with the `node`
  /// created for this [RenderObject], the `config` to be applied to that node
  /// and the `children` [SemanticNode]s that descendants of this RenderObject
  /// have generated.
  ///
  /// By default, the method will annotate `node` with `config` and add the
  /// `children` to it.
  ///
  /// Subclasses can override this method to add additional [SemanticsNode]s
  /// to the tree. If new [SemanticsNode]s are instantiated in this method
  /// they must be disposed in [clearSemantics].
  void assembleSemanticsNode(
    SemanticsNode node,
    SemanticsConfiguration config,
    Iterable<SemanticsNode> children,
  ) {
    assert(node == _semantics);
    node.updateWith(config: config, childrenInInversePaintOrder: children);
  }
```
### Event
```dart
  // EVENTS

  /// 重载这个方法来处理点击在这个渲染对象上的点击事件
  @override
  void handleEvent(PointerEvent event, covariant HitTestEntry entry) { }
```
### Hit Testing
```dart
  // HIT TESTING

  // 渲染对象子类应该实现如下和方法：
  //
  // bool hitTest(HitTestResult result, { Offset position }) {
  //   // 如果给定的 position 不在 这个渲染对象节点之内，返回 false.
  //   // Otherwise:
  //   // For each child that intersects the position, in z-order starting from
  //   // the top, call hitTest() for that child, passing it /result/, and the
  //   // coordinates converted to the child's coordinate origin, and stop at
  //   // the first child that returns true.
  //   // Then, add yourself to /result/, and return true.
  // }
  //
  // If you add yourself to /result/ and still return false, then that means you
  // will see events but so will objects below you.
```
### Other
```dart
  /// 尝试让这个或者部分后代 [RenderObject] 节点出现在屏幕上
  ///
  /// 如果提供了 `descendant`，那么让它可视化，否则让这个渲染对象可视化
  ///
  /// The optional `rect` parameter describes which area of that [RenderObject]
  /// should be shown on screen. If `rect` is null, the entire
  /// [RenderObject] (as defined by its [paintBounds]) will be revealed. The
  /// `rect` parameter is interpreted relative to the coordinate system of
  /// `descendant` if that argument is provided and relative to this
  /// [RenderObject] otherwise.
  ///
  /// The `duration` parameter can be set to a non-zero value to bring the
  /// target object on screen in an animation defined by `curve`.
  void showOnScreen({
    RenderObject descendant,
    Rect rect,
    Duration duration = Duration.zero,
    Curve curve = Curves.ease,
  }) {
    if (parent is RenderObject) {
      final RenderObject renderParent = parent;
      renderParent.showOnScreen(
        descendant: descendant ?? this,
        rect: rect,
        duration: duration,
        curve: curve,
      );
    }
  }
}
```




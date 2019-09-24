# Flutter RenderObject 源码分析

[TOC]

## RenderObject

```dart
abstract class RenderObject extends AbstractNode 
    with DiagnosticableTreeMixin implements HitTestTarget 
{
  /// Initializes internal fields for subclasses.
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

  // LAYOUT

  /// 为孩子设置有用的信息，仅通过父节点的 setupParentData 方法设置，而不是直接修改
  ParentData parentData;

  /// 可以在子RenderObject还未加入到子元素列表时，为它附加parentData
  void setupParentData(covariant RenderObject child) {
    assert(_debugCanPerformMutations);
    if (child.parentData is! ParentData)
      child.parentData = ParentData();
  }

  /// 将一个RenderObejct设置为孩子，会触发重新布局
  @override
  void adoptChild(RenderObject child) {
    assert(_debugCanPerformMutations);
    assert(child != null);
    setupParentData(child);
    markNeedsLayout();
    markNeedsCompositingBitsUpdate();
    markNeedsSemanticsUpdate();
    super.adoptChild(child);
  }

  /// 移除一个孩子，会触发重新布局
  @override
  void dropChild(RenderObject child) {
    assert(_debugCanPerformMutations);
    assert(child != null);
    assert(child.parentData != null);
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

  /// 分配  PipelineOwner
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

  /// 是否需要重新布局（在实例化RenderObject的时候初始化为true）  
  bool _needsLayout = true;

  RenderObject _relayoutBoundary;
  bool _doingThisLayoutWithCallback = false;

  /// 父节点给子节点的约束
  @protected
  Constraints get constraints => _constraints;
  Constraints _constraints;
    
  /// 将这个renderobecjt标记为脏，并在PipelineOwner中注册（或延迟给父节点），
  /// 这取决于这个节点是否是relayout boundary
    
  /// 如果[sizedByParent]发生改变，调用
  /// [markNeedsLayoutForSizedByParentChange]而不是[markNeedsLayout].
    
  void markNeedsLayout() {
    /// 如果已经标记，返回
    if (_needsLayout) {
      assert(_debugSubtreeRelayoutRootAlreadyMarkedNeedsLayout());
      return;
    }
    assert(_relayoutBoundary != null);
    /// 如果不是  _relayoutBoundary ，调用markParentNeedsLayout()，
    /// 其中调用了parent.markNeedsLayout()  
    if (_relayoutBoundary != this) {
      markParentNeedsLayout();
    } else {
    /// 否则，在 PipelineOwne 注册，在 _nodesNeedingLayout加入自己
      _needsLayout = true;
      if (owner != null) {
        owner._nodesNeedingLayout.add(this);
        owner.requestVisualUpdate();
      }
    }
  }

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

  /// 在任何[sizedByParent]发生改变的时候，要调用这个方法
  /// 将父节点和自身斗标记为需要重新布局  
  void markNeedsLayoutForSizedByParentChange() {
    markNeedsLayout();
    markParentNeedsLayout();
  }

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
    assert(attached);
    assert(parent is! RenderObject);
    assert(!owner._debugDoingLayout);
    assert(_relayoutBoundary == null);
    /// 将自身设为_relayoutBoundary  
    _relayoutBoundary = this;
    assert(() {
      _debugCanParentUseSize = false;
      return true;
    }());
    owner._nodesNeedingLayout.add(this);
  }

  /// 在改变不必改变父节点布局的情况下开始布局，这个方法会被PipelineOwner调用 
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
  /// 这个方法是父节点请求子节点布局的进入点。父节点会传入一个约束
  ///
  /// 如果父节点需要子节点的布局信息来规划自身的布局，则把parentUsesSize设为true
  /// 当parentUsesSize为false时，子节点可以在不通知父节的情况下改动自身约束
  ///
  /// 子类不应该直接重载[layout] 而是重载[performResize]、[performLayout]
  /// 父节点的[performLayout]必须无条件地调用子节点的[layout]
  void layout(Constraints constraints, { bool parentUsesSize = false }) {
    assert(constraints != null);
    ...
    RenderObject relayoutBoundary;
      
    if (!parentUsesSize || sizedByParent || constraints.isTight || parent is! RenderObject) {
      /// 如果父节点不适用自身大小，或大小仅受父布局大小影响，或是固定大小，或父节点不是RenderObject
      /// 则把自身设为relayoutBoundary
      relayoutBoundary = this;
    } else {
      /// 否则，relayoutBoundary则是父节点的relayoutBoundary  
      final RenderObject parent = this.parent;
      relayoutBoundary = parent._relayoutBoundary;
    }
    ...
     
    /// 将自身的约束设为父节点传入的约束    
    _constraints = constraints;
    /// 设置 relayoutBoundary（在判断时通过 _relayoutBoundary == this来确定重新布局边界）
    _relayoutBoundary = relayoutBoundary;
    
    /// 开始布局
    try {
      performLayout();
      markNeedsSemanticsUpdate();
      assert(() { debugAssertDoesMeetConstraints(); return true; }());
    } catch (e, stack) {
      _debugReportException('performLayout', e, stack);
    }
    ...
    /// 完成后取消标记    
    _needsLayout = false;
    /// 标记需要绘制  
    markNeedsPaint();
  }

  /// 大小是否仅受父节点的constraint的影响
  @protected
  bool get sizedByParent => false;

  /// 只用constraint来更新渲染对象
  ///
  /// 子类设置[sizedByParent]为true，一个重载这个方法来计算他们的大小
  ///
  /// 只在[sizedByParent]为true时调用这个方法
    
  @protected
  void performResize();

  /// 布局。必须在这个方法里调用孩子的layout
  @protected
  void performLayout();

  ... /// 未实现的屏幕旋转

  // PAINTING

  /// render object 是否与其父节点分开重绘
  ///
  /// 设为true时，这个属性将会应用于这个节点及其所有后代
  bool get isRepaintBoundary => false;

  /// Whether this render object always needs compositing.
  ///
  /// Override this in subclasses to indicate that your paint function always
  /// creates at least one composited layer. For example, videos should return
  /// true if they use hardware decoders.
  ///
  /// You must call [markNeedsCompositingBitsUpdate] if the value of this getter
  /// changes. (This is implied when [adoptChild] or [dropChild] are called.)
  @protected
  bool get alwaysNeedsCompositing => false;

  OffsetLayer _layer;
  /// The compositing layer that this render object uses to repaint.
  ///
  /// Call only when [isRepaintBoundary] is true and the render object has
  /// already painted.
  ///
  /// To access the layer in debug code, even when it might be inappropriate to
  /// access it (e.g. because it is dirty), consider [debugLayer].
  OffsetLayer get layer {
    assert(isRepaintBoundary, 'You can only access RenderObject.layer for render objects that are repaint boundaries.');
    assert(!_needsPaint);
    return _layer;
  }
  /// In debug mode, the compositing layer that this render object uses to repaint.
  ///
  /// This getter is intended for debugging purposes only. In release builds, it
  /// always returns null. In debug builds, it returns the layer even if the layer
  /// is dirty.
  ///
  /// For production code, consider [layer].
  OffsetLayer get debugLayer {
    OffsetLayer result;
    assert(() {
      result = _layer;
      return true;
    }());
    return result;
  }

  bool _needsCompositingBitsUpdate = false; // set to true when a child is added
  /// Mark the compositing state for this render object as dirty.
  ///
  /// This is called to indicate that the value for [needsCompositing] needs to
  /// be recomputed during the next [flushCompositingBits] engine phase.
  ///
  /// When the subtree is mutated, we need to recompute our
  /// [needsCompositing] bit, and some of our ancestors need to do the
  /// same (in case ours changed in a way that will change theirs). To
  /// this end, [adoptChild] and [dropChild] call this method, and, as
  /// necessary, this method calls the parent's, etc, walking up the
  /// tree to mark all the nodes that need updating.
  ///
  /// This method does not schedule a rendering frame, because since
  /// it cannot be the case that _only_ the compositing bits changed,
  /// something else will have scheduled a frame for us.
  void markNeedsCompositingBitsUpdate() {
    if (_needsCompositingBitsUpdate)
      return;
    _needsCompositingBitsUpdate = true;
    if (parent is RenderObject) {
      final RenderObject parent = this.parent;
      if (parent._needsCompositingBitsUpdate)
        return;
      if (!isRepaintBoundary && !parent.isRepaintBoundary) {
        parent.markNeedsCompositingBitsUpdate();
        return;
      }
    }
    assert(() {
      final AbstractNode parent = this.parent;
      if (parent is RenderObject)
        return parent._needsCompositing;
      return true;
    }());
    // parent is fine (or there isn't one), but we are dirty
    if (owner != null)
      owner._nodesNeedingCompositingBitsUpdate.add(this);
  }

  bool _needsCompositing; // initialized in the constructor
  /// Whether we or one of our descendants has a compositing layer.
  ///
  /// If this node needs compositing as indicated by this bit, then all ancestor
  /// nodes will also need compositing.
  ///
  /// Only legal to call after [PipelineOwner.flushLayout] and
  /// [PipelineOwner.flushCompositingBits] have been called.
  bool get needsCompositing {
    assert(!_needsCompositingBitsUpdate); // make sure we don't use this bit when it is dirty
    return _needsCompositing;
  }

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

  
  /// 初始标记为true  
  bool _needsPaint = true;

  /// Mark this render object as having changed its visual appearance.
  ///
  /// Rather than eagerly updating this render object's display list
  /// in response to writes, we instead mark the render object as needing to
  /// paint, which schedules a visual update. As part of the visual update, the
  /// rendering pipeline will give this render object an opportunity to update
  /// its display list.
  ///
  /// This mechanism batches the painting work so that multiple sequential
  /// writes are coalesced, removing redundant computation.
  ///
  /// Once [markNeedsPaint] has been called on a render object,
  /// [debugNeedsPaint] returns true for that render object until just after
  /// the pipeline owner has called [paint] on the render object.
  ///
  /// See also:
  ///
  ///  * [RepaintBoundary], to scope a subtree of render objects to their own
  ///    layer, thus limiting the number of nodes that [markNeedsPaint] must mark
  ///    dirty.
  void markNeedsPaint() {
    assert(owner == null || !owner.debugDoingPaint);
    if (_needsPaint)
      return;
    _needsPaint = true;
    if (isRepaintBoundary) {
      assert(() {
        if (debugPrintMarkNeedsPaintStacks)
          debugPrintStack(label: 'markNeedsPaint() called for $this');
        return true;
      }());
      // If we always have our own layer, then we can just repaint
      // ourselves without involving any other nodes.
      assert(_layer != null);
      if (owner != null) {
        owner._nodesNeedingPaint.add(this);
        owner.requestVisualUpdate();
      }
    } else if (parent is RenderObject) {
      // We don't have our own layer; one of our ancestors will take
      // care of updating the layer we're in and when they do that
      // we'll get our paint() method called.
      assert(_layer == null);
      final RenderObject parent = this.parent;
      parent.markNeedsPaint();
      assert(parent == this.parent);
    } else {
      assert(() {
        if (debugPrintMarkNeedsPaintStacks)
          debugPrintStack(label: 'markNeedsPaint() called for $this (root of render tree)');
        return true;
      }());
      // If we're the root of the render tree (probably a RenderView),
      // then we have to paint ourselves, since nobody else can paint
      // us. We don't add ourselves to _nodesNeedingPaint in this
      // case, because the root is always told to paint regardless.
      if (owner != null)
        owner.requestVisualUpdate();
    }
  }

  // Called when flushPaint() tries to make us paint but our layer is detached.
  // To make sure that our subtree is repainted when it's finally reattached,
  // even in the case where some ancestor layer is itself never marked dirty, we
  // have to mark our entire detached subtree as dirty and needing to be
  // repainted. That way, we'll eventually be repainted.
  void _skippedPaintingOnLayer() {
    assert(attached);
    assert(isRepaintBoundary);
    assert(_needsPaint);
    assert(_layer != null);
    assert(!_layer.attached);
    AbstractNode ancestor = parent;
    while (ancestor is RenderObject) {
      final RenderObject node = ancestor;
      if (node.isRepaintBoundary) {
        if (node._layer == null)
          break; // looks like the subtree here has never been painted. let it handle itself.
        if (node._layer.attached)
          break; // it's the one that detached us, so it's the one that will decide to repaint us.
        node._needsPaint = true;
      }
      ancestor = node.parent;
    }
  }

  /// Bootstrap the rendering pipeline by scheduling the very first paint.
  ///
  /// Requires that this render object is attached, is the root of the render
  /// tree, and has a composited layer.
  ///
  /// See [RenderView] for an example of how this function is used.
  void scheduleInitialPaint(ContainerLayer rootLayer) {
    assert(rootLayer.attached);
    assert(attached);
    assert(parent is! RenderObject);
    assert(!owner._debugDoingPaint);
    assert(isRepaintBoundary);
    assert(_layer == null);
    _layer = rootLayer;
    assert(_needsPaint);
    owner._nodesNeedingPaint.add(this);
  }

  /// Replace the layer. This is only valid for the root of a render
  /// object subtree (whatever object [scheduleInitialPaint] was
  /// called on).
  ///
  /// This might be called if, e.g., the device pixel ratio changed.
  void replaceRootLayer(OffsetLayer rootLayer) {
    assert(rootLayer.attached);
    assert(attached);
    assert(parent is! RenderObject);
    assert(!owner._debugDoingPaint);
    assert(isRepaintBoundary);
    assert(_layer != null); // use scheduleInitialPaint the first time
    _layer.detach();
    _layer = rootLayer;
    markNeedsPaint();
  }

  void _paintWithContext(PaintingContext context, Offset offset) {
    assert(() {
      if (_debugDoingThisPaint) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('Tried to paint a RenderObject reentrantly.'),
          describeForError(
            'The following RenderObject was already being painted when it was '
            'painted again'
          ),
          ErrorDescription(
            'Since this typically indicates an infinite recursion, it is '
            'disallowed.'
          )
        ]);
      }
      return true;
    }());
    // If we still need layout, then that means that we were skipped in the
    // layout phase and therefore don't need painting. We might not know that
    // yet (that is, our layer might not have been detached yet), because the
    // same node that skipped us in layout is above us in the tree (obviously)
    // and therefore may not have had a chance to paint yet (since the tree
    // paints in reverse order). In particular this will happen if they have
    // a different layer, because there's a repaint boundary between us.
    if (_needsLayout)
      return;
    assert(() {
      if (_needsCompositingBitsUpdate) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary(
            'Tried to paint a RenderObject before its compositing bits were '
            'updated.'
          ),
          describeForError(
            'The following RenderObject was marked as having dirty compositing '
            'bits at the time that it was painted',
          ),
          ErrorDescription(
            'A RenderObject that still has dirty compositing bits cannot be '
            'painted because this indicates that the tree has not yet been '
            'properly configured for creating the layer tree.'
          ),
          ErrorHint(
            'This usually indicates an error in the Flutter framework itself.'
          )
        ]);
      }
      return true;
    }());
    RenderObject debugLastActivePaint;
    assert(() {
      _debugDoingThisPaint = true;
      debugLastActivePaint = _debugActivePaint;
      _debugActivePaint = this;
      assert(!isRepaintBoundary || _layer != null);
      return true;
    }());
    _needsPaint = false;
    try {
      paint(context, offset);
      assert(!_needsLayout); // check that the paint() method didn't mark us dirty again
      assert(!_needsPaint); // check that the paint() method didn't mark us dirty again
    } catch (e, stack) {
      _debugReportException('paint', e, stack);
    }
    assert(() {
      debugPaint(context, offset);
      _debugActivePaint = debugLastActivePaint;
      _debugDoingThisPaint = false;
      return true;
    }());
  }

  /// An estimate of the bounds within which this render object will paint.
  /// Useful for debugging flags such as [debugPaintLayerBordersEnabled].
  ///
  /// These are also the bounds used by [showOnScreen] to make a [RenderObject]
  /// visible on screen.
  Rect get paintBounds;

  /// Override this method to paint debugging information.
  void debugPaint(PaintingContext context, Offset offset) { }

  /// Paint this render object into the given context at the given offset.
  ///
  /// Subclasses should override this method to provide a visual appearance
  /// for themselves. The render object's local coordinate system is
  /// axis-aligned with the coordinate system of the context's canvas and the
  /// render object's local origin (i.e, x=0 and y=0) is placed at the given
  /// offset in the context's canvas.
  ///
  /// Do not call this function directly. If you wish to paint yourself, call
  /// [markNeedsPaint] instead to schedule a call to this function. If you wish
  /// to paint one of your children, call [PaintingContext.paintChild] on the
  /// given `context`.
  ///
  /// When painting one of your children (via a paint child function on the
  /// given context), the current canvas held by the context might change
  /// because draw operations before and after painting children might need to
  /// be recorded on separate compositing layers.
  void paint(PaintingContext context, Offset offset) { }

  /// Applies the transform that would be applied when painting the given child
  /// to the given matrix.
  ///
  /// Used by coordinate conversion functions to translate coordinates local to
  /// one render object into coordinates local to another render object.
  void applyPaintTransform(covariant RenderObject child, Matrix4 transform) {
    assert(child.parent == this);
  }

  /// Applies the paint transform up the tree to `ancestor`.
  ///
  /// Returns a matrix that maps the local paint coordinate system to the
  /// coordinate system of `ancestor`.
  ///
  /// If `ancestor` is null, this method returns a matrix that maps from the
  /// local paint coordinate system to the coordinate system of the
  /// [PipelineOwner.rootNode]. For the render tree owner by the
  /// [RendererBinding] (i.e. for the main render tree displayed on the device)
  /// this means that this method maps to the global coordinate system in
  /// logical pixels. To get physical pixels, use [applyPaintTransform] from the
  /// [RenderView] to further transform the coordinate.
  Matrix4 getTransformTo(RenderObject ancestor) {
    assert(attached);
    if (ancestor == null) {
      final AbstractNode rootNode = owner.rootNode;
      if (rootNode is RenderObject)
        ancestor = rootNode;
    }
    final List<RenderObject> renderers = <RenderObject>[];
    for (RenderObject renderer = this; renderer != ancestor; renderer = renderer.parent) {
      assert(renderer != null); // Failed to find ancestor in parent chain.
      renderers.add(renderer);
    }
    final Matrix4 transform = Matrix4.identity();
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

  // EVENTS

  /// Override this method to handle pointer events that hit this render object.
  @override
  void handleEvent(PointerEvent event, covariant HitTestEntry entry) { }


  // HIT TESTING

  // RenderObject subclasses are expected to have a method like the following
  // (with the signature being whatever passes for coordinates for this
  // particular class):
  //
  // bool hitTest(HitTestResult result, { Offset position }) {
  //   // If the given position is not inside this node, then return false.
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


  /// Returns a human understandable name.
  @override
  String toStringShort() {
    String header = describeIdentity(this);
    if (_relayoutBoundary != null && _relayoutBoundary != this) {
      int count = 1;
      RenderObject target = parent;
      while (target != null && target != _relayoutBoundary) {
        target = target.parent;
        count += 1;
      }
      header += ' relayoutBoundary=up$count';
    }
    if (_needsLayout)
      header += ' NEEDS-LAYOUT';
    if (_needsPaint)
      header += ' NEEDS-PAINT';
    if (_needsCompositingBitsUpdate)
      header += ' NEEDS-COMPOSITING-BITS-UPDATE';
    if (!attached)
      header += ' DETACHED';
    return header;
  }

  @override
  String toString({ DiagnosticLevel minLevel = DiagnosticLevel.debug }) => toStringShort();

  /// Returns a description of the tree rooted at this node.
  /// If the prefix argument is provided, then every line in the output
  /// will be prefixed by that string.
  @override
  String toStringDeep({
    String prefixLineOne = '',
    String prefixOtherLines = '',
    DiagnosticLevel minLevel = DiagnosticLevel.debug,
  }) {
    RenderObject debugPreviousActiveLayout;
    assert(() {
      debugPreviousActiveLayout = _debugActiveLayout;
      _debugActiveLayout = null;
      return true;
    }());
    final String result = super.toStringDeep(
      prefixLineOne: prefixLineOne,
      prefixOtherLines: prefixOtherLines,
      minLevel: minLevel,
    );
    assert(() {
      _debugActiveLayout = debugPreviousActiveLayout;
      return true;
    }());
    return result;
  }

  /// Returns a one-line detailed description of the render object.
  /// This description is often somewhat long.
  ///
  /// This includes the same information for this RenderObject as given by
  /// [toStringDeep], but does not recurse to any children.
  @override
  String toStringShallow({
    String joiner = ', ',
    DiagnosticLevel minLevel = DiagnosticLevel.debug,
  }) {
    RenderObject debugPreviousActiveLayout;
    assert(() {
      debugPreviousActiveLayout = _debugActiveLayout;
      _debugActiveLayout = null;
      return true;
    }());
    final String result = super.toStringShallow(joiner: joiner, minLevel: minLevel);
    assert(() {
      _debugActiveLayout = debugPreviousActiveLayout;
      return true;
    }());
    return result;
  }

  @protected
  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(FlagProperty('needsCompositing', value: _needsCompositing, ifTrue: 'needs compositing'));
    properties.add(DiagnosticsProperty<dynamic>('creator', debugCreator, defaultValue: null, level: DiagnosticLevel.debug));
    properties.add(DiagnosticsProperty<ParentData>('parentData', parentData, tooltip: _debugCanParentUseSize == true ? 'can use size' : null, missingIfNull: true));
    properties.add(DiagnosticsProperty<Constraints>('constraints', constraints, missingIfNull: true));
    // don't access it via the "layer" getter since that's only valid when we don't need paint
    properties.add(DiagnosticsProperty<OffsetLayer>('layer', _layer, defaultValue: null));
    properties.add(DiagnosticsProperty<SemanticsNode>('semantics node', _semantics, defaultValue: null));
    properties.add(FlagProperty(
      'isBlockingSemanticsOfPreviouslyPaintedNodes',
      value: _semanticsConfiguration.isBlockingSemanticsOfPreviouslyPaintedNodes,
      ifTrue: 'blocks semantics of earlier render objects below the common boundary',
    ));
    properties.add(FlagProperty('isSemanticBoundary', value: _semanticsConfiguration.isSemanticBoundary, ifTrue: 'semantic boundary'));
  }

  @override
  List<DiagnosticsNode> debugDescribeChildren() => <DiagnosticsNode>[];

  /// Attempt to make (a portion of) this or a descendant [RenderObject] visible
  /// on screen.
  ///
  /// If `descendant` is provided, that [RenderObject] is made visible. If
  /// `descendant` is omitted, this [RenderObject] is made visible.
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

  /// Adds a debug representation of a [RenderObject] optimized for including in
  /// error messages.
  ///
  /// The default [style] of [DiagnosticsTreeStyle.shallow] ensures that all of
  /// the properties of the render object are included in the error output but
  /// none of the children of the object are.
  ///
  /// You should always include a RenderObject in an error message if it is the
  /// [RenderObject] causing the failure or contract violation of the error.
  DiagnosticsNode describeForError(String name, { DiagnosticsTreeStyle style = DiagnosticsTreeStyle.shallow }) {
    return toDiagnosticsNode(name: name, style: style);
  }
}
```


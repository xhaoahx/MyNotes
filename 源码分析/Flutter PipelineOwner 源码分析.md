# FLutter PipelineOwner 源码分析

```dart
/// 渲染流水线持有者管理渲染流水线
///
/// 渲染流水线持有者提供一个接口来驱动渲染流水线，并存储在渲染流水线的每个阶段中请求访问的渲染对象的状态。
/// 为了冲刷流水线，依次调用以下函数:
///
/// 1. [flushLayout] 更新任何需要计算布局的渲染对象。
///    在这个阶段，每个渲染对象的大小和位置被计算出来。
///    在此阶段，渲染对象可能会标记它们的绘画或合成状态为dirty
/// 2. [flushCompositingBits] 更新任何有dirty compositing bits标记的渲染对象。
///    在这个阶段，每个渲染对象都知道它的子对象是否需要合成。这些信息在绘制阶段用于选择如何实现视觉效果(如
////   裁剪)。如果渲染对象有一个合成的子对象，它需要使用一个[Layer]来创建剪辑，以便剪辑应用到合成的子对
///    象(它将被绘制到自己的[Layer])。
/// 3. [flushPaint] 访问任何需要绘制的渲染对象。
///    在这个阶段，渲染对象有机会将绘制命令记录到[PictureLayer]上，并构建其他合成[Layer]。
/// 4. 最后，如果启用了语义，[flushSemantics]将编译渲染对象的语义。这种语义信息被辅助技术用来提高渲染树
///    的可访问性。
///
/// [RendererBinding]保存屏幕上所有可见的渲染对象。可以创建其他渲染流水线持有者来管理屏幕外的渲染对象。
class PipelineOwner {
  PipelineOwner({
    this.onNeedVisualUpdate,
    this.onSemanticsOwnerCreated,
    this.onSemanticsOwnerDisposed,
  });

  /// 当与此渲染流水线持有者所关联的渲染对象希望更新其视觉外观时调用。
  /// 该函数的典型实现将调度一个任务来刷新流水线的各个阶段。
  /// 这个函数可以被快速连续调用多次。实现应该注意快速丢弃重复的调用。
  final VoidCallback onNeedVisualUpdate;

  /// Called whenever this pipeline owner creates a semantics object.
  ///
  /// Typical implementations will schedule the creation of the initial
  /// semantics tree.
  final VoidCallback onSemanticsOwnerCreated;

  /// Called whenever this pipeline owner disposes its semantics owner.
  ///
  /// Typical implementations will tear down the semantics tree.
  final VoidCallback onSemanticsOwnerDisposed;

  /// 调用 [onNeedVisualUpdate]
  void requestVisualUpdate() {
    if (onNeedVisualUpdate != null)
      onNeedVisualUpdate();
  }

  /// 渲染流水线持有者管理的树根对象
  ///
  /// 这个对象不一定是 [RenderObject].
  AbstractNode get rootNode => _rootNode;
  AbstractNode _rootNode;
  /// 更新rootNode
  /// 之后会调用 rootNode 的 attach 来更新其 owner
  /// 通常，一颗树只有一个 owner  
  set rootNode(AbstractNode value) {
    if (_rootNode == value)
      return;
    _rootNode?.detach();
    _rootNode = value;
    _rootNode?.attach(this);
  }

  List<RenderObject> _nodesNeedingLayout = <RenderObject>[];

  /// 更新所有标记dirty渲染对象的布局信息。
  /// 这个函数是渲染流水线的核心阶段之一。布局信息在绘制之前被全部应用到渲染对象上，以便渲染对象出现在屏幕
  /// 上的更新到最新的位置。
  void flushLayout() {
    if (!kReleaseMode) {
      Timeline.startSync('Layout', arguments: timelineWhitelistArguments);
    }
    try {
      while (_nodesNeedingLayout.isNotEmpty) {
        final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
        /// 清空 _nodesNeedingLayout 列表  
        _nodesNeedingLayout = <RenderObject>[];
        for (RenderObject node in 
             dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)
        ) {
          if (node._needsLayout && node.owner == this)
            node._layoutWithoutResize();
        }
      }
    } finally {
      if (!kReleaseMode) {
        Timeline.finishSync();
      }
    }
  }

  // See [RenderObject.invokeLayoutCallback].
  void _enableMutationsToDirtySubtrees(VoidCallback callback) {
    assert(_debugDoingLayout);
    bool oldState;
    try {
      callback();
    } finally {
    }
  }

  final List<RenderObject> _nodesNeedingCompositingBitsUpdate = <RenderObject>[];
  /// Updates the [RenderObject.needsCompositing] bits.
  ///
  /// 在 [flushLayout] 之后且在 [flushPaint] 之前被调用
  void flushCompositingBits() {
    if (!kReleaseMode) {
      Timeline.startSync('Compositing bits');
    }
    _nodesNeedingCompositingBitsUpdate.sort(
        (RenderObject a, RenderObject b) => a.depth - b.depth);
      
    for (RenderObject node in _nodesNeedingCompositingBitsUpdate) {
      if (node._needsCompositingBitsUpdate && node.owner == this)
        node._updateCompositingBits();
    }
    _nodesNeedingCompositingBitsUpdate.clear();
    if (!kReleaseMode) {
      Timeline.finishSync();
    }
  }

  List<RenderObject> _nodesNeedingPaint = <RenderObject>[];

  /// 更新所有标记dirty的渲染对象的绘制
  /// 调用RenderObject.repaintCompositedChild  
  void flushPaint() {
    if (!kReleaseMode) {
      Timeline.startSync('Paint', arguments: timelineWhitelistArguments);
    }
    try {
      final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
      _nodesNeedingPaint = <RenderObject>[];
      for (RenderObject node in dirtyNodes..sort(
          (RenderObject a, RenderObject b) => b.depth - a.depth)) {
        if (node._needsPaint && node.owner == this) {
          if (node._layer.attached) {
            PaintingContext.repaintCompositedChild(node);
          } else {
            node._skippedPaintingOnLayer();
          }
        }
      }
    } finally {
      if (!kReleaseMode) {
        Timeline.finishSync();
      }
    }
  }

 
  SemanticsOwner get semanticsOwner => _semanticsOwner;
  SemanticsOwner _semanticsOwner;

  int get debugOutstandingSemanticsHandles => _outstandingSemanticsHandles;
  int _outstandingSemanticsHandles = 0;

  SemanticsHandle ensureSemantics({ VoidCallback listener }) {
    _outstandingSemanticsHandles += 1;
    if (_outstandingSemanticsHandles == 1) {
      assert(_semanticsOwner == null);
      _semanticsOwner = SemanticsOwner();
      if (onSemanticsOwnerCreated != null)
        onSemanticsOwnerCreated();
    }
    return SemanticsHandle._(this, listener);
  }

  void _didDisposeSemanticsHandle() {
    assert(_semanticsOwner != null);
    _outstandingSemanticsHandles -= 1;
    if (_outstandingSemanticsHandles == 0) {
      _semanticsOwner.dispose();
      _semanticsOwner = null;
      if (onSemanticsOwnerDisposed != null)
        onSemanticsOwnerDisposed();
    }
  }

  final Set<RenderObject> _nodesNeedingSemantics = <RenderObject>{};

  void flushSemantics() {
    if (_semanticsOwner == null)
      return;
    if (!kReleaseMode) {
      Timeline.startSync('Semantics');
    }
    try {
      final List<RenderObject> nodesToProcess = _nodesNeedingSemantics.toList()
        ..sort((RenderObject a, RenderObject b) => a.depth - b.depth);
      _nodesNeedingSemantics.clear();
      for (RenderObject node in nodesToProcess) {
        if (node._needsSemanticsUpdate && node.owner == this)
          node._updateSemantics();
      }
      _semanticsOwner.sendSemanticsUpdate();
    } finally {
      if (!kReleaseMode) {
        Timeline.finishSync();
      }
    }
  }
}
```


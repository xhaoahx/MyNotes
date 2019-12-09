# Flutter 布局流程

## 请求布局阶段

```dart
  
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

/// RenderPipelineOwner.requestVisualUpdate
/// 调用到了初始化 binding 的时候为其绑定的 RedneringBinding.ensureVisualUpdate 方法
void requestVisualUpdate() {
    if (onNeedVisualUpdate != null)
        onNeedVisualUpdate();
}
```
##  引擎回调阶段
```dart
/// 以下流程同构建流程一样
void ensureVisualUpdate() {
    switch (schedulerPhase) {
        case SchedulerPhase.idle:
        case SchedulerPhase.postFrameCallbacks:
            scheduleFrame();
            return;
        case SchedulerPhase.transientCallbacks:
        case SchedulerPhase.midFrameMicrotasks:
        case SchedulerPhase.persistentCallbacks:
            return;
    }
}

/// 当完成 buildOwner.buildScope(renderViewElement) 调用之后,会调用到 super.drawFrame();
@override
void drawFrame() {
    try {
        // build 阶段
        // buildOwner.buildScope 的参数是 runApp 时初始化的 renderView 对应的 Element
        if (renderViewElement != null)
            buildOwner.buildScope(renderViewElement);
        // 这里才调用到 RenderObject.drawFrame
        super.drawFrame();
        buildOwner.finalizeTree();
    } finally {
        ...
    }
    _needToReportFirstFrame = false;
}

/// 即 RenderingBinding.drawFrame
/// pipelineOwner.flushLayout() 即开启了递归地布局阶段
@protected
void drawFrame() {
    assert(renderView != null);
    pipelineOwner.flushLayout();
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();
    renderView.compositeFrame(); // this sends the bits to the GPU
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
}
```

## 布局阶段

```dart
/// 遍历 pipelineOwner._nodesNeedingLayout 中每一个节点,并调用其 _layoutWithoutResize 方法
/// (_layoutWithoutResize 命名是因为在 markNeedsLayout 时,每次加入 _nodesNeedingLayout 中的节点都是 
/// layoutBoundary.显然,这样的重新布局的边界的大小是无需依赖于子节点而改变的,它只受到父级传入的约束的影响)
void flushLayout() {
    try {
        while (_nodesNeedingLayout.isNotEmpty) {
            final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
            _nodesNeedingLayout = <RenderObject>[];
            for (RenderObject node in dirtyNodes..sort(
                (RenderObject a, RenderObject b) => a.depth - b.depth)
            ) {
                if (node._needsLayout && node.owner == this)
                    node._layoutWithoutResize();
            }
        }
    } finally {
        ...
    }
}

/// 在不改变自身大小的情况布局
void _layoutWithoutResize() 
    ...
    try {
        /// 首先会调用自身的 performLayout 方法
        performLayout();
        markNeedsSemanticsUpdate();
    } catch (e, stack) {
       ...
    }
    _needsLayout = false;
    /// 显然,每次布局完成之后都需要重绘
    markNeedsPaint();
}

/// 完成这个节点的布局。
/// 如果[sizedByParent]为 true，那么这个函数不应该改变这个渲染对象的尺寸。相反，这些工作应该由
/// [performResize]来完成。
/// 如果[sizedByParent]是 false 的，那么这个函数应该改变渲染对象的尺寸，并命令它的子对象进行布局void
/// performLayout 方法应该无条件地调用所有孩子节点的 layout 方法
/// 
/// performLayout 是由子类具体实现的
performLayout();
```

### 举例说明

#### 1.以
# Flutter 框架流程

[TOC]

## 从setState开始


```dart
void setState(VoidCallback fn) {
    ...
    /// 在这里调用传入的 fn    
    final dynamic result = fn() as dynamic;
    /// 直接调用 markNeedsBuild 引发元素树的重建
    _element.markNeedsBuild();
}
```



## markNeedsBuild

源码里有这么一段介绍：

Marks the element as dirty and adds it to the global list of widgets to
rebuild in the next frame.

Since it is inefficient to build an element twice in one frame,
applications and widgets should be structured so as to only mark
widgets dirty during event handlers before the frame begins, not during
the build itself.

标记这个元素为dirty,然后把它添加到一个全局的列表中（这个列表里的widgets会在下一帧进行重建）。
因为在一帧里多次建造一个元素效率低下，所以标记dirty应该在建立帧之前，而不是在构建时进行。



删掉各种assert之后，mark Needs Build只有几行。。。

 ```dart
void markNeedsBuild() {
    /// 如果这个element处于非active（ 通常当页面处于不可见的状态时）的状态，则不重建
    if (!_active)
        return;
	/// 如果已经处于dirty状态了，就返回
    if (dirty)
        return;
    /// 标记dirty
    _dirty = true;
    /// 
    owner.scheduleBuildFor(this);
}
 ```

然后调用到了BuildOwner.scheduleBuildFor

```dart
void scheduleBuildFor(Element element) {
    // element中关于_dirtyElements描述如下：
    // Whether this（element） is in owner._dirtyElements. This is used to know whether we
    // should be adding the element back into the list when it's reactivated.
    // 这个元素element是否在owner的_dirtyElement中。
    // 这决定了在element reactivated 时是否要加入_dirtyElements。
    
    if (element._inDirtyList) {
        _dirtyElementsNeedsResorting = true;
        return;
    }
    // 在 RenderBinding 中 设置buildOwner.onBuildScheduled = _handleBuildScheduled;
    // 在这里被调用
    if (!_scheduledFlushDirtyElements && onBuildScheduled != null) {
        _scheduledFlushDirtyElements = true;
        onBuildScheduled();
    }
    _dirtyElements.add(element);
    element._inDirtyList = true;
    assert(() {
        if (debugPrintScheduleBuildForStacks)
            debugPrint('...dirty list is now: $_dirtyElements');
        return true;
    }());
}
```



## _handleBuildScheduled...

```dart
void _handleBuildScheduled() {
    ensureVisualUpdate();
}

void ensureVisualUpdate() {
    switch (schedulerPhase) {
        // 只有在空闲 或 帧尾回调阶段时才能调用 scheduleFrame();    
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

void scheduleFrame() {
    if (_hasScheduledFrame || !_framesEnabled)
        return;
    ensureFrameCallbacksRegistered();
    window.scheduleFrame();
    _hasScheduledFrame = true;
}

/// 为windows绑定了两个方法
@protected
void ensureFrameCallbacksRegistered() {
    window.onBeginFrame ??= _handleBeginFrame;
    window.onDrawFrame ??= _handleDrawFrame;
}

/// 最后调用到了本地方法
void scheduleFrame() native 'Window_scheduleFrame';

/// 这个方法的表述如下
/// Requests that, at the next appropriate opportunity, the [onBeginFrame]
/// and [onDrawFrame] callbacks be invoked.
///
/// 请求调用这个方法之后，在接下来的某个合适的时机，
/// [onBeginFrame] 和 [onDrawFrame] 回调会被激活
```

## _handleBeginFrame

```dart
void _handleBeginFrame(Duration rawTimeStamp) {
    /// _warmUpFrame在建立第一帧的时候有用，不调用 handleBeginFrame(rawTimeStamp);
    if (_warmUpFrame) {
        _ignoreNextEngineDrawFrame = true;
        return;
    }
    /// 直接调用到
    handleBeginFrame(rawTimeStamp);
}

void handleBeginFrame(Duration rawTimeStamp) {
    Timeline.startSync('Frame', arguments: timelineWhitelistArguments);
    _firstRawTimeStampInEpoch ??= rawTimeStamp;
    _currentFrameTimeStamp = _adjustForEpoch(rawTimeStamp ?? _lastRawTimeStamp);
    if (rawTimeStamp != null)
        _lastRawTimeStamp = rawTimeStamp;

    assert(schedulerPhase == SchedulerPhase.idle);
    /// 取消标记
    _hasScheduledFrame = false;
    try {
        // TRANSIENT FRAME CALLBACKS
        // 瞬态帧回调阶段
        Timeline.startSync('Animate', arguments: timelineWhitelistArguments);
        _schedulerPhase = SchedulerPhase.transientCallbacks;
        final Map<int, _FrameCallbackEntry> callbacks = _transientCallbacks;
        _transientCallbacks = <int, _FrameCallbackEntry>{};
        callbacks.forEach((int id, _FrameCallbackEntry callbackEntry) {
            /// 减小时间复杂度的方式：利用hashSet暂存移除的元素
            if (!_removedIds.contains(id))
                //  逐个调用 _transientCallbacks 
                _invokeFrameCallback(
                callbackEntry.callback, 
                _currentFrameTimeStamp, 
                callbackEntry.debugStack
            );
        });
        _removedIds.clear();
    } finally {
        // 处理微任务队列阶段
        _schedulerPhase = SchedulerPhase.midFrameMicrotasks;
    }
}

void _invokeFrameCallback(
    FrameCallback callback, 
    Duration timeStamp, 
    [ StackTrace callbackStack ]) 
{
    try {
      /// 调用传入的回调  
      callback(timeStamp);
    } catch (exception, exceptionStack) {
      ...
    }
}
```

## _handleDrawFrame
```dart
void _handleDrawFrame() {
    // 同 _handleBeginFrame
    if (_ignoreNextEngineDrawFrame) {
        _ignoreNextEngineDrawFrame = false;
        return;
    }
    
    handleDrawFrame();
}


void handleDrawFrame() {
    assert(_schedulerPhase == SchedulerPhase.midFrameMicrotasks);
    Timeline.finishSync(); // end the "Animate" phase
    try {
        // PERSISTENT FRAME CALLBACKS
        // 永久帧回调阶段
        // 在RendererBinding.initInstances时调用了
        // addPersistentFrameCallback(_handlePersistentFrameCallback);
        // 为_persistentCallbacks添加了drawFrame回调
        _schedulerPhase = SchedulerPhase.persistentCallbacks;
        for (FrameCallback callback in _persistentCallbacks)
            _invokeFrameCallback(callback, _currentFrameTimeStamp);

        // POST-FRAME CALLBACKS
        // 帧尾回调阶段
        _schedulerPhase = SchedulerPhase.postFrameCallbacks;
        final List<FrameCallback> localPostFrameCallbacks =
            List<FrameCallback>.from(_postFrameCallbacks);
        _postFrameCallbacks.clear();
        for (FrameCallback callback in localPostFrameCallbacks)
            _invokeFrameCallback(callback, _currentFrameTimeStamp);
    } finally {
        _schedulerPhase = SchedulerPhase.idle;
        Timeline.finishSync(); // end the Frame
        _currentFrameTimeStamp = null;
        // 完成一帧
    }
}
```



## drawFrame

```dart
/// 由于WidgetBinding重载了这个方法，所以先调用这个

@override
void drawFrame() {
    try {
        // build 阶段
        // buildOwner.buildScope的参数是runapp时初始化的renderView
        if (renderViewElement != null)
            buildOwner.buildScope(renderViewElement);
        // 这里才调用到RenderObject.drawFrame
        super.drawFrame();
        buildOwner.finalizeTree();
    } finally {
      ...
    }
    if (!kReleaseMode) {
        if (_needToReportFirstFrame && _reportFirstFrame) {
            developer.Timeline.instantSync('Widgets built first useful frame');
        }
    }
    _needToReportFirstFrame = false;
}



/// Pump the rendering pipeline to generate a frame.
///
/// This method is called by [handleDrawFrame], which itself is called
/// automatically by the engine when it is time to lay out and paint a frame.
///
/// Each frame consists of the following phases:
///
/// 1. The animation phase: The [handleBeginFrame] method, which is registered
/// with [Window.onBeginFrame], invokes all the transient frame callbacks
/// registered with [scheduleFrameCallback], in registration order. This
/// includes all the [Ticker] instances that are driving [AnimationController]
/// objects, which means all of the active [Animation] objects tick at this
/// point.
///
/// 2. Microtasks: After [handleBeginFrame] returns, any microtasks that got
/// scheduled by transient frame callbacks get to run. This typically includes
/// callbacks for futures from [Ticker]s and [AnimationController]s that
/// completed this frame.
///
/// After [handleBeginFrame], [handleDrawFrame], which is registered with
/// [Window.onDrawFrame], is called, which invokes all the persistent frame
/// callbacks, of which the most notable is this method, [drawFrame], which
/// proceeds as follows:
///
/// 3. The layout phase: All the dirty [RenderObject]s in the system are laid
/// out (see [RenderObject.performLayout]). See [RenderObject.markNeedsLayout]
/// for further details on marking an object dirty for layout.
///
/// 4. The compositing bits phase: The compositing bits on any dirty
/// [RenderObject] objects are updated. See
/// [RenderObject.markNeedsCompositingBitsUpdate].
///
/// 5. The paint phase: All the dirty [RenderObject]s in the system are
/// repainted (see [RenderObject.paint]). This generates the [Layer] tree. See
/// [RenderObject.markNeedsPaint] for further details on marking an object
/// dirty for paint.
///
/// 6. The compositing phase: The layer tree is turned into a [Scene] and
/// sent to the GPU.
///
/// 7. The semantics phase: All the dirty [RenderObject]s in the system have
/// their semantics updated (see [RenderObject.semanticsAnnotator]). This
/// generates the [SemanticsNode] tree. See
/// [RenderObject.markNeedsSemanticsUpdate] for further details on marking an
/// object dirty for semantics.
///
/// For more details on steps 3-7, see [PipelineOwner].
///
/// 8. The finalization phase: After [drawFrame] returns, [handleDrawFrame]
/// then invokes post-frame callbacks (registered with [addPostFrameCallback]).
///
/// Some bindings (for example, the [WidgetsBinding]) add extra steps to this
/// list (for example, see [WidgetsBinding.drawFrame]).
//
// When editing the above, also update widgets/binding.dart's copy.


/// 推动渲染流水线生成一帧。
///
/// 这个方法是由[handleDrawFrame]调用的，在时机合适时，引擎自动调用它完成绘制和布局
///
/// 每一帧包括以下阶段:
///
/// 1.动画阶段:[handleBeginFrame]方法，调用在[scheduleFrameCallback]注册所有回调（以注
/// 册时的顺序。这包括驱动[AnimationController]的所有[Ticker]实例对象。这意味着所有的活动
//  [Animation]对象都在这里开始进行
/// 点。
///
/// 2.微任务:在[handleBeginFrame]之后，得到的任何微任务被临时帧调度回调得到运行。这通常包括
/// 从[Ticker]s和[AnimationController]返回的用于完成这一帧的future。
///
/// 在[handleBeginFrame]、[handleDrawFrame]之后，在[Window.onDrawFrame]注册的永久帧
/// 回调会被调用。其中最值得注意的是这个方法 [drawFrame]，它
/// 的过程如下:
///
/// 3.布局阶段:系统中所有标记为dirty的[RenderObject]进行布局
///
/// 4.The compositing bits phase: The compositing bits on any dirty
/// [RenderObject] objects are updated. See
/// [RenderObject.markNeedsCompositingBitsUpdate].
///
/// 5.绘制阶段:系统中所有标记的dirty的[RenderObject]都将重新绘制，这将生成[Layer]树。
///
/// 6.合成阶段:图层树变成一个[Scene]并发送到GPU。
///
/// 7.The semantics phase: All the dirty [RenderObject]s in the system have
/// their semantics updated (see [RenderObject.semanticsAnnotator]). This
/// generates the [SemanticsNode] tree. See
/// [RenderObject.markNeedsSemanticsUpdate] for further details on marking an
/// object dirty for semantics.
///
/// 有关步骤3-7的更多细节，请参见[PipelineOwner]。
///
/// 8.结束阶段:[drawFrame]返回后，[handleDrawFrame]
/// 然后调用post-frame回调(用[addPostFrameCallback]注册)。
///
/// 一些绑定(例如，[WidgetsBinding])为此添加了额外的步骤列表(例如
/// [WidgetsBinding.drawFrame])。
/// 

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



## buildScope

```dart
void buildScope(Element context, [ VoidCallback callback ]) {
    if (callback == null && _dirtyElements.isEmpty)
        return;
    Timeline.startSync('Build', arguments: timelineWhitelistArguments);
    try {
        _scheduledFlushDirtyElements = true;
        if (callback != null) {
            _dirtyElementsNeedsResorting = false;
            try {
                callback();
            } finally {
            }
        }
        
        /// 按元素的深度，是否标记dirty排序
        /// static int _sort(Element a, Element b) {
        ///    if (a.depth < b.depth)
        ///      return -1;
        ///    if (b.depth < a.depth)
        ///      return 1;
        ///    if (b.dirty && !a.dirty)
        ///      return -1;
        ///    if (a.dirty && !b.dirty)
        ///      return 1;
        ///    return 0;
        ///  }        
        
        _dirtyElements.sort(Element._sort);
        _dirtyElementsNeedsResorting = false;
        int dirtyCount = _dirtyElements.length;
        int index = 0;
        while (index < dirtyCount) {
            try {
                /// 重建dirty的element
                _dirtyElements[index].rebuild();
            } catch (e, stack) {
                ...
            }
            index += 1;
            if (dirtyCount < _dirtyElements.length || _dirtyElementsNeedsResorting) {
                _dirtyElements.sort(Element._sort);
                _dirtyElementsNeedsResorting = false;
                dirtyCount = _dirtyElements.length;
                while (index > 0 && _dirtyElements[index - 1].dirty) {
// It is possible for previously dirty but inactive widgets to move right in the list.
// We therefore have to move the index left in the list to account for this.
// We don't know how many could have moved. However, we do know that the only possible
// change to the list is that nodes that were previously to the left of the index have
// now moved to be to the right of the right-most cleaned node, and we do know that
// all the clean nodes were to the left of the index. So we move the index left
// until just after the right-most clean node.
                    index -= 1;
                }
            }
        }
    } finally {
        for (Element element in _dirtyElements) {
            /// 取消标记
            element._inDirtyList = false;
        }
        /// 清空列表
        _dirtyElements.clear();
        _scheduledFlushDirtyElements = false;
        _dirtyElementsNeedsResorting = null;
        Timeline.finishSync();
    }
}
```



## rebuild

```dart
void rebuild() {
    if (!_active || !_dirty)
        return;
    /// 这里调用到了由子类实现的performRebuild();
    performRebuild();
	/// 断言已经取消标记
    assert(!_dirty);
}
```



## updateChild

```dart
/// Summarizes(四种情况)：
///
/// |                     | **newWidget == null**   | **newWidget != null**           |
/// | :-----------------: | :---------------------  | :-------------------------------|
/// |  **child == null**  |1  Returns null.         |3  Returns new [Element].        |
/// |  **child != null**  |2  Old child is removed, |4 Old child updated if possible, |  
//  |                     |   returns null. 		|  returns child or new [Element].|


@protected
Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
    /// 情况 2
    if (newWidget == null) {
        if (child != null)
            deactivateChild(child);
        return null;
    }
    if (child != null) {
        /// 情况 4
        /// widget 没有改变
        if (child.widget == newWidget) {
            if (child.slot != newSlot)
                /// 只跟新slot
                updateSlotForChild(child, newSlot);
            /// 返回传入的child，而不是重建
            return child;
        }
        if (Widget.canUpdate(child.widget, newWidget)) {
            /// 如果可以更新
            if (child.slot != newSlot)
                updateSlotForChild(child, newSlot);
            child.update(newWidget);
        }
        /// 如果不能更新widget，那么deactivate Element
        deactivateChild(child);
    }
    /// 情况1，3
    return inflateWidget(newWidget, newSlot);
}
```



### deactivateChild

```dart
/// 当旧的widget不为空，而新的widget为空时调用
/// 解除了引用，将自身加入到owner._inactiveElements中
@protected
void deactivateChild(Element child) {
    child._parent = null;
    child.detachRenderObject();
    owner._inactiveElements.add(child); // this eventually calls child.deactivate()
}
```



### updateSlotForChild

```dart

/// Slots的介绍如下：
/// 父级为孩子设定的位置信息（孩子处于child list中的哪个位置）
///
/// 单子elemen必须使用null作为slot
/// slots是孩子在父级children列表中的位置信息

void updateSlotForChild(Element child, dynamic newSlot) {
    void visit(Element element) {
        element._updateSlot(newSlot);
        if (element is! RenderObjectElement)
            /// visitChildren方法是由子类实现的
            /// 多子Element会逐个对其子元素递归调用visit
            /// 单子Element只对唯一的子元素递归调用visit
            element.visitChildren(visit);
    }
    visit(child);
}

void _updateSlot(dynamic newSlot) {
    ...
    _slot = newSlot;
}
```



### update

```dart
void update(covariant Widget newWidget) {
    // This code is hot when hot reloading, so we try to
    // only call _AssertionError._evaluateAssertion once.
    // 重新挂载_widget
    _widget = newWidget;
}
```


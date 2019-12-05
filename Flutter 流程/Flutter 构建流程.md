# Flutter 构建流程

[TOC]

## 请求帧阶段


```dart
/// 我们从setState 开始，分析每次改变 App 状态的时候，整个框架是如何构建出一帧的
void setState(VoidCallback fn) {
    ...
    /// 在这里调用传入的 fn    
    final dynamic result = fn() as dynamic;
    /// 直接调用自身的element.markNeedsBuild 引发元素树的重建
    _element.markNeedsBuild();
}

/// 源码里有这么一段介绍：
/// 标记这个元素为dirty,然后把它添加到一个全局的列表中（这个列表里的widgets会在下一帧进行重建）。
/// 因为在一帧里多次建造一个元素效率低下，所以标记dirty应该在建立帧之前，而不是在构建时进行。
/// 这和 RenderObject 的 markNeedsLayout 和 markNeedsPaint 方法类似，都是用延迟标记的方式来操作
void markNeedsBuild() {
    /// 如果这个element处于非active（通常当页面处于不可见的状态时）的状态，则不重建
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

/// 然后调用到了 BuildOwner.scheduleBuildFor
/// 把自己加入到 BuildOwner 的_dirtyElements列表中
void scheduleBuildFor(Element element) {
    // element中关于_dirtyElements描述如下：
    // 这个元素 element 是否在 owner 的 _dirtyElement 中。
    // 这决定了在 element reactivated 时是否要加入 _dirtyElements。
    // 如果自己已经在列表当中，这说明自己再次被加入到了 dirty 节点列表，需要重新对列表进行排序
    // 设置 _dirtyElementsNeedsResorting 为 true，之后引擎回调之后会对此进行处理
    if (element._inDirtyList) {
        _dirtyElementsNeedsResorting = true;
        return;
    }
    // 在 RenderBinding 中 设置buildOwner.onBuildScheduled = _handleBuildScheduled;
    // 在这里被调用,向引擎调度一帧
    if (!_scheduledFlushDirtyElements && onBuildScheduled != null) {
        _scheduledFlushDirtyElements = true;
        onBuildScheduled();
    }
    _dirtyElements.add(element);
    element._inDirtyList = true;
}

void _handleBuildScheduled() {
    ensureVisualUpdate();
}

void ensureVisualUpdate() {
    switch (schedulerPhase) {
        /// 只有在空闲 或 帧尾回调阶段时才能调用 scheduleFrame();    
        /// 即向引擎调度一整    
        case SchedulerPhase.idle:
        case SchedulerPhase.postFrameCallbacks:
            scheduleFrame();
            return;
        /// 在瞬时回调,微任务队列,永久帧回调阶段,则返回    
        case SchedulerPhase.transientCallbacks:
        case SchedulerPhase.midFrameMicrotasks:
        case SchedulerPhase.persistentCallbacks:
            return;
    }
}

void scheduleFrame() {
    /// 如果已经规划了一帧,返回，这里确保了不会同时调用多帧
    if (_hasScheduledFrame || !_framesEnabled)
        return;
    ensureFrameCallbacksRegistered();
    window.scheduleFrame();
    _hasScheduledFrame = true;
}

/// 这个方法每帧都会被调用，确保为windows绑定了两个方法
@protected
void ensureFrameCallbacksRegistered() {
    window.onBeginFrame ??= _handleBeginFrame;
    window.onDrawFrame ??= _handleDrawFrame;
}

/// 最后调用到了本地方法
/// 这个方法的表述如下
/// 请求调用这个方法之后，在接下来的某个合适的时机，
/// [onBeginFrame] 和 [onDrawFrame] 回调会被引擎激活
void scheduleFrame() native 'Window_scheduleFrame';
```



## 引擎回调阶段


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

    /// 取消已调度帧标记
    _hasScheduledFrame = false;
    try {
        // TRANSIENT FRAME CALLBACKS
        // 瞬时帧回调阶段(即触发动画阶段)
        Timeline.startSync('Animate', arguments: timelineWhitelistArguments);
        _schedulerPhase = SchedulerPhase.transientCallbacks;
        final Map<int, _FrameCallbackEntry> callbacks = _transientCallbacks;
        _transientCallbacks = <int, _FrameCallbackEntry>{};
        callbacks.forEach((int id, _FrameCallbackEntry callbackEntry) {
            /// 减小时间复杂度的方式：利用hashSet暂存移除的元素
            if (!_removedIds.contains(id))
                //  逐个调用 _transientCallbacks 注册的方法
                _invokeFrameCallback(
                    callbackEntry.callback, 
                    _currentFrameTimeStamp, 
                    callbackEntry.debugStack
            	);
        });
        /// 清空列表
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
      /// 调用注册的回调  
      callback(timeStamp);
    } catch (exception, exceptionStack) {
      ...
    }
}

void _handleDrawFrame() {
    // 同 _handleBeginFrame
    if (_ignoreNextEngineDrawFrame) {
        _ignoreNextEngineDrawFrame = false;
        return;
    }
    handleDrawFrame();
}

void handleDrawFrame() {

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

/// 这是在初始化 Binding阶段加入到永久帧回调当中的函数，也就是说，每帧必定调用这个函数
void _handlePersistentFrameCallback(Duration timeStamp) {
    drawFrame();
}

/// 由于WidgetBinding重载了这个方法，所以先调用这个
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
```



## 构建阶段

```dart
/// 建立用于更新小部件树的范围，并调用给定的“回调”(如果有的话)。然后，按深度顺序构建所有使用
/// [scheduleBuildFor]构建标记为dirty的元素。
///
/// 这种机制可以防止构建方法过渡地要求其他构建方法运行，从而可能导致无限循环。
///
/// 脏列表在' callback '返回后处理，使用 [scheduleBuildFor] 按深度顺序构建所有标记为 dirty 的元素。
/// 如果元素在此方法运行时被标记为 dirty，则它们必须比“context”节点更深，并且比此遍历中已经构建完成的任
/// 何节点更深。（这意味着，在一次构建中，如果子节点再次被标记dirty的话，只能是当前构建节点的下层节点。这
/// 会改变 脏节点列表的长度，或者是修改需要重排序的标记为true）
/// 
/// 要在不执行任何其他工作的情况下刷新当前脏节点列表，可以不使用回调调用此函数。这就是框架在
/// [WidgetsBinding.drawFrame]中的每一帧所做的事情。
/// 
/// 一次只能激活一个[buildScope]。
void buildScope(Element context, [ VoidCallback callback ]) {
    /// 如果不需要回调并且标记了 dirty 的元素的列表为空的话，显然不需要重建，直接返回
    if (callback == null && _dirtyElements.isEmpty)
        return;
    try {
        _scheduledFlushDirtyElements = true;
        /// 这里调用了一次传入的回调
        if (callback != null) {
            Element debugPreviousBuildTarget;
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
       
        /// 在这里排序了一次，清除了需要重排序标记
        _dirtyElements.sort(Element._sort);
        _dirtyElementsNeedsResorting = false;
        int dirtyCount = _dirtyElements.length;
        int index = 0;
        
        /// 对列表中的每一个需要重建的 element 调用它的 rebuild 函数
        while (index < dirtyCount) {
            try {
                _dirtyElements[index].rebuild();
            } catch (e, stack) {
                ...
            }
            index += 1;
            /// 列表的长度放生了改变，或是是需要重新排序，则需要再次遍历列表，重新构建element
            if (dirtyCount < _dirtyElements.length || _dirtyElementsNeedsResorting) {
                /// 重新排序
                _dirtyElements.sort(Element._sort);
                _dirtyElementsNeedsResorting = false;
                dirtyCount = _dirtyElements.length;
                /// 找到最右（列表最末）一个不为 dirty 的 element
                while (index > 0 && _dirtyElements[index - 1].dirty) { 
                    // 之前标记为 dirty 但是 inactive 的 widget 可以在列表中右移。
                    // 因此，我们不得不把索引移到列表的左边来解决这个问题。
                    // 我们不知道会有多少widget会移动。
                    // 然而，我们知道，唯一的可能时是先前在左边的节点移动到了已清理的节点之后。
                    index -= 1;
                }
            }
        }
       /// 这时候已经清理了所有的脏节点，然后需要清除 ._inDirtyList 
    } finally {
        /// 清除inDirtyList标记
        for (Element element in _dirtyElements) {
            element._inDirtyList = false;
        }
        /// 清空列表
        _dirtyElements.clear();
        _scheduledFlushDirtyElements = false;
        _dirtyElementsNeedsResorting = null;
    }
}
```



## Element 重建阶段

### 关键函数

以下是构建过程中的几个关键函数，各个 Element 在构建的过程可能会重复地使用它们

#### mount 

```dart
void mount(Element parent, dynamic newSlot) {
    /// 建立对父级的引用关系
    _parent = parent;
    /// 更新插槽
    _slot = newSlot;
    /// 更新深度
    _depth = _parent != null ? _parent.depth + 1 : 1;
    /// 显然，一个 Element 被建立了之后就会处于 active 状态
    _active = true;
    /// 持有 BuildOwner 的引用，便于调用其 scheduleBuildFor 方法
    if (parent != null) // Only assign ownership if the parent is non-null
        _owner = parent.owner;
    // 如果使用了 GlobalKey，则会在全局映射表中注册这个 element
    if (widget.key is GlobalKey) {
        final GlobalKey key = widget.key;
        key._register(this);
    }
    /// 更新遗传信息
    _updateInheritance();
}

/// 获取到父级的遗传映射表
void _updateInheritance() {
    _inheritedWidgets = _parent?._inheritedWidgets;
}
```

#### rebuild

```dart
/// element.rebuild，这个方法不会被重载，并且只做了必要的检查工作，直接调用 performRebuild 
void rebuild() {
    if (!_active || !_dirty)
        return;
    performRebuild();
}
```

#### updateChild

```dart
/// 总结(四种情况):
/// 其中： 
/// child：子Element
/// newWidget 子Element 持有的 widget，即自身持有的 Widget.build 方法的返回值
/// 返回值： 新的 子Element
///
/// |                     | **newWidget == null**   | **newWidget != null**           |
/// | :-----------------: | :---------------------  | :-------------------------------|
/// |  **child == null**  |1  Returns null.         |3  Returns new [Element].        |
/// |  **child != null**  |2  Old child is removed, |4  Old child updated if possible,|  
/// |                     |   returns null. 		|  returns child or new [Element].|
/// 由图：分为四种情况：
/// 1.子 element 为 null，且 newWidget 为 null，返回 null
/// 2.子 element 不为 null，且 newWidget 为 null，即移除子element，返回 null
/// 3.子 element 为 null，且 newWidget 不为 null，新建子 element（通常这是首次构建的情况）
/// 4.子 element 不为 null，且 newWidget 不为 null，如果引用不同的话，尝试使用 newWidget 去更新子
///   element，如果不能更新，则会卸载原有的 element，并创建新的 子element

@protected
Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
    /// 情况 1、2,直接返回 null 或移除子 element
    if (newWidget == null) {
        if (child != null)
            deactivateChild(child);
        return null;
    }
    if (child != null) {
        /// 情况 4
        /// widget 没有改变(引用的对象不变，通常是因为build方法返回了同一个对象)
        if (child.widget == newWidget) {
            if (child.slot != newSlot)
                /// 只更新 slot
                updateSlotForChild(child, newSlot);
            /// 直接返回传入的 child（即原有的子element），没有重建
            return child;
        }
        /// canUpdate判断了运行类型并且Key相同,则可以更新
        if (Widget.canUpdate(child.widget, newWidget)) {
            /// 如果插槽不同，则更新插槽
            /// 通常，单子element 的插槽是null，多子 element 的插槽是前一个元素
            if (child.slot != newSlot)
                updateSlotForChild(child, newSlot);
            /// 调用了由子类实现的update
            child.update(newWidget);
            /// 直接返回更新完成的子 element
            return child;
        }
        /// 如果不能更新widget，那么移除 Element
        deactivateChild(child);
    }
    /// 情况3
    return inflateWidget(newWidget, newSlot);
}
```

#### inflateWidget

```dart
/// 将给定的 widget 扩展成新的 子element
Element inflateWidget(Widget newWidget, dynamic newSlot) {
    final Key key = newWidget.key;
    /// 如果 新widegt 的 key 是 GlobalKey 的话
    if (key is GlobalKey) {
        final Element newChild = _retakeInactiveElement(key, newWidget);
        if (newChild != null) {
            /// 把从别处获取到的 element 作为自己的孩子
            newChild._activateWithParent(this, newSlot);
            /// 继续递归构建
            final Element updatedChild = updateChild(newChild, newWidget, newSlot);
            return updatedChild;
        }
    }
    /// 否则，完全重新建立一个 element 作为子element
    final Element newChild = newWidget.createElement();
    /// 递归地构建子 element，即调用子element.mount方法
    newChild.mount(this, newSlot);
    return newChild;
}
```

#### _retakeInactiveElement

```dart
Element _retakeInactiveElement(GlobalKey key, Widget newWidget) {
    // 这里重新获取的元素的“不活动”可能是前瞻性的:如果我们从当前具有 GlobalKey 的元素中获取元素，那么我
    // 们知道该元素将很快不再具有该元素的子元素。这种假设可能的唯一错误是，GlobalKey被复制了，
    // 我们将在下面使用 _debugTrackElementThatWillNeedToBeRebuiltDueToGlobalKeyShenanigans
    // 调用来跟踪它
    
    // 如果 key 没有对应的 element，返回 null
    final Element element = key._currentElement;
    if (element == null)
        return null;
    // 如果不能用给定的 Widget 来更新获取到的 element，返回 null
    if (!Widget.canUpdate(element.widget, newWidget))
        return null;
    // 运行到这里意味着可以用新给定 widget 来更新 element（这个element通常来源于树中的其它位置）
    // 那么把他移植过来
    final Element parent = element._parent;
    // 把 element 从它的父节点移除
    if (parent != null) {
        parent.forgetChild(element);
        parent.deactivateChild(element);
    }
    // 把 elment 从脏节点列表中移除，这回导致脏节点列表长度改变，需要重新排序
    owner._inactiveElements.remove(element);
    return element;
}
```

#### deactivate

```dart
void deactivate() {
    /// 解除自己的依赖
    if (_dependencies != null && _dependencies.isNotEmpty) {
        for (InheritedElement dependency in _dependencies)
            dependency._dependents.remove(this);
    }
    _inheritedWidgets = null;
    // 最终把 _active 标记置为 false
    _active = false;
}
```

#### deactivateChild

```dart
/// 当旧的子 widget 不为空，而新的子 widget 为空时调用
@protected
void deactivateChild(Element child) {
    // 解除孩子的引用
    child._parent = null;
    // 解除孩子的渲染对象
    child.detachRenderObject();
    // 把自身添加到 BuildOwner _inactiveElements列表中，
    // 最终触发了 child.deactivate(),并递归地调用 child 的子树的deactivate()
    owner._inactiveElements.add(child);
}

/// 移除所有孩子的 RenderObject
/// 有 renderObejectElement 通常会重载这一方法
void detachRenderObject() {
    visitChildren((Element child) {
        child.detachRenderObject();
    });
    _slot = null;
}
```

#### updateSlotForChild

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
            /// visitChildren 方法是由子类实现的
            /// 多子 Element 会逐个对其子元素递归调用 visit
            /// 单子 Element 只对唯一的子元素递归调用 visit
            element.visitChildren(visit);
    }
    visit(child);
}

void _updateSlot(dynamic newSlot) {
    ...
    _slot = newSlot;
}
```



### 举例说明

注意到 performRebuild 会由子类 element 各自实现，以下举例说明

#### 1.以ComponentElement为例


```dart
/// 1.mount
void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _firstBuild();
    assert(_child != null);
}

/// 2.performRebuild
void performRebuild() {
    Widget built;
    try {
        /// 这里调用到了 ComponentElement.build 方法,这个方法由它的子类实现
        /// 这里以 StatelessComponent 为例：
        /// “Widget build() => widget.build(this);”
        /// 即把自身作为 Buildcontext 传给 Widget，调用它的 build方法，得到子 widget，并试图用它来
        /// 更新子 element
        built = build();
    } catch (e, stack) {
        ... 
    } finally {
        _dirty = false;
    }
    /// 试图用新产生的 widget 去更新 _child
    try {
        /// 直接把 updateChild 方法的返回值作为 child
        /// 注意：这里的构建是递归进行的，所以，这个赋值会在函数返回后进行
        _child = updateChild(_child, built, slot);
    } catch (e, stack) {
        ...  
    }
}

/// 3.update
/// 其 update 方法由子类实现
/// StatelessElement 中：
void update(StatelessWidget newWidget) {
    _dirty = true;
    rebuild();
}
/// StatefulElement 中：
@override
void update(StatefulWidget newWidget) {
    super.update(newWidget);
    final StatefulWidget oldWidget = _state._widget;
    _dirty = true;
    _state._widget = widget;
    try {
       final dynamic debugCheckForReturnedFuture = 
            _state.didUpdateWidget(oldWidget) as dynamic;
    } finally {
       ...
    }
    rebuild();
}
```

#### 2.以RenderObjectElement为例

```dart
/// 1.mount
void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    /// 调用 widget 的 createRenderObject 来得到并持有一个 RenderObject，
    _renderObject = widget.createRenderObject(this);
    /// 将得到的 renderObeject 挂载到 renderObeject 树上
    attachRenderObject(newSlot);
    _dirty = false;
}


/// 2.performRebuild
/// 只调用了 updateRenderObject 用新的 Widget 的信息来更新持有的 renderObject
/// 注意：这里并没有重新创建 renderObject，而是把引用传递过去，widget 应该只修改其值域
void performRebuild() {
    widget.updateRenderObject(this, renderObject);
    _dirty = false;
}

/// 3.update
/// 只用更新的完成的Widget来更新持有的渲染对象的信息
void update(covariant RenderObjectWidget newWidget) {
    super.update(newWidget);
    widget.updateRenderObject(this, renderObject);
    _dirty = false;
}
```

#### 3.以ProxyElement为例

```dart
/// 1.mount 同 ComponentElement
/// 2.performRebuild 同 ComponentElement
/// 3.update
void update(ProxyWidget newWidget) {
    final ProxyWidget oldWidget = widget;;
    super.update(newWidget);
    updated(oldWidget);
    _dirty = true;
    rebuild();
}

/// 通知所有的监听者
void updated(covariant ProxyWidget oldWidget) {
    notifyClients(oldWidget);
}
```



## 布局和绘制阶段

这里给出布局和绘制的简要描述，后文将详细分析布局和绘制流程

```dart
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
/// 8.结束阶段:[drawFrame]返回后，[handleDrawFrame]
/// 然后调用post-frame回调(用[addPostFrameCallback]注册)。
///
/// 一些绑定(例如，[WidgetsBinding])为此添加了额外的步骤列表(例如
/// [WidgetsBinding.drawFrame])。
/// 

@protected
void drawFrame() {
    pipelineOwner.flushLayout();
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();
    renderView.compositeFrame(); // this sends the bits to the GPU
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
}
```



# Flutter 手势流程

## HitTestResult

```dart
/// 点击测试结果
class HitTestResult {
  HitTestResult()
     : _path = <HitTestEntry>[],
       _transforms = Queue<Matrix4>();

  /// 包装‘result’(通常是[HitTestResult]的一个子树)来创建一个通用的[HitTestResult]。
  ///
  /// 添加到返回的[HitTestEntry]中的[HitTestEntry]也被添加到包装的' result '中
  /// (两者共享相同的底层数据结构来存储[HitTestEntry])。
  HitTestResult.wrap(HitTestResult result)
     : _path = result._path,
       _transforms = result._transforms;

  /// 在 hit 测试期间记录的[HitTestEntry]对象的不可修改列表。
  ///
  /// 路径中的第一个条目是最具体的，通常是被测试的树的叶子上的那个。事件传播从最具体的(例如：(第一个)
  /// [HitTestEntry]并按顺序通过路径。
  Iterable<HitTestEntry> get path => _path;
  final List<HitTestEntry> _path;

  final Queue<Matrix4> _transforms;

  /// Add a [HitTestEntry] to the path.
  ///
  /// The new entry is added at the end of the path, which means entries should
  /// be added in order from most specific to least specific, typically during an
  /// upward walk of the tree being hit tested.
  void add(HitTestEntry entry) {
    assert(entry._transform == null);
    entry._transform = _transforms.isEmpty ? null : _transforms.last;
    _path.add(entry);
  }

  /// Pushes a new transform matrix that is to be applied to all future
  /// [HitTestEntry]s added via [add] until it is removed via [popTransform].
  ///
  /// This method is only to be used by subclasses, which must provide
  /// coordinate space specific public wrappers around this function for their
  /// users (see [BoxHitTestResult.addWithPaintTransform] for such an example).
  ///
  /// The provided `transform` matrix should describe how to transform
  /// [PointerEvent]s from the coordinate space of the method caller to the
  /// coordinate space of its children. In most cases `transform` is derived
  /// from running the inverted result of [RenderObject.applyPaintTransform]
  /// through [PointerEvent.removePerspectiveTransform] to remove
  /// the perspective component.
  ///
  /// [HitTestable]s need to call this method indirectly through a convenience
  /// method defined on a subclass before hit testing a child that does not
  /// have the same origin as the parent. After hit testing the child,
  /// [popTransform] has to be called to remove the child-specific `transform`.
  ///
  /// See also:
  ///  * [BoxHitTestResult.addWithPaintTransform], which is a public wrapper
  ///    around this function for hit testing on [RenderBox]s.
  ///  * [SliverHitTestResult.addWithAxisOffset], which is a public wrapper
  ///    around this function for hit testing on [RenderSlivers]s.
  @protected
  void pushTransform(Matrix4 transform) {
    _transforms.add(
        _transforms.isEmpty ? transform : (
            transform * _transforms.last as Matrix4
        )
    );
  }

  /// Removes the last transform added via [pushTransform].
  ///
  /// This method is only to be used by subclasses, which must provide
  /// coordinate space specific public wrappers around this function for their
  /// users (see [BoxHitTestResult.addWithPaintTransform] for such an example).
  ///
  /// This method must be called after hit testing is done on a child that
  /// required a call to [pushTransform].
  ///
  /// See also:
  ///
  ///  * [pushTransform], which describes the use case of this function pair in
  ///    more details.
  @protected
  void popTransform() {
    assert(_transforms.isNotEmpty);
    _transforms.removeLast();
  } 
}
```

## HitTestEntry

```dart
/// 在一次特定[HitTestTarget]的命中测试中收集的数据。
///
/// 继承该对象，将额外的信息从 hitTest 阶段传递到事件传播阶段。
class HitTestEntry {
  HitTestEntry(this.target);

  /// 点击测试过程中遇见的 [HitTestTarget]
  /// 注意：“abstract class RenderObject extends AbstractNode implements HitTestTarget”
  /// 即每个 RenderObject 都是一个 点击测试目标，都具有对应的 hitTest，handleEvent 方法  
  final HitTestTarget target;

  Matrix4 get transform => _transform;
  Matrix4 _transform;
}
```



## 流程分析

```dart
/// 在 GestureBinding 的初始化中，绑定了如下方法
@override
void initInstances() {
   ...
   window.onPointerDataPacket = _handlePointerDataPacket;
}

/// Window.onPointerDataPacket描述如下：
/// “当指针数据可用时调用的回调。框架在设置回调的同一Zone中调用这个回调”
/// 也就是说，每当由点击事件发生时，native会将数据打包，并回调Window.onPointerDataPacket，即：

void _handlePointerDataPacket(ui.PointerDataPacket packet) {
    // 我们将指针数据转换成逻辑像素，这样就可以以一种与设备无关的方式定义触摸溅射
    _pendingPointerEvents.addAll(PointerEventConverter.expand(
        packet.data, 
        window.devicePixelRatio
    ));
    if (!locked)
        _flushPointerEventQueue();
}

/// 首先调用了 PointerEventConverter.expand，生成了一个指针事件列表

/// 将给定的指针数据包展开为一系列框架指针事件。
///
/// ‘devicePixelRatio’参数(通常给出的值来自[dart:ui.Window.devicePixelRatio])用于将输入的数据从
/// 物理坐标转换为逻辑像素
static Iterable<PointerEvent> expand(
    Iterable<ui.PointerData> data, double devicePixelRatio
) sync* {
    for (ui.PointerData datum in data) {
        final Offset position = Offset(
            datum.physicalX, datum.physicalY
        ) / devicePixelRatio;
        final Offset delta = Offset(
            datum.physicalDeltaX, datum.physicalDeltaY
        ) / devicePixelRatio;

        final double radiusMinor = _toLogicalPixels(datum.radiusMinor, devicePixelRatio);
        final double radiusMajor = _toLogicalPixels(datum.radiusMajor, devicePixelRatio);
        final double radiusMin = _toLogicalPixels(datum.radiusMin, devicePixelRatio);
        final double radiusMax = _toLogicalPixels(datum.radiusMax, devicePixelRatio);
        final Duration timeStamp = datum.timeStamp;
        final PointerDeviceKind kind = datum.kind;
        /// 以下，将来自 naive 根据类型来转换成对应的指针事件，并组成一个列表  
        if (datum.signalKind == null || datum.signalKind == ui.PointerSignalKind.none) {
            switch (datum.change) {
                case ui.PointerChange.add:
                    yield PointerAddedEvent(
                        timeStamp: timeStamp,
                        kind: kind,
                        device: datum.device,
                        position: position,
                        obscured: datum.obscured,
                        pressureMin: datum.pressureMin,
                        pressureMax: datum.pressureMax,
                        distance: datum.distance,
                        distanceMax: datum.distanceMax,
                        radiusMin: radiusMin,
                        radiusMax: radiusMax,
                        orientation: datum.orientation,
                        tilt: datum.tilt,
                    );
                    break;
                case ui.PointerChange.hover:
                    yield PointerHoverEvent(
                        ...
                    );
                    break;
                case ui.PointerChange.down:
                    yield PointerDownEvent(
                        ...
                    );
                    break;
                case ui.PointerChange.move:
                    yield PointerMoveEvent(
                        ...
                    );
                    break;
                case ui.PointerChange.up:
                    yield PointerUpEvent(
                        ...
                    );
                    break;
                case ui.PointerChange.cancel:
                    yield PointerCancelEvent(
                        ...
                    );
                    break;
                case ui.PointerChange.remove:
                    yield PointerRemovedEvent(
                        ...
                    );
                    break;
            }
        } else {
            switch (datum.signalKind) {
                case ui.PointerSignalKind.scroll:
                    final Offset scrollDelta =
                        Offset(
                        datum.scrollDeltaX, datum.scrollDeltaY
                    ) / devicePixelRatio;
                    yield PointerScrollEvent(
                        ...
                    );
                    break;
                case ui.PointerSignalKind.none:
                    assert(false);
                    break;
                case ui.PointerSignalKind.unknown:
                    // Ignore unknown signals.
                    break;
            }
        }
    }
}


/// 随后调用到  _flushPointerEventQueue，其中 _pendingPointerEvents 是一个队列
/// 每次出列一个事件，并调用 _handlePointerEvent
///
/// 其中关于 hitTests 的描述如下：
/// 当前处于关闭状态的所有指针的状态。悬停指针的状态不会被跟踪，因为这需要对每一帧进行点击测试。
void _flushPointerEventQueue() {
    assert(!locked);
    while (_pendingPointerEvents.isNotEmpty)
        _handlePointerEvent(_pendingPointerEvents.removeFirst());
}，

/// 处理每个指针事件    
void _handlePointerEvent(PointerEvent event) {
    HitTestResult hitTestResult;
    /// 如果是指针落下事件：
    if (event is PointerDownEvent || event is PointerSignalEvent) {
        hitTestResult = HitTestResult();
        /// 调用自身的 hitTest 方法，
        hitTest(hitTestResult, event.position);
        /// 记录结果
        if (event is PointerDownEvent) {
            _hitTests[event.pointer] = hitTestResult;
        }
    } else if (event is PointerUpEvent || event is PointerCancelEvent) {
        /// 移除结果
        hitTestResult = _hitTests.remove(event.pointer);
    } else if (event.down) {
        // 因为与指针事件(如PointerMoveEvents)应派遣他们最初的PointerDownEvent是相同的地方,在发现
        // 指针落下的我们会时候重用的路径,而不是每次我们得到这样一个事件都做点击检测。
        hitTestResult = _hitTests[event.pointer];
    }

    /// 如果最终得到了一个结果的话，派发事件
    if (hitTestResult != null ||
        event is PointerHoverEvent ||
        event is PointerAddedEvent ||
        event is PointerRemovedEvent) {
        dispatchEvent(event, hitTestResult);
    }
}

/// GestureBinding 实现的 hitTest 方法如下，事实上，这个方法被 RenderBinding重载了
void hitTest(HitTestResult result, Offset position) {
    result.add(HitTestEntry(this));
}

/// 重载如下：
@override
void hitTest(HitTestResult result, Offset position) {
    /// 这里会调用到 renderView.hitTest （渲染对象树的根节点）方法
    /// renderView.hitTest 会调用其子树的 hitTest 方法，这意味 hitTest 是自顶向下递归进行的
    renderView.hitTest(result, position: position);
    super.hitTest(result, position);
}

/// RenderView.hitTest
bool hitTest(HitTestResult result, { Offset position }) {
    /// 调用孩子节点的 hitTest
    if (child != null)
        child.hitTest(BoxHitTestResult.wrap(result), position: position);
    /// 把自身添加到点击测试结果中
    result.add(HitTestEntry(this));
    return true;
}

/// 接下来的流程是由 child.hitTest的实现决定的
/// 总之，最后会得到一个 HitTestEntry 列表，其中的每个 HitTestEntry 里包含一个 RenderObejct  
```




# Flutter 手势流程



## PointEvent

```dart
abstract class PointerEvent extends Diagnosticable {
  const PointerEvent({
    this.timeStamp = Duration.zero,
    this.pointer = 0,
    this.kind = PointerDeviceKind.touch,
    this.device = 0,
    this.position = Offset.zero,
    Offset localPosition,
    this.delta = Offset.zero,
    Offset localDelta,
    this.buttons = 0,
    this.down = false,
    this.obscured = false,
    this.pressure = 1.0,
    this.pressureMin = 1.0,
    this.pressureMax = 1.0,
    this.distance = 0.0,
    this.distanceMax = 0.0,
    this.size = 0.0,
    this.radiusMajor = 0.0,
    this.radiusMinor = 0.0,
    this.radiusMin = 0.0,
    this.radiusMax = 0.0,
    this.orientation = 0.0,
    this.tilt = 0.0,
    this.platformData = 0,
    this.synthesized = false,
    this.transform,
    this.original,
  }) : localPosition = localPosition ?? position,
       localDelta = localDelta ?? delta;


  /// 事件的派发时间，相对于任意时间轴。
  final Duration timeStamp;

  /// 指针的唯一标识符，不能重复使用。每个新指针落下的时候更新。
  final int pointer;

  /// 为其生成事件的输入设备的类型。
  final PointerDeviceKind kind;

  /// 指向设备的唯一标识符，跨交互重用。
  final int device;

  /// 指针位置的坐标，以全局坐标空间中的逻辑像素为单位。
  final Offset position;

  /// 根据[transform]将[position]变换为事件接收者的局部坐标系统。
  ///
  /// 如果未转换此事件，则按原样返回[position]。
  final Offset localPosition;

  /// 指针自上次[PointerMoveEvent]或[PointerHoverEvent]后移动的逻辑像素距离。
  final Offset delta;

  /// 根据[transform]将[delta]变换为事件接收者的局部坐标系统。
  /// 如果这个事件没有被转换，[delta]将按原样返回。
  final Offset localDelta;

  /// 使用*按钮常量的位字段，如[kPrimaryMouseButton]， [kSecondaryStylusButton]等。
  ///
  /// For example, if this has the value 6 and the
  /// [kind] is [PointerDeviceKind.invertedStylus], then this indicates an
  /// upside-down stylus with both its primary and secondary buttons pressed.
  final int buttons;

  /// 指针是否落下的。
  ///
  /// 对于触摸或者手写笔，这意味着它(手指、笔)与输入表面接触。对鼠标来说，这意味着按下了一个按钮。
  final bool down;

  /// 触摸的压力
  final double pressure;

  /// 触摸的最小压力
  final double pressureMin;

  /// 触摸的最大压力
  final double pressureMax;

  /// 检测到的物体到输入表面的距离。
  ///
  /// 例如，这个值可以是手写笔或手指到触摸屏的距离，单位可以是任意的(不一定是线性的)。
  /// 如果指针落下，它的值是 0.0。
  final double distance;

  /// 指针的最小距离
  double get distanceMin => 0.0;

  /// 指针的最大距离
  final double distanceMax;

  /// 按下屏幕的面积。
  ///
  /// 这个值是按比例缩小的范围在0和1之间。它可以用来确定触摸事件的大小。这个值只设置在Android上,是设备特
  /// 定的近似可检测范围内的值。
  /// 例如,0.1可能意味着手指尖的触碰,0.2是满手指,和0.3整个手掌。
  ///
  /// 因为这个值使用特定于设备的范围并且未被未校准,它的使用是有限的,主要是为了能够在[AndroidView]上保留
  /// 重建原始指针事件
  final double size;

  /// 接触椭圆沿主轴的半径，以逻辑像素为单位。
  final double radiusMajor;

  /// 接触椭圆沿副轴的半径，以逻辑像素为单位。
  final double radiusMinor;

  /// 该指针可以报告的[radiusMajor]和[radiusMinor]的最大值，以逻辑像素为单位。
  final double radiusMin;

  /// 该指针可以报告的[radiusMajor]和[radiusMinor]的最小值，以逻辑像素为单位。
  final double radiusMax;

  final double orientation;

  final double tilt;

  /// 与事件关联的不透明的特定于平台的数据。
  final int platformData;

  /// 设置事件是否由 Flutter 合成。
  /// 我们偶尔会合成不完全翻译的[ui.PointerData]，以覆盖指针行为中的小的跨操作系统差异。
  /// 例如，在结束事件上，当发出结束事件时，Android总是删除报告间隔期间发生的任何位置更改。
  /// 在iOS系统中，与之前的移动事件相比，微小的不正确的位置变化可以在结束事件上报告。
  /// 我们合成一个[PointerEvent]来覆盖在那种情况下两个事件之间的差异。
  final bool synthesized;

  /// 用于将此事件从全局坐标空间转换为事件接收者坐标空间的转换。
  /// 这个值影响[localPosition]和[localDelta]的内容。如果该值为null，则将其视为恒等转换。
  ///
  /// 与绘制的变换不同，这种变换通常不包含任何“透视图”组件。
  /// 这意味着矩阵的第三行和第三列应该等于“0,0,1,0”。
  /// 这确保[localPosition]描述了事件接收器的本地坐标系统中用户实际接触屏幕的位置。
  final Matrix4 transform;

  /// 应用任何[transform]之前的原始未转换[PointerEvent]。
  ///
  /// 如果[transform]为空，或者是恒等变换，这个值为空。
  ///
  /// 当位于不同坐标空间的多个事件接收者接收到一个事件时，它们都接收转换到其本地坐标空间的事件。可以使用
  /// [original]属性确定所有转换后的事件实际上是否源自同一指针交互。
  final PointerEvent original;

  /// 将事件从全局坐标空间转换为事件接收器的坐标空间。
  ///
  /// 事件接收者的坐标空间由“transform”来描述。“transform”的空值被视为恒等变换。
  ///
  /// 例如，如果转换对事件没有影响，则该方法可以返回相同的对象实例。
  ///
  /// 变换不是可交换的。如果在一个[transform]值不为null的[PointerEvent]上调用此方法，则提供的' 
  /// transform '将覆盖该值。
  PointerEvent transformed(Matrix4 transform);

  /// 返回将“position”转换为“transform”所描述的坐标系统。
  ///
  /// The z-value of `position` is assumed to be 0.0. If `transform` is null,
  /// `position` is returned as-is.
  static Offset transformPosition(Matrix4 transform, Offset position) {
    if (transform == null) {
      return position;
    }
    final Vector3 position3 = Vector3(position.dx, position.dy, 0.0);
    final Vector3 transformed3 = transform.perspectiveTransform(position3);
    return Offset(transformed3.x, transformed3.y);
  }

  /// Transforms `untransformedDelta` into the coordinate system described by
  /// `transform`.
  ///
  /// It uses the provided `untransformedEndPosition` and
  /// `transformedEndPosition` of the provided delta to increase accuracy.
  ///
  /// If `transform` is null, `untransformedDelta` is returned.
  static Offset transformDeltaViaPositions({
    @required Offset untransformedEndPosition,
    Offset transformedEndPosition,
    @required Offset untransformedDelta,
    @required Matrix4 transform,
  }) {
    if (transform == null) {
      return untransformedDelta;
    }
    // We could transform the delta directly with the transformation matrix.
    // While that is mathematically equivalent, in practice we are seeing a
    // greater precision error with that approach. Instead, we are transforming
    // start and end point of the delta separately and calculate the delta in
    // the new space for greater accuracy.
    transformedEndPosition ??= transformPosition(transform, untransformedEndPosition);
    final Offset transformedStartPosition = transformPosition(transform, untransformedEndPosition - untransformedDelta);
    return transformedEndPosition - transformedStartPosition;
  }

  /// 从变换矩阵中移除透视，即 z 轴的值
  static Matrix4 removePerspectiveTransform(Matrix4 transform) {
    final Vector4 vector = Vector4(0, 0, 1, 0);
    return transform.clone()
      ..setColumn(2, vector)
      ..setRow(2, vector);
  }
}
```



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



## GestureArenaMember

```dart
/// 表示一个参加手势竞技场的对象，GestureRecognizer 是这个类的子类
///
/// 接收来自GestureArena的回调，当对象在手势竞技中获胜或失败时通知它。不管是什么原因导致竞技场事件被解
/// 决，每个竞技场都会调用该成员的一个[acceptGesture]或[rejectGesture]。
/// 例如，如果一个成员解决竞技场事件本身，该成员仍然会收到一个[acceptGesture]回调.
abstract class GestureArenaMember {
  /// 被接受
  void acceptGesture(int pointer);

  /// 被拒绝
  void rejectGesture(int pointer);
}
```



## GestureArenaEntry

```dart
/// 用于向竞技场传递信息的接口
///
/// 一个给定的[GestureArenaMember]可以在多个带有不同指针id的领域中有多个entry。
class GestureArenaEntry {
  GestureArenaEntry._(this._arena, this._pointer, this._member);

  final GestureArenaManager _arena;
  final int _pointer;
  final GestureArenaMember _member;

  /// Call this member to claim victory (with accepted) or admit defeat (with rejected).
  ///
  /// It's fine to attempt to resolve a gesture recognizer for an arena that is
  /// already resolved.
  void resolve(GestureDisposition disposition) {
    _arena._resolve(_pointer, _member, disposition);
  }
}
```



## _GestureArena

```dart
class _GestureArena {
  /// 需要竞争的成员
  final List<GestureArenaMember> members = <GestureArenaMember>[];
  bool isOpen = true;
  bool isHeld = false;
  bool hasPendingSweep = false;

  /// 如果一个成员试图在竞技场仍然开放的时候获胜，它将成为“渴望胜利者”。当我们向新的参与者关闭竞技场时，我
  /// 们会寻找唯一一个渴望胜利者，如果有的话，它便是获胜者。
  GestureArenaMember eagerWinner;

  void add(GestureArenaMember member) {
    assert(isOpen);
    members.add(member);
  }
}
```



## GestureArenaManager

```dart
/// 第一个被接受或者最后一个不被拒绝的成员即是获胜者
///
class GestureArenaManager {
  final Map<int, _GestureArena> _arenas = <int, _GestureArena>{};

  /// 添加一个新成员 (例如 gesture recognizer)到竞技场内
  GestureArenaEntry add(int pointer, GestureArenaMember member) {
    final _GestureArena state = _arenas.putIfAbsent(pointer, () {
      return _GestureArena();
    });
    state.add(member);
    return GestureArenaEntry._(this, pointer, member);
  }

  /// 阻止新成员的加入
  void close(int pointer) {
    final _GestureArena state = _arenas[pointer];
    if (state == null)
      return; // This arena either never existed or has been resolved.
    state.isOpen = false;
    _tryToResolveArena(pointer, state);
  }

  /// 强制决出胜利者，即把胜利给与第一个成员
  ///
  /// 通常在[PointerUpEvent]的所有其他处理完成之后进行清扫。
  /// 它确保了多种被动手势不会造成阻碍用户与应用程序交互的僵局。
  ///
  /// Recognizers that wish to delay resolving an arena past [PointerUpEvent]
  /// should call [hold] to delay sweep until [release] is called.
  ///
  void sweep(int pointer) {
    final _GestureArena state = _arenas[pointer];
    if (state == null)
      return; // This arena either never existed or has been resolved.
    if (state.isHeld) {
      state.hasPendingSweep = true;
      return; // This arena is being held for a long-lived member.
    }
    _arenas.remove(pointer);
    if (state.members.isNotEmpty) {
      // 第一个成原理获胜
      state.members.first.acceptGesture(pointer);
      // 其他的成员被拒绝
      for (int i = 1; i < state.members.length; i++)
        state.members[i].rejectGesture(pointer);
    }
  }

  /// 阻止竞技场被清扫
  ///
  /// 通常，获胜者是在[sweep]处理完所有其他[PointerUpEvent]后在一个竞技场中选择的。
  //  如果一个手势识别器想要从[PointerUpEvent]中延迟解析一个竞技场，该识别器可以使用此功能[hold]
  /// 竞技场开启。要释放这样的保持状态并让竞技场事件被解决，调用[release]
  ///
  void hold(int pointer) {
    final _GestureArena state = _arenas[pointer];
    if (state == null)
      return; // This arena either never existed or has been resolved.
    state.isHeld = true;
  }

  /// 释放保持状态，允许竞技场被清扫
  ///
  void release(int pointer) {
    final _GestureArena state = _arenas[pointer];
    if (state == null)
      return;
    state.isHeld = false;
    if (state.hasPendingSweep)
      sweep(pointer);
  }

  /// 拒绝或者接受手势识别器
  ///
  /// 被 [add] 方法添加的 成员调用 [GestureArenaEntry.resolve] 而调用 
  void _resolve(int pointer, GestureArenaMember member, GestureDisposition disposition) {
    final _GestureArena state = _arenas[pointer];
    if (state == null)
      return; // This arena has already resolved.
    if (disposition == GestureDisposition.rejected) {
      state.members.remove(member);
      member.rejectGesture(pointer);
      if (!state.isOpen)
        _tryToResolveArena(pointer, state);
    } else {
      if (state.isOpen) {
        state.eagerWinner ??= member;
      } else {
        _resolveInFavorOf(pointer, state, member);
      }
    }
  }

  void _tryToResolveArena(int pointer, _GestureArena state) {
    if (state.members.length == 1) {
      scheduleMicrotask(() => _resolveByDefault(pointer, state));
    } else if (state.members.isEmpty) {
      _arenas.remove(pointer);
    } else if (state.eagerWinner != null) {
      _resolveInFavorOf(pointer, state, state.eagerWinner);
    }
  }

  /// 有一个渴望获胜者时调用，调用它的acceptGesture方法  
  void _resolveByDefault(int pointer, _GestureArena state) {
    if (!_arenas.containsKey(pointer))
      return; // Already resolved earlier.
    final List<GestureArenaMember> members = state.members;
    _arenas.remove(pointer);
    state.members.first.acceptGesture(pointer);
  }

  /// 在没有渴望获胜者的时候调用，拒绝除了 member 以外的所有成员
  void _resolveInFavorOf(int pointer, _GestureArena state, GestureArenaMember member) {
    _arenas.remove(pointer);
    for (GestureArenaMember rejectedMember in state.members) {
      if (rejectedMember != member)
        rejectedMember.rejectGesture(pointer);
    }
    member.acceptGesture(pointer);
  }

}
```



## 流程分析

### 引擎回调阶段

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
```
### 点击测试阶段
```dart
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
    /// 注意：因为递归的缘故，自身会被最后添加到列表当中
    result.add(HitTestEntry(this));
    return true;
}

/// 接下来的流程是由 child.hitTest的实现决定的
/// 总之，最后会得到一个 HitTestEntry 列表，其中的每个 HitTestEntry 里包含一个 RenderObejct
/// 注意：对于子类的 hitTest实现来说，通常是采用递归的方式进行的，即先调用child的hitTest，随后再将自身
/// 加入到列表当中，这样会导致树中最下层的RenderObject（也就是距离屏幕最近的那个，最先被点击的那个）处于
/// 列表的最前位置
```
### 事件派发阶段
```dart
/// 之后会调用 dispatchEvent 来派发事件
@override // from HitTestDispatcher
void dispatchEvent(PointerEvent event, HitTestResult hitTestResult) {
    // 没有命中测试信息意味着这是一个悬停或指针添加/删除事件。
    if (hitTestResult == null) {
        try {
            pointerRouter.route(event);
        } catch (exception, stack) {
            ...
        }
        return;
    }
    /// 遍历列表，调用每一个 hitTestEntry 的 handleEvent 方法，由于列表的排列方式，自身的
    /// handleEvent会最后被调用
    for (HitTestEntry entry in hitTestResult.path) {
        try {
            entry.target.handleEvent(event.transformed(entry.transform), entry);
        } catch (exception, stack) {
           ...
        }
    }
}

/// PointerRouter.route 
void route(PointerEvent event) {
    final Map<PointerRoute, Matrix4> routes = _routeMap[event.pointer];
    final Map<PointerRoute, Matrix4> copiedGlobalRoutes = Map<PointerRoute, Matrix4>.from(_globalRoutes);
    if (routes != null) {
        _dispatchEventToRoutes(
            event,
            routes,
            Map<PointerRoute, Matrix4>.from(routes),
        );
    }
    _dispatchEventToRoutes(event, _globalRoutes, copiedGlobalRoutes);
}

/// RenderObject（hitTestTarget）子类的 handleEvent 的实现（以RenderPointerListener为例）
@override
void handleEvent(PointerEvent event, HitTestEntry entry) {
    assert(debugHandleEvent(event, entry));
    if (onPointerDown != null && event is PointerDownEvent)
        return onPointerDown(event);
    if (onPointerMove != null && event is PointerMoveEvent)
        return onPointerMove(event);
    if (onPointerUp != null && event is PointerUpEvent)
        return onPointerUp(event);
    if (onPointerCancel != null && event is PointerCancelEvent)
        return onPointerCancel(event);
    if (onPointerSignal != null && event is PointerSignalEvent)
        return onPointerSignal(event);
}

/// 这会触发对应的事件会调用。再GestureDetector中向RenderPointerListener注册了如下会调用
/// 其中_recognizers是根据我们在使用GestureDetector注册的手势回调生成的对应的识别器。
/// 当指针按下事件来临时，将它派发给各个Recognizer
void _handlePointerDown(PointerDownEvent event) {
    assert(_recognizers != null);
    for (GestureRecognizer recognizer in _recognizers.values)
        recognizer.addPointer(event);
}

/// 这里再通过 DoubleTapGestureRecognizer 来说明 addPointer 方法
/// 由以上的 recognizer.addPointer(event)，会调用到 
/// DoubleTapGestureRecognizer._trackFirstTap方法
void _trackFirstTap(PointerEvent event) {
    _stopDoubleTapTimer();
    final _TapTracker tracker = _TapTracker(
        event: event as PointerDownEvent,
        /// 注意到 GestureRecognizer 实现了 GestureArenaMember
        /// 这里向 GestureBinding 中把自己作为 成员加入了竞技场中
        entry: GestureBinding.instance.gestureArena.add(event.pointer, this),
        doubleTapMinTime: kDoubleTapMinTime,
    );
    _trackers[event.pointer] = tracker;
    tracker.startTrackingPointer(_handleEvent, event.transform);
}
```
###  手势竞技阶段
```dart
/// 总之，再调用完 hitTestResult 中的每一个 entry 的handleEvent 方法后，手势竞技场类应该已经存在了一
/// 些成员。随后，竞技场会关闭，得到一个胜利者，即一个Recognizer，来识别手势，并产生手势效果

/// 在hitTest时，最后把自身加入了列表之中，故最后才会调用 GestureBinding 的 handleEvent方法
/// 把所有的点击事件加入到“竞技场”之中，最后得出一个测试结果
@override // from HitTestTarget
void handleEvent(PointerEvent event, HitTestEntry entry) {
    pointerRouter.route(event);
    /// 指针落下时，关闭竞技场
    if (event is PointerDownEvent) {
        gestureArena.close(event.pointer);
    /// 指针抬起时，强行得出一个获胜者    
    } else if (event is PointerUpEvent) {
        gestureArena.sweep(event.pointer);
    } else if (event is PointerSignalEvent) {
        pointerSignalResolver.resolve(event);
    }
}

/// 得到获胜者之后，调用了它的 acceptGesture方法，这个方法通常会触发注册的手势回调，于是Flutter就完成
/// 了手势的响应
```




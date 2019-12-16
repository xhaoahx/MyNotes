# Flutter  AbsorbPointer 和 IgnorePointer 源码分析

## AbsorbPointer

```dart
/// 一个能在点击测试中吸收指针事件的组件
///
/// 当 [absorbing] 为 true, 这个组件通过在自身停止点击测试来阻止子树接受到点击事件.
/// 它如同普通组件一样会在布局和绘制中占用控件.
/// 它只是阻止了子树接受到定位事件, 因为它重载的 [RenderBox.hitTest] 返回 true 
///
class AbsorbPointer extends SingleChildRenderObjectWidget {
  const AbsorbPointer({
    Key key,
    this.absorbing = true,
    Widget child,
    this.ignoringSemantics,
  }) : assert(absorbing != null),
       super(key: key, child: child);

  /// 是否需要阻止子树接受点击测试
  /// 无论这个值如何，这个组件依旧会在布局和绘制中占用空间
  final bool absorbing;
    
  final bool ignoringSemantics;

  @override
  RenderAbsorbPointer createRenderObject(BuildContext context) {
    return RenderAbsorbPointer(
      absorbing: absorbing,
      ignoringSemantics: ignoringSemantics,
    );
  }

  @override
  void updateRenderObject(BuildContext context, RenderAbsorbPointer renderObject) {
    renderObject
      ..absorbing = absorbing
      ..ignoringSemantics = ignoringSemantics;
  }
}
```



## RenderAbsorbPointer

```dart
class RenderAbsorbPointer extends RenderProxyBox {
  RenderAbsorbPointer({
    RenderBox child,
    bool absorbing = true,
    bool ignoringSemantics,
  }) : assert(absorbing != null),
       _absorbing = absorbing,
       _ignoringSemantics = ignoringSemantics,
       super(child);

  /// 是否需要吸收指针事件
  bool get absorbing => _absorbing;
  bool _absorbing;
  set absorbing(bool value) {
    if (_absorbing == value)
      return;
    _absorbing = value;
    if (ignoringSemantics == null)
      markNeedsSemanticsUpdate();
  }

  /// 语义相关，在修改这个值的时候会触发语义更新
  bool get ignoringSemantics => _ignoringSemantics;
  bool _ignoringSemantics;
  set ignoringSemantics(bool value) {
    if (value == _ignoringSemantics)
      return;
    final bool oldEffectiveValue = _effectiveIgnoringSemantics;
    _ignoringSemantics = value;
    if (oldEffectiveValue != _effectiveIgnoringSemantics)
      markNeedsSemanticsUpdate();
  }

  bool get _effectiveIgnoringSemantics => ignoringSemantics ?? absorbing;

  @override
  bool hitTest(BoxHitTestResult result, { Offset position }) {
    return absorbing
        /// 如果点击的位置在自身的大小之内的话，那么返回 true，即中断点击测试。否则返回false
        ? size.contains(position)
        /// 否则进行正常的点击测试
        : super.hitTest(result, position: position);
        /// [RenderProxyBox.hitTest]
        /// bool hitTest(BoxHitTestResult result, { @required Offset position }) {
        ///   if (_size.contains(position)) {
        ///     if (hitTestChildren(result, position: position) || hitTestSelf(position)) {
        ///       result.add(BoxHitTestEntry(this, position));
        ///       return true;
        ///     }
        ///   }
        ///   return false;
        /// }
  }
}
```



## IgnorePointer

```dart
/// 功能和 AbsorbPointer 类似
class IgnorePointer extends SingleChildRenderObjectWidget {
  const IgnorePointer({
    Key key,
    this.ignoring = true,
    this.ignoringSemantics,
    Widget child,
  }) : assert(ignoring != null),
       super(key: key, child: child);
    
  final bool ignoring;

  final bool ignoringSemantics;

  @override
  RenderIgnorePointer createRenderObject(BuildContext context) {
    return RenderIgnorePointer(
      ignoring: ignoring,
      ignoringSemantics: ignoringSemantics,
    );
  }

  @override
  void updateRenderObject(BuildContext context, RenderIgnorePointer renderObject) {
    renderObject
      ..ignoring = ignoring
      ..ignoringSemantics = ignoringSemantics;
  }
}

```



## RenderIgnorePointer

```dart
class RenderIgnorePointer extends RenderProxyBox {
  RenderIgnorePointer({
    RenderBox child,
    bool ignoring = true,
    bool ignoringSemantics,
  }) : _ignoring = ignoring,
       _ignoringSemantics = ignoringSemantics,
       super(child) {
    assert(_ignoring != null);
  }

  bool get ignoring => _ignoring;
  bool _ignoring;
  set ignoring(bool value) {
    assert(value != null);
    if (value == _ignoring)
      return;
    _ignoring = value;
    if (_ignoringSemantics == null || !_ignoringSemantics)
      markNeedsSemanticsUpdate();
  }

  bool get ignoringSemantics => _ignoringSemantics;
  bool _ignoringSemantics;
  set ignoringSemantics(bool value) {
    if (value == _ignoringSemantics)
      return;
    final bool oldEffectiveValue = _effectiveIgnoringSemantics;
    _ignoringSemantics = value;
    if (oldEffectiveValue != _effectiveIgnoringSemantics)
      markNeedsSemanticsUpdate();
  }

  bool get _effectiveIgnoringSemantics => ignoringSemantics ?? ignoring;

  /// 也是通过返回 true 的方法来阻止点击测试的传播
  /// 与 AbsorbPointer 不同的是， IgnorePointer 在 ignoring 为 true 时不会对自身进行点击测试：
  /// 即调用 size.contain(position)，这意味无论如何点击测试都会继续传播
  /// 而 AbsorbPointer 会对自身进行点击测试，如果点击位置在自身的大小之内，才会阻止点击测试  
  @override
  bool hitTest(BoxHitTestResult result, { Offset position }) {
    return !ignoring && super.hitTest(result, position: position);
  }

  @override
  void visitChildrenForSemantics(RenderObjectVisitor visitor) {
    if (child != null && !_effectiveIgnoringSemantics)
      visitor(child);
  }
}
```


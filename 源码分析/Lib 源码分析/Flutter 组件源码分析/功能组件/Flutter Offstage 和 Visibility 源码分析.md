# Flutter Offstage 和 Visibility 源码分析

## Offstatge

```dart
/// 将孩子在树中布局，但是不绘制子树。这会使得孩子不能被测试，并且不在其父节点中占用空间
/// 被这个组件隐藏的子树的状态会被维持
///
/// 子树中的动画不会被停止，即是子树是不可见的
/// 如果需要停止子树的动画，使用 [TickMode] 组件
class Offstage extends SingleChildRenderObjectWidget {
  const Offstage({ Key key, this.offstage = true, Widget child })
    : assert(offstage != null),
      super(key: key, child: child);

  /// 是否应该隐藏子树
  final bool offstage;

  @override
  RenderOffstage createRenderObject(BuildContext context) => RenderOffstage(offstage: offstage);

  @override
  void updateRenderObject(BuildContext context, RenderOffstage renderObject) {
    renderObject.offstage = offstage;
  }

  @override
  _OffstageElement createElement() => _OffstageElement(this);
}
```
### _OffstatgeElement
```dart
class _OffstageElement extends SingleChildRenderObjectElement {
  _OffstageElement(Offstage widget) : super(widget);

  @override
  Offstage get widget => super.widget as Offstage;
}
```
### RenderOffstage
```dart
/// 渲染 Offstage
class RenderOffstage extends RenderProxyBox {
  /// Creates an offstage render object.
  RenderOffstage({
    bool offstage = true,
    RenderBox child,
  }) : _offstage = offstage,
       super(child);

  /// 是否应该隐藏子树，当这个值改变的时候，会触发重新布局
  bool get offstage => _offstage;
  bool _offstage;
  set offstage(bool value) {
    if (value == _offstage)
      return;
    _offstage = value;
    /// 注意：
    /// 在 offstage 的情况下，自身的固有宽高都是 0.0，自身的大小只受到父级给定的约束影响，故 sizedByParent 
    /// 为 true；否则，自身的大小受到子树布局和大小的影响
    /// 因此 offstage 实际会导致 sizedByParent 的改变
    markNeedsLayoutForSizedByParentChange();
  }

  /// 在子树 offstatge 的情况下
  /// 其最大最小固有宽高都是 0.0
  @override
  double computeMinIntrinsicWidth(double height) {
    if (offstage)
      return 0.0;
    return super.computeMinIntrinsicWidth(height);
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    if (offstage)
      return 0.0;
    return super.computeMaxIntrinsicWidth(height);
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    if (offstage)
      return 0.0;
    return super.computeMinIntrinsicHeight(width);
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    if (offstage)
      return 0.0;
    return super.computeMaxIntrinsicHeight(width);
  }

  @override
  double computeDistanceToActualBaseline(TextBaseline baseline) {
    if (offstage)
      return null;
    return super.computeDistanceToActualBaseline(baseline);
  }

  /// 当 offstage 的时候，自身的大小只受到父级给定的约束影响，故 sizedByParent 为 true
  /// 否则，自身的大小受到子树布局和大小的影响
  /// 注意，当 offstage 的时候，自身是一个重新布局边界
  @override
  bool get sizedByParent => offstage;

  /// 采用父节点给定的约束的最小值来作为自身的大小
  @override
  void performResize() {
    size = constraints.smallest;
  }

  
  @override
  void performLayout() {
    /// 将父节点给定的布局直接传给子树
    if (offstage) {
      child?.layout(constraints);
    } else {
      super.performLayout();
    }
  }

  /// 当 offstage 的时候，阻止子树受到点击测试
  @override
  bool hitTest(BoxHitTestResult result, { Offset position }) {
    return !offstage && super.hitTest(result, position: position);
  }

  /// 当 offstage 的时候，不对子树进行绘制
  @override
  void paint(PaintingContext context, Offset offset) {
    if (offstage)
      return;
    super.paint(context, offset);
  }

  @override
  void visitChildrenForSemantics(RenderObjectVisitor visitor) {
    if (offstage)
      return;
    super.visitChildrenForSemantics(visitor);
  }
}

```

## Visibility

```dart
/// 控制隐藏或者展示子树的组件 
///
/// 默认情况下，[visible] 控制了 [child] 是否被包括到子树中;
/// [visible] = false 的时候， [replacement] (通常是一个大小为 0 的盒子) 将会被包含子树中 
/// 
/// 事实上，Visibility 是一个使用各种组件嵌套而成的的组件，在不同情况下，会使用不同组合形式来实现保存状态
class Visibility extends StatelessWidget {
  const Visibility({
    Key key,
    @required this.child,
    this.replacement = const SizedBox.shrink(),
    this.visible = true,
    this.maintainState = false,
    this.maintainAnimation = false,
    this.maintainSize = false,
    this.maintainSemantics = false,
    this.maintainInteractivity = false,
  }) :  super(key: key);

  /// 需要被展示或者隐藏的孩子，被 [visible] 属性控制
  final Widget child;

  /// 当 [visible] = false 的时候，展示的组件, 假设所有的 maintain 标记都为 false
  /// 默认情况下，是一个大小为 0 的盒子
  final Widget replacement;

  /// 控制是否要展示 widget
  final bool visible;

  /// 是否应该维持子树的状态
  ///
  /// 如果这个属性为 true,[Offstage] 组件将会被用于隐藏孩子，而不是使用 replacement 来取代子树
  ///
  /// 如果这个属性为 false, 那么 [maintainAnimation] 必须是 false.
  final bool maintainState;

  /// 子树的动画状态是否应该被维持 [visible].
  ///
  /// 假如此属性的值是 false, 将不会使用 [TickerMode] 组件
  final bool maintainAnimation;

  /// 是否维持子树可能放置的位置
  ///
  /// 假如这个属性为 true，将使用 [Opacity] 来隐藏（即设置 opacity = 0.0）子树，而不是使用 [Offstage].
  final bool maintainSize;

  final bool maintainSemantics;

  /// 是否维持子树的可交互性
  ///
  /// 如果要设置为 true, 那么 [maintainSize] 必须设置为 true.
  final bool maintainInteractivity;

  @override
  Widget build(BuildContext context) {
    /// 如果要维持子树的位置，那么使用 Opacity 组件来隐藏子树
    if (maintainSize) {
      Widget result = child;
      /// 如果不需要维持子树的可交互性，那么还需要嵌套一层忽略指针的组件
      if (!maintainInteractivity) {
        result = IgnorePointer(
          child: child,
          ignoring: !visible,
          ignoringSemantics: !visible && !maintainSemantics,
        );
      }
      return Opacity(
        opacity: visible ? 1.0 : 0.0,
        alwaysIncludeSemantics: maintainSemantics,
        child: result,
      );
    }
    
    /// 如果需要维持子树的状态，使用 Offstage 来隐藏（即只进行构建和布局，而不绘制）子树
    if (maintainState) {
      Widget result = child;
      /// 如果不需要维持子树的动画状态，那么使用 TickMode，并禁用其 tick 来阻止动画进行
      if (!maintainAnimation)
        result = TickerMode(child: child, enabled: visible);
      return Offstage(
        child: result,
        offstage: !visible,
      );
    }
    /// 所有的状态都不需要被维持，可以使用 replacement 组件来替换子树
    /// 并且当 visible 为 true 的时候重新构建并绘制子树
    return visible ? child : replacement;
  }
}

```


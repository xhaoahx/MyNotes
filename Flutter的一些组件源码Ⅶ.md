# Flutter的一些组件源码Ⅶ

[TOC]

## Offstage

### Offstage
```dart
class Offstage extends SingleChildRenderObjectWidget {
  const Offstage({ Key key, this.offstage = true, Widget child })
    : assert(offstage != null),
      super(key: key, child: child);
  final bool offstage;

  /// 将 offstage 交给 RenderObejct 处理  
  @override
  RenderOffstage createRenderObject(BuildContext context) => 
      RenderOffstage(offstage: offstage);

  @override
  void updateRenderObject(BuildContext context, RenderOffstage renderObject) {
    renderObject.offstage = offstage;
  }

  @override
  _OffstageElement createElement() => _OffstageElement(this);
}
```


### _offstageElement

```dart
class _OffstageElement extends SingleChildRenderObjectElement {
  _OffstageElement(Offstage widget) : super(widget);
  @override
  Offstage get widget => super.widget;
}
```



### RenderOffstage

```dart
class RenderOffstage extends RenderProxyBox {
  /// Creates an offstage render object.
  RenderOffstage({
    bool offstage = true,
    RenderBox child,
  }) : assert(offstage != null),
       _offstage = offstage,
       super(child);

  /// 是否将子树从从界面隐藏
  /// 它不会移除子树，而是不做布局、绘制  
  bool get offstage => _offstage;
  bool _offstage;
  set offstage(bool value) {
    assert(value != null);
    if (value == _offstage)
      return;
    _offstage = value;
    markNeedsLayoutForSizedByParentChange();
  }

  /// 以下四个方法在 offstage == true 计算布局的时候，均返回0.0  
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

  @override
  bool get sizedByParent => offstage;

  @override
  void performResize() {
    assert(offstage);
    /// size 为 Size（0.0）  
    size = constraints.smallest;
  }

  @override
  void performLayout() {
    if (offstage) {
      child?.layout(constraints);
    } else {
      super.performLayout();
    }
  }

  @override
  bool hitTest(BoxHitTestResult result, { Offset position }) {
    return !offstage && super.hitTest(result, position: position);
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    /// 不做绘制  
    if (offstage)
      return;
    super.paint(context, offset);
  }

  @override
  void visitChildrenForSemantics(RenderObjectVisitor visitor) {
    /// 不遍历子树（假装没有子树）  
    if (offstage)
      return;
    super.visitChildrenForSemantics(visitor);
  }
}
```
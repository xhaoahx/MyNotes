# Flutter OverflowBox 源码分析

[TOC]

## RenderShiftedBox

见Flutter CustomLayout 源码分析

<a herf = 'Flutter CustomLayout 源码分析.md'>链接</a> 



## RenderAligningShiftedBox

```dart
abstract class RenderAligningShiftedBox extends RenderShiftedBox {
  RenderAligningShiftedBox({
    AlignmentGeometry alignment = Alignment.center,
    @required TextDirection textDirection,
    RenderBox child,
  }) : assert(alignment != null),
       _alignment = alignment,
       _textDirection = textDirection,
       super(child);

  @protected
  RenderAligningShiftedBox.mixin(
      AlignmentGeometry alignment, 
      TextDirection textDirection, 
      RenderBox child
  ) : this(alignment: alignment, textDirection: textDirection, child: child);

  Alignment _resolvedAlignment;

  void _resolve() {
    if (_resolvedAlignment != null) return;
    _resolvedAlignment = alignment.resolve(textDirection);
  }

  void _markNeedResolution() {
    _resolvedAlignment = null;
    markNeedsLayout();
  }

  AlignmentGeometry get alignment => _alignment;
  AlignmentGeometry _alignment;
  
  /// 修改alignment，会触发layout  
  set alignment(AlignmentGeometry value) {
    if (_alignment == value) return;
    _alignment = value;
    _markNeedResolution();
  }

    
  TextDirection get textDirection => _textDirection;
  TextDirection _textDirection;
  set textDirection(TextDirection value) {
    if (_textDirection == value) return;
    _textDirection = value;
    _markNeedResolution();
  }

  /// 为child运用alignment
  @protected
  void alignChild() {
    _resolve();
    final BoxParentData childParentData = child.parentData;
    childParentData.offset = _resolvedAlignment.alongOffset(size - child.size);
  }

}
```



## OverflowBox

```dart
class OverflowBox extends SingleChildRenderObjectWidget {
  /// 创建一个能使孩子能溢出自身的widget
  const OverflowBox({
    Key key,
    this.alignment = Alignment.center,
    this.minWidth,
    this.maxWidth,
    this.minHeight,
    this.maxHeight,
    Widget child,
  }) : super(key: key, child: child);

  final AlignmentGeometry alignment;

  /// 设置最小宽度（为null则使用父级约束的最小宽度）
  final double minWidth;
 /// 设置最大宽度（为null则使用父级约束的最大宽度）
  final double maxWidth;

   /// 设置最小高度（为null则使用父级约束的最小高度）
  final double minHeight;

  /// 设置最大高度（为null则使用父级约束的最大高度）
  final double maxHeight;

  @override
  RenderConstrainedOverflowBox createRenderObject(BuildContext context) {
    return RenderConstrainedOverflowBox(
      alignment: alignment,
      minWidth: minWidth,
      maxWidth: maxWidth,
      minHeight: minHeight,
      maxHeight: maxHeight,
      textDirection: Directionality.of(context),
    );
  }

  @override
  void updateRenderObject(BuildContext context, RenderConstrainedOverflowBox renderObject) {
    renderObject
      ..alignment = alignment
      ..minWidth = minWidth
      ..maxWidth = maxWidth
      ..minHeight = minHeight
      ..maxHeight = maxHeight
      ..textDirection = Directionality.of(context);
  }
}
```



## RenderConstrainedOverflowBox

```dart
class RenderConstrainedOverflowBox extends RenderAligningShiftedBox {
  RenderConstrainedOverflowBox({
    RenderBox child,
    double minWidth,
    double maxWidth,
    double minHeight,
    double maxHeight,
    AlignmentGeometry alignment = Alignment.center,
    TextDirection textDirection,
  }) : _minWidth = minWidth,
       _maxWidth = maxWidth,
       _minHeight = minHeight,
       _maxHeight = maxHeight,
       super(child: child, alignment: alignment, textDirection: textDirection);

  /// 最大最小宽高，发生改变时触发重新布局
  double get minWidth => _minWidth;
  double _minWidth;
  set minWidth(double value) {
    if (_minWidth == value)
      return;
    _minWidth = value;
    markNeedsLayout();
  }

  double get maxWidth => _maxWidth;
  double _maxWidth;
  set maxWidth(double value) {
    if (_maxWidth == value)
      return;
    _maxWidth = value;
    markNeedsLayout();
  }
    
  double get minHeight => _minHeight;
  double _minHeight;
  set minHeight(double value) {
    if (_minHeight == value)
      return;
    _minHeight = value;
    markNeedsLayout();
  }
    
  double get maxHeight => _maxHeight;
  double _maxHeight;
  set maxHeight(double value) {
    if (_maxHeight == value)
      return;
    _maxHeight = value;
    markNeedsLayout();
  }

    
  /// 用给定的最大最小宽高创建约束  
  BoxConstraints _getInnerConstraints(BoxConstraints constraints) {
    return BoxConstraints(
      minWidth: _minWidth ?? constraints.minWidth,
      maxWidth: _maxWidth ?? constraints.maxWidth,
      minHeight: _minHeight ?? constraints.minHeight,
      maxHeight: _maxHeight ?? constraints.maxHeight,
    );
  }

  @override
  bool get sizedByParent => true;

  @override
  void performResize() {
    size = constraints.biggest;
  }

  @override
  void performLayout() {
    if (child != null) {
      child.layout(_getInnerConstraints(constraints), parentUsesSize: true);
      alignChild();
    }
  }
}
```



## SizedOverflowBox

```dart
class SizedOverflowBox extends SingleChildRenderObjectWidget {
  const SizedOverflowBox({
    Key key,
    @required this.size,
    this.alignment = Alignment.center,
    Widget child,
  }) : assert(size != null),
       assert(alignment != null),
       super(key: key, child: child);

  final AlignmentGeometry alignment;

  final Size size;

  @override
  RenderSizedOverflowBox createRenderObject(BuildContext context) {
    return RenderSizedOverflowBox(
      alignment: alignment,
      requestedSize: size,
      textDirection: Directionality.of(context),
    );
  }

  @override
  void updateRenderObject(BuildContext context, RenderSizedOverflowBox renderObject) {
    renderObject
      ..alignment = alignment
      ..requestedSize = size
      ..textDirection = Directionality.of(context);
  }
}
```



##  RenderSizedOverflowBox 

```dart
class RenderSizedOverflowBox extends RenderAligningShiftedBox {
  RenderSizedOverflowBox({
    RenderBox child,
    @required Size requestedSize,
    AlignmentGeometry alignment = Alignment.center,
    TextDirection textDirection,
  }) : assert(requestedSize != null),
       _requestedSize = requestedSize,
       super(child: child, alignment: alignment, textDirection: textDirection);

  Size get requestedSize => _requestedSize;
  Size _requestedSize;
    
  set requestedSize(Size value) {
    assert(value != null);
    if (_requestedSize == value)
      return;
    _requestedSize = value;
    markNeedsLayout();
  }

  @override
  double computeMinIntrinsicWidth(double height) {
    return _requestedSize.width;
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    return _requestedSize.width;
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    return _requestedSize.height;
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    return _requestedSize.height;
  }

  @override
  double computeDistanceToActualBaseline(TextBaseline baseline) {
    if (child != null)
      return child.getDistanceToActualBaseline(baseline);
    return super.computeDistanceToActualBaseline(baseline);
  }

  @override
  void performLayout() {
    size = constraints.constrain(_requestedSize);
    if (child != null) {
      child.layout(constraints, parentUsesSize: true);
      alignChild();
    }
  }
}
```



## FractionallySizedBox

```dart
class FractionallySizedBox extends SingleChildRenderObjectWidget {
  const FractionallySizedBox({
    Key key,
    this.alignment = Alignment.center,
    this.widthFactor,
    this.heightFactor,
    Widget child,
  }) : assert(alignment != null),
       assert(widthFactor == null || widthFactor >= 0.0),
       assert(heightFactor == null || heightFactor >= 0.0),
       super(key: key, child: child);
    
  final double widthFactor;
  final double heightFactor;

  final AlignmentGeometry alignment;

  @override
  RenderFractionallySizedOverflowBox createRenderObject(BuildContext context) {
    return RenderFractionallySizedOverflowBox(
      alignment: alignment,
      widthFactor: widthFactor,
      heightFactor: heightFactor,
      textDirection: Directionality.of(context),
    );
  }

  @override
  void updateRenderObject(
      BuildContext context,
      RenderFractionallySizedOverflowBox renderObject) 
  {
    renderObject
      ..alignment = alignment
      ..widthFactor = widthFactor
      ..heightFactor = heightFactor
      ..textDirection = Directionality.of(context);
  }
}
```



## RenderFractionallySizedOverflowBox

```dart
class RenderFractionallySizedOverflowBox extends RenderAligningShiftedBox {
  /// 创建一个能将child在一定空间里限制为指定的size的盒子
  RenderFractionallySizedOverflowBox({
    RenderBox child,
    double widthFactor,
    double heightFactor,
    AlignmentGeometry alignment = Alignment.center,
    TextDirection textDirection,
  }) : _widthFactor = widthFactor,
       _heightFactor = heightFactor,
       super(child: child, alignment: alignment, textDirection: textDirection) {
  }

  /// 更改以下两个值会触发重新布局
  double get widthFactor => _widthFactor;
  double _widthFactor;
  set widthFactor(double value) {
    assert(value == null || value >= 0.0);
    if (_widthFactor == value) return;
    _widthFactor = value;
    markNeedsLayout();
  }

  double get heightFactor => _heightFactor;
  double _heightFactor;
  set heightFactor(double value) {
    assert(value == null || value >= 0.0);
    if (_heightFactor == value)
      return;
    _heightFactor = value;
    markNeedsLayout();
  }

  /// 根据_widthFactor和_heightFactor来计算新的约束
  BoxConstraints _getInnerConstraints(BoxConstraints constraints) {
    double minWidth = constraints.minWidth;
    double maxWidth = constraints.maxWidth;
    if (_widthFactor != null) {
      final double width = maxWidth * _widthFactor;
      minWidth = width;
      maxWidth = width;
    }
    double minHeight = constraints.minHeight;
    double maxHeight = constraints.maxHeight;
    if (_heightFactor != null) {
      final double height = maxHeight * _heightFactor;
      minHeight = height;
      maxHeight = height;
    }
    return BoxConstraints(
      minWidth: minWidth,
      maxWidth: maxWidth,
      minHeight: minHeight,
      maxHeight: maxHeight,
    );
  }

  /// 根据_widthFactor和_heightFactor来计算最大最小固有宽高   
  @override
  double computeMinIntrinsicWidth(double height) {
    double result;
    if (child == null) {
      result = super.computeMinIntrinsicWidth(height);
    } else {
      result = child.getMinIntrinsicWidth(height * (_heightFactor ?? 1.0));
    }
    return result / (_widthFactor ?? 1.0);
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    double result;
    if (child == null) {
      result = super.computeMaxIntrinsicWidth(height);
    } else {
      result = child.getMaxIntrinsicWidth(height * (_heightFactor ?? 1.0));
    }
    return result / (_widthFactor ?? 1.0);
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    double result;
    if (child == null) {
      result = super.computeMinIntrinsicHeight(width);
    } else {
      result = child.getMinIntrinsicHeight(width * (_widthFactor ?? 1.0));
    }
    return result / (_heightFactor ?? 1.0);
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    double result;
    if (child == null) {
      result = super.computeMaxIntrinsicHeight(width);
    } else {
      result = child.getMaxIntrinsicHeight(width * (_widthFactor ?? 1.0));
    }
    return result / (_heightFactor ?? 1.0);
  }

  @override
  void performLayout() {
    if (child != null) {
      child.layout(_getInnerConstraints(constraints), parentUsesSize: true);
      size = constraints.constrain(child.size);
      alignChild();
    } else {
      size = 
          constraints.constrain(_getInnerConstraints(constraints).constrain(Size.zero));
    }
  }
}
```


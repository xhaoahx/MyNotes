# Flutter Stack 源码分析



[TOC]

## Positioned

```dart
class Positioned extends ParentDataWidget<Stack> {
  const Positioned({
    Key key,
    this.left,
    this.top,
    this.right,
    this.bottom,
    this.width,
    this.height,
    @required Widget child,
  }) : assert(left == null || right == null || width == null),
       assert(top == null || bottom == null || height == null),
       super(key: key, child: child);

  Positioned.fromRect({
    Key key,
    Rect rect,
    @required Widget child,
  }) : left = rect.left,
       top = rect.top,
       width = rect.width,
       height = rect.height,
       right = null,
       bottom = null,
       super(key: key, child: child);

  /// Creates a Positioned object with the values from the given [RelativeRect].
  ///
  /// This sets the [left], [top], [right], and [bottom] properties from the
  /// given [RelativeRect]. The [height] and [width] properties are set to null.
  Positioned.fromRelativeRect({
    Key key,
    RelativeRect rect,
    @required Widget child,
  }) : left = rect.left,
       top = rect.top,
       right = rect.right,
       bottom = rect.bottom,
       width = null,
       height = null,
       super(key: key, child: child);

  const Positioned.fill({
    Key key,
    this.left = 0.0,
    this.top = 0.0,
    this.right = 0.0,
    this.bottom = 0.0,
    @required Widget child,
  }) : width = null,
       height = null,
       super(key: key, child: child);

  /// 根据Direction来创建
  factory Positioned.directional({
    Key key,
    @required TextDirection textDirection,
    double start,
    double top,
    double end,
    double bottom,
    double width,
    double height,
    @required Widget child,
  }) {
    assert(textDirection != null);
    double left;
    double right;
    switch (textDirection) {
      case TextDirection.rtl:
        left = end;
        right = start;
        break;
      case TextDirection.ltr:
        left = start;
        right = end;
        break;
    }
    return Positioned(
      key: key,
      left: left,
      top: top,
      right: right,
      bottom: bottom,
      width: width,
      height: height,
      child: child,
    );
  }

  /// 距离Stack边界的左上右下
  final double left;
  final double top;
  final double right;
  final double bottom;

  /// 宽度约束
  final double width;
  /// 高度约束
  final double height;

  /// 修改parentData，标记需要重新布局  
  @override
  void applyParentData(RenderObject renderObject) {
    final StackParentData parentData = renderObject.parentData;
    bool needsLayout = false;

    if (parentData.left != left) {
      parentData.left = left;
      needsLayout = true;
    }

    if (parentData.top != top) {
      parentData.top = top;
      needsLayout = true;
    }

    if (parentData.right != right) {
      parentData.right = right;
      needsLayout = true;
    }

    if (parentData.bottom != bottom) {
      parentData.bottom = bottom;
      needsLayout = true;
    }

    if (parentData.width != width) {
      parentData.width = width;
      needsLayout = true;
    }

    if (parentData.height != height) {
      parentData.height = height;
      needsLayout = true;
    }

    if (needsLayout) {
     final AbstractNode targetParent = renderObject.parent;
      if (targetParent is RenderObject)
        targetParent.markNeedsLayout();
    }
  }
}
```



## StackParentData

```dart
class StackParentData extends ContainerBoxParentData<RenderBox> {
  /// 距离上右下左的距离  
  double top;
  double right;
  double bottom;
  double left;

  /// 孩子宽高
  double width;
  double height;

  /// Get or set the current values in terms of a RelativeRect object.
  RelativeRect get rect => RelativeRect.fromLTRB(left, top, right, bottom);
  set rect(RelativeRect value) {
    top = value.top;
    right = value.right;
    bottom = value.bottom;
    left = value.left;
  }

  /// 是否是固定的
  /// 当上右下左宽高有其一被指定时，为固定状态  
  bool get isPositioned => 
      top != null 
      || right != null 
      || bottom != null 
      || left != null 
      || width != null 
      || height != null;

  ...
}
```



## Stack

```dart
class Stack extends MultiChildRenderObjectWidget {
  Stack({
    Key key,
    this.alignment = AlignmentDirectional.topStart,
    this.textDirection,
    this.fit = StackFit.loose,
    this.overflow = Overflow.clip,
    List<Widget> children = const <Widget>[],
  }) : super(key: key, children: children);

  /// 排列Stack中非固定的元素
  final AlignmentGeometry alignment;

  /// 文字方向
  final TextDirection textDirection;

  /// 栈的（？填充）方式
  ///   
  final StackFit fit;

  /// 对于Overflow Element的行为
  final Overflow overflow;

  @override
  RenderStack createRenderObject(BuildContext context) {
    return RenderStack(
      alignment: alignment,
      textDirection: textDirection ?? Directionality.of(context),
      fit: fit,
      overflow: overflow,
    );
  }

  @override
  void updateRenderObject(BuildContext context, RenderStack renderObject) {
    renderObject
      ..alignment = alignment
      ..textDirection = textDirection ?? Directionality.of(context)
      ..fit = fit
      ..overflow = overflow;
  }

}
```



## RenderStack

```dart
class RenderStack extends RenderBox
    /// 提供使用链表管理子类的mixin
    with ContainerRenderObjectMixin<RenderBox, StackParentData>,
	/// 提供paint，caculate baseline等方法的默认实现
    RenderBoxContainerDefaultsMixin<RenderBox, StackParentData> {

  RenderStack({
    List<RenderBox> children,
    AlignmentGeometry alignment = AlignmentDirectional.topStart,
    TextDirection textDirection,
    StackFit fit = StackFit.loose,
    Overflow overflow = Overflow.clip,
  }) : _alignment = alignment,
       _textDirection = textDirection,
       _fit = fit,
       _overflow = overflow 
  {
    addAll(children);
  }

  bool _hasVisualOverflow = false;

  @override
  void setupParentData(RenderBox child) {
    if (child.parentData is! StackParentData)
      child.parentData = StackParentData();
  }

  Alignment _resolvedAlignment;

  void _resolve() {
    if (_resolvedAlignment != null)
      return;
    _resolvedAlignment = alignment.resolve(textDirection);
  }

  void _markNeedResolution() {
    _resolvedAlignment = null;
    markNeedsLayout();
  }

  /// 如何排列不是postioned的元素
  AlignmentGeometry get alignment => _alignment;
  AlignmentGeometry _alignment;
  /// 每当aligment改变时，重新布局      
  set alignment(AlignmentGeometry value) {
    assert(value != null);
    if (_alignment == value)
      return;
    _alignment = value;
    _markNeedResolution();
  }

  TextDirection get textDirection => _textDirection;
  TextDirection _textDirection;
  set textDirection(TextDirection value) {
    if (_textDirection == value)
      return;
    _textDirection = value;
    _markNeedResolution();
  }

  /// 设置不为position的元素的大小
  ///
  /// ([StackFit.loose])  将宽松的constraints传递给孩子
  /// ([StackFit.expand]) 将扩展到最大的consraints传递给孩子
        
  StackFit get fit => _fit;
  StackFit _fit;
        
  /// 改变 fit 时重新布局      
  set fit(StackFit value) {
    assert(value != null);
    if (_fit != value) {
      _fit = value;
      markNeedsLayout();
    }
  }

  /// 溢出的元素是否应该被裁剪
  Overflow get overflow => _overflow;
  Overflow _overflow;
  /// overflow 改变时需要重绘（不要布局）      
  set overflow(Overflow value) {
    if (_overflow != value) {
      _overflow = value;
      markNeedsPaint();
    }
  }

  double _getIntrinsicDimension(double mainChildSizeGetter(RenderBox child)) {
    double extent = 0.0;
    RenderBox child = firstChild;
    while (child != null) {
      final StackParentData childParentData = child.parentData;
      if (!childParentData.isPositioned)
        extent = math.max(extent, mainChildSizeGetter(child));
      child = childParentData.nextSibling;
    }
    return extent;
  }

  @override
  double computeMinIntrinsicWidth(double height) {
    return _getIntrinsicDimension((RenderBox child) => 
                                  child.getMinIntrinsicWidth(height));
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    return _getIntrinsicDimension((RenderBox child) => 
                                  child.getMaxIntrinsicWidth(height));
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    return _getIntrinsicDimension((RenderBox child) => 
                                  child.getMinIntrinsicHeight(width));
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    return _getIntrinsicDimension((RenderBox child) => 
                                  child.getMaxIntrinsicHeight(width));
  }

  @override
  double computeDistanceToActualBaseline(TextBaseline baseline) {
    return defaultComputeDistanceToHighestActualBaseline(baseline);
  }

        
  @override
  void performLayout() {
    _resolve();
    /// 一开始将 hasVisuaOverflow 设置 false  
    _hasVisualOverflow = false;
    /// 是否有 非positioned child  
    bool hasNonPositionedChildren = false;
    if (childCount == 0) {
      size = constraints.biggest;
      assert(size.isFinite);
      return;
    }

    /// 默认采用最小约束作为宽高  
    double width = constraints.minWidth;
    double height = constraints.minHeight;

    /// 设置对于非positioned child 的约束  
    BoxConstraints nonPositionedConstraints;
      
    switch (fit) {
      case StackFit.loose:
        /// 宽松    
        nonPositionedConstraints = constraints.loosen();
        break;
      case StackFit.expand:
        /// 严格最大约束  
        nonPositionedConstraints = BoxConstraints.tight(constraints.biggest);
        break;
      case StackFit.passthrough:
        /// 传递    
        nonPositionedConstraints = constraints;
        break;
    }

    /// 通过链表api来遍历children  
    RenderBox child = firstChild;
    while (child != null) {
      final StackParentData childParentData = child.parentData;
      /// 设置非 positioned 标记  
      if (!childParentData.isPositioned) {
        hasNonPositionedChildren = true;
		/// 布局一个 非positioned child
        child.layout(nonPositionedConstraints, parentUsesSize: true);

        final Size childSize = child.size;
        /// 试图将宽高增大  
        width = math.max(width, childSize.width);
        height = math.max(height, childSize.height);
      }

      /// 下一个child  
      child = childParentData.nextSibling;
    }

    /// 如果有 非positioned child，设置size为自适应的宽高  
    if (hasNonPositionedChildren) {
      size = Size(width, height);
    /// 否则，设置size为最大约束    
    } else {
      size = constraints.biggest;
    }

    child = firstChild;
    while (child != null) {
      final StackParentData childParentData = child.parentData;

      if (!childParentData.isPositioned) {
        childParentData.offset = _resolvedAlignment.alongOffset(size - child.size);
      } else {
        BoxConstraints childConstraints = const BoxConstraints();

        /// 约束宽度 
        if (childParentData.left != null && childParentData.right != null)
          childConstraints = childConstraints.tighten(
          	width: size.width - childParentData.right - childParentData.left
          );
        else if (childParentData.width != null)
          childConstraints = childConstraints.tighten(width: childParentData.width);
		/// 约束高度
        if (childParentData.top != null && childParentData.bottom != null)
          childConstraints = childConstraints.tighten(
            height: size.height - childParentData.bottom - childParentData.top
          );
        else if (childParentData.height != null)
          childConstraints = childConstraints.tighten(height: childParentData.height);
		
        /// 请求子元素布局  
        child.layout(childConstraints, parentUsesSize: true);

        /// offset.x
        double x;
        if (childParentData.left != null) {
          x = childParentData.left;
        } else if (childParentData.right != null) {
          x = size.width - childParentData.right - child.size.width;
        } else {
          x = _resolvedAlignment.alongOffset(size - child.size).dx;
        }
 
        /// 如果上下左右有溢出，则 _hasVisualOverflow 为true  
        if (x < 0.0 || x + child.size.width > size.width)
          _hasVisualOverflow = true;

        /// offset.y   
        double y;
        if (childParentData.top != null) {
          y = childParentData.top;
        } else if (childParentData.bottom != null) {
          y = size.height - childParentData.bottom - child.size.height;
        } else {
          y = _resolvedAlignment.alongOffset(size - child.size).dy;
        }

        if (y < 0.0 || y + child.size.height > size.height)
          _hasVisualOverflow = true;

        /// 设置paint offset  
        childParentData.offset = Offset(x, y);
      }

      child = childParentData.nextSibling;
    }
  }

  @override
  bool hitTestChildren(BoxHitTestResult result, { Offset position }) {
    return defaultHitTestChildren(result, position: position);
  }

  @protected
  void paintStack(PaintingContext context, Offset offset) {
    /// 采用默认实现的绘制方法，根据layout设置的offset来绘制每一个child  
    defaultPaint(context, offset);
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    /// 如果有溢出并且设置Overflow 为 clip，那么根据offset和size来裁剪paint  
    if (_overflow == Overflow.clip && _hasVisualOverflow) {
      context.pushClipRect(needsCompositing, offset, Offset.zero & size, paintStack);
    /// 否则直接绘制    
    } else {
      paintStack(context, offset);
    }
  }
}
```



## IndexedStack

```dart
class IndexedStack extends Stack {
  /// Creates a [Stack] widget that paints a single child.
  ///
  /// The [index] argument must not be null.
  IndexedStack({
    Key key,
    AlignmentGeometry alignment = AlignmentDirectional.topStart,
    TextDirection textDirection,
    StackFit sizing = StackFit.loose,
    this.index = 0,
    List<Widget> children = const <Widget>[],
  }) : super(key: key, alignment: alignment, textDirection: textDirection, fit: sizing, children: children);

  /// The index of the child to show.
  final int index;

  @override
  RenderIndexedStack createRenderObject(BuildContext context) {
    return RenderIndexedStack(
      index: index,
      alignment: alignment,
      textDirection: textDirection ?? Directionality.of(context),
    );
  }

  @override
  void updateRenderObject(BuildContext context, RenderIndexedStack renderObject) {
    renderObject
      ..index = index
      ..alignment = alignment
      ..textDirection = textDirection ?? Directionality.of(context);
  }
}
```



## RenderIndexedStack

```dart
class RenderIndexedStack extends RenderStack {
  /// Creates a stack render object that paints a single child.
  ///
  /// If the [index] parameter is null, nothing is displayed.
  RenderIndexedStack({
    List<RenderBox> children,
    AlignmentGeometry alignment = AlignmentDirectional.topStart,
    TextDirection textDirection,
    int index = 0,
  }) : _index = index,
       super(
         children: children,
         alignment: alignment,
         textDirection: textDirection,
       );

  ...

  /// 需要展示的child的index，设为为null则不展示
  int get index => _index;
  int _index;
  set index(int value) {
    if (_index != value) {
      _index = value;
      markNeedsLayout();
    }
  }

  RenderBox _childAtIndex() {
    RenderBox child = firstChild;
    int i = 0;
    /// 事件复杂度O（n），通过链表访问，得到第index个child
    while (child != null && i < index) {
      final StackParentData childParentData = child.parentData;
      child = childParentData.nextSibling;
      i += 1;
    }
    return child;
  }

  @override
  bool hitTestChildren(BoxHitTestResult result, { @required Offset position }) {
    if (firstChild == null || index == null)
      return false;
    final RenderBox child = _childAtIndex();
    final StackParentData childParentData = child.parentData;
    return result.addWithPaintOffset(
      offset: childParentData.offset,
      position: position,
      hitTest: (BoxHitTestResult result, Offset transformed) {
        return child.hitTest(result, position: transformed);
      },
    );
  }

  /// 没有重载父类的layout方法，只是绘制指定的child  
  @override
  void paintStack(PaintingContext context, Offset offset) {
    if (firstChild == null || index == null)
      return;
    final RenderBox child = _childAtIndex();
    final StackParentData childParentData = child.parentData;
    context.paintChild(child, childParentData.offset + offset);
  }
}
```


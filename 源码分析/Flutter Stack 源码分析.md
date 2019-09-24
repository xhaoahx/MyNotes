# Flutter Stack 源码分析



[TOC]

## StackParentData

```dart
class StackParentData extends ContainerBoxParentData<RenderBox> {
  /// The distance by which the child's top edge is inset from the top of the stack.
  double top;

  /// The distance by which the child's right edge is inset from the right of the stack.
  double right;

  /// The distance by which the child's bottom edge is inset from the bottom of the stack.
  double bottom;

  /// The distance by which the child's left edge is inset from the left of the stack.
  double left;

  /// The child's width.
  ///
  /// Ignored if both left and right are non-null.
  double width;

  /// The child's height.
  ///
  /// Ignored if both top and bottom are non-null.
  double height;

  /// Get or set the current values in terms of a RelativeRect object.
  RelativeRect get rect => RelativeRect.fromLTRB(left, top, right, bottom);
  set rect(RelativeRect value) {
    top = value.top;
    right = value.right;
    bottom = value.bottom;
    left = value.left;
  }

  /// Whether this child is considered positioned.
  ///
  /// A child is positioned if any of the top, right, bottom, or left properties
  /// are non-null. Positioned children do not factor into determining the size
  /// of the stack but are instead placed relative to the non-positioned
  /// children in the stack.
  bool get isPositioned => top != null || right != null || bottom != null || left != null || width != null || height != null;

  @override
  String toString() {
    final List<String> values = <String>[
      if (top != null) 'top=${debugFormatDouble(top)}',
      if (right != null) 'right=${debugFormatDouble(right)}',
      if (bottom != null) 'bottom=${debugFormatDouble(bottom)}',
      if (left != null) 'left=${debugFormatDouble(left)}',
      if (width != null) 'width=${debugFormatDouble(width)}',
      if (height != null) 'height=${debugFormatDouble(height)}',
    ];
    if (values.isEmpty)
      values.add('not positioned');
    values.add(super.toString());
    return values.join('; ');
  }
}
```



## Stack

```dart
class Stack extends MultiChildRenderObjectWidget {
  /// Creates a stack layout widget.
  ///
  /// By default, the non-positioned children of the stack are aligned by their
  /// top left corners.
  Stack({
    Key key,
    this.alignment = AlignmentDirectional.topStart,
    this.textDirection,
    this.fit = StackFit.loose,
    this.overflow = Overflow.clip,
    List<Widget> children = const <Widget>[],
  }) : super(key: key, children: children);

  /// How to align the non-positioned and partially-positioned children in the
  /// stack.
  ///
  /// The non-positioned children are placed relative to each other such that
  /// the points determined by [alignment] are co-located. For example, if the
  /// [alignment] is [Alignment.topLeft], then the top left corner of
  /// each non-positioned child will be located at the same global coordinate.
  ///
  /// Partially-positioned children, those that do not specify an alignment in a
  /// particular axis (e.g. that have neither `top` nor `bottom` set), use the
  /// alignment to determine how they should be positioned in that
  /// under-specified axis.
  ///
  /// Defaults to [AlignmentDirectional.topStart].
  ///
  /// See also:
  ///
  ///  * [Alignment], a class with convenient constants typically used to
  ///    specify an [AlignmentGeometry].
  ///  * [AlignmentDirectional], like [Alignment] for specifying alignments
  ///    relative to text direction.
  final AlignmentGeometry alignment;

  /// The text direction with which to resolve [alignment].
  ///
  /// Defaults to the ambient [Directionality].
  final TextDirection textDirection;

  /// How to size the non-positioned children in the stack.
  ///
  /// The constraints passed into the [Stack] from its parent are either
  /// loosened ([StackFit.loose]) or tightened to their biggest size
  /// ([StackFit.expand]).
  final StackFit fit;

  /// Whether overflowing children should be clipped. See [Overflow].
  ///
  /// Some children in a stack might overflow its box. When this flag is set to
  /// [Overflow.clip], children cannot paint outside of the stack's box.
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

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(DiagnosticsProperty<AlignmentGeometry>('alignment', alignment));
    properties.add(EnumProperty<TextDirection>('textDirection', textDirection, defaultValue: null));
    properties.add(EnumProperty<StackFit>('fit', fit));
    properties.add(EnumProperty<Overflow>('overflow', overflow));
  }
}
```



## RenderStack

```dart
class RenderStack extends RenderBox
    with ContainerRenderObjectMixin<RenderBox, StackParentData>,
         RenderBoxContainerDefaultsMixin<RenderBox, StackParentData> {
  /// Creates a stack render object.
  ///
  /// By default, the non-positioned children of the stack are aligned by their
  /// top left corners.
  RenderStack({
    List<RenderBox> children,
    AlignmentGeometry alignment = AlignmentDirectional.topStart,
    TextDirection textDirection,
    StackFit fit = StackFit.loose,
    Overflow overflow = Overflow.clip,
  }) : assert(alignment != null),
       assert(fit != null),
       assert(overflow != null),
       _alignment = alignment,
       _textDirection = textDirection,
       _fit = fit,
       _overflow = overflow {
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

  /// How to align the non-positioned or partially-positioned children in the
  /// stack.
  ///
  /// The non-positioned children are placed relative to each other such that
  /// the points determined by [alignment] are co-located. For example, if the
  /// [alignment] is [Alignment.topLeft], then the top left corner of
  /// each non-positioned child will be located at the same global coordinate.
  ///
  /// Partially-positioned children, those that do not specify an alignment in a
  /// particular axis (e.g. that have neither `top` nor `bottom` set), use the
  /// alignment to determine how they should be positioned in that
  /// under-specified axis.
  ///
  /// If this is set to an [AlignmentDirectional] object, then [textDirection]
  /// must not be null.
  AlignmentGeometry get alignment => _alignment;
  AlignmentGeometry _alignment;
  set alignment(AlignmentGeometry value) {
    assert(value != null);
    if (_alignment == value)
      return;
    _alignment = value;
    _markNeedResolution();
  }

  /// The text direction with which to resolve [alignment].
  ///
  /// This may be changed to null, but only after the [alignment] has been changed
  /// to a value that does not depend on the direction.
  TextDirection get textDirection => _textDirection;
  TextDirection _textDirection;
  set textDirection(TextDirection value) {
    if (_textDirection == value)
      return;
    _textDirection = value;
    _markNeedResolution();
  }

  /// How to size the non-positioned children in the stack.
  ///
  /// The constraints passed into the [RenderStack] from its parent are either
  /// loosened ([StackFit.loose]) or tightened to their biggest size
  /// ([StackFit.expand]).
  StackFit get fit => _fit;
  StackFit _fit;
  set fit(StackFit value) {
    assert(value != null);
    if (_fit != value) {
      _fit = value;
      markNeedsLayout();
    }
  }

  /// Whether overflowing children should be clipped. See [Overflow].
  ///
  /// Some children in a stack might overflow its box. When this flag is set to
  /// [Overflow.clip], children cannot paint outside of the stack's box.
  Overflow get overflow => _overflow;
  Overflow _overflow;
  set overflow(Overflow value) {
    assert(value != null);
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
      assert(child.parentData == childParentData);
      child = childParentData.nextSibling;
    }
    return extent;
  }

  @override
  double computeMinIntrinsicWidth(double height) {
    return _getIntrinsicDimension((RenderBox child) => child.getMinIntrinsicWidth(height));
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    return _getIntrinsicDimension((RenderBox child) => child.getMaxIntrinsicWidth(height));
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    return _getIntrinsicDimension((RenderBox child) => child.getMinIntrinsicHeight(width));
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    return _getIntrinsicDimension((RenderBox child) => child.getMaxIntrinsicHeight(width));
  }

  @override
  double computeDistanceToActualBaseline(TextBaseline baseline) {
    return defaultComputeDistanceToHighestActualBaseline(baseline);
  }

  @override
  void performLayout() {
    _resolve();
    assert(_resolvedAlignment != null);
    _hasVisualOverflow = false;
    bool hasNonPositionedChildren = false;
    if (childCount == 0) {
      size = constraints.biggest;
      assert(size.isFinite);
      return;
    }

    double width = constraints.minWidth;
    double height = constraints.minHeight;

    BoxConstraints nonPositionedConstraints;
    assert(fit != null);
    switch (fit) {
      case StackFit.loose:
        nonPositionedConstraints = constraints.loosen();
        break;
      case StackFit.expand:
        nonPositionedConstraints = BoxConstraints.tight(constraints.biggest);
        break;
      case StackFit.passthrough:
        nonPositionedConstraints = constraints;
        break;
    }
    assert(nonPositionedConstraints != null);

    RenderBox child = firstChild;
    while (child != null) {
      final StackParentData childParentData = child.parentData;

      if (!childParentData.isPositioned) {
        hasNonPositionedChildren = true;

        child.layout(nonPositionedConstraints, parentUsesSize: true);

        final Size childSize = child.size;
        width = math.max(width, childSize.width);
        height = math.max(height, childSize.height);
      }

      child = childParentData.nextSibling;
    }

    if (hasNonPositionedChildren) {
      size = Size(width, height);
      assert(size.width == constraints.constrainWidth(width));
      assert(size.height == constraints.constrainHeight(height));
    } else {
      size = constraints.biggest;
    }

    assert(size.isFinite);

    child = firstChild;
    while (child != null) {
      final StackParentData childParentData = child.parentData;

      if (!childParentData.isPositioned) {
        childParentData.offset = _resolvedAlignment.alongOffset(size - child.size);
      } else {
        BoxConstraints childConstraints = const BoxConstraints();

        if (childParentData.left != null && childParentData.right != null)
          childConstraints = childConstraints.tighten(width: size.width - childParentData.right - childParentData.left);
        else if (childParentData.width != null)
          childConstraints = childConstraints.tighten(width: childParentData.width);

        if (childParentData.top != null && childParentData.bottom != null)
          childConstraints = childConstraints.tighten(height: size.height - childParentData.bottom - childParentData.top);
        else if (childParentData.height != null)
          childConstraints = childConstraints.tighten(height: childParentData.height);

        child.layout(childConstraints, parentUsesSize: true);

        double x;
        if (childParentData.left != null) {
          x = childParentData.left;
        } else if (childParentData.right != null) {
          x = size.width - childParentData.right - child.size.width;
        } else {
          x = _resolvedAlignment.alongOffset(size - child.size).dx;
        }

        if (x < 0.0 || x + child.size.width > size.width)
          _hasVisualOverflow = true;

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

        childParentData.offset = Offset(x, y);
      }

      assert(child.parentData == childParentData);
      child = childParentData.nextSibling;
    }
  }

  @override
  bool hitTestChildren(BoxHitTestResult result, { Offset position }) {
    return defaultHitTestChildren(result, position: position);
  }

  /// Override in subclasses to customize how the stack paints.
  ///
  /// By default, the stack uses [defaultPaint]. This function is called by
  /// [paint] after potentially applying a clip to contain visual overflow.
  @protected
  void paintStack(PaintingContext context, Offset offset) {
    defaultPaint(context, offset);
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    if (_overflow == Overflow.clip && _hasVisualOverflow) {
      context.pushClipRect(needsCompositing, offset, Offset.zero & size, paintStack);
    } else {
      paintStack(context, offset);
    }
  }

  @override
  Rect describeApproximatePaintClip(RenderObject child) => _hasVisualOverflow ? Offset.zero & size : null;

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(DiagnosticsProperty<AlignmentGeometry>('alignment', alignment));
    properties.add(EnumProperty<TextDirection>('textDirection', textDirection));
    properties.add(EnumProperty<StackFit>('fit', fit));
    properties.add(EnumProperty<Overflow>('overflow', overflow));
  }
}
```


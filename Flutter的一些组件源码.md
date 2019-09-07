# Flutter的一些组件源码

[TOC]

##  RenderConstrainedBox 

```dart
class RenderConstrainedBox extends RenderProxyBox {
    /// 创造一个约束其孩子的渲染盒
    RenderConstrainedBox({
        RenderBox child,
        @required BoxConstraints additionalConstraints,
    }) : assert(additionalConstraints != null),
    assert(additionalConstraints.debugAssertIsValid()),
    _additionalConstraints = additionalConstraints,
    super(child);

    /// 额外约束仅仅在布局的时候施加给孩子
    BoxConstraints get additionalConstraints => _additionalConstraints;
    BoxConstraints _additionalConstraints;
    set additionalConstraints(BoxConstraints value) {
        if (_additionalConstraints == value)
            return;
        _additionalConstraints = value;
        /// 改变额外约束的时候会触发重建
        markNeedsLayout();
    }

    /// 计算最小固有宽
    @override
    double computeMinIntrinsicWidth(double height) {
        if (_additionalConstraints.hasBoundedWidth && 
            _additionalConstraints.hasTightWidth)
            return _additionalConstraints.minWidth;
        final double width = super.computeMinIntrinsicWidth(height);
        assert(width.isFinite);
        if (!_additionalConstraints.hasInfiniteWidth)
            return _additionalConstraints.constrainWidth(width);
        return width;
    }

    /// 计算最大固有宽
    @override
    double computeMaxIntrinsicWidth(double height) {
        if (_additionalConstraints.hasBoundedWidth && 
            _additionalConstraints.hasTightWidth)
            return _additionalConstraints.minWidth;
        final double width = super.computeMaxIntrinsicWidth(height);
        assert(width.isFinite);
        if (!_additionalConstraints.hasInfiniteWidth)
            return _additionalConstraints.constrainWidth(width);
        return width;
    }

    /// 计算最小固有高
    @override
    double computeMinIntrinsicHeight(double width) {
        if (_additionalConstraints.hasBoundedHeight && 
            _additionalConstraints.hasTightHeight)
            return _additionalConstraints.minHeight;
        final double height = super.computeMinIntrinsicHeight(width);
        assert(height.isFinite);
        if (!_additionalConstraints.hasInfiniteHeight)
            return _additionalConstraints.constrainHeight(height);
        return height;
    }

    /// 计算最大固有高
    @override
    double computeMaxIntrinsicHeight(double width) {
        if (_additionalConstraints.hasBoundedHeight && 
            _additionalConstraints.hasTightHeight)
            return _additionalConstraints.minHeight;
        final double height = super.computeMaxIntrinsicHeight(width);
        assert(height.isFinite);
        if (!_additionalConstraints.hasInfiniteHeight)
            return _additionalConstraints.constrainHeight(height);
        return height;
    }

    @override
    /// 布局
    void performLayout() {
        /// 有孩子的话，请求孩子布局
        if (child != null) {
            child.layout(
                /// 将自身的额外约束与父级传入的约束合并，作为孩子的约束。
                _additionalConstraints.enforce(constraints), 
                /// 是否使用孩子的大小作为自身的大小
                parentUsesSize: true
            );
            /// 将自身大小设为孩子的大小
            size = child.size;
        } else {
            /// 将自身的额外约束与父级传入的约束合并，并转换为大小。（取最小值，即尽量小）
            size = _additionalConstraints.enforce(constraints).constrain(Size.zero);
        }
    }
    ...
}


BoxConstraints enforce(BoxConstraints constraints) {
    return BoxConstraints(
        minWidth: minWidth.clamp(constraints.minWidth, constraints.maxWidth),
        maxWidth: maxWidth.clamp(constraints.minWidth, constraints.maxWidth),
        minHeight: minHeight.clamp(constraints.minHeight, constraints.maxHeight),
        maxHeight: maxHeight.clamp(constraints.minHeight, constraints.maxHeight),
    );
}
```



## RenderPositionedBox 

```dart
class RenderPositionedBox extends RenderAligningShiftedBox {
  /// 创建一个能够确定孩子位置的渲染对象
  RenderPositionedBox({
    RenderBox child,
    double widthFactor,
    double heightFactor,
    AlignmentGeometry alignment = Alignment.center,
    TextDirection textDirection,
  }) : assert(widthFactor == null || widthFactor >= 0.0),
       assert(heightFactor == null || heightFactor >= 0.0),
       _widthFactor = widthFactor,
       _heightFactor = heightFactor,
       super(child: child, alignment: alignment, textDirection: textDirection);

  /// 值改变时触发重新布局
  double get widthFactor => _widthFactor;
  double _widthFactor;
  set widthFactor(double value) {
    assert(value == null || value >= 0.0);
    if (_widthFactor == value)
      return;
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

  /// 布局  
    @override
    void performLayout() {
        final bool shrinkWrapWidth = _widthFactor != null || constraints.maxWidth == double.infinity;
        final bool shrinkWrapHeight = _heightFactor != null || constraints.maxHeight == double.infinity;
    	
        /// 如果有孩子，将父节点传入的约束宽松，作为子节点的约束，并请求其布局
        if (child != null) {
            child.layout(constraints.loosen(), parentUsesSize: true);
            size = constraints.constrain(
                Size(
                    shrinkWrapWidth ? 
                    child.size.width * (_widthFactor ?? 1.0) : double.infinity,
                    shrinkWrapHeight ? 
                    child.size.height * (_heightFactor ?? 1.0) : double.infinity
                )
            );
            alignChild();
        } else {
            size = constraints.constrain(
                Size(shrinkWrapWidth ? 0.0 : double.infinity,
                     shrinkWrapHeight ? 0.0 : double.infinity)
            );
        }
    }
 	...
}


/// Returns new box constraints that remove the minimum width and height requirements.
/// 返回一个移除了最小高度和宽度条件的渲染盒
BoxConstraints loosen() {
    assert(debugAssertIsValid());
    return BoxConstraints(
        minWidth: 0.0,
        maxWidth: maxWidth,
        minHeight: 0.0,
        maxHeight: maxHeight,
    );
}
```


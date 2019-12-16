# Flutter Opacity 源码分析

## Opcatity

```dart
/// 一个能更改子节点绘制透明度的组件
class Opacity extends SingleChildRenderObjectWidget {
  const Opacity({
    Key key,
    @required this.opacity,
    this.alwaysIncludeSemantics = false,
    Widget child,
  }) : assert(opacity != null && opacity >= 0.0 && opacity <= 1.0),
       assert(alwaysIncludeSemantics != null),
       super(key: key, child: child);
 
  /// 需要绘制的透明度
  /// 1.0 和 0.0 的透明度绘制时最快的，否则需要绘制在额外的一层上，开销较大
  final double opacity;

  final bool alwaysIncludeSemantics;

  @override
  RenderOpacity createRenderObject(BuildContext context) {
    return RenderOpacity(
      opacity: opacity,
      alwaysIncludeSemantics: alwaysIncludeSemantics,
    );
  }

  @override
  void updateRenderObject(BuildContext context, RenderOpacity renderObject) {
    renderObject
      ..opacity = opacity
      ..alwaysIncludeSemantics = alwaysIncludeSemantics;
  }
}
```



## RenderOpacity

```dart
class RenderOpacity extends RenderProxyBox {
  /// Opacity 的值必须在 0.0 和 1.0 之间
  RenderOpacity({
    double opacity = 1.0,
    bool alwaysIncludeSemantics = false,
    RenderBox child,
  }) : _opacity = opacity,
       _alwaysIncludeSemantics = alwaysIncludeSemantics,
       _alpha = ui.Color.getAlphaFromOpacity(opacity),
       super(child);

  /// 显然，当透明度为 0.0 或 1.0 或没有子节点的时候，不需要重新开启一个图层来绘制
  /// 否则，总是需要新开一个图层来绘制透明度变化
  @override
  bool get alwaysNeedsCompositing => child != null && (_alpha != 0 && _alpha != 255);

  int _alpha;

  /// 更改透明的时候会触发重绘
  double get opacity => _opacity;
  double _opacity;
  set opacity(double value) {
    if (_opacity == value)
      return;
    final bool didNeedCompositing = alwaysNeedsCompositing;
    final bool wasVisible = _alpha != 0;
    _opacity = value;
    _alpha = ui.Color.getAlphaFromOpacity(_opacity);
    if (didNeedCompositing != alwaysNeedsCompositing)
      markNeedsCompositingBitsUpdate();
    markNeedsPaint();
    if (wasVisible != (_alpha != 0) && !alwaysIncludeSemantics)
      markNeedsSemanticsUpdate();
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    if (child != null) {
      /// 当透明度为 0 时，不需要绘制子节点，也不用保留图层
      if (_alpha == 0) {
        layer = null;
        return;
      }
      /// 当透明度为 1 时，只需要将子节点绘制到当前图层上，也不用保留原先的透明度图层
      if (_alpha == 255) {
        layer = null;
        context.paintChild(child, offset);
        return;
      }
      assert(needsCompositing);
      /// 否则，应该新建一个透明度图层，并将这个绘制方法作为回调传入
      layer = context.pushOpacity(offset, _alpha, super.paint, oldLayer: layer);
    }
  }
}
```



关于透明度图层

```dart
/// 这个方法只新建一个透明度层
OpacityLayer pushOpacity(
    Offset offset, 
    int alpha, 
    PaintingContextCallback painter, { 
    OpacityLayer oldLayer 
}) {
    final OpacityLayer layer = oldLayer ?? OpacityLayer();
    layer
        ..alpha = alpha
        ..offset = offset;
    pushLayer(layer, painter, Offset.zero);
    return layer;
}

/// 将给定的图层当成当前图层的子节点
void pushLayer(
    ContainerLayer childLayer, 
    PaintingContextCallback painter, 
    Offset offset, { 
    Rect childPaintBounds 
}) {
    if (childLayer.hasChildren) {
        childLayer.removeAllChildren();
    }
    /// 停止在当前图层 PictureLayer 上的录制，并获取到 _picture，然后清空引用
    stopRecordingIfNeeded();
    appendLayer(childLayer);
    /// void appendLayer(Layer layer) {
    ///   assert(!_isRecording);
    ///   layer.remove();
    ///   /// 附加到当前容器图层上，作为 pictureLayer 的右兄弟节点
    ///   _containerLayer.append(layer);
    /// }
    /// 然后使用给定的 childLayer 当作 containerLayer 创建新的 Context
    final PaintingContext childContext = 
        createChildContext(childLayer, childPaintBounds ?? estimatedBounds);
    /// 用新创建的 PaintContext 做为参数来绘制
    /// 注意：传入的 painter 其实是 RenderObject.paint 方法，其中通常会获取 Canvas
    /// 而获取 Canvas 会新建一个 PictureLayer 用于记录绘制，而这个 PictuerLayer 会作为 
    /// PaintContext.containerLayer 的孩子。也就是说，经过以下 painter 调用，实际上实在 childLayer 上绘制
    ///（即运用了 childLayer 的效果（例如透明度和 transform ）来绘制
    painter(childContext, offset);
    childContext.stopRecordingIfNeeded();
}
```



通俗来说，Opacity 组件其实是操作了 RenderOpacity 来新建一个 透明度图层，然后框架会自动将 Opacity 的子树绘制在这个透明度图层上。

与此类似地，BackdropFilter，Transform 也是采用了新建变换图层的方式来影响其子树的绘制效果
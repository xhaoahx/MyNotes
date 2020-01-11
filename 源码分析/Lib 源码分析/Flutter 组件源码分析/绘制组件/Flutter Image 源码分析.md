# Flutter Image 源码分析

## 函数签名

```dart
typedef ImageFrameBuilder = Widget Function(
  BuildContext context,
  Widget child,
  int frame,
  bool wasSynchronouslyLoaded,
);

typedef ImageLoadingBuilder = Widget Function(
  BuildContext context,
  Widget child,
  ImageChunkEvent loadingProgress,
);
```



## Image

```dart
/// A widget that displays an image.
///
/// 提供了以下几种构造器
///
///  * [new Image], 从 [ImageProvider] 获取图片
///  * [new Image.asset], 从 [AssetBundle] 使用 key 来获取图片
///  * [new Image.network], 使用 url 获取图片.
///  * [new Image.file], 从 [File] 中获取图片
///  * [new Image.memory], 从 [Uint8List] 获取图片
///
/// 如果想使用与像素密度有关的图片，请确保在 Image 组件上层使用了 [MaterialApp]，[WidgetsApp] 或者, 
/// [MediaQuery] 组件
///
/// 图片使用 [paintImage] 方法来绘制
class Image extends StatefulWidget {
  /// 
  const Image({
    Key key,
    /// 必须给出 ImageProvider
    @required this.image,
    this.frameBuilder,
    this.loadingBuilder,
    this.semanticLabel,
    this.excludeFromSemantics = false,
    this.width,
    this.height,
    this.color,
    this.colorBlendMode,
    this.fit,
    this.alignment = Alignment.center,
    this.repeat = ImageRepeat.noRepeat,
    this.centerSlice,
    this.matchTextDirection = false,
    this.gaplessPlayback = false,
    this.filterQuality = FilterQuality.low,
  }) : super(key: key);

  Image.network(
    String src, {
    Key key,
    double scale = 1.0,
    this.frameBuilder,
    this.loadingBuilder,
    this.semanticLabel,
    this.excludeFromSemantics = false,
    this.width,
    this.height,
    this.color,
    this.colorBlendMode,
    this.fit,
    this.alignment = Alignment.center,
    this.repeat = ImageRepeat.noRepeat,
    this.centerSlice,
    this.matchTextDirection = false,
    this.gaplessPlayback = false,
    this.filterQuality = FilterQuality.low,
    Map<String, String> headers,
    int cacheWidth,
    int cacheHeight,
  }) : image = ResizeImage.resizeIfNeeded(
      cacheWidth, 
      cacheHeight, 
      NetworkImage(
          src, 
          scale: scale, 
          headers: headers
      )
  ),
  super(key: key);

 
  Image.file(
    File file, {
    Key key,
    double scale = 1.0,
    this.frameBuilder,
    this.semanticLabel,
    this.excludeFromSemantics = false,
    this.width,
    this.height,
    this.color,
    this.colorBlendMode,
    this.fit,
    this.alignment = Alignment.center,
    this.repeat = ImageRepeat.noRepeat,
    this.centerSlice,
    this.matchTextDirection = false,
    this.gaplessPlayback = false,
    this.filterQuality = FilterQuality.low,
    int cacheWidth,
    int cacheHeight,
  }) : image = ResizeImage.resizeIfNeeded(
      cacheWidth, 
      cacheHeight, 
      FileImage(
          file, 
          scale: scale)
 	  ),
      super(key: key);

  /// 对图片使用 package 如下
  /// Assets used by the package itself should also be displayed using the
  /// [package] argument as above.
  ///
  /// If the desired asset is specified in the `pubspec.yaml` of the package, it
  /// is bundled automatically with the app. In particular, assets used by the
  /// package itself must be specified in its `pubspec.yaml`.
  ///
  /// A package can also choose to have assets in its 'lib/' folder that are not
  /// specified in its `pubspec.yaml`. In this case for those images to be
  /// bundled, the app has to specify which ones to include. For instance a
  /// package named `fancy_backgrounds` could have:
  ///
  /// ```
  /// lib/backgrounds/background1.png
  /// lib/backgrounds/background2.png
  /// lib/backgrounds/background3.png
  /// ```
  ///
  /// To include, say the first image, the `pubspec.yaml` of the app should
  /// specify it in the assets section:
  ///
  /// ```yaml
  ///   assets:
  ///     - packages/fancy_backgrounds/backgrounds/background1.png
  /// ```
  ///
  /// The `lib/` is implied, so it should not be included in the asset path.
  Image.asset(
    String name, {
    Key key,
    AssetBundle bundle,
    this.frameBuilder,
    this.semanticLabel,
    this.excludeFromSemantics = false,
    double scale,
    this.width,
    this.height,
    this.color,
    this.colorBlendMode,
    this.fit,
    this.alignment = Alignment.center,
    this.repeat = ImageRepeat.noRepeat,
    this.centerSlice,
    this.matchTextDirection = false,
    this.gaplessPlayback = false,
    String package,
    this.filterQuality = FilterQuality.low,
    int cacheWidth,
    int cacheHeight,
  }) : image = ResizeImage.resizeIfNeeded(
      cacheWidth, 
      cacheHeight, 
      scale != null
         ? ExactAssetImage(
             name, 
             bundle: bundle, 
             scale: scale, 
             package: package)
         : AssetImage(
             name, 
             bundle: bundle, 
             package: package
         )
      ),
      loadingBuilder = null,
      super(key: key);

  
  Image.memory(
    Uint8List bytes, {
    Key key,
    double scale = 1.0,
    this.frameBuilder,
    this.semanticLabel,
    this.excludeFromSemantics = false,
    this.width,
    this.height,
    this.color,
    this.colorBlendMode,
    this.fit,
    this.alignment = Alignment.center,
    this.repeat = ImageRepeat.noRepeat,
    this.centerSlice,
    this.matchTextDirection = false,
    this.gaplessPlayback = false,
    this.filterQuality = FilterQuality.low,
    int cacheWidth,
    int cacheHeight,
  }) : image = ResizeImage.resizeIfNeeded(
      cacheWidth, 
      cacheHeight, 
      MemoryImage(
          bytes, 
          scale: scale)
      ),
      loadingBuilder = null,
      super(key: key);

  /// 被展示的图片
  final ImageProvider image;

  final ImageFrameBuilder frameBuilder;

  /// 指定图片加载时的组件
  /// 
  /// 其中提供的 ChunkEvent 可用于指示图片加载百分比
  final ImageLoadingBuilder loadingBuilder;

  /// 要求的图片的宽高
  final double width;
  final double height;

  /// 如果非 null，则使用 [colorBlendMode] 将此颜色与图片的每一像素混合
  final Color color;

  /// Used to set the [FilterQuality] of the image.
  ///
  /// Use the [FilterQuality.low] quality setting to scale the image with
  /// bilinear interpolation, or the [FilterQuality.none] which corresponds
  /// to nearest-neighbor.
  final FilterQuality filterQuality;

  /// 颜色的混合模式
  final BlendMode colorBlendMode;

  /// 图片如何布局
  final BoxFit fit;

  /// 图片的排列
  final AlignmentGeometry alignment;

  /// 如果图片没能覆盖整个边界，则使用此“重复”效果
  final ImageRepeat repeat;

  /// 中心切片为9块图像。
  ///
  /// 中心切片内的图像区域将被水平和垂直拉伸，以适应图像的目标。图像中心切片上下的区域只会被水平拉伸，图像中心
  /// 切片左右的区域只会被垂直拉伸。
  ///
  /// 应用为 .9 图
  final Rect centerSlice;
    
  final bool matchTextDirection;

  /// 当 image provider 发生改变的是否，是否继续尝试原有的图片 (true),或者什么也不显示 (false)
  final bool gaplessPlayback;

  final String semanticLabel;
  final bool excludeFromSemantics;

  @override
  _ImageState createState() => _ImageState();
}
```

## _ImageState
```dart
/// 注意：
/// 这里混入了 WidgetsBindingObserver，
/// 可用监听一系列原生事件
class _ImageState extends State<Image> with WidgetsBindingObserver {
  ImageStream _imageStream;
  ImageInfo _imageInfo;
  ImageChunkEvent _loadingProgress;
  bool _isListeningToStream = false;
  bool _invertColors;
  int _frameNumber;
  bool _wasSynchronouslyLoaded;

  @override
  void initState() {
    super.initState();
    /// 开始观测事件
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void dispose() {
    /// 停止观测
    WidgetsBinding.instance.removeObserver(this);
    _stopListeningToStream();
    super.dispose();
  }

  /// 注意：
  /// didChangeDependencies 方法会在 initState 之后立刻调用一次，用于更新依赖
  /// 并且会在依赖更新之后调用
  @override
  void didChangeDependencies() {
    _updateInvertColors();
    /// 从这里开始获取图片
    _resolveImage();

    /// 如果依赖的 tickerMode 为 true，也就是允许动画的话，那么，监听流（可能此图片是多帧的并且有对应的动画效
    /// 果）以便完成动画
    if (TickerMode.of(context))
      _listenToStream();
    else
      _stopListeningToStream();

    super.didChangeDependencies();
  }

  @override
  void didUpdateWidget(Image oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (_isListeningToStream 
        && (widget.loadingBuilder == null) != (oldWidget.loadingBuilder == null)
    ) {
      _imageStream.removeListener(_getListener(oldWidget.loadingBuilder));
      _imageStream.addListener(_getListener());
    }
    if (widget.image != oldWidget.image)
      _resolveImage();
  }

  @override
  void didChangeAccessibilityFeatures() {
    super.didChangeAccessibilityFeatures();
    setState(() {
      _updateInvertColors();
    });
  }

  @override
  void reassemble() {
    _resolveImage(); // in case the image cache was flushed
    super.reassemble();
  }

 /// 改变图片颜色，便于视障人士使用
  void _updateInvertColors() {
    _invertColors = MediaQuery.of(context, nullOk: true)?.invertColors
        ?? SemanticsBinding.instance.accessibilityFeatures.invertColors;
  }

  void _resolveImage() {
    final ImageStream newStream = widget.image.resolve(
        createLocalImageConfiguration(
           context,
           /// 注意：
           /// 只有在 widget 中同时指定了宽高，才会被采用
           size: widget.width != null && widget.height != null 
               ? Size(
                   widget.width, 
                   widget.height
               ) : null,
      	)
    );
    _updateSourceStream(newStream);
  }

  /// 从已有的配置中得到一个 监听器
  ImageStreamListener _getListener([ImageLoadingBuilder loadingBuilder]) {
    loadingBuilder ??= widget.loadingBuilder;
    return ImageStreamListener(
      /// 这里传入的是 onImage 回调，即，当每有一帧图片完成的时候，调用此回调
      _handleImageFrame,
      onChunk: loadingBuilder == null ? null : _handleImageChunk,
    );
  }

  void _handleImageFrame(ImageInfo imageInfo, bool synchronousCall) {
    setState(() {
      _imageInfo = imageInfo;
      _loadingProgress = null;
      _frameNumber = _frameNumber == null ? 0 : _frameNumber + 1;
      _wasSynchronouslyLoaded |= synchronousCall;
    });
  }

  void _handleImageChunk(ImageChunkEvent event) {
    setState(() {
      _loadingProgress = event;
    });
  }

  // 将 _imageStream 更新成 newStream，并且将其监听器转移到新的图片流上
  void _updateSourceStream(ImageStream newStream) {
    /// 这意味着两个流是相等的
    if (_imageStream?.key == newStream?.key)
      return;

    if (_isListeningToStream)
      _imageStream.removeListener(_getListener());

    if (!widget.gaplessPlayback)
      setState(() { _imageInfo = null; });

    setState(() {
      _loadingProgress = null;
      _frameNumber = null;
      _wasSynchronouslyLoaded = false;
    });

    _imageStream = newStream;
    if (_isListeningToStream)
      _imageStream.addListener(_getListener());
  }

  /// 开始监听 _imageStream
  void _listenToStream() {
    if (_isListeningToStream)
      return;
    _imageStream.addListener(_getListener());
    _isListeningToStream = true;
  }

  /// 停止监听 _imageStream
  void _stopListeningToStream() {
    if (!_isListeningToStream)
      return;
    _imageStream.removeListener(_getListener());
    _isListeningToStream = false;
  }

  @override
  Widget build(BuildContext context) {
    /// 底层使用的组件是 RawImage
    Widget result = RawImage(
      /// 注意：
      /// 这里的 image 是 UI.Image；图片还没有加载完成的时候，是 null
      image: _imageInfo?.image,
      width: widget.width,
      height: widget.height,
      scale: _imageInfo?.scale ?? 1.0,
      color: widget.color,
      colorBlendMode: widget.colorBlendMode,
      fit: widget.fit,
      alignment: widget.alignment,
      repeat: widget.repeat,
      centerSlice: widget.centerSlice,
      matchTextDirection: widget.matchTextDirection,
      invertColors: _invertColors,
      filterQuality: widget.filterQuality,
    );

    if (!widget.excludeFromSemantics) {
      result = Semantics(
        container: widget.semanticLabel != null,
        image: true,
        label: widget.semanticLabel ?? '',
        child: result,
      );
    }

    /// 如果 widget 的 frameBuilder 不为 null 的话，在外面套一层
    if (widget.frameBuilder != null)
      result = widget.frameBuilder(context, result, _frameNumber, _wasSynchronouslyLoaded);

    /// 如果 widget 的 loadingBuilder 不为 null 的话，在外面套一层
    if (widget.loadingBuilder != null)
      result = widget.loadingBuilder(context, result, _loadingProgress);

    return result;
  }
}
```


## RawImage
```dart
/// 直接展示 [dart:ui.Image] 的组件
class RawImage extends LeafRenderObjectWidget {
  const RawImage({
    Key key,
    this.image,
    this.width,
    this.height,
    this.scale = 1.0,
    this.color,
    this.colorBlendMode,
    this.fit,
    this.alignment = Alignment.center,
    this.repeat = ImageRepeat.noRepeat,
    this.centerSlice,
    this.matchTextDirection = false,
    this.invertColors = false,
    this.filterQuality = FilterQuality.low,
  }) : super(key: key);

  /// 被用于展示的图片
  final ui.Image image;


  final double width;
  final double height;

  /// Specifies the image's scale.
  ///
  /// Used when determining the best display size for the image.
  final double scale;
  final Color color;
  final FilterQuality filterQuality;
  final BlendMode colorBlendMode;
  final BoxFit fit;
  final AlignmentGeometry alignment;
  final ImageRepeat repeat;
  final Rect centerSlice;
  final bool matchTextDirection;
  final bool invertColors;

  @override
  RenderImage createRenderObject(BuildContext context) {
    return RenderImage(
      image: image,
      width: width,
      height: height,
      scale: scale,
      color: color,
      colorBlendMode: colorBlendMode,
      fit: fit,
      alignment: alignment,
      repeat: repeat,
      centerSlice: centerSlice,
      matchTextDirection: matchTextDirection,
      textDirection: matchTextDirection || alignment is! Alignment ? Directionality.of(context) : null,
      invertColors: invertColors,
      filterQuality: filterQuality,
    );
  }

  @override
  void updateRenderObject(BuildContext context, RenderImage renderObject) {
    renderObject
      ..image = image
      ..width = width
      ..height = height
      ..scale = scale
      ..color = color
      ..colorBlendMode = colorBlendMode
      ..alignment = alignment
      ..fit = fit
      ..repeat = repeat
      ..centerSlice = centerSlice
      ..matchTextDirection = matchTextDirection
      ..textDirection = matchTextDirection || alignment is! Alignment ? Directionality.of(context) : null
      ..invertColors = invertColors
      ..filterQuality = filterQuality;
  }
}
```



## RenderImage

```dart
/// 渲染图片
///
/// 渲染图像试图为自己找到一个大小，适合给定的约束，并保持图像的内在高宽比。
class RenderImage extends RenderBox {

  RenderImage({
    ui.Image image,
    double width,
    double height,
    double scale = 1.0,
    Color color,
    BlendMode colorBlendMode,
    BoxFit fit,
    AlignmentGeometry alignment = Alignment.center,
    ImageRepeat repeat = ImageRepeat.noRepeat,
    Rect centerSlice,
    bool matchTextDirection = false,
    TextDirection textDirection,
    bool invertColors = false,
    FilterQuality filterQuality = FilterQuality.low,
  }) : _image = image,
       _width = width,
       _height = height,
       _scale = scale,
       _color = color,
       _colorBlendMode = colorBlendMode,
       _fit = fit,
       _alignment = alignment,
       _repeat = repeat,
       _centerSlice = centerSlice,
       _matchTextDirection = matchTextDirection,
       _invertColors = invertColors,
       _textDirection = textDirection,
       _filterQuality = filterQuality 
  {
    _updateColorFilter();
  }

  Alignment _resolvedAlignment;
  bool _flipHorizontally;

  void _resolve() {
    if (_resolvedAlignment != null)
      return;
    _resolvedAlignment = alignment.resolve(textDirection);
    _flipHorizontally = matchTextDirection && textDirection == TextDirection.rtl;
  }

  void _markNeedResolution() {
    _resolvedAlignment = null;
    _flipHorizontally = null;
    markNeedsPaint();
  }

  /// The image to display.
  ui.Image get image => _image;
  ui.Image _image;
  set image(ui.Image value) {
    if (value == _image)
      return;
    _image = value;
    /// 图片更新时需要重绘
    markNeedsPaint();
    /// 当宽高其一没有被指定，则需要重新布局（如果在 widegt 中同时指定了宽高，有利于提高效率）
    if (_width == null || _height == null)
      markNeedsLayout();
  }

  /// 宽高改变的时候，需要重新布局
  double get width => _width;
  double _width;
  set width(double value) {
    if (value == _width)
      return;
    _width = value;
    markNeedsLayout();
  }

  double get height => _height;
  double _height;
  set height(double value) {
    if (value == _height)
      return;
    _height = value;
    markNeedsLayout();
  }

  /// 比例发生改变的时候，需要重新布局
  double get scale => _scale;
  double _scale;
  set scale(double value) {
    assert(value != null);
    if (value == _scale)
      return;
    _scale = value;
    markNeedsLayout();
  }

  ColorFilter _colorFilter;

  void _updateColorFilter() {
    if (_color == null)
      _colorFilter = null;
    else
      _colorFilter = ColorFilter.mode(_color, _colorBlendMode ?? BlendMode.srcIn);
  }

  Color get color => _color;
  Color _color;
  set color(Color value) {
    if (value == _color)
      return;
    _color = value;
    _updateColorFilter();
    markNeedsPaint();
  }

  FilterQuality get filterQuality => _filterQuality;
  FilterQuality _filterQuality;
  set filterQuality(FilterQuality value) {
    assert(value != null);
    if (value == _filterQuality)
      return;
    _filterQuality = value;
    markNeedsPaint();
  }

  BlendMode get colorBlendMode => _colorBlendMode;
  BlendMode _colorBlendMode;
  set colorBlendMode(BlendMode value) {
    if (value == _colorBlendMode)
      return;
    _colorBlendMode = value;
    _updateColorFilter();
    markNeedsPaint();
  }

  /// 如何将图像放置到布局期间分配的空间。
  ///
  /// 注意，如果此字段发生了改变，则只需要重新绘制而无需布局
  BoxFit get fit => _fit;
  BoxFit _fit;
  set fit(BoxFit value) {
    if (value == _fit)
      return;
    _fit = value;
    markNeedsPaint();
  }

  /// How to align the image within its bounds.
  ///
  /// If this is set to a text-direction-dependent value, [textDirection] must
  /// not be null.
  AlignmentGeometry get alignment => _alignment;
  AlignmentGeometry _alignment;
  set alignment(AlignmentGeometry value) {
    assert(value != null);
    if (value == _alignment)
      return;
    _alignment = value;
    _markNeedResolution();
  }

  ImageRepeat get repeat => _repeat;
  ImageRepeat _repeat;
  set repeat(ImageRepeat value) {
    assert(value != null);
    if (value == _repeat)
      return;
    _repeat = value;
    markNeedsPaint();
  }

  Rect get centerSlice => _centerSlice;
  Rect _centerSlice;
  set centerSlice(Rect value) {
    if (value == _centerSlice)
      return;
    _centerSlice = value;
    markNeedsPaint();
  }

  bool get invertColors => _invertColors;
  bool _invertColors;
  set invertColors(bool value) {
    if (value == _invertColors)
      return;
    _invertColors = value;
    markNeedsPaint();
  }

  bool get matchTextDirection => _matchTextDirection;
  bool _matchTextDirection;
  set matchTextDirection(bool value) {
    assert(value != null);
    if (value == _matchTextDirection)
      return;
    _matchTextDirection = value;
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

  /// 从给定的 constraints 中获取渲染此图片的大小
  ///
  ///  - 图片的渲染大小必须在给定的约束以内
  ///  - 图片的长宽比应该遵循图片固有的长宽比
  ///  - The RenderImage's dimension are maximal subject to being smaller than
  ///    the intrinsic size of the image.
  Size _sizeForConstraints(BoxConstraints constraints) {
    // Folds the given |width| and |height| into |constraints| so they can all
    // be treated uniformly.
    constraints = BoxConstraints.tightFor(
      width: _width,
      height: _height,
    ).enforce(constraints);
	
    /// 如果没有图片（未加载完成的状态），则使用约束的最小值
    if (_image == null)
      return constraints.smallest;

    /// 试图在维持原始宽高比的情况下约束大小
    return constraints.constrainSizeAndAttemptToPreserveAspectRatio(Size(
      _image.width.toDouble() / _scale,
      _image.height.toDouble() / _scale,
    ));
  }

  @override
  double computeMinIntrinsicWidth(double height) {
    assert(height >= 0.0);
    if (_width == null && _height == null)
      return 0.0;
    return _sizeForConstraints(BoxConstraints.tightForFinite(height: height)).width;
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    assert(height >= 0.0);
    return _sizeForConstraints(BoxConstraints.tightForFinite(height: height)).width;
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    assert(width >= 0.0);
    if (_width == null && _height == null)
      return 0.0;
    return _sizeForConstraints(BoxConstraints.tightForFinite(width: width)).height;
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    assert(width >= 0.0);
    return _sizeForConstraints(BoxConstraints.tightForFinite(width: width)).height;
  }

  /// 由于此渲染对象覆盖了其 Size（即没有 Padding），因此 Size 内的每一点都是可以点击的，直接返回 true
  @override
  bool hitTestSelf(Offset position) => true;

  @override
  void performLayout() {
    /// 在布局期间得到 size
    size = _sizeForConstraints(constraints);
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    if (_image == null)
      return;
    _resolve();

    /// decoration_image.dart 中的全局方法，在给定的 Canvas 上绘制图片
    /// （其实现还是调用了 canvas.drawImageRect 方法）
    paintImage(
      canvas: context.canvas,
      rect: offset & size,
      image: _image,
      scale: _scale,
      colorFilter: _colorFilter,
      fit: _fit,
      alignment: _resolvedAlignment,
      centerSlice: _centerSlice,
      repeat: _repeat,
      flipHorizontally: _flipHorizontally,
      invertColors: invertColors,
      filterQuality: _filterQuality,
    );
  }
}

```


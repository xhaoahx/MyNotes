# Flutter FadeInImage 源码分析



## FadeInImage
```dart
class FadeInImage extends StatelessWidget {
  /// 常量构造器
  const FadeInImage({
    Key key,
    /// 图片尚未加载完成时的占位符
    @required this.placeholder,
    @required this.image,
    this.excludeFromSemantics = false,
    this.imageSemanticLabel,
    this.fadeOutDuration = const Duration(milliseconds: 300),
    this.fadeOutCurve = Curves.easeOut,
    this.fadeInDuration = const Duration(milliseconds: 700),
    this.fadeInCurve = Curves.easeIn,
    this.width,
    this.height,
    this.fit,
    this.alignment = Alignment.center,
    this.repeat = ImageRepeat.noRepeat,
    this.matchTextDirection = false,
  }) : super(key: key);
  
  
  FadeInImage.memoryNetwork({
    Key key,
    /// 使用内存图片作为 占位符
    @required Uint8List placeholder,
    /// 图片 url
    @required String image,
    double placeholderScale = 1.0,
    double imageScale = 1.0,
    this.excludeFromSemantics = false,
    this.imageSemanticLabel,
    this.fadeOutDuration = const Duration(milliseconds: 300),
    this.fadeOutCurve = Curves.easeOut,
    this.fadeInDuration = const Duration(milliseconds: 700),
    this.fadeInCurve = Curves.easeIn,
    this.width,
    this.height,
    this.fit,
    this.alignment = Alignment.center,
    this.repeat = ImageRepeat.noRepeat,
    this.matchTextDirection = false,
    int placeholderCacheWidth,
    int placeholderCacheHeight,
    int imageCacheWidth,
    int imageCacheHeight,
  }) : placeholder = ResizeImage.resizeIfNeeded(
      	placeholderCacheWidth, 
      	placeholderCacheHeight, 
      	MemoryImage(
          placeholder, 
          scale: placeholderScale
      	)
      ),
      image = ResizeImage.resizeIfNeeded(
          imageCacheWidth, 
          imageCacheHeight, 
          NetworkImage(
              image, 
              scale: imageScale
          )
      ),
      super(key: key);

  FadeInImage.assetNetwork({
    Key key,
    @required String placeholder,
    @required String image,
    AssetBundle bundle,
    double placeholderScale,
    double imageScale = 1.0,
    this.excludeFromSemantics = false,
    this.imageSemanticLabel,
    this.fadeOutDuration = const Duration(milliseconds: 300),
    this.fadeOutCurve = Curves.easeOut,
    this.fadeInDuration = const Duration(milliseconds: 700),
    this.fadeInCurve = Curves.easeIn,
    this.width,
    this.height,
    this.fit,
    this.alignment = Alignment.center,
    this.repeat = ImageRepeat.noRepeat,
    this.matchTextDirection = false,
    int placeholderCacheWidth,
    int placeholderCacheHeight,
    int imageCacheWidth,
    int imageCacheHeight,
  }) : placeholder = 
      placeholderScale != null
      ? ResizeImage.resizeIfNeeded(
    		placeholderCacheWidth, 
      		placeholderCacheHeight, 
      		ExactAssetImage(
          		placeholder, 
          		bundle: bundle, 
          		scale: placeholderScale
           )
  	  ) : ResizeImage.resizeIfNeeded(
      		placeholderCacheWidth, 
      		placeholderCacheHeight, 
      		AssetImage(
                   placeholder, 
                   bundle: bundle
            )
       ),
       image = ResizeImage.resizeIfNeeded(
           imageCacheWidth, 
           imageCacheHeight, 
           NetworkImage(
               image, 
               scale: imageScale
           )
       ),
       super(key: key);

  /// 图片尚未加载完成时的占位符
  final ImageProvider placeholder;
  /// 要显示的图片
  final ImageProvider image;
  /// 淡出间隔
  final Duration fadeOutDuration;
  /// 淡出曲线
  final Curve fadeOutCurve;
  /// 淡入间隔
  final Duration fadeInDuration;
  /// 淡入曲线
  final Curve fadeInCurve;

  final double width;
  final double height;
  final BoxFit fit;
  final AlignmentGeometry alignment;
  final ImageRepeat repeat;
  final bool matchTextDirection;
  final bool excludeFromSemantics;
  final String imageSemanticLabel;

  /// 利用给定的 provider 构建 Image 组件（widget）
  Image _image({
    @required ImageProvider image,
    ImageFrameBuilder frameBuilder,
  }) {
    assert(image != null);
    return Image(
      image: image,
      frameBuilder: frameBuilder,
      width: width,
      height: height,
      fit: fit,
      alignment: alignment,
      repeat: repeat,
      matchTextDirection: matchTextDirection,
      gaplessPlayback: true,
      excludeFromSemantics: true,
    );
  }

  @override
  Widget build(BuildContext context) {
    Widget result = _image(
      image: image,
      frameBuilder: (BuildContext context, Widget child, int frame, bool wasSynchronouslyLoaded) {
        /// 如果是同步地加载，则直接返回 child
        if (wasSynchronouslyLoaded)
          return child;
        /// 返回一个 ImplicitlyAnimatedWidget
        return _AnimatedFadeOutFadeIn(
          target: child,
          placeholder: _image(image: placeholder),
          isTargetLoaded: frame != null,
          fadeInDuration: fadeInDuration,
          fadeOutDuration: fadeOutDuration,
          fadeInCurve: fadeInCurve,
          fadeOutCurve: fadeOutCurve,
        );
      },
    );

    if (!excludeFromSemantics) {
      result = Semantics(
        container: imageSemanticLabel != null,
        image: true,
        label: imageSemanticLabel ?? '',
        child: result,
      );
    }

    return result;
  }
}
```



## _AnimatedFadeOutFadeIn

```dart
class _AnimatedFadeOutFadeIn extends ImplicitlyAnimatedWidget {
  const _AnimatedFadeOutFadeIn({
    Key key,
    @required this.target,
    @required this.placeholder,
    @required this.isTargetLoaded,
    @required this.fadeOutDuration,
    @required this.fadeOutCurve,
    @required this.fadeInDuration,
    @required this.fadeInCurve,
  }) : super(
      key: key, 
      duration: fadeInDuration + fadeOutDuration
  );

  final Widget target;
  final Widget placeholder;
  final bool isTargetLoaded;
  final Duration fadeInDuration;
  final Duration fadeOutDuration;
  final Curve fadeInCurve;
  final Curve fadeOutCurve;

  @override
  _AnimatedFadeOutFadeInState createState() => _AnimatedFadeOutFadeInState();
}
```



## _AnimatedFadeOutFadeInState

```dart
class _AnimatedFadeOutFadeInState extends ImplicitlyAnimatedWidgetState<_AnimatedFadeOutFadeIn> {
  Tween<double> _targetOpacity;
  Tween<double> _placeholderOpacity;
  Animation<double> _targetOpacityAnimation;
  Animation<double> _placeholderOpacityAnimation;

  @override
  void forEachTween(TweenVisitor<dynamic> visitor) {
    /// 目标图片的透明度补间
    _targetOpacity = visitor(
      _targetOpacity,
      widget.isTargetLoaded ? 1.0 : 0.0,
      (dynamic value) => Tween<double>(begin: value as double),
    ) as Tween<double>;
    /// 占位符的透明度补间
    _placeholderOpacity = visitor(
      _placeholderOpacity,
      widget.isTargetLoaded ? 0.0 : 1.0,
      (dynamic value) => Tween<double>(begin: value as double),
    ) as Tween<double>;
  }

  @override
  void didUpdateTweens() {
    /// 在 0 到 fadeOutDuration 的时间内是 _placeholderOpacity 对应的动画值，
    /// 在 fadeInDuration 到结束的时间内是 0.0
    _placeholderOpacityAnimation = animation.drive(
        TweenSequence<double>(<TweenSequenceItem<double>>[
      		TweenSequenceItem<double>(
        		tween: _placeholderOpacity.chain(CurveTween(curve: widget.fadeOutCurve)),
        		weight: widget.fadeOutDuration.inMilliseconds.toDouble(),
      		),
      		TweenSequenceItem<double>(
        		tween: ConstantTween<double>(0),
        		weight: widget.fadeInDuration.inMilliseconds.toDouble(),
      		),
   		])
    );
    /// 在 0 到 fadeOutDuration 的时间内内是 0.0
    /// 在 fadeInDuration 到结束的时间是 _targetOpacity 对应的动画值，
    _targetOpacityAnimation = animation.drive(
        TweenSequence<double>(<TweenSequenceItem<double>>[
      		TweenSequenceItem<double>(
        		tween: ConstantTween<double>(0),
        		weight: widget.fadeOutDuration.inMilliseconds.toDouble(),
      		),
      		TweenSequenceItem<double>(
        		tween: _targetOpacity.chain(CurveTween(curve: widget.fadeInCurve)),
        		weight: widget.fadeInDuration.inMilliseconds.toDouble(),
      		),
    	])
    );
    if (!widget.isTargetLoaded && _isValid(_placeholderOpacity) && _isValid(_targetOpacity)) {
      // Jump (don't fade) back to the placeholder image, so as to be ready
      // for the full animation when the new target image becomes ready.
      controller.value = controller.upperBound;
    }
  }

  bool _isValid(Tween<double> tween) {
    return tween.begin != null && tween.end != null;
  }

  @override
  Widget build(BuildContext context) {
    return Stack(
      fit: StackFit.passthrough,
      alignment: AlignmentDirectional.center,
      textDirection: TextDirection.ltr,
      children: <Widget>[
        FadeTransition(
          opacity: _targetOpacityAnimation,
          child: widget.target,
        ),
        FadeTransition(
          opacity: _placeholderOpacityAnimation,
          child: widget.placeholder,
        ),
      ],
    );
  }
}
```
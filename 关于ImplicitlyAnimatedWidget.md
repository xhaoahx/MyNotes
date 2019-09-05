# 关于ImplicitlyAnimatedWidget

[TOC]



## 函数原型：

```dart

/// [TweenVisitor]的参数之一，被[AnimatedWidgetBaseState.forEachTween]使用
/// 返回一个把 targetValue 作为终点值的补间
typedef TweenConstructor<T> = Tween<T> Function(T targetValue);

/// tween 参数必须包含当前的 tween值，在首次建造state的时候为null。
typedef TweenVisitor<T> = Tween<T> Function(Tween<T> tween, T targetValue, TweenConstructor<T> constructor);
```



## 源码

```dart
abstract class ImplicitlyAnimatedWidget extends StatefulWidget {
 
  const ImplicitlyAnimatedWidget({
    Key key,
    this.curve = Curves.linear,
    @required this.duration,
    this.reverseDuration,
  }) : assert(curve != null),
       assert(duration != null),
       super(key: key);
    
  final Curve curve;
  final Duration duration;
  final Duration reverseDuration;

  @override
  ImplicitlyAnimatedWidgetState<ImplicitlyAnimatedWidget> createState();

  ...
}	



abstract class ImplicitlyAnimatedWidgetState<T extends ImplicitlyAnimatedWidget> extends State<T> with SingleTickerProviderStateMixin<T> {

  @protected
  AnimationController get controller => _controller;
  AnimationController _controller;

  /// The animation driving this widget's implicit animations.
  Animation<double> get animation => _animation;
  Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: widget.duration,
      reverseDuration: widget.reverseDuration,
      debugLabel: kDebugMode ? '${widget.toStringShort()}' : null,
      vsync: this,
    );
    /// 首次更新曲线，把控制器作为动画
    _updateCurve();
    /// 首次构建tween会调用到子类的forEachTween方法
    _constructTweens();
    /// 调用子类的didUpdateTweens()方法
    didUpdateTweens();
  }

  @override
  void didUpdateWidget(T oldWidget) {
    super.didUpdateWidget(oldWidget);
    /// 曲线不同则更新曲线
    if (widget.curve != oldWidget.curve)
      _updateCurve();
    _controller.duration = widget.duration;
    _controller.reverseDuration = widget.reverseDuration;
    /// _constructTweens()方法返回真，则需要开始动画
    if (_constructTweens()) {
      /// 这里在_updateTween里调用了constructor，更改了tween并返回
      forEachTween((Tween<dynamic> tween, dynamic targetValue, TweenConstructor<dynamic> constructor) {
        /// 把 tween更新，将新值作为end
        _updateTween(tween, targetValue);
        return tween;
      });
      /// 把控制器置0，开始动画  
      _controller
        ..value = 0.0
        ..forward();
      /// 调用子类的方法  
      didUpdateTweens();
    }
  }

  void _updateCurve() {
    if (widget.curve != null)
      _animation = CurvedAnimation(parent: _controller, curve: widget.curve);
    else
      _animation = _controller;
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  bool _shouldAnimateTween(Tween<dynamic> tween, dynamic targetValue) {
    return targetValue != (tween.end ?? tween.begin);
  }

  void _updateTween(Tween<dynamic> tween, dynamic targetValue) {
    if (tween == null)
      return;
    tween
      ..begin = tween.evaluate(_animation)
      ..end = targetValue;
    /// 其中 T evaluate(Animation<double> animation) => transform(animation.value); 
    /// 即transform动画的当前值  
    /// 把targetValue作为end
  }

  bool _constructTweens() {
    bool shouldStartAnimation = false;
    forEachTween((Tween<dynamic> tween, dynamic targetValue, TweenConstructor<dynamic> constructor) {
      if (targetValue != null) {
        tween ??= constructor(targetValue);
        /// 如果tween的值不为开始或结束值（为null则是开始值）的话，开始动画
        if (_shouldAnimateTween(tween, targetValue))
          shouldStartAnimation = true;
      } else {
        tween = null;
      }
      return tween;
    });
    return shouldStartAnimation;
  }

  /
  @protected
  void forEachTween(TweenVisitor<dynamic> visitor);

  @protected
  void didUpdateTweens() { }
}
```



## 以Animated Align为例

```dart
/// 当每次alignment改变时，会触发动画
class AnimatedAlign extends ImplicitlyAnimatedWidget {
  const AnimatedAlign({
    Key key,
    @required this.alignment,
    this.child,
    Curve curve = Curves.linear,
    @required Duration duration,
    Duration reverseDuration,
  }) : assert(alignment != null),
       super(key: key, curve: curve, duration: duration, reverseDuration: reverseDuration);

  final AlignmentGeometry alignment;
  final Widget child;

  @override
  _AnimatedAlignState createState() => _AnimatedAlignState();
    
  ...
}

///  AlignmentGeometryTween
class AlignmentGeometryTween extends Tween<AlignmentGeometry> {
  AlignmentGeometryTween({
    AlignmentGeometry begin,
    AlignmentGeometry end,
  }) : super(begin: begin, end: end);
  @override
  AlignmentGeometry lerp(double t) => AlignmentGeometry.lerp(begin, end, t);
    
 ///  assert(t != null);
 ///   if (a == null && b == null)
 ///     return null;
 ///   if (a == null)
 ///     return b * t;
 ///   if (b == null)
 ///     return a * (1.0 - t);
 ///   if (a is Alignment && b is Alignment)
 ///     return Alignment.lerp(a, b, t);
 ///   if (a is AlignmentDirectional && b is AlignmentDirectional)
 ///     return AlignmentDirectional.lerp(a, b, t);
 ///   return _MixedAlignment(
 ///     ui.lerpDouble(a._x, b._x, t),
 ///     ui.lerpDouble(a._start, b._start, t),
 ///     ui.lerpDouble(a._y, b._y, t),
 ///   );  
}


class _AnimatedAlignState extends AnimatedWidgetBaseState<AnimatedAlign> {
  AlignmentGeometryTween _alignment;

  /// 调用 visitor 返回一个将widget.aligment作begin的AlignmentGeometryTween
  @override
  void forEachTween(TweenVisitor<dynamic> visitor) {
    _alignment = visitor(_alignment, widget.alignment, (dynamic value) => AlignmentGeometryTween(begin: value));
    /// 第三个参数实则是Tween<T> Function(T targetValue);  
  }

  @override
  Widget build(BuildContext context) {
    return Align(
      alignment: _alignment.evaluate(animation),
      child: widget.child,
    );
  }
  
  ...  
  }
}

```




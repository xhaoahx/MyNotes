# Flutter ImplicitlyAnimatedWidget 源码分析

[TOC]



## 函数原型：

```dart
/// [TweenVisitor] 的参数之一，被 [AnimatedWidgetBaseState.forEachTween] 使用
/// 返回一个把 targetValue 作为终点值的补间
typedef TweenConstructor<T> = Tween<T> Function(T targetValue);

/// tween 参数必须包含当前的 tween值，在首次建造 state 的时候为 null。
typedef TweenVisitor<T> = Tween<T> Function(
    Tween<T> tween, 
    T targetValue, 
    TweenConstructor<T> constructor
);
```



## ImplicitylyAnimatedWidget

```dart
abstract class ImplicitlyAnimatedWidget extends StatefulWidget {
 
  const ImplicitlyAnimatedWidget({
    Key key,
    this.curve = Curves.linear,
    @required this.duration,
    this.reverseDuration,
  }) : super(key: key);
    
  final Curve curve;
  final Duration duration;
  final Duration reverseDuration;

  @override
  ImplicitlyAnimatedWidgetState<ImplicitlyAnimatedWidget> createState();
}	
```



## ImplicitlyAnimatedWidgetState

```dart
abstract class ImplicitlyAnimatedWidgetState<
		T extends ImplicitlyAnimatedWidget> 
	extends State<T> 
	with SingleTickerProviderStateMixin<T> {

  @protected
  AnimationController get controller => _controller;
  AnimationController _controller;

  /// 间接动画
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
    /// 首次构建 tween 会调用到子类的 forEachTween 方法
    _constructTweens();
    /// 更新 tween 完成回调
    didUpdateTweens();
  }

  @override
  void didUpdateWidget(T oldWidget) {
    super.didUpdateWidget(oldWidget);
    /// 更新曲线
    if (widget.curve != oldWidget.curve)
      _updateCurve();
    /// 更新周期 和 逆转周期
    _controller.duration = widget.duration;
    _controller.reverseDuration = widget.reverseDuration;
    /// _constructTweens() 方法返回真，则需要开始动画
    if (_constructTweens()) {
      TweenVisitor visitor = (
          Tween<dynamic> tween, 
          dynamic targetValue, 
          TweenConstructor<dynamic> constructor
      ) {
        /// 把 tween 更新，将新值作为 end
        _updateTween(tween, targetValue);
        return tween;
      };
        
      forEachTween(visitor);
      /// 由于已经更新了 tween，这时候 controller 的 0.0 值映射到 旧 tween 的当前值。
      /// 把 controller 置 0.0 并且开始动画  
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

  /// 如果目标值不是补间的结尾或者起点的话，则需要开始动画
  bool _shouldAnimateTween(Tween<dynamic> tween, dynamic targetValue) {
    return targetValue != (tween.end ?? tween.begin);
  }

  /// 更新给定的 tween
  void _updateTween(Tween<dynamic> tween, dynamic targetValue) {
    if (tween == null)
      return;
    /// 将 _animation 的当前值作为 tween.begin
    /// 将 targetValue 作为 tween.end
    tween
      /// tween.evaluate 方法返回 在给定 animation 的补间值
      /// T evaluate(Animation<double> animation) => transform(animation.value); 
      ..begin = tween.evaluate(_animation)
      ..end = targetValue;
  }

  bool _constructTweens() {
    bool shouldStartAnimation = false;
    TweenVisitor visitor = (
        Tween<dynamic> tween, 
        dynamic targetValue, 
        TweenConstructor<dynamic> constructor
    ) {
      if (targetValue != null) {
        /// 起始状态时，tween 为 null，这时候会构造调用 constructor 构造一个初始 tween
        tween ??= constructor(targetValue);
        /// 如果 tween 的值不为开始或结束值（为 null 则是开始值）的话，则应该开始动画，设置标记为 true
        if (_shouldAnimateTween(tween, targetValue))
          shouldStartAnimation = true;
      } else {
        tween = null;
      }
      return tween;
    };
      
    forEachTween(visitor);
    return shouldStartAnimation;
  }

  /// 要求子类实现，给定 visitor，子类调用 visitor 来更新其保存 tween
  @protected
  void forEachTween(TweenVisitor<dynamic> visitor);

  /// 更新 tween 完成后的回调，不一定要实现
  @protected
  void didUpdateTweens() { }
}
```



## 以 Animated Align 为例

### AnimatedAlign

```dart
/// 当每次 alignment 改变时，会触发动画（从旧 alignment 过渡到新 alignment）
class AnimatedAlign extends ImplicitlyAnimatedWidget {
  const AnimatedAlign({
    Key key,
    @required this.alignment,
    this.child,
    Curve curve = Curves.linear,
    @required Duration duration,
    Duration reverseDuration,
  }) : super(
      key: key, 
      curve: curve, 
      duration: duration,
      reverseDuration: reverseDuration
  );

  final AlignmentGeometry alignment;
  final Widget child;

  @override
  _AnimatedAlignState createState() => _AnimatedAlignState();
    
  ...
}
```

### AlignmentGeometryTween
```dart
class AlignmentGeometryTween extends Tween<AlignmentGeometry> {
  AlignmentGeometryTween({
    AlignmentGeometry begin,
    AlignmentGeometry end,
  }) : super(begin: begin, end: end);
  @override
  AlignmentGeometry lerp(double t) => AlignmentGeometry.lerp(begin, end, t);
    
 /// dynamic lerp(dynamic a,dynamic b,dynamic t){
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
 /// }
}
```

### _AnimatedAlignState
```dart

class _AnimatedAlignState extends AnimatedWidgetBaseState<AnimatedAlign> {
  AlignmentGeometryTween _alignment;

  /// forEachTween 方法的实现实际是要求利用给定的 visitor 来更新所有的 tween
  @override
  void forEachTween(TweenVisitor<dynamic> visitor) {
    /// 将现有的 tween 更新到从 当前补间值开始到目标值结束 的新 tween
    _alignment = visitor(
        /// 现有的 tween
        _alignment, 
        /// 目标值
        widget.alignment, 
        /// 如果 现有 tween 为 null，则会 调用此构造器新建一个 tween
        (dynamic value) => AlignmentGeometryTween(begin: value)
    );
  }

  /// 子类只需调用 tween.evaluate（animation）即可实现动画（animation 字段源自超类）
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




# Flutter Animation 流程



[TOC]

## 从SingleTickerProviderStateMixin开始

```dart
/// 在需要动画的时侯，State可以混入这个mixin，提供一个TickeProvider
/// 这个mixin实现了TickerProvider，并重写了didChangeDependencies

mixin SingleTickerProviderStateMixin<T extends StatefulWidget> 
    on State<T> implements TickerProvider 
{
    Ticker _ticker;

    @override
    Ticker createTicker(TickerCallback onTick) {
        _ticker = Ticker(onTick, debugLabel: kDebugMode ? 'created by $this' : null);
        return _ticker;
    }

    @override
    void dispose() {
        super.dispose();
    }

    @override
    void didChangeDependencies() {
        if (_ticker != null)
            _ticker.muted = !TickerMode.of(context);
        super.didChangeDependencies();
    }
    ...
}
```



## Ticker

```dart
/// 在每一帧到来时，Ticker会回调它的_onTick

class Ticker {
  Ticker(this._onTick, { this.debugLabel }) {...}

  TickerFuture _future;

  /// 是否处于沉默状态
  /// 处于沉默状态时，ticker依旧运行，但不会回调_onTick  
  bool get muted => _muted;
  bool _muted = false;
 
  set muted(bool value) {
    if (value == muted)
      return;
    _muted = value;
    if (value) {
      unscheduleTick();
    } else if (shouldScheduleTick) {
      scheduleTick();
    }
  }

  bool get isTicking {
    if (_future == null)
      return false;
    if (muted)
      return false;
    if (SchedulerBinding.instance.framesEnabled)
      return true;
    if (SchedulerBinding.instance.schedulerPhase != SchedulerPhase.idle)
      return true; // for example, we might be in a warm-up frame or forced frame
    return false;
  }

  bool get isActive => _future != null;

  Duration _startTime;

  /// 开始每一帧调用_onTick（通过将_onTick加入 _transientCallbacks）
  TickerFuture start() {
    _future = TickerFuture._();
    if (shouldScheduleTick) {
      scheduleTick();
    }
    if (
        SchedulerBinding.instance.schedulerPhase.index > 
        SchedulerPhase.idle.index &&     
        SchedulerBinding.instance.schedulerPhase.index < 		
        SchedulerPhase.postFrameCallbacks.index
    )
      _startTime = SchedulerBinding.instance.currentFrameTimeStamp;
    return _future;
  }

  /// 注销 _onTick
  /// 这个方法会把 isActive 设为 false  
  void stop({ bool canceled = false }) {
    if (!isActive)
      return;

    final TickerFuture localFuture = _future;
    _future = null;
    _startTime = null;

    unscheduleTick();
    if (canceled) {
      localFuture._cancel(this);
    } else {
      localFuture._complete();
    }
  }

  /// 每一帧的回调	
  final TickerCallback _onTick;

  int _animationId;

  /// 当请求完成后，ScheduleBinding 会分配一个 animationId
  @protected
  bool get scheduled => _animationId != null;

  /// 是否应该调度Tick
  /// 当 muted = false 并且 isActive = true 并且 scheduled = false 时为 true
  /// 否则为 false  
  @protected
  bool get shouldScheduleTick => !muted && isActive && !scheduled;

  /// 被注册的每帧回调。
  /// 在这个方法中，会调用scheduleTick再次将自身注册到回调当中，
  /// 所以，在不被取消的情况下，这个方法会被连续调用  
  void _tick(Duration timeStamp) {
    _animationId = null;
    _startTime ??= timeStamp;
    _onTick(timeStamp - _startTime);

    if (shouldScheduleTick)
      scheduleTick(rescheduling: true);
  }

  /// 将_tick注册为每帧瞬时回调（即将_tick加入到 transientCallbacks 中）
  /// !注意!
  /// scheduleFrameCallback会返回一个整数作为编号，并调用Window.scheduleFrame来请求一帧 
  /// 之后engine会回调handleBeginFrame，并调用transientCallbacks中的所有回调
  
  @protected
  void scheduleTick({ bool rescheduling = false }) 
      _animationId = SchedulerBinding.instance.
      scheduleFrameCallback(_tick, rescheduling: rescheduling);
  }

  /// 取消已经注册回调
  @protected
  void unscheduleTick() {
      if (scheduled) {
          SchedulerBinding.instance.cancelFrameCallbackWithId(_animationId);
          _animationId = null;
      }
      assert(!shouldScheduleTick);
  }

  /// 吸收另外一个Ticker
  void absorbTicker(Ticker originalTicker) {
    
    if (originalTicker._future != null) {
      _future = originalTicker._future;
      _startTime = originalTicker._startTime;
      if (shouldScheduleTick)
        scheduleTick();
      originalTicker._future = null;
      originalTicker.unscheduleTick();
    }
    originalTicker.dispose();
  }

  @mustCallSuper
  void dispose() {
    if (_future != null) {
      final TickerFuture localFuture = _future;
      _future = null;
      unscheduleTick();
      localFuture._cancel(this);
    }
  }

```



## scheduleFrameCallback

```dart
/// 这里把回调加入到了 SchedulerBinding 的 _traSchedulerBindinglbacks 中
/// 当 drawBeginFrame 被调用时，_traSchedulerBindinglbacks 里的回调会依次被调用
int scheduleFrameCallback(FrameCallback callback, { bool rescheduling = false }) {
    scheduleFrame();
    _nextFrameCallbackId += 1;
    _transientCallbacks[_nextFrameCallbackId] = 
        _FrameCallbackEntry(callback, rescheduling: rescheduling);
    return _nextFrameCallbackId;
}
```



## AnimationController

```dart
/// 动画控制器，用于提供最原始的tick
class AnimationController extends Animation<double>
  with AnimationEagerListenerMixin, AnimationLocalListenersMixin, AnimationLocalStatusListenersMixin {
  
  AnimationController({
    double value,
    this.duration,
    this.reverseDuration,
    this.debugLabel,
    this.lowerBound = 0.0,
    this.upperBound = 1.0,
    this.animationBehavior = AnimationBehavior.normal,
    @required TickerProvider vsync,
  }) : 
    _direction = _AnimationDirection.forward {
    /// 通常写法是 AnimationController(vsync:this)
    /// 其中this是混入了 SingleTickerProviderStateMixin 的 State    
    _ticker = vsync.createTicker(_tick);
    _internalSetValue(value ?? lowerBound);
  }

  /// 创建一个没有上下界的 AnimationController
  AnimationController.unbounded({
    double value = 0.0,
    this.duration,
    this.reverseDuration,
    this.debugLabel,
    @required TickerProvider vsync,
    this.animationBehavior = AnimationBehavior.preserve,
  }) : assert(value != null),
       assert(vsync != null),
       lowerBound = double.negativeInfinity,
       upperBound = double.infinity,
       _direction = _AnimationDirection.forward {
    _ticker = vsync.createTicker(_tick);
    _internalSetValue(value);
  }

  /// 下界（dismissed）
  final double lowerBound;

  /// 上界（compeleted）
  final double upperBound;

  final String debugLabel;

  final AnimationBehavior animationBehavior;

  /// 供外界获取自身（便于不使用引用传递）
  Animation<double> get view => this;

  /// 正向动画时间  
  Duration duration;

  /// 逆向动画时间  
  Duration reverseDuration;

  Ticker _ticker;
    
  /// 在每一帧的时候回调这个方法  
  void _tick(Duration elapsed) {
    _lastElapsedDuration = elapsed;
    final double elapsedInSeconds =
        elapsed.inMicroseconds.toDouble() / Duration.microsecondsPerSecond;
    _value = _simulation.x(elapsedInSeconds).clamp(lowerBound, upperBound);
      
    if (_simulation.isDone(elapsedInSeconds)) {
      _status = (_direction == _AnimationDirection.forward) ?
        AnimationStatus.completed :
        AnimationStatus.dismissed;
      stop(canceled: false);
    }
    /// 通知监听者。
    /// e.g.
    /// AnimationController controller = AnimationController(vsync:this)
    /// ..addListener((){
    /// 	setState((){}) 
  	/// })  
    /// 这样每一帧都会回调setState，触发重建
    notifyListeners();
    _checkStatusChanged();
  }  
    
  /// 重新绑定Ticker
  void resync(TickerProvider vsync) {
    final Ticker oldTicker = _ticker;
    _ticker = vsync.createTicker(_tick);
    _ticker.absorbTicker(oldTicker);
  }

  /// 模拟。用于控制动画的开始结束、并获取速度，距离  
  Simulation _simulation;

  /// 动画值
  @override
  double get value => _value;
  double _value;
  
  /// 暂停动画并为其设置一个新值。 
  set value(double newValue) {
    stop();
    _internalSetValue(newValue);
    notifyListeners();
    _checkStatusChanged();
  }

  /// 重置
  void reset() {
    value = lowerBound;
  }

  /// 动画速度
  double get velocity {
    if (!isAnimating)
      return 0.0;
    /// 通过_simulation来获取速度  
    return _simulation.dx(lastElapsedDuration.inMicroseconds.toDouble() 
                          / Duration.microsecondsPerSecond);
  }

  /// 设置新值  
  void _internalSetValue(double newValue) {
    _value = newValue.clamp(lowerBound, upperBound);
    if (_value == lowerBound) {
      _status = AnimationStatus.dismissed;
    } else if (_value == upperBound) {
      _status = AnimationStatus.completed;
    } else {
      _status = (_direction == _AnimationDirection.forward) ?
        AnimationStatus.forward :
        AnimationStatus.reverse;
    }
  }

  /// 从动画开始到最近一次调用_tick的时间
  /// 动画停止时，这个值为null  
  Duration get lastElapsedDuration => _lastElapsedDuration;
  Duration _lastElapsedDuration;

  bool get isAnimating => _ticker != null && _ticker.isActive;

  _AnimationDirection _direction;

  @override
  AnimationStatus get status => _status;
  AnimationStatus _status;

  /// 以下的forward、reverse、animateTo、animateBack 都调用了_animateToInternal
  TickerFuture forward({ double from }) {
    _direction = _AnimationDirection.forward;
    if (from != null)
      value = from;
    return _animateToInternal(upperBound);
  }

  TickerFuture reverse({ double from }) {
    _direction = _AnimationDirection.reverse;
    if (from != null)
      value = from;
    return _animateToInternal(lowerBound);
  }

  TickerFuture animateTo(double target, { Duration duration, Curve curve = Curves.linear }) {
    _direction = _AnimationDirection.forward;
    return _animateToInternal(target, duration: duration, curve: curve);
  }
 
  TickerFuture animateBack(double target, { Duration duration, Curve curve = Curves.linear }) {
    _direction = _AnimationDirection.reverse;
    return _animateToInternal(target, duration: duration, curve: curve);
  }

   
  /// 向指定的值开始动画   
  TickerFuture _animateToInternal(double target, { 
      Duration duration, 
      Curve curve = Curves.linear 
  }) {
    double scale = 1.0;
    if (SemanticsBinding.instance.disableAnimations) {
      switch (animationBehavior) {
        case AnimationBehavior.normal:
          // 框架无法处理时间为0的动画，所以设置为持续时间的0.05来限制只运行一帧
          scale = 0.05;
          break;
        case AnimationBehavior.preserve:
          break;
      }
    }
    Duration simulationDuration = duration;
    if (simulationDuration == null) {
      final double range = upperBound - lowerBound;
      /// 剩余范围（按百分比表示）  
      final double remainingFraction = 
          range.isFinite ? (target - _value).abs() / range : 1.0;
      final Duration directionDuration =
        (_direction == _AnimationDirection.reverse && reverseDuration != null)
        ? reverseDuration
        : this.duration;
      /// 剩余时间  
      simulationDuration = directionDuration * remainingFraction;
    } else if (target == value) {
      /// 已经处于目标值，则无需动画
      simulationDuration = Duration.zero;
    }
    /// 暂停动画  
    stop();
    /// 时长为0，则完成动画  
    if (simulationDuration == Duration.zero) {
      if (value != target) {
        _value = target.clamp(lowerBound, upperBound);
        notifyListeners();
      }
      _status = (_direction == _AnimationDirection.forward) ?
        AnimationStatus.completed :
        AnimationStatus.dismissed;
      _checkStatusChanged();
      return TickerFuture.complete();
    }
    /// 开始动画  
    return _startSimulation(
        _InterpolationSimulation(_value, target, simulationDuration, curve, scale));
  }

  /// 在一个区间内（可以指定最大和最小值）反复运行
  TickerFuture repeat({ double min, double max, bool reverse = false, Duration period }) {
    min ??= lowerBound;
    max ??= upperBound;
    period ??= duration;
    stop();
    return _startSimulation(_RepeatingSimulation(_value, min, max, reverse, period, _directionSetter));
  }

  void _directionSetter(_AnimationDirection direction) {
    _direction = direction;
    _status = (_direction == _AnimationDirection.forward) ?
      AnimationStatus.forward :
      AnimationStatus.reverse;
    _checkStatusChanged();
  }

  /// 以给定的速度完成动画
  TickerFuture fling({ double velocity = 1.0, AnimationBehavior animationBehavior }) {
    _direction = velocity < 0.0 ? _AnimationDirection.reverse : _AnimationDirection.forward;
    final double target = velocity < 0.0 ? lowerBound - _kFlingTolerance.distance
                                         : upperBound + _kFlingTolerance.distance;
    double scale = 1.0;
    final AnimationBehavior behavior = animationBehavior ?? this.animationBehavior;
    if (SemanticsBinding.instance.disableAnimations) {
      switch (behavior) {
        case AnimationBehavior.normal:
          scale = 200.0;
          break;
        case AnimationBehavior.preserve:
          break;
      }
    }
    final Simulation simulation = SpringSimulation(
        _kFlingSpringDescription, value, target, velocity * scale)
      ..tolerance = _kFlingTolerance;
    stop();
    return _startSimulation(simulation);
  }

  /// 用给定的Simulation来完成动画
  TickerFuture animateWith(Simulation simulation) {
    assert(
      _ticker != null,
      'AnimationController.animateWith() called after AnimationController.dispose()\n'
      'AnimationController methods should not be used after calling dispose.'
    );
    stop();
    _direction = _AnimationDirection.forward;
    return _startSimulation(simulation);
  }

  /// 开始动画
  /// 由于Tickcer._tick方法的特性，会连续每帧请求回调，而不用每次调用这个方法  
  TickerFuture _startSimulation(Simulation simulation) {
    _simulation = simulation;
    _lastElapsedDuration = Duration.zero;
    _value = simulation.x(0.0).clamp(lowerBound, upperBound);
    /// 注册每帧回调  
    /// 这里让 _ticker 向ScheduleBinding 请求一帧，触发动画
    /// 由于 
    final TickerFuture result = _ticker.start();
    _status = (_direction == _AnimationDirection.forward) ?
      AnimationStatus.forward :
      AnimationStatus.reverse;
    _checkStatusChanged();
    return result;
  }

  /// 停止动画，这会调用到 _ticker.stop，从而注销回调，停止自动调用_tick
  void stop({ bool canceled = true }) {
    _simulation = null;
    _lastElapsedDuration = null;
    _ticker.stop(canceled: canceled);
  }

  @override
  void dispose() {
    _ticker.dispose();
    _ticker = null;
    super.dispose();
  }

  AnimationStatus _lastReportedStatus = AnimationStatus.dismissed;
    
  void _checkStatusChanged() {
    final AnimationStatus newStatus = status;
    if (_lastReportedStatus != newStatus) {
      _lastReportedStatus = newStatus;
      notifyStatusListeners(newStatus);
    }
  }
}
```



## Simulation

```dart
abstract class Simulation {
  Simulation({ this.tolerance = Tolerance.defaultTolerance });
  /// 在给定时间的位置
  double x(double time);
  /// 值给定时间的速度
  double dx(double time);
  /// 是否在给定的时间已经完成	
  bool isDone(double time);
  /// 允许误差。
  /// 在某些情况下，simulation的函数（例如渐近线）可能无法达到结束的条件。
  /// 通过设置一定的容忍误差，在误差范围内simulation被考虑为 完成  
  Tolerance tolerance;
}
```



##  SpringSimulation

```dart
/// 通过给定的 弹簧 描述、开始距离、结束距离 和初始速度 来创建一个弹簧模拟。
class SpringSimulation extends Simulation {
  SpringSimulation(
    SpringDescription spring,
    double start,
    double end,
    double velocity, {
    Tolerance tolerance = Tolerance.defaultTolerance,
  }) : _endPosition = end,
       _solution = _SpringSolution(spring, start - end, velocity),
       super(tolerance: tolerance);

  final double _endPosition;
  final _SpringSolution _solution;
  SpringType get type => _solution.type;

  @override
  double x(double time) => _endPosition + _solution.x(time);

  @override
  double dx(double time) => _solution.dx(time);

  @override
  bool isDone(double time) {
    return nearZero(_solution.x(time), tolerance.distance) &&
           nearZero(_solution.dx(time), tolerance.velocity);
  }
}
```



## _InterpolationSimulation

```dart
class _InterpolationSimulation extends Simulation {
    _InterpolationSimulation(
        this._begin, this._end,
        Duration duration, 
        this._curve, 
        double scale
) : _durationInSeconds = (duration.inMicroseconds * scale) /                
    Duration.microsecondsPerSecond;

    /// Simulation 总时间
    final double _durationInSeconds;
    final double _begin;
    final double _end;
    final Curve _curve;

    /// 对给定的时间返回一个插值
    @override
    double x(double timeInSeconds) {
        final double t = (timeInSeconds / _durationInSeconds).clamp(0.0, 1.0);
        if (t == 0.0)
            return _begin;
        else if (t == 1.0)
            return _end;
        else
            return _begin + (_end - _begin) * _curve.transform(t);
    }

    /// 返回在给定的时间的速度
    @override
    double dx(double timeInSeconds) {
        final double epsilon = tolerance.time;
        return (x(timeInSeconds + epsilon) - x(timeInSeconds - epsilon)) / (2 * epsilon);
    }

    /// 是否已经完成
    /// 当给定的时间大于Simulation总时长时，视作完成
    @override
    bool isDone(double timeInSeconds) => timeInSeconds > _durationInSeconds;
}
```


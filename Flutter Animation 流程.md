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
    if (SchedulerBinding.instance.schedulerPhase.index > SchedulerPhase.idle.index &&
        SchedulerBinding.instance.schedulerPhase.index < SchedulerPhase.postFrameCallbacks.index)
      _startTime = SchedulerBinding.instance.currentFrameTimeStamp;
    return _future;
  }

  /// 停止调用 _onTick
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

  void _tick(Duration timeStamp) {
    assert(isTicking);
    assert(scheduled);
    _animationId = null;

    _startTime ??= timeStamp;
    _onTick(timeStamp - _startTime);

    // The onTick callback may have scheduled another tick already, for
    // example by calling stop then start again.
    if (shouldScheduleTick)
      scheduleTick(rescheduling: true);
  }

  /// 注册回调
    @protected
    void scheduleTick({ bool rescheduling = false }) {
        assert(!scheduled);
        assert(shouldScheduleTick);
        _animationId = SchedulerBinding.instance.scheduleFrameCallback(
            _tick, rescheduling: rescheduling);
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
      originalTicker._future = null; // so that it doesn't get disposed when we dispose of originalTicker
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

  /// 供外界获取自身（方便不使用引用传递）
  Animation<double> get view => this;

  Duration duration;

  Duration reverseDuration;

  Ticker _ticker;
    
  /// 在每一帧的时候回调这个方法  
  void _tick(Duration elapsed) {
    _lastElapsedDuration = elapsed;
    final double elapsedInSeconds = elapsed.inMicroseconds.toDouble() / Duration.microsecondsPerSecond;
    assert(elapsedInSeconds >= 0.0);
    _value = _simulation.x(elapsedInSeconds).clamp(lowerBound, upperBound);
    if (_simulation.isDone(elapsedInSeconds)) {
      _status = (_direction == _AnimationDirection.forward) ?
        AnimationStatus.completed :
        AnimationStatus.dismissed;
      stop(canceled: false);
    }
    notifyListeners();
    _checkStatusChanged();
  }  
    

  /// Recreates the [Ticker] with the new [TickerProvider].
  void resync(TickerProvider vsync) {
    final Ticker oldTicker = _ticker;
    _ticker = vsync.createTicker(_tick);
    _ticker.absorbTicker(oldTicker);
  }

  Simulation _simulation;

  /// 动画值
  @override
  double get value => _value;
  double _value;
  
  /// 暂停动画并为其设置一个新值。 
  set value(double newValue) {
    assert(newValue != null);
    stop();
    _internalSetValue(newValue);
    notifyListeners();
    _checkStatusChanged();
  }

  
  void reset() {
    value = lowerBound;
  }

  /// 动画速度
  double get velocity {
    if (!isAnimating)
      return 0.0;
    return _simulation.dx(lastElapsedDuration.inMicroseconds.toDouble() 
                          / Duration.microsecondsPerSecond);
  }

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
      final double remainingFraction = range.isFinite ? (target - _value).abs() / range : 1.0;
      final Duration directionDuration =
        (_direction == _AnimationDirection.reverse && reverseDuration != null)
        ? reverseDuration
        : this.duration;
      simulationDuration = directionDuration * remainingFraction;
    } else if (target == value) {
      // Already at target, don't animate.
      simulationDuration = Duration.zero;
    }
    stop();
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

  TickerFuture _startSimulation(Simulation simulation) {
    assert(simulation != null);
    assert(!isAnimating);
    _simulation = simulation;
    _lastElapsedDuration = Duration.zero;
    _value = simulation.x(0.0).clamp(lowerBound, upperBound);
    /// 注册每帧回调  
    final TickerFuture result = _ticker.start();
    _status = (_direction == _AnimationDirection.forward) ?
      AnimationStatus.forward :
      AnimationStatus.reverse;
    _checkStatusChanged();
    return result;
  }

  /// 停止动画  
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


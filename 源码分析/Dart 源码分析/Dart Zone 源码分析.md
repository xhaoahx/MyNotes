# Dart Zone 源码分析



## 函数原型

```dart
/// 返回值为 R 的空回调
typedef R ZoneCallback<R>();
/// 接受 T 为参数，返回 R 的一元回调
typedef R ZoneUnaryCallback<R, T>(T arg);
/// 接受 T1，T2 为参数，返回 R 的二元回调
typedef R ZoneBinaryCallback<R, T1, T2>(T1 arg1, T2 arg2);

/// 未处理的错误回调
typedef HandleUncaughtErrorHandler = 
    void Function(
    	Zone self,
    	ZoneDelegate parent, 
    	Zone zone, 
    	Object error, 
    	StackTrace stackTrace
	);

/// 
typedef RunHandler = 
    R Function<R>(
    	Zone self, 
    	ZoneDelegate parent, 
    	Zone zone, 
    	R Function() f
	);

typedef RunUnaryHandler = 
    R Function<R, T>(
    	Zone self,
    	ZoneDelegate parent, 
    	Zone zone, 
    	R Function(T arg) f,
    	T arg
	);

typedef RunBinaryHandler = 
    R Function<R, T1, T2>(
   		Zone self, 
    	ZoneDelegate parent,
    	Zone zone, 
    	R Function(T1 arg1, T2 arg2) f,
    	T1 arg1, 
    	T2 arg2
	);


typedef RegisterCallbackHandler = 
    ZoneCallback<R> Function<R>(
        Zone self, 
    	ZoneDelegate parent,
    	Zone zone, 
    	R Function() f
	);

typedef RegisterUnaryCallbackHandler = 
    ZoneUnaryCallback<R, T> Function<R, T>(
    	Zone self, 
    	ZoneDelegate parent, 
    	Zone zone, R Function(T arg) f
	);

typedef RegisterBinaryCallbackHandler = 
    ZoneBinaryCallback<R, T1, T2> Function<R, T1, T2>(
    	Zone self,
    	ZoneDelegate parent, 
    	Zone zone, 
    	R Function(T1 arg1, T2 arg2) f
	);

typedef AsyncError ErrorCallbackHandler(
    	Zone self, 
    	ZoneDelegate parent,
    	Zone zone, 
    	Object error, 
    	StackTrace stackTrace
);

typedef void ScheduleMicrotaskHandler(
    Zone self, 
    ZoneDelegate parent, 
    Zone zone, void f()
);

typedef Timer CreateTimerHandler(
    Zone self, 
    ZoneDelegate parent, 
    Zone zone, 
    Duration duration, 
    void f()
);

typedef Timer CreatePeriodicTimerHandler(
    Zone self, 
    ZoneDelegate parent,
    Zone zone,
    Duration period, 
    void f(Timer timer)
);

typedef void PrintHandler(
    Zone self, 
    ZoneDelegate parent, 
    Zone zone, 
    String line
);

typedef Zone ForkHandler(
    Zone self, 
    ZoneDelegate parent, 
    Zone zone,
    ZoneSpecification specification, 
    Map zoneValues
);
```



## AsyncError

```dart
/// 异步错误
class AsyncError implements Error {
  final Object error;
  final StackTrace stackTrace;

  AsyncError(this.error, this.stackTrace);

  String toString() => '$error';
}
```



## _ZoneFunction


```dart
/// 记录了 function 的其对应的 zone
class _ZoneFunction<T extends Function> {
  final _Zone zone;
  final T function;
  const _ZoneFunction(this.zone, this.function);
}
```



## ZoneSpecification

```dart
/**
 * 此类用于向复制的 Zone 提供规范的配置信息.
 *
 * 当复制一个 Zone 的时候，我们可以通过提供回调来重载其默认的行为。这些回调通常在此类中给出
 *
 * handlers 与 [Zone] 上的同名方法具有相同的签名，但是接收三个额外的参数:
 *
 *   1. 回调将要被安装的 zone (the "self" zone).
 *   2. 源自父级 zone 的 [ZoneDelegate]
 *   3. 第一个收到请求的 zone（在事件冒泡之前）
 *
 * handlers 可以停止请求的传播(只需不调用父级 handlers)，也可以转发到父级 zone，在传递过程中可能会修改参数。
 */
abstract class ZoneSpecification {

  const factory ZoneSpecification({
      HandleUncaughtErrorHandler handleUncaughtError,
      RunHandler run,
      RunUnaryHandler runUnary,
      RunBinaryHandler runBinary,
      RegisterCallbackHandler registerCallback,
      RegisterUnaryCallbackHandler registerUnaryCallback,
      RegisterBinaryCallbackHandler registerBinaryCallback,
      ErrorCallbackHandler errorCallback,
      ScheduleMicrotaskHandler scheduleMicrotask,
      CreateTimerHandler createTimer,
      CreatePeriodicTimerHandler createPeriodicTimer,
      PrintHandler print,
      ForkHandler fork
  /// 工厂构造函数的写法，即使用 ZoneSpecification() 会构建一个 _ZoneSpecification 类
  }) = _ZoneSpecification;

  /// 拷贝构造
  factory ZoneSpecification.from(
      ZoneSpecification other,{
      HandleUncaughtErrorHandler handleUncaughtError,
      RunHandler run,
      RunUnaryHandler runUnary,
      RunBinaryHandler runBinary,
      RegisterCallbackHandler registerCallback,
      RegisterUnaryCallbackHandler registerUnaryCallback,
      RegisterBinaryCallbackHandler registerBinaryCallback,
      ErrorCallbackHandler errorCallback,
      ScheduleMicrotaskHandler scheduleMicrotask,
      CreateTimerHandler createTimer,
      CreatePeriodicTimerHandler createPeriodicTimer,
      PrintHandler print,
      ForkHandler fork
   }) {
    return new ZoneSpecification(
        handleUncaughtError: handleUncaughtError ?? other.handleUncaughtError,
        run: run ?? other.run,
        runUnary: runUnary ?? other.runUnary,
        runBinary: runBinary ?? other.runBinary,
        registerCallback: registerCallback ?? other.registerCallback,
        registerUnaryCallback:
            registerUnaryCallback ?? other.registerUnaryCallback,
        registerBinaryCallback:
            registerBinaryCallback ?? other.registerBinaryCallback,
        errorCallback: errorCallback ?? other.errorCallback,
        scheduleMicrotask: scheduleMicrotask ?? other.scheduleMicrotask,
        createTimer: createTimer ?? other.createTimer,
        createPeriodicTimer: createPeriodicTimer ?? other.createPeriodicTimer,
        print: print ?? other.print,
        fork: fork ?? other.fork);
  }

  /// 接口，只读
  HandleUncaughtErrorHandler get handleUncaughtError;
  RunHandler get run;
  RunUnaryHandler get runUnary;
  RunBinaryHandler get runBinary;
  RegisterCallbackHandler get registerCallback;
  RegisterUnaryCallbackHandler get registerUnaryCallback;
  RegisterBinaryCallbackHandler get registerBinaryCallback;
  ErrorCallbackHandler get errorCallback;
  ScheduleMicrotaskHandler get scheduleMicrotask;
  CreateTimerHandler get createTimer;
  CreatePeriodicTimerHandler get createPeriodicTimer;
  PrintHandler get print;
  ForkHandler get fork;
}

/// ZoneSpecification 的实现类
class _ZoneSpecification implements ZoneSpecification {
  const _ZoneSpecification({
      this.handleUncaughtError,
      this.run,
      this.runUnary,
      this.runBinary,
      this.registerCallback,
      this.registerUnaryCallback,
      this.registerBinaryCallback,
      this.errorCallback,
      this.scheduleMicrotask,
      this.createTimer,
      this.createPeriodicTimer,
      this.print,
      this.fork
  });

  final HandleUncaughtErrorHandler handleUncaughtError;
  final RunHandler run;
  final RunUnaryHandler runUnary;
  final RunBinaryHandler runBinary;
  final RegisterCallbackHandler registerCallback;
  final RegisterUnaryCallbackHandler registerUnaryCallback;
  final RegisterBinaryCallbackHandler registerBinaryCallback;
  final ErrorCallbackHandler errorCallback;
  final ScheduleMicrotaskHandler scheduleMicrotask;
  final CreateTimerHandler createTimer;
  final CreatePeriodicTimerHandler createPeriodicTimer;
  final PrintHandler print;
  final ForkHandler fork;
}
```



## ZoneDelegate

```dart
/**
 * An adapted view of the parent zone.
 *
 * 这个类允许一个 zone 调用其父级 zone 的方法，同时维持对自身的了解
 *
 * 自定义 zone (通过 [Zone.fork] 或者 [runZoned]) 能够提供绝大部分 zone 方法的自定义实现。这类似于重载 Zone 
 * 的方法，同时却不继承 Zone
 *
 * 自定义的 zone 函数（例如提供自 [ZoneSpecification]），通常来说记录或者包裹了它的参数并且通过给定的  
 * [ZoneDelegate] 来将实际操作委托给它的父级 zone，
 *
 * 虽然区域可以访问它们的父级 zone (通过 [Zone.parent])，但出于以下两个原因，建议调用所提供的父级 Delegate 
 * 上的方法:
 * 1.委托方法接受一个附加的“zone”参数，该参数是操作已在其中启动的区域。
 * 2.委托调用效率更高，因为其实现知道如何跳过 zone 只委托给父类 zone。
 */
abstract class ZoneDelegate {
  void handleUncaughtError(Zone zone, error, StackTrace stackTrace);
  R run<R>(Zone zone, R f());
  R runUnary<R, T>(Zone zone, R f(T arg), T arg);
  R runBinary<R, T1, T2>(Zone zone, R f(T1 arg1, T2 arg2), T1 arg1, T2 arg2);
  ZoneCallback<R> registerCallback<R>(Zone zone, R f());
  ZoneUnaryCallback<R, T> registerUnaryCallback<R, T>(Zone zone, R f(T arg));
  ZoneBinaryCallback<R, T1, T2> registerBinaryCallback<R, T1, T2>(
      Zone zone, R f(T1 arg1, T2 arg2));
  AsyncError errorCallback(Zone zone, Object error, StackTrace stackTrace);
  void scheduleMicrotask(Zone zone, void f());
  Timer createTimer(Zone zone, Duration duration, void f());
  Timer createPeriodicTimer(Zone zone, Duration period, void f(Timer timer));
  void print(Zone zone, String line);
  Zone fork(Zone zone, ZoneSpecification specification, Map zoneValues);
}

/// ZoneDelegate 的实现类
class _ZoneDelegate implements ZoneDelegate {
  final _Zone _delegationTarget;

  _ZoneDelegate(this._delegationTarget);

  void handleUncaughtError(Zone zone, error, StackTrace stackTrace) {
    var implementation = _delegationTarget._handleUncaughtError;
    _Zone implZone = implementation.zone;
    HandleUncaughtErrorHandler handler = implementation.function;
    return handler(
        implZone, _parentDelegate(implZone), zone, error, stackTrace);
  }

  R run<R>(Zone zone, R f()) {
    var implementation = _delegationTarget._run;
    _Zone implZone = implementation.zone;
    RunHandler handler = implementation.function;
    return handler(implZone, _parentDelegate(implZone), zone, f);
  }

  R runUnary<R, T>(Zone zone, R f(T arg), T arg) {
    var implementation = _delegationTarget._runUnary;
    _Zone implZone = implementation.zone;
    RunUnaryHandler handler = implementation.function;
    return handler(implZone, _parentDelegate(implZone), zone, f, arg);
  }

  R runBinary<R, T1, T2>(Zone zone, R f(T1 arg1, T2 arg2), T1 arg1, T2 arg2) {
    var implementation = _delegationTarget._runBinary;
    _Zone implZone = implementation.zone;
    RunBinaryHandler handler = implementation.function;
    return handler(implZone, _parentDelegate(implZone), zone, f, arg1, arg2);
  }

  ZoneCallback<R> registerCallback<R>(Zone zone, R f()) {
    var implementation = _delegationTarget._registerCallback;
    _Zone implZone = implementation.zone;
    RegisterCallbackHandler handler = implementation.function;
    return handler(implZone, _parentDelegate(implZone), zone, f);
  }

  ZoneUnaryCallback<R, T> registerUnaryCallback<R, T>(Zone zone, R f(T arg)) {
    var implementation = _delegationTarget._registerUnaryCallback;
    _Zone implZone = implementation.zone;
    RegisterUnaryCallbackHandler handler = implementation.function;
    return handler(implZone, _parentDelegate(implZone), zone, f);
  }

  ZoneBinaryCallback<R, T1, T2> registerBinaryCallback<R, T1, T2>(
      Zone zone, R f(T1 arg1, T2 arg2)) {
    var implementation = _delegationTarget._registerBinaryCallback;
    _Zone implZone = implementation.zone;
    RegisterBinaryCallbackHandler handler = implementation.function;
    return handler(implZone, _parentDelegate(implZone), zone, f);
  }

  AsyncError errorCallback(Zone zone, Object error, StackTrace stackTrace) {
    var implementation = _delegationTarget._errorCallback;
    _Zone implZone = implementation.zone;
    if (identical(implZone, _rootZone)) return null;
    ErrorCallbackHandler handler = implementation.function;
    return handler(
        implZone, _parentDelegate(implZone), zone, error, stackTrace);
  }

  void scheduleMicrotask(Zone zone, f()) {
    var implementation = _delegationTarget._scheduleMicrotask;
    _Zone implZone = implementation.zone;
    ScheduleMicrotaskHandler handler = implementation.function;
    handler(implZone, _parentDelegate(implZone), zone, f);
  }

  Timer createTimer(Zone zone, Duration duration, void f()) {
    var implementation = _delegationTarget._createTimer;
    _Zone implZone = implementation.zone;
    CreateTimerHandler handler = implementation.function;
    return handler(implZone, _parentDelegate(implZone), zone, duration, f);
  }

  Timer createPeriodicTimer(Zone zone, Duration period, void f(Timer timer)) {
    var implementation = _delegationTarget._createPeriodicTimer;
    _Zone implZone = implementation.zone;
    CreatePeriodicTimerHandler handler = implementation.function;
    return handler(implZone, _parentDelegate(implZone), zone, period, f);
  }

  void print(Zone zone, String line) {
    var implementation = _delegationTarget._print;
    _Zone implZone = implementation.zone;
    PrintHandler handler = implementation.function;
    handler(implZone, _parentDelegate(implZone), zone, line);
  }

  Zone fork(Zone zone, ZoneSpecification specification, Map zoneValues) {
    var implementation = _delegationTarget._fork;
    _Zone implZone = implementation.zone;
    ForkHandler handler = implementation.function;
    return handler(
        implZone, _parentDelegate(implZone), zone, specification, zoneValues);
  }
}

```



## Zone

```dart
/**
 * [Zone] 表示在异步调用之间保持稳定的环境。
 *
 * 代码总是在一个 zone 的上下文中执行，通过过 [Zone.current] 来获取。初始的 'main' 函数在默认的 zone 
 * [Zone.root] 中运行。可以在不同的 zone 中运行代妈：例如，使用 runZoned 来创建一个新 zone，或者，
 * 使用 [Zone.run] （这个 zone 通过 [Zone.fork] 获取到）在给定的上下文中运行代码
 *
 * 开发者可以重载一些已经存在的 zone 的方法，例如，自定义的 zone 可以替代或者修改 print 函数、timer、 
 * mircotasks的行为；或者确定如何解决未被处理的异常
 *
 * The [Zone] class is not subclassable, but users can provide custom zones by
 * forking an existing zone (usually [Zone.current]) with a [ZoneSpecification].
 * This is similar to creating a new class that extends the base `Zone` class
 * and that overrides some methods, except without actually creating a new
 * class. Instead the overriding methods are provided as functions that
 * explicitly take the equivalent of their own class, the "super" class and the
 * `this` object as parameters.
 *
 * 异步回调总是在它们被调度的 zone 的上下文中运行。这通过以下两部实现：
 * 1.回调首先使用 [registerCallback]，[registerUnaryCallback]，[registerBinaryCallback] 来注册。这允许 
 *   zone 来记录被注册的回调的存在，并且可能修改它（通过返回一个不同的回调）。这些代码执行注册工作（例如，
 *   Future.then）并且能够记住当前的 zone，并且，随后，可以在记录的 zone 中运行那些代码
 * 2.在随后的一个时间点，被注册的回调可以在记录的 zone 中运行
 * （以上两点通俗的讲，就是在 向 zone 中注册回调的时候，回调一定会在 注册它的 zone 中运行）
 *
 * 平台的内部代码处理了以上关系，并且用户无需关心。然而，通过底层系统或者 native 拓展来开发新的异步操作必须遵
 * 循协议才能与 zone 兼容。
 *
 * 通常情况下，zone 提供了 [bindCallback]（并且对应的 [bindUnaryCallback] and [bindBinaryCallback]），来
 * 使得遵循 zone 协议变得更加的容易：这些函数首先调用对应的 注册函数然后 包裹返回的函数，因此，随后在异步调用
 * 时，它可以在相同的 zone 中运行 
 *
 * 类似地，当回调被 [Zone.runGuarded] 激活的时候，zone 提供 [bindCallbackGuarded]（并且对应的 
 * [bindUnaryCallbackGuarded] and [bindBinaryCallbackGuarded]），
 */
abstract class Zone {
  // Private constructor so that it is not possible instantiate a Zone class.
  Zone._();

  /**
   * zone 的根节点
   * 所有的 isolate 块函数（`main` or 衍生的函数），从 zone 的根节点中开始运行（也就是说 [Zone.current] 与 
   * [Zone.root] 是同样的（identical））。如果没有 自定义的 zone 被创建，剩下的 程序总是在 zone 的根节点中
   * 运行。
   *
   * zone 的根节点实现了所有 zone 操作的默认行为。许多方法(如[registerCallback])，只完成了函数所需的最基本的
   * 工作，并且只作为自定义 zone 的 hook 提供。其他的方法，例如 [scheduleMicrotask]，与底层系统交互来实现需
   * 要的行为
   */
  /// 注意后文有 _RootZone 的实现类，并且创建唯一 const 实例：_rootZone = _rootZone();
  static const Zone root = _rootZone;

  /** 当前运行的 zone */
  static Zone _current = _rootZone;

  /** 当前激活的 zone */
  static Zone get current => _current;

  /**
   * 处理未捕获的异常.
   *
   * 有两种异步错误被此函数处理：
   * 1.在异步回调用中未捕获的异常。例如，传递给 [Timer.run] 的函数中，包含有一个 throw（即使使用 try 来运行 
   *   Timer.run，此 throw 也不会被捕获）
   * 2.通过 [Future] 或者 [Stream] 链推入的 异步错误，但是没有为其注册错误处理回调。
   *   大多数的异步类，例如 [Future] or [Stream] 将错误推送给它们的监听者。错误会一直传递，直到有一个监听者处 
   *   理了错误（例如，[Future.catchError]），或者没有更多监听者。在后面一种情况下，futures and streams 激
   *   活了 zone's [handleUncaughtError]
   *
   * 默认情况下，当没有被捕获的 异步错误 在 zone 的根节点中被处理，它们被当作 未捕获的同步错误。
   */
  void handleUncaughtError(error, StackTrace stackTrace);

  /**
   * 这个 zone 的父节点。
   *
   * zone 被 [fork] 从一个已存在的 zone 上创建，或者被 [runZoned] 从 [current] zone 上创建。新创建的 zone 
   * 的父节点是它被 [fork] 的来源 
   */
  Zone get parent;

  /**
   * error zone 是处理未捕获异常的 zone
   *
   * 通常情况下，是提供了 [handleUncaughtError] 方法的 最近的父节点 zone
   *
   * 异步错误绝不会跨 zone 边界二分不同的错误处理句柄
   * Asynchronous errors never cross zone boundaries between zones with
   * different error handlers.
   *
   * 用例:
   * ```dart
   * import 'dart:async';
   *
   * main() {
   *   var future;
   *   runZoned(() {
   *     // 此异步错误被自定义的 zone 所捕获，也就是下方的 onError 回调
   *     future = new Future.error("asynchronous error");
   *   }, onError: (e) { print(e); });  // 创建错误处理句柄
   *   // 此 catchError 方法绝对不会被激活
   *   // 因为，异步异常总是在它们产生的 zone 中被处理，这里 zone 指上方 runZoned 的自定义 Zone
   *   future.catchError((e) { throw "is never reached"; });
   * }
   * ```
   *
   * 注意：异常亦不会传递到 zone 的子节点中
   * either:
   * ```
   * import 'dart:async';
   *
   * main() {
   *   runZoned(() {
   *     // 下方的代码中的异常不会被子 zone（也就是下方 runZoned产生的 zone，它被视为此 zone 的子节点）所处
   *     // 理，而是触发本 zone 的 onError 回调
   *     var future = new Future.error("asynchronous error");
   *     runZoned(() {
   *       future.catchError((e) { throw "is never reached"; });
   *     }, onError: (e) { throw "is never reached"; });
   *   }, onError: (e) { print("Caught by outer zone: $e"); });
   * }
   * ```
   */
  /// 综上，这个字段保存了此 zone 未捕获的异常将会被抛出到的 zone
  Zone get errorZone;

  /**
   * 检查是否两个 zone 有相同的  error Zone
   */
  bool inSameErrorZone(Zone otherZone);

  /**
   * 创建一个 `this` 的子 zone
   *
   * 新的 zone 使用给定的 [specification] 来重载当前 zone 的行为。所有的 [specification] 中为 null 的字段
   * 将遗传本 zone（this） 的行为
   *
   * 新的 zone 遗传
   * The new zone inherits the stored values (accessed through [operator []])
   * of this zone and updates them with values from [zoneValues], which either
   * adds new values or overrides existing ones.
   *
   * 注意到 fork 操作时可拦截的，因此 zone 能够改变 zone specification（或者 zoneValues），则能够控制子类的
   * 行为
   */
  Zone fork({ZoneSpecification specification, Map zoneValues});

  /**
   * 在此 zone 执行给定的 [action]
   *
   * 默认情况下 (在 [root] zone 中实现), 运行 [action] 并将  [current] 设置成 this
   *
   * 如果 [action] 抛出异常，同步异常将不会被此 zone 的 errorhandler 捕获。
   * 使用 [runGuarded] 来捕获可能的同步异常 
   * 
   * 因为 zone 的根节点是唯一能够修改的 current 值的 zone，自定义 zone 拦截 run 应该委托给它们的父节点 zone
   * 它应该在此调用前做出行动
   */
  R run<R>(R action());
  R runUnary<R, T>(R action(T argument), T argument);
  R runBinary<R, T1, T2>(
      R action(T1 argument1, T2 argument2), 
      T1 argument1, 
      T2 argument2
  );

  /**
   * 在此 zone 执行给定的 [action] 并且捕获同步异常
   *
   * 此方法等价于：
   * ```dart
   * try {
   *   this.run(action);
   * } catch (e, s) {
   *   this.handleUncaughtError(e, s);
   * }
   * ```
   */
  void runGuarded(void action());
  void runUnaryGuarded<T>(void action(T argument), T argument);
  void runBinaryGuarded<T1, T2>(
      void action(T1 argument1, T2 argument2), 
      T1 argument1, 
      T2 argument2
  );

  /**
   * 在此 zone 中注册给定的回调
   * 
   * 当实行异步
   *
   * 在实现使用回调的异步原语时，回调必须在用户提供回调的地方使用[registerCallback]注册。这允许区域同时记录它
   * 们需要的其他信息，甚至可能包装回调，以便回调在稍后在相同的区域中运行时准备好(使用[run])。例如，区域可能决
   * 定使用回调存储堆栈跟踪(在注册时)。
   *
   * 返回应该用来替代提供的[回调]的回调。通常情况下，专区只返回原始回调。
   *
   * 自定义区域可能会拦截此操作。[Zone.root] 中的默认实现。返回原始的回调函数。
   */
  ZoneCallback<R> registerCallback<R>(R callback());
  ZoneUnaryCallback<R, T> registerUnaryCallback<R, T>(R callback(T arg));
  ZoneBinaryCallback<R, T1, T2> registerBinaryCallback<R, T1, T2>(
      R callback(T1 arg1, T2 arg2)
  );

  /**
   *  注册给定的回调 [callback] 并且返回一个将要在此 zone 中执行的函数
   *
   *  等价于：
   *
   *      ZoneCallback registered = this.registerCallback(callback);
   *      return () => this.run(registered);
   *
   */
  ZoneCallback<R> bindCallback<R>(R callback());
  ZoneUnaryCallback<R, T> bindUnaryCallback<R, T>(R callback(T argument));
  ZoneBinaryCallback<R, T1, T2> bindBinaryCallback<R, T1, T2>(
      R callback(T1 argument1, T2 argument2));

  /**
   * 注册给定的回调 [callback] 并且返回一个将要在此 zone 中执行的函数
   *
   * When the function executes, errors are caught and treated as uncaught
   * errors.
   *
   * Equivalent to:
   *
   *     ZoneCallback registered = this.registerCallback(callback);
   *     return () => this.runGuarded(registered);
   *
   */
  void Function() bindCallbackGuarded(void callback());
  void Function(T) bindUnaryCallbackGuarded<T>(void callback(T argument));
  void Function(T1, T2) bindBinaryCallbackGuarded<T1, T2>(
      void callback(T1 argument1, T2 argument2));

  /**
   * 以编程方式添加到“Future”或“Stream”中的时候，拦截错误
   * 
   * 当调用 [Completer.completeError], [StreamController.addError] 或者一些 [Future] 的构造函数，当前的 
   * zone 允许拦截并且替换错误
   *
   * 当直接接收到 error 的时候，Future 构造器激活此函数。例如 [Future.error] 或者同步地拦截错误 
   * [Future.sync]
   *
   * There is no guarantee that an error is only sent through [errorCallback]
   * once. Libraries that use intermediate controllers or completers might
   * end up invoking [errorCallback] multiple times.
   *
   * Returns `null` if no replacement is desired. Otherwise returns an instance
   * of [AsyncError] holding the new pair of error and stack trace.
   *
   * Although not recommended, the returned instance may have its `error` member
   * ([AsyncError.error]) be equal to `null` in which case the error should be
   * replaced by a [NullThrownError].
   *
   * Custom zones may intercept this operation.
   *
   * Implementations of a new asynchronous primitive that converts synchronous
   * errors to asynchronous errors rarely need to invoke [errorCallback], since
   * errors are usually reported through future completers or stream
   * controllers.
   */
  AsyncError errorCallback(Object error, StackTrace stackTrace);

  /**
   * 在这个 zone 中运行给定的 [callback]
   *
   * 全局函数 [scheduleMicrotask] 的实现委托给 current zone [scheduleMicrotask]。根 zone 的实现与底层系统
   * 交互来调度给定的函数作为 microtask
   *
   * 自定义 zone 可能拦截此操作（例如包装给定的函数）
   */
  void scheduleMicrotask(void callback());

  /**
   * 在此 zone 中执行回调的地方创建一个计时器。
   */
  Timer createTimer(Duration duration, void callback());

  /**
   * 在此 zone 中执行回调的地方创建一个周期计时器。
   */
  Timer createPeriodicTimer(Duration period, void callback(Timer timer));

  /**
   * 打印给定的 [line]
   *
   * 全局的 [print] 函数的实现委托给 current zone 的 [print] 函数，这使得它可以被拦截
   *
   * 用例如下:
   * ```dart
   * import 'dart:async';
   *
   * main() {
   *   runZoned(() {
   *     // 最终打印结果: "Intercepted: in zone".
   *     print("in zone");
   *   }, zoneSpecification: new ZoneSpecification(
   *       print: (Zone self, ZoneDelegate parent, Zone zone, String line) {
   *     parent.print(zone, "Intercepted: $line");
   *   }));
   * }
   * ```
   */
  void print(String line);

  /**
   * 调用此函数来进入给定的 zone（将 _current 修改成给定的 zone ，并且返回之前的 _current）
   * 
   * 返回先前的 current zone
   */
  static Zone _enter(Zone zone) {
    Zone previous = _current;
    _current = zone;
    return previous;
  }

  /**
   * 调用此函数来离开给定的 zone（将 _current 修改成给定的 zone）
   */
  static void _leave(Zone previous) {
    Zone._current = previous;
  }

  /**
   * 获取与给定的 key 关联的 zone-value
   *
   * 如果 zone 不包含给定的 value,在父节点 zone 中继续查找。如果仍然没有找到，返回 null
   * 
   * 可以采用任何对象作为 key，只要它实现了 '== 运算符' 和 'hashCode' 方法
   *
   * 为了控制 key 的方法为了，zone 可以同意或者拒绝对 zone-value 的访问
   */
  operator [](Object key);
}

```



## _Zone

```dart
/**
 * Zone 的基本实现类
 */
abstract class _Zone implements Zone {
  const _Zone();

  _ZoneFunction<Function> get _run;
  _ZoneFunction<Function> get _runUnary;
  _ZoneFunction<Function> get _runBinary;
  _ZoneFunction<Function> get _registerCallback;
  _ZoneFunction<Function> get _registerUnaryCallback;
  _ZoneFunction<Function> get _registerBinaryCallback;
  _ZoneFunction<ErrorCallbackHandler> get _errorCallback;
  _ZoneFunction<ScheduleMicrotaskHandler> get _scheduleMicrotask;
  _ZoneFunction<CreateTimerHandler> get _createTimer;
  _ZoneFunction<CreatePeriodicTimerHandler> get _createPeriodicTimer;
  _ZoneFunction<PrintHandler> get _print;
  _ZoneFunction<ForkHandler> get _fork;
  _ZoneFunction<HandleUncaughtErrorHandler> get _handleUncaughtError;
  _Zone get parent;
  ZoneDelegate get _delegate;
  Map get _map;

  @override
  bool inSameErrorZone(Zone otherZone) {
    return identical(this, otherZone) || identical(errorZone, otherZone.errorZone);
  }
}
```



## _CustomZone

```dart
/// 实际上的 zone，并且实现了所有的可遗传函数
class _CustomZone extends _Zone {
  /// 以下重载 _Zone 中的 get 方法
  _ZoneFunction<Function> _run;
  _ZoneFunction<Function> _runUnary;
  _ZoneFunction<Function> _runBinary;
  _ZoneFunction<Function> _registerCallback;
  _ZoneFunction<Function> _registerUnaryCallback;
  _ZoneFunction<Function> _registerBinaryCallback;
  _ZoneFunction<ErrorCallbackHandler> _errorCallback;
  _ZoneFunction<ScheduleMicrotaskHandler> _scheduleMicrotask;
  _ZoneFunction<CreateTimerHandler> _createTimer;
  _ZoneFunction<CreatePeriodicTimerHandler> _createPeriodicTimer;
  _ZoneFunction<PrintHandler> _print;
  _ZoneFunction<ForkHandler> _fork;
  _ZoneFunction<HandleUncaughtErrorHandler> _handleUncaughtError;

  // 这个 zone 的一个缓存 delegate
  ZoneDelegate _delegateCache;

  /// 父节点 zone
  final _Zone parent;

  /// zone 的作用域值声明映射。
  ///
  /// 总是 [HashMap].
  final Map _map;

  /// 获取到 delegate
  ZoneDelegate get _delegate {
    if (_delegateCache != null) return _delegateCache;
    _delegateCache = new _ZoneDelegate(this);
    return _delegateCache;
  }

  /// _CustomZone 构造函数
  _CustomZone(
      this.parent, 
      ZoneSpecification specification, 
      this._map
  ) {
    /// 根节点 zone 总是实现了如下所有函数，因此，它不会访问父节点（总是为 null）。
    /// 其余的 zone 总是有非 null 的父节点
    ///
    /// 注意：
    /// 以下方法实现了 _Zone 所有的  getter 
    /// 以下方法的实现，都是先判断给定的 specification 是否提供的  对应的函数，
    /// 如果有，则将自身 （this） 和对应的函数 包装成 _ZoneFunction；
    /// 否则，遗传父节点 zone 中对应的函数
    _run = (specification.run != null)
        ? new _ZoneFunction<Function>(this, specification.run)
        : parent._run;
    _runUnary = (specification.runUnary != null)
        ? new _ZoneFunction<Function>(this, specification.runUnary)
        : parent._runUnary;
    _runBinary = (specification.runBinary != null)
        ? new _ZoneFunction<Function>(this, specification.runBinary)
        : parent._runBinary;
    _registerCallback = (specification.registerCallback != null)
        ? new _ZoneFunction<Function>(this, specification.registerCallback)
        : parent._registerCallback;
    _registerUnaryCallback = (specification.registerUnaryCallback != null)
        ? new _ZoneFunction<Function>(this, specification.registerUnaryCallback)
        : parent._registerUnaryCallback;
    _registerBinaryCallback = (specification.registerBinaryCallback != null)
        ? new _ZoneFunction<Function>(
            this, specification.registerBinaryCallback)
        : parent._registerBinaryCallback;
    _errorCallback = (specification.errorCallback != null)
        ? new _ZoneFunction<ErrorCallbackHandler>(
            this, specification.errorCallback)
        : parent._errorCallback;
    _scheduleMicrotask = (specification.scheduleMicrotask != null)
        ? new _ZoneFunction<ScheduleMicrotaskHandler>(
            this, specification.scheduleMicrotask)
        : parent._scheduleMicrotask;
    _createTimer = (specification.createTimer != null)
        ? new _ZoneFunction<CreateTimerHandler>(this, specification.createTimer)
        : parent._createTimer;
    _createPeriodicTimer = (specification.createPeriodicTimer != null)
        ? new _ZoneFunction<CreatePeriodicTimerHandler>(
            this, specification.createPeriodicTimer)
        : parent._createPeriodicTimer;
    _print = (specification.print != null)
        ? new _ZoneFunction<PrintHandler>(this, specification.print)
        : parent._print;
    _fork = (specification.fork != null)
        ? new _ZoneFunction<ForkHandler>(this, specification.fork)
        : parent._fork;
    _handleUncaughtError = (specification.handleUncaughtError != null)
        ? new _ZoneFunction<HandleUncaughtErrorHandler>(
            this, specification.handleUncaughtError)
        : parent._handleUncaughtError;
  }

  /**
   * 最近的错误处理 zone
   *
   * 如果 this 有错误处理器（error-handler），返回 "this",否则，返回 父节点的 errorZone 
   */
  Zone get errorZone => _handleUncaughtError.zone;

  /// 以下实现了 Zone 对应的方法
  void runGuarded(void f()) {
    try {
      run(f);
    } catch (e, s) {
      handleUncaughtError(e, s);
    }
  }

  void runUnaryGuarded<T>(void f(T arg), T arg) {
    try {
      runUnary(f, arg);
    } catch (e, s) {
      handleUncaughtError(e, s);
    }
  }

  void runBinaryGuarded<T1, T2>(void f(T1 arg1, T2 arg2), T1 arg1, T2 arg2) {
    try {
      runBinary(f, arg1, arg2);
    } catch (e, s) {
      handleUncaughtError(e, s);
    }
  }

  /// 以下所有 bindXXXCallback 方法的实现都返回了 () => this.runXXX，
  /// 也就是所，在一个给定的 zone 上调用其 bindXXXCallback（function）方法，会得到一个回调。当激活此回调的是
  /// 否，会将给定的 function 在 zone 中运行
  /// 通俗的讲，就是讲给定的回调和给定的 zone 绑定在一起，得到一个新的回调
  ZoneCallback<R> bindCallback<R>(R f()) {
    var registered = registerCallback(f);
    return () => this.run(registered);
  }

  ZoneUnaryCallback<R, T> bindUnaryCallback<R, T>(R f(T arg)) {
    var registered = registerUnaryCallback(f);
    return (arg) => this.runUnary(registered, arg);
  }

  ZoneBinaryCallback<R, T1, T2> bindBinaryCallback<R, T1, T2>(
      R f(T1 arg1, T2 arg2)) {
    var registered = registerBinaryCallback(f);
    return (arg1, arg2) => this.runBinary(registered, arg1, arg2);
  }

  void Function() bindCallbackGuarded(void f()) {
    var registered = registerCallback(f);
    return () => this.runGuarded(registered);
  }

  void Function(T) bindUnaryCallbackGuarded<T>(void f(T arg)) {
    var registered = registerUnaryCallback(f);
    return (arg) => this.runUnaryGuarded(registered, arg);
  }

  void Function(T1, T2) bindBinaryCallbackGuarded<T1, T2>(
      void f(T1 arg1, T2 arg2)) {
    var registered = registerBinaryCallback(f);
    return (arg1, arg2) => this.runBinaryGuarded(registered, arg1, arg2);
  }

  operator [](Object key) {
    var result = _map[key];
    if (result != null || _map.containsKey(key)) return result;
    // If we are not the root zone, look up in the parent zone.
    if (parent != null) {
      // We do not optimize for repeatedly looking up a key which isn't
      // there. That would require storing the key and keeping it alive.
      // Copying the key/value from the parent does not keep any new values
      // alive.
      var value = parent[key];
      if (value != null) {
        _map[key] = value;
      }
      return value;
    }
    assert(this == _rootZone);
    return null;
  }

  // 能够通过 Specification 自定义的方法
  /// _parentDelegate 详见 ## 全局函数 1
  /// 注意：
  /// 以下 XXX 的实现访问了 XXX 对应的 _XXX(_ZoneFunction)，并且获取到其对应的 zone 和 parentDelegate
  /// 并且将 delegate，this，一同作为 _XXX.function 的参数完成调用
  void handleUncaughtError(error, StackTrace stackTrace) {
    var implementation = this._handleUncaughtError;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    HandleUncaughtErrorHandler handler = implementation.function;
    return handler(
        implementation.zone, parentDelegate, this, error, stackTrace);
  }

  Zone fork({ZoneSpecification specification, Map zoneValues}) {
    var implementation = this._fork;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    ForkHandler handler = implementation.function;
    return handler(
        implementation.zone, parentDelegate, this, specification, zoneValues);
  }

  R run<R>(R f()) {
    var implementation = this._run;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    RunHandler handler = implementation.function;
    return handler(implementation.zone, parentDelegate, this, f);
  }

  R runUnary<R, T>(R f(T arg), T arg) {
    var implementation = this._runUnary;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    RunUnaryHandler handler = implementation.function;
    return handler(implementation.zone, parentDelegate, this, f, arg);
  }

  R runBinary<R, T1, T2>(R f(T1 arg1, T2 arg2), T1 arg1, T2 arg2) {
    var implementation = this._runBinary;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    RunBinaryHandler handler = implementation.function;
    return handler(implementation.zone, parentDelegate, this, f, arg1, arg2);
  }

  ZoneCallback<R> registerCallback<R>(R callback()) {
    var implementation = this._registerCallback;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    RegisterCallbackHandler handler = implementation.function;
    return handler(implementation.zone, parentDelegate, this, callback);
  }

  ZoneUnaryCallback<R, T> registerUnaryCallback<R, T>(R callback(T arg)) {
    var implementation = this._registerUnaryCallback;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    RegisterUnaryCallbackHandler handler = implementation.function;
    return handler(implementation.zone, parentDelegate, this, callback);
  }

  ZoneBinaryCallback<R, T1, T2> registerBinaryCallback<R, T1, T2>(
      R callback(T1 arg1, T2 arg2)) {
    var implementation = this._registerBinaryCallback;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    RegisterBinaryCallbackHandler handler = implementation.function;
    return handler(implementation.zone, parentDelegate, this, callback);
  }

  AsyncError errorCallback(Object error, StackTrace stackTrace) {
    var implementation = this._errorCallback;
    assert(implementation != null);
    final Zone implementationZone = implementation.zone;
    if (identical(implementationZone, _rootZone)) return null;
    final ZoneDelegate parentDelegate = _parentDelegate(implementationZone);
    ErrorCallbackHandler handler = implementation.function;
    return handler(implementationZone, parentDelegate, this, error, stackTrace);
  }

  void scheduleMicrotask(void f()) {
    var implementation = this._scheduleMicrotask;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    ScheduleMicrotaskHandler handler = implementation.function;
    return handler(implementation.zone, parentDelegate, this, f);
  }

  Timer createTimer(Duration duration, void f()) {
    var implementation = this._createTimer;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    CreateTimerHandler handler = implementation.function;
    return handler(implementation.zone, parentDelegate, this, duration, f);
  }

  Timer createPeriodicTimer(Duration duration, void f(Timer timer)) {
    var implementation = this._createPeriodicTimer;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    CreatePeriodicTimerHandler handler = implementation.function;
    return handler(implementation.zone, parentDelegate, this, duration, f);
  }

  void print(String line) {
    var implementation = this._print;
    assert(implementation != null);
    ZoneDelegate parentDelegate = _parentDelegate(implementation.zone);
    PrintHandler handler = implementation.function;
    return handler(implementation.zone, parentDelegate, this, line);
  }
}

```



## 全局函数 1

```dart
/// 返回给定 zone 的父节点的 delegate
ZoneDelegate _parentDelegate(_Zone zone) {
  if (zone.parent == null) return null;
  return zone.parent._delegate;
}

/// 以下为根节点 zone 的各方法的实现，parent 参数恒为 null
void _rootHandleUncaughtError(
    Zone self, 
    ZoneDelegate parent, 
    Zone zone, 
    error, 
    StackTrace stackTrace
) {
  _schedulePriorityAsyncCallback(() {
    error ??= new NullThrownError();
    if (stackTrace == null) throw error;
    _rethrow(error, stackTrace);
  });
}

external void _rethrow(Object error, StackTrace stackTrace);

R _rootRun<R>(
    Zone self, 
    ZoneDelegate parent, 
    Zone zone, 
    R f()
) {
  /// Zone._current == this，即当前执行的 zone 就是 根节点 zone
  if (Zone._current == zone) return f();
  /// 以下操作在接下来的函数中经常有使用
  /// 修改 Zone._current 为给定的 zone
  Zone old = Zone._enter(zone);
  try {
    return f();
  } finally {
    /// 退出给定的 zone
    Zone._leave(old);
  }
}

R _rootRunUnary<R, T>(
    Zone self,
    ZoneDelegate parent,
    Zone zone, 
    R f(T arg), 
    T arg
) {
  if (Zone._current == zone) return f(arg);

  Zone old = Zone._enter(zone);
  try {
    return f(arg);
  } finally {
    Zone._leave(old);
  }
}

R _rootRunBinary<R, T1, T2>(
    Zone self,
    ZoneDelegate parent, 
    Zone zone,
    R f(T1 arg1, T2 arg2),
    T1 arg1,
    T2 arg2
) {
  if (Zone._current == zone) return f(arg1, arg2);

  Zone old = Zone._enter(zone);
  try {
    return f(arg1, arg2);
  } finally {
    Zone._leave(old);
  }
}

ZoneCallback<R> _rootRegisterCallback<R>(
    Zone self, 
    ZoneDelegate parent, 
    Zone zone, R f()
) {
  return f;
}

ZoneUnaryCallback<R, T> _rootRegisterUnaryCallback<R, T>(
    Zone self, 
    ZoneDelegate parent,
    Zone zone, 
    R f(T arg)
) {
  return f;
}

ZoneBinaryCallback<R, T1, T2> _rootRegisterBinaryCallback<R, T1, T2>(
    Zone self, 
    ZoneDelegate parent, 
    Zone zone,
    R f(T1 arg1, T2 arg2)
) {
  return f;
}

AsyncError _rootErrorCallback(
    Zone self, 
    ZoneDelegate parent,
    Zone zone,
    Object error, 
    StackTrace stackTrace
) => null;

/// 调度微任务
void _rootScheduleMicrotask(
    Zone self,
    ZoneDelegate parent,
    Zone zone, 
    void f()
) {
  if (!identical(_rootZone, zone)) {
    bool hasErrorHandler = !_rootZone.inSameErrorZone(zone);
    if (hasErrorHandler) {
      f = zone.bindCallbackGuarded(f);
    } else {
      f = zone.bindCallback(f);
    }
    // Use root zone as event zone if the function is already bound.
    zone = _rootZone;
  }
  _scheduleAsyncCallback(f);
}

Timer _rootCreateTimer(
    Zone self, 
    ZoneDelegate parent, 
    Zone zone,
    Duration duration, 
    void callback()
) {
  if (!identical(_rootZone, zone)) {
    callback = zone.bindCallback(callback);
  }
  return Timer._createTimer(duration, callback);
}

Timer _rootCreatePeriodicTimer(
    Zone self, 
    ZoneDelegate parent,
    Zone zone,
    Duration duration, 
    void callback(Timer timer)
) {
  if (!identical(_rootZone, zone)) {
    // TODO(floitsch): the return type should be 'void'.
    callback = zone.bindUnaryCallback<dynamic, Timer>(callback);
  }
  return Timer._createPeriodicTimer(duration, callback);
}

void _rootPrint(
    Zone self, 
    ZoneDelegate parent, 
    Zone zone, 
    String line
) {
  printToConsole(line);
}

void _printToZone(String line) {
  Zone.current.print(line);
}

Zone _rootFork(
    Zone self, 
    ZoneDelegate parent, 
    Zone zone,
    ZoneSpecification specification, 
    Map zoneValues
) {
  // TODO(floitsch): it would be nice if we could get rid of this hack.
  // Change the static zoneOrDirectPrint function to go through zones
  // from now on.
  printToZone = _printToZone;

  if (specification == null) {
    specification = const ZoneSpecification();
  } else if (specification is! _ZoneSpecification) {
    throw new ArgumentError(
        "ZoneSpecifications must be instantiated"
        " with the provided constructor."
    );
  }
  Map valueMap;
  if (zoneValues == null) {
    if (zone is _Zone) {
      valueMap = zone._map;
    } else {
      valueMap = new HashMap();
    }
  } else {
    valueMap = new HashMap.from(zoneValues);
  }
  return new _CustomZone(zone, specification, valueMap);
}
```



## _RootZone

```dart
class _RootZone extends _Zone {
  const _RootZone();

  _ZoneFunction<Function> get _run =>
      const _ZoneFunction<Function>(_rootZone, _rootRun);
  _ZoneFunction<Function> get _runUnary =>
      const _ZoneFunction<Function>(_rootZone, _rootRunUnary);
  _ZoneFunction<Function> get _runBinary =>
      const _ZoneFunction<Function>(_rootZone, _rootRunBinary);
  _ZoneFunction<Function> get _registerCallback =>
      const _ZoneFunction<Function>(_rootZone, _rootRegisterCallback);
  _ZoneFunction<Function> get _registerUnaryCallback =>
      const _ZoneFunction<Function>(_rootZone, _rootRegisterUnaryCallback);
  _ZoneFunction<Function> get _registerBinaryCallback =>
      const _ZoneFunction<Function>(_rootZone, _rootRegisterBinaryCallback);
  _ZoneFunction<ErrorCallbackHandler> get _errorCallback =>
      const _ZoneFunction<ErrorCallbackHandler>(_rootZone, _rootErrorCallback);
  _ZoneFunction<ScheduleMicrotaskHandler> get _scheduleMicrotask =>
      const _ZoneFunction<ScheduleMicrotaskHandler>(
          _rootZone, _rootScheduleMicrotask);
  _ZoneFunction<CreateTimerHandler> get _createTimer =>
      const _ZoneFunction<CreateTimerHandler>(_rootZone, _rootCreateTimer);
  _ZoneFunction<CreatePeriodicTimerHandler> get _createPeriodicTimer =>
      const _ZoneFunction<CreatePeriodicTimerHandler>(
          _rootZone, _rootCreatePeriodicTimer);
  _ZoneFunction<PrintHandler> get _print =>
      const _ZoneFunction<PrintHandler>(_rootZone, _rootPrint);
  _ZoneFunction<ForkHandler> get _fork =>
      const _ZoneFunction<ForkHandler>(_rootZone, _rootFork);
  _ZoneFunction<HandleUncaughtErrorHandler> get _handleUncaughtError =>
      const _ZoneFunction<HandleUncaughtErrorHandler>(
          _rootZone, _rootHandleUncaughtError);

  // The parent zone.
  _Zone get parent => null;

  /// The zone's scoped value declaration map.
  ///
  /// This is always a [HashMap].
  Map get _map => _rootMap;

  static final _rootMap = new HashMap();

  static ZoneDelegate _rootDelegate;

  ZoneDelegate get _delegate {
    if (_rootDelegate != null) return _rootDelegate;
    return _rootDelegate = new _ZoneDelegate(this);
  }

  /**
   * The closest error-handling zone.
   *
   * Returns `this` if `this` has an error-handler. Otherwise returns the
   * parent's error-zone.
   */
  Zone get errorZone => this;

  // Zone interface.

  void runGuarded(void f()) {
    try {
      if (identical(_rootZone, Zone._current)) {
        f();
        return;
      }
      _rootRun(null, null, this, f);
    } catch (e, s) {
      handleUncaughtError(e, s);
    }
  }

  void runUnaryGuarded<T>(void f(T arg), T arg) {
    try {
      if (identical(_rootZone, Zone._current)) {
        f(arg);
        return;
      }
      _rootRunUnary(null, null, this, f, arg);
    } catch (e, s) {
      handleUncaughtError(e, s);
    }
  }

  void runBinaryGuarded<T1, T2>(void f(T1 arg1, T2 arg2), T1 arg1, T2 arg2) {
    try {
      if (identical(_rootZone, Zone._current)) {
        f(arg1, arg2);
        return;
      }
      _rootRunBinary(null, null, this, f, arg1, arg2);
    } catch (e, s) {
      handleUncaughtError(e, s);
    }
  }

  ZoneCallback<R> bindCallback<R>(R f()) {
    return () => this.run<R>(f);
  }

  ZoneUnaryCallback<R, T> bindUnaryCallback<R, T>(R f(T arg)) {
    return (arg) => this.runUnary<R, T>(f, arg);
  }

  ZoneBinaryCallback<R, T1, T2> bindBinaryCallback<R, T1, T2>(
      R f(T1 arg1, T2 arg2)) {
    return (arg1, arg2) => this.runBinary<R, T1, T2>(f, arg1, arg2);
  }

  void Function() bindCallbackGuarded(void f()) {
    return () => this.runGuarded(f);
  }

  void Function(T) bindUnaryCallbackGuarded<T>(void f(T arg)) {
    return (arg) => this.runUnaryGuarded(f, arg);
  }

  void Function(T1, T2) bindBinaryCallbackGuarded<T1, T2>(
      void f(T1 arg1, T2 arg2)) {
    return (arg1, arg2) => this.runBinaryGuarded(f, arg1, arg2);
  }

  operator [](Object key) => null;

  // Methods that can be customized by the zone specification.

  void handleUncaughtError(error, StackTrace stackTrace) {
    _rootHandleUncaughtError(null, null, this, error, stackTrace);
  }

  Zone fork({ZoneSpecification specification, Map zoneValues}) {
    return _rootFork(null, null, this, specification, zoneValues);
  }

  R run<R>(R f()) {
    if (identical(Zone._current, _rootZone)) return f();
    return _rootRun(null, null, this, f);
  }

  R runUnary<R, T>(R f(T arg), T arg) {
    if (identical(Zone._current, _rootZone)) return f(arg);
    return _rootRunUnary(null, null, this, f, arg);
  }

  R runBinary<R, T1, T2>(R f(T1 arg1, T2 arg2), T1 arg1, T2 arg2) {
    if (identical(Zone._current, _rootZone)) return f(arg1, arg2);
    return _rootRunBinary(null, null, this, f, arg1, arg2);
  }

  ZoneCallback<R> registerCallback<R>(R f()) => f;

  ZoneUnaryCallback<R, T> registerUnaryCallback<R, T>(R f(T arg)) => f;

  ZoneBinaryCallback<R, T1, T2> registerBinaryCallback<R, T1, T2>(
          R f(T1 arg1, T2 arg2)) =>
      f;

  AsyncError errorCallback(Object error, StackTrace stackTrace) => null;

  void scheduleMicrotask(void f()) {
    _rootScheduleMicrotask(null, null, this, f);
  }

  Timer createTimer(Duration duration, void f()) {
    return Timer._createTimer(duration, f);
  }

  Timer createPeriodicTimer(Duration duration, void f(Timer timer)) {
    return Timer._createPeriodicTimer(duration, f);
  }

  void print(String line) {
    printToConsole(line);
  }
}

const _rootZone = const _RootZone();
```



## 全局函数 2

```dart
/**
 * Runs [body] in its own zone.
 *
 * Creates a new zone using [Zone.fork] based on [zoneSpecification] and
 * [zoneValues], then runs [body] in that zone and returns the result.
 *
 * If [onError] is provided, it must have one of the types
 * * `void Function(Object)`
 * * `void Function(Object, StackTrace)`
 * and the [onError] handler is used *both* to handle asynchronous errors
 * by overriding [ZoneSpecification.handleUncaughtError] in [zoneSpecification],
 * if any, *and* to handle errors thrown synchronously by the call to [body].
 *
 * If an error occurs synchronously in [body],
 * then throwing in the [onError] handler
 * makes the call to `runZone` throw that error,
 * and otherwise the call to `runZoned` returns `null`.
 *
 * If the zone specification has a `handleUncaughtError` value or the [onError]
 * parameter is provided, the zone becomes an error-zone.
 *
 * Errors will never cross error-zone boundaries by themselves.
 * Errors that try to cross error-zone boundaries are considered uncaught in
 * their originating error zone.
 *
 *     var future = new Future.value(499);
 *     runZoned(() {
 *       var future2 = future.then((_) { throw "error in first error-zone"; });
 *       runZoned(() {
 *         var future3 = future2.catchError((e) { print("Never reached!"); });
 *       }, onError: (e) { print("unused error handler"); });
 *     }, onError: (e) { print("catches error of first error-zone."); });
 *
 * Example:
 *
 *     runZoned(() {
 *       new Future(() { throw "asynchronous error"; });
 *     }, onError: print);  // Will print "asynchronous error".
 *
 * It is possible to manually pass an error from one error zone to another
 * by re-throwing it in the new zone. If [onError] throws, that error will
 * occur in the original zone where [runZoned] was called.
 */
R runZoned<R>(
    R body(),{
    Map zoneValues, 
    ZoneSpecification zoneSpecification, 
    Function onError
}) {
  if (onError == null) {
    return _runZoned<R>(body, zoneValues, zoneSpecification);
  }
  void Function(Object) unaryOnError;
  void Function(Object, StackTrace) binaryOnError;
  if (onError is void Function(Object, StackTrace)) {
    binaryOnError = onError;
  } else if (onError is void Function(Object)) {
    unaryOnError = onError;
  } else {
    throw new ArgumentError("onError callback must take either an Object "
        "(the error), or both an Object (the error) and a StackTrace.");
  }
  HandleUncaughtErrorHandler errorHandler = (Zone self, ZoneDelegate parent,
      Zone zone, error, StackTrace stackTrace) {
    try {
      if (binaryOnError != null) {
        self.parent.runBinary(binaryOnError, error, stackTrace);
      } else {
        assert(unaryOnError != null);
        self.parent.runUnary(unaryOnError, error);
      }
    } catch (e, s) {
      if (identical(e, error)) {
        parent.handleUncaughtError(zone, error, stackTrace);
      } else {
        parent.handleUncaughtError(zone, e, s);
      }
    }
  };
  if (zoneSpecification == null) {
    zoneSpecification =
        new ZoneSpecification(handleUncaughtError: errorHandler);
  } else {
    zoneSpecification = new ZoneSpecification.from(zoneSpecification,
        handleUncaughtError: errorHandler);
  }
  try {
    return _runZoned<R>(body, zoneValues, zoneSpecification);
  } catch (e, stackTrace) {
    if (binaryOnError != null) {
      binaryOnError(e, stackTrace);
    } else {
      assert(unaryOnError != null);
      unaryOnError(e);
    }
  }
  return null;
}

/// Runs [body] in a new zone based on [zoneValues] and [specification].
R _runZoned<R>(
    R body(), 
    Map zoneValues, 
    ZoneSpecification specification
) =>Zone.current
        .fork(specification: specification, zoneValues: zoneValues)
        .run<R>(body);

```


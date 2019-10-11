# Dart Futrue 分析

[TOC]

## FutureOr

```dart
/// 一个能表示 Future<T>` 或 `T` 的对象
///
/// 这个类声明是内部future-or-value泛型类型的公共替代。对该类的引用被解析为内部类型。
///
/// It is a compile-time error for any class to extend, mix in or implement
/// `FutureOr`.
///
/// 注意: 'FutureOr<T>'类型在非强类型模式下被解释为'dynamic'。
///
/// 作为一个推论, 
/// `FutureOr<Object>` 等价于 `FutureOr<FutureOr<Object>>`， 
/// `FutureOr<Future<Object>>` 等价于 `Future<Object>`.
abstract class FutureOr<T> {
  // 私有构造函数，因此它是不可子类化、可混合或可实例化的  
  FutureOr._() {
    throw new UnsupportedError("FutureOr can't be instantiated");
  }
}
```



## Futrue

```dart
abstract class Future<T> {
  /// 一个以 `null` 完成的 `Future<Null>`
  static final _Future<Null> _nullFuture =
      new _Future<Null>.zoneValue(null, Zone.root);

  /// 一个以 `false` 完成的 `Future<bool>`
  static final _Future<bool> _falseFuture =
      new _Future<bool>.zoneValue(false, Zone.root);

  /**
   * 创建一个包含用[Timer.run]异步调用[[computation]结果的future
   *
   * 如果执行的[computation]抛出了异常, 返回以 `error` 完成的 futrue
   *
   * 如果返回的值本身是一个[Future]，则创建的Future的完成将一直等到返回的Future完成，然后将以相同
   * 的结果完成。
   *
   * 如果返回一个非Futrue值，则返回的Futrue将使用该值完成。
   */
  factory Future(FutureOr<T> computation()) {
    _Future<T> result = new _Future<T>();
    Timer.run(() {
      try {
        result._complete(computation());
      } catch (e, s) {
        _completeWithErrorCallback(result, e, s);
      }
    });
    return result;
  }

   /**
   * 创建一个包含与[scheduleMicrotask]异步调用[computation]的结果的future
   *
   * 如果执行的[computation]抛出了异常, 返回以 `error` 完成的 futrue
   *
   * 如果返回的值本身是一个[Future]，则创建的Future的完成将一直等到返回的Future完成，然后将以相同
   * 的结果完成。
   *
   * 如果返回一个非Futrue值，则返回的Futrue将使用该值完成。
   */
  factory Future.microtask(FutureOr<T> computation()) {
    _Future<T> result = new _Future<T>();
    scheduleMicrotask(() {
      try {
        result._complete(computation());
      } catch (e, s) {
        _completeWithErrorCallback(result, e, s);
      }
    });
    return result;
  }

  /**
   * 创建一个包含立即调用[computation]的结果的future
   *
   * 如果执行的[computation]抛出了异常, 返回以 `error` 完成的 futrue
   *
   * 如果返回的值本身是一个[Future]，则创建的Future的完成将一直等到返回的Future完成，然后将以相同
   * 的结果完成。
   *
   * 如果返回一个非Futrue值，则返回的Futrue将使用该值完成。
   */
  factory Future.sync(FutureOr<T> computation()) {
    try {
      var result = computation();
      if (result is Future<T>) {
        return result;
      } else {
        return new _Future<T>.value(result);
      }
    } catch (error, stackTrace) {
      var future = new _Future<T>();
      AsyncError replacement = Zone.current.errorCallback(error, stackTrace);
      if (replacement != null) {
        future._asyncCompleteError(
            _nonNullError(replacement.error), replacement.stackTrace);
      } else {
        future._asyncCompleteError(error, stackTrace);
      }
      return future;
    }
  }

  /**
   * Creates a future completed with [value].
   *
   * If [value] is a future, the created future waits for the
   * [value] future to complete, and then completes with the same result.
   * Since a [value] future can complete with an error, so can the future
   * created by [Future.value], even if the name suggests otherwise.
   *
   * If [value] is not a [Future], the created future is completed
   * with the [value] value,
   * equivalently to `new Future<T>.sync(() => value)`.
   *
   * Use [Completer] to create a future and complete it later.
   */
  factory Future.value([FutureOr<T> value]) {
    return new _Future<T>.immediate(value);
  }

  /**
   * Creates a future that completes with an error.
   *
   * The created future will be completed with an error in a future microtask.
   * This allows enough time for someone to add an error handler on the future.
   * If an error handler isn't added before the future completes, the error
   * will be considered unhandled.
   *
   * If [error] is `null`, it is replaced by a [NullThrownError].
   *
   * Use [Completer] to create a future and complete it later.
   */
  factory Future.error(Object error, [StackTrace stackTrace]) {
    error = _nonNullError(error);
    if (!identical(Zone.current, _rootZone)) {
      AsyncError replacement = Zone.current.errorCallback(error, stackTrace);
      if (replacement != null) {
        error = _nonNullError(replacement.error);
        stackTrace = replacement.stackTrace;
      }
    }
    return new _Future<T>.immediateError(error, stackTrace);
  }

  /**
   * Creates a future that runs its computation after a delay.
   *
   * The [computation] will be executed after the given [duration] has passed,
   * and the future is completed with the result of the computation.
   *
   * If [computation] returns a future,
   * the future returned by this constructor will complete with the value or
   * error of that future.
   *
   * If the duration is 0 or less,
   * it completes no sooner than in the next event-loop iteration,
   * after all microtasks have run.
   *
   * If [computation] is omitted,
   * it will be treated as if [computation] was `() => null`,
   * and the future will eventually complete with the `null` value.
   *
   * If calling [computation] throws, the created future will complete with the
   * error.
   *
   * See also [Completer] for a way to create and complete a future at a
   * later time that isn't necessarily after a known fixed duration.
   */
  factory Future.delayed(Duration duration, [FutureOr<T> computation()]) {
    _Future<T> result = new _Future<T>();
    new Timer(duration, () {
      if (computation == null) {
        result._complete(null);
      } else {
        try {
          result._complete(computation());
        } catch (e, s) {
          _completeWithErrorCallback(result, e, s);
        }
      }
    });
    return result;
  }

  /**
   * Waits for multiple futures to complete and collects their results.
   *
   * Returns a future which will complete once all the provided futures
   * have completed, either with their results, or with an error if any
   * of the provided futures fail.
   *
   * The value of the returned future will be a list of all the values that
   * were produced in the order that the futures are provided by iterating
   * [futures].
   *
   * If any future completes with an error,
   * then the returned future completes with that error.
   * If further futures also complete with errors, those errors are discarded.
   *
   * If `eagerError` is true, the returned future completes with an error
   * immediately on the first error from one of the futures. Otherwise all
   * futures must complete before the returned future is completed (still with
   * the first error; the remaining errors are silently dropped).
   *
   * In the case of an error, [cleanUp] (if provided), is invoked on any
   * non-null result of successful futures.
   * This makes it possible to `cleanUp` resources that would otherwise be
   * lost (since the returned future does not provide access to these values).
   * The [cleanUp] function is unused if there is no error.
   *
   * The call to [cleanUp] should not throw. If it does, the error will be an
   * uncaught asynchronous error.
   */
  static Future<List<T>> wait<T>(Iterable<Future<T>> futures,
      {bool eagerError: false, void cleanUp(T successValue)}) {
    final _Future<List<T>> result = new _Future<List<T>>();
    List<T> values; // Collects the values. Set to null on error.
    int remaining = 0; // How many futures are we waiting for.
    var error; // The first error from a future.
    StackTrace stackTrace; // The stackTrace that came with the error.

    // Handle an error from any of the futures.
    // TODO(jmesserly): use `void` return type once it can be inferred for the
    // `then` call below.
    handleError(theError, StackTrace theStackTrace) {
      remaining--;
      if (values != null) {
        if (cleanUp != null) {
          for (var value in values) {
            if (value != null) {
              // Ensure errors from cleanUp are uncaught.
              new Future.sync(() {
                cleanUp(value);
              });
            }
          }
        }
        values = null;
        if (remaining == 0 || eagerError) {
          result._completeError(theError, theStackTrace);
        } else {
          error = theError;
          stackTrace = theStackTrace;
        }
      } else if (remaining == 0 && !eagerError) {
        result._completeError(error, stackTrace);
      }
    }

    try {
      // As each future completes, put its value into the corresponding
      // position in the list of values.
      for (var future in futures) {
        int pos = remaining;
        future.then((T value) {
          remaining--;
          if (values != null) {
            values[pos] = value;
            if (remaining == 0) {
              result._completeWithValue(values);
            }
          } else {
            if (cleanUp != null && value != null) {
              // Ensure errors from cleanUp are uncaught.
              new Future.sync(() {
                cleanUp(value);
              });
            }
            if (remaining == 0 && !eagerError) {
              result._completeError(error, stackTrace);
            }
          }
        }, onError: handleError);
        // Increment the 'remaining' after the call to 'then'.
        // If that call throws, we don't expect any future callback from
        // the future, and we also don't increment remaining.
        remaining++;
      }
      if (remaining == 0) {
        return new Future.value(const []);
      }
      values = new List<T>(remaining);
    } catch (e, st) {
      // The error must have been thrown while iterating over the futures
      // list, or while installing a callback handler on the future.
      if (remaining == 0 || eagerError) {
        // Throw a new Future.error.
        // Don't just call `result._completeError` since that would propagate
        // the error too eagerly, not giving the callers time to install
        // error handlers.
        // Also, don't use `_asyncCompleteError` since that one doesn't give
        // zones the chance to intercept the error.
        return new Future.error(e, st);
      } else {
        // Don't allocate a list for values, thus indicating that there was an
        // error.
        // Set error to the caught exception.
        error = e;
        stackTrace = st;
      }
    }
    return result;
  }

  /**
   * Returns the result of the first future in [futures] to complete.
   *
   * The returned future is completed with the result of the first
   * future in [futures] to report that it is complete,
   * whether it's with a value or an error.
   * The results of all the other futures are discarded.
   *
   * If [futures] is empty, or if none of its futures complete,
   * the returned future never completes.
   */
  static Future<T> any<T>(Iterable<Future<T>> futures) {
    var completer = new Completer<T>.sync();
    var onValue = (T value) {
      if (!completer.isCompleted) completer.complete(value);
    };
    var onError = (error, StackTrace stack) {
      if (!completer.isCompleted) completer.completeError(error, stack);
    };
    for (var future in futures) {
      future.then(onValue, onError: onError);
    }
    return completer.future;
  }

  /**
   * Performs an action for each element of the iterable, in turn.
   *
   * The [action] may be either synchronous or asynchronous.
   *
   * Calls [action] with each element in [elements] in order.
   * If the call to [action] returns a `Future<T>`, the iteration waits
   * until the future is completed before continuing with the next element.
   *
   * Returns a [Future] that completes with `null` when all elements have been
   * processed.
   *
   * Non-[Future] return values, and completion-values of returned [Future]s,
   * are discarded.
   *
   * Any error from [action], synchronous or asynchronous,
   * will stop the iteration and be reported in the returned [Future].
   */
  static Future forEach<T>(Iterable<T> elements, FutureOr action(T element)) {
    var iterator = elements.iterator;
    return doWhile(() {
      if (!iterator.moveNext()) return false;
      var result = action(iterator.current);
      if (result is Future) return result.then(_kTrue);
      return true;
    });
  }

  // Constant `true` function, used as callback by [forEach].
  static bool _kTrue(_) => true;

  /**
   * Performs an operation repeatedly until it returns `false`.
   *
   * The operation, [action], may be either synchronous or asynchronous.
   *
   * The operation is called repeatedly as long as it returns either the [bool]
   * value `true` or a `Future<bool>` which completes with the value `true`.
   *
   * If a call to [action] returns `false` or a [Future] that completes to
   * `false`, iteration ends and the future returned by [doWhile] is completed
   * with a `null` value.
   *
   * If a call to [action] throws or a future returned by [action] completes
   * with an error, iteration ends and the future returned by [doWhile]
   * completes with the same error.
   *
   * Calls to [action] may happen at any time,
   * including immediately after calling `doWhile`.
   * The only restriction is a new call to [action] won't happen before
   * the previous call has returned, and if it returned a `Future<bool>`, not
   * until that future has completed.
   */
  static Future doWhile(FutureOr<bool> action()) {
    _Future doneSignal = new _Future();
    void Function(bool) nextIteration;
    // Bind this callback explicitly so that each iteration isn't bound in the
    // context of all the previous iterations' callbacks.
    // This avoids, e.g., deeply nested stack traces from the stack trace
    // package.
    nextIteration = Zone.current.bindUnaryCallbackGuarded((bool keepGoing) {
      while (keepGoing) {
        FutureOr<bool> result;
        try {
          result = action();
        } catch (error, stackTrace) {
          // Cannot use _completeWithErrorCallback because it completes
          // the future synchronously.
          _asyncCompleteWithErrorCallback(doneSignal, error, stackTrace);
          return;
        }
        if (result is Future<bool>) {
          result.then(nextIteration, onError: doneSignal._completeError);
          return;
        }
        keepGoing = result;
      }
      doneSignal._complete(null);
    });
    nextIteration(true);
    return doneSignal;
  }

  /**
   * Register callbacks to be called when this future completes.
   *
   * When this future completes with a value,
   * the [onValue] callback will be called with that value.
   * If this future is already completed, the callback will not be called
   * immediately, but will be scheduled in a later microtask.
   *
   * If [onError] is provided, and this future completes with an error,
   * the `onError` callback is called with that error and its stack trace.
   * The `onError` callback must accept either one argument or two arguments
   * where the latter is a [StackTrace].
   * If `onError` accepts two arguments,
   * it is called with both the error and the stack trace,
   * otherwise it is called with just the error object.
   * The `onError` callback must return a value or future that can be used
   * to complete the returned future, so it must be something assignable to
   * `FutureOr<R>`.
   *
   * Returns a new [Future]
   * which is completed with the result of the call to `onValue`
   * (if this future completes with a value)
   * or to `onError` (if this future completes with an error).
   *
   * If the invoked callback throws,
   * the returned future is completed with the thrown error
   * and a stack trace for the error.
   * In the case of `onError`,
   * if the exception thrown is `identical` to the error argument to `onError`,
   * the throw is considered a rethrow,
   * and the original stack trace is used instead.
   *
   * If the callback returns a [Future],
   * the future returned by `then` will be completed with
   * the same result as the future returned by the callback.
   *
   * If [onError] is not given, and this future completes with an error,
   * the error is forwarded directly to the returned future.
   *
   * In most cases, it is more readable to use [catchError] separately, possibly
   * with a `test` parameter, instead of handling both value and error in a
   * single [then] call.
   *
   * Note that futures don't delay reporting of errors until listeners are
   * added. If the first `then` or `catchError` call happens after this future
   * has completed with an error then the error is reported as unhandled error.
   * See the description on [Future].
   */
  Future<R> then<R>(FutureOr<R> onValue(T value), {Function onError});

  /**
   * Handles errors emitted by this [Future].
   *
   * This is the asynchronous equivalent of a "catch" block.
   *
   * Returns a new [Future] that will be completed with either the result of
   * this future or the result of calling the `onError` callback.
   *
   * If this future completes with a value,
   * the returned future completes with the same value.
   *
   * If this future completes with an error,
   * then [test] is first called with the error value.
   *
   * If `test` returns false, the exception is not handled by this `catchError`,
   * and the returned future completes with the same error and stack trace
   * as this future.
   *
   * If `test` returns `true`,
   * [onError] is called with the error and possibly stack trace,
   * and the returned future is completed with the result of this call
   * in exactly the same way as for [then]'s `onError`.
   *
   * If `test` is omitted, it defaults to a function that always returns true.
   * The `test` function should not throw, but if it does, it is handled as
   * if the `onError` function had thrown.
   *
   * Note that futures don't delay reporting of errors until listeners are
   * added. If the first `catchError` (or `then`) call happens after this future
   * has completed with an error then the error is reported as unhandled error.
   * See the description on [Future].
   */
  // The `Function` below stands for one of two types:
  // - (dynamic) -> FutureOr<T>
  // - (dynamic, StackTrace) -> FutureOr<T>
  // Given that there is a `test` function that is usually used to do an
  // `isCheck` we should also expect functions that take a specific argument.
  // Note: making `catchError` return a `Future<T>` in non-strong mode could be
  // a breaking change.
  Future<T> catchError(Function onError, {bool test(Object error)});

  /**
   * Registers a function to be called when this future completes.
   *
   * The [action] function is called when this future completes, whether it
   * does so with a value or with an error.
   *
   * This is the asynchronous equivalent of a "finally" block.
   *
   * The future returned by this call, `f`, will complete the same way
   * as this future unless an error occurs in the [action] call, or in
   * a [Future] returned by the [action] call. If the call to [action]
   * does not return a future, its return value is ignored.
   *
   * If the call to [action] throws, then `f` is completed with the
   * thrown error.
   *
   * If the call to [action] returns a [Future], `f2`, then completion of
   * `f` is delayed until `f2` completes. If `f2` completes with
   * an error, that will be the result of `f` too. The value of `f2` is always
   * ignored.
   *
   * This method is equivalent to:
   *
   *     Future<T> whenComplete(action()) {
   *       return this.then((v) {
   *         var f2 = action();
   *         if (f2 is Future) return f2.then((_) => v);
   *         return v
   *       }, onError: (e) {
   *         var f2 = action();
   *         if (f2 is Future) return f2.then((_) { throw e; });
   *         throw e;
   *       });
   *     }
   */
  Future<T> whenComplete(FutureOr action());

  /**
   * Creates a [Stream] containing the result of this future.
   *
   * The stream will produce single data or error event containing the
   * completion result of this future, and then it will close with a
   * done event.
   *
   * If the future never completes, the stream will not produce any events.
   */
  Stream<T> asStream();

  /**
   * Time-out the future computation after [timeLimit] has passed.
   *
   * Returns a new future that completes with the same value as this future,
   * if this future completes in time.
   *
   * If this future does not complete before `timeLimit` has passed,
   * the [onTimeout] action is executed instead, and its result (whether it
   * returns or throws) is used as the result of the returned future.
   * The [onTimeout] function must return a [T] or a `Future<T>`.
   *
   * If `onTimeout` is omitted, a timeout will cause the returned future to
   * complete with a [TimeoutException].
   */
  Future<T> timeout(Duration timeLimit, {FutureOr<T> onTimeout()});
}
```



## _Future

```dart
class _Future<T> implements Future<T> {
  /// Initial state, waiting for a result. In this state, the
  /// [resultOrListeners] field holds a single-linked list of
  /// [_FutureListener] listeners.
  static const int _stateIncomplete = 0;

  /// Pending completion. Set when completed using [_asyncComplete] or
  /// [_asyncCompleteError]. It is an error to try to complete it again.
  /// [resultOrListeners] holds listeners.
  static const int _statePendingComplete = 1;

  /// The future has been chained to another future. The result of that
  /// other future becomes the result of this future as well.
  /// [resultOrListeners] contains the source future.
  static const int _stateChained = 2;

  /// The future has been completed with a value result.
  static const int _stateValue = 4;

  /// The future has been completed with an error result.
  static const int _stateError = 8;

  /** Whether the future is complete, and as what. */
  int _state = _stateIncomplete;

  /**
   * Zone that the future was completed from.
   * This is the zone that an error result belongs to.
   *
   * Until the future is completed, the field may hold the zone that
   * listener callbacks used to create this future should be run in.
   */
  final Zone _zone;

  /**
   * Either the result, a list of listeners or another future.
   *
   * The result of the future is either a value or an error.
   * A result is only stored when the future has completed.
   *
   * The listeners is an internally linked list of [_FutureListener]s.
   * Listeners are only remembered while the future is not yet complete,
   * and it is not chained to another future.
   *
   * The future is another future that his future is chained to. This future
   * is waiting for the other future to complete, and when it does, this future
   * will complete with the same result.
   * All listeners are forwarded to the other future.
   */
  var _resultOrListeners;

  // This constructor is used by async/await.
  _Future() : _zone = Zone.current;

  _Future.immediate(FutureOr<T> result) : _zone = Zone.current {
    _asyncComplete(result);
  }

  /** Creates a future with the value and the specified zone. */
  _Future.zoneValue(T value, this._zone) {
    _setValue(value);
  }

  _Future.immediateError(var error, [StackTrace stackTrace])
      : _zone = Zone.current {
    _asyncCompleteError(error, stackTrace);
  }

  /** Creates a future that is already completed with the value. */
  _Future.value(T value) : this.zoneValue(value, Zone.current);

  bool get _mayComplete => _state == _stateIncomplete;
  bool get _isPendingComplete => _state == _statePendingComplete;
  bool get _mayAddListener => _state <= _statePendingComplete;
  bool get _isChained => _state == _stateChained;
  bool get _isComplete => _state >= _stateValue;
  bool get _hasError => _state == _stateError;

  static List<Function> _continuationFunctions(_Future<Object> future) {
    List<Function> result = null;
    while (true) {
      if (future._mayAddListener) return result;
      assert(!future._isComplete);
      assert(!future._isChained);
      // So _resultOrListeners contains listeners.
      _FutureListener<Object, Object> listener = future._resultOrListeners;
      if (listener != null &&
          listener._nextListener == null &&
          listener.isAwait) {
        (result ??= <Function>[]).add(listener.handleValue);
        future = listener.result;
        assert(!future._isComplete);
      } else {
        break;
      }
    }
    return result;
  }

  void _setChained(_Future source) {
    assert(_mayAddListener);
    _state = _stateChained;
    _resultOrListeners = source;
  }

  Future<R> then<R>(FutureOr<R> f(T value), {Function onError}) {
    Zone currentZone = Zone.current;
    if (!identical(currentZone, _rootZone)) {
      f = currentZone.registerUnaryCallback<FutureOr<R>, T>(f);
      if (onError != null) {
        // In checked mode, this checks that onError is assignable to one of:
        //   dynamic Function(Object)
        //   dynamic Function(Object, StackTrace)
        onError = _registerErrorHandler(onError, currentZone);
      }
    }
    _Future<R> result = new _Future<R>();
    _addListener(new _FutureListener<T, R>.then(result, f, onError));
    return result;
  }

  /// Registers a system created result and error continuation.
  ///
  /// Used by the implementation of `await` to listen to a future.
  /// The system created liseners are not registered in the zone,
  /// and the listener is marked as being from an `await`.
  /// This marker is used in [_continuationFunctions].
  Future<E> _thenAwait<E>(
      FutureOr<E> f(T value), Function onError) {
    _Future<E> result = new _Future<E>();
    _addListener(new _FutureListener<T, E>.thenAwait(result, f, onError));
    return result;
  }

  Future<T> catchError(Function onError, {bool test(error)}) {
    _Future<T> result = new _Future<T>();
    if (!identical(result._zone, _rootZone)) {
      onError = _registerErrorHandler(onError, result._zone);
      if (test != null) test = result._zone.registerUnaryCallback(test);
    }
    _addListener(new _FutureListener<T, T>.catchError(result, onError, test));
    return result;
  }

  Future<T> whenComplete(dynamic action()) {
    _Future<T> result = new _Future<T>();
    if (!identical(result._zone, _rootZone)) {
      action = result._zone.registerCallback<dynamic>(action);
    }
    _addListener(new _FutureListener<T, T>.whenComplete(result, action));
    return result;
  }

  Stream<T> asStream() => new Stream<T>.fromFuture(this);

  void _setPendingComplete() {
    assert(_mayComplete);
    _state = _statePendingComplete;
  }

  void _clearPendingComplete() {
    assert(_isPendingComplete);
    _state = _stateIncomplete;
  }

  AsyncError get _error {
    assert(_hasError);
    return _resultOrListeners;
  }

  _Future get _chainSource {
    assert(_isChained);
    return _resultOrListeners;
  }

  // This method is used by async/await.
  void _setValue(T value) {
    assert(!_isComplete); // But may have a completion pending.
    _state = _stateValue;
    _resultOrListeners = value;
  }

  void _setErrorObject(AsyncError error) {
    assert(!_isComplete); // But may have a completion pending.
    _state = _stateError;
    _resultOrListeners = error;
  }

  void _setError(Object error, StackTrace stackTrace) {
    _setErrorObject(new AsyncError(error, stackTrace));
  }

  /// Copy the completion result of [source] into this future.
  ///
  /// Used when a chained future notices that its source is completed.
  void _cloneResult(_Future source) {
    assert(!_isComplete);
    assert(source._isComplete);
    _state = source._state;
    _resultOrListeners = source._resultOrListeners;
  }

  void _addListener(_FutureListener listener) {
    assert(listener._nextListener == null);
    if (_mayAddListener) {
      listener._nextListener = _resultOrListeners;
      _resultOrListeners = listener;
    } else {
      if (_isChained) {
        // Delegate listeners to chained source future.
        // If the source is complete, instead copy its values and
        // drop the chaining.
        _Future source = _chainSource;
        if (!source._isComplete) {
          source._addListener(listener);
          return;
        }
        _cloneResult(source);
      }
      assert(_isComplete);
      // Handle late listeners asynchronously.
      _zone.scheduleMicrotask(() {
        _propagateToListeners(this, listener);
      });
    }
  }

  void _prependListeners(_FutureListener listeners) {
    if (listeners == null) return;
    if (_mayAddListener) {
      _FutureListener existingListeners = _resultOrListeners;
      _resultOrListeners = listeners;
      if (existingListeners != null) {
        _FutureListener cursor = listeners;
        while (cursor._nextListener != null) {
          cursor = cursor._nextListener;
        }
        cursor._nextListener = existingListeners;
      }
    } else {
      if (_isChained) {
        // Delegate listeners to chained source future.
        // If the source is complete, instead copy its values and
        // drop the chaining.
        _Future source = _chainSource;
        if (!source._isComplete) {
          source._prependListeners(listeners);
          return;
        }
        _cloneResult(source);
      }
      assert(_isComplete);
      listeners = _reverseListeners(listeners);
      _zone.scheduleMicrotask(() {
        _propagateToListeners(this, listeners);
      });
    }
  }

  _FutureListener _removeListeners() {
    // Reverse listeners before returning them, so the resulting list is in
    // subscription order.
    assert(!_isComplete);
    _FutureListener current = _resultOrListeners;
    _resultOrListeners = null;
    return _reverseListeners(current);
  }

  _FutureListener _reverseListeners(_FutureListener listeners) {
    _FutureListener prev;
    _FutureListener current = listeners;
    while (current != null) {
      _FutureListener next = current._nextListener;
      current._nextListener = prev;
      prev = current;
      current = next;
    }
    return prev;
  }

  // Take the value (when completed) of source and complete target with that
  // value (or error). This function could chain all Futures, but is slower
  // for _Future than _chainCoreFuture, so you must use _chainCoreFuture
  // in that case.
  static void _chainForeignFuture(Future source, _Future target) {
    assert(!target._isComplete);
    assert(source is! _Future);

    // Mark the target as chained (and as such half-completed).
    target._setPendingComplete();
    try {
      source.then((value) {
        assert(target._isPendingComplete);
        // The "value" may be another future if the foreign future
        // implementation is mis-behaving,
        // so use _complete instead of _completeWithValue.
        target._clearPendingComplete(); // Clear this first, it's set again.
        target._complete(value);
      },
          // TODO(floitsch): eventually we would like to make this non-optional
          // and dependent on the listeners of the target future. If none of
          // the target future's listeners want to have the stack trace we don't
          // need a trace.
          onError: (error, [StackTrace stackTrace]) {
        assert(target._isPendingComplete);
        target._completeError(error, stackTrace);
      });
    } catch (e, s) {
      // This only happens if the `then` call threw synchronously when given
      // valid arguments.
      // That requires a non-conforming implementation of the Future interface,
      // which should, hopefully, never happen.
      scheduleMicrotask(() {
        target._completeError(e, s);
      });
    }
  }

  // Take the value (when completed) of source and complete target with that
  // value (or error). This function expects that source is a _Future.
  static void _chainCoreFuture(_Future source, _Future target) {
    assert(target._mayAddListener); // Not completed, not already chained.
    while (source._isChained) {
      source = source._chainSource;
    }
    if (source._isComplete) {
      _FutureListener listeners = target._removeListeners();
      target._cloneResult(source);
      _propagateToListeners(target, listeners);
    } else {
      _FutureListener listeners = target._resultOrListeners;
      target._setChained(source);
      source._prependListeners(listeners);
    }
  }

  void _complete(FutureOr<T> value) {
    assert(!_isComplete);
    if (value is Future<T>) {
      if (value is _Future<T>) {
        _chainCoreFuture(value, this);
      } else {
        _chainForeignFuture(value, this);
      }
    } else {
      _FutureListener listeners = _removeListeners();
      _setValue(value);
      _propagateToListeners(this, listeners);
    }
  }

  void _completeWithValue(T value) {
    assert(!_isComplete);
    assert(value is! Future<T>);

    _FutureListener listeners = _removeListeners();
    _setValue(value);
    _propagateToListeners(this, listeners);
  }

  void _completeError(Object error, [StackTrace stackTrace]) {
    assert(!_isComplete);

    _FutureListener listeners = _removeListeners();
    _setError(error, stackTrace);
    _propagateToListeners(this, listeners);
  }

  void _asyncComplete(FutureOr<T> value) {
    assert(!_isComplete);
    // Two corner cases if the value is a future:
    //   1. the future is already completed and an error.
    //   2. the future is not yet completed but might become an error.
    // The first case means that we must not immediately complete the Future,
    // as our code would immediately start propagating the error without
    // giving the time to install error-handlers.
    // However the second case requires us to deal with the value immediately.
    // Otherwise the value could complete with an error and report an
    // unhandled error, even though we know we are already going to listen to
    // it.

    if (value is Future<T>) {
      _chainFuture(value);
      return;
    }
    _setPendingComplete();
    _zone.scheduleMicrotask(() {
      _completeWithValue(value);
    });
  }

  void _chainFuture(Future<T> value) {
    if (value is _Future<T>) {
      if (value._hasError) {
        // Delay completion to allow the user to register callbacks.
        _setPendingComplete();
        _zone.scheduleMicrotask(() {
          _chainCoreFuture(value, this);
        });
      } else {
        _chainCoreFuture(value, this);
      }
      return;
    }
    // Just listen on the foreign future. This guarantees an async delay.
    _chainForeignFuture(value, this);
  }

  void _asyncCompleteError(error, StackTrace stackTrace) {
    assert(!_isComplete);

    _setPendingComplete();
    _zone.scheduleMicrotask(() {
      _completeError(error, stackTrace);
    });
  }

  /**
   * Propagates the value/error of [source] to its [listeners], executing the
   * listeners' callbacks.
   */
  static void _propagateToListeners(_Future source, _FutureListener listeners) {
    while (true) {
      assert(source._isComplete);
      bool hasError = source._hasError;
      if (listeners == null) {
        if (hasError) {
          AsyncError asyncError = source._error;
          source._zone
              .handleUncaughtError(asyncError.error, asyncError.stackTrace);
        }
        return;
      }
      // Usually futures only have one listener. If they have several, we
      // call handle them separately in recursive calls, continuing
      // here only when there is only one listener left.
      while (listeners._nextListener != null) {
        _FutureListener listener = listeners;
        listeners = listener._nextListener;
        listener._nextListener = null;
        _propagateToListeners(source, listener);
      }
      _FutureListener listener = listeners;
      final sourceResult = source._resultOrListeners;
      // Do the actual propagation.
      // Set initial state of listenerHasError and listenerValueOrError. These
      // variables are updated with the outcome of potential callbacks.
      // Non-error results, including futures, are stored in
      // listenerValueOrError and listenerHasError is set to false. Errors
      // are stored in listenerValueOrError as an [AsyncError] and
      // listenerHasError is set to true.
      bool listenerHasError = hasError;
      var listenerValueOrError = sourceResult;

      // Only if we either have an error or callbacks, go into this, somewhat
      // expensive, branch. Here we'll enter/leave the zone. Many futures
      // don't have callbacks, so this is a significant optimization.
      if (hasError || listener.handlesValue || listener.handlesComplete) {
        Zone zone = listener._zone;
        if (hasError && !source._zone.inSameErrorZone(zone)) {
          // Don't cross zone boundaries with errors.
          AsyncError asyncError = source._error;
          source._zone
              .handleUncaughtError(asyncError.error, asyncError.stackTrace);
          return;
        }

        Zone oldZone;
        if (!identical(Zone.current, zone)) {
          // Change zone if it's not current.
          oldZone = Zone._enter(zone);
        }

        // These callbacks are abstracted to isolate the try/catch blocks
        // from the rest of the code to work around a V8 glass jaw.
        void handleWhenCompleteCallback() {
          // The whenComplete-handler is not combined with normal value/error
          // handling. This means at most one handleX method is called per
          // listener.
          assert(!listener.handlesValue);
          assert(!listener.handlesError);
          var completeResult;
          try {
            completeResult = listener.handleWhenComplete();
          } catch (e, s) {
            if (hasError && identical(source._error.error, e)) {
              listenerValueOrError = source._error;
            } else {
              listenerValueOrError = new AsyncError(e, s);
            }
            listenerHasError = true;
            return;
          }
          if (completeResult is Future) {
            if (completeResult is _Future && completeResult._isComplete) {
              if (completeResult._hasError) {
                listenerValueOrError = completeResult._error;
                listenerHasError = true;
              }
              // Otherwise use the existing result of source.
              return;
            }
            // We have to wait for the completeResult future to complete
            // before knowing if it's an error or we should use the result
            // of source.
            var originalSource = source;
            listenerValueOrError = completeResult.then((_) => originalSource);
            listenerHasError = false;
          }
        }

        void handleValueCallback() {
          try {
            listenerValueOrError = listener.handleValue(sourceResult);
          } catch (e, s) {
            listenerValueOrError = new AsyncError(e, s);
            listenerHasError = true;
          }
        }

        void handleError() {
          try {
            AsyncError asyncError = source._error;
            if (listener.matchesErrorTest(asyncError) &&
                listener.hasErrorCallback) {
              listenerValueOrError = listener.handleError(asyncError);
              listenerHasError = false;
            }
          } catch (e, s) {
            if (identical(source._error.error, e)) {
              listenerValueOrError = source._error;
            } else {
              listenerValueOrError = new AsyncError(e, s);
            }
            listenerHasError = true;
          }
        }

        if (listener.handlesComplete) {
          handleWhenCompleteCallback();
        } else if (!hasError) {
          if (listener.handlesValue) {
            handleValueCallback();
          }
        } else {
          if (listener.handlesError) {
            handleError();
          }
        }

        // If we changed zone, oldZone will not be null.
        if (oldZone != null) Zone._leave(oldZone);

        // If the listener's value is a future we need to chain it. Note that
        // this can only happen if there is a callback.
        if (listenerValueOrError is Future) {
          Future chainSource = listenerValueOrError;
          // Shortcut if the chain-source is already completed. Just continue
          // the loop.
          _Future result = listener.result;
          if (chainSource is _Future) {
            if (chainSource._isComplete) {
              listeners = result._removeListeners();
              result._cloneResult(chainSource);
              source = chainSource;
              continue;
            } else {
              _chainCoreFuture(chainSource, result);
            }
          } else {
            _chainForeignFuture(chainSource, result);
          }
          return;
        }
      }
      _Future result = listener.result;
      listeners = result._removeListeners();
      if (!listenerHasError) {
        result._setValue(listenerValueOrError);
      } else {
        AsyncError asyncError = listenerValueOrError;
        result._setErrorObject(asyncError);
      }
      // Prepare for next round.
      source = result;
    }
  }

  Future<T> timeout(Duration timeLimit, {FutureOr<T> onTimeout()}) {
    if (_isComplete) return new _Future.immediate(this);
    _Future<T> result = new _Future<T>();
    Timer timer;
    if (onTimeout == null) {
      timer = new Timer(timeLimit, () {
        result._completeError(
            new TimeoutException("Future not completed", timeLimit));
      });
    } else {
      Zone zone = Zone.current;
      onTimeout = zone.registerCallback(onTimeout);
      timer = new Timer(timeLimit, () {
        try {
          result._complete(zone.run(onTimeout));
        } catch (e, s) {
          result._completeError(e, s);
        }
      });
    }
    this.then((T v) {
      if (timer.isActive) {
        timer.cancel();
        result._completeWithValue(v);
      }
    }, onError: (e, StackTrace s) {
      if (timer.isActive) {
        timer.cancel();
        result._completeError(e, s);
      }
    });
    return result;
  }
}
```



## Completer

```dart
/**
 * A way to produce Future objects and to complete them later
 * with a value or error.
 *
 * Most of the time, the simplest way to create a future is to just use
 * one of the [Future] constructors to capture the result of a single
 * asynchronous computation:
 * ```
 * new Future(() { doSomething(); return result; });
 * ```
 * or, if the future represents the result of a sequence of asynchronous
 * computations, they can be chained using [Future.then] or similar functions
 * on [Future]:
 * ```
 * Future doStuff(){
 *   return someAsyncOperation().then((result) {
 *     return someOtherAsyncOperation(result);
 *   });
 * }
 * ```
 * If you do need to create a Future from scratch — for example,
 * when you're converting a callback-based API into a Future-based
 * one — you can use a Completer as follows:
 * ```
 * class AsyncOperation {
 *   Completer _completer = new Completer();
 *
 *   Future<T> doOperation() {
 *     _startOperation();
 *     return _completer.future; // Send future object back to client.
 *   }
 *
 *   // Something calls this when the value is ready.
 *   void _finishOperation(T result) {
 *     _completer.complete(result);
 *   }
 *
 *   // If something goes wrong, call this.
 *   void _errorHappened(error) {
 *     _completer.completeError(error);
 *   }
 * }
 * ```
 */
abstract class Completer<T> {
  /**
   * Creates a new completer.
   *
   * The general workflow for creating a new future is to 1) create a
   * new completer, 2) hand out its future, and, at a later point, 3) invoke
   * either [complete] or [completeError].
   *
   * The completer completes the future asynchronously. That means that
   * callbacks registered on the future are not called immediately when
   * [complete] or [completeError] is called. Instead the callbacks are
   * delayed until a later microtask.
   *
   * Example:
   * ```
   * var completer = new Completer();
   * handOut(completer.future);
   * later: {
   *   completer.complete('completion value');
   * }
   * ```
   */
  factory Completer() => new _AsyncCompleter<T>();

  /**
   * Completes the future synchronously.
   *
   * This constructor should be avoided unless the completion of the future is
   * known to be the final result of another asynchronous operation. If in doubt
   * use the default [Completer] constructor.
   *
   * Using an normal, asynchronous, completer will never give the wrong
   * behavior, but using a synchronous completer incorrectly can cause
   * otherwise correct programs to break.
   *
   * A synchronous completer is only intended for optimizing event
   * propagation when one asynchronous event immediately triggers another.
   * It should not be used unless the calls to [complete] and [completeError]
   * are guaranteed to occur in places where it won't break `Future` invariants.
   *
   * Completing synchronously means that the completer's future will be
   * completed immediately when calling the [complete] or [completeError]
   * method on a synchronous completer, which also calls any callbacks
   * registered on that future.
   *
   * Completing synchronously must not break the rule that when you add a
   * callback on a future, that callback must not be called until the code
   * that added the callback has completed.
   * For that reason, a synchronous completion must only occur at the very end
   * (in "tail position") of another synchronous event,
   * because at that point, completing the future immediately is be equivalent
   * to returning to the event loop and completing the future in the next
   * microtask.
   *
   * Example:
   *
   *     var completer = new Completer.sync();
   *     // The completion is the result of the asynchronous onDone event.
   *     // No other operation is performed after the completion. It is safe
   *     // to use the Completer.sync constructor.
   *     stream.listen(print, onDone: () { completer.complete("done"); });
   *
   * Bad example. Do not use this code. Only for illustrative purposes:
   *
   *     var completer = new Completer.sync();
   *     completer.future.then((_) { bar(); });
   *     // The completion is the result of the asynchronous onDone event.
   *     // However, there is still code executed after the completion. This
   *     // operation is *not* safe.
   *     stream.listen(print, onDone: () {
   *       completer.complete("done");
   *       foo();  // In this case, foo() runs after bar().
   *     });
   */
  factory Completer.sync() => new _SyncCompleter<T>();

  /**
   * The future that is completed by this completer.
   *
   * The future that is completed when [complete] or [completeError] is called.
   */
  Future<T> get future;

  /**
   * Completes [future] with the supplied values.
   *
   * The value must be either a value of type [T]
   * or a future of type `Future<T>`.
   *
   * If the value is itself a future, the completer will wait for that future
   * to complete, and complete with the same result, whether it is a success
   * or an error.
   *
   * Calling [complete] or [completeError] must be done at most once.
   *
   * All listeners on the future are informed about the value.
   */
  void complete([FutureOr<T> value]);

  /**
   * Complete [future] with an error.
   *
   * Calling [complete] or [completeError] must be done at most once.
   *
   * Completing a future with an error indicates that an exception was thrown
   * while trying to produce a value.
   *
   * If [error] is `null`, it is replaced by a [NullThrownError].
   *
   * If `error` is a `Future`, the future itself is used as the error value.
   * If you want to complete with the result of the future, you can use:
   * ```
   * thisCompleter.complete(theFuture)
   * ```
   * or if you only want to handle an error from the future:
   * ```
   * theFuture.catchError(thisCompleter.completeError);
   * ```
   */
  void completeError(Object error, [StackTrace stackTrace]);

  /**
   * Whether the [future] has been completed.
   *
   * Reflects whether [complete] or [completeError] has been called.
   * A `true` value doesn't necessarily mean that listeners of this future
   * have been invoked yet, either because the completer usually waits until
   * a later microtask to propagate the result, or because [complete]
   * was called with a future that hasn't completed yet.
   *
   * When this value is `true`, [complete] and [completeError] must not be
   * called again.
   */
  bool get isCompleted;
}
```



## _Completer

```dart
abstract class _Completer<T> implements Completer<T> {
  final _Future<T> future = new _Future<T>();

  void complete([FutureOr<T> value]);

  void completeError(Object error, [StackTrace stackTrace]) {
    error = _nonNullError(error);
    if (!future._mayComplete) throw new StateError("Future already completed");
    AsyncError replacement = Zone.current.errorCallback(error, stackTrace);
    if (replacement != null) {
      error = _nonNullError(replacement.error);
      stackTrace = replacement.stackTrace;
    }
    _completeError(error, stackTrace);
  }

  void _completeError(Object error, StackTrace stackTrace);

  // The future's _isComplete doesn't take into account pending completions.
  // We therefore use _mayComplete.
  bool get isCompleted => !future._mayComplete;
}
```


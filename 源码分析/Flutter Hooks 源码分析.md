# Flutter Hooks 源码分析



## framework

```dart
/// 不可变的 hook 配置类
@immutable
abstract class Hook<R> {
  /// Allows subclasses to have a `const` constructor
  const Hook({this.keys});

  /// 注册一个 [Hook] 并返回它持有的值
  /// [use] 只能在 [HookWidget.build] 时被调用，且需要被无条件的调用
  static R use<R>(Hook<R> hook) {
    return HookElement._currentContext._use(hook);
  }

  /// 一组用于决定 [HookState] 是否应该被重用或者重新创建的 的对象
  /// 当 [Hook] 被创建，框架调用 [Hook.shouldPreserveState] 来检查这个数组是否相等
  /// 如果他们不相等，原先的 [HookState] 会被销毁, 新的对象会通过调用 [Hook.createState] 来创建
  /// 紧接着会调用 [HookState.initHook].
  final List<Object> keys;

  /// 决定是否销毁或者重用对象的算法：
  /// 判断方式：
  /// 1. 检查数组引用： `hook1.keys == hook2.keys` (通常他们是 immutable 的)
  /// 2. 检查内容变化： If there's any difference in the content of [Hook.keys], using 
  /// `operator==` (通过 == 来判断)
  static bool shouldPreserveState(Hook hook1, Hook hook2) {
    final p1 = hook1.keys;
    final p2 = hook2.keys;
    /// 引用相同，重用 
    if (p1 == p2) {
      return true;
    }
    /// 其中一个为 null 或者 长度发生了改变，重建 
    if ((p1 != p2 && (p1 == null || p2 == null)) || p1.length != p2.length) {
      return false;
    }

    /// 内容相同，重用，否则重建
    var i1 = p1.iterator;
    var i2 = p2.iterator;
    while (true) {
      if (!i1.moveNext() || !i2.moveNext()) {
        return true;
      }
      if (i1.current != i2.current) {
        return false;
      }
    }
  }

  /// 新建一个与 hook 关联的 hookState
  /// 子类应该重载这个方法来创建与之关联的 HookState
  /// The framework can call this method multiple times over the lifetime of a [HookWidget]. For example,
  /// if the hook is used multiple times, a separate [HookState] must be created for each usage.
  @protected
  HookState<R, Hook<R>> createState();
}

/// [HookWidget] 的内部逻辑
abstract class HookState<R, T extends Hook<R>> {
  /// Equivalent of [State.context] for [HookState]
  @protected
  BuildContext get context => _element.context;
  State _element;

  /// 等价于 [State.widget]
  T get hook => _hook;
  T _hook;

  /// 等价于 [State.initState]
  @protected
  void initHook() {}

  /// 等价于 [State.dispose]
  @protected
  void dispose() {}

  /// 在 [HookWidget.build] 完成之后立即调用
  @protected
  void didBuild() {}

  /// 当 [HookState] 被请求时调用
  ///
  /// [build] is where an [HookState] may use other hooks. This restriction is made to ensure that hooks are unconditionally always requested
  @protected
  R build(BuildContext context);

  /// 等价于 [State.didUpdateWidget]
  @protected
  void didUpdateHook(T oldHook) {}

  /// In addition to this method being invoked, it is guaranteed that the
  /// [build] method will be invoked when a reassemble is signaled. Most
  /// widgets therefore do not need to do anything in the [reassemble] method.
  ///
  /// See also:
  ///
  ///  * [State.reassemble]
  void reassemble() {}

  /// 等价于 [State.setState]
  @protected
  void setState(VoidCallback fn) { // void setState([VoidCallback fn])
    /// 这个方法直接调用 所持有的 State.setState方法（setState方法是 @protected)  
    /// 会触发 widget 状态刷新，进而刷新 UI  
    // ignore: invalid_use_of_protected_member
    _element.setState(fn);
  }
}

/// 将 [HookWidget] 作为其配置的 [Element]
class HookElement extends StatefulElement {
  /// 一个对于所有的 [HookElement] 公用的域 
  /// 在 [HookWidget.build] 时赋值，值为其对应的 elmenet
  /// 在 [HookWidget.build] 完成后被置 null，这限制了 [Hook.use] 的使用范围
  static HookElement _currentContext;

  HookElement(HookWidget widget) : super(widget);

  /// 用于操作 HookState 的内部域（指针？？） 
  Iterator<HookState> _currentHook;
  int _hookIndex;
    
  /// 每个 [HookElement] 都持有一个 HookState 数组
  /// 通过维护这个数组来提供数据，刷新数据，进而影响UI
  List<HookState> _hooks;
  bool _didFinishBuildOnce = false;

  bool _debugDidReassemble;
  bool _debugShouldDispose;
  bool _debugIsInitHook;

  @override
  HookWidget get widget => super.widget as HookWidget;

  /// 重载 StatefulElement.build 方法  
  /// 这个方法实际上调用了 HookWidget.build  
  @override
  Widget build() {
    /// 初始化迭代器和索引  
    /// 在首次 build 时，_hooks 是 null，直到第一次调用 Hook.use方法
    _currentHook = _hooks?.iterator;
    _currentHook?.moveNext();
    _hookIndex = 0;
      
    /// 设置 Buildcontext  
    HookElement._currentContext = this; 
      
    /// 在 StatfulElement 中  
    /// Widget build() => state.build(this); 
    /// 在 HookWidgetState 中  
    /// Widget build(BuildContext context) => widget.build(context); 
    /// 而在 Widget.build 中，我们使用了 useHook，进而调用了到 [_use] 方法  
    final result = super.build();
      
    /// 设置 Buildcontext  
    HookElement._currentContext = null;

    /// 时候已经首次构建
    _didFinishBuildOnce = true;
    return result;
  }

  /// A read-only list of all hooks available.
  ///
  /// These should not be used directly and are exposed
  @visibleForTesting
  List<HookState> get debugHooks => List<HookState>.unmodifiable(_hooks);

  @override
  InheritedWidget inheritFromWidgetOfExactType(
      Type targetType,
      {Object aspect}) 
  {
    return super.inheritFromWidgetOfExactType(targetType, aspect: aspect);
  }

  /// buildScope 会调用标记了 dirty 节点的 rebuild 方法，然后是 performRebuild 方法  
  /// ComponentElement 对 performRebuild 的实现调用了 build
  /// 在 build 之后会立刻调用 updateChild，这时候会调用到 hook 的 didBuild  
  /// try {
  ///    built = build();
  /// }
  /// ...
  /// try {
  ///    _child = updateChild(_child, built, slot);
  /// }
  @override
  Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
    if (_hooks != null) {
      for (final hook in _hooks.reversed) {
        try {
          hook.didBuild();
        } catch (exception, stack) {
          ...
        }
      }
    }
    return super.updateChild(child, newWidget, newSlot);
  }

  /// 在 element 的时候调用 hook.dispose(); 
  @override
  void unmount() {
    super.unmount();
    if (_hooks != null) {
      for (final hook in _hooks) {
        try {
          hook.dispose();
        } catch (exception, stack) {
          ...
        }
      }
    }
  }

  /// 在 element 的时候调用 hook.reassemble();   
  @override
  void reassemble() {
    super.reassemble();
    /// hook.reassemble 只在 debug 模式有效  
    assert(() {
      _debugDidReassemble = true;
      if (_hooks != null) {
        for (final hook in _hooks) {
          hook.reassemble();
        }
      }
      return true;
    }());
  }

  /// 当调用 [Hook.use] 时，会调用到 [HookElement._currentContext._use]
  /// 这限制了 use 的使用范围  
  R _use<R>(Hook<R> hook) {
    HookState<R, Hook<R>> hookState;
    /// 首次构建 Widget，_hooks 为 null，所以 _currentHook == null  
    if (_currentHook == null) {
      hookState = _createHookState(hook);
      /// 初始化 _hooks  
      _hooks ??= [];  
      /// 每次调用 Hook.use 会添加一个 State  
      _hooks.add(hookState);
    } else {
      if (!_didFinishBuildOnce && _currentHook.current == null) {
        hookState = _pushHook(hook);
        _currentHook.moveNext();
      } else {
		
        /// 引用相等，重用 state
        if (_currentHook.current.hook == hook) {
          hookState = _currentHook.current as HookState<R, Hook<R>>;
          _currentHook.moveNext();
        /// 应该保存状态，重用 state    
        } else if (Hook.shouldPreserveState(_currentHook.current.hook, hook)) {
          hookState = _currentHook.current as HookState<R, Hook<R>>;
          _currentHook.moveNext();
          final previousHook = hookState._hook;
          /// 只更新 _hook，并调用 didUpdateHook
          hookState
            .._hook = hook
            ..didUpdateHook(previousHook);
        /// 否则，使用新的 hook 来创建 State，并替换原有的 State    
        } else {
          hookState = _replaceHookAt(_hookIndex, hook);
          _resetsIterator(hookState);
          _currentHook.moveNext();
        }
      }
    }
    _hookIndex++;
    return hookState.build(this);
  }

  /// 用新的 hook 替换指定 index 的 hookState 
  HookState<R, Hook<R>> _replaceHookAt<R>(int index, Hook<R> hook) {
    _hooks.removeAt(_hookIndex).dispose();
    var hookState = _createHookState(hook);
    _hooks.insert(_hookIndex, hookState);
    return hookState;
  }

  /// 用新的 hook 在 index 位置插入 的 hookState   
  HookState<R, Hook<R>> _insertHookAt<R>(int index, Hook<R> hook) {
    var hookState = _createHookState(hook);
    _hooks.insert(index, hookState);
    _resetsIterator(hookState);
    return hookState;
  }

  /// push 一个 新的 hook 
  HookState<R, Hook<R>> _pushHook<R>(Hook<R> hook) {
    var hookState = _createHookState(hook);
    _hooks.add(hookState);
    _resetsIterator(hookState);
    return hookState;
  }

  /// 添加一个 State 时需要重置 iterator
  void _resetsIterator(HookState hookState) {
    _currentHook = _hooks.iterator;
    while (_currentHook.current != hookState) {
      _currentHook.moveNext();
    }
  }

  /// 这个方法调用 [hook.createState()] 来创建 HookState，并初始化它的域 
  HookState<R, Hook<R>> _createHookState<R>(Hook<R> hook) {
    final state = hook.createState()
      .._element = this.state
      .._hook = hook
      ..initHook();
    return state;
  }
}

/// 能够使用 [Hook] 的 [Widget]
abstract class HookWidget extends StatefulWidget {
  /// Initializes [key] for subclasses.
  const HookWidget({Key key}) : super(key: key);

  @override
  HookElement createElement() => HookElement(this);

  @override
  _HookWidgetState createState() => _HookWidgetState();

  @protected
  Widget build(BuildContext context);
}

class _HookWidgetState extends State<HookWidget> {
  @override
  Widget build(BuildContext context) {
    return widget.build(context);
  }
}

/// 获取 BuildContext
BuildContext useContext() {
  return HookElement._currentContext;
}

```



## misc

```dart
part of 'hooks.dart';

/// A state holder that allows mutations by dispatching actions.
abstract class Store<State, Action> {
  /// The current state.
  ///
  /// This value may change after a call to [dispatch].
  State get state;

  /// Dispatches an action.
  ///
  /// Actions are dispatched synchronously.
  /// It is impossible to try to dispatch actions during [HookWidget.build].
  void dispatch(Action action);
}

/// Composes an [Action] and a [State] to create a new [State].
///
/// [Reducer] must never return `null`, even if [state] or [action] are `null`.
typedef Reducer<State, Action> = State Function(State state, Action action);

/// An alternative to [useState] for more complex states.
///
/// [useReducer] manages an read only state that can be updated
/// by dispatching actions which are interpreted by a [Reducer].
///
/// [reducer] is immediatly called on first build with [initialAction]
/// and [initialState] as parameter.
///
/// It is possible to change the [reducer] by calling [useReducer]
///  with a new [Reducer].
///
/// See also:
///  * [Reducer]
///  * [Store]
Store<State, Action> useReducer<State extends Object, Action>(
  Reducer<State, Action> reducer, {
  State initialState,
  Action initialAction,
}) {
  return Hook.use(_ReducerdHook(reducer,
      initialAction: initialAction, initialState: initialState));
}

class _ReducerdHook<State, Action> extends Hook<Store<State, Action>> {
  final Reducer<State, Action> reducer;
  final State initialState;
  final Action initialAction;

  const _ReducerdHook(this.reducer, {this.initialState, this.initialAction})
      : assert(reducer != null);

  @override
  _ReducerdHookState<State, Action> createState() =>
      _ReducerdHookState<State, Action>();
}

class _ReducerdHookState<State, Action>
    extends HookState<Store<State, Action>, _ReducerdHook<State, Action>>
    implements Store<State, Action> {
  @override
  State state;

  @override
  void initHook() {
    super.initHook();
    state = hook.reducer(hook.initialState, hook.initialAction);
    assert(state != null);
  }

  @override
  void dispatch(Action action) {
    final res = hook.reducer(state, action);
    assert(res != null);
    if (state != res) {
      setState(() {
        state = res;
      });
    }
  }

  @override
  Store<State, Action> build(BuildContext context) {
    return this;
  }
}

/// Returns the previous argument called to [usePrevious].
T usePrevious<T>(T val) {
  return Hook.use(_PreviousHook(val));
}

class _PreviousHook<T> extends Hook<T> {
  _PreviousHook(this.value);

  final T value;

  @override
  _PreviousHookState<T> createState() => _PreviousHookState();
}

class _PreviousHookState<T> extends HookState<T, _PreviousHook<T>> {
  T previous;

  @override
  void didUpdateHook(_PreviousHook<T> old) {
    previous = old.value;
  }

  @override
  T build(BuildContext context) => previous;
}

/// Runs the callback on every hot reload
/// similar to reassemble in the Stateful widgets
///
/// See also:
///
///  * [State.reassemble]
void useReassemble(VoidCallback callback) {
  assert(() {
    Hook.use(_ReassembleHook(callback));
    return true;
  }());
}

class _ReassembleHook extends Hook<void> {
  final VoidCallback callback;

  _ReassembleHook(this.callback) : assert(callback != null);

  @override
  _ReassembleHookState createState() => _ReassembleHookState();
}

class _ReassembleHookState extends HookState<void, _ReassembleHook> {
  @override
  void reassemble() {
    super.reassemble();
    hook.callback();
  }

  @override
  void build(BuildContext context) {}
}
```



## primitives

```dart
part of 'hooks.dart';

/// Cache the instance of a complex object.
///
/// [useMemoized] will immediatly call [valueBuilder] on first call and store its result.
/// Later, when [HookWidget] rebuilds, the call to [useMemoized] will return the previously created instance without calling [valueBuilder].
///
/// A later call of [useMemoized] with different [keys] will call [useMemoized] again to create a new instance.
T useMemoized<T>(T Function() valueBuilder,
    [List<Object> keys = const <dynamic>[]]) {
  return Hook.use(_MemoizedHook(
    valueBuilder,
    keys: keys,
  ));
}

class _MemoizedHook<T> extends Hook<T> {
  final T Function() valueBuilder;

  const _MemoizedHook(this.valueBuilder,
      {List<Object> keys = const <dynamic>[]})
      : assert(valueBuilder != null),
        assert(keys != null),
        super(keys: keys);

  @override
  _MemoizedHookState<T> createState() => _MemoizedHookState<T>();
}

class _MemoizedHookState<T> extends HookState<T, _MemoizedHook<T>> {
  T value;

  @override
  void initHook() {
    super.initHook();
    value = hook.valueBuilder();
  }

  @override
  T build(BuildContext context) {
    return value;
  }
}

/// Watches a value and calls a callback whenever the value changed.
///
/// [useValueChanged] takes a [valueChange] callback and calls it whenever [value] changed.
/// [valueChange] will _not_ be called on the first [useValueChanged] call.
///
/// [useValueChanged] can also be used to interpolate
/// Whenever [useValueChanged] is called with a diffent [value], calls [valueChange].
/// The value returned by [useValueChanged] is the latest returned value of [valueChange] or `null`.
///
/// The following example calls [AnimationController.forward] whenever `color` changes
///
/// ```dart
/// AnimationController controller;
/// Color color;
///
/// useValueChanged(color, (_, __)) {
///     controller.forward();
/// });
/// ```
R useValueChanged<T, R>(T value, R valueChange(T oldValue, R oldResult)) {
  return Hook.use(_ValueChangedHook(value, valueChange));
}

class _ValueChangedHook<T, R> extends Hook<R> {
  final R Function(T oldValue, R oldResult) valueChanged;
  final T value;

  const _ValueChangedHook(this.value, this.valueChanged)
      : assert(valueChanged != null);

  @override
  _ValueChangedHookState<T, R> createState() => _ValueChangedHookState<T, R>();
}

class _ValueChangedHookState<T, R>
    extends HookState<R, _ValueChangedHook<T, R>> {
  R _result;

  @override
  void didUpdateHook(_ValueChangedHook<T, R> oldHook) {
    super.didUpdateHook(oldHook);
    if (hook.value != oldHook.value) {
      _result = hook.valueChanged(oldHook.value, _result);
    }
  }

  @override
  R build(BuildContext context) {
    return _result;
  }
}

/// Useful for side-effects and optionally canceling them.
///
/// [useEffect] is called synchronously on every [HookWidget.build], unless
typedef Dispose = void Function();

/// [keys] is specified. In which case [useEffect] is called again only if
/// any value inside [keys] as changed.
///
/// It takes an [effect] callback and calls it synchronously.
/// That [effect] may optionally return a function, which will be called when the [effect] is called again or if the widget is disposed.
///
/// By default [effect] is called on every [HookWidget.build] call, unless [keys] is specified.
/// In which case, [effect] is called once on the first [useEffect] call and whenever something within [keys] change/
///
/// The following example call [useEffect] to subscribes to a [Stream] and cancel the subscription when the widget is disposed.
/// ALso ifthe [Stream] change, it will cancel the listening on the previous [Stream] and listen to the new one.
///
/// ```dart
/// Stream stream;
/// useEffect(() {
///     final subscription = stream.listen(print);
///     // This will cancel the subscription when the widget is disposed
///     // or if the callback is called again.
///     return subscription.cancel;
///   },
///   // when the stream change, useEffect will call the callback again.
///   [stream],
/// );
/// ```
void useEffect(Dispose Function() effect, [List<Object> keys]) {
  Hook.use(_EffectHook(effect, keys));
}

class _EffectHook extends Hook<void> {
  final Dispose Function() effect;

  const _EffectHook(this.effect, [List<Object> keys])
      : assert(effect != null),
        super(keys: keys);

  @override
  _EffectHookState createState() => _EffectHookState();
}

class _EffectHookState extends HookState<void, _EffectHook> {
  Dispose disposer;

  @override
  void initHook() {
    super.initHook();
    scheduleEffect();
  }

  @override
  void didUpdateHook(_EffectHook oldHook) {
    super.didUpdateHook(oldHook);

    if (hook.keys == null) {
      if (disposer != null) {
        disposer();
      }
      scheduleEffect();
    }
  }

  @override
  void build(BuildContext context) {}

  @override
  void dispose() {
    if (disposer != null) {
      disposer();
    }
  }

  void scheduleEffect() {
    disposer = hook.effect();
  }
}

/// Create variable and subscribes to it.
///
/// Whenever [ValueNotifier.value] updates, it will mark the caller [HookWidget]
/// as needing build.
/// On first call, inits [ValueNotifier] to [initialData]. [initialData] is ignored
/// on subsequent calls.
///
/// The following example showcase a basic counter application.
///
/// ```dart
/// class Counter extends HookWidget {
///   @override
///   Widget build(BuildContext context) {
///     final counter = useState(0);
///
///     return GestureDetector(
///       // automatically triggers a rebuild of Counter widget
///       onTap: () => counter.value++,
///       child: Text(counter.value.toString()),
///     );
///   }
/// }
/// ```
///
/// See also:
///
///  * [ValueNotifier]
///  * [useStreamController], an alternative to [ValueNotifier] for state.
ValueNotifier<T> useState<T>([T initialData]) {
  return Hook.use(_StateHook(initialData: initialData));
}

class _StateHook<T> extends Hook<ValueNotifier<T>> {
  final T initialData;

  const _StateHook({this.initialData});

  @override
  _StateHookState<T> createState() => _StateHookState();
}

class _StateHookState<T> extends HookState<ValueNotifier<T>, _StateHook<T>> {
  ValueNotifier<T> _state;

  @override
  void initHook() {
    super.initHook();
    _state = ValueNotifier(hook.initialData)..addListener(_listener);
  }

  @override
  void dispose() {
    _state.dispose();
  }

  @override
  ValueNotifier<T> build(BuildContext context) {
    return _state;
  }

  void _listener() {
    setState(() {});
  }
}

```


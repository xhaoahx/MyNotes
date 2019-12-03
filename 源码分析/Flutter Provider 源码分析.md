# Flutter Provider 源码分析

[TOC]

## SingleChildCloneableWidget 

```dart
abstract class SingleChildCloneableWidget implements Widget {
  /// 用一个新[child]来克隆当前的 Provider
  ///
  /// 实现要求：包含[Key]和其他必须的值域
  SingleChildCloneableWidget cloneWithChild(Widget child);
}
```



## InheritedProvider

```dart
class InheritedProvider<T> extends InheritedWidget {
    /// 允许自定义 UpdateShouldNotify
    const InheritedProvider({
        Key key,
        @required T value,
        UpdateShouldNotify<T> updateShouldNotify,
        Widget child,
    })  : _value = value,
    _updateShouldNotify = updateShouldNotify,

    /// 向后代暴露的值，[Provider.of] 来获取这个值
    /// 避免修改这个值。如果修改是必须的，需要新建一个ProviderWidget来替换原有的Widget
    final T _value;
    final UpdateShouldNotify<T> _updateShouldNotify;

    @override
    bool updateShouldNotify(InheritedProvider<T> oldWidget) {
        if (_updateShouldNotify != null) {
            return _updateShouldNotify(oldWidget._value, _value);
        }
        return oldWidget._value != _value;
    }
}
```



## MultiProvider 

```dart
/// 方便同时部署多个 Provider
class MultiProvider extends StatelessWidget
    implements SingleChildCloneableWidget {
  const MultiProvider({
    Key key,
    @required this.providers,
    this.child,
  })  : assert(providers != null),
        super(key: key);

  /// Provider列表会被转化成自顶向下的树状结构
  ///
  /// 例如[A,B,C]会被转化为如下结构
  ///   A
  ///   |
  ///   B
  ///   |
  ///   C
  ///   |
  /// child
  final List<SingleChildCloneableWidget> providers;

  final Widget child;

  @override
  Widget build(BuildContext context) {
    var tree = child;
    /// 将provider逆置，再逐个调用 cloneWithChild 
    for (final provider in providers.reversed) {
      tree = provider.cloneWithChild(tree);
    }
    return tree;
  }

  /// 通过将先前的 MultiProvider 作为自己的子树，来实现嵌套 
  @override
  MultiProvider cloneWithChild(Widget child) {
    return MultiProvider(
      key: key,
      providers: providers,
      child: child,
    );
  }
}
```



## StateDelegate

```dart
typedef ValueBuilder<T> = T Function(BuildContext context);

/// 用于销毁[T]释放内存的函数回调
typedef Disposer<T> = void Function(BuildContext context, T value);


/// 用于控制 State 的状态
/// 持有 State 的 Context 和 SetState 方法
abstract class StateDelegate {
  BuildContext _context;

  BuildContext get context => _context;

  /// 持有 [StatefulWidget]对应的 State 的 setState 方法，刷新状态，触发重建  
  StateSetter _setState;

  /// Notify the framework that the internal state of this object has changed.
  ///
  /// See the discussion on [State.setState] for more information.
  @protected
  StateSetter get setState => _setState;

  /// 在 initState 中被调用，初始化 Delegate
  @protected
  @mustCallSuper
  void initDelegate() {}

  /// 无论何时 [State.didUpdateWidget] 被调用时，调用这个方法
  @protected
  @mustCallSuper
  void didUpdateDelegate(covariant StateDelegate old) {}

  @protected
  @mustCallSuper
  void dispose() {}
}
```



## DelegateWidget

```dart
abstract class DelegateWidget extends StatefulWidget {
  const DelegateWidget({
    Key key,
    @required this.delegate,
  });

  /// 当前 [DelegateWidget] 的 State
  @protected
  final StateDelegate delegate;

  /// 要求实现的 build 方法，这个方法被 State.build 调用
  @protected
  Widget build(BuildContext context);

  @override
  StatefulElement createElement() => _DelegateElement(this);

  @override
  _DelegateWidgetState createState() => _DelegateWidgetState();
}


class _DelegateWidgetState extends State<DelegateWidget> {
  @override
  void initState() {
    super.initState();
    _mountDelegate();
    _initDelegate();
  }

  void _initDelegate() {
    widget.delegate.initDelegate();
  }

  /// 使 Delegate 持有 setState方法 和 context    
  void _mountDelegate() {
    widget.delegate
      .._context = context
      .._setState = setState;
  }

  /// 解除 widget.delegate 持有 setState 方法和 context     
  void _unmountDelegate(StateDelegate delegate) {
    delegate
      .._context = null
      .._setState = null;
  }

  @override
  void didUpdateWidget(DelegateWidget oldWidget) {
    super.didUpdateWidget(oldWidget);
    /// 当 Widget 发生改变的时候，如果 delegate 的应用发生变化，需要解除oldWidget的delegate的引用 
    if (widget.delegate != oldWidget.delegate) {
      _mountDelegate();
      /// 更甚，当 delegate 的类型改变的时候，旧的delegate会被销毁，并初始化新的 delegate  
      if (widget.delegate.runtimeType != oldWidget.delegate.runtimeType) {
        oldWidget.delegate.dispose();
        _initDelegate();
      } else {
        widget.delegate.didUpdateDelegate(oldWidget.delegate);
      }
      _unmountDelegate(oldWidget.delegate);
    }
  }

  /// 调用widget.build方法
  @override
  Widget build(BuildContext context) => widget.build(context);

  @override
  void dispose() {
    widget.delegate.dispose();
    _unmountDelegate(widget.delegate);
    super.dispose();
  }
}

/// 对应Element
class _DelegateElement extends StatefulElement {
  _DelegateElement(DelegateWidget widget) : super(widget);

  bool _debugIsInitDelegate = false;

  @override
  DelegateWidget get widget => super.widget as DelegateWidget;

  @override
  InheritedWidget inheritFromElement(Element ancestor, {Object aspect}) {
    return super.inheritFromElement(ancestor, aspect: aspect);
  }
}
```



### ValueStateDelegate

```dart
/// 向后代提供一个常量
abstract class ValueStateDelegate<T> extends StateDelegate {
  /// value 不应该被直接修改
  T get value;
}
```



### SingleValueDelegate & BuilderStateDelegate

```dart
/// 适用于 Value 模式，储存已有的 value
class SingleValueDelegate<T> extends ValueStateDelegate<T> {
  SingleValueDelegate(this.value);
  @override
  final T value;
}

/// 适用于Builder模式，实例化value
class BuilderStateDelegate<T> extends ValueStateDelegate<T> {
    BuilderStateDelegate(
        this._builder, 
        {Disposer<T> dispose}
    ) : assert(_builder != null),
    _dispose = dispose;

    final ValueBuilder<T> _builder;
    final Disposer<T> _dispose;

    T _value;
    
    @override
    T get value => _value;

    /// _value在此时被构建
    @override
    void initDelegate() {
        super.initDelegate();
        _value = _builder(context);
    }

    @override
    void didUpdateDelegate(BuilderStateDelegate<T> old) {
        super.didUpdateDelegate(old);
        _value = old.value;
    }

    @override
    void dispose() {
        /// 等价于：
        /// if(_dispose != null) _dispose(context,value)
        _dispose?.call(context, value);
        super.dispose();
    }
}
```



## ValueDelegateWidget

```dart
/// 将 delegate 对外暴露
abstract class ValueDelegateWidget<T> extends DelegateWidget {
    ValueDelegateWidget({
        Key key,
        @required ValueStateDelegate<T> delegate,
    }) : super(key: key, delegate: delegate);

    @override
    @protected
    ValueStateDelegate<T> get delegate => super.delegate as ValueStateDelegate<T>;
}
```



## Provider

```dart
class Provider<T> 
    extends ValueDelegateWidget<T>
    implements SingleChildCloneableWidget {
    /// 创建一个持有value的provider，并将value暴露给子树，供其获取
    /// 同时，可提供一个dispose，用于释放内存
    Provider({
        Key key,
        @required ValueBuilder<T> builder,
        Disposer<T> dispose,
        Widget child,
    }) : this._(
        key: key,
        /// 采用builder模式的delegate
        delegate: BuilderStateDelegate<T>(builder, dispose: dispose),
        updateShouldNotify: null,
        child: child,
    );

    /// 通过已有的值来创建一个[Provider].
    Provider.value({
        Key key,
        @required T value,
        UpdateShouldNotify<T> updateShouldNotify,
        Widget child,
    }) : this._(
        key: key,
        /// 直接提供值的delegate
        delegate: SingleValueDelegate<T>(value),
        updateShouldNotify: updateShouldNotify,
        child: child,
    );

    Provider._({
        Key key,
        @required ValueStateDelegate<T> delegate,
        this.updateShouldNotify,
        this.child,
    }) : super(key: key, delegate: delegate);

    /// 向上查找，获取到最近的[InheritedProvider<T>]，并将其Value返回
    ///
    /// If [listen] 如果为true，之后value发生改变时将会触发widget的[State.build] 或是 
    /// StatefulWidget的[State.didChangeDependencies]
    static T of<T>(BuildContext context, {bool listen = true}) {
        // 获取到Tyep InheritedProvider<T>
        final type = _typeOf<InheritedProvider<T>>();
        final provider = listen
            ? context.inheritFromWidgetOfExactType(type) 
              as InheritedProvider<T>
            : 
            context.
            ancestorInheritedElementForWidgetOfExactType(type)?.widget
              as InheritedProvider<T>;
		/// 最容易报错的地方
        if (provider == null) {
            throw ProviderNotFoundError(T, context.widget.runtimeType);
        }

        return provider._value;
    }

    final UpdateShouldNotify<T> updateShouldNotify;

    @override
    Provider<T> cloneWithChild(Widget child) {
        return Provider._(
            key: key,
            delegate: delegate,
            updateShouldNotify: updateShouldNotify,
            child: child,
        );
    }

    final Widget child;

    /// Provider其实是一个StatefulWidget
    /// 其build方法被DelegateState.build(BuildContext context)调用
    /// （DelegateState.build(BuildContext context) => widget.build(context)）
    /// 注：这个build方法重载的是DelegateWidget的build方法，StatefulWidget并不具有build
    /// （Widget build(BuildContext context)）

    @override
    Widget build(BuildContext context) {
        return InheritedProvider<T>(
            value: delegate.value,
            updateShouldNotify: updateShouldNotify,
            child: child,
        );
    }
}
```



### 归纳

Provider >> ValueDelegateWidget >> DelegateWidget

树中关系：

   Widget树												                                Element树

​    Parent                                                                                      Parent
​	     |                                                                                                |
Provider >> DelegateWidget                     			           DelegateElement
​         |												                                                |
InheritedProvider								                              InheritedElement
​        |												                                                 |
   Child											                                                Child



## ListenableProvider 

```dart
/// 可监听 Provider
class ListenableProvider<T extends Listenable> 
    extends ValueDelegateWidget<T>
    implements SingleChildCloneableWidget 
{
    ListenableProvider({
        Key key,
        @required ValueBuilder<T> builder,
        Disposer<T> dispose,
        Widget child,
    }) : this._(
        key: key,
        delegate: _BuilderListenableDelegate(builder, dispose: dispose),
        child: child,
    );

    ListenableProvider.value({
        Key key,
        @required T value,
        Widget child,
    }) : this._(
        key: key,
        delegate: _ValueListenableDelegate(value),
        child: child,
    );

    ListenableProvider._valueDispose({
        Key key,
        @required T value,
        Disposer<T> disposer,
        Widget child,
    }) : this._(
        key: key,
        delegate: _ValueListenableDelegate(value, disposer),
        child: child,
    );

    ListenableProvider._({
        Key key,
        @required _ListenableDelegateMixin<T> delegate,
        this.child,
    }) : super(key: key, delegate: delegate);

    /// The widget that is below the current [ListenableProvider] widget in the
    /// tree.
    ///
    /// {@macro flutter.widgets.child}
    final Widget child;

    @override
    ListenableProvider<T> cloneWithChild(Widget child) {
        return ListenableProvider._(
            key: key,
            delegate: delegate as _ListenableDelegateMixin<T>,
            child: child,
        );
    }

    @override
    Widget build(BuildContext context) {
        final delegate = this.delegate as _ListenableDelegateMixin<T>;
        return InheritedProvider<T>(
            value: delegate.value,
            updateShouldNotify: delegate.updateShouldNotify,
            child: child,
        );
    }
}
```



### ValueListenableDelegate & BuilderListenableDelegate

```dart
/// 同SingleValueDelegate & BuilderStateDelegate
class _ValueListenableDelegate<T extends Listenable>
    extends SingleValueDelegate<T> 
    with _ListenableDelegateMixin<T> 
{
    _ValueListenableDelegate(T value, [this.disposer]) : super(value);

    final Disposer<T> disposer;

    @override
    void didUpdateDelegate(_ValueListenableDelegate<T> oldDelegate) {
        super.didUpdateDelegate(oldDelegate);
        if (oldDelegate.value != value) {
            _removeListener?.call();
            oldDelegate.disposer?.call(context, oldDelegate.value);
            if (value != null) startListening(value, rebuild: true);
        }
    }

    @override
    void startListening(T listenable, {bool rebuild = false}) {
        assert(disposer == null || debugCheckIsNewlyCreatedListenable(listenable));
        super.startListening(listenable, rebuild: rebuild);
    }
}

class _BuilderListenableDelegate<T extends Listenable>
    extends BuilderStateDelegate<T> 
    with _ListenableDelegateMixin<T> 
{
    _BuilderListenableDelegate(ValueBuilder<T> builder, {Disposer<T> dispose})
        : super(builder, dispose: dispose);

    @override
    void startListening(T listenable, {bool rebuild = false}) {
        assert(debugCheckIsNewlyCreatedListenable(listenable));
        super.startListening(listenable, rebuild: rebuild);
    }
}
```



### ListenableDelegateMixin

```dart
/// 将delegate.setState加入到 T (T extends<Listenable>)的监听者中
///
/// 因为InheritedProvider是StatfulWidget的孩子，所以当 T 调用notifyListeners 时，会触发
/// setState，并在下一帧调用InheritedElement.update方法，此方法自动调用notifyClinets，调用所以对此
/// InheritedElement有依赖的Elment的didChangeDependencies方法，刷新那些widget

mixin _ListenableDelegateMixin<T extends Listenable> on ValueStateDelegate<T> {
    UpdateShouldNotify<T> updateShouldNotify;
    VoidCallback _removeListener;
        
    @override
    void initDelegate() {
        super.initDelegate();
        if (value != null) startListening(value);
    }

    @override
    void didUpdateDelegate(StateDelegate old) {
        super.didUpdateDelegate(old);
        final delegate = old as _ListenableDelegateMixin<T>;

        _removeListener = delegate._removeListener;
        updateShouldNotify = delegate.updateShouldNotify;
    }

    /// 向 listenable 中添加一个listener，每当listenable的notifyListeners 被调用的时候，执行
    /// setState，当这个delegate被销毁的时候，移除listener
    void startListening(T listenable, {bool rebuild = false}) {
        var buildCount = 0;
        final setState = this.setState;
        final listener = () => setState(() => buildCount++);

        var capturedBuildCount = buildCount;
        if (rebuild) capturedBuildCount--;
        updateShouldNotify = (_, __) {
            final res = buildCount != capturedBuildCount;
            capturedBuildCount = buildCount;
            return res;
        };

        listenable.addListener(listener);
        _removeListener = () {
            listenable.removeListener(listener);
            _removeListener = null;
            updateShouldNotify = null;
        };
    }

    @override
    void dispose() {
        _removeListener?.call();
        super.dispose();
    }
}
```



## ChangeNotifierProvider

```dart
/// 只是对 ListenableProvider 的简单封装
class ChangeNotifierProvider<T extends ChangeNotifier>
    extends ListenableProvider<T> 
    implements SingleChildCloneableWidget 
{
    static void _disposer(BuildContext context, ChangeNotifier notifier) =>
        notifier?.dispose();
    
    ChangeNotifierProvider({
        Key key,
        @required ValueBuilder<T> builder,
        Widget child,
    }) : super(key: key, builder: builder, dispose: _disposer, child: child);

    ChangeNotifierProvider.value({
        Key key,
        @required T value,
        Widget child,
    }) : super.value(key: key, value: value, child: child);
}
```



## StreamProvider

```dart
class StreamProvider<T> 
    extends ValueDelegateWidget<Stream<T>>
    implements SingleChildCloneableWidget {
    
  StreamProvider({
    Key key,
    @required ValueBuilder<Stream<T>> builder,
    T initialData,
    ErrorBuilder<T> catchError,
    UpdateShouldNotify<T> updateShouldNotify,
    Widget child,
  }) : this._(
          key: key,
          delegate: BuilderStateDelegate<Stream<T>>(builder),
          initialData: initialData,
          catchError: catchError,
          updateShouldNotify: updateShouldNotify,
          child: child,
        );

  StreamProvider.controller({
    Key key,
    @required ValueBuilder<StreamController<T>> builder,
    T initialData,
    ErrorBuilder<T> catchError,
    UpdateShouldNotify<T> updateShouldNotify,
    Widget child,
  }) : this._(
          key: key,
          delegate: _StreamControllerBuilderDelegate(builder),
          initialData: initialData,
          catchError: catchError,
          updateShouldNotify: updateShouldNotify,
          child: child,
        );

  StreamProvider.value({
    Key key,
    @required Stream<T> value,
    T initialData,
    ErrorBuilder<T> catchError,
    UpdateShouldNotify<T> updateShouldNotify,
    Widget child,
  }) : this._(
          key: key,
          delegate: SingleValueDelegate(value),
          initialData: initialData,
          catchError: catchError,
          updateShouldNotify: updateShouldNotify,
          child: child,
        );

  StreamProvider._({
    Key key,
    @required ValueStateDelegate<Stream<T>> delegate,
    this.initialData,
    this.catchError,
    this.updateShouldNotify,
    this.child,
  }) : super(key: key, delegate: delegate);

  final T initialData;
  final Widget child;
  final ErrorBuilder<T> catchError;
  final UpdateShouldNotify<T> updateShouldNotify;

  @override
  StreamProvider<T> cloneWithChild(Widget child) {
    return StreamProvider._(
      key: key,
      delegate: delegate,
      updateShouldNotify: updateShouldNotify,
      initialData: initialData,
      catchError: catchError,
      child: child,
    );
  }

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<T>(
      stream: delegate.value,
      initialData: initialData,
      builder: (_, snapshot) {
        return InheritedProvider<T>(
          value: _snapshotToValue(snapshot, context, catchError, this),
          child: child,
          updateShouldNotify: updateShouldNotify,
        );
      },
    );
  }
}
```



### StreamControllerBuilderDelegate

```dart
class _StreamControllerBuilderDelegate<T>
    extends ValueStateDelegate<Stream<T>> {
  _StreamControllerBuilderDelegate(this._builder) : assert(_builder != null);

  StreamController<T> _controller;
  final ValueBuilder<StreamController<T>> _builder;

  @override
  Stream<T> value;

  @override
  void initDelegate() {
    super.initDelegate();
    _controller = _builder(context);
    value = _controller?.stream;
  }

  @override
  void didUpdateDelegate(_StreamControllerBuilderDelegate<T> old) {
    super.didUpdateDelegate(old);
    value = old.value;
    _controller = old._controller;
  }

  @override
  void dispose() {
    _controller?.close();
    super.dispose();
  }
}
```



## FutureBuilder

```dart
class FutureProvider<T> 
    extends ValueDelegateWidget<Future<T>>
    implements SingleChildCloneableWidget 
{
  FutureProvider({
    Key key,
    @required ValueBuilder<Future<T>> builder,
    T initialData,
    ErrorBuilder<T> catchError,
    UpdateShouldNotify<T> updateShouldNotify,
    Widget child,
  }) : this._(
          key: key,
          initialData: initialData,
          catchError: catchError,
          updateShouldNotify: updateShouldNotify,
          delegate: BuilderStateDelegate(builder),
          child: child,
        );

  FutureProvider.value({
    Key key,
    @required Future<T> value,
    T initialData,
    ErrorBuilder<T> catchError,
    UpdateShouldNotify<T> updateShouldNotify,
    Widget child,
  }) : this._(
          key: key,
          initialData: initialData,
          catchError: catchError,
          updateShouldNotify: updateShouldNotify,
          delegate: SingleValueDelegate(value),
          child: child,
        );

  FutureProvider._({
    Key key,
    @required ValueStateDelegate<Future<T>> delegate,
    this.initialData,
    this.catchError,
    this.updateShouldNotify,
    this.child,
  }) : super(key: key, delegate: delegate);

  final T initialData;
  final Widget child;
  final ErrorBuilder<T> catchError;

  final UpdateShouldNotify<T> updateShouldNotify;

  @override
  FutureProvider<T> cloneWithChild(Widget child) {
    return FutureProvider._(
      key: key,
      delegate: delegate,
      updateShouldNotify: updateShouldNotify,
      initialData: initialData,
      catchError: catchError,
      child: child,
    );
  }

  @override
  Widget build(BuildContext context) {
    /// 仅在外层使用FutureBuilder来实现异步构建
    return FutureBuilder<T>(
      future: delegate.value,
      initialData: initialData,
      builder: (_, snapshot) {
        return InheritedProvider<T>(
          value: _snapshotToValue(snapshot, context, catchError, this),
          updateShouldNotify: updateShouldNotify,
          child: child,
        );
      },
    );
  }
}
```





### 附：ChangeNotifier

```dart
class ChangeNotifier implements Listenable {
    ObserverList<VoidCallback> _listeners = ObserverList<VoidCallback>();

    /// 是否有listeners
    @protected
    bool get hasListeners {
        return _listeners.isNotEmpty;
    }

    /// 注册一个listener
    @override
    void addListener(VoidCallback listener) {
        _listeners.add(listener);
    }

    /// 移除一个listener
    @override
    void removeListener(VoidCallback listener) {
        assert(_debugAssertNotDisposed());
        _listeners.remove(listener);
    }

    /// 释放——listeners的内存
    @mustCallSuper
    void dispose() {
        _listeners = null;
    }

    @protected
    @visibleForTesting
    void notifyListeners() {
      if (_listeners != null) {
        final List<VoidCallback> localListeners = List<VoidCallback>.from(_listeners);
        for (VoidCallback listener in localListeners) {
          // 这里采用了一个O(n^2)的算法
          // 在 flutter 源码里，通常采用一个 HashSet 来储存被移除的 listener
          // 通过 if(!_removedListener.contains(listener)) 来判断  
          // 查找时间复杂度O(1)  
          // 在 listener 不多的情况下，其实没有太大的差别 
          try {
            if (_listeners.contains(listener))
              listener();
          } catch (exception, stack) {
            ...
          }
        }
      }
    }
}
```


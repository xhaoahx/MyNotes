# Flutter Consumer 源码分析(Deprecated)

奇怪状态管理库，可用性不高，思想可以借鉴

等待后期版本更新

```dart
class Consumer<T> {
  T _state;
  StreamController controller;
  Stream stream;

  Consumer(this._state) {
    controller = StreamController.broadcast();
    stream = controller.stream;
    stream.listen((data) {
      _state = data;
    });
  }

  Widget build({
    @required List<dynamic> Function(T s) memo,
    @required Widget Function(BuildContext ctx, T state) builder,
  }) {
    return _ConsumerWidget<T>(ctrl: this, memo: memo, builder: builder);
  }

  T getState() {
    return _state;
  }

  T setState(Function(T state) fn) {
    fn(_state);
    controller.add(_state);
    return _state;
  }
}

/// ## _ConsumerWidget
///
/// > _ConsumerWidget is like react.context.consumer style's state manage widget
///
/// builder[required]: use return widget
/// memo[option]: (state) => [], like react.useMemo, only array object is changed, widget can be update
/// shouldWidgetUpdate[option]: (state) => bool, like react.shouldComponentUpdate, intercept update at return false;
/// _ConsumerWidget listen Store.stream at initState, and cancel listen at widget dispose.
class _ConsumerWidget<T> extends StatefulWidget {
  final Consumer<T> ctrl;
  final List<dynamic> Function(T state) memo;
  final Widget Function(BuildContext ctx, T state) builder;

  _ConsumerWidget(
      {@required this.ctrl,
      @required this.builder,
      @required this.memo,
      Key key})
      : super(key: key);

  @override
  _ConsumerWidgetState createState() =>
      _ConsumerWidgetState<T>(ctrl, memo, builder);
}

class _ConsumerWidgetState<T> extends State<_ConsumerWidget> {
  StreamSubscription _sub;
  List<dynamic> _lastMemo;
  final Consumer<T> _ctrl;
  final List<dynamic> Function(T state) _memo;
  final Widget Function(BuildContext ctx, T state) _builder;

  _ConsumerWidgetState(this._ctrl, this._memo, this._builder);

  @override
  void initState() {
    super.initState();
    /// 通过判断提供的Memo数组的引用判断状态是否被更新了，并决定是否要调用setState  
    _lastMemo = [..._memo(_ctrl.getState())];

    _sub = _ctrl.stream.listen((data) {
      if (_lastMemo.length > 0) {
        bool isUpdate = false;
        List nowMemo = [..._memo(_ctrl.getState())];
        for (var i = 0; i < _lastMemo.length; i++) {
          if (_lastMemo[i] != nowMemo[i]) {
            isUpdate = true;
            break;
          }
        }
        if (isUpdate == true) {
          if (mounted) {
            _lastMemo = nowMemo;

            setState(() {});
          }
        }
      }
    });
  }

  @override
  void dispose() {
    super.dispose();
    _sub.cancel();
  }

  @override
  Widget build(BuildContext context) {
    return _builder(context, _ctrl.getState());
  }
}
```


# Flutter GlobalKey 源码分析

[TOC]

## GlobalKey

```dart
abstract class GlobalKey<T extends State<StatefulWidget>> extends Key {
  /// 自身为抽象类，构造方法实际上返回的是一个 LabeledGlobalkey
  factory GlobalKey({ String debugLabel }) => LabeledGlobalKey<T>(debugLabel);

  const GlobalKey.constructor() : super.empty();

  /// 全局静态map，用GlobalKey作为键，element作为值，形成映射  
  static final Map<GlobalKey, Element> _registry = <GlobalKey, Element>{};

  /// 注册一个 element  
  void _register(Element element) {
    _registry[this] = element;
  }

  /// 注销一个 element  
  void _unregister(Element element) {
    if (_registry[this] == element)
      _registry.remove(this);
  }

  Element get _currentElement => _registry[this];

  BuildContext get currentContext => _currentElement;

  Widget get currentWidget => _currentElement?.widget;

  /// 在构造GlobalKey时，以<...State>作为类型才能正确地获取到相应的State
  T get currentState {
    final Element element = _currentElement;
    if (element is StatefulElement) {
      final StatefulElement statefulElement = element;
      final State state = statefulElement.state;
      if (state is T)
        return state;
    }
    return null;
  }
}
```



## LabeledGlobalKey

```dart
/// 带 debug label 的 Key
class LabeledGlobalKey<T extends State<StatefulWidget>> extends GlobalKey<T> {
  LabeledGlobalKey(this._debugLabel) : super.constructor();

  final String _debugLabel;

  @override
  String toString() {
    final String label = _debugLabel != null ? ' $_debugLabel' : '';
    if (runtimeType == LabeledGlobalKey)
      return '[GlobalKey#${shortHash(this)}$label]';
    return '[${describeIdentity(this)}$label]';
  }
}
```


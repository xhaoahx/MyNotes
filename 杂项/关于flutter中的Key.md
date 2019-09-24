# 关于flutter中的Key

[TOC]



## Key

在flutter的key.dart中有这么一段：

```dart
A [Key] is an identifier for [Widget]s, [Element]s and [SemanticsNode]s.
A new widget will only be used to update an existing element if its key is 
    the same as the key of the current widget associated with the element.
Keys must be unique amongst the [Element]s with the same parent.
Subclasses of [Key] should either subclass [LocalKey] or [GlobalKey].

///    Key是Widget、Element和SemanticsNode的一个标识符。
///    一个现存的Widget能被一个新的Widget更新 仅当 关联着Element的Widget和新的Widget有相同的Key。
///    Key对于Elements中有着同样parent的元素必须是独特的
///    Key的子类必须是LocalKey或Globalkey的子类。
```

Key类的声明：

```dart
abstract class Key {
  /// Construct a [ValueKey<String>] with the given [String].
  ///
  /// This is the simplest way to create keys.
  const factory Key(String value) = ValueKey<String>;

  /// Default constructor, used by subclasses.
  ///
  /// Useful so that subclasses can call us, because the [new Key] factory
  /// constructor shadows the implicit constructor.
  /// 这个构造函数掩盖了隐含的构造器（指工厂方法）
  @protected
  const Key.empty();
}
```

值得注意的是，Key有一个工厂方法，和一个命名构造函数：Key.empty()。此外，有一段注释说明了GolbalKey:

```dart
By contrast, [GlobalKey]s must be unique across the entire app.
与此相对的，GlobalKey对于整个APP必须是独特的
```

## LocalKey

LocalKey对Key做了一层封装：

```da
abstract class LocalKey extends Key {
  /// Default constructor, used by subclasses.
  const LocalKey() : super.empty();
}
```

##ValueKey

```dart
class ValueKey<T> extends LocalKey {
  /// Creates a key that delegates its [operator==] to the given value.
  const ValueKey(this.value);

  /// The value to which this key delegates its [operator==]
  final T value;

  @override
  bool operator ==(dynamic other) {
    if (other.runtimeType != runtimeType)
      return false;
    final ValueKey<T> typedOther = other;
    return value == typedOther.value;
  }

  @override
  int get hashCode => hashValues(runtimeType, value);

  @override
  String toString() {
    final String valueString = T == String ? '<\'$value\'>' : '<$value>';
    // 其中 \' 是一个' 即 '<\'$123131\'>' = <'aksdja'>
    // The crazy on the next line is a workaround for
    // https://github.com/dart-lang/sdk/issues/33297
    if (runtimeType == _TypeLiteral<ValueKey<T>>().type)
      return '[$valueString]';
    return '[$T $valueString]';
  }
}

class _TypeLiteral<T> {
  Type get type => T;
}
```

在GitHub上有一段关于 _TypeLiteral<ValueKey<T>>().type 的解释：

```dart

Suppose you want a Type variable to contain the type of a List<int>.
	假设你想有一个类型变量来保存List<int>.
It should be possible to do it as:
	你可以这么做：
Type type = List<int>;
	
Instead you have to do things like:
	你也可以这么做：
Type type = (<T>() => T)<List<int>>();
/*这里的(<T>() => T)是一个函数，即:
         <T>(){return T;},
  然后：
         <List<int>>(){return List<int>;},
*/
or:

class GetType<T> { 
    const GetType(); 
    Type get type => T; 
}
/*通过GetType类的getter来自动获取type*/
Type type = const GetType<List<int>>().type;
```

<font color = red>注意</font>，假如通过Key('hello')来创建Key，会调用ValueKey<T>的构造器，即创建了一个ValueKey的实例。

```dart
/// 举例
class A {
  const factory A(String value) = B;
  const A.create();
}
class B extends A{
  const B(this.value):super.create();
  final String value;
}
class C{}
class D extends C{}

void main(){
  B b = A('hello');// OK
  A a = B('hello');// OK，这意味着A创建了一个B,并使得a引用它

  C c = D();
  D d = C();//这一行会有错误：C不是D类
}
```



## UniqueKey

```dart
class UniqueKey extends LocalKey {
  /// Creates a key that is equal only to itself.
   	创造一个只与自己相等的Key
  // ignore: prefer_const_constructors_in_immutables , never use const for this class
    这里不使用const 构造器 ，无须使用对于这个类使用const
  UniqueKey();

  @override
  String toString() => '[#${shortHash(this)}]';
}
```



## GolbalKey

接下来是GlobalKey。

```dart
/// A key that is unique across the entire app.
/// 	它对于整个APP来说是独特的，可以有多个，但不能一样
/// Global keys uniquely identify elements. Global keys provide access to other
/// 	GlobalKey独特地识别element,并能够访问其他与Element有关的的对象，例如buildcontext
/// objects that are associated with elements, such as the a [BuildContext] and,
/// for [StatefulWidget]s, a [State].
///
/// Widgets that have global keys reparent their subtrees when they are moved
/// from one location in the tree to another location in the tree. In order to
/// reparent its subtree, a widget must arrive at its new location in the tree
/// in the same animation frame in which it was removed from its old location in
/// the tree.
///
/// 有GloBalKey的Widget在从一棵树上的某个位置移动另一个位置的时候将他们的子树重定父级。为了将他们的子树
/// 重定父级，一个Widget在从旧的位置移动的到新的位置的过程中必须处于相同的动画框架里
///
///
/// Global keys are relatively expensive. If you don't need any of the features
/// listed above, consider using a [Key], [ValueKey], [ObjectKey], or
/// [UniqueKey] instead.
///
/// You cannot simultaneously include two widgets in the tree with the same
/// global key. Attempting to do so will assert at runtime.
///
/// See also the discussion at [Widget.key].
@optionalTypeArgs
abstract class GlobalKey<T extends State<StatefulWidget>> extends Key {
  /// Creates a [LabeledGlobalKey], which is a [GlobalKey] with a label used for
  /// debugging.
  ///
  /// The label is purely for debugging and not used for comparing the identity
  /// of the key.
  factory GlobalKey({ String debugLabel }) => LabeledGlobalKey<T>(debugLabel);

  /// Creates a global key without a label.
  ///
  /// Used by subclasses because the factory constructor shadows the implicit
  /// constructor.
  const GlobalKey.constructor() : super.empty();

  static final Map<GlobalKey, Element> _registry = <GlobalKey, Element>{};
  static final Set<Element> _debugIllFatedElements = HashSet<Element>();
  static final Map<GlobalKey, Element> _debugReservations = <GlobalKey, Element>{};

  void _register(Element element) {
    assert(() {
      if (_registry.containsKey(this)) {
        assert(element.widget != null);
        assert(_registry[this].widget != null);
        assert(element.widget.runtimeType != _registry[this].widget.runtimeType);
        _debugIllFatedElements.add(_registry[this]);
      }
      return true;
    }());
    _registry[this] = element;
  }

  void _unregister(Element element) {
    assert(() {
      if (_registry.containsKey(this) && _registry[this] != element) {
        assert(element.widget != null);
        assert(_registry[this].widget != null);
        assert(element.widget.runtimeType != _registry[this].widget.runtimeType);
      }
      return true;
    }());
    if (_registry[this] == element)
      _registry.remove(this);
  }

  void _debugReserveFor(Element parent) {
    assert(() {
      assert(parent != null);
      if (_debugReservations.containsKey(this) && _debugReservations[this] != parent) {
        // It's possible for an element to get built multiple times in one
        // frame, in which case it'll reserve the same child's key multiple
        // times. We catch multiple children of one widget having the same key
        // by verifying that an element never steals elements from itself, so we
        // don't care to verify that here as well.
        final String older = _debugReservations[this].toString();
        final String newer = parent.toString();
        if (older != newer) {
          throw FlutterError(
            'Multiple widgets used the same GlobalKey.\n'
            'The key $this was used by multiple widgets. The parents of those widgets were:\n'
            '- $older\n'
            '- $newer\n'
            'A GlobalKey can only be specified on one widget at a time in the widget tree.'
          );
        }
        throw FlutterError(
          'Multiple widgets used the same GlobalKey.\n'
          'The key $this was used by multiple widgets. The parents of those widgets were '
          'different widgets that both had the following description:\n'
          '  $newer\n'
          'A GlobalKey can only be specified on one widget at a time in the widget tree.'
        );
      }
      _debugReservations[this] = parent;
      return true;
    }());
  }

  static void _debugVerifyIllFatedPopulation() {
    assert(() {
      Map<GlobalKey, Set<Element>> duplicates;
      for (Element element in _debugIllFatedElements) {
        if (element._debugLifecycleState != _ElementLifecycle.defunct) {
          assert(element != null);
          assert(element.widget != null);
          assert(element.widget.key != null);
          final GlobalKey key = element.widget.key;
          assert(_registry.containsKey(key));
          duplicates ??= <GlobalKey, Set<Element>>{};
          final Set<Element> elements = duplicates.putIfAbsent(key, () => HashSet<Element>());
          elements.add(element);
          elements.add(_registry[key]);
        }
      }
      _debugIllFatedElements.clear();
      _debugReservations.clear();
      if (duplicates != null) {
        final StringBuffer buffer = StringBuffer();
        buffer.writeln('Multiple widgets used the same GlobalKey.\n');
        for (GlobalKey key in duplicates.keys) {
          final Set<Element> elements = duplicates[key];
          buffer.writeln('The key $key was used by ${elements.length} widgets:');
          for (Element element in elements)
            buffer.writeln('- $element');
        }
        buffer.write('A GlobalKey can only be specified on one widget at a time in the widget tree.');
        throw FlutterError(buffer.toString());
      }
      return true;
    }());
  }

  Element get _currentElement => _registry[this];

  /// The build context in which the widget with this key builds.
  ///
  /// The current context is null if there is no widget in the tree that matches
  /// this global key.
  BuildContext get currentContext => _currentElement;

  /// The widget in the tree that currently has this global key.
  ///
  /// The current widget is null if there is no widget in the tree that matches
  /// this global key.
  Widget get currentWidget => _currentElement?.widget;

  /// The [State] for the widget in the tree that currently has this global key.
  ///
  /// The current state is null if (1) there is no widget in the tree that
  /// matches this global key, (2) that widget is not a [StatefulWidget], or the
  /// associated [State] object is not a subtype of `T`.
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
```


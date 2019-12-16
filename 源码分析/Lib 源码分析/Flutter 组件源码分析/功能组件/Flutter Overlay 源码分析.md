Flutter Overlay 源码分析



[TOC]

## OverlayEntry

```dart
/// 覆盖层实体
/// 持有 builder（用于构建子树）
class OverlayEntry {
  /// 创建一个覆盖层实体
  ///
  /// 若要插入一层 [Overlay], 首先通过 [Overlay.of] 来获取到 overlay，然后调用
  /// [OverlayState.insert]
  /// 若要移除一层 [Overlay], 调用其 [remove]
  /// 
  OverlayEntry({
    @required this.builder,
    bool opaque = false,
    bool maintainState = false,
  }) : assert(builder != null),
       assert(opaque != null),
       assert(maintainState != null),
       _opaque = opaque,
       _maintainState = maintainState;

  /// 在覆盖层里创建widget
  ///
  /// 若要重建这个widget（重新调用builder），调用[markNeedsBuild]
  final WidgetBuilder builder;

  /// 覆盖层是否布满全屏
  /// 若设置为 true 并且 maintainState 为 false，为了提升性能，则不会渲染底层视图
  bool get opaque => _opaque;
  bool _opaque;
  set opaque(bool value) {
    if (_opaque == value)
      return;
    _opaque = value;
    _overlay._didChangeEntryOpacity();
  }

  /// 是否保持状态  
  bool get maintainState => _maintainState;
  bool _maintainState;
  set maintainState(bool value) {
    if (_maintainState == value)
      return;
    _maintainState = value;
    _overlay._didChangeEntryOpacity();
  }

  OverlayState _overlay;
  final GlobalKey<_OverlayEntryState> _key = GlobalKey<_OverlayEntryState>();

  /// 移除此覆盖层
  void remove() {
    final OverlayState overlay = _overlay;
    _overlay = null;
    /// 若正处于帧构建阶段，则在帧尾回调移除此覆盖层  
    if (SchedulerBinding.instance.schedulerPhase == SchedulerPhase.persistentCallbacks) {
      SchedulerBinding.instance.addPostFrameCallback((Duration duration) {
        overlay._remove(this);
      });
    /// 否则立即移除此覆盖层    
    } else {
      overlay._remove(this);
    }
  }

  /// 刷新widget
  void markNeedsBuild() {
    _key.currentState?._markNeedsBuild();
  }
}
```



## _OverlayEntry

```dart
/// 注意到 OverlayEntry 是用来保存数据和引用
/// 而 _OverlayEntry 才是 widget，持有 OverlayEntry
class _OverlayEntry extends StatefulWidget {
  _OverlayEntry(this.entry)
    : assert(entry != null),
      super(key: entry._key);

  final OverlayEntry entry;

  @override
  _OverlayEntryState createState() => _OverlayEntryState();
}
```


## _OverlayEntryState

```dart
class _OverlayEntryState extends State<_OverlayEntry> {
  @override
  Widget build(BuildContext context) {
    return widget.entry.builder(context);
  }

  /// 这个方法触发 setState 调用了 build 方法，实际上直接调用了 widget.entry 的 builder 回调
  /// 所以触发的是 builder 的重建，即重新构建了子树
  /// 而 setState 是保护方法，所以包装一层供外界调用
  void _markNeedsBuild() {
    setState(() { /* the state that changed is in the builder */ });
  }
}
```



## Overlay

```dart
/// 一个可以独立管理其（覆盖层）的[Stack]
///
class Overlay extends StatefulWidget {
  const Overlay({
    Key key,
    this.initialEntries = const <OverlayEntry>[],
  }) : uper(key: key);

  /// 初始状态下的覆盖层
  /// 仅在 OverlayState.initState 时使用 
  final List<OverlayEntry> initialEntries;

  static OverlayState of(BuildContext context, { Widget debugRequiredFor }) {
    final OverlayState result = 
        // ancestorStateOfType 通过 TypeMatcher 来获取指定类型的 State
        context.ancestorStateOfType(const TypeMatcher<OverlayState>());
    return result;
  }

  @override
  OverlayState createState() => OverlayState();
}
```



## OverlayState

```dart
/// 注意到这里 mixin 了一个 TickerProviderStateMixin
class OverlayState extends State<Overlay> with TickerProviderStateMixin {
  final List<OverlayEntry> _entries = <OverlayEntry>[];

  @override
  void initState() {
    super.initState();
    insertAll(widget.initialEntries);
  }

  // 返回给定 OverlayEntry 的下标
  int _insertionIndex(OverlayEntry below, OverlayEntry above) {
    if (below != null)
      return _entries.indexOf(below);
    if (above != null)
      return _entries.indexOf(above) + 1;
    return _entries.length;
  }

  /// 插入一个覆盖层 OverlayEntry 会使其持有 _overlay 持有此 OverlayState 的引用
  void insert(OverlayEntry entry, { OverlayEntry below, OverlayEntry above }) {
    entry._overlay = this;
    setState(() {
      _entries.insert(_insertionIndex(below, above), entry);
    });
  }

  /// 插入一些覆盖层
  void insertAll(Iterable<OverlayEntry> entries, { OverlayEntry below, OverlayEntry above }) {
    if (entries.isEmpty) return;
    for (OverlayEntry entry in entries) {
      entry._overlay = this;
    }
    setState(() {
      _entries.insertAll(_insertionIndex(below, above), entries);
    });
  }

  /// 移除列表中的所有覆盖层，然后按照指定的顺序再次插入它们
  void rearrange(Iterable<OverlayEntry> newEntries, { OverlayEntry below, OverlayEntry above }) {
    final List<OverlayEntry> newEntriesList = 
        newEntries is List<OverlayEntry> ? newEntries : newEntries.toList(growable: false);
    if (newEntriesList.isEmpty)
      return;
    if (listEquals(_entries, newEntriesList))
      return;
    final LinkedHashSet<OverlayEntry> old = LinkedHashSet<OverlayEntry>.from(_entries);
    for (OverlayEntry entry in newEntriesList) {
      entry._overlay ??= this;
    }
    setState(() {
      _entries.clear();
      _entries.addAll(newEntriesList);
      old.removeAll(newEntriesList);
      _entries.insertAll(_insertionIndex(below, above), old);
    });
  }

  /// 移除一个覆盖层  
  void _remove(OverlayEntry entry) {
    if (mounted) {
      setState(() {
        _entries.remove(entry);
      });
    }
  }

  /// 
  void _didChangeEntryOpacity() {
    setState(() {
      // We use the opacity of the entry in our build function, which means we
      // our state has changed.
    });
  }

  @override
  Widget build(BuildContext context) {
    // 以下的两个列表填充的顺序都是从后向前的
    // 对于 offstage children 来说，它们无关紧要，因为没有被渲染，
    // 对于 onstage children 来说，它们会先被反转然后再加入树中
    /// 注意：
    /// onstateChildren 里包含的是需要被渲染的 _OverlayEntry，
    /// 而 offstateChildren 里包含的是不需要被渲染的 _OverlayEntry（但是需要维持构建状态）
    final List<Widget> onstageChildren = <Widget>[];
    final List<Widget> offstageChildren = <Widget>[];
    /// 是否是不透明的
    bool onstage = true;
    /// 对于 _entries，越靠近尾端的 entry 应该显示在越上层
    for (int i = _entries.length - 1; i >= 0; i -= 1) {
      final OverlayEntry entry = _entries[i];
      /// 如果是不透明的，那么把当前的 entry 作为信息给 _OverlayEntry，加入需要渲染的列表中
      if (onstage) {
        /// 这里的 _OverlayEntry 是 StatfulWidget，OverlayEntry 只是一个保存信息类
        onstageChildren.add(_OverlayEntry(entry));
        /// 如果有一个 entry 为 opaque（不透明），那么它之后的 entry 不可见，即无需渲染
        if (entry.opaque)
          onstage = false;
        /// 如果一个不可见的 entry 之后的 entry 需要保持状态，
        /// 那么把它加入 offstageChildren 中，用于维持构建状态，而不要渲染  
      } else if (entry.maintainState) {
        offstageChildren.add(TickerMode(enabled: false, child: _OverlayEntry(entry)));
      }
      /// else
      /// 如果 一个 entry 不可见并且无需维持状态，显然那么它无需渲染
    }
      
    /// 注意：
    /// 如果一个 entry 是不可见并且无需保存状态，那么 setState 之后他不会加入 onstageChildren 或 
    /// offstageChildren 中，于是被从子树中移除  
    /// 而 offstageChildren 无需渲染，故顺序无关紧要
    return _Theatre(
      onstage: Stack(
        fit: StackFit.expand,
        /// 这里反转了 onstageChildren ，排在后面的元素会放在到前面，于是保证了 entry 的顺序 
        children: onstageChildren.reversed.toList(growable: false),
      ),
      offstage: offstageChildren,
    );
  }

}

```



## _Theatre

```dart
class _Theatre extends RenderObjectWidget {
  _Theatre({
    this.onstage,
    @required this.offstage,
  }) : assert(offstage != null),
       assert(!offstage.any((Widget child) => child == null));

  /// 子 widget 
  final Stack onstage;
  
  /// 只构建而不渲染
  final List<Widget> offstage;

  @override
  _TheatreElement createElement() => _TheatreElement(this);

  @override
  _RenderTheatre createRenderObject(BuildContext context) => _RenderTheatre();
}
```



## _TheatreElement

```dart
class _TheatreElement extends RenderObjectElement {
  _TheatreElement(_Theatre widget)
    : super(widget);

  @override
  _Theatre get widget => super.widget;

  @override
  _RenderTheatre get renderObject => super.renderObject;

  /// _onstage 是一个单一的 elment（stack）  
  Element _onstage;
  static final Object _onstageSlot = Object();

  /// _offstage 是一组 element（需要维持状态而不渲染）  
  List<Element> _offstage;
  final Set<Element> _forgottenOffstageChildren = HashSet<Element>();

  /// 这个 element 把 onstate 作为自己的子 element
  /// 因此，这个 element 持有的 renderObject 的孩子只能是 renderStatk，
  /// 并且，绘制时只对 RenderStack 的子节点进行绘制 
  /// 插入一个 childRenderObject  
  @override
  void insertChildRenderObject(RenderBox child, dynamic slot) {
    if (slot == _onstageSlot) {
      renderObject.child = child;
    } else {
      renderObject.insert(child, after: slot?.renderObject);
    }
  }

  /// 移动一个 childRenderObject    
  @override
  void moveChildRenderObject(RenderBox child, dynamic slot) {
    if (slot == _onstageSlot) {
      renderObject.remove(child);
      renderObject.child = child;
    } else {
      if (renderObject.child == child) {
        renderObject.child = null;
        renderObject.insert(child, after: slot?.renderObject);
      } else {
        renderObject.move(child, after: slot?.renderObject);
      }
    }
  }

  /// 移除一个 childRenderObject    
  @override
  void removeChildRenderObject(RenderBox child) {
    if (renderObject.child == child) {
      renderObject.child = null;
    } else {
      renderObject.remove(child);
    }
  }

  @override
  void visitChildren(ElementVisitor visitor) {
    if (_onstage != null)
      visitor(_onstage);
    for (Element child in _offstage) {
      if (!_forgottenOffstageChildren.contains(child))
        visitor(child);
    }
  }

  @override
  bool forgetChild(Element child) {
    if (child == _onstage) {
      _onstage = null;
    } else {
      _forgottenOffstageChildren.add(child);
    }
    return true;
  }

  @override
  void mount(Element parent, dynamic newSlot) {
    /// 根据 widget 来创建对应的 _onstage 和 _offstage  
    super.mount(parent, newSlot);  
    _onstage = updateChild(_onstage, widget.onstage, _onstageSlot);
    _offstage = List<Element>(widget.offstage.length);
    Element previousChild;
    /// 注意在这里调用了 inflateWidget 来构建了一次 offstage 列表得到 element，维持了状态
    for (int i = 0; i < _offstage.length; i += 1) {
      final Element newChild = inflateWidget(widget.offstage[i], previousChild);
      _offstage[i] = newChild;
      previousChild = newChild;
    }
  }

  @override
  void update(_Theatre newWidget) {
    super.update(newWidget);
    assert(widget == newWidget);
    _onstage = updateChild(_onstage, widget.onstage, _onstageSlot);
    _offstage = updateChildren(
        _offstage, 
        widget.offstage, 
        forgottenChildren: _forgottenOffstageChildren
    );
    _forgottenOffstageChildren.clear();
  }
}
```



## _RenderTheatre

```dart
class _RenderTheatre extends RenderBox
  with RenderObjectWithChildMixin<RenderStack>, 
	   RenderProxyBoxMixin<RenderStack>,
       ContainerRenderObjectMixin<RenderBox, StackParentData> 
{
  @override
  void setupParentData(RenderObject child) {
    if (child.parentData is! StackParentData)
      child.parentData = StackParentData();
  }

  @override
  void redepthChildren() {
    if (child != null)  redepthChild(child);
    super.redepthChildren();
  }

  @override
  void visitChildren(RenderObjectVisitor visitor) {
    if (child != null)
      visitor(child);
    super.visitChildren(visitor);
  }
}
```

### 总结

Overlay 通常使用在组件树的较上层，其子树可以通过 Overlay.of(context) 来获取到 Overlay 组件对应的 OverlayState
OverlayEntry 只作为信息类出现，一个 OverlayEntry 即对应一个 _OverlayEntry 组件，在 Overlay 的 build 方法中，会根据其 opaque 属性来把 OverlayEntry 映射成 _OverlayEntry,并使用 OverlayEntry.builder 来构建子树

通常可以调用 OverlayState 来在当前视图上层创建覆盖层（例如弹出对话框。普通的页面也是一个Overlay）
即：

1. 使用 Overlay.insert (OverlayEntry entry) 来在当前视图上方插入一个新的视图
   可以把所有视图看作是层叠的状态，即当成一个覆盖层列表（当列表中有一层是“不透明”状态的时候，其下层覆盖层是无需被渲染的，这个渲染管理工作由 RenderTheatre 完成）
2. 使用 OverlayEntry.remove 来把它对应的覆盖层从覆盖层列表中移除。
3. OverlayEntry.markNeedsBuild 方法可以用来从外界刷新其 Builder 方法返回的 Widget。

事实上，Overlay 可以类比成一个 Stack，插入的 OverlayEntry 可以被看做其子节点。故其渲染方式可以被看做是后插入的被渲染在最上层。因此，如果有一层覆盖层处于不透明的状态时，其下层的覆盖层无需被渲染
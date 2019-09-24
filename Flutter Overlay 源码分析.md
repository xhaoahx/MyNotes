Flutter Overlay 源码分析



[TOC]

## OverlayEntry

```dart
/// 覆盖层实体
/// 持有builder（用于构建子树）
class OverlayEntry {
  /// 创建一个覆盖层实体
  ///
  /// 若要插入一层 [Overlay], 首先通过 [Overlay.of] 来获取到 overlay，然后调用
  /// [OverlayState.insert]
  /// 若要移除一层 [Overlay], 调用 [remove]
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
  /// 若设置为 true 并且 maintainState 为 false，为了提升性能，则不会渲染底层view 
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
class _OverlayEntry extends StatefulWidget {
  _OverlayEntry(this.entry)
    : assert(entry != null),
      super(key: entry._key);

  final OverlayEntry entry;

  @override
  _OverlayEntryState createState() => _OverlayEntryState();
}

class _OverlayEntryState extends State<_OverlayEntry> {
  @override
  Widget build(BuildContext context) {
    return widget.entry.builder(context);
  }

  /// 这个方法触发setState调用了build方法，实际上直接调用了widget.entry(OverlayEntry)的builder，
  /// 所以触发的是builder的重建  
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
  }) : assert(initialEntries != null),
       super(key: key);

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
class OverlayState extends State<Overlay> with TickerProviderStateMixin {
  final List<OverlayEntry> _entries = <OverlayEntry>[];

  @override
  void initState() {
    super.initState();
    insertAll(widget.initialEntries);
  }

  // 返回指定插入位置的index  
  int _insertionIndex(OverlayEntry below, OverlayEntry above) {
    if (below != null)
      return _entries.indexOf(below);
    if (above != null)
      return _entries.indexOf(above) + 1;
    return _entries.length;
  }

  /// 以下方法在插入一个   
  /// 插入一个覆盖层 OverlayEntry 会将其_overlay设为自身，
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
    final List<OverlayEntry> newEntriesList = newEntries is List<OverlayEntry> ? newEntries : newEntries.toList(growable: false);
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

  void _didChangeEntryOpacity() {
    setState(() {
      // We use the opacity of the entry in our build function, which means we
      // our state has changed.
    });
  }

  @override
  Widget build(BuildContext context) {
    // 以下的两个列表都是从后向前的
    // 对于 offstage children 来说，它们无关紧要，因为没有被渲染，对于 onstage children 来说，
    // 它们会先被反转然后再加入树中  
    final List<Widget> onstageChildren = <Widget>[];
    final List<Widget> offstageChildren = <Widget>[];
    bool onstage = true;
    for (int i = _entries.length - 1; i >= 0; i -= 1) {
      final OverlayEntry entry = _entries[i];
      if (onstage) {
        /// 这里的 _OverlayEntry 是 StatfulWidget，OverlayEntry只是一个实体类 
        onstageChildren.add(_OverlayEntry(entry));
        /// 如果有一个entry为opaque（不透明），那么它之后的entry不可见  
        if (entry.opaque)
          onstage = false;
        /// 如果一个不可见的entry之后的entry需要保持状态，
        /// 那么把它加入 offstageChildren 中  
      } else if (entry.maintainState) {
        offstageChildren.add(TickerMode(enabled: false, child: _OverlayEntry(entry)));
      }
    }
      
    /// 注意：
    /// 如果一个entry是不可见并且无需保存状态，那么setState之后他不会加入onstageChildren 或 
    /// offstageChildren 中，于是被从子树中移除    
    return _Theatre(
      onstage: Stack(
        fit: StackFit.expand,
        /// 这里反转了 onstageChildren ，排在后面的元素会放在到前面 
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

  final Stack onstage;

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

  /// _onstage是一个单一的elment（stack）  
  Element _onstage;
  static final Object _onstageSlot = Object();

  /// _offstage是一组element（需要维持状态而不渲染）  
  List<Element> _offstage;
  final Set<Element> _forgottenOffstageChildren = HashSet<Element>();

  /// 插入一个childRenderObject  
  @override
  void insertChildRenderObject(RenderBox child, dynamic slot) {
    if (slot == _onstageSlot) {
      renderObject.child = child;
    } else {
      renderObject.insert(child, after: slot?.renderObject);
    }
  }

  /// 移动一个childRenderObject    
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

  /// 移除一个childRenderObject    
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
    /// 根据widget来创建对应的_onstage和_offstage  
    super.mount(parent, newSlot);  
    _onstage = updateChild(_onstage, widget.onstage, _onstageSlot);
    _offstage = List<Element>(widget.offstage.length);
    Element previousChild;
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


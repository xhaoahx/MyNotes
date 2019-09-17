# Flutter中的Element

[TOC]



## Element

```dart
/// Element的生命周期如下:
///
/// *框架通过调用[Widget.createElement]创建一个使用该widget作为初始配置的element。
/// *框架调用[mount]将新创建的element添加到树上一个指定的父节点上的一个指定位置。[mount]方法负责扩展任
///  何子部件并调用其[attachRenderObject]，将任何有关的的渲染对象附加到渲染树上。
/// *此时，element被认为是“活动的”，并且可能会出现在屏幕山。
/// *在某个时候，父节点可能会决定更改配置此element的widget，例如因为父节点重新构建状态。当这种情况 发生
///  时，框架将调用新widget的[update]。新widget将始终具有与旧wiedget相同的[runtimeType]和[key]。
///  如果父节点希望更改[runtimeType]或[key]或这个widget在树中的位置，它可以通过卸载这个elemtent来实
///  现，并在此位置填充一个新的widget。
/// *在某个时候，祖先可能会决定删除某个element(或者中间祖先)，该祖先通过在自身调用[deactiveChild]来完
///  成。将中间祖先deactive会从渲染树中删除该元素的渲染对象并添加这个element到[owner]的非活动元素列
///  表，从而导致框架对该element调用[deactive]。
/// *此时，element被认为是“非活动的”，并不会出现屏幕上。元素能保持在非活动状态，直到当前动画帧的结束。在
///  动画的最后帧，任何仍然不活动的元素将被unmount。
/// *如果element重新合并到树中(例如，因为它或他的祖先element有可重用的[GlobalKey])，框架将会从
///  [owner]的非活动elemten列表中删除元素并调用它的[activate]，并将element的渲染对象重新添加到渲染
///  树。此时，element再次被认为是“active”，并可能会出现在屏幕上。
/// *如果element在当前动画结束帧时没有重新合并到树中，框架将调用该element的[unmount]。
/// *此时，元素被认为是“defunct”，之后不再会合并到树中。


abstract class Element extends DiagnosticableTree implements BuildContext {
  /// widget通常重载createElement来创建element、
  /// 使element持有widget  
  Element(Widget widget)
    : assert(widget != null),
      _widget = widget;

  /// 父element  
  Element _parent;

  @override
  bool operator ==(Object other) => identical(this, other);

  @override
  int get hashCode => _cachedHash;
  final int _cachedHash = _nextHashCode = (_nextHashCode + 1) % 0xffffff;// 16777215
  static int _nextHashCode = 1;

  /// 父节点为子节点设置的位置
  ///
  /// 如果子类element只有一个孩子，那么这个域为null
  dynamic get slot => _slot;
  dynamic _slot;

  /// element在树中的深度
  int get depth => _depth;
  int _depth;

  /// 排序，按照深度小大，是否标记dirty 先后排序  
  static int _sort(Element a, Element b) {
    if (a.depth < b.depth)
      return -1;
    if (b.depth < a.depth)
      return 1;
    if (b.dirty && !a.dirty)
      return -1;
    if (a.dirty && !b.dirty)
      return 1;
    return 0;
  }

  @override
  Widget get widget => _widget;
  Widget _widget;

  /// 控制element生命流程的owner
  @override
  BuildOwner get owner => _owner;
  BuildOwner _owner;

  bool _active = false;

  /// 热重载相关
  @mustCallSuper
  @protected
  void reassemble() {
    markNeedsBuild();
    visitChildren((Element child) {
      child.reassemble();
    });
  }

  /// 向下查找第一个渲染对象
  RenderObject get renderObject {
    RenderObject result;
    void visit(Element element) {
      assert(result == null); // this verifies that there's only one child
      if (element is RenderObjectElement)
        result = element.renderObject;
      else
        element.visitChildren(visit);
    }
    visit(this);
    return result;
  }

  ...
      
  // debug：生命周期
  _ElementLifecycle _debugLifecycleState = _ElementLifecycle.initial;

  /// 访问所有的孩子
  /// 有孩子的子类element必须重载这个方法  
  /// typedef ElementVisitor = void Function(Element element);  
  void visitChildren(ElementVisitor visitor) { }

  /// 访问子element，直接调用 visitChildren
  @override
  void visitChildElements(ElementVisitor visitor) {
    visitChildren(visitor);
  }

  /// 更新孩子
  @protected
  Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
    assert(() {
      if (newWidget != null && newWidget.key is GlobalKey) {
        final GlobalKey key = newWidget.key;
        key._debugReserveFor(this);
      }
      return true;
    }());
    if (newWidget == null) {
      if (child != null)
        deactivateChild(child);
      return null;
    }
    if (child != null) {
      if (child.widget == newWidget) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        return child;
      }
      if (Widget.canUpdate(child.widget, newWidget)) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        child.update(newWidget);
        assert(child.widget == newWidget);
        assert(() {
          child.owner._debugElementWasRebuilt(child);
          return true;
        }());
        return child;
      }
      deactivateChild(child);
      assert(child._parent == null);
    }
    return inflateWidget(newWidget, newSlot);
  }

  /// 将这个element插入指定父节点的指定位置上
  @mustCallSuper
  void mount(Element parent, dynamic newSlot) {
    _parent = parent;
    _slot = newSlot;
    /// 深度+1
    _depth = _parent != null ? _parent.depth + 1 : 1;
    /// 标记active  
    _active = true;
    if (parent != null) // Only assign ownership if the parent is non-null
      _owner = parent.owner;
    if (widget.key is GlobalKey) {
      final GlobalKey key = widget.key;
      key._register(this);
    }
    _updateInheritance();
  }

  /// 更新widget
  @mustCallSuper
  void update(covariant Widget newWidget) {
    _widget = newWidget;
  }

  /// 更改一个在child列表中的位置
  @protected
  void updateSlotForChild(Element child, dynamic newSlot) {
    void visit(Element element) {
      element._updateSlot(newSlot);
      if (element is! RenderObjectElement)
        element.visitChildren(visit);
    }
    visit(child);
  }

  /// 更新位置  
  void _updateSlot(dynamic newSlot) {
    _slot = newSlot;
  }

  /// 更新深度  
  void _updateDepth(int parentDepth) {
    final int expectedDepth = parentDepth + 1;
    if (_depth < expectedDepth) {
      _depth = expectedDepth;
      visitChildren((Element child) {
        child._updateDepth(expectedDepth);
      });
    }
  }

  /// 将[renderObject]从渲染树里移除。
  ///
  /// 被[deactivateChild]调用。
  void detachRenderObject() {
    visitChildren((Element child) {
      child.detachRenderObject();
    });
    _slot = null;
  }

  /// 添加[renderObject] 到渲染树上指定的位置
  void attachRenderObject(dynamic newSlot) {
    assert(_slot == null);
    visitChildren((Element child) {
      child.attachRenderObject(newSlot);
    });
    _slot = newSlot;
  }

  Element _retakeInactiveElement(GlobalKey key, Widget newWidget) {
    final Element element = key._currentElement;
    if (element == null)
      return null;
    if (!Widget.canUpdate(element.widget, newWidget))
      return null;
    
    final Element parent = element._parent;
    if (parent != null) {
        parent.owner._debugTrackElementThatWillNeedToBeRebuiltDueToGlobalKeyShenanigans(
          parent,
          key,
        );
        return true;
      }());
      parent.forgetChild(element);
      parent.deactivateChild(element);
    }
    owner._inactiveElements.remove(element);
    return element;
  }

  /// 将widget扩展成element
  @protected
  Element inflateWidget(Widget newWidget, dynamic newSlot) {
    final Key key = newWidget.key;
    if (key is GlobalKey) {
      final Element newChild = _retakeInactiveElement(key, newWidget);
      if (newChild != null) {
        newChild._activateWithParent(this, newSlot);
        final Element updatedChild = updateChild(newChild, newWidget, newSlot);
        return updatedChild;
      }
    }
    newChild.mount(this, newSlot);
    return newChild;
  }

  ...
      
  /// deactivate一个孩子    
  @protected
  void deactivateChild(Element child) {
    child._parent = null;
    child.detachRenderObject();
    owner._inactiveElements.add(child); // this eventually calls child.deactivate()
  }

  /// 将child从element的children中移除
  /// eactivateChild之后会被调用来解除关联
  @protected
  void forgetChild(Element child);

  void _activateWithParent(Element parent, dynamic newSlot) {
    _updateDepth(_parent.depth);
    _activateRecursively(this);
    attachRenderObject(newSlot);
  }

  /// 递归地调用	activate
  static void _activateRecursively(Element element) {
    element.activate();
    element.visitChildren(_activateRecursively);
  }

  /// 使生命周期从inactive过渡到active
  @mustCallSuper
  void activate() {
    final bool hadDependencies = 
        (_dependencies != null && _dependencies.isNotEmpty) || 
        _hadUnsatisfiedDependencies;
    _active = true;
    _dependencies?.clear();
    _hadUnsatisfiedDependencies = false;
    _updateInheritance();
    if (_dirty)
      owner.scheduleBuildFor(this);
    if (hadDependencies)
      didChangeDependencies();
  }

  /// 使生命周期从active过渡到inactive
  @mustCallSuper
  void deactivate() {
    if (_dependencies != null && _dependencies.isNotEmpty) {
      for (InheritedElement dependency in _dependencies)
        dependency._dependents.remove(this);
    }
    _inheritedWidgets = null;
    _active = false;
    assert(() { _debugLifecycleState = _ElementLifecycle.inactive; return true; }());
  }

  @mustCallSuper
  void debugDeactivated() {
    assert(_debugLifecycleState == _ElementLifecycle.inactive);
  }

  /// 使生命周期从inactive过渡到defunct
  @mustCallSuper
  void unmount() {
    if (widget.key is GlobalKey) {
      final GlobalKey key = widget.key;
      key._unregister(this);
    }
  }

  @override
  RenderObject findRenderObject() => renderObject;

  /// 只有当renderObejct是RenderBox时才返回大小	
  @override
  Size get size {
    final RenderObject renderObject = findRenderObject();
    if (renderObject is RenderBox)
      return renderObject.size;
    return null;
  }

  /// 一个从类型到element实例的映射
  Map<Type, InheritedElement> _inheritedWidgets;
  /// 依赖集合
  Set<InheritedElement> _dependencies;
  bool _hadUnsatisfiedDependencies = false;

  /// 建立遗传关系
  /// 自身建立一个_dependencies（哈希集合），并将ancestor（InheritedElement ）加入其中
  /// 调用InheritedElement祖先的updateDependencies，在其——dependents中加入自身

  /// InheritedElement.setDependencies
  /// @protected
  /// void setDependencies(Element dependent, Object value) {
  ///   _dependents[dependent] = value;
  /// }
  @override
  InheritedWidget inheritFromElement(InheritedElement ancestor, { Object aspect }) {
    assert(ancestor != null);
    _dependencies ??= HashSet<InheritedElement>();
    _dependencies.add(ancestor);
    ancestor.updateDependencies(this, aspect);
    return ancestor.widget;
  }

  /// 返回指定类型的inheritFromElement持有的widget
  @override
  InheritedWidget inheritFromWidgetOfExactType(Type targetType, { Object aspect }) {
    /// 在_inheritedWidgets中通过<Type>查找对应的element
    final InheritedElement ancestor = _inheritedWidgets == null ?
        null : 
        _inheritedWidgets[targetType];
    if (ancestor != null) {
      assert(ancestor is InheritedElement);
      return inheritFromElement(ancestor, aspect: aspect);
    }
    _hadUnsatisfiedDependencies = true;
    return null;
  }

  /// 只在_inheritedWidgets中通过<Type>查找对应的element，不建立联系
  @override
  InheritedElement ancestorInheritedElementForWidgetOfExactType(Type targetType) {
    final InheritedElement ancestor = _inheritedWidgets == null ? null : _inheritedWidgets[targetType];
    return ancestor;
  }

  /// 获取父级的_inheritedWidgets（关系映射表）
  void _updateInheritance() {
    _inheritedWidgets = _parent?._inheritedWidgets;
  }

  /// 向上查找第一个指定类型的Widget
  @override
  Widget ancestorWidgetOfExactType(Type targetType) {
    Element ancestor = _parent;
    while (ancestor != null && ancestor.widget.runtimeType != targetType)
      ancestor = ancestor._parent;
    return ancestor?.widget;
  }

  /// 向上查找第一个指定类型的State
  @override
  State ancestorStateOfType(TypeMatcher matcher) {
    Element ancestor = _parent;
    while (ancestor != null) {
      if (ancestor is StatefulElement && matcher.check(ancestor.state))
        break;
      ancestor = ancestor._parent;
    }
    final StatefulElement statefulAncestor = ancestor;
    return statefulAncestor?.state;
  }

  /// 查找根指定类型的State
  @override
  State rootAncestorStateOfType(TypeMatcher matcher) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    Element ancestor = _parent;
    StatefulElement statefulAncestor;
    while (ancestor != null) {
      if (ancestor is StatefulElement && matcher.check(ancestor.state))
        statefulAncestor = ancestor;
      ancestor = ancestor._parent;
    }
    return statefulAncestor?.state;
  }

   /// 向上查找第一个指定类型的RenderObeject
  @override
  RenderObject ancestorRenderObjectOfType(TypeMatcher matcher) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    Element ancestor = _parent;
    while (ancestor != null) {
      if (ancestor is RenderObjectElement && matcher.check(ancestor.renderObject))
        break;
      ancestor = ancestor._parent;
    }
    final RenderObjectElement renderObjectAncestor = ancestor;
    return renderObjectAncestor?.renderObject;
  }

  /// 访问先祖elment
  @override
  void visitAncestorElements(bool visitor(Element element)) {
    Element ancestor = _parent;
    while (ancestor != null && visitor(ancestor))
      ancestor = ancestor._parent;
  }

  /// 依赖发生改变时的回调
  @mustCallSuper
  void didChangeDependencies() {
    markNeedsBuild();
  }

  ...
      
  bool get dirty => _dirty;
  bool _dirty = true;

  // 是否在 owner._dirtyElements 中
  bool _inDirtyList = false;

  ... 

  void markNeedsBuild() {
    if (dirty)
      return;
    _dirty = true;
    owner.scheduleBuildFor(this);
  }

  /// 当element被标记为dirty时，[BuildOwner]调用[BuildOwner.scheduleBuildFor]。
  /// 首次build时调用到[mount]
  /// 发生改变时调用到[update]。
  void rebuild() {
    Element debugPreviousBuildTarget;
    performRebuild();
  }

  /// 子类必须重载
  @protected
  void performRebuild();
}

```



## InheritedElement

```dart
class InheritedElement extends ProxyElement {
  InheritedElement(InheritedWidget widget) : super(widget);

  @override
  InheritedWidget get widget => super.widget;

  /// 在实例化 InheritedElement 时，实例化一个_dependents，供后代获取
  final Map<Element, Object> _dependents = HashMap<Element, Object>();

  @override
  void _updateInheritance() {
    final Map<Type, InheritedElement> incomingWidgets = _parent?._inheritedWidgets;
    if (incomingWidgets != null)
      _inheritedWidgets = HashMap<Type, InheritedElement>.from(incomingWidgets);
    else
      _inheritedWidgets = HashMap<Type, InheritedElement>();
      
    /// 将自身加入到类型遗传映射表中供后代获取  
    _inheritedWidgets[widget.runtimeType] = this;
  }

  ...

  /// 以某个element为键，返回其在_denpents中的映射值    
  @protected
  Object getDependencies(Element dependent) {
    return _dependents[dependent];
  }

  @protected
  void setDependencies(Element dependent, Object value) {
    _dependents[dependent] = value;
  }

  @protected
  void updateDependencies(Element dependent, Object aspect) {
    setDependencies(dependent, null);
  }

  /// 通知某个dependent，调用其didChangeDependencies
  @protected
  void notifyDependent(covariant InheritedWidget oldWidget, Element dependent) {
    dependent.didChangeDependencies();
  }
    
  /// 这个方法直接调用了notifyClients
  void updated(InheritedWidget oldWidget) {
    if (widget.updateShouldNotify(oldWidget))
      super.updated(oldWidget);
  }

  /// 通知所有的dependent
  @override
  void notifyClients(InheritedWidget oldWidget) {
    for (Element dependent in _dependents.keys) {
      notifyDependent(oldWidget, dependent);
    }
  }
}
```




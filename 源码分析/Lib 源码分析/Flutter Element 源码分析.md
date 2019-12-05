# Flutter Element 源码分析

[TOC]

## Element

```dart
/// Element的生命周期如下:
///
/// *框架通过调用[Widget.createElement]创建一个使用该widget作为初始配置的element
///
/// *框架调用[mount]将新创建的element添加到树上一个指定的父节点上的一个指定位置。[mount]方法负责扩展任
///  何子部件并调用其[attachRenderObject]，将任何关联的的RenderObject附加到渲染树上。
///
/// *此时，element被认为是“Active”，并且可能会出现在屏幕上
///
/// *在某个时候，父节点可能会决定更改配置此element的widget，例如因为父节点重新构建状态。当这种情况发生的
///  时，框架将调用新widget的[update]。新widget将始终具有与旧wiedget相同的[runtimeType]和[key]。
///  如果父节点希望更改[runtimeType]或[key]或这个widget在树中的位置，它可以通过卸载这个elemtent来实
///  现，并在此位置扩展一个新的widget
///
/// *在某个时候，祖先可能会决定删除某个element(或者中间祖先)，该祖先通过在自身调用[deactiveChild]来完
///  成。将中间祖先deactive会从渲染树中删除该element的RenderObject并添加这个element到[owner]的非活
///  动元素列表，从而导致框架对该element调用[deactive]
///
/// *此时，element被认为是“非活动的”，且不会出现屏幕上。元素能保持在非活动状态，直到当前动画帧的结束。在
///  动画的最后帧，任何仍然不活动的元素将被unmount
///
/// *如果element重新合并到树中(例如，因为它或他的祖先element有可重用的[GlobalKey])，框架将会从
///  [owner]的非活动elemten列表中删除元素并调用它的[activate]，并将element的渲染对象重新添加到渲染
///  树。此时，element再次被认为是“active”，并可能会出现在屏幕上
///
/// *如果element在当前动画结束帧时没有重新合并到树中，框架将调用该element的[unmount]
///
/// *此时，元素被认为是“defunct”，之后不再会合并到树中


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
    /// 移除旧的widget
    /// deactivate child
    if (newWidget == null) {
      if (child != null)
        deactivateChild(child);
      return null;
    }
    if (child != null) {
      /// Widget没有改变  
      if (child.widget == newWidget) {
        /// 如果位置可能发生了变更  
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        return child;
      }
      /// 只更新的widget的数据  
      if (Widget.canUpdate(child.widget, newWidget)) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        child.update(newWidget);
        return child;
      }
      /// 否则deactivate child
      deactivateChild(child);
    }
    /// 扩充新的element  
    return inflateWidget(newWidget, newSlot);
  }

  /// 将这个element插入指定父节点的指定位置上
  @mustCallSuper
  void mount(Element parent, dynamic newSlot) {
    _parent = parent;
    _slot = newSlot;
    /// 深度+1
    _depth = _parent != null ? _parent.depth + 1 : 1;
    /// 标记active为true  
    _active = true;
    /// 分配owner  
    if (parent != null)
      _owner = parent.owner;
    /// 注册global key  
    if (widget.key is GlobalKey) {
      final GlobalKey key = widget.key;
      key._register(this);
    }
    /// 更新遗传信息  
    _updateInheritance();
  }

  /// 更新widget，子类经常重载这个方法
  @mustCallSuper
  void update(covariant Widget newWidget) {
    _widget = newWidget;
  }

  /// 更改一个子element何其后代的位置
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
    visitChildren((Element child) {
      child.attachRenderObject(newSlot);
    });
    _slot = newSlot;
  }

  /// 激活一个 inactive 的 globalkey 所对应的element，并将其返回 
  Element _retakeInactiveElement(GlobalKey key, Widget newWidget) {
    final Element element = key._currentElement;
    if (element == null) return null;
    if (!Widget.canUpdate(element.widget, newWidget)) return null;
    
    final Element parent = element._parent;
    if (parent != null) {
      parent.forgetChild(element);
      parent.deactivateChild(element);
    }
    owner._inactiveElements.remove(element);
    return element;
  }

  /// 将 widget 扩展成 element
  @protected
  Element inflateWidget(Widget newWidget, dynamic newSlot) {
    final Key key = newWidget.key;
    /// 如果widget带有一个globalkey，
    /// 那么试图使用新的widget作为配置恢复原先widget对应的element  
    if (key is GlobalKey) {
      final Element newChild = _retakeInactiveElement(key, newWidget);
      /// 恢复成功  
      if (newChild != null) {
        newChild._activateWithParent(this, newSlot);
        /// 更新子树  
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
    owner._inactiveElements.add(child); // 这最终会调用child.deactivate()
  }

  /// 将child从element的children中移除
  /// eactivateChild之后会被调用来解除关联
  @protected
  void forgetChild(Element child);

  /// 给定一个新element作为parent，激活整个子树  
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
    /// 清除已有依赖  
    _dependencies?.clear();
    _hadUnsatisfiedDependencies = false;
    /// 更新遗传信息  
    _updateInheritance();
    /// 如果标记为dirty，那么构建一次  
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
  }

  /// 使生命周期从inactive过渡到defunct
  @mustCallSuper
  void unmount() {
    if (widget.key is GlobalKey) {
      final GlobalKey key = widget.key;
      key._unregister(this);
    }
  }

  /// 向下查找第一个RenderObject  
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
    
  /// 依赖（依赖于InheritedElement）集合
  Set<InheritedElement> _dependencies;
  bool _hadUnsatisfiedDependencies = false;

  /// 建立遗传关系
  /// 自身建立一个_dependencies（哈希集合），并将ancestor（InheritedElement ）加入集合中
  /// 调用InheritedElement祖先的updateDependencies，在其_dependents中加入自身

  /// InheritedElement.setDependencies
  /// @protected
  /// void setDependencies(Element dependent, Object value) {
  ///   _dependents[dependent] = value;
  /// }
  ///  
  @override
  InheritedWidget inheritFromElement(InheritedElement ancestor, { Object aspect }) {
    _dependencies ??= HashSet<InheritedElement>();
    /// 把遗传节点祖先加入自身的依赖  
    _dependencies.add(ancestor);
    /// 在遗传节点祖先的后代中加入自身  
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
      return inheritFromElement(ancestor, aspect: aspect);
    }
    /// 标记有未满足的遗传  
    _hadUnsatisfiedDependencies = true;
    return null;
  }

  /// 只在_inheritedWidgets中通过<Type>查找对应的element，不建立联系
  @override
  InheritedElement ancestorInheritedElementForWidgetOfExactType(Type targetType) {
    final InheritedElement ancestor = 
        _inheritedWidgets == null ? 
        null : 
        _inheritedWidgets[targetType];
    return ancestor;
  }

  /// 引用父级的_inheritedWidgets（关系映射表）
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

  /// 依赖发生改变时的回调，标记需要重建
  @mustCallSuper
  void didChangeDependencies() {
    markNeedsBuild();
  }

  ...
      
  /// dirty标记，标记这个element是否需要重建    
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
    performRebuild();
  }

  /// 子类必须重载
  @protected
  void performRebuild();
}

```



## ComponeElement

```dart
abstract class ComponentElement extends Element {
  ComponentElement(Widget widget) : super(widget);

  Element _child;

  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _firstBuild();
  }

  void _firstBuild() {
    rebuild();
  }

  /// 当首次建立被[mount]，或需要更新时被[rebuild]自动调用
  @override
  void performRebuild() {
    if (!kReleaseMode && debugProfileBuildsEnabled)
      Timeline.startSync('${widget.runtimeType}',  arguments: 
                         timelineWhitelistArguments);

    Widget built;
    try {
      built = build();
      debugWidgetBuilderValue(widget, built);
    } catch (e, stack) {
      ...
    } finally {
      _dirty = false;
      ...
    }
    try {
      _child = updateChild(_child, built, slot);
      ...
    } catch (e, stack) {
      ...
      _child = updateChild(null, built, slot);
    }

    if (!kReleaseMode && debugProfileBuildsEnabled)
      Timeline.finishSync();
  }

  /// 子类必须重载这个方法
  @protected
  Widget build();

  @override
  void visitChildren(ElementVisitor visitor) {
    /// 如果有孩子才访问  
    if (_child != null) visitor(_child);
  }

  @override
  void forgetChild(Element child) {
    _child = null;
  }
}
```



## ProxyElement

```dart
abstract class ProxyElement extends ComponentElement {
  /// Initializes fields for subclasses.
  ProxyElement(ProxyWidget widget) : super(widget);

  @override
  ProxyWidget get widget => super.widget;

  @override
  Widget build() => widget.child;

  /// 每次update时，会通知listener  
  @override
  void update(ProxyWidget newWidget) {
    final ProxyWidget oldWidget = widget;
    super.update(newWidget);
    updated(oldWidget);
    _dirty = true;
    rebuild();
  }

  /// widget发生改变时调用
  @protected
  void updated(covariant ProxyWidget oldWidget) {
    notifyClients(oldWidget);
  }

  @protected
  void notifyClients(covariant ProxyWidget oldWidget);
}
```





## ParentDataElement

```dart
class ParentDataElement<T extends RenderObjectWidget> extends ProxyElement {
  ParentDataElement(ParentDataWidget<T> widget) : super(widget);

  @override
  ParentDataWidget<T> get widget => super.widget;

  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
  }

  void _applyParentData(ParentDataWidget<T> widget) {
    void applyParentDataToChild(Element child) {
      if (child is RenderObjectElement) {
        child._updateParentData(widget);
      } else {
        child.visitChildren(applyParentDataToChild);
      }
    }
    visitChildren(applyParentDataToChild);
  }

  /// Calls [ParentDataWidget.applyParentData] on the given widget, passing it
  /// the [RenderObject] whose parent data this element is ultimately
  /// responsible for.
  ///
  /// 这个方法允许改变[RenderObject.parentData]而不触发重建
  /// 这样做通常不是很好，当以下两个情况确实值得的：
  ///
  ///  * 处于build和layout阶段，[ParentData]不会影响build和layout，并且无法在build和layout之前
  ///    确定数据的值 （例如，它取决于一个孩子的布局）
  ///
  ///  * 处于paint和layout阶段，[ParentData]不会影响paint和layout，并且无法在paint和layout之前
  ///    确定数据的值 （例如，它取决合成）
  ///
  /// 在下一次构建时，会使用更新的数据作为配置
  ///
  ///
  /// The new widget must have the same child as the current widget.
  ///
  /// 使用它的一个例子是[AutomaticKeepAlive]widget。如果它在生成它的一个后代时收到通知，告诉它
  /// 它的子widget必须保持alive，它将应用一个[KeepAlive]widget
  /// 这是安全的，因为根据定义，孩子已经保持alive,因此这不会改变父节点的帧行为。
  /// 它比仅仅为了更新[KeepAlive]widget而请求额外的帧更有效。
  void applyWidgetOutOfTurn(ParentDataWidget<T> newWidget) {
    _applyParentData(newWidget);
  }

  @override
  void notifyClients(ParentDataWidget<T> oldWidget) {
    _applyParentData(widget);
  }
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
    /// incomminWidget是对父节点遗传映射表的引用  
    final Map<Type, InheritedElement> incomingWidgets = _parent?._inheritedWidgets;
    if (incomingWidgets != null)
      /// 拷贝一份  
      _inheritedWidgets = HashMap<Type, InheritedElement>.from(incomingWidgets);
    else
      /// 空表  
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

  /// 将一个element作为dependt  
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



## StatelessElement

```dart
class StatelessElement extends ComponentElement {
  StatelessElement(StatelessWidget widget) : super(widget);

  @override
  StatelessWidget get widget => super.widget;

  @override
  Widget build() => widget.build(this);

  @override
  void update(StatelessWidget newWidget) {
    super.update(newWidget);
    assert(widget == newWidget);
    _dirty = true;
    rebuild();
  }
}
```



## RenderObjectElment

```dart
/// ### Slots
///
/// 每个子[元素]对应一个[RenderObject]，它应该作为这个元素的render对象的子renderObjet
/// 但是，元素的直接子元素可能不是最终生成它们对应的实际[RenderObject]的子元素。例如，
/// [StatelessElement] ([StatelessWidget]的元，简单地对应于[RenderObject]，它的
/// [StatelessWidget.build]返回的元素)
///
/// 因此，每个子节点都被分配了一个_slot_token。这是一个标识符，对这个[RenderObjectElement]节点是私有
/// 的。当其后代最终产生(RenderObject)并准备将它附加到这个节点的RenderObject,它将_slots_token传回到
/// 这个节点,这使得这个节点能够轻易地确定这个RenderObject在父RenderObject中相对其他孩子的位置。

abstract class RenderObjectElement extends Element {
  /// Creates an element that uses the given widget as its configuration.
  RenderObjectElement(RenderObjectWidget widget) : super(widget);

  @override
  RenderObjectWidget get widget => super.widget;

  /// The underlying [RenderObject] for this element.
  @override
  RenderObject get renderObject => _renderObject;
  RenderObject _renderObject;

  RenderObjectElement _ancestorRenderObjectElement;

  RenderObjectElement _findAncestorRenderObjectElement() {
    Element ancestor = _parent;
    while (ancestor != null && ancestor is! RenderObjectElement)
      ancestor = ancestor._parent;
    return ancestor;
  }

  ParentDataElement<RenderObjectWidget> _findAncestorParentDataElement() {
    Element ancestor = _parent;
    while (ancestor != null && ancestor is! RenderObjectElement) {
      if (ancestor is ParentDataElement<RenderObjectWidget>)
        return ancestor;
      ancestor = ancestor._parent;
    }
    return null;
  }

  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _renderObject = widget.createRenderObject(this);
    assert(() { _debugUpdateRenderObjectOwner(); return true; }());
    assert(_slot == newSlot);
    attachRenderObject(newSlot);
    _dirty = false;
  }

  @override
  void update(covariant RenderObjectWidget newWidget) {
    super.update(newWidget);
    assert(widget == newWidget);
    assert(() { _debugUpdateRenderObjectOwner(); return true; }());
    widget.updateRenderObject(this, renderObject);
    _dirty = false;
  }

  void _debugUpdateRenderObjectOwner() {
    assert(() {
      _renderObject.debugCreator = DebugCreator(this);
      return true;
    }());
  }

  @override
  void performRebuild() {
    widget.updateRenderObject(this, renderObject);
    _dirty = false;
  }

  /// Updates the children of this element to use new widgets.
  ///
  /// Attempts to update the given old children list using the given new
  /// widgets, removing obsolete elements and introducing new ones as necessary,
  /// and then returns the new child list.
  ///
  /// During this function the `oldChildren` list must not be modified. If the
  /// caller wishes to remove elements from `oldChildren` re-entrantly while
  /// this function is on the stack, the caller can supply a `forgottenChildren`
  /// argument, which can be modified while this function is on the stack.
  /// Whenever this function reads from `oldChildren`, this function first
  /// checks whether the child is in `forgottenChildren`. If it is, the function
  /// acts as if the child was not in `oldChildren`.
  ///
  /// This function is a convenience wrapper around [updateChild], which updates
  /// each individual child. When calling [updateChild], this function uses the
  /// previous element as the `newSlot` argument.
  @protected
  List<Element> updateChildren(List<Element> oldChildren, List<Widget> newWidgets, { Set<Element> forgottenChildren }) {
    assert(oldChildren != null);
    assert(newWidgets != null);

    Element replaceWithNullIfForgotten(Element child) {
      return forgottenChildren != null && forgottenChildren.contains(child) ? null : child;
    }

    // This attempts to diff the new child list (newWidgets) with
    // the old child list (oldChildren), and produce a new list of elements to
    // be the new list of child elements of this element. The called of this
    // method is expected to update this render object accordingly.

    // The cases it tries to optimize for are:
    //  - the old list is empty
    //  - the lists are identical
    //  - there is an insertion or removal of one or more widgets in
    //    only one place in the list
    // If a widget with a key is in both lists, it will be synced.
    // Widgets without keys might be synced but there is no guarantee.

    // The general approach is to sync the entire new list backwards, as follows:
    // 1. Walk the lists from the top, syncing nodes, until you no longer have
    //    matching nodes.
    // 2. Walk the lists from the bottom, without syncing nodes, until you no
    //    longer have matching nodes. We'll sync these nodes at the end. We
    //    don't sync them now because we want to sync all the nodes in order
    //    from beginning to end.
    // At this point we narrowed the old and new lists to the point
    // where the nodes no longer match.
    // 3. Walk the narrowed part of the old list to get the list of
    //    keys and sync null with non-keyed items.
    // 4. Walk the narrowed part of the new list forwards:
    //     * Sync non-keyed items with null
    //     * Sync keyed items with the source if it exists, else with null.
    // 5. Walk the bottom of the list again, syncing the nodes.
    // 6. Sync null with any items in the list of keys that are still
    //    mounted.

    int newChildrenTop = 0;
    int oldChildrenTop = 0;
    int newChildrenBottom = newWidgets.length - 1;
    int oldChildrenBottom = oldChildren.length - 1;

    final List<Element> newChildren = oldChildren.length == newWidgets.length ?
        oldChildren : List<Element>(newWidgets.length);

    Element previousChild;

    // Update the top of the list.
    while ((oldChildrenTop <= oldChildrenBottom) && (newChildrenTop <= newChildrenBottom)) {
      final Element oldChild = replaceWithNullIfForgotten(oldChildren[oldChildrenTop]);
      final Widget newWidget = newWidgets[newChildrenTop];
      assert(oldChild == null || oldChild._debugLifecycleState == _ElementLifecycle.active);
      if (oldChild == null || !Widget.canUpdate(oldChild.widget, newWidget))
        break;
      final Element newChild = updateChild(oldChild, newWidget, previousChild);
      assert(newChild._debugLifecycleState == _ElementLifecycle.active);
      newChildren[newChildrenTop] = newChild;
      previousChild = newChild;
      newChildrenTop += 1;
      oldChildrenTop += 1;
    }

    // Scan the bottom of the list.
    while ((oldChildrenTop <= oldChildrenBottom) && (newChildrenTop <= newChildrenBottom)) {
      final Element oldChild = replaceWithNullIfForgotten(oldChildren[oldChildrenBottom]);
      final Widget newWidget = newWidgets[newChildrenBottom];
      assert(oldChild == null || oldChild._debugLifecycleState == _ElementLifecycle.active);
      if (oldChild == null || !Widget.canUpdate(oldChild.widget, newWidget))
        break;
      oldChildrenBottom -= 1;
      newChildrenBottom -= 1;
    }

    // Scan the old children in the middle of the list.
    final bool haveOldChildren = oldChildrenTop <= oldChildrenBottom;
    Map<Key, Element> oldKeyedChildren;
    if (haveOldChildren) {
      oldKeyedChildren = <Key, Element>{};
      while (oldChildrenTop <= oldChildrenBottom) {
        final Element oldChild = replaceWithNullIfForgotten(oldChildren[oldChildrenTop]);
        assert(oldChild == null || oldChild._debugLifecycleState == _ElementLifecycle.active);
        if (oldChild != null) {
          if (oldChild.widget.key != null)
            oldKeyedChildren[oldChild.widget.key] = oldChild;
          else
            deactivateChild(oldChild);
        }
        oldChildrenTop += 1;
      }
    }

    // Update the middle of the list.
    while (newChildrenTop <= newChildrenBottom) {
      Element oldChild;
      final Widget newWidget = newWidgets[newChildrenTop];
      if (haveOldChildren) {
        final Key key = newWidget.key;
        if (key != null) {
          oldChild = oldKeyedChildren[key];
          if (oldChild != null) {
            if (Widget.canUpdate(oldChild.widget, newWidget)) {
              // we found a match!
              // remove it from oldKeyedChildren so we don't unsync it later
              oldKeyedChildren.remove(key);
            } else {
              // Not a match, let's pretend we didn't see it for now.
              oldChild = null;
            }
          }
        }
      }
      assert(oldChild == null || Widget.canUpdate(oldChild.widget, newWidget));
      final Element newChild = updateChild(oldChild, newWidget, previousChild);
      assert(newChild._debugLifecycleState == _ElementLifecycle.active);
      assert(oldChild == newChild || oldChild == null || oldChild._debugLifecycleState != _ElementLifecycle.active);
      newChildren[newChildrenTop] = newChild;
      previousChild = newChild;
      newChildrenTop += 1;
    }

    // We've scanned the whole list.
    assert(oldChildrenTop == oldChildrenBottom + 1);
    assert(newChildrenTop == newChildrenBottom + 1);
    assert(newWidgets.length - newChildrenTop == oldChildren.length - oldChildrenTop);
    newChildrenBottom = newWidgets.length - 1;
    oldChildrenBottom = oldChildren.length - 1;

    // Update the bottom of the list.
    while ((oldChildrenTop <= oldChildrenBottom) && (newChildrenTop <= newChildrenBottom)) {
      final Element oldChild = oldChildren[oldChildrenTop];
      assert(replaceWithNullIfForgotten(oldChild) != null);
      assert(oldChild._debugLifecycleState == _ElementLifecycle.active);
      final Widget newWidget = newWidgets[newChildrenTop];
      assert(Widget.canUpdate(oldChild.widget, newWidget));
      final Element newChild = updateChild(oldChild, newWidget, previousChild);
      assert(newChild._debugLifecycleState == _ElementLifecycle.active);
      assert(oldChild == newChild || oldChild == null || oldChild._debugLifecycleState != _ElementLifecycle.active);
      newChildren[newChildrenTop] = newChild;
      previousChild = newChild;
      newChildrenTop += 1;
      oldChildrenTop += 1;
    }

    // Clean up any of the remaining middle nodes from the old list.
    if (haveOldChildren && oldKeyedChildren.isNotEmpty) {
      for (Element oldChild in oldKeyedChildren.values) {
        if (forgottenChildren == null || !forgottenChildren.contains(oldChild))
          deactivateChild(oldChild);
      }
    }

    return newChildren;
  }

  @override
  void deactivate() {
    super.deactivate();
    assert(!renderObject.attached,
      'A RenderObject was still attached when attempting to deactivate its '
      'RenderObjectElement: $renderObject');
  }

  @override
  void unmount() {
    super.unmount();
    assert(!renderObject.attached,
      'A RenderObject was still attached when attempting to unmount its '
      'RenderObjectElement: $renderObject');
    widget.didUnmountRenderObject(renderObject);
  }

  void _updateParentData(ParentDataWidget<RenderObjectWidget> parentData) {
    parentData.applyParentData(renderObject);
  }

  @override
  void _updateSlot(dynamic newSlot) {
    assert(slot != newSlot);
    super._updateSlot(newSlot);
    assert(slot == newSlot);
    _ancestorRenderObjectElement.moveChildRenderObject(renderObject, slot);
  }

  @override
  void attachRenderObject(dynamic newSlot) {
    assert(_ancestorRenderObjectElement == null);
    _slot = newSlot;
    _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
    _ancestorRenderObjectElement?.insertChildRenderObject(renderObject, newSlot);
    final ParentDataElement<RenderObjectWidget> parentDataElement = _findAncestorParentDataElement();
    if (parentDataElement != null)
      _updateParentData(parentDataElement.widget);
  }

  @override
  void detachRenderObject() {
    if (_ancestorRenderObjectElement != null) {
      _ancestorRenderObjectElement.removeChildRenderObject(renderObject);
      _ancestorRenderObjectElement = null;
    }
    _slot = null;
  }

  /// Insert the given child into [renderObject] at the given slot.
  ///
  /// The semantics of `slot` are determined by this element. For example, if
  /// this element has a single child, the slot should always be null. If this
  /// element has a list of children, the previous sibling is a convenient value
  /// for the slot.
  @protected
  void insertChildRenderObject(covariant RenderObject child, covariant dynamic slot);

  /// Move the given child to the given slot.
  ///
  /// The given child is guaranteed to have [renderObject] as its parent.
  ///
  /// The semantics of `slot` are determined by this element. For example, if
  /// this element has a single child, the slot should always be null. If this
  /// element has a list of children, the previous sibling is a convenient value
  /// for the slot.
  ///
  /// This method is only ever called if [updateChild] can end up being called
  /// with an existing [Element] child and a `slot` that differs from the slot
  /// that element was previously given. [MultiChildRenderObjectElement] does this,
  /// for example. [SingleChildRenderObjectElement] does not (since the `slot` is
  /// always null). An [Element] that has a specific set of slots with each child
  /// always having the same slot (and where children in different slots are never
  /// compared against each other for the purposes of updating one slot with the
  /// element from another slot) would never call this.
  @protected
  void moveChildRenderObject(covariant RenderObject child, covariant dynamic slot);

  /// Remove the given child from [renderObject].
  ///
  /// The given child is guaranteed to have [renderObject] as its parent.
  @protected
  void removeChildRenderObject(covariant RenderObject child);

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(DiagnosticsProperty<RenderObject>('renderObject', renderObject, defaultValue: null));
  }
}
```



## StatefulElement

```dart
class StatefulElement extends ComponentElement {
  StatefulElement(StatefulWidget widget)
      : _state = widget.createState(),
        super(widget) 
  {
    /// 为State提供自身和widget的引用        
    _state._element = this;
    _state._widget = widget;
  }

  @override
  Widget build() => state.build(this);

  State<StatefulWidget> get state => _state;
  State<StatefulWidget> _state;

  @override
  void reassemble() {
    state.reassemble();
    super.reassemble();
  }

  @override
  void _firstBuild() {
    try {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(true);
      /// 调用state的initState  
      final dynamic debugCheckForReturnedFuture = _state.initState() as dynamic;
    } finally {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(false);
    }
    /// 调用didChangeDependencies  
    _state.didChangeDependencies();
    super._firstBuild();
  }

  @override
  void update(StatefulWidget newWidget) {
    super.update(newWidget);
    final StatefulWidget oldWidget = _state._widget;
    // Notice that we mark ourselves as dirty before calling didUpdateWidget to
    // let authors call setState from within didUpdateWidget without triggering
    // asserts.
    _dirty = true;
    _state._widget = widget;
    try {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(true);
      final dynamic debugCheckForReturnedFuture 
          = _state.didUpdateWidget(oldWidget) as dynamic;
    } finally {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(false);
    }
    rebuild();
  }

  @override
  void activate() {
    super.activate();
    markNeedsBuild();
  }

  @override
  void deactivate() {
    _state.deactivate();
    super.deactivate();
  }

  @override
  void unmount() {
    super.unmount();
    /// 释放State的资源，并解除引用  
    _state.dispose();
    _state._element = null;
    _state = null;
  }

  @override
  InheritedWidget inheritFromElement(Element ancestor, { Object aspect }) {
    return super.inheritFromElement(ancestor, aspect: aspect);
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    _state.didChangeDependencies();
  }

  }
}
```



## LeafRenderObjectElement

```dart
/// An [Element] that uses a [LeafRenderObjectWidget] as its configuration.
class LeafRenderObjectElement extends RenderObjectElement {
  /// Creates an element that uses the given widget as its configuration.
  LeafRenderObjectElement(LeafRenderObjectWidget widget) : super(widget);

  /// 因为没有孩子，所以以下方法均不能使用	
  @override
  void forgetChild(Element child) {
    assert(false);
  }

  @override
  void insertChildRenderObject(RenderObject child, dynamic slot) {
    assert(false);
  }

  @override
  void moveChildRenderObject(RenderObject child, dynamic slot) {
    assert(false);
  }

  @override
  void removeChildRenderObject(RenderObject child) {
    assert(false);
  }

}
```





## SingleChildRenderObjectElement

```dart
class SingleChildRenderObjectElement extends RenderObjectElement{
  SingleChildRenderObjectElement(SingleChildRenderObjectWidget widget) : super(widget);

  @override
  SingleChildRenderObjectWidget get widget => super.widget;

  Element _child;

  @override
  void visitChildren(ElementVisitor visitor) {
    if (_child != null)
      visitor(_child);
  }

  @override
  void forgetChild(Element child) {
    _child = null;
  }

  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _child = updateChild(_child, widget.child, null);
  }

  @override
  void update(SingleChildRenderObjectWidget newWidget) {
    super.update(newWidget);
    assert(widget == newWidget);
    _child = updateChild(_child, widget.child, null);
  }

  @override
  void insertChildRenderObject(RenderObject child, dynamic slot) {
    final RenderObjectWithChildMixin<RenderObject> renderObject = this.renderObject;
    renderObject.child = child;
  }

  /// 只有一个孩子，无法移动  
  @override
  void moveChildRenderObject(RenderObject child, dynamic slot) {
    assert(false);
  }

  @override
  void removeChildRenderObject(RenderObject child) {
    final RenderObjectWithChildMixin<RenderObject> renderObject = this.renderObject;
    renderObject.child = null;
  }
}
```



## MultiChildRenderObjectElment

```dart
class MultiChildRenderObjectElement extends RenderObjectElement {
  /// Creates an element that uses the given widget as its configuration.
  MultiChildRenderObjectElement(MultiChildRenderObjectWidget widget)
    : assert(!debugChildrenHaveDuplicateKeys(widget, widget.children)),
      super(widget);

  @override
  MultiChildRenderObjectWidget get widget => super.widget;

  /// 过滤后的子element列表
  @protected
  @visibleForTesting
  Iterable<Element> get children => 
      _children.where((Element child) => !_forgottenChildren.contains(child));

  /// 真正的子element列表  
  List<Element> _children;
  /// 使用HashSet（查找时间O（１）），来过滤已被移除的孩子
  final Set<Element> _forgottenChildren = HashSet<Element>();

  @override
  void insertChildRenderObject(RenderObject child, Element slot) {
    final ContainerRenderObjectMixin<
        	RenderObject, 
      		ContainerParentDataMixin<RenderObject>
       　 > renderObject = this.renderObject;

    renderObject.insert(child, after: slot?.renderObject);

  }

  @override
  void moveChildRenderObject(RenderObject child, dynamic slot) {
    final ContainerRenderObjectMixin<
        	RenderObject, 
      		ContainerParentDataMixin<RenderObject>
          > renderObject = this.renderObject;

    renderObject.move(child, after: slot?.renderObject);

  }

  @override
  void removeChildRenderObject(RenderObject child) {
    final ContainerRenderObjectMixin<
        	RenderObject, 
      		ContainerParentDataMixin<RenderObject>
         > renderObject = this.renderObject;

    renderObject.remove(child);

  }

  @override
  void visitChildren(ElementVisitor visitor) {
    for (Element child in _children) {
      if (!_forgottenChildren.contains(child))
        visitor(child);
    }
  }

  @override
  void forgetChild(Element child) {
    _forgottenChildren.add(child);
  }

  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _children = List<Element>(widget.children.length);
    Element previousChild;
     
    for (int i = 0; i < _children.length; i += 1) {
      final Element newChild = inflateWidget(widget.children[i], previousChild);
      _children[i] = newChild;
      previousChild = newChild;
    }
  }

  @override
  void update(MultiChildRenderObjectWidget newWidget) {
    super.update(newWidget);
    _children = updateChildren(
        _children, 
        widget.children,
        forgottenChildren: _forgottenChildren
    );
    _forgottenChildren.clear();
  }
}
```


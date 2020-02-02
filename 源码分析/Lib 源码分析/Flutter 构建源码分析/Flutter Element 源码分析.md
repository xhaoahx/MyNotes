# Flutter Element 源码分析

[TOC]

## Element

```dart
/// Element的生命周期如下:
///
/// *框架通过调用 [Widget.createElement] 创建一个使用该 widget 作为初始配置的 element
///
/// *框架调用 [mount] 将新创建的 element 添加到树上一个指定的父节点上的一个指定位置(slot)。
///  [mount] 方法负责扩展任何子 element（即递归地构建子树）并调用其 [attachRenderObject]，将任何关联
///  的 RenderObject 附加到渲染对象树上。
///
/// *此时，element被认为是“Active”，并且可能会出现在屏幕上
///
/// *在某个时候，父节点可能会决定更改配置此 element 的 widget，例如因为父节点重新构建状态。当这种情况发
///  生的时侯，框架将调用新 widget 的 [update]。新 widget 将始终具有与旧 wiedget 相同的 
///  [runtimeType] 和 [key]。 如果父节点希望更改 [runtimeType] 或 [key] 或这个 widget 在树中的位
///  置，它可以通过卸载这个 elemtent 来实现，并在此位置将一个新的 widget 扩展成 element
///
/// *在某个时候，祖先可能会决定删除某个 element (或者中间祖先)，该祖先通过在自身调用 [deactiveChild] 来
///  完成卸载。将中间祖先 deactive 会从渲染对象树中删除该 element 的 RenderObject 并添加这个 
///  element 到[BuildOwner]的非活动元素列表，从而导致框架对该 element 调用 [deactive]
///
/// *此时，element 被认为是“inActive”，且不会出现屏幕上。一个 element 能保持在非活动状态，直到当前动画
///  帧的结束(这里结束是指 drawFrame 回调完成之后，finalizeTree 会调用所有非活动的 element.unmount 
///  方法)。在动画帧的最后，任何仍然不活动的 element 将被 unmount
///
/// *如果 element 被重新合并到树中(例如，因为它或他的祖先 element 有可重用的 [GlobalKey] )，框架将会
///  从 [BuildOwner] 的非活动 elemtent 列表中删除此 element 并调用它的 [activate]，并将 element 的 
///  RenderObject 重新添加到渲染对象树中。此时，element 再次被认为是 “active”，并可能会出现在屏幕上
///
/// *如果 element 在当前动画结束帧时没有重新合并到树中，框架将调用该 element 的 [unmount]
///
/// *此时，元素被认为是“defunct”，之后不再会合并到树中
abstract class Element extends DiagnosticableTree implements BuildContext {
  /// widget 通常重载 createElement 来创建 element（即 "creatElement() => XXXElement(this);" ）、
  /// 使 element 持有 widget  
  Element(Widget widget) : _widget = widget;

  /// 父 element  
  Element _parent;

  /// 判断相等的唯一方式：是否引用了同一 element 
  @override
  bool operator ==(Object other) => identical(this, other);

  @override
  int get hashCode => _cachedHash;
  final int _cachedHash = _nextHashCode = (_nextHashCode + 1) % 0xffffff;// 16777215
  static int _nextHashCode = 1;

  /// 父节点为子节点设置的位置
  ///
  /// 如果子类 element 只有一个孩子，那么这个域为 null
  /// 多孩子的情况可以参考 MultiChildRenderObjectElement 的实现 
    
  /// Slots
  ///
  /// 每个子 [element] 对应一个 [RenderObject]，它应该作为这个 element 的 renderObject 的子 
  /// renderObjet。但是，element 的当前的 child 列表可能不会最终对应 RenderObject 的 element 
  /// 例如，[StatelessElement] ([StatelessWidget] 的 element)，简单地对应于它 build 方法返回的
  /// Element 子树的最上层的 [RenderObject]
  ///
  /// 因此，每个子节点都被分配了一个 _slot_token。这是一个标识符，对这个 [RenderObjectElement] 节点
  /// 是私有的。当其后代最终产生(RenderObject)并准备将它附加到这个节点的 RenderObject,它将
  /// _slots_token 传回到这个节点,这使得这个节点能够轻易地确定这个 RenderObject 在父 RenderObject 
  /// 中相对其他孩子的位置。
  dynamic get slot => _slot;
  dynamic _slot;

  /// element 在树中的深度
  int get depth => _depth;
  int _depth;

  /// 排序，按照深度小大，是否标记 dirty 先后排序  
  /// 即深度小的会被再在前面。如果深度相同，那么被标记 dirty 的会被排在前面  
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

  /// 这个 element 对应  widget
  @override
  Widget get widget => _widget;
  Widget _widget;

  /// 控制 element 生命周期的 BuildOwner
  @override
  BuildOwner get owner => _owner;
  BuildOwner _owner;

  /// 是否处于活动状态  
  bool _active = false;

  /// 热重载相关
  /// 这个方法被开发工具调用，会整颗 element 树的 markNeedsBuild 方法被调用，即重建整颗树  
  @mustCallSuper
  @protected
  void reassemble() {
    markNeedsBuild();
    visitChildren((Element child) {
      child.reassemble();
    });
  }

  /// 向下查找第一个渲染对象，作为自身对应的 RenderObject
  RenderObject get renderObject {
    RenderObject result;
    void visit(Element element) {
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
  /// 有孩子的子类 element 必须重载这个方法  
  /// typedef ElementVisitor = void Function(Element element);  
  void visitChildren(ElementVisitor visitor) { }

  /// 访问子 element，直接调用 visitChildren
  @override
  void visitChildElements(ElementVisitor visitor) {
    visitChildren(visitor);
  }

  /// 更新孩子
  @protected
  Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
    /// 移除旧的 widget，或是没有子节点，前者会导致子节点的 unmount 被调用
    if (newWidget == null) {
      if (child != null)
        deactivateChild(child);
      return null;
    }
    if (child != null) {
      /// Widget 的引用没有发生改变  
      if (child.widget == newWidget) {
        /// 如果位置可能发生了变更，那么只需要更新 slot  
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        return child;
      }
      /// 如果用新的 widget 去替换 旧的 widget  
      if (Widget.canUpdate(child.widget, newWidget)) {
        /// 如果位置可能发生了变更，那么只需要更新 slot   
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        /// 用新的 widget 来替换旧的 widget，并更新的对应的 element 或者 RenderObject 数据  
        child.update(newWidget);
        return child;
      }
      /// 否则 deactivate child
      deactivateChild(child);
    }
    /// 将 newWidet 扩充成新的 element 做为孩子  
    return inflateWidget(newWidget, newSlot);
  }

  /// 将这个 element 插入指定父节点的指定位置上，并建立对父级的引用关系，并更新必要的信息
  @mustCallSuper
  void mount(Element parent, dynamic newSlot) {
    _parent = parent;
    _slot = newSlot;
    /// 深度 + 1
    _depth = _parent != null ? _parent.depth + 1 : 1;
    /// 标记 active 为 true  
    _active = true;
    /// 分配 owner  
    if (parent != null)
      _owner = parent.owner;
    /// 注册 global key  
    if (widget.key is GlobalKey) {
      final GlobalKey key = widget.key;
      key._register(this);
    }
    /// 更新遗传信息  
    _updateInheritance();
  }

  /// 更新 widget，子类经常重载这个方法。
  ///
  /// 在 StatefulWidget 对应的 State 调用自身的 setState 后，框架会对标记为 dirty 的所有的节点进行 rebuild
  /// 此时，如果某个 widget 可以更新（而不是 rebuild，也就是说其对应的  key 和 runtimeType 对应不变的话），
  /// 则会调用此方法进行更新
  @mustCallSuper
  void update(covariant Widget newWidget) {
    _widget = newWidget;
  }

  /// 更改一个子 element 何其后代的位置
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

  /// 将 [renderObject] 从渲染树里移除。
  ///
  /// 被 [deactivateChild] 调用。
  void detachRenderObject() {
    visitChildren((Element child) {
      child.detachRenderObject();
    });
    _slot = null;
  }

  /// 添加 [renderObject] 到渲染树上指定的位置
  void attachRenderObject(dynamic newSlot) {
    visitChildren((Element child) {
      child.attachRenderObject(newSlot);
    });
    _slot = newSlot;
  }

  /// 重新获得一个 globalkey 所对应的非活动 element，并将其返回 
  Element _retakeInactiveElement(GlobalKey key, Widget newWidget) {
    final Element element = key._currentElement;
    if (element == null) return null;
    if (!Widget.canUpdate(element.widget, newWidget)) return null;
    
    final Element parent = element._parent;
    if (parent != null) {
      parent.forgetChild(element);
      parent.deactivateChild(element);
    }
    /// 将找到的 element 从 buildOwner 的非活动 element 列表中移除
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
      
  /// deactivate 一个孩子    
  @protected
  void deactivateChild(Element child) {
    child._parent = null;
    child.detachRenderObject();
    owner._inactiveElements.add(child); // 这最终会调用child.deactivate
  }

  /// 将 child 从 element 的 children 中移除
  /// deactivateChild 之后会被调用来解除关联
  @protected
  void forgetChild(Element child);

  /// 给定一个新 element 作为 parent，激活整个子树 
  /// 通常，这个方法会在 _retakeInactiveElement 之后调用，用于将 element 及其子树移栽到树中的另一个
  /// 位置 
  void _activateWithParent(Element parent, dynamic newSlot) {
    _updateDepth(_parent.depth);
    _activateRecursively(this);
    attachRenderObject(newSlot);
  }

  /// 递归地调用	activate，激活给定 element 的子树
  static void _activateRecursively(Element element) {
    element.activate();
    element.visitChildren(_activateRecursively);
  }

  /// 使生命周期从 inactive 过渡到 active
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

  /// 使生命周期从 active 过渡到 inactive
  /// 如果有依赖的话，会在依赖的节点的依赖着列表中移除自身  
  @mustCallSuper
  void deactivate() {
    if (_dependencies != null && _dependencies.isNotEmpty) {
      for (InheritedElement dependency in _dependencies)
        dependency._dependents.remove(this);
    }
    _inheritedWidgets = null;
    _active = false;
  }

  /// 使生命周期从inactive 过渡到 defunct
  @mustCallSuper
  void unmount() {
    /// 如果这个 element 对应了一个 GlobalKey 的话，需要注销  
    if (widget.key is GlobalKey) {
      final GlobalKey key = widget.key;
      key._unregister(this);
    }
  }

  /// 向下查找第一个 RenderObject  
  @override
  RenderObject findRenderObject() => renderObject;

  /// 只有当 renderObejct 是 RenderBox 时才返回大小	
  @override
  Size get size {
    final RenderObject renderObject = findRenderObject();
    if (renderObject is RenderBox)
      return renderObject.size;
    return null;
  }

  /// 一个从类型到 element 实例的映射，通常被 xxx.of(Context) 使用，用于获取对应类型的依赖  
  Map<Type, InheritedElement> _inheritedWidgets;
    
  /// 依赖（依赖于 InheritedElement ）集合
  Set<InheritedElement> _dependencies;
  
  /// 是否由未被满足的依赖   
  bool _hadUnsatisfiedDependencies = false;

  /// 建立遗传关系
  /// 自身建立一个_dependencies（哈希集合），并将 ancestor（InheritedElement ）加入集合中
  /// 调用 InheritedElement 祖先的 updateDependencies，在其 _dependents 中加入自身

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

  /// 返回指定类型的 inheritFromElement 持有的 widget
  @override
  InheritedWidget inheritFromWidgetOfExactType(Type targetType, { Object aspect }) {
    /// 在 _inheritedWidgets 中通过 <Type> 查找对应的 element
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

  /// 只在 _inheritedWidgets 中通过 <Type> 查找对应的 element，不建立联系
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

  /// 向上查找第一个指定类型的 Widget
  @override
  Widget ancestorWidgetOfExactType(Type targetType) {
    Element ancestor = _parent;
    while (ancestor != null && ancestor.widget.runtimeType != targetType)
      ancestor = ancestor._parent;
    return ancestor?.widget;
  }

  /// 向上查找第一个指定类型的 State
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

  /// 查找根指定类型的 State
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

   /// 向上查找第一个指定类型的 RenderObeject
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

  /// 访问先祖 elment
  /// 如果 visitor 返回 false 的话会停止查找  
  @override
  void visitAncestorElements(bool visitor(Element element)) {
    Element ancestor = _parent;
    while (ancestor != null && visitor(ancestor))
      ancestor = ancestor._parent;
  }

  /// 依赖发生改变时的回调，标记需要重建
  /// 通常被 ProxyElement 的 notifyClinets 调用
  @mustCallSuper
  void didChangeDependencies() {
    markNeedsBuild();
  }

  ...
      
  /// dirty 标记，标记这个 element 是否需要重建    
  bool get dirty => _dirty;
  bool _dirty = true;

  /// 是否在 owner._dirtyElements 中
  bool _inDirtyList = false;

  ... 

  void markNeedsBuild() {
    if (dirty)
      return;
    _dirty = true;
    owner.scheduleBuildFor(this);
  }

  /// 如果在这个节点上调用 markNeedsBuild 方法，那么会把这个节点加入到 BuildOwner._dirtyElements 
  //  列表中 在下一个vsycn 信号到来的时候，引擎会调用到 buildScope 方法，对   
  /// buildOwner._dirtyElements 中的每一个节点 element 调用其 rebuild 方法，子类必须重载
  /// performRebuild方法来完成重建 element 树
  void rebuild() {
    performRebuild();
  }

  /// 子类必须重载这个方法来完成重建的具体操作
  @protected
  void performRebuild();
}

```



## ComponeElement

StatelessElement，StatefulElement，ProxyElement 的基类

```dart
abstract class ComponentElement extends Element {
  ComponentElement(Widget widget) : super(widget);

  Element _child;

  /// 在 mount 的时候调用 _firstBuild 方法
  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _firstBuild();
  }

  /// 子类可以重载这个方法来完成一些额外的操作  
  void _firstBuild() {
    rebuild();
  }

  /// 当首次建立被 [mount]，或需要更新时被 [rebuild] 自动调用
  /// 利用 build 方法来创建这个 element 持有的 widget 的子 widget
  /// 并使用新得到的 widget 来更新自身持有的 element  
  @override
  void performRebuild() {
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
    }
  }

  /// 子类必须重载这个方法来返回子 widget（或者 widget 子树） 
  @protected
  Widget build();

  @override
  void visitChildren(ElementVisitor visitor) {
    /// 如果有子 element才访问  
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
  ProxyElement(ProxyWidget widget) : super(widget);

  @override
  ProxyWidget get widget => super.widget;

  @override
  Widget build() => widget.child;

  /// 每次 update 时，会通知 listener  
  @override
  void update(ProxyWidget newWidget) {
    final ProxyWidget oldWidget = widget;
    super.update(newWidget);
    updated(oldWidget);
    _dirty = true;
    rebuild();
  }

  /// widget 发生改变时调用，使用旧的  widget 通知所有的监听者
  @protected
  void updated(covariant ProxyWidget oldWidget) {
    notifyClients(oldWidget);
  }

  @protected
  void notifyClients(covariant ProxyWidget oldWidget);
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
    _dirty = true;
    rebuild();
  }
}
```



## StatefulElement

```dart
class StatefulElement extends ComponentElement {
  StatefulElement(StatefulWidget widget)
      /// 注意，state 在 StatefulElement 被创建的同时被创建
      : _state = widget.createState(),
        super(widget) 
  {
    /// 为 State 提供自身和 widget 的引用        
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

  /// 递归地调用子树的 _updateParentData(如果子节点的是 RenderObjectElement) 
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

  /// 调用给定 widget 的[ParentDataWidget.applyParentData]
  ///
  /// 这个方法允许改变[RenderObject.parentData]而不触发重建
  /// 这样做通常不是很好，当以下两个情况确实值得的：
  ///
  ///  * 处于 build 和 layout 阶段，[ParentData] 不会影响 build 和 layout，并且无法在 build 和
  ///    layout 之前确定数据的值 （例如，它取决于一个孩子的布局）
  ///
  ///  * 处于 paint 阶段，[ParentData] 不会影响 paint，并且无法在 paint 之前确定数据的值 （例如，它
  ///    取决于合成阶段）
  ///
  /// 在下一次构建时，会使用更新的数据作为配置
  ///
  /// The new widget must have the same child as the current widget.
  ///
  /// 使用它的一个例子是 [AutomaticKeepAlive] widget。如果它在生成它的一个后代时收到通知，告诉它
  /// 它的子 widget 必须保持 alive，它将应用一个 [KeepAlive] widget
  /// 这是安全的，因为根据定义，孩子已经保持 alive ,因此这不会改变父节点的帧行为。
  /// 它比仅仅为了更新 [KeepAlive] widge 而请求额外的帧更有效。
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

  /// 在实例化 InheritedElement 时，实例化一个 _dependents，当一个后代将这个 element 作为依赖的时
  /// 候，会将它加入到这个依赖者列表中。之后当这个 element 被 rebuild 的时候会通知 依赖者列表中的每一
  /// 个节点，并调用它的 didChangeDependencies  
  final Map<Element, Object> _dependents = HashMap<Element, Object>();

  @override
  void _updateInheritance() {
    /// incomminWidget 是对父节点遗传映射表的引用  
    final Map<Type, InheritedElement> incomingWidgets = _parent?._inheritedWidgets;
    if (incomingWidgets != null)
      /// 拷贝一份作为自身的映射表  
      _inheritedWidgets = HashMap<Type, InheritedElement>.from(incomingWidgets);
    else
      /// 空表  
      _inheritedWidgets = HashMap<Type, InheritedElement>();
      
    /// 将自身加入到类型遗传映射表中供后代获取  
    _inheritedWidgets[widget.runtimeType] = this;
  }

  ...

  /// 以某个 element 为键，返回其在 _denpents 中的映射值    
  @protected
  Object getDependencies(Element dependent) {
    return _dependents[dependent];
  }

  /// 将一个 element 作为 dependt，并将它映射成一个值  
  @protected
  void setDependencies(Element dependent, Object value) {
    _dependents[dependent] = value;
  }

  @protected
  void updateDependencies(Element dependent, Object aspect) {
    setDependencies(dependent, null);
  }

  /// 通知某个 dependent，调用其 didChangeDependencies
  @protected
  void notifyDependent(covariant InheritedWidget oldWidget, Element dependent) {
    dependent.didChangeDependencies();
  }
    
  /// 这个方法直接调用了 notifyClients，也就是说 inheritedElement 被更新的时候，会通知所有将这个 
  /// inheritedElement作为依赖的子 element
  void updated(InheritedWidget oldWidget) {
    if (widget.updateShouldNotify(oldWidget))
      super.updated(oldWidget);
  }

  /// 通知所有的 dependent
  @override
  void notifyClients(InheritedWidget oldWidget) {
    for (Element dependent in _dependents.keys) {
      notifyDependent(oldWidget, dependent);
    }
  }
}
```





## RenderObjectElment

```dart
abstract class RenderObjectElement extends Element {
  RenderObjectElement(RenderObjectWidget widget) : super(widget);

  @override
  RenderObjectWidget get widget => super.widget;

  @override
  RenderObject get renderObject => _renderObject;
  RenderObject _renderObject;

  /// 找到一个祖先 RenderObjectElement
  RenderObjectElement _ancestorRenderObjectElement;

  RenderObjectElement _findAncestorRenderObjectElement() {
    Element ancestor = _parent;
    while (ancestor != null && ancestor is! RenderObjectElement)
      ancestor = ancestor._parent;
    return ancestor;
  }

  /// 找到一个祖先 ParentDataElement
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
    /// 这里断言了 slot 没有改变，这意味着当这个 element 被替换成新的 element 的时候，对应的 
    /// renderObject 插槽不会改变  
    assert(_slot == newSlot);
    attachRenderObject(newSlot);
    _dirty = false;
  }

  @override
  void update(covariant RenderObjectWidget newWidget) {
    super.update(newWidget);
    widget.updateRenderObject(this, renderObject);
    _dirty = false;
  }

  /// 仅仅更新一下渲染对象
  @override
  void performRebuild() {
    widget.updateRenderObject(this, renderObject);
    _dirty = false;
  }

  /// 使用新的 widget 列表来更新旧 widget 列表
  @protected
  List<Element> updateChildren(
      List<Element> oldChildren, 
      List<Widget> newWidgets, { 
      Set<Element> forgottenChildren
  }) {

    /// 如果给定的 child 已经被遗忘的话，返回 null，否则返回 child
    Element replaceWithNullIfForgotten(Element child) {
      return forgottenChildren != null && forgottenChildren.contains(child) 
          ? null 
          : child;
    }

    // 这将尝试将新的子列表(newWidgets)与旧的子列表(oldChildren)进行比较，并生成一个新的元素列表作为
    // 该元素的新子元素列表。此方法的调用将相应地更新此渲染对象。
    // 它试图优化的案例有:
    //   -旧的列表是空的
    //   -名单是一样的
    //   -在列表中只有一个地方插入或删除一个或多个小部件
    // 如果两个列表中都有一个带有同一个 key 的 widget，那么它将被同步。
    // 没有 key 的 widget 可能会被同步，但没有保证。

    // 一般的方法是将整个新列表向后同步，如下所示:
    // 1. 从首部遍历列表，同步节点，直到不再有匹配的节点。
    // 2. 从尾部遍历列表，不同步节点，直到不再有匹配的节点。
    //    我们将在最后同步这些节点。我们现在不同步它们，因为我们想要同步所有节点，从开始到结束。
    // 此时，我们将旧列表和新列表缩小到节点不再匹配的地方。
    // 3.遍历旧列表中剩余的部分以获得 key 列表，并用 null 同步不含 key 的项同步。
    // 4.从后先前遍历剩余列表:
    //   *用null同步非键值项
    //   *将键控项与源同步(如果存在)，否则为空。
    // 5. 从前向后遍历列表同步节点
    // 6. 将空值与列表中的任何仍然挂载的项同步。

    int newChildrenTop = 0;
    int oldChildrenTop = 0;
    int newChildrenBottom = newWidgets.length - 1;
    int oldChildrenBottom = oldChildren.length - 1;

    /// 用于存放更新以后 child 列表 
    final List<Element> newChildren = oldChildren.length == newWidgets.length 
        ? oldChildren 
        : List<Element>(newWidgets.length);

    /// 上一个子节点，被用作 slot
    Element previousChild;

    // 1.从首部遍历列表，同步节点，直到不再有匹配的节点。
    while (
        (oldChildrenTop <= oldChildrenBottom) && 
        (newChildrenTop <= newChildrenBottom)
    ) {
      /// 如果对应 index 的 oldChild 已经被遗忘的话，得到 null 
      final Element oldChild = replaceWithNullIfForgotten(oldChildren[oldChildrenTop]);
      /// 获得对应 index 的 widget  
      final Widget newWidget = newWidgets[newChildrenTop];
      /// 如果不能用新的 widget 来更新旧的 widget 的话，进入阶段2
      if (oldChild == null || !Widget.canUpdate(oldChild.widget, newWidget))
        break;
      /// 得到更新后的 child
      /// 注意：这里把 previousChild 做为 slots 传入
      final Element newChild = updateChild(oldChild, newWidget, previousChild);
      /// 放置到对应的 index 上 
      newChildren[newChildrenTop] = newChild;
      previousChild = newChild;
      /// index 后移
      newChildrenTop += 1;
      oldChildrenTop += 1;
    }

    // 2.从尾部遍历列表，不同步节点，直到不再有匹配的节点。
    while (
        (oldChildrenTop <= oldChildrenBottom) && 
        (newChildrenTop <= newChildrenBottom)
    ) {
      final Element oldChild = 
                 replaceWithNullIfForgotten(oldChildren[oldChildrenBottom]);
      final Widget newWidget = newWidgets[newChildrenBottom];
      /// 如果不能用新的 widget 来更新旧的 widget 的话，进入阶段3
      if (oldChild == null || !Widget.canUpdate(oldChild.widget, newWidget))
        break;
      /// 注意这里并没有进行节点移动操作
      oldChildrenBottom -= 1;
      newChildrenBottom -= 1;
    }

    // 3.遍历旧列表中剩余的部分以获得 key 列表，并用 null 同步不含 key 的项同步。
    // 这个值为 true 的话，说明有无法匹配的 child
    final bool haveOldChildren = (oldChildrenTop <= oldChildrenBottom);
    // 如果 child 含有key，会保存对应的 element，否则直接 deactivateChild(oldChild);
    Map<Key, Element> oldKeyedChildren;
    if (haveOldChildren) {
      oldKeyedChildren = <Key, Element>{};
      while (oldChildrenTop <= oldChildrenBottom) {
        final Element oldChild = replaceWithNullIfForgotten(oldChildren[oldChildrenTop]);
        if (oldChild != null) {
          if (oldChild.widget.key != null)
            oldKeyedChildren[oldChild.widget.key] = oldChild;
          else
           de deactivateChild(oldChild);
        }
        oldChildrenTop += 1;
      }
    }

    // 4.从后先前遍历剩余列表:
    //   *用null同步非键值项
    //   *将键控项与源同步(如果存在)，否则为空。
    while (newChildrenTop <= newChildrenBottom) {
      Element oldChild;
      final Widget newWidget = newWidgets[newChildrenTop];
      if (haveOldChildren) {
        final Key key = newWidget.key;
        if (key != null) {
          // 在新 widget 列表中查找是否有对应 key
          oldChild = oldKeyedChildren[key];
          // 如果有，则会将 key 从（oldKeyedChildren移除（通常这种情况是 widget 的顺序发生了变化）
          if (oldChild != null) {
            if (Widget.canUpdate(oldChild.widget, newWidget)) {
              oldKeyedChildren.remove(key);
            } else {
              // Not a match, let's pretend we didn't see it for now.
              oldChild = null;
            }
          }
        }
      }
      // 到达这里时有两种情况：
      // 1.能够在旧 element 列表中找到一个能与新 widget 的key对应的 element，那么使用新 widget 去更新它，得
      //   到一个 child
      // 2.如果不能，则会将新 widget 扩展成一个 element，作为 child 
      /// 注意：这里把 previousChild 做为 slots 传入  
      final Element newChild = updateChild(
          oldChild, 
          newWidget, 
          //
          previousChild
      );
      // 放置在对应的位置上  
      newChildren[newChildrenTop] = newChild;
      previousChild = newChild;
      newChildrenTop += 1;
    }

    // 更新列表的尾部
    newChildrenBottom = newWidgets.length - 1;
    oldChildrenBottom = oldChildren.length - 1;

    // 使用新 widget 来更新对应的 child
    while (
        (oldChildrenTop <= oldChildrenBottom) && 
        (newChildrenTop <= newChildrenBottom)
    ) {
      final Element oldChild = oldChildren[oldChildrenTop];
      final Widget newWidget = newWidgets[newChildrenTop];
      final Element newChild = updateChild(oldChild, newWidget, previousChild);
      newChildren[newChildrenTop] = newChild;
      previousChild = newChild;
      newChildrenTop += 1;
      oldChildrenTop += 1;
    }

    // 清理没有被重用的 element
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
  }

  /// 同 mount 方法挂载 renderObject 一样，这里卸载了 renderObject
  @override
  void unmount() {
    super.unmount();
    widget.didUnmountRenderObject(renderObject);
  }

  void _updateParentData(ParentDataWidget<RenderObjectWidget> parentData) {
    parentData.applyParentData(renderObject);
  }

  /// 更新插槽需要移动 renderObejct
  @override
  void _updateSlot(dynamic newSlot) {
    super._updateSlot(newSlot);
    _ancestorRenderObjectElement.moveChildRenderObject(renderObject, slot);
  }

  // 向上查找第一个 RenderObjectElement，并将自身的 renderObject 作为它的 renderObject 的孩子
  @override
  void attachRenderObject(dynamic newSlot) {
    _slot = newSlot;
    _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
    _ancestorRenderObjectElement?.insertChildRenderObject(renderObject, newSlot);
    final ParentDataElement<RenderObjectWidget> parentDataElement = _findAncestorParentDataElement();
    if (parentDataElement != null)
      _updateParentData(parentDataElement.widget);
  }

  // 这个方法将 renderobject 从其父节移除
  @override
  void detachRenderObject() {
    if (_ancestorRenderObjectElement != null) {
      _ancestorRenderObjectElement.removeChildRenderObject(renderObject);
      _ancestorRenderObjectElement = null;
    }
    _slot = null;
  }

  @protected
  void insertChildRenderObject(covariant RenderObject child, covariant dynamic slot);

  @protected
  void moveChildRenderObject(covariant RenderObject child, covariant dynamic slot);
    
  @protected
  void removeChildRenderObject(covariant RenderObject child);
}
```



## RootRenderObjectElement

```dart
/// element 树的根节点
///
/// 只有根元素可以显式地设置其所有者。所有其他元素都从它们的父元素继承它们的所有者。
abstract class RootRenderObjectElement extends RenderObjectElement {
  RootRenderObjectElement(RenderObjectWidget widget) : super(widget);

  void assignOwner(BuildOwner owner) {
    _owner = owner;
  }

  @override
  void mount(Element parent, dynamic newSlot) {
    // 显然这个 element 没有 parent
    assert(parent == null);
    assert(newSlot == null);
    super.mount(parent, newSlot);
  }
}
```



## LeafRenderObjectElement

```dart
/// 没有子节点的渲染对象 element
class LeafRenderObjectElement extends RenderObjectElement {
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
/// 有且仅有一个孩子的渲染对象 element
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
    /// 注意，这里的 slot 恒定为 null
    _child = updateChild(_child, widget.child, null);
  }

  @override
  void update(SingleChildRenderObjectWidget newWidget) {
    super.update(newWidget);
    _child = updateChild(_child, widget.child, null);
  }

  /// 插入一个子渲染对象
  /// 由于这个 element 持有的渲染对象只能有一个孩子，故直接把给定的 child 作为 renderObject 的 child  
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
  MultiChildRenderObjectElement(MultiChildRenderObjectWidget widget) : super(widget);

  @override
  MultiChildRenderObjectWidget get widget => super.widget;

  /// 过滤后的子 element 列表
  Iterable<Element> get children => 
      _children.where((Element child) => !_forgottenChildren.contains(child));

  /// 真正的子 element 列表  
  List<Element> _children;
  /// 使用 HashSet（查找时间O（１）），来过滤已被移除的孩子
  final Set<Element> _forgottenChildren = HashSet<Element>();

  /// 以下的方法通常是由几个 mixin 实现的
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

  /// 当要移除一个 child 的时候，不会真正的移除它，而是把它加入到遗忘孩子列表中，并在更新的时候将这个列表当作
  /// 参数传入
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


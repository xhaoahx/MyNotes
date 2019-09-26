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
  /// Creates an element that uses the given widget as its configuration.
  ComponentElement(Widget widget) : super(widget);

  Element _child;

  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    assert(_child == null);
    assert(_active);
    _firstBuild();
    assert(_child != null);
  }

  void _firstBuild() {
    rebuild();
  }

  /// Calls the [StatelessWidget.build] method of the [StatelessWidget] object
  /// (for stateless widgets) or the [State.build] method of the [State] object
  /// (for stateful widgets) and then updates the widget tree.
  ///
  /// Called automatically during [mount] to generate the first build, and by
  /// [rebuild] when the element needs updating.
  @override
  void performRebuild() {
    if (!kReleaseMode && debugProfileBuildsEnabled)
      Timeline.startSync('${widget.runtimeType}',  arguments: timelineWhitelistArguments);

    assert(_debugSetAllowIgnoredCallsToMarkNeedsBuild(true));
    Widget built;
    try {
      built = build();
      debugWidgetBuilderValue(widget, built);
    } catch (e, stack) {
      built = ErrorWidget.builder(
        _debugReportException(
          ErrorDescription('building $this'),
          e,
          stack,
          informationCollector: () sync* {
            yield DiagnosticsDebugCreator(DebugCreator(this));
          },
        )
      );
    } finally {
      // We delay marking the element as clean until after calling build() so
      // that attempts to markNeedsBuild() during build() will be ignored.
      _dirty = false;
      assert(_debugSetAllowIgnoredCallsToMarkNeedsBuild(false));
    }
    try {
      _child = updateChild(_child, built, slot);
      assert(_child != null);
    } catch (e, stack) {
      built = ErrorWidget.builder(
        _debugReportException(
          ErrorDescription('building $this'),
          e,
          stack,
          informationCollector: () sync* {
            yield DiagnosticsDebugCreator(DebugCreator(this));
          },
        )
      );
      _child = updateChild(null, built, slot);
    }

    if (!kReleaseMode && debugProfileBuildsEnabled)
      Timeline.finishSync();
  }

  /// Subclasses should override this function to actually call the appropriate
  /// `build` function (e.g., [StatelessWidget.build] or [State.build]) for
  /// their widget.
  @protected
  Widget build();

  @override
  void visitChildren(ElementVisitor visitor) {
    if (_child != null)
      visitor(_child);
  }

  @override
  void forgetChild(Element child) {
    assert(child == _child);
    _child = null;
  }
}
```



## ProxyElement

```dart
/// An [Element] that uses a [ProxyWidget] as its configuration.
abstract class ProxyElement extends ComponentElement {
  /// Initializes fields for subclasses.
  ProxyElement(ProxyWidget widget) : super(widget);

  @override
  ProxyWidget get widget => super.widget;

  @override
  Widget build() => widget.child;

  @override
  void update(ProxyWidget newWidget) {
    final ProxyWidget oldWidget = widget;
    assert(widget != null);
    assert(widget != newWidget);
    super.update(newWidget);
    assert(widget == newWidget);
    updated(oldWidget);
    _dirty = true;
    rebuild();
  }

  /// Called during build when the [widget] has changed.
  ///
  /// By default, calls [notifyClients]. Subclasses may override this method to
  /// avoid calling [notifyClients] unnecessarily (e.g. if the old and new
  /// widgets are equivalent).
  @protected
  void updated(covariant ProxyWidget oldWidget) {
    notifyClients(oldWidget);
  }

  /// Notify other objects that the widget associated with this element has
  /// changed.
  ///
  /// Called during [update] (via [updated]) after changing the widget
  /// associated with this element but before rebuilding this element.
  @protected
  void notifyClients(covariant ProxyWidget oldWidget);
}
```





## ParentDataElement

```dart

/// An [Element] that uses a [ParentDataWidget] as its configuration.
class ParentDataElement<T extends RenderObjectWidget> extends ProxyElement {
  /// Creates an element that uses the given widget as its configuration.
  ParentDataElement(ParentDataWidget<T> widget) : super(widget);

  @override
  ParentDataWidget<T> get widget => super.widget;

  @override
  void mount(Element parent, dynamic newSlot) {
    assert(() {
      final List<Widget> badAncestors = <Widget>[];
      Element ancestor = parent;
      while (ancestor != null) {
        if (ancestor is ParentDataElement<RenderObjectWidget>) {
          badAncestors.add(ancestor.widget);
        } else if (ancestor is RenderObjectElement) {
          if (widget.debugIsValidAncestor(ancestor.widget))
            break;
          badAncestors.add(ancestor.widget);
        }
        ancestor = ancestor._parent;
      }
      if (ancestor != null && badAncestors.isEmpty)
        return true;
      // TODO(jacobr): switch to describing the invalid parent chain in terms
      // of DiagnosticsNode objects when possible.
      throw FlutterError.fromParts(<DiagnosticsNode>[
        ErrorSummary('Incorrect use of ParentDataWidget.'),
        // TODO(jacobr): fix this constructor call to use FlutterErrorBuilder.
        ...widget.debugDescribeInvalidAncestorChain(
          description: '$this',
          ownershipChain: ErrorDescription(parent.debugGetCreatorChain(10)),
          foundValidAncestor: ancestor != null,
          badAncestors: badAncestors,
        ),
      ]);
    }());
    super.mount(parent, newSlot);
  }

  void _applyParentData(ParentDataWidget<T> widget) {
    void applyParentDataToChild(Element child) {
      if (child is RenderObjectElement) {
        child._updateParentData(widget);
      } else {
        assert(child is! ParentDataElement<RenderObjectWidget>);
        child.visitChildren(applyParentDataToChild);
      }
    }
    visitChildren(applyParentDataToChild);
  }

  /// Calls [ParentDataWidget.applyParentData] on the given widget, passing it
  /// the [RenderObject] whose parent data this element is ultimately
  /// responsible for.
  ///
  /// This allows a render object's [RenderObject.parentData] to be modified
  /// without triggering a build. This is generally ill-advised, but makes sense
  /// in situations such as the following:
  ///
  ///  * Build and layout are currently under way, but the [ParentData] in question
  ///    does not affect layout, and the value to be applied could not be
  ///    determined before build and layout (e.g. it depends on the layout of a
  ///    descendant).
  ///
  ///  * Paint is currently under way, but the [ParentData] in question does not
  ///    affect layout or paint, and the value to be applied could not be
  ///    determined before paint (e.g. it depends on the compositing phase).
  ///
  /// In either case, the next build is expected to cause this element to be
  /// configured with the given new widget (or a widget with equivalent data).
  ///
  /// Only [ParentDataWidget]s that return true for
  /// [ParentDataWidget.debugCanApplyOutOfTurn] can be applied this way.
  ///
  /// The new widget must have the same child as the current widget.
  ///
  /// An example of when this is used is the [AutomaticKeepAlive] widget. If it
  /// receives a notification during the build of one of its descendants saying
  /// that its child must be kept alive, it will apply a [KeepAlive] widget out
  /// of turn. This is safe, because by definition the child is already alive,
  /// and therefore this will not change the behavior of the parent this frame.
  /// It is more efficient than requesting an additional frame just for the
  /// purpose of updating the [KeepAlive] widget.
  void applyWidgetOutOfTurn(ParentDataWidget<T> newWidget) {
    assert(newWidget != null);
    assert(newWidget.debugCanApplyOutOfTurn());
    assert(newWidget.child == widget.child);
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



## StatelessElement

```dart
/// An [Element] that uses a [StatelessWidget] as its configuration.
class StatelessElement extends ComponentElement {
  /// Creates an element that uses the given widget as its configuration.
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



## StatefulElement

```dart
/// An [Element] that uses a [StatefulWidget] as its configuration.
class StatefulElement extends ComponentElement {
  /// Creates an element that uses the given widget as its configuration.
  StatefulElement(StatefulWidget widget)
      : _state = widget.createState(),
        super(widget) {
    assert(() {
      if (!_state._debugTypesAreRight(widget)) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('StatefulWidget.createState must return a subtype of State<${widget.runtimeType}>'),
          ErrorDescription(
            'The createState function for ${widget.runtimeType} returned a state '
            'of type ${_state.runtimeType}, which is not a subtype of '
            'State<${widget.runtimeType}>, violating the contract for createState.'
          ),
        ]);
      }
      return true;
    }());
    assert(_state._element == null);
    _state._element = this;
    assert(_state._widget == null);
    _state._widget = widget;
    assert(_state._debugLifecycleState == _StateLifecycle.created);
  }

  @override
  Widget build() => state.build(this);

  /// The [State] instance associated with this location in the tree.
  ///
  /// There is a one-to-one relationship between [State] objects and the
  /// [StatefulElement] objects that hold them. The [State] objects are created
  /// by [StatefulElement] in [mount].
  State<StatefulWidget> get state => _state;
  State<StatefulWidget> _state;

  @override
  void reassemble() {
    state.reassemble();
    super.reassemble();
  }

  @override
  void _firstBuild() {
    assert(_state._debugLifecycleState == _StateLifecycle.created);
    try {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(true);
      final dynamic debugCheckForReturnedFuture = _state.initState() as dynamic;
      assert(() {
        if (debugCheckForReturnedFuture is Future) {
          throw FlutterError.fromParts(<DiagnosticsNode>[
            ErrorSummary('${_state.runtimeType}.initState() returned a Future.'),
            ErrorDescription('State.initState() must be a void method without an `async` keyword.'),
            ErrorHint(
              'Rather than awaiting on asynchronous work directly inside of initState, '
              'call a separate method to do this work without awaiting it.'
            ),
          ]);
        }
        return true;
      }());
    } finally {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(false);
    }
    assert(() { _state._debugLifecycleState = _StateLifecycle.initialized; return true; }());
    _state.didChangeDependencies();
    assert(() { _state._debugLifecycleState = _StateLifecycle.ready; return true; }());
    super._firstBuild();
  }

  @override
  void update(StatefulWidget newWidget) {
    super.update(newWidget);
    assert(widget == newWidget);
    final StatefulWidget oldWidget = _state._widget;
    // Notice that we mark ourselves as dirty before calling didUpdateWidget to
    // let authors call setState from within didUpdateWidget without triggering
    // asserts.
    _dirty = true;
    _state._widget = widget;
    try {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(true);
      final dynamic debugCheckForReturnedFuture = _state.didUpdateWidget(oldWidget) as dynamic;
      assert(() {
        if (debugCheckForReturnedFuture is Future) {
          throw FlutterError.fromParts(<DiagnosticsNode>[
            ErrorSummary('${_state.runtimeType}.didUpdateWidget() returned a Future.'),
            ErrorDescription( 'State.didUpdateWidget() must be a void method without an `async` keyword.'),
            ErrorHint(
              'Rather than awaiting on asynchronous work directly inside of didUpdateWidget, '
              'call a separate method to do this work without awaiting it.'
            ),
          ]);
        }
        return true;
      }());
    } finally {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(false);
    }
    rebuild();
  }

  @override
  void activate() {
    super.activate();
    // Since the State could have observed the deactivate() and thus disposed of
    // resources allocated in the build method, we have to rebuild the widget
    // so that its State can reallocate its resources.
    assert(_active); // otherwise markNeedsBuild is a no-op
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
    _state.dispose();
    assert(() {
      if (_state._debugLifecycleState == _StateLifecycle.defunct)
        return true;
      throw FlutterError.fromParts(<DiagnosticsNode>[
        ErrorSummary('${_state.runtimeType}.dispose failed to call super.dispose.'),
        ErrorDescription(
          'dispose() implementations must always call their superclass dispose() method, to ensure '
         'that all the resources used by the widget are fully released.'
        ),
      ]);
    }());
    _state._element = null;
    _state = null;
  }

  @override
  InheritedWidget inheritFromElement(Element ancestor, { Object aspect }) {
    assert(ancestor != null);
    assert(() {
      final Type targetType = ancestor.widget.runtimeType;
      if (state._debugLifecycleState == _StateLifecycle.created) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('inheritFromWidgetOfExactType($targetType) or inheritFromElement() was called before ${_state.runtimeType}.initState() completed.'),
          ErrorDescription(
            'When an inherited widget changes, for example if the value of Theme.of() changes, '
            'its dependent widgets are rebuilt. If the dependent widget\'s reference to '
            'the inherited widget is in a constructor or an initState() method, '
            'then the rebuilt dependent widget will not reflect the changes in the '
            'inherited widget.',
          ),
          ErrorHint(
            'Typically references to inherited widgets should occur in widget build() methods. Alternatively, '
            'initialization based on inherited widgets can be placed in the didChangeDependencies method, which '
            'is called after initState and whenever the dependencies change thereafter.'
          ),
        ]);
      }
      if (state._debugLifecycleState == _StateLifecycle.defunct) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('inheritFromWidgetOfExactType($targetType) or inheritFromElement() was called after dispose(): $this'),
          ErrorDescription(
            'This error happens if you call inheritFromWidgetOfExactType() on the '
            'BuildContext for a widget that no longer appears in the widget tree '
            '(e.g., whose parent widget no longer includes the widget in its '
            'build). This error can occur when code calls '
            'inheritFromWidgetOfExactType() from a timer or an animation callback.'
          ),
          ErrorHint(
            'The preferred solution is to cancel the timer or stop listening to the '
            'animation in the dispose() callback. Another solution is to check the '
            '"mounted" property of this object before calling '
            'inheritFromWidgetOfExactType() to ensure the object is still in the '
            'tree.'
          ),
          ErrorHint(
            'This error might indicate a memory leak if '
            'inheritFromWidgetOfExactType() is being called because another object '
            'is retaining a reference to this State object after it has been '
            'removed from the tree. To avoid memory leaks, consider breaking the '
            'reference to this object during dispose().'
          ),
        ]);
      }
      return true;
    }());
    return super.inheritFromElement(ancestor, aspect: aspect);
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    _state.didChangeDependencies();
  }

  @override
  DiagnosticsNode toDiagnosticsNode({ String name, DiagnosticsTreeStyle style }) {
    return _ElementDiagnosticableTreeNode(
      name: name,
      value: this,
      style: style,
      stateful: true,
    );
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(DiagnosticsProperty<State<StatefulWidget>>('state', state, defaultValue: null));
  }
}
```



## LeafRenderObjectElement

```dar

/// An [Element] that uses a [LeafRenderObjectWidget] as its configuration.
class LeafRenderObjectElement extends RenderObjectElement {
  /// Creates an element that uses the given widget as its configuration.
  LeafRenderObjectElement(LeafRenderObjectWidget widget) : super(widget);

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

  @override
  List<DiagnosticsNode> debugDescribeChildren() {
    return widget.debugDescribeChildren();
  }
}
```





## SingleChildRenderObjectElement

```dart
/// An [Element] that uses a [SingleChildRenderObjectWidget] as its configuration.
///
/// The child is optional.
///
/// This element subclass can be used for RenderObjectWidgets whose
/// RenderObjects use the [RenderObjectWithChildMixin] mixin. Such widgets are
/// expected to inherit from [SingleChildRenderObjectWidget].
class SingleChildRenderObjectElement extends RenderObjectElement {
  /// Creates an element that uses the given widget as its configuration.
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
    assert(child == _child);
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
    assert(slot == null);
    assert(renderObject.debugValidateChild(child));
    renderObject.child = child;
    assert(renderObject == this.renderObject);
  }

  @override
  void moveChildRenderObject(RenderObject child, dynamic slot) {
    assert(false);
  }

  @override
  void removeChildRenderObject(RenderObject child) {
    final RenderObjectWithChildMixin<RenderObject> renderObject = this.renderObject;
    assert(renderObject.child == child);
    renderObject.child = null;
    assert(renderObject == this.renderObject);
  }
}
```



## MultiChildRenderObjectElment

```dart
/// An [Element] that uses a [MultiChildRenderObjectWidget] as its configuration.
///
/// This element subclass can be used for RenderObjectWidgets whose
/// RenderObjects use the [ContainerRenderObjectMixin] mixin with a parent data
/// type that implements [ContainerParentDataMixin<RenderObject>]. Such widgets
/// are expected to inherit from [MultiChildRenderObjectWidget].
class MultiChildRenderObjectElement extends RenderObjectElement {
  /// Creates an element that uses the given widget as its configuration.
  MultiChildRenderObjectElement(MultiChildRenderObjectWidget widget)
    : assert(!debugChildrenHaveDuplicateKeys(widget, widget.children)),
      super(widget);

  @override
  MultiChildRenderObjectWidget get widget => super.widget;

  /// The current list of children of this element.
  ///
  /// This list is filtered to hide elements that have been forgotten (using
  /// [forgetChild]).
  @protected
  @visibleForTesting
  Iterable<Element> get children => _children.where((Element child) => !_forgottenChildren.contains(child));

  List<Element> _children;
  // We keep a set of forgotten children to avoid O(n^2) work walking _children
  // repeatedly to remove children.
  final Set<Element> _forgottenChildren = HashSet<Element>();

  @override
  void insertChildRenderObject(RenderObject child, Element slot) {
    final ContainerRenderObjectMixin<RenderObject, ContainerParentDataMixin<RenderObject>> renderObject = this.renderObject;
    assert(renderObject.debugValidateChild(child));
    renderObject.insert(child, after: slot?.renderObject);
    assert(renderObject == this.renderObject);
  }

  @override
  void moveChildRenderObject(RenderObject child, dynamic slot) {
    final ContainerRenderObjectMixin<RenderObject, ContainerParentDataMixin<RenderObject>> renderObject = this.renderObject;
    assert(child.parent == renderObject);
    renderObject.move(child, after: slot?.renderObject);
    assert(renderObject == this.renderObject);
  }

  @override
  void removeChildRenderObject(RenderObject child) {
    final ContainerRenderObjectMixin<RenderObject, ContainerParentDataMixin<RenderObject>> renderObject = this.renderObject;
    assert(child.parent == renderObject);
    renderObject.remove(child);
    assert(renderObject == this.renderObject);
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
    assert(_children.contains(child));
    assert(!_forgottenChildren.contains(child));
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
    assert(widget == newWidget);
    _children = updateChildren(_children, widget.children, forgottenChildren: _forgottenChildren);
    _forgottenChildren.clear();
  }
}
```


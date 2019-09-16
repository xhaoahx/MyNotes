# Flutter中的Element



```dart
/// An instantiation of a [Widget] at a particular location in the tree.
///
/// Widgets describe how to configure a subtree but the same widget can be used
/// to configure multiple subtrees simultaneously because widgets are immutable.
/// An [Element] represents the use of a widget to configure a specific location
/// in the tree. Over time, the widget associated with a given element can
/// change, for example, if the parent widget rebuilds and creates a new widget
/// for this location.
///
/// Elements form a tree. Most elements have a unique child, but some widgets
/// (e.g., subclasses of [RenderObjectElement]) can have multiple children.
///
/// Elements have the following lifecycle:
///
///  * The framework creates an element by calling [Widget.createElement] on the
///    widget that will be used as the element's initial configuration.
///  * The framework calls [mount] to add the newly created element to the tree
///    at a given slot in a given parent. The [mount] method is responsible for
///    inflating any child widgets and calling [attachRenderObject] as
///    necessary to attach any associated render objects to the render tree.
///  * At this point, the element is considered "active" and might appear on
///    screen.
///  * At some point, the parent might decide to change the widget used to
///    configure this element, for example because the parent rebuilt with new
///    state. When this happens, the framework will call [update] with the new
///    widget. The new widget will always have the same [runtimeType] and key as
///    old widget. If the parent wishes to change the [runtimeType] or key of
///    the widget at this location in the tree, it can do so by unmounting this
///    element and inflating the new widget at this location.
///  * At some point, an ancestor might decide to remove this element (or an
///    intermediate ancestor) from the tree, which the ancestor does by calling
///    [deactivateChild] on itself. Deactivating the intermediate ancestor will
///    remove that element's render object from the render tree and add this
///    element to the [owner]'s list of inactive elements, causing the framework
///    to call [deactivate] on this element.
///  * At this point, the element is considered "inactive" and will not appear
///    on screen. An element can remain in the inactive state only until
///    the end of the current animation frame. At the end of the animation
///    frame, any elements that are still inactive will be unmounted.
///  * If the element gets reincorporated into the tree (e.g., because it or one
///    of its ancestors has a global key that is reused), the framework will
///    remove the element from the [owner]'s list of inactive elements, call
///    [activate] on the element, and reattach the element's render object to
///    the render tree. (At this point, the element is again considered "active"
///    and might appear on screen.)
///  * If the element does not get reincorporated into the tree by the end of
///    the current animation frame, the framework will call [unmount] on the
///    element.
///  * At this point, the element is considered "defunct" and will not be
///    incorporated into the tree in the future.
abstract class Element extends DiagnosticableTree implements BuildContext {
  /// Creates an element that uses the given widget as its configuration.
  ///
  /// Typically called by an override of [Widget.createElement].
  Element(Widget widget)
    : assert(widget != null),
      _widget = widget;

  Element _parent;

  // Custom implementation of `operator ==` optimized for the ".of" pattern
  // used with `InheritedWidgets`.
  @override
  bool operator ==(Object other) => identical(this, other);

  // Custom implementation of hash code optimized for the ".of" pattern used
  // with `InheritedWidgets`.
  //
  // `Element.inheritFromWidgetOfExactType` relies heavily on hash-based
  // `Set` look-ups, putting this getter on the performance critical path.
  //
  // The value is designed to fit within the SMI representation. This makes
  // the cached value use less memory (one field and no extra heap objects) and
  // cheap to compare (no indirection).
  //
  // See also:
  //
  //  * https://dart.dev/articles/dart-vm/numeric-computation, which
  //    explains how numbers are represented in Dart.
  @override
  int get hashCode => _cachedHash;
  final int _cachedHash = _nextHashCode = (_nextHashCode + 1) % 0xffffff;
  static int _nextHashCode = 1;

  /// Information set by parent to define where this child fits in its parent's
  /// child list.
  ///
  /// Subclasses of Element that only have one child should use null for
  /// the slot for that child.
  dynamic get slot => _slot;
  dynamic _slot;

  /// An integer that is guaranteed to be greater than the parent's, if any.
  /// The element at the root of the tree must have a depth greater than 0.
  int get depth => _depth;
  int _depth;

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

  /// The configuration for this element.
  @override
  Widget get widget => _widget;
  Widget _widget;

  /// The object that manages the lifecycle of this element.
  @override
  BuildOwner get owner => _owner;
  BuildOwner _owner;

  bool _active = false;

  /// {@template flutter.widgets.reassemble}
  /// Called whenever the application is reassembled during debugging, for
  /// example during hot reload.
  ///
  /// This method should rerun any initialization logic that depends on global
  /// state, for example, image loading from asset bundles (since the asset
  /// bundle may have changed).
  ///
  /// This function will only be called during development. In release builds,
  /// the `ext.flutter.reassemble` hook is not available, and so this code will
  /// never execute.
  ///
  /// Implementers should not rely on any ordering for hot reload source update,
  /// reassemble, and build methods after a hot reload has been initiated. It is
  /// possible that a [Timer] (e.g. an [Animation]) or a debugging session
  /// attached to the isolate could trigger a build with reloaded code _before_
  /// reassemble is called. Code that expects preconditions to be set by
  /// reassemble after a hot reload must be resilient to being called out of
  /// order, e.g. by fizzling instead of throwing. That said, once reassemble is
  /// called, build will be called after it at least once.
  /// {@endtemplate}
  ///
  /// See also:
  ///
  ///  * [State.reassemble]
  ///  * [BindingBase.reassembleApplication]
  ///  * [Image], which uses this to reload images.
  @mustCallSuper
  @protected
  void reassemble() {
    markNeedsBuild();
    visitChildren((Element child) {
      child.reassemble();
    });
  }

  bool _debugIsInScope(Element target) {
    Element current = this;
    while (current != null) {
      if (target == current)
        return true;
      current = current._parent;
    }
    return false;
  }

  /// The render object at (or below) this location in the tree.
  ///
  /// If this object is a [RenderObjectElement], the render object is the one at
  /// this location in the tree. Otherwise, this getter will walk down the tree
  /// until it finds a [RenderObjectElement].
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

  @override
  List<DiagnosticsNode> describeMissingAncestor({ @required Type expectedAncestorType }) {
    final List<DiagnosticsNode> information = <DiagnosticsNode>[];
    final List<Element> ancestors = <Element>[];
    visitAncestorElements((Element element) {
      ancestors.add(element);
      return true;
    });

    information.add(DiagnosticsProperty<Element>(
      'The specific widget that could not find a $expectedAncestorType ancestor was',
      this,
      style: DiagnosticsTreeStyle.errorProperty,
    ));

    if (ancestors.isNotEmpty) {
      information.add(describeElements('The ancestors of this widget were', ancestors));
    } else {
      information.add(ErrorDescription(
        'This widget is the root of the tree, so it has no '
        'ancestors, let alone a "$expectedAncestorType" ancestor.'
      ));
    }
    return information;
  }

  /// Returns a list of [Element]s from the current build context to the error report.
  static DiagnosticsNode describeElements(String name, Iterable<Element> elements) {
    return DiagnosticsBlock(
      name: name,
      children: elements.map<DiagnosticsNode>((Element element) => DiagnosticsProperty<Element>('', element)).toList(),
      allowTruncate: true,
    );
  }

  @override
  DiagnosticsNode describeElement(String name, {DiagnosticsTreeStyle style = DiagnosticsTreeStyle.errorProperty}) {
    return DiagnosticsProperty<Element>(name, this, style: style);
  }

  @override
  DiagnosticsNode describeWidget(String name, {DiagnosticsTreeStyle style = DiagnosticsTreeStyle.errorProperty}) {
    return DiagnosticsProperty<Element>(name, this, style: style);
  }

  @override
  DiagnosticsNode describeOwnershipChain(String name) {
    // TODO(jacobr): make this structured so clients can support clicks on
    // individual entries. For example, is this an iterable with arrows as
    // separators?
    return StringProperty(name, debugGetCreatorChain(10));
  }

  // This is used to verify that Element objects move through life in an
  // orderly fashion.
  _ElementLifecycle _debugLifecycleState = _ElementLifecycle.initial;

  /// Calls the argument for each child. Must be overridden by subclasses that
  /// support having children.
  ///
  /// There is no guaranteed order in which the children will be visited, though
  /// it should be consistent over time.
  ///
  /// Calling this during build is dangerous: the child list might still be
  /// being updated at that point, so the children might not be constructed yet,
  /// or might be old children that are going to be replaced. This method should
  /// only be called if it is provable that the children are available.
  void visitChildren(ElementVisitor visitor) { }

  /// Calls the argument for each child considered onstage.
  ///
  /// Classes like [Offstage] and [Overlay] override this method to hide their
  /// children.
  ///
  /// Being onstage affects the element's discoverability during testing when
  /// you use Flutter's [Finder] objects. For example, when you instruct the
  /// test framework to tap on a widget, by default the finder will look for
  /// onstage elements and ignore the offstage ones.
  ///
  /// The default implementation defers to [visitChildren] and therefore treats
  /// the element as onstage.
  ///
  /// See also:
  ///
  ///  * [Offstage] widget that hides its children.
  ///  * [Finder] that skips offstage widgets by default.
  ///  * [RenderObject.visitChildrenForSemantics], in contrast to this method,
  ///    designed specifically for excluding parts of the UI from the semantics
  ///    tree.
  void debugVisitOnstageChildren(ElementVisitor visitor) => visitChildren(visitor);

  /// Wrapper around [visitChildren] for [BuildContext].
  @override
  void visitChildElements(ElementVisitor visitor) {
    assert(() {
      if (owner == null || !owner._debugStateLocked)
        return true;
      throw FlutterError.fromParts(<DiagnosticsNode>[
        ErrorSummary('visitChildElements() called during build.'),
        ErrorDescription(
          'The BuildContext.visitChildElements() method can\'t be called during '
          'build because the child list is still being updated at that point, '
          'so the children might not be constructed yet, or might be old children '
          'that are going to be replaced.'
        )
      ]);
    }());
    visitChildren(visitor);
  }

  /// Update the given child with the given new configuration.
  ///
  /// This method is the core of the widgets system. It is called each time we
  /// are to add, update, or remove a child based on an updated configuration.
  ///
  /// If the `child` is null, and the `newWidget` is not null, then we have a new
  /// child for which we need to create an [Element], configured with `newWidget`.
  ///
  /// If the `newWidget` is null, and the `child` is not null, then we need to
  /// remove it because it no longer has a configuration.
  ///
  /// If neither are null, then we need to update the `child`'s configuration to
  /// be the new configuration given by `newWidget`. If `newWidget` can be given
  /// to the existing child (as determined by [Widget.canUpdate]), then it is so
  /// given. Otherwise, the old child needs to be disposed and a new child
  /// created for the new configuration.
  ///
  /// If both are null, then we don't have a child and won't have a child, so we
  /// do nothing.
  ///
  /// The [updateChild] method returns the new child, if it had to create one,
  /// or the child that was passed in, if it just had to update the child, or
  /// null, if it removed the child and did not replace it.
  ///
  /// The following table summarizes the above:
  ///
  /// |                     | **newWidget == null**  | **newWidget != null**   |
  /// | :-----------------: | :--------------------- | :---------------------- |
  /// |  **child == null**  |  Returns null.         |  Returns new [Element]. |
  /// |  **child != null**  |  Old child is removed, returns null. | Old child updated if possible, returns child or new [Element]. |
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

  /// Add this element to the tree in the given slot of the given parent.
  ///
  /// The framework calls this function when a newly created element is added to
  /// the tree for the first time. Use this method to initialize state that
  /// depends on having a parent. State that is independent of the parent can
  /// more easily be initialized in the constructor.
  ///
  /// This method transitions the element from the "initial" lifecycle state to
  /// the "active" lifecycle state.
  @mustCallSuper
  void mount(Element parent, dynamic newSlot) {
    assert(_debugLifecycleState == _ElementLifecycle.initial);
    assert(widget != null);
    assert(_parent == null);
    assert(parent == null || parent._debugLifecycleState == _ElementLifecycle.active);
    assert(slot == null);
    assert(depth == null);
    assert(!_active);
    _parent = parent;
    _slot = newSlot;
    _depth = _parent != null ? _parent.depth + 1 : 1;
    _active = true;
    if (parent != null) // Only assign ownership if the parent is non-null
      _owner = parent.owner;
    if (widget.key is GlobalKey) {
      final GlobalKey key = widget.key;
      key._register(this);
    }
    _updateInheritance();
    assert(() { _debugLifecycleState = _ElementLifecycle.active; return true; }());
  }

  /// Change the widget used to configure this element.
  ///
  /// The framework calls this function when the parent wishes to use a
  /// different widget to configure this element. The new widget is guaranteed
  /// to have the same [runtimeType] as the old widget.
  ///
  /// This function is called only during the "active" lifecycle state.
  @mustCallSuper
  void update(covariant Widget newWidget) {
    // This code is hot when hot reloading, so we try to
    // only call _AssertionError._evaluateAssertion once.
    assert(_debugLifecycleState == _ElementLifecycle.active
        && widget != null
        && newWidget != null
        && newWidget != widget
        && depth != null
        && _active
        && Widget.canUpdate(widget, newWidget));
    _widget = newWidget;
  }

  /// Change the slot that the given child occupies in its parent.
  ///
  /// Called by [MultiChildRenderObjectElement], and other [RenderObjectElement]
  /// subclasses that have multiple children, when child moves from one position
  /// to another in this element's child list.
  @protected
  void updateSlotForChild(Element child, dynamic newSlot) {
    assert(_debugLifecycleState == _ElementLifecycle.active);
    assert(child != null);
    assert(child._parent == this);
    void visit(Element element) {
      element._updateSlot(newSlot);
      if (element is! RenderObjectElement)
        element.visitChildren(visit);
    }
    visit(child);
  }

  void _updateSlot(dynamic newSlot) {
    assert(_debugLifecycleState == _ElementLifecycle.active);
    assert(widget != null);
    assert(_parent != null);
    assert(_parent._debugLifecycleState == _ElementLifecycle.active);
    assert(depth != null);
    _slot = newSlot;
  }

  void _updateDepth(int parentDepth) {
    final int expectedDepth = parentDepth + 1;
    if (_depth < expectedDepth) {
      _depth = expectedDepth;
      visitChildren((Element child) {
        child._updateDepth(expectedDepth);
      });
    }
  }

  /// Remove [renderObject] from the render tree.
  ///
  /// The default implementation of this function simply calls
  /// [detachRenderObject] recursively on its child. The
  /// [RenderObjectElement.detachRenderObject] override does the actual work of
  /// removing [renderObject] from the render tree.
  ///
  /// This is called by [deactivateChild].
  void detachRenderObject() {
    visitChildren((Element child) {
      child.detachRenderObject();
    });
    _slot = null;
  }

  /// Add [renderObject] to the render tree at the location specified by [slot].
  ///
  /// The default implementation of this function simply calls
  /// [attachRenderObject] recursively on its child. The
  /// [RenderObjectElement.attachRenderObject] override does the actual work of
  /// adding [renderObject] to the render tree.
  void attachRenderObject(dynamic newSlot) {
    assert(_slot == null);
    visitChildren((Element child) {
      child.attachRenderObject(newSlot);
    });
    _slot = newSlot;
  }

  Element _retakeInactiveElement(GlobalKey key, Widget newWidget) {
    // The "inactivity" of the element being retaken here may be forward-looking: if
    // we are taking an element with a GlobalKey from an element that currently has
    // it as a child, then we know that that element will soon no longer have that
    // element as a child. The only way that assumption could be false is if the
    // global key is being duplicated, and we'll try to track that using the
    // _debugTrackElementThatWillNeedToBeRebuiltDueToGlobalKeyShenanigans call below.
    final Element element = key._currentElement;
    if (element == null)
      return null;
    if (!Widget.canUpdate(element.widget, newWidget))
      return null;
    assert(() {
      if (debugPrintGlobalKeyedWidgetLifecycle)
        debugPrint('Attempting to take $element from ${element._parent ?? "inactive elements list"} to put in $this.');
      return true;
    }());
    final Element parent = element._parent;
    if (parent != null) {
      assert(() {
        if (parent == this) {
          throw FlutterError.fromParts(<DiagnosticsNode>[
            ErrorSummary('A GlobalKey was used multiple times inside one widget\'s child list.'),
            DiagnosticsProperty<GlobalKey>('The offending GlobalKey was', key),
            parent.describeElement('The parent of the widgets with that key was'),
            element.describeElement('The first child to get instantiated with that key became'),
            DiagnosticsProperty<Widget>('The second child that was to be instantiated with that key was', widget, style: DiagnosticsTreeStyle.errorProperty),
            ErrorDescription('A GlobalKey can only be specified on one widget at a time in the widget tree.')
          ]);
        }
        parent.owner._debugTrackElementThatWillNeedToBeRebuiltDueToGlobalKeyShenanigans(
          parent,
          key,
        );
        return true;
      }());
      parent.forgetChild(element);
      parent.deactivateChild(element);
    }
    assert(element._parent == null);
    owner._inactiveElements.remove(element);
    return element;
  }

  /// Create an element for the given widget and add it as a child of this
  /// element in the given slot.
  ///
  /// This method is typically called by [updateChild] but can be called
  /// directly by subclasses that need finer-grained control over creating
  /// elements.
  ///
  /// If the given widget has a global key and an element already exists that
  /// has a widget with that global key, this function will reuse that element
  /// (potentially grafting it from another location in the tree or reactivating
  /// it from the list of inactive elements) rather than creating a new element.
  ///
  /// The element returned by this function will already have been mounted and
  /// will be in the "active" lifecycle state.
  @protected
  Element inflateWidget(Widget newWidget, dynamic newSlot) {
    assert(newWidget != null);
    final Key key = newWidget.key;
    if (key is GlobalKey) {
      final Element newChild = _retakeInactiveElement(key, newWidget);
      if (newChild != null) {
        assert(newChild._parent == null);
        assert(() { _debugCheckForCycles(newChild); return true; }());
        newChild._activateWithParent(this, newSlot);
        final Element updatedChild = updateChild(newChild, newWidget, newSlot);
        assert(newChild == updatedChild);
        return updatedChild;
      }
    }
    final Element newChild = newWidget.createElement();
    assert(() { _debugCheckForCycles(newChild); return true; }());
    newChild.mount(this, newSlot);
    assert(newChild._debugLifecycleState == _ElementLifecycle.active);
    return newChild;
  }

  void _debugCheckForCycles(Element newChild) {
    assert(newChild._parent == null);
    assert(() {
      Element node = this;
      while (node._parent != null)
        node = node._parent;
      assert(node != newChild); // indicates we are about to create a cycle
      return true;
    }());
  }

  /// Move the given element to the list of inactive elements and detach its
  /// render object from the render tree.
  ///
  /// This method stops the given element from being a child of this element by
  /// detaching its render object from the render tree and moving the element to
  /// the list of inactive elements.
  ///
  /// This method (indirectly) calls [deactivate] on the child.
  ///
  /// The caller is responsible for removing the child from its child model.
  /// Typically [deactivateChild] is called by the element itself while it is
  /// updating its child model; however, during [GlobalKey] reparenting, the new
  /// parent proactively calls the old parent's [deactivateChild], first using
  /// [forgetChild] to cause the old parent to update its child model.
  @protected
  void deactivateChild(Element child) {
    assert(child != null);
    assert(child._parent == this);
    child._parent = null;
    child.detachRenderObject();
    owner._inactiveElements.add(child); // this eventually calls child.deactivate()
    assert(() {
      if (debugPrintGlobalKeyedWidgetLifecycle) {
        if (child.widget.key is GlobalKey)
          debugPrint('Deactivated $child (keyed child of $this)');
      }
      return true;
    }());
  }

  /// Remove the given child from the element's child list, in preparation for
  /// the child being reused elsewhere in the element tree.
  ///
  /// This updates the child model such that, e.g., [visitChildren] does not
  /// walk that child anymore.
  ///
  /// The element will still have a valid parent when this is called. After this
  /// is called, [deactivateChild] is called to sever the link to this object.
  @protected
  void forgetChild(Element child);

  void _activateWithParent(Element parent, dynamic newSlot) {
    assert(_debugLifecycleState == _ElementLifecycle.inactive);
    _parent = parent;
    assert(() {
      if (debugPrintGlobalKeyedWidgetLifecycle)
        debugPrint('Reactivating $this (now child of $_parent).');
      return true;
    }());
    _updateDepth(_parent.depth);
    _activateRecursively(this);
    attachRenderObject(newSlot);
    assert(_debugLifecycleState == _ElementLifecycle.active);
  }

  static void _activateRecursively(Element element) {
    assert(element._debugLifecycleState == _ElementLifecycle.inactive);
    element.activate();
    assert(element._debugLifecycleState == _ElementLifecycle.active);
    element.visitChildren(_activateRecursively);
  }

  /// Transition from the "inactive" to the "active" lifecycle state.
  ///
  /// The framework calls this method when a previously deactivated element has
  /// been reincorporated into the tree. The framework does not call this method
  /// the first time an element becomes active (i.e., from the "initial"
  /// lifecycle state). Instead, the framework calls [mount] in that situation.
  ///
  /// See the lifecycle documentation for [Element] for additional information.
  @mustCallSuper
  void activate() {
    assert(_debugLifecycleState == _ElementLifecycle.inactive);
    assert(widget != null);
    assert(owner != null);
    assert(depth != null);
    assert(!_active);
    final bool hadDependencies = (_dependencies != null && _dependencies.isNotEmpty) || _hadUnsatisfiedDependencies;
    _active = true;
    // We unregistered our dependencies in deactivate, but never cleared the list.
    // Since we're going to be reused, let's clear our list now.
    _dependencies?.clear();
    _hadUnsatisfiedDependencies = false;
    _updateInheritance();
    assert(() { _debugLifecycleState = _ElementLifecycle.active; return true; }());
    if (_dirty)
      owner.scheduleBuildFor(this);
    if (hadDependencies)
      didChangeDependencies();
  }

  /// Transition from the "active" to the "inactive" lifecycle state.
  ///
  /// The framework calls this method when a previously active element is moved
  /// to the list of inactive elements. While in the inactive state, the element
  /// will not appear on screen. The element can remain in the inactive state
  /// only until the end of the current animation frame. At the end of the
  /// animation frame, if the element has not be reactivated, the framework will
  /// unmount the element.
  ///
  /// This is (indirectly) called by [deactivateChild].
  ///
  /// See the lifecycle documentation for [Element] for additional information.
  @mustCallSuper
  void deactivate() {
    assert(_debugLifecycleState == _ElementLifecycle.active);
    assert(widget != null);
    assert(depth != null);
    assert(_active);
    if (_dependencies != null && _dependencies.isNotEmpty) {
      for (InheritedElement dependency in _dependencies)
        dependency._dependents.remove(this);
      // For expediency, we don't actually clear the list here, even though it's
      // no longer representative of what we are registered with. If we never
      // get re-used, it doesn't matter. If we do, then we'll clear the list in
      // activate(). The benefit of this is that it allows Element's activate()
      // implementation to decide whether to rebuild based on whether we had
      // dependencies here.
    }
    _inheritedWidgets = null;
    _active = false;
    assert(() { _debugLifecycleState = _ElementLifecycle.inactive; return true; }());
  }

  /// Called, in debug mode, after children have been deactivated (see [deactivate]).
  ///
  /// This method is not called in release builds.
  @mustCallSuper
  void debugDeactivated() {
    assert(_debugLifecycleState == _ElementLifecycle.inactive);
  }

  /// Transition from the "inactive" to the "defunct" lifecycle state.
  ///
  /// Called when the framework determines that an inactive element will never
  /// be reactivated. At the end of each animation frame, the framework calls
  /// [unmount] on any remaining inactive elements, preventing inactive elements
  /// from remaining inactive for longer than a single animation frame.
  ///
  /// After this function is called, the element will not be incorporated into
  /// the tree again.
  ///
  /// See the lifecycle documentation for [Element] for additional information.
  @mustCallSuper
  void unmount() {
    assert(_debugLifecycleState == _ElementLifecycle.inactive);
    assert(widget != null);
    assert(depth != null);
    assert(!_active);
    if (widget.key is GlobalKey) {
      final GlobalKey key = widget.key;
      key._unregister(this);
    }
    assert(() { _debugLifecycleState = _ElementLifecycle.defunct; return true; }());
  }

  @override
  RenderObject findRenderObject() => renderObject;

  @override
  Size get size {
    assert(() {
      if (_debugLifecycleState != _ElementLifecycle.active) {
        // TODO(jacobr): is this a good separation into contract and violation?
        // I have added a line of white space.
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('Cannot get size of inactive element.'),
          ErrorDescription(
            'In order for an element to have a valid size, the element must be '
            'active, which means it is part of the tree.\n'
            'Instead, this element is in the $_debugLifecycleState state.'
          ),
          describeElement('The size getter was called for the following element')
        ]);
      }
      if (owner._debugBuilding) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('Cannot get size during build.'),
          ErrorDescription(
            'The size of this render object has not yet been determined because '
            'the framework is still in the process of building widgets, which '
            'means the render tree for this frame has not yet been determined. '
            'The size getter should only be called from paint callbacks or '
            'interaction event handlers (e.g. gesture callbacks).'
          ),
          ErrorSpacer(),
          ErrorHint(
            'If you need some sizing information during build to decide which '
            'widgets to build, consider using a LayoutBuilder widget, which can '
            'tell you the layout constraints at a given location in the tree. See '
            '<https://api.flutter.dev/flutter/widgets/LayoutBuilder-class.html> '
            'for more details.'
          ),
          ErrorSpacer(),
          describeElement('The size getter was called for the following element')
        ]);
      }
      return true;
    }());
    final RenderObject renderObject = findRenderObject();
    assert(() {
      if (renderObject == null) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('Cannot get size without a render object.'),
          ErrorHint(
            'In order for an element to have a valid size, the element must have '
            'an associated render object. This element does not have an associated '
            'render object, which typically means that the size getter was called '
            'too early in the pipeline (e.g., during the build phase) before the '
            'framework has created the render tree.'
          ),
          describeElement('The size getter was called for the following element')
        ]);
      }
      if (renderObject is RenderSliver) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('Cannot get size from a RenderSliver.'),
          ErrorHint(
            'The render object associated with this element is a '
            '${renderObject.runtimeType}, which is a subtype of RenderSliver. '
            'Slivers do not have a size per se. They have a more elaborate '
            'geometry description, which can be accessed by calling '
            'findRenderObject and then using the "geometry" getter on the '
            'resulting object.'
          ),
          describeElement('The size getter was called for the following element'),
          renderObject.describeForError('The associated render sliver was'),
        ]);
      }
      if (renderObject is! RenderBox) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('Cannot get size from a render object that is not a RenderBox.'),
          ErrorHint(
            'Instead of being a subtype of RenderBox, the render object associated '
            'with this element is a ${renderObject.runtimeType}. If this type of '
            'render object does have a size, consider calling findRenderObject '
            'and extracting its size manually.'
          ),
          describeElement('The size getter was called for the following element'),
          renderObject.describeForError('The associated render object was')
        ]);
      }
      final RenderBox box = renderObject;
      if (!box.hasSize) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('Cannot get size from a render object that has not been through layout.'),
          ErrorHint(
            'The size of this render object has not yet been determined because '
            'this render object has not yet been through layout, which typically '
            'means that the size getter was called too early in the pipeline '
            '(e.g., during the build phase) before the framework has determined '
           'the size and position of the render objects during layout.'
          ),
          describeElement('The size getter was called for the following element'),
          box.describeForError('The render object from which the size was to be obtained was')
        ]);
      }
      if (box.debugNeedsLayout) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('Cannot get size from a render object that has been marked dirty for layout.'),
          ErrorHint(
            'The size of this render object is ambiguous because this render object has '
            'been modified since it was last laid out, which typically means that the size '
            'getter was called too early in the pipeline (e.g., during the build phase) '
            'before the framework has determined the size and position of the render '
            'objects during layout.'
          ),
          describeElement('The size getter was called for the following element'),
          box.describeForError('The render object from which the size was to be obtained was'),
          ErrorHint(
            'Consider using debugPrintMarkNeedsLayoutStacks to determine why the render '
            'object in question is dirty, if you did not expect this.'
          ),
        ]);
      }
      return true;
    }());
    if (renderObject is RenderBox)
      return renderObject.size;
    return null;
  }

  Map<Type, InheritedElement> _inheritedWidgets;
  Set<InheritedElement> _dependencies;
  bool _hadUnsatisfiedDependencies = false;

  bool _debugCheckStateIsActiveForAncestorLookup() {
    assert(() {
      if (_debugLifecycleState != _ElementLifecycle.active) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('Looking up a deactivated widget\'s ancestor is unsafe.'),
          ErrorDescription(
            'At this point the state of the widget\'s element tree is no longer '
            'stable.'
          ),
          ErrorHint(
            'To safely refer to a widget\'s ancestor in its dispose() method, '
            'save a reference to the ancestor by calling inheritFromWidgetOfExactType() '
            'in the widget\'s didChangeDependencies() method.'
          )
        ]);
      }
      return true;
    }());
    return true;
  }

  @override
  InheritedWidget inheritFromElement(InheritedElement ancestor, { Object aspect }) {
    assert(ancestor != null);
    _dependencies ??= HashSet<InheritedElement>();
    _dependencies.add(ancestor);
    ancestor.updateDependencies(this, aspect);
    return ancestor.widget;
  }

  @override
  InheritedWidget inheritFromWidgetOfExactType(Type targetType, { Object aspect }) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    final InheritedElement ancestor = _inheritedWidgets == null ? null : _inheritedWidgets[targetType];
    if (ancestor != null) {
      assert(ancestor is InheritedElement);
      return inheritFromElement(ancestor, aspect: aspect);
    }
    _hadUnsatisfiedDependencies = true;
    return null;
  }

  @override
  InheritedElement ancestorInheritedElementForWidgetOfExactType(Type targetType) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    final InheritedElement ancestor = _inheritedWidgets == null ? null : _inheritedWidgets[targetType];
    return ancestor;
  }

  void _updateInheritance() {
    assert(_active);
    _inheritedWidgets = _parent?._inheritedWidgets;
  }

  @override
  Widget ancestorWidgetOfExactType(Type targetType) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    Element ancestor = _parent;
    while (ancestor != null && ancestor.widget.runtimeType != targetType)
      ancestor = ancestor._parent;
    return ancestor?.widget;
  }

  @override
  State ancestorStateOfType(TypeMatcher matcher) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    Element ancestor = _parent;
    while (ancestor != null) {
      if (ancestor is StatefulElement && matcher.check(ancestor.state))
        break;
      ancestor = ancestor._parent;
    }
    final StatefulElement statefulAncestor = ancestor;
    return statefulAncestor?.state;
  }

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

  @override
  void visitAncestorElements(bool visitor(Element element)) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    Element ancestor = _parent;
    while (ancestor != null && visitor(ancestor))
      ancestor = ancestor._parent;
  }

  /// Called when a dependency of this element changes.
  ///
  /// The [inheritFromWidgetOfExactType] registers this element as depending on
  /// inherited information of the given type. When the information of that type
  /// changes at this location in the tree (e.g., because the [InheritedElement]
  /// updated to a new [InheritedWidget] and
  /// [InheritedWidget.updateShouldNotify] returned true), the framework calls
  /// this function to notify this element of the change.
  @mustCallSuper
  void didChangeDependencies() {
    assert(_active); // otherwise markNeedsBuild is a no-op
    assert(_debugCheckOwnerBuildTargetExists('didChangeDependencies'));
    markNeedsBuild();
  }

  bool _debugCheckOwnerBuildTargetExists(String methodName) {
    assert(() {
      if (owner._debugCurrentBuildTarget == null) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary(
            '$methodName for ${widget.runtimeType} was called at an '
            'inappropriate time.'
          ),
          ErrorDescription('It may only be called while the widgets are being built.'),
          ErrorHint(
            'A possible cause of this error is when $methodName is called during '
            'one of:\n'
            ' * network I/O event\n'
            ' * file I/O event\n'
            ' * timer\n'
            ' * microtask (caused by Future.then, async/await, scheduleMicrotask)'
          )
        ]);
      }
      return true;
    }());
    return true;
  }

  /// Returns a description of what caused this element to be created.
  ///
  /// Useful for debugging the source of an element.
  String debugGetCreatorChain(int limit) {
    final List<String> chain = <String>[];
    Element node = this;
    while (chain.length < limit && node != null) {
      chain.add(node.toStringShort());
      node = node._parent;
    }
    if (node != null)
      chain.add('\u22EF');
    return chain.join(' \u2190 ');
  }

  /// Returns the parent chain from this element back to the root of the tree.
  ///
  /// Useful for debug display of a tree of Elements with only nodes in the path
  /// from the root to this Element expanded.
  List<Element> debugGetDiagnosticChain() {
    final List<Element> chain = <Element>[this];
    Element node = _parent;
    while (node != null) {
      chain.add(node);
      node = node._parent;
    }
    return chain;
  }

  /// A short, textual description of this element.
  @override
  String toStringShort() {
    return widget != null ? '${widget.toStringShort()}' : '[$runtimeType]';
  }

  @override
  DiagnosticsNode toDiagnosticsNode({ String name, DiagnosticsTreeStyle style }) {
    return _ElementDiagnosticableTreeNode(
      name: name,
      value: this,
      style: style,
    );
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.defaultDiagnosticsTreeStyle= DiagnosticsTreeStyle.dense;
    properties.add(ObjectFlagProperty<int>('depth', depth, ifNull: 'no depth'));
    properties.add(ObjectFlagProperty<Widget>('widget', widget, ifNull: 'no widget'));
    if (widget != null) {
      properties.add(DiagnosticsProperty<Key>('key', widget?.key, showName: false, defaultValue: null, level: DiagnosticLevel.hidden));
      widget.debugFillProperties(properties);
    }
    properties.add(FlagProperty('dirty', value: dirty, ifTrue: 'dirty'));
    if (_dependencies != null && _dependencies.isNotEmpty) {
      final List<DiagnosticsNode> diagnosticsDependencies = _dependencies
        .map((InheritedElement element) => element.widget.toDiagnosticsNode(style: DiagnosticsTreeStyle.sparse))
        .toList();
      properties.add(DiagnosticsProperty<List<DiagnosticsNode>>('dependencies', diagnosticsDependencies));
    }
  }

  @override
  List<DiagnosticsNode> debugDescribeChildren() {
    final List<DiagnosticsNode> children = <DiagnosticsNode>[];
    visitChildren((Element child) {
      if (child != null) {
        children.add(child.toDiagnosticsNode());
      } else {
        children.add(DiagnosticsNode.message('<null child>'));
      }
    });
    return children;
  }

  /// Returns true if the element has been marked as needing rebuilding.
  bool get dirty => _dirty;
  bool _dirty = true;

  // Whether this is in owner._dirtyElements. This is used to know whether we
  // should be adding the element back into the list when it's reactivated.
  bool _inDirtyList = false;

  // Whether we've already built or not. Set in [rebuild].
  bool _debugBuiltOnce = false;

  // We let widget authors call setState from initState, didUpdateWidget, and
  // build even when state is locked because its convenient and a no-op anyway.
  // This flag ensures that this convenience is only allowed on the element
  // currently undergoing initState, didUpdateWidget, or build.
  bool _debugAllowIgnoredCallsToMarkNeedsBuild = false;
  bool _debugSetAllowIgnoredCallsToMarkNeedsBuild(bool value) {
    assert(_debugAllowIgnoredCallsToMarkNeedsBuild == !value);
    _debugAllowIgnoredCallsToMarkNeedsBuild = value;
    return true;
  }

  /// Marks the element as dirty and adds it to the global list of widgets to
  /// rebuild in the next frame.
  ///
  /// Since it is inefficient to build an element twice in one frame,
  /// applications and widgets should be structured so as to only mark
  /// widgets dirty during event handlers before the frame begins, not during
  /// the build itself.
  void markNeedsBuild() {
    assert(_debugLifecycleState != _ElementLifecycle.defunct);
    if (!_active)
      return;
    assert(owner != null);
    assert(_debugLifecycleState == _ElementLifecycle.active);
    assert(() {
      if (owner._debugBuilding) {
        assert(owner._debugCurrentBuildTarget != null);
        assert(owner._debugStateLocked);
        if (_debugIsInScope(owner._debugCurrentBuildTarget))
          return true;
        if (!_debugAllowIgnoredCallsToMarkNeedsBuild) {
          final List<DiagnosticsNode> information = <DiagnosticsNode>[
            ErrorSummary('setState() or markNeedsBuild() called during build.'),
            ErrorDescription(
              'This ${widget.runtimeType} widget cannot be marked as needing to build because the framework '
              'is already in the process of building widgets.  A widget can be marked as '
              'needing to be built during the build phase only if one of its ancestors '
              'is currently building. This exception is allowed because the framework '
              'builds parent widgets before children, which means a dirty descendant '
              'will always be built. Otherwise, the framework might not visit this '
              'widget during this build phase.'
            ),
            describeElement(
              'The widget on which setState() or markNeedsBuild() was called was',
            )
          ];
          if (owner._debugCurrentBuildTarget != null)
            information.add(owner._debugCurrentBuildTarget.describeWidget('The widget which was currently being built when the offending call was made was'));
          throw FlutterError.fromParts(information);
        }
        assert(dirty); // can only get here if we're not in scope, but ignored calls are allowed, and our call would somehow be ignored (since we're already dirty)
      } else if (owner._debugStateLocked) {
        assert(!_debugAllowIgnoredCallsToMarkNeedsBuild);
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('setState() or markNeedsBuild() called when widget tree was locked.'),
          ErrorDescription(
            'This ${widget.runtimeType} widget cannot be marked as needing to build '
            'because the framework is locked.'
          ),
          describeElement('The widget on which setState() or markNeedsBuild() was called was')
        ]);
      }
      return true;
    }());
    if (dirty)
      return;
    _dirty = true;
    owner.scheduleBuildFor(this);
  }

  /// Called by the [BuildOwner] when [BuildOwner.scheduleBuildFor] has been
  /// called to mark this element dirty, by [mount] when the element is first
  /// built, and by [update] when the widget has changed.
  void rebuild() {
    assert(_debugLifecycleState != _ElementLifecycle.initial);
    if (!_active || !_dirty)
      return;
    assert(() {
      if (debugOnRebuildDirtyWidget != null) {
        debugOnRebuildDirtyWidget(this, _debugBuiltOnce);
      }
      if (debugPrintRebuildDirtyWidgets) {
        if (!_debugBuiltOnce) {
          debugPrint('Building $this');
          _debugBuiltOnce = true;
        } else {
          debugPrint('Rebuilding $this');
        }
      }
      return true;
    }());
    assert(_debugLifecycleState == _ElementLifecycle.active);
    assert(owner._debugStateLocked);
    Element debugPreviousBuildTarget;
    assert(() {
      debugPreviousBuildTarget = owner._debugCurrentBuildTarget;
      owner._debugCurrentBuildTarget = this;
      return true;
    }());
    performRebuild();
    assert(() {
      assert(owner._debugCurrentBuildTarget == this);
      owner._debugCurrentBuildTarget = debugPreviousBuildTarget;
      return true;
    }());
    assert(!_dirty);
  }

  /// Called by rebuild() after the appropriate checks have been made.
  @protected
  void performRebuild();
}

```




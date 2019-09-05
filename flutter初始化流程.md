[TOC]
# 绑定根Widget,Element,RenderObject
```dart
/// app入口
void main() => runApp(MyApp());

void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..attachRootWidget(app)
    ..scheduleWarmUpFrame();
}
   
/// 在 WidgetsFlutterBinding.ensureInitialized()中的RendererBinding中，首先实例化了
/// PipelineOwner类：
/// _pipelineOwner = PipelineOwner(
///      onNeedVisualUpdate: ensureVisualUpdate,
///      onSemanticsOwnerCreated: _handleSemanticsOwnerCreated,
///      onSemanticsOwnerDisposed: _handleSemanticsOwnerDisposed
/// );
///
/// 其中
/// void ensureVisualUpdate() {
///  switch (schedulerPhase) {
///      case SchedulerPhase.idle:
///      case SchedulerPhase.postFrameCallbacks:
///        scheduleFrame();
///        return;
///      case SchedulerPhase.transientCallbacks:
///      case SchedulerPhase.midFrameMicrotasks:
///      case SchedulerPhase.persistentCallbacks:
///        return;
///   }
/// }
///
/// 然后为window绑定了回调方法
/// final Window window = Window._();
///
/// window
///      ..onMetricsChanged = handleMetricsChanged // 当屏幕像素高度改变的回调
///      ..onTextScaleFactorChanged = handleTextScaleFactorChanged // 字体大小改变回调
///      ..onPlatformBrightnessChanged = handlePlatformBrightnessChanged // 亮度改变
///      ..onSemanticsEnabledChanged = _handleSemanticsEnabledChanged 
///      ..onSemanticsAction = _handleSemanticsAction;
/// WidgetsBinding
///
/// 然后初始化widgetsBinding mixin，
/// 并创建了BuildOwner对象
/// mixin WidgetsBinding on BindingBase, SchedulerBinding, GestureBinding, 
/// RendererBinding, ///SemanticsBinding {
///	...
///    BuildOwner get buildOwner => _buildOwner;
///    final BuildOwner _buildOwner = BuildOwner();
/// ...
/// }
///
/// 然后为其绑定了方法
/// buildOwner.onBuildScheduled = _handleBuildScheduled;
///     window.onLocaleChanged = handleLocaleChanged;
///     window.onAccessibilityFeaturesChanged = handleAccessibilityFeaturesChanged;
///
///
///

... 

/// WidgetsFlutterBinding.attachRootWidget
/// _renderViewElement 即为 element树的根节点    
void attachRootWidget(Widget rootWidget) {
    /// _renderViewElement即为element树的根节点
    _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
        container: renderView,
        debugShortDescription: '[root]',
        child: rootWidget,/// 作为 RenderObjectToWidgetAdapter.child
    ).attachToRenderTree(buildOwner, renderViewElement);
} 

///  RenderObjectToWidgetAdapter.attachToRenderTree
RenderObjectToWidgetElement<T> attachToRenderTree(BuildOwner owner, [RenderObjectToWidgetElement<T> element ]) {
    if (element == null) {
        ...
        owner.buildScope(element, () {
            element.mount(null, null);
        });
    } else {
       ...
    }
    return element;
}


/// BuildOwner.buildScope
void buildScope(Element context, [ VoidCallback callback ]) {
    ...
    try {
        _scheduledFlushDirtyElements = true;
        if (callback != null) {
            ...
            try {
                callback();
                /// 调用传入的 callback，即element.mount(null, null);
            } finally {
              ...
            }
        }
    } finally {
      ...
    }
}

/// RenderObjectToWidgetElement.mount
void mount(Element parent, dynamic newSlot) {
    assert(parent == null);
    super.mount(parent, newSlot);
    _rebuild();
}


/// 三次super.mount后调用到Element.mount
void mount(Element parent, dynamic newSlot) {
    ...
    _parent = parent; /// null
    _slot = newSlot;  /// null 
    _depth = _parent != null ? _parent.depth + 1 : 1;  /// 1
    _active = true;
    if (parent != null) 
        _owner = parent.owner;
    if (widget.key is GlobalKey) {
        final GlobalKey key = widget.key;
        key._register(this);
    }
    _updateInheritance();/// 见后文
    assert(() { _debugLifecycleState = _ElementLifecycle.active; return true; }());
}

/// 返回到第二次super.mount，调用到 RenderObjectElement.mount
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    /// 这里的widget是RenderObjectToWidgetAdapter
    /// 重载了方法 createRenderObject(BuildContext context) => container; 
    /// 这里的  container 即 container: renderView,
    _renderObject = widget.createRenderObject(this);
    assert(() { _debugUpdateRenderObjectOwner(); return true; }());
    assert(_slot == newSlot);
    attachRenderObject(newSlot);
    _dirty = false;
  }


/// RenderObjectToWidgetElement._rebuild
void _rebuild() {
    try {
        _child = updateChild(_child, widget.child, _rootChildSlot);
        /// 此时_child （子element）为null
        /// widget 是RenderObjectToWidgetAdapter
        /// widget.child就是runApp()里传入的widget
        /// _rootChildSlot 为 static const Object _rootChildSlot = Object();
        /// _child 被修改为 updateChild 返回的子element
        assert(_child != null);
    } 
    ...
}

/// Element.updateChild
Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
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
    /// 由于child == null && newWidget != null,调用inflateWidget，返回新建的子element
    return inflateWidget(newWidget, newSlot);
}

/// Element.infalateWidget
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
    /// runApp()传入的widget的createElement()在这里被调用
    assert(() { _debugCheckForCycles(newChild); return true; }());
    newChild.mount(this, newSlot);
    /// 调用子element的mount,递归地建立子element
    assert(newChild._debugLifecycleState == _ElementLifecycle.active);
    return newChild;
}


/// 至此，根节点的绑定完成
```



# 调用顺序

- 父elemen先调用mount方法建立与爷element的关联（获取buildOwner等）
- （调用其自身的performRebuild方法）
- 父element调用其引用的widget的.build方法（即 widgetd 的 build(BuildContext context）。这里的 context 即父element本身)，得到子widget
- 之后立即调用__child = updateChild(_child, built, slot)方法，试图用新widget去更新旧widget
- （若_child为null，即还没有子element）然后父element再调用widget的createElement 方法（一般返回的是 XXXElement(this),即调用 XXXElement的构造器)，得到子element
- 子element重复以上步骤





# 以StatelessElement 为例

```dart
/// ComponentElement.mount
void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    ...
}

/// 先super.mount调用element.mount建立父子element的关联
void mount(Element parent, dynamic newSlot) {
    ...
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

/// ComponentElement.mount
void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    ...
    _firstBuild();
    ...
}


/// 然后是ComponentElement.firstBuild
void _firstBuild() {
    rebuild();
}

/// ComponetElement.rebuild
void rebuild() {
  	...
    performRebuild();
    ...
}

/// ComponentElement.performRebuild
void performRebuild() {
    ...
    Widget built;
    try {
        built = build();
        ...
    }
    ...
}

/// 调用在StatelessWidget中重载的 build 方法：
class StatelessElement extends ComponentElement {
  
  StatelessElement(StatelessWidget widget) : super(widget);

  @override
  StatelessWidget get widget => super.widget;

  @override
  Widget build() => widget.build(this);
  /// 调用widget.build
  
  ...  
}

/// ComponentElement.performRebuild
void performRebuild() {
    ...
    try {
       _child = updateChild(_child, built, slot);
       assert(_child != null);
    } 
    ...
}


/// Element.updateChild
Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
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
    /// 由于child == null && newWidget != null,调用inflateWidget，返回新建的子element
    return inflateWidget(newWidget, newSlot);
}

/// Element.infalateWidget
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
    /// runApp()传入的widget的createElement()在这里被调用
    assert(() { _debugCheckForCycles(newChild); return true; }());
    newChild.mount(this, newSlot);
    /// 调用子element的mount,建立子element
    assert(newChild._debugLifecycleState == _ElementLifecycle.active);
    return newChild;
}

///又回到了element.updateChild方法，继续构建子element
```



# 以StatelessElement 为例

```dart
/// 在这个方法以前，流程同StatelessWidget
/// ComponentElement.performRebuild
void performRebuild() {
    ...
    Widget built;
    try {
        built = build();
        ...
    }
    ...
}

/// StatefulWidget.build
/// 需要注意的是，在父element的绑定阶段，调用widget.createElement
/// 即StatefulWidget.createElement(this),自动调用widget.createState()，从而产生state

/// StatefulElement的构造器
StatefulElement(StatefulWidget widget)
    : _state = widget.createState(),
	  super(widget) 
{
    _state._element = this;
    _state._widget = widget;
   /// State 持有对widget和element的引用
}

/// StatefulWidget.build
@override
Widget build() => state.build(this);

/// ComponentElement.performRebuild
void performRebuild() {
    ...
    try {
       _child = updateChild(_child, built, slot);
       assert(_child != null);
    } 
    ...
}

///又回到了element.updateChild方法，继续构建子element
```



# 以InheritedElement为例

```da

```





# 遗传映射表

```dart
void _updateInheritance() {
    assert(_active);
    _inheritedWidgets = _parent?._inheritedWidgets;
}

/// 其中
Map<Type, InheritedElement> _inheritedWidgets;
```


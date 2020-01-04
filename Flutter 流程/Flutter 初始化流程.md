# Flutter 初始化流程

[TOC]



## Widget 树，ELement 树，RenderObject 树的根节点

### RenderView

```dart
/// RenderOBject 树的根节点
///
/// 注意混入了 RenderObjectWithChildMixin，这个 mixin 提供了单孩子渲染对象的管理
class RenderView extends RenderObject with RenderObjectWithChildMixin<RenderBox> {
  /// 这个类要求其孩子必须为 RenderBox
  RenderView({
    RenderBox child,
    @required ViewConfiguration configuration,
    @required ui.Window window,
  }) : _configuration = configuration,
       _window = window {
    this.child = child;
  }

  Size get size => _size;
  Size _size = Size.zero;

  /// 根节点的布局约束配置
  /// 包含了屏幕物理像素宽度和屏幕像素比
  ViewConfiguration get configuration => _configuration;
  ViewConfiguration _configuration;
  
  /// 更改这个配置的时候会触发全局的布局绘制更新
  set configuration(ViewConfiguration value) {
    if (configuration == value) return;
    _configuration = value;
    replaceRootLayer(_updateMatricesAndCreateNewRootLayer());
    markNeedsLayout();
  }

  final ui.Window _window;

  bool automaticSystemUiAdjustment = true;

  /// 为第一帧的渲染做准备，以此来驱动渲染流水线
  ///
  /// 这个方法只应该被调用一次且必须在改变 [configuration] 之前调用
  /// 这个方法通常在调用了构造函数之后立刻调用
  ///
  /// 这个方法通常不会实际地调度第一帧。通常，在这个方法完成之后调用 owner 的
  /// [PipelineOwner.requestVisualUpdate] 来完成调度
  void prepareInitialFrame() {
    scheduleInitialLayout();
    scheduleInitialPaint(_updateMatricesAndCreateNewRootLayer());
  }

  /// 根节点的变化矩阵
  Matrix4 _rootTransform;

  /// 将 ViewConfiguration 转化成根节点的变换矩阵，并根据此矩阵来创建一个新的合成层并返回
  Layer _updateMatricesAndCreateNewRootLayer() {
    _rootTransform = configuration.toMatrix();
    final ContainerLayer rootLayer = TransformLayer(transform: _rootTransform);
    /// 新建的层会将这个 RenderView 作为其 owner
    rootLayer.attach(this);
    return rootLayer;
  }

  /// 显然，根RenderObject 的大小是固定的，不会随时间而改变。故不能使用这个方法  
  @override
  void performResize() {
    assert(false);
  }

  /// 从根节点向下递归进行布局
  @override
  void performLayout() {
    _size = configuration.size;
    if (child != null)
      /// 注意，这里给出的约束是紧缩的
      child.layout(BoxConstraints.tight(_size));
  }

  /// 确定给定位置上的所有的渲染对象的集合
  ///
  /// 如果给定的点包含在此渲染对象或其子渲染对象中，则返回 true。
  /// 将包含该点的任何渲染对象添加到给定的 HitTestResult 中。随后，手势竞技场会做出决断
  ///
  /// [position] 参数处于在 render 视图的坐标系统中，也就是说，在逻辑像素中。
  /// 这并不一定是根 [Layer] 所期望的坐标系统，根 [Layer] 坐标系统通常位于物理(设备)像素中。
  bool hitTest(HitTestResult result, { Offset position }) {
    if (child != null)
      child.hitTest(BoxHitTestResult.wrap(result), position: position);
    result.add(HitTestEntry(this));
    /// 显然对于任何一点，都处于根渲染对象的范围内
    return true;
  }

  /// 鼠标相关
  Iterable<MouseTrackerAnnotation> hitTestMouseTrackers(Offset position) {
    return layer.findAll<MouseTrackerAnnotation>(position * 
                                                 configuration.devicePixelRatio);
  }

  /// 显然，根节点是重绘边界  
  @override
  bool get isRepaintBoundary => true;

  /// 根节点的绘制，直接在当前 PaintContext 上绘制子树
  @override
  void paint(PaintingContext context, Offset offset) {
    if (child != null)
      context.paintChild(child, offset);
  }

  /// 把当前变换矩阵加入到给定的变换矩阵上，也就是说，将两个矩阵相乘
  @override
  void applyPaintTransform(RenderBox child, Matrix4 transform) {
    transform.multiply(_rootTransform);
  }

  /// 将合成层树提交给引擎
  ///
  /// 实际上是使屏幕显示一帧
  void compositeFrame() {
    Timeline.startSync('Compositing', arguments: timelineWhitelistArguments);
    try {
      final ui.SceneBuilder builder = ui.SceneBuilder();
      final ui.Scene scene = layer.buildScene(builder);
      if (automaticSystemUiAdjustment)
        _updateSystemChrome();
      _window.render(scene);
      scene.dispose();
    } finally {
      Timeline.finishSync();
    }
  }

  void _updateSystemChrome() {
    final Rect bounds = paintBounds;
    final Offset top = Offset(bounds.center.dx, _window.padding.top / 
                              _window.devicePixelRatio);
    final Offset bottom = Offset(bounds.center.dx, bounds.center.dy - 
                                 _window.padding.bottom / _window.devicePixelRatio);
    final SystemUiOverlayStyle upperOverlayStyle = layer.find<SystemUiOverlayStyle>(top);
    // Only android has a customizable system navigation bar.
    SystemUiOverlayStyle lowerOverlayStyle;
    switch (defaultTargetPlatform) {
      case TargetPlatform.android:
        lowerOverlayStyle = layer.find<SystemUiOverlayStyle>(bottom);
        break;
      case TargetPlatform.iOS:
      case TargetPlatform.fuchsia:
        break;
    }
    // If there are no overlay styles in the UI don't bother updating.
    if (upperOverlayStyle != null || lowerOverlayStyle != null) {
      final SystemUiOverlayStyle overlayStyle = SystemUiOverlayStyle(
        statusBarBrightness: upperOverlayStyle?.statusBarBrightness,
        statusBarIconBrightness: upperOverlayStyle?.statusBarIconBrightness,
        statusBarColor: upperOverlayStyle?.statusBarColor,
        systemNavigationBarColor: lowerOverlayStyle?.systemNavigationBarColor,
        systemNavigationBarDividerColor: 
          lowerOverlayStyle?.systemNavigationBarDividerColor,
        systemNavigationBarIconBrightness: 
          lowerOverlayStyle?.systemNavigationBarIconBrightness,
      );
      SystemChrome.setSystemUIOverlayStyle(overlayStyle);
    }
  }

  /// （预测）的绘制边界，即屏幕的物理像素（？？）大小
  @override
  Rect get paintBounds => Offset.zero & (size * configuration.devicePixelRatio);

  @override
  Rect get semanticBounds {
    assert(_rootTransform != null);
    return MatrixUtils.transformRect(_rootTransform, Offset.zero & size);
  }
}
```



### RenderObjectToWidgetElement

```dart
/// Element 树的根节点
/// 桥接 [RenderObject] 到 [Element] 树
class RenderObjectToWidgetElement<T extends RenderObject> 
	extends RootRenderObjectElement 
{
  RenderObjectToWidgetElement(RenderObjectToWidgetAdapter<T> widget) : 	super(widget);

  @override
  RenderObjectToWidgetAdapter<T> get widget => super.widget;

  Element _child;

  /// 根 Element 的 slots	
  static const Object _rootChildSlot = Object();

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

  /// 注意到以下的两个方法都会调用到 _rebuild（通常来说，mount 也可以看作一次重建，从无到有的重建）
  @override
  void mount(Element parent, dynamic newSlot) {
  	/// 显然，这个 element 没有 parent element
    assert(parent == null);
    super.mount(parent, newSlot);
    _rebuild();
  }

  @override
  void update(RenderObjectToWidgetAdapter<T> newWidget) {
    super.update(newWidget);
    /// RenderObjectToWidgetAdapter 是不可变的，不能更换的
    assert(widget == newWidget);
    _rebuild();
  }

  // When we are assigned a new widget, we store it here
  // until we are ready to update to it.
  Widget _newWidget;

  @override
  void performRebuild() {
    if (_newWidget != null) {
      final Widget newWidget = _newWidget;
      _newWidget = null;
      update(newWidget);
    }
    super.performRebuild();
  }

  /// 重建
  /// 直接调用 updateChild 来构建 element 子树，其中
  /// 1.widget.child 是我们传递给 runApp 的 widget，它会被自动作为 RenderObjectToWidgetAdapter 的子树
  /// 2._rootChildSlot 是一个常量
  void _rebuild() {
    try {
      _child = updateChild(_child, widget.child, _rootChildSlot);
    } catch (exception, stack) {
      final Widget error = ErrorWidget.builder(details);
      _child = updateChild(null, error, _rootChildSlot);
    }
  }

  @override
  RenderObjectWithChildMixin<T> get renderObject => super.renderObject;

  /// 插入子渲染对象（即替换唯一的渲染对象子节点）
  @override
  void insertChildRenderObject(RenderObject child, dynamic slot) {
    renderObject.child = child;
  }

  @override
  void moveChildRenderObject(RenderObject child, dynamic slot) {
  	/// 这个 element 对应的 RenderObject 只有一个孩子，故不能调用这个方法
    assert(false);
  }

  /// 移除子渲染对象
  @override
  void removeChildRenderObject(RenderObject child) {
    renderObject.child = null;
  }
}
```



###  RenderObjectToWidgetAdapter

```dart
/// widegt 树的根节点
/// 这个 Widget 其实是一个反模式，
/// 通常来说，树的构建流程通常是根据给定的 widget 来创建 element 和 renderObject，
/// 而这个 RenderObjectToWidgetAdapter，顾名思义，是将给定 renderObject 转化成一个 widget（或者说是提供一
/// 个对应适配的 widet）
class RenderObjectToWidgetAdapter<T extends RenderObject> extends RenderObjectWidget {
  RenderObjectToWidgetAdapter({
    this.child,
    /// 这个参数是渲染对象树的根节点，即 RenderView。这里体现的此 widget 是对根渲染对象的适配  
    this.container,
    this.debugShortDescription,
  }) : super(key: GlobalObjectKey(container));

  final Widget child;
  /// 这个 widget 持有 renderview，作为 createRenderView 的返回值  
  final RenderObjectWithChildMixin<T> container;
  final String debugShortDescription;

  /// 建立根 element  
  @override
  RenderObjectToWidgetElement<T> createElement() => RenderObjectToWidgetElement<T>(this);

  /// 建立根 renderObject  
  @override
  RenderObjectWithChildMixin<T> createRenderObject(BuildContext context) => container;

  /// 由于这个 widegt 对应的渲染对象是先于这个 widget 被创建的，故逻辑上不能使用这个 widget 来更新渲染对象
  @override
  void updateRenderObject(BuildContext context, RenderObject renderObject) { }

  /// 
  RenderObjectToWidgetElement<T> attachToRenderTree(
      BuildOwner owner, [ 
      RenderObjectToWidgetElement<T> element ]
  ) {
    /// element == null 意味着这是初始化流程，也就是本篇的分析流程  
    if (element == null) {
      /// 锁定状态，详见 [BuildOwner.locakState]
      owner.lockState(() {
        /// 在 [BuildOwner.locakState] 会调用此回调
        /// 注意，根 element 是在此处被创建的，并将 BuildOnwer 分配给根 elelment
        element = createElement();
        element.assignOwner(owner);
      });  
      /// 此时，就可以首次调用 owner.buildScope 来完成对整棵 element 树的构建
      owner.buildScope(element, () {
          element.mount(null, null);
      });
      /// 随后，会请求一次视觉更新，也就是请求一帧
      /// 会依次进行 构建，布局，绘制等流程
      SchedulerBinding.instance.ensureVisualUpdate();
    /// 否则，将会重建整棵 element 树
    } else {
      element._newWidget = this;
      element.markNeedsBuild();
    }
    /// 返回根 element
    return element;
  }
}
```





## 从 runApp 开始

```dart
/// app入口
void main() => runApp(MyApp());

void runApp(Widget app) {
 /// 以下调用相当于：
 /// WidgetsBinding widgetBinding = WidgetsFlutterBinding.ensureInitialized();
 /// widgetBinding.attachRootWidget(app);
 /// widgetBinding.scheduleWarmUpFrame();
  WidgetsFlutterBinding.ensureInitialized()
    ..attachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```

### ensureInitialized
```dart
/// runApp 首先直接调用到 WidgetBinding 类的静态方法：
/// 第一次构建（运行）的时候，由于 WidgetsBinding.instance == null，首先调用 WidgetsFlutterBinding()
/// 即构建一个 WidgetsFlutterBinding，自动调用其超类的构造函数，即 BindingBase()
static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
        WidgetsFlutterBinding();
    return WidgetsBinding.instance;
}

/// BindingBase 的构建函数：
BindingBase() {
	...
    initInstances();
    ...
    /// 以下调用根据运行模式（如 debug 模式或者 release 模式）来决定一些运行参数
    initServiceExtensions();
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
///      ..onPlatformBrightnessChanged = handlePlatformBrightnessChanged // 亮度改变回调
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
```

### attachRootWidget

```dart
/// WidgetsFlutterBinding.attachRootWidget
/// _renderViewElement 即为 element树的根节点    
void attachRootWidget(Widget rootWidget) {
    /// _renderViewElement即为element树的根节点
    _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
        container: renderView,
        debugShortDescription: '[root]',
        child: rootWidget,/// 作为 RenderObjectToWidgetAdapter.child
        
    /// WidgetsFlutterBinding持有buildowner    
    ).attachToRenderTree(buildOwner, renderViewElement);
} 

///  RenderObjectToWidgetAdapter.attachToRenderTree
RenderObjectToWidgetElement<T> attachToRenderTree(BuildOwner owner, [RenderObjectToWidgetElement<T> element ]) {
     if (element == null) {
        owner.lockState(() {
        /// 创建根element    
        element = createElement();
        element.assignOwner(owner);
      });
      owner.buildScope(element, () {
        element.mount(null, null);
      });
    } else {
       ...
    }
    return element;
}

///BuildOwner.lockState
void lockState(void callback()) {
    try {
        callback();
    }
    ...
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
    super.mount(parent, newSlot);
    _rebuild();
}


/// 三次super.mount后调用到Element.mount
void mount(Element parent, dynamic newSlot) {
    ...
    _parent = parent; /// null
    _slot = newSlot;  /// null 
    _depth = _parent != null ? _parent.depth + 1 : 1;  /// 1
    _active = true; /// 设置为active状态
    if (parent != null) 
        _owner = parent.owner;
    if (widget.key is GlobalKey) {
        final GlobalKey key = widget.key;
        /// 在GlobalKey的映射表（GlobalKey到element的映射）中注册这个element
        key._register(this);
    }
    _updateInheritance();/// 见后文
}

/// 返回到第二次super.mount，调用到 RenderObjectElement.mount
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    /// 这里的widget是RenderObjectToWidgetAdapter
    /// 重载了方法 createRenderObject(BuildContext context) => container; 
    /// 这里的 container 即 renderView,
    _renderObject = widget.createRenderObject(this);
    attachRenderObject(newSlot);
    _dirty = false;
  }

/// 第一次super.mount只调用了super.mount

/// RenderObjectElement.attachRenderObject
void attachRenderObject(dynamic newSlot) {
    assert(_ancestorRenderObjectElement == null);
    _slot = newSlot;
    _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
    _ancestorRenderObjectElement?.insertChildRenderObject(renderObject, newSlot);
    final ParentDataElement<RenderObjectWidget> parentDataElement = 
        _findAncestorParentDataElement();
    if (parentDataElement != null)
        _updateParentData(parentDataElement.widget);
}

/// RenderObjectToWidgetElement._rebuild
void _rebuild() {
    try {
        /// 此时_child （子element）为null
        /// widget 是RenderObjectToWidgetAdapter
        /// widget.child就是runApp()里传入的widget
        /// _rootChildSlot 为 static const Object _rootChildSlot = Object();
        /// _child 被修改为 updateChild 返回的子element
        _child = updateChild(_child, widget.child, _rootChildSlot);
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
            newChild._activateWithParent(this, newSlot);
            final Element updatedChild = updateChild(newChild, newWidget, newSlot);
            return updatedChild;
        }
    }
    /// runApp()传入的widget的createElement()在这里被调用
    final Element newChild = newWidget.createElement();
    /// 调用子element的mount,递归地建立子element
    newChild.mount(this, newSlot);
    return newChild;
}

/// 至此，根节点的绑定完成
```



### scheduleWarmUpFrame

```dart

```



## 递归构建的调用顺序

- 父elemen先调用mount方法建立与爷element的关联（获取buildOwner,挂载RenderObject等）
- （调用其自身的performRebuild方法）
- 父element调用其引用的widget的.build方法（即 widget 的 build(BuildContext context）。这里的 context 即父element本身)，得到子widget
- 之后立即调用 child = updateChild(_child, built, slot)方法，试图用新widget去更新旧widget
- （若_child为null，即还没有子element）然后父element再调用widget的createElement 方法（一般返回的是 XXXElement(this),即调用 XXXElement的构造器)，得到子element
- 子element重复以上步骤



## 构建举例



### 以 StatelessElement 为例

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
  /// 调用widget.build(context)
  
  ...  
}

/// ComponentElement.performRebuild
void performRebuild() {
    ...
    try {
       _child = updateChild(_child, built, slot);
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
    final Key key = newWidget.key;
    if (key is GlobalKey) {
        final Element newChild = _retakeInactiveElement(key, newWidget);
        if (newChild != null) {
            newChild._activateWithParent(this, newSlot);
            final Element updatedChild = updateChild(newChild, newWidget, newSlot);
            return updatedChild;
        }
    }
    /// runApp()传入的widget的createElement()在这里被调用
    final Element newChild = newWidget.createElement();
    /// 调用子element的mount,建立子element
    newChild.mount(this, newSlot);
    return newChild;
}

///又回到了element.updateChild方法，继续构建子element
```



### 以StatefulElement 为例

```dart
/// 在_firstBuild方法以前，流程同StatelessWidget
/// 由于 StatefulElement 重载了 _firstBuild，所以先调用
void _firstBuild() {
    try {
        /// 注意这里实际上调到用state.initState()
        /// 后面进行了return不能是future的检查
        final dynamic debugCheckForReturnedFuture = _state.initState() as dynamic;
    } finally {
        ...
    }
    /// 然后立即调用didChangeDependencies
    _state.didChangeDependencies();
    super._firstBuild();
}

/// 然后是 ComponentElement.firstBuild
void _firstBuild() {
    rebuild();
}

 void rebuild() {
    /// 如果是 非active 或者 非dirty，不重建 
    if (!_active || !_dirty)
      return;
    Element debugPreviousBuildTarget;
    performRebuild();
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

/// State<StatefulWidget>.build
@override
Widget build() => state.build(this);

/// ComponentElement.performRebuild
void performRebuild() {
    ...
    try {
       _child = updateChild(_child, built, slot);
    } 
    ...
}

///又回到了element.updateChild方法，继续构建子element
```



### 以SingleChildRenderObjectWidget为例

```dart
/// mount
@override
void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _child = updateChild(_child, widget.child, null);
}

/// RenderObejcetElement.mount
@override
void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    ...
}

/// Element.mount
void mount(Element parent, dynamic newSlot) {
    ...
}

/// 回到RenderObejcetElement.mount
void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    /// 调用了widget的createRenderObject创建了renderObject
    _renderObject = widget.createRenderObject(this);
    /// 将RenderObject关联到指定的slots上
    attachRenderObject(newSlot);
    _dirty = false;
}

/// RenderObject.attachRenderObject
void attachRenderObject(dynamic newSlot) {
    _slot = newSlot;
    _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
    _ancestorRenderObjectElement?.insertChildRenderObject(renderObject, newSlot);
    final ParentDataElement<RenderObjectWidget> parentDataElement = 
        _findAncestorParentDataElement();
    if (parentDataElement != null)
        _updateParentData(parentDataElement.widget);
}

/// 找到最近的renderObject祖先返回
RenderObjectElement _findAncestorRenderObjectElement() {
    Element ancestor = _parent;
    while (ancestor != null && ancestor is! RenderObjectElement)
        ancestor = ancestor._parent;
    return ancestor;
}

/// 然后调用：（其中renderObject是刚才找到返回的先祖renderObject)
/// _ancestorRenderObjectElement?.insertChildRenderObject(renderObject, newSlot);
insertChildRenderObject(RenderObject child, dynamic slot) {
    final RenderObjectWithChildMixin<RenderObject> renderObject = this.renderObject;
    /// 建立联系（把祖先的后代设为自己）
    renderObject.child = child;
}

/// 然后是递归建立
_child = updateChild(_child, widget.child, null);

```



### 以MultiChildRenderObjectWidget为例

```dart
/// mount
void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    ...
}

/// 同singlechildRenderObjectWidget,此期间会挂载自己的RenderObejct
@override
void mount(Element parent, dynamic newSlot) {...}

/// 回到mount
void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _children = List<Element>(widget.children.length);
    Element previousChild;
    /// 逐个对widget中的child调用inflateWidget
    for (int i = 0; i < _children.length; i += 1) {
        final Element newChild = inflateWidget(widget.children[i], previousChild);
        /// 保存新建的element
        _children[i] = newChild;
        /// 记录前一个child,作为slots
        previousChild = newChild;
    }
}


```

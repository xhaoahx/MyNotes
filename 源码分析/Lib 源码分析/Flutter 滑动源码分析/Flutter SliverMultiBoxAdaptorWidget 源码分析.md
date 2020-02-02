# Flutter SliverMultiBoxAdaptorWidget 源码分析



## SliverMultiBoxAdaptorWidget

```dart
/// 拥有许多盒子孩子的滑块的基类
///
/// 帮助子类使用给定的 [SliverChildDelegate] 来延迟构建它们的孩子
///
/// 被 [delegate] 返回的组件会被缓存，并且，只有在 delegate's [shouldRebuild] 方法返回 ture 的时候，组件的 
/// 会被重新构建
abstract class SliverMultiBoxAdaptorWidget extends SliverWithKeepAliveWidget {
  /// Initializes fields for subclasses.
  const SliverMultiBoxAdaptorWidget({
    Key key,
    @required this.delegate,
  }) : assert(delegate != null),
       super(key: key);

  /// 为这个组件提供孩子列表的 delegate
  ///
  /// 孩子的构建通过 delegate 来延迟，来避免构建比 [Viewport] 可见范围更多的组件
  /// The delegate that provides the children for this widget.
  final SliverChildDelegate delegate;

  @override
  SliverMultiBoxAdaptorElement createElement() => SliverMultiBoxAdaptorElement(this);

  @override
  RenderSliverMultiBoxAdaptor createRenderObject(BuildContext context);

  /// 返回一个估计的最大的所有孩子的滚动举例
  ///
  /// 子类应该重载此函数，如果它们有额外的关于最大滚动距离的信息
  ///
  /// 这个方法被 [SliverMultiBoxAdaptorElement] 使用，来实现 [RenderSliverBoxChildManager] 接口
  /// 
  /// 这个方法的默认实现通过 [SliverChildDelegate.estimateMaxScrollOffset] 延迟给 [delegate]
  double estimateMaxScrollOffset(
    SliverConstraints constraints,
    int firstIndex,
    int lastIndex,
    double leadingScrollOffset,
    double trailingScrollOffset,
  ) {
    return delegate.estimateMaxScrollOffset(
      firstIndex,
      lastIndex,
      leadingScrollOffset,
      trailingScrollOffset,
    );
  }
}

```



## RenderSliverBoxChildManager 

```dart
/// 被 [RenderSliverMultiBoxAdaptor] 使用的用于管理其孩子的委托
///
/// [RenderSliverMultiBoxAdaptor] 对象延迟地具体化它们的孩子来避免使用过多的资源来构架在视窗中不可见的孩子
/// （也就是说，还没有在视窗中可见的孩子将不会被构建）。此委托允许这些对象创建和删除子对象，并估计整个子列表占
/// 用的滚动偏移量范围
abstract class RenderSliverBoxChildManager {
  /// 在布局期间，如果需要添加一个新的孩子，调用此方法。新创建的孩子应该被插入到孩子列表中的一个合适的位置（在
  /// `after` 孩子之后（如果 `after` 为 null，则插入到列表头））。它的下标和滚动偏移将会被自动地设置
  ///
  /// index 参数给出了需要被展示的 child。并且可以要求提供一个下标为负数的元素。例如：如果用户从 child 0 滚动
  /// 到 child 10，并且孩子列表发生变换，例如变得更小，随后当用于向回滚动时，这个方法最终要求提供一个下标为 -1 
  /// 的孩子
  ///
  /// 如果没有对应下标的孩子，什么也不做
  ///
  /// 
  ///
  /// If no child corresponds to `index`, then do nothing.
  ///
  /// 下标 0 表示那个子节点 取决于 [renderslivermultiboxadapter.constraints] 中指定的 
  /// [GrowthDirection]。例如：如果孩子是按照字母排列的，并且 [SliverConstraints.growthDirection] == 
  /// [GrowthDirection.forward]，那么下标 0 指代了 A，并且 下标 25 指代了 Z。于此同时，如果 
  /// SliverConstraints.growthDirection] == [GrowthDirection.reverse]，那么下标 0 指代了 Z，下标 25 指代
  /// 了 A。
  ///
  /// 在 [createChild] 方法期间从 [RenderSliverMultiBoxAdaptor] 移除其他的孩子是有效的（如果它们在此帧中没
  /// 有被创建或者更新）。而添加任何的孩子到此渲染对象上是无效的。
  ///
  /// 如果这个方法不能够创建一个比 0 要大的下标对应的孩子，那么 [computeMaxScrollOffset] 必须要能够返回一个
  /// 精确的值
  void createChild(int index, { @required RenderBox after });

  /// 在 child 列表中移除一个给定的 child
  ///
  /// 被 [RenderSliverMultiBoxAdaptor.collectGarbage] 方法调用，
  ///
  /// 给定的 child 的下标能够通过 [RenderSliverMultiBoxAdaptor.indexOf] 的方法获取。
  void removeChild(RenderBox child);

  double estimateMaxScrollOffset(
    SliverConstraints constraints, {
    int firstIndex,
    int lastIndex,
    double leadingScrollOffset,
    double trailingScrollOffset,
  });

  /// child 列表的总长度
  int get childCount;

  /// 在 [RenderSliverMultiBoxAdaptor.adoptChild] 或者 [RenderSliverMultiBoxAdaptor.move] 被调用
  ///
  /// 子类必须能够确保 [SliverMultiBoxAdaptorParentData.index] 域在此方法返回后能够正确地表示它在孩子列表中
  /// 的下标
  void didAdoptChild(RenderBox child);

  /// Called during layout to indicate whether this object provided insufficient
  /// children for the [RenderSliverMultiBoxAdaptor] to fill the
  /// [SliverConstraints.remainingPaintExtent].
  ///
  /// Typically called unconditionally at the start of layout with false and
  /// then later called with true when the [RenderSliverMultiBoxAdaptor]
  /// fails to create a child required to fill the
  /// [SliverConstraints.remainingPaintExtent].
  ///
  /// Useful for subclasses to determine whether newly added children could
  /// affect the visible contents of the [RenderSliverMultiBoxAdaptor].
  void setDidUnderflow(bool value);

  /// 布局开始之后立即调用
  void didStartLayout() { }

  /// 布局完成之后立即调用
  void didFinishLayout() { }
}
```



## SliverMultiBoxAdaptorElement

```dart
/// 一个能够为 [SliverMultiBoxAdaptorWidget] 延迟构建其孩子的 element [SliverMultiBoxAdaptorWidget].
///
/// 实现了 [RenderSliverBoxChildManager]
class SliverMultiBoxAdaptorElement 
    extends RenderObjectElement 
    implements RenderSliverBoxChildManager 
{
  /// Creates an element that lazily builds children for the given widget.
  SliverMultiBoxAdaptorElement(SliverMultiBoxAdaptorWidget widget) : super(widget);

  @override
  SliverMultiBoxAdaptorWidget get widget => super.widget as SliverMultiBoxAdaptorWidget;

  @override
  RenderSliverMultiBoxAdaptor get renderObject => super.renderObject as RenderSliverMultiBoxAdaptor;

  /// 对应 widget 发生了更新的时候调用
  @override
  void update(covariant SliverMultiBoxAdaptorWidget newWidget) {
    final SliverMultiBoxAdaptorWidget oldWidget = widget;
    super.update(newWidget);
    final SliverChildDelegate newDelegate = newWidget.delegate;
    final SliverChildDelegate oldDelegate = oldWidget.delegate;
    /// 如果 delegate 的引用（或是重载的 == 方法），或者其运行类型，或者其 shouldRebuild 方法返回 true
    /// 则需要重建
    if (  newDelegate != oldDelegate 
       && (  newDelegate.runtimeType != oldDelegate.runtimeType 
          || newDelegate.shouldRebuild(oldDelegate)
          )
       )
      performRebuild();
  }

  // 我们在两个时刻扩展 widget 
  //   1.当我们自身需要重建的时候（也就是 performRebuild 方法被调用）
  //   2.当我们的渲染对象需要添加一个新的孩子的时候（详见 createChild）
  // 两种情况下，我们都会缓存缓存新建的 widget
  // 在情况 1 ，我们会重置缓存列表
  final Map<int, Widget> _childWidgets = HashMap<int, Widget>();
  /// 平衡树映射表
  final SplayTreeMap<int, Element> _childElements = SplayTreeMap<int, Element>();
  RenderBox _currentBeforeChild;

  /// 注意，这个方法中使用的 updateChild 方法是此类中重载的方法
  @override
  void performRebuild() {
    _childWidgets.clear(); // 重置缓存
    super.performRebuild();
    _currentBeforeChild = null;
    try {
      final SplayTreeMap<int, Element> newChildren = SplayTreeMap<int, Element>();

      /// 这个函数更新了给定 index 对应的 element：
      /// 也就是将 newChildren 和 childElement 表做对比，如果某个 index 对应的 element 发生了变化，就卸载
      /// 原来的 element，并且更新为 newChildren 中对应的 element
      void processElement(int index) {
        /// 如果当前 index 对应 element 不为 null 且发生了改变
        if (  _childElements[index] != null 
           && _childElements[index] != newChildren[index]) 
        {
          /// 当前 index 对应的 element 没有被使用，因此它应该被卸载
          // This index has an old child that isn't used anywhere and should be deactivated.
          _childElements[index] = updateChild(_childElements[index], null, index);
        }
        
        /// 其中：
        /// _build(index) 方法调用 delegate 中的 build 方法根据给定 index 来构建 widget（并缓存） 
        final Element newChild = updateChild(newChildren[index], _build(index), index);
        if (newChild != null) {
          _childElements[index] = newChild;
          final SliverMultiBoxAdaptorParentData parentData = 
              newChild.renderObject.parentData as SliverMultiBoxAdaptorParentData;
          if (!parentData.keptAlive)
            _currentBeforeChild = newChild.renderObject as RenderBox;
        } else {
          _childElements.remove(index);
        }
      }

      /// 根据每个 element 的 widget 对应的 key 来判断是否有 element 发生移动（改变 index），如果没有 key 
      /// 的话，则直接将原有 element 和其 index 放入到 newChildren 中。由此可以得到一个 newChildren 表，
      /// 记录了所有被复用的 element。
      for (final int index in _childElements.keys.toList()) {
        final Key key = _childElements[index].widget.key;
        final int newIndex = (key == null) ? null : widget.delegate.findIndexByKey(key);
        if (newIndex != null && newIndex != index) {
          newChildren[newIndex] = _childElements[index];
          // We need to make sure the original index gets processed.
          newChildren.putIfAbsent(index, () => null);
          // We do not want the remapped child to get deactivated during processElement.
          _childElements.remove(index);
        } else {
          newChildren.putIfAbsent(index, () => _childElements[index]);
        }
      }

      newChildren.keys.forEach(processElement);
        
      if (_didUnderflow) {
        final int lastKey = _childElements.lastKey() ?? -1;
        final int rightBoundary = lastKey + 1;
        newChildren[rightBoundary] = _childElements[rightBoundary];
        processElement(rightBoundary);
      }
    } finally {
      _currentlyUpdatingChildIndex = null;
      renderObject.debugChildIntegrityEnabled = true;
    }
  }

  Widget _build(int index) {
    /// 如果给定的下表没有对应的 widget，那么调用 delegate 的 build 方法构建一个，否则直接返回
    ///
    /// SliverChildDelegate.build 方法如下，构建给定下标对相应的 child
    /// Widget build(BuildContext context, int index);
    return _childWidgets.putIfAbsent(index, () => widget.delegate.build(this, index));
  }

  /// 实现 RenderSliverBoxChildManager.createChild 方法
  @override
  void createChild(int index, { @required RenderBox after }) {
    owner.buildScope(this, () {
      /// 是否插入到首位
      final bool insertFirst = after == null;
      /// 找到插入位置的前一个元素
      _currentBeforeChild = insertFirst 
          ? null 
          : (_childElements[index-1].renderObject as RenderBox);
      Element newChild;
      try {
        _currentlyUpdatingChildIndex = index;
        newChild = updateChild(_childElements[index], _build(index), index);
      } finally {
        _currentlyUpdatingChildIndex = null;
      }
      if (newChild != null) {
        _childElements[index] = newChild;
      } else {
        _childElements.remove(index);
      }
    });
  }

  /// 此重载方法只对滚动偏移进行了记录，作用同 super.updateChild
  @override
  Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
    final SliverMultiBoxAdaptorParentData oldParentData = 
        child?.renderObject?.parentData as SliverMultiBoxAdaptorParentData;
      
    final Element newChild = super.updateChild(child, newWidget, newSlot);
      
    final SliverMultiBoxAdaptorParentData newParentData = 
        newChild?.renderObject?.parentData as SliverMultiBoxAdaptorParentData;

    // 如果 renderObject 发生了替换，需要保存旧的滚动偏移
    if (  oldParentData != newParentData 
       && oldParentData != null 
       && newParentData != null
    ) {
      newParentData.layoutOffset = oldParentData.layoutOffset;
    }
    return newChild;
  }

  @override
  void forgetChild(Element child) {
    _childElements.remove(child.slot as int);
  }

  /// 实现 RenderSliverBoxChildManager.removeChild
  @override
  void removeChild(RenderBox child) {
    final int index = renderObject.indexOf(child);
    owner.buildScope(this, () {
      try {
        _currentlyUpdatingChildIndex = index;
        final Element result = updateChild(_childElements[index], null, index);
      } finally {
        _currentlyUpdatingChildIndex = null;
      }
      _childElements.remove(index);
    });
  }

  /// 推断最大滚动偏移
  static double _extrapolateMaxScrollOffset(
    int firstIndex,
    int lastIndex,
    double leadingScrollOffset,
    double trailingScrollOffset,
    int childCount,
  ) {
    /// 如果最后下标是孩子个数减一，意味着最后一个元素的偏移量就是最大滚动长度，直接返回
    if (lastIndex == childCount - 1)
      return trailingScrollOffset;
    /// 已经具体化的孩子的个数，即最大下标减去最小下标加一
    final int reifiedCount = lastIndex - firstIndex + 1;
    /// 每个孩子平均占用的滚动偏移
    final double averageExtent = (trailingScrollOffset - leadingScrollOffset) / reifiedCount;
    final int remainingCount = childCount - lastIndex - 1;
    return trailingScrollOffset + averageExtent * remainingCount;
  }

  @override
  double estimateMaxScrollOffset(
    SliverConstraints constraints, {
    int firstIndex,
    int lastIndex,
    double leadingScrollOffset,
    double trailingScrollOffset,
  }) {
    final int childCount = this.childCount;
    /// 如果 childCount 为 null，则默认为无穷列表
    if (childCount == null)
      return double.infinity;
    return widget.estimateMaxScrollOffset(
      constraints,
      firstIndex,
      lastIndex,
      leadingScrollOffset,
      trailingScrollOffset,
    ) ?? _extrapolateMaxScrollOffset(
      firstIndex,
      lastIndex,
      leadingScrollOffset,
      trailingScrollOffset,
      childCount,
    );
  }

  @override
  int get childCount => widget.delegate.estimatedChildCount;

  @override
  void didStartLayout() {}

  @override
  void didFinishLayout() {
    final int firstIndex = _childElements.firstKey() ?? 0;
    final int lastIndex = _childElements.lastKey() ?? 0;
    widget.delegate.didFinishLayout(firstIndex, lastIndex);
  }

  int _currentlyUpdatingChildIndex;

  @override
  void didAdoptChild(RenderBox child) {
    final SliverMultiBoxAdaptorParentData childParentData =
        child.parentData as SliverMultiBoxAdaptorParentData;
    childParentData.index = _currentlyUpdatingChildIndex;
  }

  bool _didUnderflow = false;

  @override
  void setDidUnderflow(bool value) {
    _didUnderflow = value;
  }

  /// 以下重载 RenderBox 中的方法 
  @override
  void insertChildRenderObject(covariant RenderObject child, int slot) {
    renderObject.insert(child as RenderBox, after: _currentBeforeChild);
  }

  @override
  void moveChildRenderObject(covariant RenderObject child, int slot) {
    renderObject.move(child as RenderBox, after: _currentBeforeChild);
  }

  @override
  void removeChildRenderObject(covariant RenderObject child) {
    renderObject.remove(child as RenderBox);
  }

  @override
  void visitChildren(ElementVisitor visitor) {
    _childElements.values.toList().forEach(visitor);
  }
}

```



##  RenderSliverMultiBoxAdaptor 

```dart
/// 一个拥有许多盒子孩子的滑块（例如 SliverList，SliverGrid）
/// 
/// [RenderSliverMultiBoxAdaptor] 是拥有多个盒子孩子的滑块的基类。所有的孩子被 
/// [RenderSliverBoxChildManager] 所管理，则使得子类能够在布局的时候延迟地构建孩子。通常子类将只构建需要填满 
/// [SliverConstraints.remainingPaintExtent] 的孩子
abstract class RenderSliverMultiBoxAdaptor 
    extends RenderSliver
    /// 提供以双向链表的方式管理子节点
    with ContainerRenderObjectMixin<
    		RenderBox, 
			SliverMultiBoxAdaptorParentData>,
         RenderSliverHelpers, 
		RenderSliverWithKeepAliveMixin 
{

  /// Creates a sliver with multiple box children.
  ///
  /// The [childManager] argument must not be null.
  RenderSliverMultiBoxAdaptor({
    @required RenderSliverBoxChildManager childManager,
  }) : _childManager = childManager;

  @override
  void setupParentData(RenderObject child) {
    if (child.parentData is! SliverMultiBoxAdaptorParentData)
      child.parentData = SliverMultiBoxAdaptorParentData();
  }

  @protected
  RenderSliverBoxChildManager get childManager => _childManager;
  final RenderSliverBoxChildManager _childManager;

  /// 在不可见的情况下仍需要保持活动的子节点
  final Map<int, RenderBox> _keepAliveBucket = <int, RenderBox>{};

  @override
  void adoptChild(RenderObject child) {
    super.adoptChild(child);
    final SliverMultiBoxAdaptorParentData childParentData = 
        child.parentData as SliverMultiBoxAdaptorParentData;
    if (!childParentData._keptAlive)
      childManager.didAdoptChild(child as RenderBox);
  }

  @override
  void insert(RenderBox child, { RenderBox after }) {
    super.insert(child, after: after);
  }

  @override
  void move(RenderBox child, { RenderBox after }) {
    // There are two scenarios:
    //
    // 1. The child is not keptAlive.
    // The child is in the childList maintained by ContainerRenderObjectMixin.
    // We can call super.move and update parentData with the new slot.
    //
    // 2. The child is keptAlive.
    // In this case, the child is no longer in the childList but might be stored in
    // [_keepAliveBucket]. We need to update the location of the child in the bucket.
    final SliverMultiBoxAdaptorParentData childParentData = 
        child.parentData as SliverMultiBoxAdaptorParentData;
    if (!childParentData.keptAlive) {
      super.move(child, after: after);
      childManager.didAdoptChild(child); // updates the slot in the parentData
      // Its slot may change even if super.move does not change the position.
      // In this case, we still want to mark as needs layout.
      markNeedsLayout();
    } else {
      // If the child in the bucket is not current child, that means someone has
      // already moved and replaced current child, and we cannot remove this child.
      if (_keepAliveBucket[childParentData.index] == child) {
        _keepAliveBucket.remove(childParentData.index);
      }
      // Update the slot and reinsert back to _keepAliveBucket in the new slot.
      childManager.didAdoptChild(child);
      // If there is an existing child in the new slot, that mean that child will
      // be moved to other index. In other cases, the existing child should have been
      // removed by updateChild. Thus, it is ok to overwrite it.
      _keepAliveBucket[childParentData.index] = child;
    }
  }

  @override
  void remove(RenderBox child) {
    final SliverMultiBoxAdaptorParentData childParentData = 
        child.parentData as SliverMultiBoxAdaptorParentData;
    if (!childParentData._keptAlive) {
      super.remove(child);
      return;
    }
    _keepAliveBucket.remove(childParentData.index);
    dropChild(child);
  }

  @override
  void removeAll() {
    super.removeAll();
    _keepAliveBucket.values.forEach(dropChild);
    _keepAliveBucket.clear();
  }

  void _createOrObtainChild(int index, { RenderBox after }) {
    void callback(SliverConstraints constraints) {
      if (_keepAliveBucket.containsKey(index)) {
        final RenderBox child = _keepAliveBucket.remove(index);
        final SliverMultiBoxAdaptorParentData childParentData = 
            child.parentData as SliverMultiBoxAdaptorParentData;
        dropChild(child);
        child.parentData = childParentData;
        insert(child, after: after);
        childParentData._keptAlive = false;
      } else {
        _childManager.createChild(index, after: after);
      }
    }
    /// invokeLayoutCallback 激活了给定的回调并做出了断言
    invokeLayoutCallback<SliverConstraints>(callback);
  }

  void _destroyOrCacheChild(RenderBox child) {
    final SliverMultiBoxAdaptorParentData childParentData = 
        child.parentData as SliverMultiBoxAdaptorParentData;
    if (childParentData.keepAlive) {
      remove(child);
      _keepAliveBucket[childParentData.index] = child;
      child.parentData = childParentData;
      super.adoptChild(child);
      childParentData._keptAlive = true;
    } else {
      _childManager.removeChild(child);
    }
  }

  @override
  void attach(PipelineOwner owner) {
    super.attach(owner);
    for (final RenderBox child in _keepAliveBucket.values)
      child.attach(owner);
  }

  @override
  void detach() {
    super.detach();
    for (final RenderBox child in _keepAliveBucket.values)
      child.detach();
  }

  @override
  void redepthChildren() {
    super.redepthChildren();
    _keepAliveBucket.values.forEach(redepthChild);
  }

  @override
  void visitChildren(RenderObjectVisitor visitor) {
    super.visitChildren(visitor);
    _keepAliveBucket.values.forEach(visitor);
  }

  @override
  void visitChildrenForSemantics(RenderObjectVisitor visitor) {
    super.visitChildren(visitor);
    // Do not visit children in [_keepAliveBucket].
  }

  /// Called during layout to create and add the child with the given index and
  /// scroll offset.
  ///
  /// Calls [RenderSliverBoxChildManager.createChild] to actually create and add
  /// the child if necessary. The child may instead be obtained from a cache;
  /// see [SliverMultiBoxAdaptorParentData.keepAlive].
  ///
  /// Returns false if there was no cached child and `createChild` did not add
  /// any child, otherwise returns true.
  ///
  /// Does not layout the new child.
  ///
  /// When this is called, there are no visible children, so no children can be
  /// removed during the call to `createChild`. No child should be added during
  /// that call either, except for the one that is created and returned by
  /// `createChild`.
  @protected
  bool addInitialChild({ int index = 0, double layoutOffset = 0.0 }) {
    _createOrObtainChild(index, after: null);
    if (firstChild != null) {
      final SliverMultiBoxAdaptorParentData firstChildParentData = 
          firstChild.parentData as SliverMultiBoxAdaptorParentData;
      firstChildParentData.layoutOffset = layoutOffset;
      return true;
    }
    childManager.setDidUnderflow(true);
    return false;
  }

  /// 在布局期间被调用，用于创建、添加、布局在 [firstChild] 之前的一个 child
  ///
  /// 在必要的时候调用 [RenderSliverBoxChildManager.createChild] 来实际上创建并添加孩子。
  ///
  /// 返回新建的 child。如果没有完成新建，则返回 null
  /// Returns the new child or null if no child was obtained.
  ///
  /// 
  /// The child that was previously the first child, as well as any subsequent
  /// children, may be removed by this call if they have not yet been laid out
  /// during this layout pass. No child should be added during that call except
  /// for the one that is created and returned by `createChild`.
  @protected
  RenderBox insertAndLayoutLeadingChild(
    BoxConstraints childConstraints, {
    bool parentUsesSize = false,
  }) {
    /// 获取一个下标小于 firstChild 的下标
    final int index = indexOf(firstChild) - 1;
    /// 尝试获取到这个下标对应的 element，并且将它添加到子节点列表的首部（注意：after: null）
    _createOrObtainChild(index, after: null);
    /// 如果成功地创建了一个，对他进行布局，并返回
    if (indexOf(firstChild) == index) {
      firstChild.layout(childConstraints, parentUsesSize: parentUsesSize);
      return firstChild;
    }
    
    childManager.setDidUnderflow(true);
    return null;
  }

  /// 在布局期间被调用，用于创建，添加，并且布局在给定 child 之后的一些 child
  ///
  /// 在必要的时候调用 [RenderSliverBoxChildManager.createChild] 来实际上创建并添加孩子。
  ///
  /// 返回新建的 child。调用者需要为此 child 配置 scrollOffset
  @protected
  RenderBox insertAndLayoutChild(
    BoxConstraints childConstraints, {
    @required RenderBox after,
    bool parentUsesSize = false,
  }) {
    assert(_debugAssertChildListLocked());
    assert(after != null);
    final int index = indexOf(after) + 1;
    _createOrObtainChild(index, after: after);
    final RenderBox child = childAfter(after);
    if (child != null && indexOf(child) == index) {
      child.layout(childConstraints, parentUsesSize: parentUsesSize);
      return child;
    }
    childManager.setDidUnderflow(true);
    return null;
  }

  /// 在布局完成之后调用，回收在 child 列表头部和尾部的元素
  ///
  /// 如果某个孩子的 [SliverMultiBoxAdaptorParentData.keepAlive] 设置为 true，那么它将会被缓存而不是移除     /// 
  /// 这个方法也收集任何之前需要 keepAlive 而现在不需要的元素。因此，这个方法应当在每次 [performLayout] 运行
  /// 之后调用
  @protected
  void collectGarbage(int leadingGarbage, int trailingGarbage) {
    void callback(SliverConstraints constraints) {
      while (leadingGarbage > 0) {
        _destroyOrCacheChild(firstChild);
        leadingGarbage -= 1;
      }
      while (trailingGarbage > 0) {
        _destroyOrCacheChild(lastChild);
        trailingGarbage -= 1;
      }
      // 请求 manager 来移除所有不再需要被 keepAlive 的孩子（这个方法会造成 _keepAliveBucket 发生改变，因此
      // 我们必须提前准备列表）
      _keepAliveBucket.values.where((RenderBox child) {
        final SliverMultiBoxAdaptorParentData childParentData = 
            child.parentData as SliverMultiBoxAdaptorParentData;
        return !childParentData.keepAlive;
      }).toList().forEach(_childManager.removeChild);
    };
    
    invokeLayoutCallback<SliverConstraints>(callback);
  }

  /// 返回给定 child 的下标
  int indexOf(RenderBox child) {
    final SliverMultiBoxAdaptorParentData childParentData = 
        child.parentData as SliverMultiBoxAdaptorParentData;
    return childParentData.index;
  }

  /// 返回给定孩子在主轴上的维度，只在布局期间有效
  /// 由于孩子是盒子，因此只需返回对应的宽或高
  @protected
  double paintExtentOf(RenderBox child) {
    switch (constraints.axis) {
      case Axis.horizontal:
        return child.size.width;
      case Axis.vertical:
        return child.size.height;
    }
    return null;
  }

  @override
  bool hitTestChildren(SliverHitTestResult result, { 
      @required double mainAxisPosition, 
      @required double crossAxisPosition 
  }) {
    RenderBox child = lastChild;
    final BoxHitTestResult boxResult = BoxHitTestResult.wrap(result);
    while (child != null) {
      if (hitTestBoxChild(
          boxResult, 
          child, 
          mainAxisPosition: mainAxisPosition, 
          crossAxisPosition: crossAxisPosition
      ))
        return true;
      child = childBefore(child);
    }
    return false;
  }

  /// 返回给定的孩子在滚动主轴上的位置
  @override
  double childMainAxisPosition(RenderBox child) {
    return childScrollOffset(child) - constraints.scrollOffset;
  }

  /// 返回给定孩子的滚动偏移
  @override
  double childScrollOffset(RenderObject child) {
    final SliverMultiBoxAdaptorParentData childParentData = 
        child.parentData as SliverMultiBoxAdaptorParentData;
    /// 这个值通常是在 performLayout 方法中被设置，由于此类为抽象类，子类（例如 SliverList）可以实现不同的算
    /// 法来实现不同的布局
    return childParentData.layoutOffset;
  }

  @override
  void applyPaintTransform(RenderObject child, Matrix4 transform) {
    applyPaintTransformForBoxChild(child as RenderBox, transform);
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    /// 如果没有孩子，直接返回
    if (firstChild == null)
      return;
    // offset 是指到左上角的偏移，无论轴方向
    // originOffset 给出了从真正原点到轴方向原点的偏移
    Offset mainAxisUnit, crossAxisUnit, originOffset;
    bool addExtent;
      
    switch (applyGrowthDirectionToAxisDirection(
        constraints.axisDirection, 
        constraints.growthDirection)
    ) {
      case AxisDirection.up:
        mainAxisUnit = const Offset(0.0, -1.0);
        crossAxisUnit = const Offset(1.0, 0.0);
        originOffset = offset + Offset(0.0, geometry.paintExtent);
        addExtent = true;
        break;
      case AxisDirection.right:
        mainAxisUnit = const Offset(1.0, 0.0);
        crossAxisUnit = const Offset(0.0, 1.0);
        originOffset = offset;
        addExtent = false;
        break;
      case AxisDirection.down:
        mainAxisUnit = const Offset(0.0, 1.0);
        crossAxisUnit = const Offset(1.0, 0.0);
        originOffset = offset;
        addExtent = false;
        break;
      case AxisDirection.left:
        mainAxisUnit = const Offset(-1.0, 0.0);
        crossAxisUnit = const Offset(0.0, 1.0);
        originOffset = offset + Offset(geometry.paintExtent, 0.0);
        addExtent = true;
        break;
    }
    RenderBox child = firstChild;
    while (child != null) {
      final double mainAxisDelta = childMainAxisPosition(child);
      final double crossAxisDelta = childCrossAxisPosition(child);
      Offset childOffset = Offset(
        originOffset.dx + mainAxisUnit.dx * mainAxisDelta + crossAxisUnit.dx * crossAxisDelta,
        originOffset.dy + mainAxisUnit.dy * mainAxisDelta + crossAxisUnit.dy * crossAxisDelta,
      );
      if (addExtent)
        childOffset += mainAxisUnit * paintExtentOf(child);

      // 如果子节点的可见区间 (mainAxisDelta, mainAxisDelta + paintExtentOf(child)) 不与绘制区域区间
      // (0,onstraints.remainingPaintExtent) 相交，则不可见。
      if (  mainAxisDelta < constraints.remainingPaintExtent 
         && mainAxisDelta + paintExtentOf(child) > 0
         )
        context.paintChild(child, childOffset);

      child = childAfter(child);
    }
  }

}

```


# Flutter的一些组件源码Ⅲ

[TOC]

## CustomSingleChildLayout

```dart
/// 简单封装了SingleChildRenderObjectWidgt
/// 重载了createRenderObject和 updateRenderObject
class CustomSingleChildLayout extends SingleChildRenderObjectWidget {
  const CustomSingleChildLayout({
    Key key,
    @required this.delegate,
    Widget child,
  }) : assert(delegate != null),
       super(key: key, child: child);

  final SingleChildLayoutDelegate delegate;

  @override
  RenderCustomSingleChildLayoutBox createRenderObject(BuildContext context) {
    return RenderCustomSingleChildLayoutBox(delegate: delegate);
  }

  @override
  void updateRenderObject(BuildContext context, RenderCustomSingleChildLayoutBox renderObject) {
    renderObject.delegate = delegate;
  }
}
```



## RenderCustomSingleChildLayoutBox

```dart
class RenderCustomSingleChildLayoutBox extends RenderShiftedBox {
  RenderCustomSingleChildLayoutBox({
    RenderBox child,
    @required SingleChildLayoutDelegate delegate,
  }) : assert(delegate != null),
       _delegate = delegate,
       super(child);

  SingleChildLayoutDelegate get delegate => _delegate;
  SingleChildLayoutDelegate _delegate;
  set delegate(SingleChildLayoutDelegate newDelegate) {
    assert(newDelegate != null);
    if (_delegate == newDelegate)
      return;
    final SingleChildLayoutDelegate oldDelegate = _delegate;
    /// 如果delegate决定无需重新布局或者新旧delegate的类型不一样，触发重新布局  
    if (newDelegate.runtimeType != oldDelegate.runtimeType ||
        newDelegate.shouldRelayout(oldDelegate))
      markNeedsLayout();
    _delegate = newDelegate;
    if (attached) {
      oldDelegate?._relayout?.removeListener(markNeedsLayout);
      newDelegate?._relayout?.addListener(markNeedsLayout);
    }
  }

  @override
  void attach(PipelineOwner owner) {
    super.attach(owner);
    _delegate?._relayout?.addListener(markNeedsLayout);
  }

  @override
  void detach() {
    _delegate?._relayout?.removeListener(markNeedsLayout);
    super.detach();
  }

  Size _getSize(BoxConstraints constraints) {
    return constraints.constrain(_delegate.getSize(constraints));
  }

  @override
  double computeMinIntrinsicWidth(double height) {
    final double width = _getSize(BoxConstraints.tightForFinite(height: height)).width;
    if (width.isFinite)
      return width;
    return 0.0;
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    final double width = _getSize(BoxConstraints.tightForFinite(height: height)).width;
    if (width.isFinite)
      return width;
    return 0.0;
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    final double height = _getSize(BoxConstraints.tightForFinite(width: width)).height;
    if (height.isFinite)
      return height;
    return 0.0;
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    final double height = _getSize(BoxConstraints.tightForFinite(width: width)).height;
    if (height.isFinite)
      return height;
    return 0.0;
  }

  @override
  void performLayout() {
    size = _getSize(constraints);
    if (child != null) {
      final BoxConstraints childConstraints = delegate.getConstraintsForChild(constraints);
      assert(childConstraints.debugAssertIsValid(isAppliedConstraint: true));
      child.layout(childConstraints, parentUsesSize: !childConstraints.isTight);
      final BoxParentData childParentData = child.parentData;
      childParentData.offset = delegate.getPositionForChild(
         size, 
         childConstraints.isTight ? childConstraints.smallest : child.size
      );
    }
  }
}
```



## SingleChildLayoutDelegate

```dart
abstract class SingleChildLayoutDelegate {
  
  const SingleChildLayoutDelegate({ Listenable relayout }) : _relayout = relayout;

  /// 可监听对象。当_relayout每次发生改变时，就会触发重新布局  
  final Listenable _relayout;

  /// 采用最大约束作为大小
  Size getSize(BoxConstraints constraints) => constraints.biggest;

  /// 默认返回约束
  BoxConstraints getConstraintsForChild(BoxConstraints constraints) => constraints;

  /// child的绘制位置，默认为（0，0）
  Offset getPositionForChild(Size size, Size childSize) => Offset.zero;

  /// 是否应该重新布局
  bool shouldRelayout(covariant SingleChildLayoutDelegate oldDelegate);
}
```



## RenderShiftedBox

```dart
abstract class RenderShiftedBox extends RenderBox with RenderObjectWithChildMixin<RenderBox> {
  /// Initializes the [child] property for subclasses.
  RenderShiftedBox(RenderBox child) {
    this.child = child;
  }

  @override
  double computeMinIntrinsicWidth(double height) {
    if (child != null)
      return child.getMinIntrinsicWidth(height);
    return 0.0;
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    if (child != null)
      return child.getMaxIntrinsicWidth(height);
    return 0.0;
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    if (child != null)
      return child.getMinIntrinsicHeight(width);
    return 0.0;
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    if (child != null)
      return child.getMaxIntrinsicHeight(width);
    return 0.0;
  }

  @override
  double computeDistanceToActualBaseline(TextBaseline baseline) {
    double result;
    if (child != null) {
      assert(!debugNeedsLayout);
      result = child.getDistanceToActualBaseline(baseline);
      final BoxParentData childParentData = child.parentData;
      if (result != null)
        result += childParentData.offset.dy;
    } else {
      result = super.computeDistanceToActualBaseline(baseline);
    }
    return result;
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    if (child != null) {
      final BoxParentData childParentData = child.parentData;
      context.paintChild(child, childParentData.offset + offset);
    }
  }

  @override
  bool hitTestChildren(BoxHitTestResult result, { Offset position }) {
    if (child != null) {
      final BoxParentData childParentData = child.parentData;
      return result.addWithPaintOffset(
        offset: childParentData.offset,
        position: position,
        hitTest: (BoxHitTestResult result, Offset transformed) {
          assert(transformed == position - childParentData.offset);
          return child.hitTest(result, position: transformed);
        },
      );
    }
    return false;
  }

}
```



### --------------------------(分割)---------------------------



## CustomMultiChildLayout

```dart
/// CustomMultiChildLayout 并没有太多要点，只是简单封装了 MultiChildRenderObjectWidget 
/// 并重载了 createRenderObject 和 updateRenderObject 方法

Class CustomMultiChildLayout extends MultiChildRenderObjectWidget {
  CustomMultiChildLayout({
    Key key,
    @required this.delegate,
    List<Widget> children = const <Widget>[],
  }) : assert(delegate != null),
       super(key: key, children: children);
    
  final MultiChildLayoutDelegate delegate;

  @override
  RenderCustomMultiChildLayoutBox createRenderObject(BuildContext context) {
    return RenderCustomMultiChildLayoutBox(delegate: delegate);
  }

  /// 将新的 MultiChildLayoutDelegate 实例用于更新 renderObject 
  @override
  void updateRenderObject(
      BuildContext context, 
      RenderCustomMultiChildLayoutBox renderObject) 
  {
    renderObject.delegate = delegate;
  }
}
```



## RenderCustomMultiChildLayoutBox

```dart
/// RenderCustomMultiChildLayoutBox 是 CustomMultiChildLayout 的 renderObject 
/// delegate决定每个子元素的约束和位置，并决定了parent的大小（这个大小不能依赖于子元素）
class RenderCustomMultiChildLayoutBox extends RenderBox
  with ContainerRenderObjectMixin<RenderBox, MultiChildLayoutParentData>,
       RenderBoxContainerDefaultsMixin<RenderBox, MultiChildLayoutParentData> {
  RenderCustomMultiChildLayoutBox({
    List<RenderBox> children,
    @required MultiChildLayoutDelegate delegate,
  }) : assert(delegate != null),
       _delegate = delegate {
    addAll(children);
  }

  /// 为子元素设置parentData         
  @override
  void setupParentData(RenderBox child) {
    if (child.parentData is! MultiChildLayoutParentData)
      child.parentData = MultiChildLayoutParentData();
  }

  /// 传入的delegate，每次修改的时候都会调用 markNeedsLayout();
  MultiChildLayoutDelegate get delegate => _delegate;
  MultiChildLayoutDelegate _delegate;
  set delegate(MultiChildLayoutDelegate value) {
    assert(value != null);
    if (_delegate == value)
      return;
    if (value.runtimeType != _delegate.runtimeType || value.shouldRelayout(_delegate))
      markNeedsLayout();
    _delegate = value;
  }

  Size _getSize(BoxConstraints constraints) {
    ...
    return constraints.constrain(_delegate.getSize(constraints));
  }

  // TODO(ianh): It's a bit dubious to be using the getSize function from the delegate to
  // figure out the intrinsic dimensions. We really should either not support intrinsics,
  // or we should expose intrinsic delegate callbacks and throw if they're not implemented.

  /// 计算固有宽高?         
  @override
  double computeMinIntrinsicWidth(double height) {
    final double width = _getSize(BoxConstraints.tightForFinite(height: height)).width;
    if (width.isFinite)
      return width;
    return 0.0;
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    final double width = _getSize(BoxConstraints.tightForFinite(height: height)).width;
    if (width.isFinite)
      return width;
    return 0.0;
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    final double height = _getSize(BoxConstraints.tightForFinite(width: width)).height;
    if (height.isFinite)
      return height;
    return 0.0;
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    final double height = _getSize(BoxConstraints.tightForFinite(width: width)).height;
    if (height.isFinite)
      return height;
    return 0.0;
  }

  @override
  void performLayout() {
    size = _getSize(constraints);
    delegate._callPerformLayout(size, firstChild);
  }

  @override
  void paint(PaintingContext context, Offset offset) {
   /// void defaultPaint(PaintingContext context, Offset offset) {
   ///   ChildType child = firstChild;
   ///   while (child != null) {
   ///     final ParentDataType childParentData = child.parentData;
   /// 	   根据在layout阶段设置的childParentData来决定位置位置   
   ///     context.paintChild(child, childParentData.offset + offset);
   ///     child = childParentData.nextSibling;
   ///   }
   /// }
    defaultPaint(context, offset);
  }

  @override
  bool hitTestChildren(BoxHitTestResult result, { Offset position }) {
    return defaultHitTestChildren(result, position: position);
  }
}
```



## MultiChildLayoutDelegate

```dart
import 'package:flutter/foundation.dart';

import 'box.dart';
import 'object.dart';

class MultiChildLayoutParentData extends ContainerBoxParentData<RenderBox> {
  /// 用于识别children
  Object id;

  @override
  String toString() => '${super.toString()}; id=$id';
}

/// 这个delegate控制多个子元素的布局
///
/// Delegate 必须是幂等的。详细地说，如果两个delegate相等，那么它们产生的布局是一样的
/// 替换delegate来重新布局
///
/// 重载 [getSize] 来控制布局的全部大小。这个大小不能依赖children的属性。
///
/// 重载 [performLayout] 来控制children的大小和位置。实现这个方法必须调用且只能调用
/// [layoutChild]一次（但是顺序可以随意）。通常，delegate将会使用[layoutChild]返回的大小
/// 来控制另一些child的布局和偏移
///
/// 重载 [shouldRelayout] 来决定当delegate改变时是否需要重新布局。
///
/// 一个典型[performLayout]实现（控制follwer元素跟随leader的大小）
///
/// enum _Slot {
///   leader,
///   follower,
/// }
///
/// class FollowTheLeader extends MultiChildLayoutDelegate {
///   @override
///   void performLayout(Size size) {
///     Size leaderSize = Size.zero;
///
///     if (hasChild(_Slot.leader)) {
///       leaderSize = layoutChild(_Slot.leader, BoxConstraints.loose(size));
///       positionChild(_Slot.leader, Offset.zero);
///     }
///
///     if (hasChild(_Slot.follower)) {
///       layoutChild(_Slot.follower, BoxConstraints.tight(leaderSize));
///       positionChild(_Slot.follower, Offset(size.width - leaderSize.width,
///           size.height - leaderSize.height));
///     }
///   }
///
///   @override
///   bool shouldRelayout(MultiChildLayoutDelegate oldDelegate) => false;
/// }

abstract class MultiChildLayoutDelegate {
  Map<Object, RenderBox> _idToChild;
  Set<RenderBox> _debugChildrenNeedingLayout;

  bool hasChild(Object childId) => _idToChild[childId] != null;

  /// 布局孩子，直接调用 child.layout 方法，并返回size
  Size layoutChild(Object childId, BoxConstraints constraints) {
    final RenderBox child = _idToChild[childId];
    ...  
    child.layout(constraints, parentUsesSize: true);
    return child.size;
  }

  /// 指定child的偏移
  ///
  /// 调用[performLayout] function 来放置每一个child
  /// 如果不调用，那么每个孩子将会放到左上角（0，0）
    
  void positionChild(Object childId, Offset offset) {
    final RenderBox child = _idToChild[childId];
    final MultiChildLayoutParentData childParentData = child.parentData;
    /// 这里通过设置 childParentData 来决定child的偏移
    childParentData.offset = offset;
  }

  void _callPerformLayout(Size size, RenderBox firstChild) {
    /// 因为递归可能使得_idToChild被修改，所以先储存一次
    final Map<Object, RenderBox> previousIdToChild = _idToChild;

    Set<RenderBox> debugPreviousChildrenNeedingLayout;
	...
        
    try {
      /// 清空 _idToChild 
      _idToChild = <Object, RenderBox>{};
      /// 获取第一个child（因为child通过链表实现，故可以直接访问一下child）  
      RenderBox child = firstChild;
      while (child != null) {
        final MultiChildLayoutParentData childParentData = child.parentData;
        /// 将child对应其id加入 _idToChild 中
        _idToChild[childParentData.id] = child;
        child = childParentData.nextSibling;
      }
      /// 调用子类重载的 performLayout（也就是我们写的布局啦）
      performLayout(size);
    } finally {
      /// 递归之后就改回来  
      _idToChild = previousIdToChild;
    }
  }
    
  Size getSize(BoxConstraints constraints) => constraints.biggest;

  void performLayout(Size size);

  bool shouldRelayout(covariant MultiChildLayoutDelegate oldDelegate);

  @override
  String toString() => '$runtimeType';
}
```

## MultiChildRenderObjectWidget

```dart
/// 抽象类MultiChildRenderObjectWidget
/// 并没有实现creatRenderObject方法
abstract class MultiChildRenderObjectWidget extends RenderObjectWidget {
  MultiChildRenderObjectWidget({ Key key, this.children = const <Widget>[] })
   : super(key: key);
    
  final List<Widget> children;

  @override
  MultiChildRenderObjectElement createElement() => MultiChildRenderObjectElement(this);
}
```



## MultiChildRenderObjectElement 

```dart
class MultiChildRenderObjectElement extends RenderObjectElement {
  MultiChildRenderObjectElement(MultiChildRenderObjectWidget widget)
    : super(widget);

  @override
  MultiChildRenderObjectWidget get widget => super.widget;

   /// 这个element的当前children
   /** Iterable.where方法描述如下
   * Returns a new lazy [Iterable] with all elements that satisfy the
   * predicate [test].
   */
    
  /// 返回 _children中不在_forgottenChildren中的元素  
  @protected
  @visibleForTesting
  Iterable<Element> get children => _children.where((Element child) => !_forgottenChildren.contains(child));

  List<Element> _children;
  // 用于减小时间复杂度
  final Set<Element> _forgottenChildren = HashSet<Element>();

  @override
  void insertChildRenderObject(RenderObject child, Element slot) {
    final ContainerRenderObjectMixin
        <RenderObject, ContainerParentDataMixin<RenderObject>> 
        renderObject = this.renderObject;
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
    /// 对于child中的每一个Widget，调用inflateWidget（Widget newWidget, dynamic newSlot）
    /// 把生成的 newChild 作为slots，调用 inflateWidget（）
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



### ------------- 最后放几个mixin(分割) ---------------



## ContainerParentDataMixin

```dart
/// 这里'mixin' + 'on' 表示该mixin只能被 ParentData 及其子类使用，并且
/// 该mixin能够访问ParentData中的方法
mixin ContainerParentDataMixin<ChildType extends RenderObject> on ParentData {
  /// 上一个child
  ChildType previousSibling;
  /// 下一个child
  ChildType nextSibling;

  /// 移除自己
  @override
  void detach() {
    super.detach();
    if (previousSibling != null) {
      final ContainerParentDataMixin<ChildType> previousSiblingParentData 
           = previousSibling.parentData;
      ...  
      /// 链表的结合方式，让上一个节点的后继变成自己的后继，从而删除自己
      previousSiblingParentData.nextSibling = nextSibling;
    } 
    if (nextSibling != null) {
      final ContainerParentDataMixin<ChildType> nextSiblingParentData = nextSibling.parentData;
      ...  
      /// 链表的结合方式，让下一个节点的前驱变成自己的前驱，从而删除自己
      nextSiblingParentData.previousSibling = previousSibling;
    }
    previousSibling = null;
    nextSibling = null;
  }
}
```



## ContainerRenderObjectMixin

```dart
/// 这个mixin只能被RenderObject和其子类使用
mixin ContainerRenderObjectMixin
	<ChildType extends RenderObject, 
	ParentDataType extends ContainerParentDataMixin<ChildType>> on RenderObject 
{
  int _childCount = 0;
  /// The number of children.
  int get childCount => _childCount;
    
  ChildType _firstChild;
  ChildType _lastChild;
    
  void _insertIntoChildList(ChildType child, { ChildType after }) {
    final ParentDataType childParentData = child.parentData;
    ...
    _childCount += 1;
    if (after == null) {
      /// 如果after为空，则插入到_firstChild之前
      childParentData.nextSibling = _firstChild;
      if (_firstChild != null) {
        final ParentDataType _firstChildParentData = _firstChild.parentData;
        _firstChildParentData.previousSibling = child;
      }
      _firstChild = child;
      _lastChild ??= child;
    } else {
      
      final ParentDataType afterParentData = after.parentData;
      if (afterParentData.nextSibling == null) {
        /// 如果插入到列表最后
        assert(after == _lastChild);
        childParentData.previousSibling = after;
        afterParentData.nextSibling = child;
        _lastChild = child;
      } else {
        /// 若是插入到其他位置
        /// 需要更改自己的ParentData、前一个孩子的ParentData 和后一个孩子的ParentData 的指向
        childParentData.nextSibling = afterParentData.nextSibling;
        childParentData.previousSibling = after;
        // set up links from siblings to child
        final ParentDataType childPreviousSiblingParentData = 
            childParentData.previousSibling.parentData;
        final ParentDataType childNextSiblingParentData = 
            childParentData.nextSibling.parentData;
        childPreviousSiblingParentData.nextSibling = child;
        childNextSiblingParentData.previousSibling = child;
      }
    }
  }
  
    
  /// 将一个孩子插入到child列表中  
  void insert(ChildType child, { ChildType after }) {
    ...
    /// 这里调用到了RenderObject.adoptChild,建立了child的parentData
    /// @override
    /// void adoptChild(RenderObject child) {
    /// 	assert(_debugCanPerformMutations);
    /// 	assert(child != null);
    /// 	setupParentData(child);
    /// 	markNeedsLayout();
    /// 	markNeedsCompositingBitsUpdate();
    /// 	markNeedsSemanticsUpdate();
    /// 	super.adoptChild(child);
    /// }
    adoptChild(child);
    _insertIntoChildList(child, after: after);
  }

  /// 这里默认将新的child加在列表的最后
  void add(ChildType child) {
    insert(child, after: _lastChild);
  }

  void addAll(List<ChildType> children) {
    children?.forEach(add);
  }

  /// 这个方法与_insertIntoChildList对应，移除一个child  
  void _removeFromChildList(ChildType child) {
    final ParentDataType childParentData = child.parentData;
    ...
    if (childParentData.previousSibling == null) {
      _firstChild = childParentData.nextSibling;
    } else {
      final ParentDataType childPreviousSiblingParentData = childParentData.previousSibling.parentData;
      childPreviousSiblingParentData.nextSibling = childParentData.nextSibling;
    }
    if (childParentData.nextSibling == null) {
      assert(_lastChild == child);
      _lastChild = childParentData.previousSibling;
    } else {
      final ParentDataType childNextSiblingParentData = childParentData.nextSibling.parentData;
      childNextSiblingParentData.previousSibling = childParentData.previousSibling;
    }
    childParentData.previousSibling = null;
    childParentData.nextSibling = null;
    _childCount -= 1;
  }

  /// 这里调用到RenderObject.dropChild
  /// @override
  /// void dropChild(RenderObject child) {
  ///  	child._cleanRelayoutBoundary();
  ///  	child.parentData.detach();
  ///  	child.parentData = null;
  ///  	super.dropChild(child);
  ///  	markNeedsLayout();
  ///  	markNeedsCompositingBitsUpdate();
  ///  	markNeedsSemanticsUpdate();
  /// }
  void remove(ChildType child) {
    _removeFromChildList(child);
    dropChild(child);
  }

  /// 移除所有子child
  void removeAll() {
    ChildType child = _firstChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      final ChildType next = childParentData.nextSibling;
      childParentData.previousSibling = null;
      childParentData.nextSibling = null;
      dropChild(child);
      child = next;
    }
    _firstChild = null;
    _lastChild = null;
    _childCount = 0;
  }

  /// 移动一个子元素到指定的child（after）后面  
  void move(ChildType child, { ChildType after }) {
    ...
    final ParentDataType childParentData = child.parentData;
    if (childParentData.previousSibling == after)
      return;
    _removeFromChildList(child);
    _insertIntoChildList(child, after: after);
    markNeedsLayout();
  }
  
  @override
  /// 为child分配 PipelineOwner 并标记需要重新布局
  void attach(PipelineOwner owner) {
    super.attach(owner);
    ChildType child = _firstChild;
    while (child != null) {
      child.attach(owner);
      final ParentDataType childParentData = child.parentData;
      child = childParentData.nextSibling;
    }
  }

  @override
  void detach() {
    super.detach();
    ChildType child = _firstChild;
    while (child != null) {
      child.detach();
      final ParentDataType childParentData = child.parentData;
      child = childParentData.nextSibling;
    }
  }

  /// 为children重新分配depth  
  @override
  void redepthChildren() {
    ChildType child = _firstChild;
    while (child != null) {
      redepthChild(child);
      final ParentDataType childParentData = child.parentData;
      child = childParentData.nextSibling;
    }
  }

  /// 访问children  
  @override
  void visitChildren(RenderObjectVisitor visitor) {
    ChildType child = _firstChild;
    while (child != null) {
      visitor(child);
      final ParentDataType childParentData = child.parentData;
      child = childParentData.nextSibling;
    }
  }

  /// The first child in the child list.
  ChildType get firstChild => _firstChild;

  /// The last child in the child list.
  ChildType get lastChild => _lastChild;

  /// The previous child before the given child in the child list.
  ChildType childBefore(ChildType child) {
    assert(child != null);
    assert(child.parent == this);
    final ParentDataType childParentData = child.parentData;
    return childParentData.previousSibling;
  }

  /// The next child after the given child in the child list.
  ChildType childAfter(ChildType child) {
    assert(child != null);
    assert(child.parent == this);
    final ParentDataType childParentData = child.parentData;
    return childParentData.nextSibling;
  }
}
```



## RenderBoxContainerDefaultsMixin

```dart
mixin RenderBoxContainerDefaultsMixin
    <ChildType extends RenderBox, 
	ParentDataType extends ContainerBoxParentData<ChildType>> 
    implements ContainerRenderObjectMixin<ChildType, ParentDataType> 
{
  /// Returns the baseline of the first child with a baseline.
  ///
  /// Useful when the children are displayed vertically in the same order they
  /// appear in the child list.
  double defaultComputeDistanceToFirstActualBaseline(TextBaseline baseline) {
    assert(!debugNeedsLayout);
    ChildType child = firstChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      final double result = child.getDistanceToActualBaseline(baseline);
      if (result != null)
        return result + childParentData.offset.dy;
      child = childParentData.nextSibling;
    }
    return null;
  }

  /// Returns the minimum baseline value among every child.
  ///
  /// Useful when the vertical position of the children isn't determined by the
  /// order in the child list.
  double defaultComputeDistanceToHighestActualBaseline(TextBaseline baseline) {
    assert(!debugNeedsLayout);
    double result;
    ChildType child = firstChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      double candidate = child.getDistanceToActualBaseline(baseline);
      if (candidate != null) {
        candidate += childParentData.offset.dy;
        if (result != null)
          result = math.min(result, candidate);
        else
          result = candidate;
      }
      child = childParentData.nextSibling;
    }
    return result;
  }

  /// 利用在delegate.performLayout阶段调用的positionChild
  /// 为child的parentData附加的offset加上传入的position，来确定实际
  /// 的点击位置，并调用child.hitTest方法      
  bool defaultHitTestChildren(BoxHitTestResult result, { Offset position }) {
    // the x, y parameters have the top left of the node's box as the origin
    ChildType child = lastChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      final bool isHit = result.addWithPaintOffset(
        offset: childParentData.offset,
        position: position,
        hitTest: (BoxHitTestResult result, Offset transformed) {
          assert(transformed == position - childParentData.offset);
          return child.hitTest(result, position: transformed);
        },
      );
      if (isHit)
        return true;
      child = childParentData.previousSibling;
    }
    return false;
  }

  /// 利用在delegate.performLayout阶段调用的positionChild
  /// 为child的parentData附加的offset来确定绘制位置
  void defaultPaint(PaintingContext context, Offset offset) {
    ChildType child = firstChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      context.paintChild(child, childParentData.offset + offset);
      child = childParentData.nextSibling;
    }
  }

  /// 获取children列表
  List<ChildType> getChildrenAsList() {
    final List<ChildType> result = <ChildType>[];
    RenderBox child = firstChild;
    while (child != null) {
      final ParentDataType childParentData = child.parentData;
      result.add(child);
      child = childParentData.nextSibling;
    }
    return result;
  }
}
```



综上，几个mixin提供了链表的管理方法，默认的绘制方法，默认的点击测试，默认的计算baseline方法，供使用它们的类调用。


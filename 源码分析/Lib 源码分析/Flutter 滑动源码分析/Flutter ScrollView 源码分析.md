# Flutter ScrollView 源码分析

## ScrollView

```dart
/// 一个可以滚动的 widget
///
/// 可滚动的组件由以下三个部分组成
///
///  1. [Scrollable] widget 监听用户的手势变换并实现滚动交互设计
///  2. viewport widget,例如 [Viewport] 或者 [ShrinkWrappingViewport],实现了只展现滚动到视窗内的部分元素
///  3. 能够将更多的滚动组件例如（list，grid）组合在一起
///
/// Flutter 所有的滚动 widget 都继承自 ScrollView
abstract class ScrollView extends StatelessWidget {
  const ScrollView({
    Key key,
    this.scrollDirection = Axis.vertical,
    this.reverse = false,
    this.controller,
    bool primary,
    ScrollPhysics physics,
    this.shrinkWrap = false,
    this.center,
    this.anchor = 0.0,
    this.cacheExtent,
    this.semanticChildCount,
    this.dragStartBehavior = DragStartBehavior.start,
  }) : super(key: key);

  /// 滚动轴方向
  final Axis scrollDirection;

  /// 时候是反向滚动
  /// 1.当滚动轴是垂直的时候，默认从下向上滚动（即向上滑动），反向则是自上向下滚动
  /// 2.当滚动轴是水平的时候，默认从左向右滚动（即向上滑动），反向则是从右向左滚动  
  final bool reverse;

  final ScrollController controller;

  /// 是否与 [PrimaryScrollController] 有关联
  final bool primary;

  /// scrollView 如何响应用户手势输入(或者说是物理效果)
  final ScrollPhysics physics;

  /// scroll view 的大小是否应该被其展示的内容所决定
  final bool shrinkWrap;

  /// 决定哪一个 child 被展示在起点位置（会根据给出的 key 查找）
  /// 默认为第一个 child
  final Key center;
    
  /// 滚动起点的相对位置
  /// anchor 的值范围为 0.0 ~ 1.0
  final double anchor;

  /// 缓存的距离（即当组件移出当前视窗之后，并不会马上被销毁，而是当它移出缓存距离之后才会被销毁）
  /// 通常一个视窗的上下各有一段缓存距离
  final double cacheExtent;

  final int semanticChildCount;
  final DragStartBehavior dragStartBehavior;

  /// 返回这个 scroll view 的滚动 [AxisDirection]
  @protected
  AxisDirection getDirection(BuildContext context) {
    return getAxisDirectionFromAxisReverseAndDirectionality(context, scrollDirection, reverse);
  }

  /// 构建用于展示的 Sliver widget 列表
  @protected
  List<Widget> buildSlivers(BuildContext context);

  
  /// 这个方法被 build 方法作为 viewportBuilder 回调传递给 Scrollable 构造函数
  ///
  /// context 是 build 方法给定的 context
  /// `offset` 参数是从 [Scrollable.viewportBuilder] 获取到的值
  /// `axisDirection` 参数是从 [getDirection] 获取到的值
  /// `slivers` 参数是从 [buildSlivers] 获取到的值
  @protected
  Widget buildViewport(
    BuildContext context,
    ViewportOffset offset,
    AxisDirection axisDirection,
    List<Widget> slivers,
  ) {
    /// 根据 shrinkWrap 选择不同的 viewport 策略
    if (shrinkWrap) {
      return ShrinkWrappingViewport(
        axisDirection: axisDirection,
        offset: offset,
        slivers: slivers,
      );
    }
    return Viewport(
      axisDirection: axisDirection,
      offset: offset,
      slivers: slivers,
      cacheExtent: cacheExtent,
      center: center,
      anchor: anchor,
    );
  }

  @override
  Widget build(BuildContext context) {
    final List<Widget> slivers = buildSlivers(context);
    final AxisDirection axisDirection = getDirection(context);

    final ScrollController scrollController = primary
      ? PrimaryScrollController.of(context)
      : controller;
    
    /// 实际上返回的是一个 Scrollable
    final Scrollable scrollable = Scrollable(
      dragStartBehavior: dragStartBehavior,
      axisDirection: axisDirection,
      controller: scrollController,
      physics: physics,
      semanticChildCount: semanticChildCount,
      viewportBuilder: (BuildContext context, ViewportOffset offset) {
        return buildViewport(context, offset, axisDirection, slivers);
      },
    );
    return primary && scrollController != null
      ? PrimaryScrollController.none(child: scrollable)
      : scrollable;
  }
}
```



## CustomScrollView

```dart
/// 简单地封装了 ScrollView，提供了几个默认的属性值
class CustomScrollView extends ScrollView {
  const CustomScrollView({
    Key key,
    Axis scrollDirection = Axis.vertical,
    bool reverse = false,
    ScrollController controller,
    bool primary,
    ScrollPhysics physics,
    bool shrinkWrap = false,
    Key center,
    double anchor = 0.0,
    double cacheExtent,
    this.slivers = const <Widget>[],
    int semanticChildCount,
    DragStartBehavior dragStartBehavior = DragStartBehavior.start,
  }) : super(
    key: key,
    scrollDirection: scrollDirection,
    reverse: reverse,
    controller: controller,
    primary: primary,
    physics: physics,
    shrinkWrap: shrinkWrap,
    center: center,
    anchor: anchor,
    cacheExtent: cacheExtent,
    semanticChildCount: semanticChildCount,
    dragStartBehavior: dragStartBehavior,
  );

  /// 要求提供一组 Sliver widget（例如 SliverList，SliverGrid）
  final List<Widget> slivers;

  @override
  List<Widget> buildSlivers(BuildContext context) => slivers;
}
```



## BoxScrollView

```dart
/// 使用了单一布局模型的 Scrollview
abstract class BoxScrollView extends ScrollView {
  const BoxScrollView({
    Key key,
    Axis scrollDirection = Axis.vertical,
    bool reverse = false,
    ScrollController controller,
    bool primary,
    ScrollPhysics physics,
    bool shrinkWrap = false,
    this.padding,
    double cacheExtent,
    int semanticChildCount,
    DragStartBehavior dragStartBehavior = DragStartBehavior.start,
  }) : super(
    key: key,
    scrollDirection: scrollDirection,
    reverse: reverse,
    controller: controller,
    primary: primary,
    physics: physics,
    shrinkWrap: shrinkWrap,
    cacheExtent: cacheExtent,
    semanticChildCount: semanticChildCount,
    dragStartBehavior: dragStartBehavior,
  );

  /// 可滚动区域四周的 padding
  final EdgeInsetsGeometry padding;

  /// 重载父类的方法，为 scrollView 添加 padding
  @override
  List<Widget> buildSlivers(BuildContext context) {
    Widget sliver = buildChildLayout(context);
    EdgeInsetsGeometry effectivePadding = padding;
    if (padding == null) {
      final MediaQueryData mediaQuery = MediaQuery.of(context, nullOk: true);
      if (mediaQuery != null) {
        final EdgeInsets mediaQueryHorizontalPadding =
            mediaQuery.padding.copyWith(top: 0.0, bottom: 0.0);
        final EdgeInsets mediaQueryVerticalPadding =
            mediaQuery.padding.copyWith(left: 0.0, right: 0.0);
        // Consume the main axis padding with SliverPadding.
        effectivePadding = scrollDirection == Axis.vertical
            ? mediaQueryVerticalPadding
            : mediaQueryHorizontalPadding;
        // Leave behind the cross axis padding.
        sliver = MediaQuery(
          data: mediaQuery.copyWith(
            padding: scrollDirection == Axis.vertical
                ? mediaQueryHorizontalPadding
                : mediaQueryVerticalPadding,
          ),
          child: sliver,
        );
      }
    }

    if (effectivePadding != null)
      sliver = SliverPadding(padding: effectivePadding, sliver: sliver);
    return <Widget>[ sliver ];
  }

  @protected
  Widget buildChildLayout(BuildContext context);
}
```



## ListView

```dart

/// A scrollable list of widgets arranged linearly.
///
/// {@youtube 560 315 https://www.youtube.com/watch?v=KJpkjHGiI5A}
///
/// [ListView] is the most commonly used scrolling widget. It displays its
/// children one after another in the scroll direction. In the cross axis, the
/// children are required to fill the [ListView].
///
/// If non-null, the [itemExtent] forces the children to have the given extent
/// in the scroll direction. Specifying an [itemExtent] is more efficient than
/// letting the children determine their own extent because the scrolling
/// machinery can make use of the foreknowledge of the children's extent to save
/// work, for example when the scroll position changes drastically.
///
/// There are four options for constructing a [ListView]:
///
///  1. The default constructor takes an explicit [List<Widget>] of children. This
///     constructor is appropriate for list views with a small number of
///     children because constructing the [List] requires doing work for every
///     child that could possibly be displayed in the list view instead of just
///     those children that are actually visible.
///
///  2. The [ListView.builder] constructor takes an [IndexedWidgetBuilder], which
///     builds the children on demand. This constructor is appropriate for list views
///     with a large (or infinite) number of children because the builder is called
///     only for those children that are actually visible.
///
///  3. The [ListView.separated] constructor takes two [IndexedWidgetBuilder]s:
///     `itemBuilder` builds child items on demand, and `separatorBuilder`
///     similarly builds separator children which appear in between the child items.
///     This constructor is appropriate for list views with a fixed number of children.
///
///  4. The [ListView.custom] constructor takes a [SliverChildDelegate], which provides
///     the ability to customize additional aspects of the child model. For example,
///     a [SliverChildDelegate] can control the algorithm used to estimate the
///     size of children that are not actually visible.
///
/// To control the initial scroll offset of the scroll view, provide a
/// [controller] with its [ScrollController.initialScrollOffset] property set.
///
/// By default, [ListView] will automatically pad the list's scrollable
/// extremities to avoid partial obstructions indicated by [MediaQuery]'s
/// padding. To avoid this behavior, override with a zero [padding] property.
///
/// {@tool sample}
/// This example uses the default constructor for [ListView] which takes an
/// explicit [List<Widget>] of children. This [ListView]'s children are made up
/// of [Container]s with [Text].
///
/// ![A ListView of 3 amber colored containers with sample text.](https://flutter.github.io/assets-for-api-docs/assets/widgets/list_view.png)
///
/// ```dart
/// ListView(
///   padding: const EdgeInsets.all(8),
///   children: <Widget>[
///     Container(
///       height: 50,
///       color: Colors.amber[600],
///       child: const Center(child: Text('Entry A')),
///     ),
///     Container(
///       height: 50,
///       color: Colors.amber[500],
///       child: const Center(child: Text('Entry B')),
///     ),
///     Container(
///       height: 50,
///       color: Colors.amber[100],
///       child: const Center(child: Text('Entry C')),
///     ),
///   ],
/// )
/// ```
/// {@end-tool}
///
/// {@tool sample}
/// This example mirrors the previous one, creating the same list using the
/// [ListView.builder] constructor. Using the [IndexedWidgetBuilder], children
/// are built lazily and can be infinite in number.
///
/// ![A ListView of 3 amber colored containers with sample text.](https://flutter.github.io/assets-for-api-docs/assets/widgets/list_view_builder.png)
///
/// ```dart
/// final List<String> entries = <String>['A', 'B', 'C'];
/// final List<int> colorCodes = <int>[600, 500, 100];
///
/// ListView.builder(
///   padding: const EdgeInsets.all(8),
///   itemCount: entries.length,
///   itemBuilder: (BuildContext context, int index) {
///     return Container(
///       height: 50,
///       color: Colors.amber[colorCodes[index]],
///       child: Center(child: Text('Entry ${entries[index]}')),
///     );
///   }
/// );
/// ```
/// {@end-tool}
///
/// {@tool sample}
/// This example continues to build from our the previous ones, creating a
/// similar list using [ListView.separated]. Here, a [Divider] is used as a
/// separator.
///
/// ![A ListView of 3 amber colored containers with sample text and a Divider
/// between each of them.](https://flutter.github.io/assets-for-api-docs/assets/widgets/list_view_separated.png)
///
/// ```dart
/// final List<String> entries = <String>['A', 'B', 'C'];
/// final List<int> colorCodes = <int>[600, 500, 100];
///
/// ListView.separated(
///   padding: const EdgeInsets.all(8),
///   itemCount: entries.length,
///   itemBuilder: (BuildContext context, int index) {
///     return Container(
///       height: 50,
///       color: Colors.amber[colorCodes[index]],
///       child: Center(child: Text('Entry ${entries[index]}')),
///     );
///   },
///   separatorBuilder: (BuildContext context, int index) => const Divider(),
/// );
/// ```
/// {@end-tool}
///
/// ## Child elements' lifecycle
///
/// ### Creation
///
/// While laying out the list, visible children's elements, states and render
/// objects will be created lazily based on existing widgets (such as when using
/// the default constructor) or lazily provided ones (such as when using the
/// [ListView.builder] constructor).
///
/// ### Destruction
///
/// When a child is scrolled out of view, the associated element subtree,
/// states and render objects are destroyed. A new child at the same position
/// in the list will be lazily recreated along with new elements, states and
/// render objects when it is scrolled back.
///
/// ### Destruction mitigation
///
/// In order to preserve state as child elements are scrolled in and out of
/// view, the following options are possible:
///
///  * Moving the ownership of non-trivial UI-state-driving business logic
///    out of the list child subtree. For instance, if a list contains posts
///    with their number of upvotes coming from a cached network response, store
///    the list of posts and upvote number in a data model outside the list. Let
///    the list child UI subtree be easily recreate-able from the
///    source-of-truth model object. Use [StatefulWidget]s in the child
///    widget subtree to store instantaneous UI state only.
///
///  * Letting [KeepAlive] be the root widget of the list child widget subtree
///    that needs to be preserved. The [KeepAlive] widget marks the child
///    subtree's top render object child for keepalive. When the associated top
///    render object is scrolled out of view, the list keeps the child's render
///    object (and by extension, its associated elements and states) in a cache
///    list instead of destroying them. When scrolled back into view, the render
///    object is repainted as-is (if it wasn't marked dirty in the interim).
///
///    This only works if [addAutomaticKeepAlives] and [addRepaintBoundaries]
///    are false since those parameters cause the [ListView] to wrap each child
///    widget subtree with other widgets.
///
///  * Using [AutomaticKeepAlive] widgets (inserted by default when
///    [addAutomaticKeepAlives] is true). Instead of unconditionally caching the
///    child element subtree when scrolling off-screen like [KeepAlive],
///    [AutomaticKeepAlive] can let whether to cache the subtree be determined
///    by descendant logic in the subtree.
///
///    As an example, the [EditableText] widget signals its list child element
///    subtree to stay alive while its text field has input focus. If it doesn't
///    have focus and no other descendants signaled for keepalive via a
///    [KeepAliveNotification], the list child element subtree will be destroyed
///    when scrolled away.
///
///    [AutomaticKeepAlive] descendants typically signal it to be kept alive
///    by using the [AutomaticKeepAliveClientMixin], then implementing the
///    [wantKeepAlive] getter and calling [updateKeepAlive].
///
/// ## Transitioning to [CustomScrollView]
///
/// A [ListView] is basically a [CustomScrollView] with a single [SliverList] in
/// its [CustomScrollView.slivers] property.
///
/// If [ListView] is no longer sufficient, for example because the scroll view
/// is to have both a list and a grid, or because the list is to be combined
/// with a [SliverAppBar], etc, it is straight-forward to port code from using
/// [ListView] to using [CustomScrollView] directly.
///
/// The [key], [scrollDirection], [reverse], [controller], [primary], [physics],
/// and [shrinkWrap] properties on [ListView] map directly to the identically
/// named properties on [CustomScrollView].
///
/// The [CustomScrollView.slivers] property should be a list containing either a
/// [SliverList] or a [SliverFixedExtentList]; the former if [itemExtent] on the
/// [ListView] was null, and the latter if [itemExtent] was not null.
///
/// The [childrenDelegate] property on [ListView] corresponds to the
/// [SliverList.delegate] (or [SliverFixedExtentList.delegate]) property. The
/// [new ListView] constructor's `children` argument corresponds to the
/// [childrenDelegate] being a [SliverChildListDelegate] with that same
/// argument. The [new ListView.builder] constructor's `itemBuilder` and
/// `itemCount` arguments correspond to the [childrenDelegate] being a
/// [SliverChildBuilderDelegate] with the equivalent arguments.
///
/// The [padding] property corresponds to having a [SliverPadding] in the
/// [CustomScrollView.slivers] property instead of the list itself, and having
/// the [SliverList] instead be a child of the [SliverPadding].
///
/// [CustomScrollView]s don't automatically avoid obstructions from [MediaQuery]
/// like [ListView]s do. To reproduce the behavior, wrap the slivers in
/// [SliverSafeArea]s.
///
/// Once code has been ported to use [CustomScrollView], other slivers, such as
/// [SliverGrid] or [SliverAppBar], can be put in the [CustomScrollView.slivers]
/// list.
///
/// {@tool sample}
///
/// Here are two brief snippets showing a [ListView] and its equivalent using
/// [CustomScrollView]:
///
/// ```dart
/// ListView(
///   shrinkWrap: true,
///   padding: const EdgeInsets.all(20.0),
///   children: <Widget>[
///     const Text('I\'m dedicating every day to you'),
///     const Text('Domestic life was never quite my style'),
///     const Text('When you smile, you knock me out, I fall apart'),
///     const Text('And I thought I was so smart'),
///   ],
/// )
/// ```
/// {@end-tool}
/// {@tool sample}
///
/// ```dart
/// CustomScrollView(
///   shrinkWrap: true,
///   slivers: <Widget>[
///     SliverPadding(
///       padding: const EdgeInsets.all(20.0),
///       sliver: SliverList(
///         delegate: SliverChildListDelegate(
///           <Widget>[
///             const Text('I\'m dedicating every day to you'),
///             const Text('Domestic life was never quite my style'),
///             const Text('When you smile, you knock me out, I fall apart'),
///             const Text('And I thought I was so smart'),
///           ],
///         ),
///       ),
///     ),
///   ],
/// )
/// ```
/// {@end-tool}
///
/// See also:
///
///  * [SingleChildScrollView], which is a scrollable widget that has a single
///    child.
///  * [PageView], which is a scrolling list of child widgets that are each the
///    size of the viewport.
///  * [GridView], which is scrollable, 2D array of widgets.
///  * [CustomScrollView], which is a scrollable widget that creates custom
///    scroll effects using slivers.
///  * [ListBody], which arranges its children in a similar manner, but without
///    scrolling.
///  * [ScrollNotification] and [NotificationListener], which can be used to watch
///    the scroll position without using a [ScrollController].
class ListView extends BoxScrollView {
  /// Creates a scrollable, linear array of widgets from an explicit [List].
  ///
  /// This constructor is appropriate for list views with a small number of
  /// children because constructing the [List] requires doing work for every
  /// child that could possibly be displayed in the list view instead of just
  /// those children that are actually visible.
  ///
  /// It is usually more efficient to create children on demand using [new
  /// ListView.builder].
  ///
  /// The `addAutomaticKeepAlives` argument corresponds to the
  /// [SliverChildListDelegate.addAutomaticKeepAlives] property. The
  /// `addRepaintBoundaries` argument corresponds to the
  /// [SliverChildListDelegate.addRepaintBoundaries] property. The
  /// `addSemanticIndexes` argument corresponds to the
  /// [SliverChildListDelegate.addSemanticIndexes] property. None
  /// may be null.
  ListView({
    Key key,
    Axis scrollDirection = Axis.vertical,
    bool reverse = false,
    ScrollController controller,
    bool primary,
    ScrollPhysics physics,
    bool shrinkWrap = false,
    EdgeInsetsGeometry padding,
    this.itemExtent,
    bool addAutomaticKeepAlives = true,
    bool addRepaintBoundaries = true,
    bool addSemanticIndexes = true,
    double cacheExtent,
    List<Widget> children = const <Widget>[],
    int semanticChildCount,
    DragStartBehavior dragStartBehavior = DragStartBehavior.start,
  }) : childrenDelegate = SliverChildListDelegate(
         children,
         addAutomaticKeepAlives: addAutomaticKeepAlives,
         addRepaintBoundaries: addRepaintBoundaries,
         addSemanticIndexes: addSemanticIndexes,
       ),
       super(
         key: key,
         scrollDirection: scrollDirection,
         reverse: reverse,
         controller: controller,
         primary: primary,
         physics: physics,
         shrinkWrap: shrinkWrap,
         padding: padding,
         cacheExtent: cacheExtent,
         semanticChildCount: semanticChildCount ?? children.length,
         dragStartBehavior: dragStartBehavior,
       );

  /// Creates a scrollable, linear array of widgets that are created on demand.
  ///
  /// This constructor is appropriate for list views with a large (or infinite)
  /// number of children because the builder is called only for those children
  /// that are actually visible.
  ///
  /// Providing a non-null `itemCount` improves the ability of the [ListView] to
  /// estimate the maximum scroll extent.
  ///
  /// The `itemBuilder` callback will be called only with indices greater than
  /// or equal to zero and less than `itemCount`.
  ///
  /// The `itemBuilder` should actually create the widget instances when called.
  /// Avoid using a builder that returns a previously-constructed widget; if the
  /// list view's children are created in advance, or all at once when the
  /// [ListView] itself is created, it is more efficient to use [new ListView].
  /// Even more efficient, however, is to create the instances on demand using
  /// this constructor's `itemBuilder` callback.
  ///
  /// The `addAutomaticKeepAlives` argument corresponds to the
  /// [SliverChildBuilderDelegate.addAutomaticKeepAlives] property. The
  /// `addRepaintBoundaries` argument corresponds to the
  /// [SliverChildBuilderDelegate.addRepaintBoundaries] property. The
  /// `addSemanticIndexes` argument corresponds to the
  /// [SliverChildBuilderDelegate.addSemanticIndexes] property. None may be
  /// null.
  ///
  /// [ListView.builder] by default does not support child reordering. If
  /// you are planning to change child order at a later time, consider using
  /// [ListView] or [ListView.custom].
  ListView.builder({
    Key key,
    Axis scrollDirection = Axis.vertical,
    bool reverse = false,
    ScrollController controller,
    bool primary,
    ScrollPhysics physics,
    bool shrinkWrap = false,
    EdgeInsetsGeometry padding,
    this.itemExtent,
    @required IndexedWidgetBuilder itemBuilder,
    int itemCount,
    bool addAutomaticKeepAlives = true,
    bool addRepaintBoundaries = true,
    bool addSemanticIndexes = true,
    double cacheExtent,
    int semanticChildCount,
    DragStartBehavior dragStartBehavior = DragStartBehavior.start,
  }) : assert(itemCount == null || itemCount >= 0),
       assert(semanticChildCount == null || semanticChildCount <= itemCount),
       childrenDelegate = SliverChildBuilderDelegate(
         itemBuilder,
         childCount: itemCount,
         addAutomaticKeepAlives: addAutomaticKeepAlives,
         addRepaintBoundaries: addRepaintBoundaries,
         addSemanticIndexes: addSemanticIndexes,
       ),
       super(
         key: key,
         scrollDirection: scrollDirection,
         reverse: reverse,
         controller: controller,
         primary: primary,
         physics: physics,
         shrinkWrap: shrinkWrap,
         padding: padding,
         cacheExtent: cacheExtent,
         semanticChildCount: semanticChildCount ?? itemCount,
         dragStartBehavior: dragStartBehavior,
       );

  /// Creates a fixed-length scrollable linear array of list "items" separated
  /// by list item "separators".
  ///
  /// This constructor is appropriate for list views with a large number of
  /// item and separator children because the builders are only called for
  /// the children that are actually visible.
  ///
  /// The `itemBuilder` callback will be called with indices greater than
  /// or equal to zero and less than `itemCount`.
  ///
  /// Separators only appear between list items: separator 0 appears after item
  /// 0 and the last separator appears before the last item.
  ///
  /// The `separatorBuilder` callback will be called with indices greater than
  /// or equal to zero and less than `itemCount - 1`.
  ///
  /// The `itemBuilder` and `separatorBuilder` callbacks should actually create
  /// widget instances when called. Avoid using a builder that returns a
  /// previously-constructed widget; if the list view's children are created in
  /// advance, or all at once when the [ListView] itself is created, it is more
  /// efficient to use [new ListView].
  ///
  /// {@tool sample}
  ///
  /// This example shows how to create [ListView] whose [ListTile] list items
  /// are separated by [Divider]s.
  ///
  /// ```dart
  /// ListView.separated(
  ///   itemCount: 25,
  ///   separatorBuilder: (BuildContext context, int index) => Divider(),
  ///   itemBuilder: (BuildContext context, int index) {
  ///     return ListTile(
  ///       title: Text('item $index'),
  ///     );
  ///   },
  /// )
  /// ```
  /// {@end-tool}
  ///
  /// The `addAutomaticKeepAlives` argument corresponds to the
  /// [SliverChildBuilderDelegate.addAutomaticKeepAlives] property. The
  /// `addRepaintBoundaries` argument corresponds to the
  /// [SliverChildBuilderDelegate.addRepaintBoundaries] property. The
  /// `addSemanticIndexes` argument corresponds to the
  /// [SliverChildBuilderDelegate.addSemanticIndexes] property. None may be
  /// null.
  ListView.separated({
    Key key,
    Axis scrollDirection = Axis.vertical,
    bool reverse = false,
    ScrollController controller,
    bool primary,
    ScrollPhysics physics,
    bool shrinkWrap = false,
    EdgeInsetsGeometry padding,
    @required IndexedWidgetBuilder itemBuilder,
    @required IndexedWidgetBuilder separatorBuilder,
    @required int itemCount,
    bool addAutomaticKeepAlives = true,
    bool addRepaintBoundaries = true,
    bool addSemanticIndexes = true,
    double cacheExtent,
  }) : assert(itemBuilder != null),
       assert(separatorBuilder != null),
       assert(itemCount != null && itemCount >= 0),
       itemExtent = null,
       childrenDelegate = SliverChildBuilderDelegate(
         (BuildContext context, int index) {
           final int itemIndex = index ~/ 2;
           Widget widget;
           if (index.isEven) {
             widget = itemBuilder(context, itemIndex);
           } else {
             widget = separatorBuilder(context, itemIndex);
             assert(() {
               if (widget == null) {
                 throw FlutterError('separatorBuilder cannot return null.');
               }
               return true;
             }());
           }
           return widget;
         },
         childCount: _computeSemanticChildCount(itemCount),
         addAutomaticKeepAlives: addAutomaticKeepAlives,
         addRepaintBoundaries: addRepaintBoundaries,
         addSemanticIndexes: addSemanticIndexes,
         semanticIndexCallback: (Widget _, int index) {
           return index.isEven ? index ~/ 2 : null;
         },
       ),
       super(
         key: key,
         scrollDirection: scrollDirection,
         reverse: reverse,
         controller: controller,
         primary: primary,
         physics: physics,
         shrinkWrap: shrinkWrap,
         padding: padding,
         cacheExtent: cacheExtent,
         semanticChildCount: _computeSemanticChildCount(itemCount),
       );

  /// Creates a scrollable, linear array of widgets with a custom child model.
  ///
  /// For example, a custom child model can control the algorithm used to
  /// estimate the size of children that are not actually visible.
  ///
  /// {@tool sample}
  ///
  /// This [ListView] uses a custom [SliverChildBuilderDelegate] to support child
  /// reordering.
  ///
  /// ```dart
  /// class MyListView extends StatefulWidget {
  ///   @override
  ///   _MyListViewState createState() => _MyListViewState();
  /// }
  ///
  /// class _MyListViewState extends State<MyListView> {
  ///   List<String> items = <String>['1', '2', '3', '4', '5'];
  ///
  ///   void _reverse() {
  ///     setState(() {
  ///       items = items.reversed.toList();
  ///     });
  ///   }
  ///
  ///   @override
  ///   Widget build(BuildContext context) {
  ///     return Scaffold(
  ///       body: SafeArea(
  ///         child: ListView.custom(
  ///           childrenDelegate: SliverChildBuilderDelegate(
  ///             (BuildContext context, int index) {
  ///               return KeepAlive(
  ///                 data: items[index],
  ///                 key: ValueKey<String>(items[index]),
  ///               );
  ///             },
  ///             childCount: items.length,
  ///             findChildIndexCallback: (Key key) {
  ///               final ValueKey valueKey = key;
  ///               final String data = valueKey.value;
  ///               return items.indexOf(data);
  ///             }
  ///           ),
  ///         ),
  ///       ),
  ///       bottomNavigationBar: BottomAppBar(
  ///         child: Row(
  ///           mainAxisAlignment: MainAxisAlignment.center,
  ///           children: <Widget>[
  ///             FlatButton(
  ///               onPressed: () => _reverse(),
  ///               child: Text('Reverse items'),
  ///             ),
  ///           ],
  ///         ),
  ///       ),
  ///     );
  ///   }
  /// }
  ///
  /// class KeepAlive extends StatefulWidget {
  ///   const KeepAlive({Key key, this.data}) : super(key: key);
  ///
  ///   final String data;
  ///
  ///   @override
  ///   _KeepAliveState createState() => _KeepAliveState();
  /// }
  ///
  /// class _KeepAliveState extends State<KeepAlive> with AutomaticKeepAliveClientMixin{
  ///   @override
  ///   bool get wantKeepAlive => true;
  ///
  ///   @override
  ///   Widget build(BuildContext context) {
  ///     super.build(context);
  ///     return Text(widget.data);
  ///   }
  /// }
  /// ```
  /// {@end-tool}
  const ListView.custom({
    Key key,
    Axis scrollDirection = Axis.vertical,
    bool reverse = false,
    ScrollController controller,
    bool primary,
    ScrollPhysics physics,
    bool shrinkWrap = false,
    EdgeInsetsGeometry padding,
    this.itemExtent,
    @required this.childrenDelegate,
    double cacheExtent,
    int semanticChildCount,
  }) : assert(childrenDelegate != null),
       super(
         key: key,
         scrollDirection: scrollDirection,
         reverse: reverse,
         controller: controller,
         primary: primary,
         physics: physics,
         shrinkWrap: shrinkWrap,
         padding: padding,
         cacheExtent: cacheExtent,
         semanticChildCount: semanticChildCount,
       );

  /// If non-null, forces the children to have the given extent in the scroll
  /// direction.
  ///
  /// Specifying an [itemExtent] is more efficient than letting the children
  /// determine their own extent because the scrolling machinery can make use of
  /// the foreknowledge of the children's extent to save work, for example when
  /// the scroll position changes drastically.
  final double itemExtent;

  /// A delegate that provides the children for the [ListView].
  ///
  /// The [ListView.custom] constructor lets you specify this delegate
  /// explicitly. The [ListView] and [ListView.builder] constructors create a
  /// [childrenDelegate] that wraps the given [List] and [IndexedWidgetBuilder],
  /// respectively.
  final SliverChildDelegate childrenDelegate;

  @override
  Widget buildChildLayout(BuildContext context) {
    if (itemExtent != null) {
      return SliverFixedExtentList(
        delegate: childrenDelegate,
        itemExtent: itemExtent,
      );
    }
    return SliverList(delegate: childrenDelegate);
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(DoubleProperty('itemExtent', itemExtent, defaultValue: null));
  }

  // Helper method to compute the semantic child count for the separated constructor.
  static int _computeSemanticChildCount(int itemCount) {
    return math.max(0, itemCount * 2 - 1);
  }
}
```


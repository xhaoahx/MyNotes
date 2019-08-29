# 关于 Flutter Gallery 中的 Animation Demo

[TOC]



## 状态栏

```dart
/// 渲染类
class _RenderStatusBarPaddingSliver extends RenderSliver {
  _RenderStatusBarPaddingSliver({
    @required double maxHeight,
    @required double scrollFactor,
  }) : assert(maxHeight != null && maxHeight >= 0.0),
       assert(scrollFactor != null && scrollFactor >= 1.0),
       _maxHeight = maxHeight,
       _scrollFactor = scrollFactor;

  // 状态栏的高度
  double get maxHeight => _maxHeight;
  double _maxHeight;
  set maxHeight(double value) {
    assert(maxHeight != null && maxHeight >= 0.0);
    if (_maxHeight == value)
      return;
    _maxHeight = value;
    // 当最大高度改变时，标记需要重新布局
    markNeedsLayout();
  }

  // 控制当ShrinkOffset改变时，状态栏的高度变化比例
  double get scrollFactor => _scrollFactor;
  double _scrollFactor;
  set scrollFactor(double value) {
    assert(scrollFactor != null && scrollFactor >= 1.0);
    if (_scrollFactor == value)
      return;
    _scrollFactor = value;
    markNeedsLayout();
  }

  @override
  void performLayout() {
    // this means when user scroll the sliver up,the status bar's height decreases.
    // 这意味着当使用者向上滑动滑块，状态栏的高度会减少
    // 高度 = （最大高度 - ScrollOffset / 比例因素）取（0，0 到 最大高度之间） 
    // assert(scrollFactor != null && scrollFactor >= 1.0),确保高度一定会减少  
    final double height = (maxHeight - constraints.scrollOffset / scrollFactor).clamp(0.0, maxHeight);
    // 通过改变 geometry来提供布局和绘制信息  
    geometry = SliverGeometry(
      paintExtent: math.min(height, constraints.remainingPaintExtent),
      scrollExtent: maxHeight,
      maxPaintExtent: maxHeight,
    );
  }
}

/// RenderSliver 中
SliverGeometry get geometry => _geometry;
SliverGeometry _geometry;
set geometry(SliverGeometry value) {
    ...
    _geometry = value;
}

/// 相对应的widget
class _StatusBarPaddingSliver extends SingleChildRenderObjectWidget {
  const _StatusBarPaddingSliver({
    Key key,
    @required this.maxHeight,
    this.scrollFactor = 5.0,
  }) : assert(maxHeight != null && maxHeight >= 0.0),
       assert(scrollFactor != null && scrollFactor >= 1.0),
       super(key: key);

  final double maxHeight;
  final double scrollFactor;

  // 构建渲染类的方法  
  @override
  _RenderStatusBarPaddingSliver createRenderObject(BuildContext context) {
    return _RenderStatusBarPaddingSliver(
      maxHeight: maxHeight,
      scrollFactor: scrollFactor,
    );
  }

  // instead of recreate a new render object,reusing it
  // 重用渲染对象，而不是重建它（提升效率） 
  @override
  void updateRenderObject(BuildContext context, _RenderStatusBarPaddingSliver renderObject) {
    renderObject
      ..maxHeight = maxHeight
      ..scrollFactor = scrollFactor;
  }
  ...
}
```



## 板块布局

```dart

// 安排板块标题，指示器，卡片。只有在布局从垂直变化到水平的时候，卡片被加入。
// 一旦布局变成水平，卡片将用PageView展示
//
// 卡片，标题和指示器的布局被两个 t 基于布局高度的参数定义：
// - tColumnToRow
//   0.0：当高度是最大高度并且布局是列的时候
//   1.0：当高度是中间高度并且布局是行的时候
// - tCollapsed
//   0.0：当高度是中间高度并且布局是行的时候
//   1.0：当高度是最小高度并且布局是行的时候
//   简言之：tColumnToRow 控制列变行（但没到最小高度），tCollapsed控制行（从最中间高度到最小高度）
//
// minHeight < midHeight < maxHeight
//
// 总体来说，这里的方法是计算列布局和行大小和各个元素的位置，利用 tColumnToRow 来插值。
// 一旦 tColumnToRow 达到 1.0 ，布局将用 tCollapsed 来控制。
// 当 tCollapsed 增加时，标题分散开来（最终仅一个可见），指示器聚集（最终全部可见）


class _AllSectionsLayout extends MultiChildLayoutDelegate {
  _AllSectionsLayout({
    this.translation,
    this.tColumnToRow,
    this.tCollapsed,
    this.cardCount,
    this.selectedIndex,
  });

  // 排列  
  final Alignment translation;
  final double tColumnToRow;
  final double tCollapsed;
  // 卡片数量  
  final int cardCount;
  // 当前选中卡片  
  final double selectedIndex;

  // 矩形插值  
  Rect _interpolateRect(Rect begin, Rect end) {
    return Rect.lerp(begin, end, tColumnToRow);
  }

  // 偏移量插值 
  Offset _interpolatePoint(Offset begin, Offset end) {
    return Offset.lerp(begin, end, tColumnToRow);
  }

  // 通过指定的大小来布局  
  // 未重载getSize，使用默认的最大约束当作大小
  // Size getSize(BoxConstraints constraints) => constraints.biggest;  
  @override
  void performLayout(Size size) {
    // 列状态下，卡片的X距离  
    final double columnCardX = size.width / 4.0;
    // 列状态下，卡片宽度  
    final double columnCardWidth = size.width - columnCardX;
    // 列状态下，卡片高度  
    final double columnCardHeight = size.height / cardCount;
    // 行状态下，卡片宽度  
    final double rowCardWidth = size.width;
    // 根据排列和大小来返回一个点距离原点的偏移
    // Offset alongSize(Size other) {
    //  final double centerX = other.width / 2.0;
    //  final double centerY = other.height / 2.0;
    //  return Offset(centerX + x * centerX, centerY + y * centerY);
    // }
    final Offset offset = translation.alongSize(size);
    // 列状态下，卡片的Y距离
    double columnCardY = 0.0;
    // 行状态下，卡片的X距离
    // 即 -（选中卡片 * 卡片宽度）
    double rowCardX = -(selectedIndex * rowCardWidth);

    // 当  tCollapsed > 0 标题分开
    // 列状态时，标题的X距离
    final double columnTitleX = size.width / 10.0;
    // 行状态时，标题的宽度
    // 当tCollapsed增加时，这个长度是增加的
    final double rowTitleWidth = size.width * ((1 + tCollapsed) / 2.25);
    // 行状态时，标题的X距离
    double rowTitleX = 
    	(size.width - rowTitleWidth) / 2.0 - selectedIndex *
        rowTitleWidth;

    // 当 tCollapsed > 0, 指示器靠近在一起
    // 计算包括padding的指示器宽度，等于原宽度 + 8.0
    const double paddedSectionIndicatorWidth = 
    	kSectionIndicatorWidth + 8.0;
    
    // 行状态时，指示器的宽度
    final double rowIndicatorWidth = 
    	paddedSectionIndicatorWidth + (1.0 - tCollapsed) * 
    	(rowTitleWidth - paddedSectionIndicatorWidth);
    	
    double rowIndicatorX = (size.width - rowIndicatorWidth) / 2.0 - selectedIndex * rowIndicatorWidth;
    
    // 计算当最大高度的列布局，中间高度的行布局时，
    // 每一张卡片，标题和指示器的大小和和原点。
    // 其余情况则为插值计算
    
    for (int index = 0; index < cardCount; index++) {

      // 布局当前index的卡片
      // 列状态时的卡片矩形
      final Rect columnCardRect = Rect.fromLTWH(
      	columnCardX, columnCardY, columnCardWidth, columnCardHeight);
      // 行状态时的卡片矩形	
      final Rect rowCardRect = Rect.fromLTWH(
      	rowCardX, 0.0, rowCardWidth, size.height);
      	
      // 通过 tColumnToRow 来算出 行列 矩形的插值，并将移动Offset的偏移	
      final Rect cardRect = _interpolateRect(
      	columnCardRect, rowCardRect).shift(offset);
      	
      final String cardId = 'card$index';
      
      // 如果有对应的孩子，则接着请求孩子布局
      if (hasChild(cardId)) {
      	// 将刚才算出的卡片矩形的大小作为严格约束传入
        layoutChild(cardId, BoxConstraints.tight(cardRect.size));
        // 将工程算出的卡片矩形的偏移作为孩子位置
        positionChild(cardId, cardRect.topLeft);
      }

      // 布局当前index的标题
      // 将刚才算出的卡片矩形的大小作为宽松约束传入，得到标题的大小
      // BoxConstraints.loose(Size size)
      // :  minWidth = 0.0,
      //    maxWidth = size.width,
      //    minHeight = 0.0,
      //    maxHeight = size.height;
      final Size titleSize = layoutChild(
        'title$index', BoxConstraints.loose(cardRect.size));
      // 列状态标题的Y距离  
      final double columnTitleY = 
      	columnCardRect.centerLeft.dy - titleSize.height / 2.0;
      // 行状态标题的X距离	
      final double rowTitleY = 
      	rowCardRect.centerLeft.dy - titleSize.height / 2.0;
      // 行状态标题的X距离（加上了卡片标题的宽度）	
      final double centeredRowTitleX = 
      	rowTitleX + (rowTitleWidth - titleSize.width) / 2.0;
      // 计算出列状态标题的远点	
      final Offset columnTitleOrigin = 
      	Offset(columnTitleX, columnTitleY);
      // 计算出行状态标题的原点	
      final Offset rowTitleOrigin = 
      	Offset(centeredRowTitleX, rowTitleY);
      	
      // 根据插值来计算 tColumnToRow 时标题的原点	
      final Offset titleOrigin = 
      	_interpolatePoint(columnTitleOrigin, rowTitleOrigin);
      // 放置标题在指定位置
      positionChild('title$index', titleOrigin + offset);

	 // 指示器类似，略过
     ...
     // 加上对应的指数，来计算下一个卡片
      columnCardY += columnCardHeight;
      rowCardX += rowCardWidth;
      rowTitleX += rowTitleWidth;
      rowIndicatorX += rowIndicatorWidth;
    }
  }

  // 是否应该重新布局
  @override
  bool shouldRelayout(_AllSectionsLayout oldDelegate) {
    return tColumnToRow != oldDelegate.tColumnToRow
      || cardCount != oldDelegate.cardCount
      || selectedIndex != oldDelegate.selectedIndex;
  }
}


class _AllSectionsView extends AnimatedWidget {
  _AllSectionsView({
    Key key,
    this.sectionIndex,
    @required this.sections,
    @required this.selectedIndex,
    this.minHeight,
    this.midHeight,
    this.maxHeight,
    this.sectionCards = const <Widget>[],
  }) : assert(sections != null),
       assert(sectionCards != null),
       assert(sectionCards.length == sections.length),
       assert(sectionIndex >= 0 && sectionIndex < sections.length),
       assert(selectedIndex != null),
       assert(selectedIndex.value >= 0.0 && selectedIndex.value < sections.length.toDouble()),
       // 将selctedIndex设置为listenable
       super(key: key, listenable: selectedIndex);

  final int sectionIndex;
  // section 配置
  final List<Section> sections;
  // 当这个值发生变化时，调用build方法
  final ValueNotifier<double> selectedIndex;
  // 最小高度
  final double minHeight;
  // 中间高度
  final double midHeight;
  // 最大高度
  final double maxHeight;
  final List<Widget> sectionCards;

  // 计算选中的卡片index和指定卡片index的差值，限定在（0.0到1.0）
  double _selectedIndexDelta(int index) {
    return (index.toDouble() - selectedIndex.value).abs().clamp(0.0, 1.0);
  }

  Widget _build(BuildContext context, BoxConstraints constraints) {
    final Size size = constraints.biggest;
    // 使用约束的最大值作为大小
    // 布局的进度时从列变成行
    final double tColumnToRow =
      1.0 - ((size.height - midHeight) / (maxHeight - midHeight)).
      clamp(0.0, 1.0);
    final double tCollapsed =
      1.0 - ((size.height - minHeight) /
             (midHeight - minHeight)).clamp(0.0, 1.0);

    // 计算指定 index 的指示器的透明度
    double _indicatorOpacity(int index) {
      return 1.0 - _selectedIndexDelta(index) * 0.5;
    }
	// 计算指定 index 的标题的透明度
    double _titleOpacity(int index) {
      return 1.0 - _selectedIndexDelta(index) * tColumnToRow * 0.5;
    }
	// 计算指定 index 标题的尺寸
    double _titleScale(int index) {
      return 1.0 - _selectedIndexDelta(index) * tColumnToRow * 0.15;
    }


    final List<Widget> children = List<Widget>.from(sectionCards);

	// 向 Children 中加入标题实体
    for (int index = 0; index < sections.length; index++) {
      final Section section = sections[index];
      children.add(LayoutId(
        id: 'title$index',
        child: SectionTitle(
          section: section,
          scale: _titleScale(index),
          opacity: _titleOpacity(index),
        ),
      ));
    }
    
    // 向 Children 中加入指示器实体
    for (int index = 0; index < sections.length; index++) {
      children.add(LayoutId(
        id: 'indicator$index',
        child: SectionIndicator(
          opacity: _indicatorOpacity(index),
        ),
      ));
    }
	
	// 利用上边定义的_AllSectionsLayout类完成布局
    return CustomMultiChildLayout(
      delegate: _AllSectionsLayout(
        translation: Alignment((selectedIndex.value - sectionIndex) * 2.0 - 1.0, -1.0),
        tColumnToRow: tColumnToRow,
        tCollapsed: tCollapsed,
        cardCount: sections.length,
        selectedIndex: selectedIndex.value,
      ),
      children: children,
    );
  }

  @override
  Widget build(BuildContext context) {
  	// 使用LayoutBuilder 是为了获取约束，进而转化为大小来完成布局
    return LayoutBuilder(builder: _build);
  }
}
```

##  定义一个新的SCrollPhysics(不懂，略过)
```dart
class _SnappingScrollPhysics extends ClampingScrollPhysics {
  const _SnappingScrollPhysics({
    ScrollPhysics parent,
    @required this.midScrollOffset,
  }) : assert(midScrollOffset != null),
       super(parent: parent);

  final double midScrollOffset;

  @override
  _SnappingScrollPhysics applyTo(ScrollPhysics ancestor) {
    return _SnappingScrollPhysics(parent: buildParent(ancestor), midScrollOffset: midScrollOffset);
  }

  Simulation _toMidScrollOffsetSimulation(double offset, double dragVelocity) {
    final double velocity = math.max(dragVelocity, minFlingVelocity);
    return ScrollSpringSimulation(spring, offset, midScrollOffset, velocity, tolerance: tolerance);
  }

  Simulation _toZeroScrollOffsetSimulation(double offset, double dragVelocity) {
    final double velocity = math.max(dragVelocity, minFlingVelocity);
    return ScrollSpringSimulation(spring, offset, 0.0, velocity, tolerance: tolerance);
  }

  @override
  Simulation createBallisticSimulation(ScrollMetrics position, double dragVelocity) {
    final Simulation simulation = super.createBallisticSimulation(position, dragVelocity);
    final double offset = position.pixels;

    if (simulation != null) {
    
      // The drag ended with sufficient velocity to 
      // trigger creating a simulation.
      
      // If the simulation is headed up towards midScrollOffset 
      // but will not reach it,then snap it there. 
      
      // Similarly if the simulation is headed down past midScrollOffset 	   
      // but will not reach zero, then snap it to zero.
      
      // 当拖拽以足够的速度结束是，会触发创建一个模拟。
      // 如果模拟是朝着midScrollOffset进行，但是没有达到它
      // 那么就会慢慢达到（坍缩到？）它（指midScrollOffset）
      // 类似地，达到0.0也会有这样的效果
      // 整个效果类似于Pageview的达到固定页面而不是停留在中间的效果
      final double simulationEnd = simulation.x(double.infinity);
      if (simulationEnd >= midScrollOffset)
        return simulation;
      if (dragVelocity > 0.0)
        return _toMidScrollOffsetSimulation(offset, dragVelocity);
      if (dragVelocity < 0.0)
        return _toZeroScrollOffsetSimulation(offset, dragVelocity);
    } else {
      // The user ended the drag with little or no velocity. If they
      // didn't leave the offset above midScrollOffset, then
      // snap to midScrollOffset if they're more than halfway there,
      // otherwise snap to zero.
      
      // 简言之：当拖拽结束时的速度较小且没有达到 midScrollOffset 的一半，
      // 则回到原点，否则向目标位置移动
      
      final double snapThreshold = midScrollOffset / 2.0;
      if (offset >= snapThreshold && offset < midScrollOffset)
        return _toMidScrollOffsetSimulation(offset, dragVelocity);
      if (offset > 0.0 && offset < snapThreshold)
        return _toZeroScrollOffsetSimulation(offset, dragVelocity);
    }
    return simulation;
  }
}
```

## Widget && State（200行的State了解一下？）
```dart
class AnimationDemoHome extends StatefulWidget {
  const AnimationDemoHome({ Key key }) : super(key: key);

  static const String routeName = '/animation';

  @override
  _AnimationDemoHomeState createState() => _AnimationDemoHomeState();
}

class _AnimationDemoHomeState extends State<AnimationDemoHome> {
  final ScrollController _scrollController = ScrollController();
  final PageController _headingPageController = PageController();
  final PageController _detailsPageController = PageController();
  // 不处于行状态时SliverPersistentHeading 的PagView不允许滑动  
  ScrollPhysics _headingScrollPhysics = const NeverScrollableScrollPhysics();
  ValueNotifier<double> selectedIndex = ValueNotifier<double>(0.0);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: _kAppBackgroundColor,
      body: Builder(
        // Insert an element so that _buildBody can find the PrimaryScrollController.
        builder: _buildBody,
      ),
    );
  }

  // 处理返回事件。若处于行状态，则动画到列状态    
  void _handleBackButton(double midScrollOffset) {
    if (_scrollController.offset >= midScrollOffset)
      _scrollController.animateTo(0.0, curve: _kScrollCurve, duration: _kScrollDuration);
    else
      Navigator.maybePop(context);
  }

  // 当且仅当滑动到midScrollOffset时启用 SliverPersistentHeading 的PagView滑动  
  // 通过设置  _headingScrollPhysics = physics 来解决
  bool _handleScrollNotification(ScrollNotification notification, double midScrollOffset)   {
    if (notification.depth == 0 && notification is ScrollUpdateNotification) {
      final ScrollPhysics physics = _scrollController.position.pixels >= midScrollOffset
       ? const PageScrollPhysics()
       : const NeverScrollableScrollPhysics();
      if (physics != _headingScrollPhysics) {
        setState(() {
          _headingScrollPhysics = physics;
        });
      }
    }
    return false;
  }

  void _maybeScroll(double midScrollOffset, int pageIndex, double xOffset) {
    if (_scrollController.offset < midScrollOffset) {
      // Scroll the overall list to the point where only one section card shows.
      // At the same time scroll the PageViews to the page at pageIndex.
      _headingPageController.animateToPage(pageIndex, curve: _kScrollCurve, duration: _kScrollDuration);
      _scrollController.animateTo(midScrollOffset, curve: _kScrollCurve, duration: _kScrollDuration);
    } else {
      // One one section card is showing: scroll one page forward or back.
      final double centerX = _headingPageController.position.viewportDimension / 2.0;
      final int newPageIndex = xOffset > centerX ? pageIndex + 1 : pageIndex - 1;
      _headingPageController.animateToPage(newPageIndex, curve: _kScrollCurve, duration: _kScrollDuration);
    }
  }

  // 当PageView 滑动时，通知 selectedIndex 
  bool _handlePageNotification(ScrollNotification notification, PageController leader, PageController follower) {
    if (notification.depth == 0 && notification is ScrollUpdateNotification) {
      selectedIndex.value = leader.page;
      if (follower.page != leader.page)
        follower.position.jumpToWithoutSettling(leader.position.pixels); // ignore: deprecated_member_use
    }
    return false;
  }

  Iterable<Widget> _detailItemsFor(Section section) {
    final Iterable<Widget> detailItems = section.details.map<Widget>((SectionDetail detail) {
      return SectionDetailView(detail: detail);
    });
    return ListTile.divideTiles(context: context, tiles: detailItems);
  }

  // 构建Header 
  Iterable<Widget> _allHeadingItems(double maxHeight, double midScrollOffset) {
    final List<Widget> sectionCards = <Widget>[];
    for (int index = 0; index < allSections.length; index++) {
      sectionCards.add(LayoutId(
        id: 'card$index',
        child: GestureDetector(
          behavior: HitTestBehavior.opaque,
          child: SectionCard(section: allSections[index]),
          onTapUp: (TapUpDetails details) {
            final double xOffset = details.globalPosition.dx;
            setState(() {
              _maybeScroll(midScrollOffset, index, xOffset);
            });
          },
        ),
      ));
    }

    final List<Widget> headings = <Widget>[];
    for (int index = 0; index < allSections.length; index++) {
      headings.add(Container(
          color: _kAppBackgroundColor,
          child: ClipRect(
            child: _AllSectionsView(
              sectionIndex: index,
              sections: allSections,
              selectedIndex: selectedIndex,
              minHeight: _kAppBarMinHeight,
              midHeight: _kAppBarMidHeight,
              maxHeight: maxHeight,
              sectionCards: sectionCards,
            ),
          ),
        )
      );
    }
    return headings;
  }

  Widget _buildBody(BuildContext context) {
    final MediaQueryData mediaQueryData = MediaQuery.of(context);
    final double statusBarHeight = mediaQueryData.padding.top;
    final double screenHeight = mediaQueryData.size.height;
    final double appBarMaxHeight = screenHeight - statusBarHeight;

    // The scroll offset that reveals the appBarMidHeight appbar.
    final double appBarMidScrollOffset = statusBarHeight + appBarMaxHeight - _kAppBarMidHeight;

    return SizedBox.expand(
      child: Stack(
        children: <Widget>[
          NotificationListener<ScrollNotification>(
            onNotification: (ScrollNotification notification) {
              return _handleScrollNotification(notification, appBarMidScrollOffset);
            },
            child: CustomScrollView(
              controller: _scrollController,
              physics: _SnappingScrollPhysics(midScrollOffset: appBarMidScrollOffset),
              slivers: <Widget>[
                // Start out below the status bar, gradually move to the top of the screen.
                _StatusBarPaddingSliver(
                  maxHeight: statusBarHeight,
                  scrollFactor: 7.0,
                ),
                // Section Headings
                SliverPersistentHeader(
                  pinned: true,
                  delegate: _SliverAppBarDelegate(
                    minHeight: _kAppBarMinHeight,
                    maxHeight: appBarMaxHeight,
                    child: NotificationListener<ScrollNotification>(
                      onNotification: (ScrollNotification notification) {
                        return _handlePageNotification(notification, _headingPageController, _detailsPageController);
                      },
                      child: PageView(
                        physics: _headingScrollPhysics,
                        controller: _headingPageController,
                        children: _allHeadingItems(appBarMaxHeight, appBarMidScrollOffset),
                      ),
                    ),
                  ),
                ),
                // Details
                SliverToBoxAdapter(
                  child: SizedBox(
                    height: 610.0,
                    child: NotificationListener<ScrollNotification>(
                      onNotification: (ScrollNotification notification) {
                        return _handlePageNotification(notification, _detailsPageController, _headingPageController);
                      },
                      child: PageView(
                        controller: _detailsPageController,
                        children: allSections.map<Widget>((Section section) {
                          return Column(
                            crossAxisAlignment: CrossAxisAlignment.stretch,
                            children: _detailItemsFor(section).toList(),
                          );
                        }).toList(),
                      ),
                    ),
                  ),
                ),
              ],
            ),
          ),
          // 返回图标  
          Positioned(
            top: statusBarHeight,
            left: 0.0,
            child: IconTheme(
              data: const IconThemeData(color: Colors.white),
              child: SafeArea(
                top: false,
                bottom: false,
                child: IconButton(
                  icon: const BackButtonIcon(),
                  tooltip: 'Back',
                  onPressed: () {
                    _handleBackButton(appBarMidScrollOffset);
                  },
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

# 后话

菜鸡想魔(zuo)改(si)一下代码由纵向列表变成横向列表？<del>不晓得能</del>不能实现。
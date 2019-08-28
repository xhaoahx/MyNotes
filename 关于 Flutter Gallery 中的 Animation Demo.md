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
// Arrange the section titles, indicators, and cards. The cards are only included when
// the layout is transitioning between vertical and horizontal. Once the layout is
// horizontal the cards are laid out by a PageView.
//
// The layout of the section cards, titles, and indicators is defined by the
// two 0.0-1.0 "t" parameters, both of which are based on the layout's height:
// - tColumnToRow
//   0.0 when height is maxHeight and the layout is a column
//   1.0 when the height is midHeight and the layout is a row
// - tCollapsed
//   0.0 when height is midHeight and the layout is a row
//   1.0 when height is minHeight and the layout is a (still) row
//
// minHeight < midHeight < maxHeight
//
// The general approach here is to compute the column layout and row size
// and position of each element and then interpolate between them using
// tColumnToRow. Once tColumnToRow reaches 1.0, the layout changes are
// defined by tCollapsed. As tCollapsed increases the titles spread out
// until only one title is visible and the indicators cluster together
// until they're all visible.

// 安排板块标题，指示器，卡片。只有在布局从垂直变化到水平的时候，卡片被加入。
// 一旦布局变成水平，卡片将用PageView展示（卡片指带有详情的部分）
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

  final Alignment translation;
  final double tColumnToRow;
  final double tCollapsed;
  final int cardCount;
  final double selectedIndex;

  Rect _interpolateRect(Rect begin, Rect end) {
    return Rect.lerp(begin, end, tColumnToRow);
  }

  Offset _interpolatePoint(Offset begin, Offset end) {
    return Offset.lerp(begin, end, tColumnToRow);
  }

  @override
  void performLayout(Size size) {
    //final double columnCardX = size.width / 5.0;
    final double columnCardX = size.width / 4.0;
    final double columnCardWidth = size.width - columnCardX;
    final double columnCardHeight = size.height / cardCount;
    final double rowCardWidth = size.width;
    final Offset offset = translation.alongSize(size);
    double columnCardY = 0.0;
    double rowCardX = -(selectedIndex * rowCardWidth);

    // When tCollapsed > 0 the titles spread apart
    final double columnTitleX = size.width / 10.0;
    final double rowTitleWidth = size.width * ((1 + tCollapsed) / 2.25);
    double rowTitleX = (size.width - rowTitleWidth) / 2.0 - selectedIndex * rowTitleWidth;

    // When tCollapsed > 0, the indicators move closer together
    //final double rowIndicatorWidth = 48.0 + (1.0 - tCollapsed) * (rowTitleWidth - 48.0);
    const double paddedSectionIndicatorWidth = kSectionIndicatorWidth + 8.0;
    final double rowIndicatorWidth = paddedSectionIndicatorWidth +
      (1.0 - tCollapsed) * (rowTitleWidth - paddedSectionIndicatorWidth);
    double rowIndicatorX = (size.width - rowIndicatorWidth) / 2.0 - selectedIndex * rowIndicatorWidth;

    // Compute the size and origin of each card, title, and indicator for the maxHeight
    // "column" layout, and the midHeight "row" layout. The actual layout is just the
    // interpolated value between the column and row layouts for t.
    for (int index = 0; index < cardCount; index++) {

      // Layout the card for index.
      final Rect columnCardRect = Rect.fromLTWH(columnCardX, columnCardY, columnCardWidth, columnCardHeight);
      final Rect rowCardRect = Rect.fromLTWH(rowCardX, 0.0, rowCardWidth, size.height);
      final Rect cardRect = _interpolateRect(columnCardRect, rowCardRect).shift(offset);
      final String cardId = 'card$index';
      if (hasChild(cardId)) {
        layoutChild(cardId, BoxConstraints.tight(cardRect.size));
        positionChild(cardId, cardRect.topLeft);
      }

      // Layout the title for index.
      final Size titleSize = layoutChild('title$index', BoxConstraints.loose(cardRect.size));
      final double columnTitleY = columnCardRect.centerLeft.dy - titleSize.height / 2.0;
      final double rowTitleY = rowCardRect.centerLeft.dy - titleSize.height / 2.0;
      final double centeredRowTitleX = rowTitleX + (rowTitleWidth - titleSize.width) / 2.0;
      final Offset columnTitleOrigin = Offset(columnTitleX, columnTitleY);
      final Offset rowTitleOrigin = Offset(centeredRowTitleX, rowTitleY);
      final Offset titleOrigin = _interpolatePoint(columnTitleOrigin, rowTitleOrigin);
      positionChild('title$index', titleOrigin + offset);

      // Layout the selection indicator for index.
      final Size indicatorSize = layoutChild('indicator$index', BoxConstraints.loose(cardRect.size));
      final double columnIndicatorX = cardRect.centerRight.dx - indicatorSize.width - 16.0;
      final double columnIndicatorY = cardRect.bottomRight.dy - indicatorSize.height - 16.0;
      final Offset columnIndicatorOrigin = Offset(columnIndicatorX, columnIndicatorY);
      final Rect titleRect = Rect.fromPoints(titleOrigin, titleSize.bottomRight(titleOrigin));
      final double centeredRowIndicatorX = rowIndicatorX + (rowIndicatorWidth - indicatorSize.width) / 2.0;
      final double rowIndicatorY = titleRect.bottomCenter.dy + 16.0;
      final Offset rowIndicatorOrigin = Offset(centeredRowIndicatorX, rowIndicatorY);
      final Offset indicatorOrigin = _interpolatePoint(columnIndicatorOrigin, rowIndicatorOrigin);
      positionChild('indicator$index', indicatorOrigin + offset);

      columnCardY += columnCardHeight;
      rowCardX += rowCardWidth;
      rowTitleX += rowTitleWidth;
      rowIndicatorX += rowIndicatorWidth;
    }
  }

  @override
  bool shouldRelayout(_AllSectionsLayout oldDelegate) {
    return tColumnToRow != oldDelegate.tColumnToRow
      || cardCount != oldDelegate.cardCount
      || selectedIndex != oldDelegate.selectedIndex;
  }
}
```


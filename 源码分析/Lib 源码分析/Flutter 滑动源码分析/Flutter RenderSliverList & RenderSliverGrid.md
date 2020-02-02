# Flutter RenderSliverList & RenderSliverGrid

## RenderSilverList

```dart
/// 一个能将所有孩子沿着主轴线性排列的滑块
///
/// [RenderSliverList] 通过 “dead reckoning” 来确定它的滚动偏移量，因为在滑块可见部分之外的孩子没有材料化，
/// 这意味着[RenderSliverList]不能知道它们的主轴范围。相反，新材料化的子元素被放置在现有子元素的旁边。如果这
/// 种推算导致了逻辑上的不一致(例如，试图将孩子放置在一个滚动偏移量上而不是零)，[RenderSliverList]会生成一个
/// [SliverGeometry.scrollOffsetCorrection]以恢复一致性。
class RenderSliverList extends RenderSliverMultiBoxAdaptor {
  RenderSliverList({
    @required RenderSliverBoxChildManager childManager,
  }) : super(childManager: childManager);

  @override
  void performLayout() {
    childManager.didStartLayout();
    childManager.setDidUnderflow(false);

    /// 由于 constraints.scrollOffset 代表了第一处在视窗内部显示的位置（视觉上应该开始绘制的位置（偏移）），
    /// 且 constraints.cacheOrigin 总是一个负值，代表缓存起点的偏移量
    /// 因此 constraints.scrollOffset + constraints.cacheOrigin 代表了实际上应该开始绘制的位置（偏移） 
    final double scrollOffset = constraints.scrollOffset + constraints.cacheOrigin;
    /// 剩余的可用绘制距离，也就是缓存距离加上绘制距离
    /// 即 math.max(0.0, remainingCacheExtent + cacheExtentCorrection)
    final double remainingExtent = constraints.remainingCacheExtent;
    /// 绘制结束的偏移
    final double targetEndScrollOffset = scrollOffset + remainingExtent;
    /// 直接把自身的约束作为孩子的约束，即主轴约束是无穷的，交叉轴约束是交叉轴的宽度
    final BoxConstraints childConstraints = constraints.asBoxConstraints();
      
    /// 需要被回收的下标
    int leadingGarbage = 0;
    int trailingGarbage = 0;
    bool reachedEnd = false;

    // 这个算法原则上是直线进行的：找到第一个覆盖了给定滚动偏移的孩子，如果有需要的话，在列表顶部创建更多的孩
    // 子；随后，遍历列表的同时更新并且布局出每一个孩子，并且在有需要的时候在列表的末尾添加更多的孩子（如果有足
    // 够多的孩子来覆盖整个视窗的话）
    //
    // 它被一个小问题复杂化了，这个小问题是，每当更新或创建一个子节点时，可能会删除一些尚未被布局的子节点，使
    // 列表处于不一致的状态，并要求重新创建丢失的节点
    //
    // 为了保持这种混乱的可处理性，这个算法从当前的第一个子节点(如果有的话)开始，然后向上和/或向下移动，这样可
    // 能被移除的节点总是位于已经布置的节点的边缘。
    
    // 确保有至少一个孩子来开始
    if (firstChild == null) {
      /// addInitialChild() 会添加第一个 child，其默认下标为 0，并且给，这个调用会把 
      /// firstChild（也就是我们刚才添加的第一个 child）.parentData.layoutOffset 设置成 0.0
      
      /// 如果添加失败，即没有任何的孩子
      if (!addInitialChild()) {
        // There are no children.
        geometry = SliverGeometry.zero;
        childManager.didFinishLayout();
        return;
      }
    }

    // 到这里，我们至少有一个孩子

    // 这些变量跟踪我们列出的孩子的范围。在这个范围内，子元素有连续的索引。在这个范围之外，孩子有可能在没有通知
    // 的情况下被移除。
    RenderBox leadingChildWithLayout, trailingChildWithLayout;

    // 查找位于 scrollOffset 或之前的最后一个子元素
    RenderBox earliestUsefulChild = firstChild;
    for (double earliestScrollOffset = childScrollOffset(earliestUsefulChild); // 0.0
        /// 循环结束的条件 earliestScrollOffset <= scrollOffset
        /// 也就是说找到一个 earliestUsefulChild，它的 layoutOffsset（也就是它的布局位置距离滚动偏移零点的
        /// 距离）比给定的 scrollOffset 要小，也就是意味着，找到 earliestUsefulChild 是第一个位于  
        /// scrollOffset 之前（或者恰好）的 child
        earliestScrollOffset > scrollOffset;
        /// 每次循环都会更新一次 earliestScrollOffset
        earliestScrollOffset = childScrollOffset(earliestUsefulChild)
    ) {
      // 尝试布局 firstChild 之前的（也就是下标小于 index（默认为 0））一个 child（如果有的话）
      earliestUsefulChild = insertAndLayoutLeadingChild(childConstraints, parentUsesSize: true);

      /// 如果 firstChild 之前没有新的 Child 了
      if (earliestUsefulChild == null) {
        final SliverMultiBoxAdaptorParentData childParentData = 
            firstChild.parentData as SliverMultiBoxAdaptorParentData;
        childParentData.layoutOffset = 0.0;

        if (scrollOffset == 0.0) {
          // [insertAndLayoutLeadingChild] 只对 firstChild 之前的孩子进行布局。在这个情况下，我们必须手动
          // 对 firstChild 进行布局
          firstChild.layout(childConstraints, parentUsesSize: true);
          earliestUsefulChild = firstChild;
          leadingChildWithLayout = earliestUsefulChild;
          trailingChildWithLayout ??= earliestUsefulChild;
          break;
        } else {
          // 在到达滚动偏移之前，我们用完了所有的子元素。
          // 我们必须通知我们的父节点此滑块不能正确地完成协议，也就是所我们需要一次修正
          // （则通常会导致对 scrolloffset 进行一次修正，并且布局流程的重新进行）
          geometry = SliverGeometry(
            scrollOffsetCorrection: -scrollOffset,
          );
          return;
        }
      }

      /// 第一个孩子的滚动偏移
      /// 就是
      final double firstChildScrollOffset = earliestScrollOffset - paintExtentOf(firstChild);
      // firstChildScrollOffset 可能会有浮点数误差
      if (firstChildScrollOffset < -precisionErrorTolerance) {
        // The first child doesn't fit within the viewport (underflow) and
        // there may be additional children above it. Find the real first child
        // and then correct the scroll position so that there's room for all and
        // so that the trailing edge of the original firstChild appears where it
        // was before the scroll offset correction.
        // TODO(hansmuller): do this work incrementally, instead of all at once,
        // i.e. find a way to avoid visiting ALL of the children whose offset
        // is < 0 before returning for the scroll correction.
        double correction = 0.0;
        while (earliestUsefulChild != null) {
          assert(firstChild == earliestUsefulChild);
          correction += paintExtentOf(firstChild);
          earliestUsefulChild = insertAndLayoutLeadingChild(childConstraints, parentUsesSize: true);
        }
        geometry = SliverGeometry(
          scrollOffsetCorrection: correction - earliestScrollOffset,
        );
        final SliverMultiBoxAdaptorParentData childParentData = 
            firstChild.parentData as SliverMultiBoxAdaptorParentData;
        childParentData.layoutOffset = 0.0;
        return;
      } 

      final SliverMultiBoxAdaptorParentData childParentData = 
          earliestUsefulChild.parentData as SliverMultiBoxAdaptorParentData;
      childParentData.layoutOffset = firstChildScrollOffset;
      assert(earliestUsefulChild == firstChild);
      leadingChildWithLayout = earliestUsefulChild;
      trailingChildWithLayout ??= earliestUsefulChild;
    } // for

    // 到达这个位置，earliestUsefulChild 就是 firstChild，并且它的 scrollOffset 恰好位于 scrollOffset 或
    // 者处于 scrollOffset 之前。并且 leadingChildWithLayout 和 trailingChildWithLayout 都为 null 或者 
    // 包含了一系列的渲染盒子，其中第一个与前面的 child 相同，最后一个要么位于滚动偏移量处，要么位于偏移量之  
    // 后。
    assert(earliestUsefulChild == firstChild);
    assert(childScrollOffset(earliestUsefulChild) <= scrollOffset);

    // 进行到这里，我们至少布局了一个孩子
    if (leadingChildWithLayout == null) {
      earliestUsefulChild.layout(childConstraints, parentUsesSize: true);
      leadingChildWithLayout = earliestUsefulChild;
      trailingChildWithLayout = earliestUsefulChild;
    }

    // 这里 earliestUsefulChild 仍然是第一个孩子，它的 scrollOffset 在外面实际的 scrollOffset 之前，并且它
    // 已经被布局了，并且他将被作为 leadingChildWithLayout。除了它，可能还有一些其他的孩子的布局也已经完成。
    bool inLayoutRange = true;
    RenderBox child = earliestUsefulChild;
    int index = indexOf(child);// 0 ？
      
    /// 最终布局的偏移量
    double endScrollOffset = childScrollOffset(child) + paintExtentOf(child);
      
    // 此函数被两处使用
    bool advance() { // 如果前进则返回 ture，在没有更多的孩子的时候返回 false
      assert(child != null);
      if (child == trailingChildWithLayout)
        inLayoutRange = false;
      child = childAfter(child);
      /// 如果没有下一个孩子
      if (child == null)
        inLayoutRange = false;
      index += 1;
      if (!inLayoutRange) {
        if (child == null || indexOf(child) != index) {
          // 我们丢失了一个孩子，如果可能的话，将它重新插入
          child = insertAndLayoutChild(
            childConstraints,
            after: trailingChildWithLayout,
            parentUsesSize: true,
          );
          /// 我们已经用完了所有的孩子
          if (child == null) {
            return false;
          }
        } else {
          // 布局新的到的 child
          child.layout(childConstraints, parentUsesSize: true);
        }
        trailingChildWithLayout = child;
      }
      assert(child != null);
        
      final SliverMultiBoxAdaptorParentData childParentData = 
          child.parentData as SliverMultiBoxAdaptorParentData;
      childParentData.layoutOffset = endScrollOffset;
      assert(childParentData.index == index);
      /// 这里，每布局一个孩子，就加上它的 paintExtent，也就是它在主轴上占用的距离
      endScrollOffset = childScrollOffset(child) + paintExtentOf(child);
      return true;
    }

    // 找到第一个 scrolloffset 之后的 child
    while (endScrollOffset < scrollOffset) {
      /// scrollOffset 之前的所有孩子都会被视作垃圾而被清理
      leadingGarbage += 1;
      if (!advance()) {
        // 我们要确保保留最后一个子元素，这样才能知道最终的滚动偏移量
        collectGarbage(leadingGarbage - 1, 0);
        /// 在这种情况下，我们只有一个孩子
        assert(firstChild == lastChild);
        final double extent = childScrollOffset(lastChild) + paintExtentOf(lastChild);
        geometry = SliverGeometry(
          scrollExtent: extent,
          paintExtent: 0.0,
          maxPaintExtent: extent,
        );
        return;
      }
    }

    // 找到最大绘制偏移之后的第一个孩子
    while (endScrollOffset < targetEndScrollOffset) {
      /// 在此之前如果用完了所有的孩子，意味着布局已经完成
      if (!advance()) {
        reachedEnd = true;
        break;
      }
    }

    // 剩余的所有的孩子将被视为垃圾而被清理
    if (child != null) {
      child = childAfter(child);
      while (child != null) {
        trailingGarbage += 1;
        child = childAfter(child);
      }
    }

    // 清理所有的垃圾
    collectGarbage(leadingGarbage, trailingGarbage);

    double estimatedMaxScrollOffset;
    if (reachedEnd) {
      estimatedMaxScrollOffset = endScrollOffset;
    } else {
      estimatedMaxScrollOffset = childManager.estimateMaxScrollOffset(
        constraints,
        firstIndex: indexOf(firstChild),
        lastIndex: indexOf(lastChild),
        leadingScrollOffset: childScrollOffset(firstChild),
        trailingScrollOffset: endScrollOffset,
      );
      assert(estimatedMaxScrollOffset >= endScrollOffset - childScrollOffset(firstChild));
    }
    final double paintExtent = calculatePaintOffset(
      constraints,
      from: childScrollOffset(firstChild),
      to: endScrollOffset,
    );
    final double cacheExtent = calculateCacheOffset(
      constraints,
      from: childScrollOffset(firstChild),
      to: endScrollOffset,
    );
    final double targetEndScrollOffsetForPaint = 
        constraints.scrollOffset + constraints.remainingPaintExtent;
    geometry = SliverGeometry(
      scrollExtent: estimatedMaxScrollOffset,
      paintExtent: paintExtent,
      cacheExtent: cacheExtent,
      maxPaintExtent: estimatedMaxScrollOffset,
      // 保守，以避免剪辑滚动期间的闪烁
      hasVisualOverflow: 
        endScrollOffset > targetEndScrollOffsetForPaint || constraints.scrollOffset > 0.0,
    );

    // We may have started the layout while scrolled to the end, which would not
    // expose a new child.
    if (estimatedMaxScrollOffset == endScrollOffset)
      childManager.setDidUnderflow(true);
    childManager.didFinishLayout();
  }
}

```



## RenderSliverGrid

```dart
/// A sliver that places multiple box children in a two dimensional arrangement.
///
/// [RenderSliverGrid] 根据 [gridDelegate] 来任意地放置它的孩子，并且每一个孩子的大小都被 [gridDelegate] 
/// 所限定
class RenderSliverGrid extends RenderSliverMultiBoxAdaptor {
  /// Creates a sliver that contains multiple box children that whose size and
  /// position are determined by a delegate.
  ///
  /// The [childManager] and [gridDelegate] arguments must not be null.
  RenderSliverGrid({
    @required RenderSliverBoxChildManager childManager,
    @required SliverGridDelegate gridDelegate,
  }) : _gridDelegate = gridDelegate,
       super(childManager: childManager);

  @override
  void setupParentData(RenderObject child) {
    if (child.parentData is! SliverGridParentData)
      child.parentData = SliverGridParentData();
  }

  SliverGridDelegate get gridDelegate => _gridDelegate;
  SliverGridDelegate _gridDelegate;
  set gridDelegate(SliverGridDelegate value) {
    assert(value != null);
    if (_gridDelegate == value)
      return;
    if (value.runtimeType != _gridDelegate.runtimeType ||
        value.shouldRelayout(_gridDelegate))
      markNeedsLayout();
    _gridDelegate = value;
  }

  @override
  double childCrossAxisPosition(RenderBox child) {
    final SliverGridParentData childParentData = child.parentData as SliverGridParentData;
    return childParentData.crossAxisOffset;
  }

  @override
  void performLayout() {
    childManager.didStartLayout();
    childManager.setDidUnderflow(false);

    final double scrollOffset = constraints.scrollOffset + constraints.cacheOrigin;
    assert(scrollOffset >= 0.0);
    final double remainingExtent = constraints.remainingCacheExtent;
    assert(remainingExtent >= 0.0);
    final double targetEndScrollOffset = scrollOffset + remainingExtent;

    final SliverGridLayout layout = _gridDelegate.getLayout(constraints);

    final int firstIndex = layout.getMinChildIndexForScrollOffset(scrollOffset);
    final int targetLastIndex = targetEndScrollOffset.isFinite ?
      layout.getMaxChildIndexForScrollOffset(targetEndScrollOffset) : null;

    if (firstChild != null) {
      final int oldFirstIndex = indexOf(firstChild);
      final int oldLastIndex = indexOf(lastChild);
      /// 列表头部的垃圾数量
      final int leadingGarbage = (firstIndex - oldFirstIndex).clamp(0, childCount) as int;
      final int trailingGarbage = targetLastIndex == null
        ? 0
        : ((oldLastIndex - targetLastIndex).clamp(0, childCount) as int);
      collectGarbage(leadingGarbage, trailingGarbage);
    } else {
      collectGarbage(0, 0);
    }

    final SliverGridGeometry firstChildGridGeometry = layout.getGeometryForChildIndex(firstIndex);
    final double leadingScrollOffset = firstChildGridGeometry.scrollOffset;
    double trailingScrollOffset = firstChildGridGeometry.trailingScrollOffset;

    if (firstChild == null) {
      if (!addInitialChild(index: firstIndex, layoutOffset: firstChildGridGeometry.scrollOffset)) {
        // There are either no children, or we are past the end of all our children.
        final double max = layout.computeMaxScrollOffset(childManager.childCount);
        geometry = SliverGeometry(
          scrollExtent: max,
          maxPaintExtent: max,
        );
        childManager.didFinishLayout();
        return;
      }
    }

    RenderBox trailingChildWithLayout;

    for (int index = indexOf(firstChild) - 1; index >= firstIndex; --index) {
      final SliverGridGeometry gridGeometry = layout.getGeometryForChildIndex(index);
      final RenderBox child = insertAndLayoutLeadingChild(
        gridGeometry.getBoxConstraints(constraints),
      );
      final SliverGridParentData childParentData = child.parentData as SliverGridParentData;
      childParentData.layoutOffset = gridGeometry.scrollOffset;
      childParentData.crossAxisOffset = gridGeometry.crossAxisOffset;
      assert(childParentData.index == index);
      trailingChildWithLayout ??= child;
      trailingScrollOffset = math.max(trailingScrollOffset, gridGeometry.trailingScrollOffset);
    }

    if (trailingChildWithLayout == null) {
      firstChild.layout(firstChildGridGeometry.getBoxConstraints(constraints));
      final SliverGridParentData childParentData = firstChild.parentData as SliverGridParentData;
      childParentData.layoutOffset = firstChildGridGeometry.scrollOffset;
      childParentData.crossAxisOffset = firstChildGridGeometry.crossAxisOffset;
      trailingChildWithLayout = firstChild;
    }

    for (int index = indexOf(trailingChildWithLayout) + 1; 
         targetLastIndex == null || index <= targetLastIndex; 
         ++index
    ) {
      final SliverGridGeometry gridGeometry = layout.getGeometryForChildIndex(index);
      final BoxConstraints childConstraints = gridGeometry.getBoxConstraints(constraints);
      RenderBox child = childAfter(trailingChildWithLayout);
      if (child == null || indexOf(child) != index) {
        child = insertAndLayoutChild(childConstraints, after: trailingChildWithLayout);
        if (child == null) {
          // We have run out of children.
          break;
        }
      } else {
        child.layout(childConstraints);
      }
      trailingChildWithLayout = child;
      assert(child != null);
      final SliverGridParentData childParentData = child.parentData as SliverGridParentData;
      childParentData.layoutOffset = gridGeometry.scrollOffset;
      childParentData.crossAxisOffset = gridGeometry.crossAxisOffset;
      assert(childParentData.index == index);
      trailingScrollOffset = math.max(trailingScrollOffset, gridGeometry.trailingScrollOffset);
    }

    final int lastIndex = indexOf(lastChild);

    assert(childScrollOffset(firstChild) <= scrollOffset);
    assert(debugAssertChildListIsNonEmptyAndContiguous());
    assert(indexOf(firstChild) == firstIndex);
    assert(targetLastIndex == null || lastIndex <= targetLastIndex);

    final double estimatedTotalExtent = childManager.estimateMaxScrollOffset(
      constraints,
      firstIndex: firstIndex,
      lastIndex: lastIndex,
      leadingScrollOffset: leadingScrollOffset,
      trailingScrollOffset: trailingScrollOffset,
    );

    final double paintExtent = calculatePaintOffset(
      constraints,
      from: leadingScrollOffset,
      to: trailingScrollOffset,
    );
    final double cacheExtent = calculateCacheOffset(
      constraints,
      from: leadingScrollOffset,
      to: trailingScrollOffset,
    );

    geometry = SliverGeometry(
      scrollExtent: estimatedTotalExtent,
      paintExtent: paintExtent,
      maxPaintExtent: estimatedTotalExtent,
      cacheExtent: cacheExtent,
      // Conservative to avoid complexity.
      hasVisualOverflow: true,
    );

    // We may have started the layout while scrolled to the end, which
    // would not expose a new child.
    if (estimatedTotalExtent == trailingScrollOffset)
      childManager.setDidUnderflow(true);
    childManager.didFinishLayout();
  }
}

```


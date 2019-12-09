# Flutter TextPainter 源码分析

```dart
/// TextPainter 其实是对 ParagraphBuilder 和 Paragraph 的封装操作
/// 当修改了其属性的，会使用和布局绘制类似的延迟处理标记机制来处理变化
/// 在 RenderObject 中使用的时候，可以在 performLayout 方法中使用 textPainter.layout，并利用其大小信息
/// 然后在 paint 方法中调用 textPainter.paint 方法

/// 一个能将 [TextSpan] 树绘制到 [Canvas] 上的对象
///
/// 使用 [TextPainter] 的步骤如下：
///
/// 1. 建立一棵 [TextSpan] 树并将它传递给 [TextPainter] 构造函数
///
/// 2. 调用 [layout] 来完成布局工作
///
/// 3. 调用 [paint] 来绘制到 canvas 上
///
/// 如果正在绘制文本的区域的宽度发生变化，返回步骤2。如果要绘制的文本发生变化，返回步骤1。
///
class TextPainter {
  TextPainter({
    InlineSpan text,
    TextAlign textAlign = TextAlign.start,
    TextDirection textDirection,
    double textScaleFactor = 1.0,
    int maxLines,
    String ellipsis,
    Locale locale,
    StrutStyle strutStyle,
    TextWidthBasis textWidthBasis = TextWidthBasis.parent,
  }) : _text = text,
       _textAlign = textAlign,
       _textDirection = textDirection,
       _textScaleFactor = textScaleFactor,
       _maxLines = maxLines,
       _ellipsis = ellipsis,
       _locale = locale,
       _strutStyle = strutStyle,
       _textWidthBasis = textWidthBasis;

  ui.Paragraph _paragraph;
  bool _needsLayout = true;

  /// 标记布局信息为 dirty 并清空已有的缓存（指 _paragraph）
  ///
  /// 使用此方法通知文本绘制器在引擎布局发生变化时进行重新布局
  /// 在大多数情况下，在框架中更新文本绘制器属性将自动调用此方法。
  void markNeedsLayout() {
    _paragraph = null;
    _needsLayout = true;
  }

  /// 要绘制的(可能样式化的)文本。
  ///
  /// 在设置完这个域之后，必须先调用 [layout] ，之后才能调用 [paint].
  ///
  /// 它提供的[InlineSpan]是以树的形式提供的，树可能包含[TextSpan]s和[WidgetSpan]s的多个实例。要获得此
  /// [TextPainter]内容的纯文本表示，请使用 [InlineSpan.toPlainText] 获取树中所有节点的完整内容。 
  /// [TextSpan.text] 将只提供树中的第一个节点的内容。
  InlineSpan get text => _text;
  InlineSpan _text;
  set text(InlineSpan value) {
    /// 这个值必须是有效的
    assert(value == null || value.debugAssertIsValid());
    if (_text == value)
      return;
    if (_text?.style != value?.style)
      _layoutTemplate = null;
    _text = value;
    markNeedsLayout();
  }

  /// 文字的排列位置
  TextAlign get textAlign => _textAlign;
  TextAlign _textAlign;
  set textAlign(TextAlign value) {
    assert(value != null);
    if (_textAlign == value)
      return;
    _textAlign = value;
    markNeedsLayout();
  }

  /// 文字方向。
  /// 在调用 [layout] 的时候，这个值必须是非 null 的  
  TextDirection get textDirection => _textDirection;
  TextDirection _textDirection;
  set textDirection(TextDirection value) {
    if (_textDirection == value)
      return;
    _textDirection = value;
    markNeedsLayout();
    _layoutTemplate = null; // Shouldn't really matter, but for strict correctness...
  }

  /// 字体缩放比例
  double get textScaleFactor => _textScaleFactor;
  double _textScaleFactor;
  set textScaleFactor(double value) {
    assert(value != null);
    if (_textScaleFactor == value)
      return;
    _textScaleFactor = value;
    markNeedsLayout();
    _layoutTemplate = null;
  }

  /// 溢出的文本将用 ellipsis 来替换，默认为'...'
  String get ellipsis => _ellipsis;
  String _ellipsis;
  set ellipsis(String value) {
    assert(value == null || value.isNotEmpty);
    if (_ellipsis == value)
      return;
    _ellipsis = value;
    markNeedsLayout();
  }

  /// 语言
  Locale get locale => _locale;
  Locale _locale;
  set locale(Locale value) {
    if (_locale == value)
      return;
    _locale = value;
    markNeedsLayout();
  }

  /// 文本可选的最大行数，必要时进行换行。
  ///
  /// 如果文本超过给定的行数，则将其截断，以便删除后续行。
  ///
  /// 设置这个域之后，必须在下一次调用 [paint] 之前调用 [layout]。
  int get maxLines => _maxLines;
  int _maxLines;
  set maxLines(int value) {
    assert(value == null || value > 0);
    if (_maxLines == value)
      return;
    _maxLines = value;
    markNeedsLayout();
  }

  /// {@template flutter.painting.textPainter.strutStyle}
  /// 用的支撑式。Strut样式定义了Strut，它设置了最小的垂直布局指标。
  ///
  /// 提供 null 将会禁用垂直度量
  StrutStyle get strutStyle => _strutStyle;
  StrutStyle _strutStyle;
  set strutStyle(StrutStyle value) {
    if (_strutStyle == value)
      return;
    _strutStyle = value;
    markNeedsLayout();
  }

  /// 如何测量文字的宽度.
  TextWidthBasis get textWidthBasis => _textWidthBasis;
  TextWidthBasis _textWidthBasis;
  set textWidthBasis(TextWidthBasis value) {
    assert(value != null);
    if (_textWidthBasis == value)
      return;
    _textWidthBasis = value;
    markNeedsLayout();
  }


  ui.Paragraph _layoutTemplate;

  /// 一个有序的 [TextBox]列表，它将段落中占位符的位置绑定在一起。
  ///
  /// 每个框按照它们在[InlineSpan]树中定义的顺序对应一个[PlaceholderSpan]。
  List<TextBox> get inlinePlaceholderBoxes => _inlinePlaceholderBoxes;
  List<TextBox> _inlinePlaceholderBoxes;

  /// 段落中每个占位符的比例的有序列表。
  ///
  /// 刻度用作占位符的高度、宽度和基线偏移量的乘法因子，主要用于处理可访问性扩展。
  ///
  List<double> get inlinePlaceholderScales => _inlinePlaceholderScales;
  List<double> _inlinePlaceholderScales;

  void setPlaceholderDimensions(List<PlaceholderDimensions> value) {
    if (value == null || value.isEmpty || listEquals(value, _placeholderDimensions)) {
      return;
    }
    _placeholderDimensions = value;
    markNeedsLayout();
  }
  List<PlaceholderDimensions> _placeholderDimensions;

  /// 从给定的 TextSpan 中获取到 ParagraphStyle。如果不能，则使用默认的 ParagraphStyle
  ui.ParagraphStyle _createParagraphStyle([ TextDirection defaultTextDirection ]) {
    return _text.style?.getParagraphStyle(
      textAlign: textAlign,
      textDirection: textDirection ?? defaultTextDirection,
      textScaleFactor: textScaleFactor,
      maxLines: _maxLines,
      ellipsis: _ellipsis,
      locale: _locale,
      strutStyle: _strutStyle,
    ) ?? ui.ParagraphStyle(
      textAlign: textAlign,
      textDirection: textDirection ?? defaultTextDirection,
      maxLines: maxLines,
      ellipsis: ellipsis,
      locale: locale,
    );
  }

  /// [text] 的逻辑像素高度
  ///
  /// 并不是 [text] 中的每行文本都有这个高度，但是对于 [text] 中的文本来说，这个高度是“典型的”，对于调整其他
  //  对象相对于典型文本行的大小很有用。
  ///
  /// 获取这个值无需提前调用 [layout].
  ///
  /// [text] 属性的样式用于确定有助于 [preferredLineHeight] 的字体设置。如果 [text] 为 null 或者没有指定样
  /// 式，则使用默认的 [TextStyle] 值(10 pixel sans-serif font)。
  double get preferredLineHeight {
    if (_layoutTemplate == null) {
      final ui.ParagraphBuilder builder = ui.ParagraphBuilder(
        _createParagraphStyle(TextDirection.rtl),
      ); // direction doesn't matter, text is just a space
      /// 尝试构建一次，得到可用的高度  
      if (text?.style != null)
        builder.pushStyle(text.style.getTextStyle(textScaleFactor: textScaleFactor));
      builder.addText(' ');
      _layoutTemplate = builder.build()
        ..layout(const ui.ParagraphConstraints(width: double.infinity));
    }
    return _layoutTemplate.height;
  }

  // 不幸的是，在这里使用精确精度浮点数会导致糟糕的布局
  // 因为浮点数不是结合律。例如，如果我们增加或减少填充，我们在估计大小和实际计算布局时将得到不同的值，因为操作
  // 将以不同的方式关联起来。为了解决这个问题，我们将小像素值四舍五入到最接近的整像素值。正确的长期修正是做布局
  // 使用固定精度计算。
  // 布局的时候应该采用整数精度浮点数
  double _applyFloatingPointHack(double layoutValue) {
    return layoutValue.ceilToDouble();
  }

  /// 最小固有宽度
  /// 如果布局宽度小于这个值的话，会造成绘制不完整
  ///
  /// 只在调用完 [layout] 后有效
  double get minIntrinsicWidth {
    assert(!_needsLayout);
    return _applyFloatingPointHack(_paragraph.minIntrinsicWidth);
  }

  /// 最大固有宽度
  /// 如果布局宽度大与这个值的话，也不会减小最终的高度
  ///
  /// 只在调用完 [layout] 后有效
  double get maxIntrinsicWidth {
    return _applyFloatingPointHack(_paragraph.maxIntrinsicWidth);
  }

  /// 文字的水平宽度
  /// 只在调用完 [layout] 后有效
  double get width {
    assert(!_needsLayout);
    return _applyFloatingPointHack(
      textWidthBasis == TextWidthBasis.longestLine ? _paragraph.longestLine : _paragraph.width,
    );
  }

  /// 文字的垂直高度
  /// 只在调用完 [layout] 后有效
  double get height {
    assert(!_needsLayout);
    return _applyFloatingPointHack(_paragraph.height);
  }

  /// 大小
  /// 只在调用完 [layout] 后有效
  Size get size {
    assert(!_needsLayout);
    return Size(width, height);
  }

  /// 返回从文本顶部到给定类型的第一个基线的距离。
  /// 只在调用完 [layout] 后有效
  double computeDistanceToActualBaseline(TextBaseline baseline) {
    assert(!_needsLayout);
    assert(baseline != null);
    switch (baseline) {
      case TextBaseline.alphabetic:
        return _paragraph.alphabeticBaseline;
      case TextBaseline.ideographic:
        return _paragraph.ideographicBaseline;
    }
    return null;
  }

  /// 是否有任何文本超出了最大行数而被截断或省略。
  ///
  /// 如果 [maxLines] 不为null，则当要绘制的行比给定的 [maxLines] 多，从而在输出中至少遗漏一行时，为 true
  /// 否则是 false
  ///
  /// 如果 [maxLines] 为空，如果[ellipes] 不是空字符串，且有一行溢出了传递给 [layout] 的' maxWidth '参数，
  /// 则为 true;否则是 false。
  ///
  /// 只有在调用 [layout] 后才有效。
  bool get didExceedMaxLines {
    return _paragraph.didExceedMaxLines;
  }

  double _lastMinWidth;
  double _lastMaxWidth;

  /// 计算绘制文本时字形的视觉位置。
  ///
  /// 文本布局的宽度将尽可能接近其最大固有宽度，同时仍然大于或等于“minWidth”，小于或等于“maxWidth”。
  void layout({ double minWidth = 0.0, double maxWidth = double.infinity }) {
    if (!_needsLayout && minWidth == _lastMinWidth && maxWidth == _lastMaxWidth)
      return;
    _needsLayout = false;
    /// 如果没有缓存，那么使用给定的 InlineSpan 来构建
    if (_paragraph == null) {
      final ui.ParagraphBuilder builder = ui.ParagraphBuilder(_createParagraphStyle());
      _text.build(builder, textScaleFactor: textScaleFactor, dimensions: _placeholderDimensions);
      /// _text.build 通常是调用 paragraphBuild.pushStyle 和 .addText 方法，来将给定的 textSpan 信息加入
      /// 到绘制栈中
      /// if (hasStyle)
      ///	builder.pushStyle(style.getTextStyle(textScaleFactor: textScaleFactor));
      /// if (text != null)
      /// 	 builder.addText(text);
      _inlinePlaceholderScales = builder.placeholderScales;
      _paragraph = builder.build();
    }
    _lastMinWidth = minWidth;
    _lastMaxWidth = maxWidth;
    /// 使用最大宽度作为约束来布局
    _paragraph.layout(ui.ParagraphConstraints(width: maxWidth));
    if (minWidth != maxWidth) {
      /// newWidth 是调用 paragraph.layout 后得到的最大固有宽度
      final double newWidth = maxIntrinsicWidth.clamp(minWidth, maxWidth);
      /// width 是调用 paragraph.layout 后得到的宽度
      /// 如果两者不同的话，使用最大固有宽度重新布局一次
      if (newWidth != width) {
        _paragraph.layout(ui.ParagraphConstraints(width: newWidth));
      }
    }
    _inlinePlaceholderBoxes = _paragraph.getBoxesForPlaceholders();
  }

  /// 将已经布局好的 paragraph 绘制在给定的 canvas 上
  void paint(Canvas canvas, Offset offset) {
    canvas.drawParagraph(_paragraph, offset);
  }

  bool _isUtf16Surrogate(int value) {
    return value & 0xF800 == 0xD800;
  }

  /// 返回给定“偏移量”之后的最近偏移量，输入光标可以定位在该偏移量上。
  int getOffsetAfter(int offset) {
    final int nextCodeUnit = _text.codeUnitAt(offset);
    if (nextCodeUnit == null)
      return null;
    return _isUtf16Surrogate(nextCodeUnit) ? offset + 2 : offset + 1;
  }

  /// 返回给定“偏移量”之前的最近偏移量，输入光标可以定位在该偏移量上。
  int getOffsetBefore(int offset) {
    final int prevCodeUnit = _text.codeUnitAt(offset - 1);
    if (prevCodeUnit == null)
      return null;
    return _isUtf16Surrogate(prevCodeUnit) ? offset - 2 : offset - 1;
  }

  static const int _zwjUtf16 = 0x200d;

  Rect _getRectFromUpstream(int offset, Rect caretPrototype) {
    final String flattenedText = _text.toPlainText(includePlaceholders: false);
    final int prevCodeUnit = _text.codeUnitAt(max(0, offset - 1));
    if (prevCodeUnit == null)
      return null;

    final bool needsSearch = _isUtf16Surrogate(prevCodeUnit) || _text.codeUnitAt(offset) == _zwjUtf16;
    int graphemeClusterLength = needsSearch ? 2 : 1;
    List<TextBox> boxes = <TextBox>[];
    while (boxes.isEmpty && flattenedText != null) {
      final int prevRuneOffset = offset - graphemeClusterLength;
      boxes = _paragraph.getBoxesForRange(prevRuneOffset, offset);
      // When the range does not include a full cluster, no boxes will be returned.
      if (boxes.isEmpty) {
        // When we are at the beginning of the line, a non-surrogate position will
        // return empty boxes. We break and try from downstream instead.
        if (!needsSearch) {
          break; // Only perform one iteration if no search is required.
        }
        if (prevRuneOffset < -flattenedText.length) {
          break; // Stop iterating when beyond the max length of the text.
        }
        // Multiply by two to log(n) time cover the entire text span. This allows
        // faster discovery of very long clusters and reduces the possibility
        // of certain large clusters taking much longer than others, which can
        // cause jank.
        graphemeClusterLength *= 2;
        continue;
      }
      final TextBox box = boxes.first;

      // If the upstream character is a newline, cursor is at start of next line
      const int NEWLINE_CODE_UNIT = 10;
      if (prevCodeUnit == NEWLINE_CODE_UNIT) {
        return Rect.fromLTRB(_emptyOffset.dx, box.bottom, _emptyOffset.dx, box.bottom + box.bottom - box.top);
      }

      final double caretEnd = box.end;
      final double dx = box.direction == TextDirection.rtl ? caretEnd - caretPrototype.width : caretEnd;
      return Rect.fromLTRB(min(dx, _paragraph.width), box.top, min(dx, _paragraph.width), box.bottom);
    }
    return null;
  }

  // Get the Rect of the cursor (in logical pixels) based off the near edge
  // of the character downstream from the given string offset.
  // TODO(garyq): Use actual extended grapheme cluster length instead of
  // an increasing cluster length amount to achieve deterministic performance.
  Rect _getRectFromDownstream(int offset, Rect caretPrototype) {
    final String flattenedText = _text.toPlainText(includePlaceholders: false);
    // We cap the offset at the final index of the _text.
    final int nextCodeUnit = _text.codeUnitAt(min(offset, flattenedText == null ? 0 : flattenedText.length - 1));
    if (nextCodeUnit == null)
      return null;
    // Check for multi-code-unit glyphs such as emojis or zero width joiner
    final bool needsSearch = _isUtf16Surrogate(nextCodeUnit) || nextCodeUnit == _zwjUtf16;
    int graphemeClusterLength = needsSearch ? 2 : 1;
    List<TextBox> boxes = <TextBox>[];
    while (boxes.isEmpty && flattenedText != null) {
      final int nextRuneOffset = offset + graphemeClusterLength;
      boxes = _paragraph.getBoxesForRange(offset, nextRuneOffset);
      // When the range does not include a full cluster, no boxes will be returned.
      if (boxes.isEmpty) {
        // When we are at the end of the line, a non-surrogate position will
        // return empty boxes. We break and try from upstream instead.
        if (!needsSearch) {
          break; // Only perform one iteration if no search is required.
        }
        if (nextRuneOffset >= flattenedText.length << 1) {
          break; // Stop iterating when beyond the max length of the text.
        }
        // Multiply by two to log(n) time cover the entire text span. This allows
        // faster discovery of very long clusters and reduces the possibility
        // of certain large clusters taking much longer than others, which can
        // cause jank.
        graphemeClusterLength *= 2;
        continue;
      }
      final TextBox box = boxes.last;
      final double caretStart = box.start;
      final double dx = box.direction == TextDirection.rtl ? caretStart - caretPrototype.width : caretStart;
      return Rect.fromLTRB(min(dx, _paragraph.width), box.top, min(dx, _paragraph.width), box.bottom);
    }
    return null;
  }

  Offset get _emptyOffset {
    assert(!_needsLayout); // implies textDirection is non-null
    assert(textAlign != null);
    switch (textAlign) {
      case TextAlign.left:
        return Offset.zero;
      case TextAlign.right:
        return Offset(width, 0.0);
      case TextAlign.center:
        return Offset(width / 2.0, 0.0);
      case TextAlign.justify:
      case TextAlign.start:
        assert(textDirection != null);
        switch (textDirection) {
          case TextDirection.rtl:
            return Offset(width, 0.0);
          case TextDirection.ltr:
            return Offset.zero;
        }
        return null;
      case TextAlign.end:
        assert(textDirection != null);
        switch (textDirection) {
          case TextDirection.rtl:
            return Offset.zero;
          case TextDirection.ltr:
            return Offset(width, 0.0);
        }
        return null;
    }
    return null;
  }

  /// Returns the offset at which to paint the caret.
  ///
  /// Valid only after [layout] has been called.
  Offset getOffsetForCaret(TextPosition position, Rect caretPrototype) {
    _computeCaretMetrics(position, caretPrototype);
    return _caretMetrics.offset;
  }

  /// Returns the tight bounded height of the glyph at the given [position].
  ///
  /// Valid only after [layout] has been called.
  double getFullHeightForCaret(TextPosition position, Rect caretPrototype) {
    _computeCaretMetrics(position, caretPrototype);
    return _caretMetrics.fullHeight;
  }

  // Cached caret metrics. This allows multiple invokes of [getOffsetForCaret] and
  // [getFullHeightForCaret] in a row without performing redundant and expensive
  // get rect calls to the paragraph.
  _CaretMetrics _caretMetrics;

  // Holds the TextPosition and caretPrototype the last caret metrics were
  // computed with. When new values are passed in, we recompute the caret metrics.
  // only as necessary.
  TextPosition _previousCaretPosition;
  Rect _previousCaretPrototype;

  // Checks if the [position] and [caretPrototype] have changed from the cached
  // version and recomputes the metrics required to position the caret.
  void _computeCaretMetrics(TextPosition position, Rect caretPrototype) {
    assert(!_needsLayout);
    if (position == _previousCaretPosition && caretPrototype == _previousCaretPrototype)
      return;
    final int offset = position.offset;
    assert(position.affinity != null);
    Rect rect;
    switch (position.affinity) {
      case TextAffinity.upstream: {
        rect = _getRectFromUpstream(offset, caretPrototype) ?? _getRectFromDownstream(offset, caretPrototype);
        break;
      }
      case TextAffinity.downstream: {
        rect = _getRectFromDownstream(offset, caretPrototype) ??  _getRectFromUpstream(offset, caretPrototype);
        break;
      }
    }
    _caretMetrics = _CaretMetrics(
      offset: rect != null ? Offset(rect.left, rect.top) : _emptyOffset,
      fullHeight: rect != null ? rect.bottom - rect.top : null,
    );

    // Cache the input parameters to prevent repeat work later.
    _previousCaretPosition = position;
    _previousCaretPrototype = caretPrototype;
  }

  /// Returns a list of rects that bound the given selection.
  ///
  /// A given selection might have more than one rect if this text painter
  /// contains bidirectional text because logically contiguous text might not be
  /// visually contiguous.
  List<TextBox> getBoxesForSelection(TextSelection selection) {
    assert(!_needsLayout);
    return _paragraph.getBoxesForRange(selection.start, selection.end);
  }

  /// Returns the position within the text for the given pixel offset.
  TextPosition getPositionForOffset(Offset offset) {
    assert(!_needsLayout);
    return _paragraph.getPositionForOffset(offset);
  }

  /// Returns the text range of the word at the given offset. Characters not
  /// part of a word, such as spaces, symbols, and punctuation, have word breaks
  /// on both sides. In such cases, this method will return a text range that
  /// contains the given text position.
  ///
  /// Word boundaries are defined more precisely in Unicode Standard Annex #29
  /// <http://www.unicode.org/reports/tr29/#Word_Boundaries>.
  TextRange getWordBoundary(TextPosition position) {
    assert(!_needsLayout);
    return _paragraph.getWordBoundary(position);
  }

  /// Returns the text range of the line at the given offset.
  ///
  /// The newline, if any, is included in the range.
  TextRange getLineBoundary(TextPosition position) {
    assert(!_needsLayout);
    return _paragraph.getLineBoundary(position);
  }

  /// Returns the full list of [LineMetrics] that describe in detail the various
  /// metrics of each laid out line.
  ///
  /// The [LineMetrics] list is presented in the order of the lines they represent.
  /// For example, the first line is in the zeroth index.
  ///
  /// [LineMetrics] contains measurements such as ascent, descent, baseline, and
  /// width for the line as a whole, and may be useful for aligning additional
  /// widgets to a particular line.
  ///
  /// Valid only after [layout] has been called.
  ///
  /// This can potentially return a large amount of data, so it is not recommended
  /// to repeatedly call this. Instead, cache the results. The cached results
  /// should be invalidated upon the next successful [layout].
  List<ui.LineMetrics> computeLineMetrics() {
    assert(!_needsLayout);
    return _paragraph.computeLineMetrics();
  }
}
```


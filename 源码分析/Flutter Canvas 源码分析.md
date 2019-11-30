# Flutter Canvas 源码分析

## Paint

```dart
/// A description of the style to use when drawing on a [Canvas].
///
/// Most APIs on [Canvas] take a [Paint] object to describe the style
/// to use for that operation.
class Paint {
  // Paint objects are encoded in two buffers:
  //
  // * _data is binary data in four-byte fields, each of which is either a
  //   uint32_t or a float. The default value for each field is encoded as
  //   zero to make initialization trivial. Most values already have a default
  //   value of zero, but some, such as color, have a non-zero default value.
  //   To encode or decode these values, XOR the value with the default value.
  //
  // * _objects is a list of unencodable objects, typically wrappers for native
  //   objects. The objects are simply stored in the list without any additional
  //   encoding.
  //
  // The binary format must match the deserialization code in paint.cc.

  final ByteData _data = ByteData(_kDataByteCount);
  static const int _kIsAntiAliasIndex = 0;
  static const int _kColorIndex = 1;
  static const int _kBlendModeIndex = 2;
  static const int _kStyleIndex = 3;
  static const int _kStrokeWidthIndex = 4;
  static const int _kStrokeCapIndex = 5;
  static const int _kStrokeJoinIndex = 6;
  static const int _kStrokeMiterLimitIndex = 7;
  static const int _kFilterQualityIndex = 8;
  static const int _kMaskFilterIndex = 9;
  static const int _kMaskFilterBlurStyleIndex = 10;
  static const int _kMaskFilterSigmaIndex = 11;
  static const int _kInvertColorIndex = 12;

  static const int _kIsAntiAliasOffset = _kIsAntiAliasIndex << 2;
  static const int _kColorOffset = _kColorIndex << 2;
  static const int _kBlendModeOffset = _kBlendModeIndex << 2;
  static const int _kStyleOffset = _kStyleIndex << 2;
  static const int _kStrokeWidthOffset = _kStrokeWidthIndex << 2;
  static const int _kStrokeCapOffset = _kStrokeCapIndex << 2;
  static const int _kStrokeJoinOffset = _kStrokeJoinIndex << 2;
  static const int _kStrokeMiterLimitOffset = _kStrokeMiterLimitIndex << 2;
  static const int _kFilterQualityOffset = _kFilterQualityIndex << 2;
  static const int _kMaskFilterOffset = _kMaskFilterIndex << 2;
  static const int _kMaskFilterBlurStyleOffset = _kMaskFilterBlurStyleIndex << 2;
  static const int _kMaskFilterSigmaOffset = _kMaskFilterSigmaIndex << 2;
  static const int _kInvertColorOffset = _kInvertColorIndex << 2;
  // If you add more fields, remember to update _kDataByteCount.
  static const int _kDataByteCount = 52;

  // Binary format must match the deserialization code in paint.cc.
  List<dynamic> _objects;
  static const int _kShaderIndex = 0;
  static const int _kColorFilterIndex = 1;
  static const int _kImageFilterIndex = 2;
  static const int _kObjectCount = 3; // Must be one larger than the largest index.

  /// Whether to apply anti-aliasing to lines and images drawn on the
  /// canvas.
  ///
  /// Defaults to true.
  bool get isAntiAlias {
    return _data.getInt32(_kIsAntiAliasOffset, _kFakeHostEndian) == 0;
  }
  set isAntiAlias(bool value) {
    // We encode true as zero and false as one because the default value, which
    // we always encode as zero, is true.
    final int encoded = value ? 0 : 1;
    _data.setInt32(_kIsAntiAliasOffset, encoded, _kFakeHostEndian);
  }

  // Must be kept in sync with the default in paint.cc.
  static const int _kColorDefault = 0xFF000000;

  /// The color to use when stroking or filling a shape.
  ///
  /// Defaults to opaque black.
  ///
  /// See also:
  ///
  ///  * [style], which controls whether to stroke or fill (or both).
  ///  * [colorFilter], which overrides [color].
  ///  * [shader], which overrides [color] with more elaborate effects.
  ///
  /// This color is not used when compositing. To colorize a layer, use
  /// [colorFilter].
  Color get color {
    final int encoded = _data.getInt32(_kColorOffset, _kFakeHostEndian);
    return Color(encoded ^ _kColorDefault);
  }
  set color(Color value) {
    assert(value != null);
    final int encoded = value.value ^ _kColorDefault;
    _data.setInt32(_kColorOffset, encoded, _kFakeHostEndian);
  }

  // Must be kept in sync with the default in paint.cc.
  static final int _kBlendModeDefault = BlendMode.srcOver.index;

  /// A blend mode to apply when a shape is drawn or a layer is composited.
  ///
  /// The source colors are from the shape being drawn (e.g. from
  /// [Canvas.drawPath]) or layer being composited (the graphics that were drawn
  /// between the [Canvas.saveLayer] and [Canvas.restore] calls), after applying
  /// the [colorFilter], if any.
  ///
  /// The destination colors are from the background onto which the shape or
  /// layer is being composited.
  ///
  /// Defaults to [BlendMode.srcOver].
  ///
  /// See also:
  ///
  ///  * [Canvas.saveLayer], which uses its [Paint]'s [blendMode] to composite
  ///    the layer when [restore] is called.
  ///  * [BlendMode], which discusses the user of [saveLayer] with [blendMode].
  BlendMode get blendMode {
    final int encoded = _data.getInt32(_kBlendModeOffset, _kFakeHostEndian);
    return BlendMode.values[encoded ^ _kBlendModeDefault];
  }
  set blendMode(BlendMode value) {
    assert(value != null);
    final int encoded = value.index ^ _kBlendModeDefault;
    _data.setInt32(_kBlendModeOffset, encoded, _kFakeHostEndian);
  }

  /// Whether to paint inside shapes, the edges of shapes, or both.
  ///
  /// Defaults to [PaintingStyle.fill].
  PaintingStyle get style {
    return PaintingStyle.values[_data.getInt32(_kStyleOffset, _kFakeHostEndian)];
  }
  set style(PaintingStyle value) {
    assert(value != null);
    final int encoded = value.index;
    _data.setInt32(_kStyleOffset, encoded, _kFakeHostEndian);
  }

  /// How wide to make edges drawn when [style] is set to
  /// [PaintingStyle.stroke]. The width is given in logical pixels measured in
  /// the direction orthogonal to the direction of the path.
  ///
  /// Defaults to 0.0, which correspond to a hairline width.
  double get strokeWidth {
    return _data.getFloat32(_kStrokeWidthOffset, _kFakeHostEndian);
  }
  set strokeWidth(double value) {
    assert(value != null);
    final double encoded = value;
    _data.setFloat32(_kStrokeWidthOffset, encoded, _kFakeHostEndian);
  }

  /// The kind of finish to place on the end of lines drawn when
  /// [style] is set to [PaintingStyle.stroke].
  ///
  /// Defaults to [StrokeCap.butt], i.e. no caps.
  StrokeCap get strokeCap {
    return StrokeCap.values[_data.getInt32(_kStrokeCapOffset, _kFakeHostEndian)];
  }
  set strokeCap(StrokeCap value) {
    assert(value != null);
    final int encoded = value.index;
    _data.setInt32(_kStrokeCapOffset, encoded, _kFakeHostEndian);
  }

  /// The kind of finish to place on the joins between segments.
  ///
  /// This applies to paths drawn when [style] is set to [PaintingStyle.stroke],
  /// It does not apply to points drawn as lines with [Canvas.drawPoints].
  ///
  /// Defaults to [StrokeJoin.miter], i.e. sharp corners.
  ///
  /// Some examples of joins:
  ///
  /// {@animation 300 300 https://flutter.github.io/assets-for-api-docs/assets/dart-ui/miter_4_join.mp4}
  ///
  /// {@animation 300 300 https://flutter.github.io/assets-for-api-docs/assets/dart-ui/round_join.mp4}
  ///
  /// {@animation 300 300 https://flutter.github.io/assets-for-api-docs/assets/dart-ui/bevel_join.mp4}
  ///
  /// The centers of the line segments are colored in the diagrams above to
  /// highlight the joins, but in normal usage the join is the same color as the
  /// line.
  ///
  /// See also:
  ///
  ///  * [strokeMiterLimit] to control when miters are replaced by bevels when
  ///    this is set to [StrokeJoin.miter].
  ///  * [strokeCap] to control what is drawn at the ends of the stroke.
  ///  * [StrokeJoin] for the definitive list of stroke joins.
  StrokeJoin get strokeJoin {
    return StrokeJoin.values[_data.getInt32(_kStrokeJoinOffset, _kFakeHostEndian)];
  }
  set strokeJoin(StrokeJoin value) {
    assert(value != null);
    final int encoded = value.index;
    _data.setInt32(_kStrokeJoinOffset, encoded, _kFakeHostEndian);
  }

  // Must be kept in sync with the default in paint.cc.
  static const double _kStrokeMiterLimitDefault = 4.0;

  /// The limit for miters to be drawn on segments when the join is set to
  /// [StrokeJoin.miter] and the [style] is set to [PaintingStyle.stroke]. If
  /// this limit is exceeded, then a [StrokeJoin.bevel] join will be drawn
  /// instead. This may cause some 'popping' of the corners of a path if the
  /// angle between line segments is animated, as seen in the diagrams below.
  ///
  /// This limit is expressed as a limit on the length of the miter.
  ///
  /// Defaults to 4.0.  Using zero as a limit will cause a [StrokeJoin.bevel]
  /// join to be used all the time.
  ///
  /// {@animation 300 300 https://flutter.github.io/assets-for-api-docs/assets/dart-ui/miter_0_join.mp4}
  ///
  /// {@animation 300 300 https://flutter.github.io/assets-for-api-docs/assets/dart-ui/miter_4_join.mp4}
  ///
  /// {@animation 300 300 https://flutter.github.io/assets-for-api-docs/assets/dart-ui/miter_6_join.mp4}
  ///
  /// The centers of the line segments are colored in the diagrams above to
  /// highlight the joins, but in normal usage the join is the same color as the
  /// line.
  ///
  /// See also:
  ///
  ///  * [strokeJoin] to control the kind of finish to place on the joins
  ///    between segments.
  ///  * [strokeCap] to control what is drawn at the ends of the stroke.
  double get strokeMiterLimit {
    return _data.getFloat32(_kStrokeMiterLimitOffset, _kFakeHostEndian);
  }
  set strokeMiterLimit(double value) {
    assert(value != null);
    final double encoded = value - _kStrokeMiterLimitDefault;
    _data.setFloat32(_kStrokeMiterLimitOffset, encoded, _kFakeHostEndian);
  }

  /// A mask filter (for example, a blur) to apply to a shape after it has been
  /// drawn but before it has been composited into the image.
  ///
  /// See [MaskFilter] for details.
  MaskFilter get maskFilter {
    switch (_data.getInt32(_kMaskFilterOffset, _kFakeHostEndian)) {
      case MaskFilter._TypeNone:
        return null;
      case MaskFilter._TypeBlur:
        return MaskFilter.blur(
          BlurStyle.values[_data.getInt32(_kMaskFilterBlurStyleOffset, _kFakeHostEndian)],
          _data.getFloat32(_kMaskFilterSigmaOffset, _kFakeHostEndian),
        );
    }
    return null;
  }
  set maskFilter(MaskFilter value) {
    if (value == null) {
      _data.setInt32(_kMaskFilterOffset, MaskFilter._TypeNone, _kFakeHostEndian);
      _data.setInt32(_kMaskFilterBlurStyleOffset, 0, _kFakeHostEndian);
      _data.setFloat32(_kMaskFilterSigmaOffset, 0.0, _kFakeHostEndian);
    } else {
      // For now we only support one kind of MaskFilter, so we don't need to
      // check what the type is if it's not null.
      _data.setInt32(_kMaskFilterOffset, MaskFilter._TypeBlur, _kFakeHostEndian);
      _data.setInt32(_kMaskFilterBlurStyleOffset, value._style.index, _kFakeHostEndian);
      _data.setFloat32(_kMaskFilterSigmaOffset, value._sigma, _kFakeHostEndian);
    }
  }

  /// Controls the performance vs quality trade-off to use when applying
  /// filters, such as [maskFilter], or when drawing images, as with
  /// [Canvas.drawImageRect] or [Canvas.drawImageNine].
  ///
  /// Defaults to [FilterQuality.none].
  // TODO(ianh): verify that the image drawing methods actually respect this
  FilterQuality get filterQuality {
    return FilterQuality.values[_data.getInt32(_kFilterQualityOffset, _kFakeHostEndian)];
  }
  set filterQuality(FilterQuality value) {
    assert(value != null);
    final int encoded = value.index;
    _data.setInt32(_kFilterQualityOffset, encoded, _kFakeHostEndian);
  }

  /// The shader to use when stroking or filling a shape.
  ///
  /// When this is null, the [color] is used instead.
  ///
  /// See also:
  ///
  ///  * [Gradient], a shader that paints a color gradient.
  ///  * [ImageShader], a shader that tiles an [Image].
  ///  * [colorFilter], which overrides [shader].
  ///  * [color], which is used if [shader] and [colorFilter] are null.
  Shader get shader {
    if (_objects == null)
      return null;
    return _objects[_kShaderIndex];
  }
  set shader(Shader value) {
    _objects ??= List<dynamic>(_kObjectCount);
    _objects[_kShaderIndex] = value;
  }

  /// A color filter to apply when a shape is drawn or when a layer is
  /// composited.
  ///
  /// See [ColorFilter] for details.
  ///
  /// When a shape is being drawn, [colorFilter] overrides [color] and [shader].
  ColorFilter get colorFilter {
    if (_objects == null || _objects[_kColorFilterIndex] == null) {
      return null;
    }
    return _objects[_kColorFilterIndex].creator;
  }

  set colorFilter(ColorFilter value) {
    final _ColorFilter nativeFilter = value?._toNativeColorFilter();
    if (nativeFilter == null) {
      if (_objects != null) {
        _objects[_kColorFilterIndex] = null;
      }
    } else {
      if (_objects == null) {
        _objects = List<dynamic>(_kObjectCount);
        _objects[_kColorFilterIndex] = nativeFilter;
      } else if (_objects[_kColorFilterIndex]?.creator != value) {
        _objects[_kColorFilterIndex] = nativeFilter;
      }
    }
  }

  /// The [ImageFilter] to use when drawing raster images.
  ///
  /// For example, to blur an image using [Canvas.drawImage], apply an
  /// [ImageFilter.blur]:
  ///
  /// ```dart
  /// import 'dart:ui' as ui;
  ///
  /// ui.Image image;
  ///
  /// void paint(Canvas canvas, Size size) {
  ///   canvas.drawImage(
  ///     image,
  ///     Offset.zero,
  ///     Paint()..imageFilter = ui.ImageFilter.blur(sigmaX: .5, sigmaY: .5),
  ///   );
  /// }
  /// ```
  ///
  /// See also:
  ///
  ///  * [MaskFilter], which is used for drawing geometry.
  ImageFilter get imageFilter {
    if (_objects == null || _objects[_kImageFilterIndex] == null)
      return null;
    return _objects[_kImageFilterIndex].creator;
  }

  set imageFilter(ImageFilter value) {
    if (value == null) {
      if (_objects != null) {
        _objects[_kImageFilterIndex] = null;
      }
    } else {
      _objects ??= List<dynamic>(_kObjectCount);
      if (_objects[_kImageFilterIndex]?.creator != value) {
        _objects[_kImageFilterIndex] = value._toNativeImageFilter();
      }
    }
  }

  /// Whether the colors of the image are inverted when drawn.
  ///
  /// Inverting the colors of an image applies a new color filter that will
  /// be composed with any user provided color filters. This is primarily
  /// used for implementing smart invert on iOS.
  bool get invertColors {
    return _data.getInt32(_kInvertColorOffset, _kFakeHostEndian) == 1;
  }
  set invertColors(bool value) {
    _data.setInt32(_kInvertColorOffset, value ? 1 : 0, _kFakeHostEndian);
  }

  @override
  String toString() {
    final StringBuffer result = StringBuffer();
    String semicolon = '';
    result.write('Paint(');
    if (style == PaintingStyle.stroke) {
      result.write('$style');
      if (strokeWidth != 0.0)
        result.write(' ${strokeWidth.toStringAsFixed(1)}');
      else
        result.write(' hairline');
      if (strokeCap != StrokeCap.butt)
        result.write(' $strokeCap');
      if (strokeJoin == StrokeJoin.miter) {
        if (strokeMiterLimit != _kStrokeMiterLimitDefault)
          result.write(' $strokeJoin up to ${strokeMiterLimit.toStringAsFixed(1)}');
      } else {
        result.write(' $strokeJoin');
      }
      semicolon = '; ';
    }
    if (isAntiAlias != true) {
      result.write('${semicolon}antialias off');
      semicolon = '; ';
    }
    if (color != const Color(_kColorDefault)) {
      if (color != null)
        result.write('$semicolon$color');
      else
        result.write('${semicolon}no color');
      semicolon = '; ';
    }
    if (blendMode.index != _kBlendModeDefault) {
      result.write('$semicolon$blendMode');
      semicolon = '; ';
    }
    if (colorFilter != null) {
      result.write('${semicolon}colorFilter: $colorFilter');
      semicolon = '; ';
    }
    if (maskFilter != null) {
      result.write('${semicolon}maskFilter: $maskFilter');
      semicolon = '; ';
    }
    if (filterQuality != FilterQuality.none) {
      result.write('${semicolon}filterQuality: $filterQuality');
      semicolon = '; ';
    }
    if (shader != null) {
      result.write('${semicolon}shader: $shader');
      semicolon = '; ';
    }
    if (imageFilter != null) {
      result.write('${semicolon}imageFilter: $imageFilter');
      semicolon = '; ';
    }
    if (invertColors)
      result.write('${semicolon}invert: $invertColors');
    result.write(')');
    return result.toString();
  }
}
```



## Canvas

```dart
/// [Canvas]是用于记录图形化操作的接口。
/// 
/// [Canvas]对象用于创建[Picture]对象，[Picture]本身可以与[SceneBuilder]一起构建[Scene]。然而，在
/// 正常情况下，这一切都是由框架来处理的。
/// 画布有一个当前转换矩阵，它被应用于所有操作。最初，变换矩阵是单位变换。可以使用[translate]、
/// [scale]、[rotate]、[skew]和[transform]方法修改它。
///
/// 画布也有一个应用于所有操作的当前剪裁区域。最初，剪辑区域是无限的。可以使用[clipRect]、[clipRRect]
/// 和[clipPath]方法修改它。
///
/// 当前的变换和剪裁可以使用堆栈保存和恢复，由[save]、[saveLayer]和[restore]方法管理。
class Canvas extends NativeFieldWrapperClass2 {
  /// 创建一个画布，用于将图形操作记录到给定的图像记录器中。
  ///
  /// 在给定“cullRect”之外完全影响像素的图形操作可能会被丢弃。但是，如果某个命令同时在 “cullRect”内部
  /// 和外部绘制，则可能会绘制这些界限之外的内容。为了确保给定区域外的像素被丢弃，可以考虑使用
  /// [clipRect]。“cullRect”是可选的;默认情况下，保留所有操作。
  ///
  /// 为了停止录制, 调用给定 recorder 的[PictureRecorder.endRecording]
  @pragma('vm:entry-point')
  Canvas(PictureRecorder recorder, [ Rect cullRect ]) : assert(recorder != null) {
    cullRect ??= Rect.largest;
    _constructor(recorder, cullRect.left, cullRect.top, cullRect.right, cullRect.bottom);
  }
  void _constructor(PictureRecorder recorder,
                    double left,
                    double top,
                    double right,
                    double bottom) native 'Canvas_constructor';

  /// 在栈中保存一次当前的变换和裁剪状态，调用 [restore]，来退栈一次
  void save() native 'Canvas_save';

  /// 在栈中保存当前转换和裁剪，然后创建一个新的组，后续调用将成为其中的一部分。当保存堆栈稍后弹出时，该组
  /// 将被平铺到一个层中，并且调用给定“paint”的[paint.colorFilter]和[paint.blendMode]。
  ///
  /// 可以创建复合效果，例如使一组绘图命令半透明。如果不使用[saveLayer]，小组的每个部分都将单独绘制，因
  /// 此重叠的地方会比没有重叠的地方颜色更深。通过使用[saveLayer]将它们组合在一起，它们可以在一开始用不
  /// 透明的颜色绘制，然后使用[saveLayer]的颜料使整个组透明。
  ///
  /// 调用 [restore] 来弹出栈并且将绘制效果运用到组上
  ///
  /// ## Using saveLayer with clips
  ///
  /// 当矩形裁剪操作(来自[clipRect])未与光栅缓冲区轴向对齐时，或剪辑操作不是直线时
  /// (例如，因为它是[clipRRect]创建的圆角矩形裁剪或[clipPath]创建的任意复杂路径剪辑)
  /// 剪辑的边缘需要抗锯齿。
  ///
  /// 如果两个draw调用在这样一个裁剪区域的边缘重叠，而不使用[saveLayer]，那么第一个绘图将首先与背景进行
  /// 反锯齿，然后第二个绘图将混合第一个绘图和背景进行反锯齿。另一方面，如果在建立剪辑后立即使用
  /// [saveLayer]，则第二个绘图将覆盖该层中的第一个，因此，在剪切和合成该层时(调用[restore]时)，第二个
  /// 绘图将仅与背景反锯齿。
  ///
  /// 例如：[CustomPainter.paint] 绘制白色圆角矩形
  ///
  /// ```dart
  /// void paint(Canvas canvas, Size size) {
  ///   Rect rect = Offset.zero & size;
  ///   canvas.save();
  ///   canvas.clipRRect(new RRect.fromRectXY(rect, 100.0, 100.0));
  ///   canvas.saveLayer(rect, Paint());
  ///   canvas.drawPaint(new Paint()..color = Colors.red);
  ///   canvas.drawPaint(new Paint()..color = Colors.white);
  ///   canvas.restore();
  ///   canvas.restore();
  /// }
  /// ```
  ///
  /// 另一方面，这个例子渲染了一个红色的轮廓，红色绘制的结果是与背景在剪裁边缘的反锯齿，
  /// 然后类似的，白色的绘制是与背景相似的反锯齿，包括已被裁剪的的红色绘制:
  ///
  /// ```dart
  /// void paint(Canvas canvas, Size size) {
  ///   // 此例相对于上例渲染更差
  ///   Rect rect = Offset.zero & size;
  ///   canvas.save();
  ///   canvas.clipRRect(new RRect.fromRectXY(rect, 100.0, 100.0));
  ///   // 这里没有调用 canvas.saveLayer(rect, Paint());
  ///   canvas.drawPaint(new Paint()..color = Colors.red);
  ///   canvas.drawPaint(new Paint()..color = Colors.white);
  ///   canvas.restore();
  /// }
  /// ```
  ///
  /// 如果裁剪命令只裁剪一个绘制操作是没有意义的。例如，下面的绘制方法绘制一对白色圆角矩形，即使
  /// 裁剪不是在一个单独的图层上完成的:
  ///
  /// ```dart
  /// void paint(Canvas canvas, Size size) {
  ///   canvas.save();
  ///   canvas.clipRRect(new RRect.fromRectXY(Offset.zero & (size / 2.0), 50.0, 50.0));
  ///   canvas.drawPaint(new Paint()..color = Colors.white);
  ///   canvas.restore();
  ///   canvas.save();
  ///   canvas.clipRRect(new RRect.fromRectXY(
  ///              size.center(Offset.zero) & (size / 2.0), 50.0, 50.0));
  ///   canvas.drawPaint(new Paint()..color = Colors.white);
  ///   canvas.restore();
  /// }
  /// ```
  ///
  /// 顺带一提，与其使用[clipRRect]和[drawPaint]来绘制这样的圆形矩形，不如使用[drawRRect]方法。
  /// 这些示例使用[drawPaint]作为“会被剪切的复杂绘制操作”的代理，以说明这一点。)
  ///
  /// 通常来说, [saveLayer] 方法相对昂贵.
  void saveLayer(Rect bounds, Paint paint) {
    assert(paint != null);
    if (bounds == null) {
      _saveLayerWithoutBounds(paint._objects, paint._data);
    } else {
      assert(_rectIsValid(bounds));
      _saveLayer(bounds.left, bounds.top, bounds.right, bounds.bottom,
                 paint._objects, paint._data);
    }
  }
  void _saveLayerWithoutBounds(List<dynamic> paintObjects, ByteData paintData)
      native 'Canvas_saveLayerWithoutBounds';
  void _saveLayer(double left,
                  double top,
                  double right,
                  double bottom,
                  List<dynamic> paintObjects,
                  ByteData paintData) native 'Canvas_saveLayer';

  void restore() native 'Canvas_restore';

  /// 获取保存栈的层数
  int getSaveCount() native 'Canvas_getSaveCount';

  /// 平移画布
  void translate(double dx, double dy) native 'Canvas_translate';

  /// 画布缩放
  void scale(double sx, [double sy]) => _scale(sx, sy ?? sx);
  void _scale(double sx, double sy) native 'Canvas_scale';

  /// 画布旋转，角度是弧度正时针方向
  void rotate(double radians) native 'Canvas_rotate';

  /// 向当前转换中添加一个轴向倾斜的倾斜，第一个参数是围绕原点顺时针方向的上升/运行单位的水平倾斜，第二个
  /// 参数是围绕原点顺时针方向的上升/运行单位的垂直倾斜。
  void skew(double sx, double sy) native 'Canvas_skew';

  /// 矩阵乘
  void transform(Float64List matrix4) {
    assert(matrix4 != null);
    if (matrix4.length != 16)
      throw ArgumentError('"matrix4" must have 16 entries.');
    _transform(matrix4);
  }
  void _transform(Float64List matrix4) native 'Canvas_transform';

  /// 将裁剪区域缩减为当前裁剪与给定矩形的交集。
  /// 如果[doAntiAlias]为真，则裁剪将运用反锯齿。
  /// 如果多个绘制命令与剪辑边界相交，这可能会导致剪辑边界的混合不正确。参见[saveLayer]讨论如何解决这个
  /// 问题。
  /// 使用[ClipOp。从当前裁剪中减去提供的矩形（即方向裁剪）。
  void clipRect(Rect rect, { ClipOp clipOp = ClipOp.intersect, bool doAntiAlias = true }) {
    _clipRect(rect.left, rect.top, rect.right, rect.bottom, clipOp.index, doAntiAlias);
  }
  void _clipRect(double left,
                 double top,
                 double right,
                 double bottom,
                 int clipOp,
                 bool doAntiAlias) native 'Canvas_clipRect';

  /// 裁剪圆角矩形
  void clipRRect(RRect rrect, {bool doAntiAlias = true}) {
    _clipRRect(rrect._value32, doAntiAlias);
  }
  void _clipRRect(Float32List rrect, bool doAntiAlias) native 'Canvas_clipRRect';

  /// 用指定的Path来裁剪
  void clipPath(Path path, {bool doAntiAlias = true}) {
    assert(path != null); // path is checked on the engine side
    assert(doAntiAlias != null);
    _clipPath(path, doAntiAlias);
  }
  void _clipPath(Path path, bool doAntiAlias) native 'Canvas_clipPath';

  /// 将给定的[颜色]绘制到画布上，应用给定的[BlendMode]，给定的颜色是源，背景是目标。
  void drawColor(Color color, BlendMode blendMode) {
    _drawColor(color.value, blendMode.index);
  }
  void _drawColor(int color, int blendMode) native 'Canvas_drawColor';

  /// 绘制线条
  void drawLine(Offset p1, Offset p2, Paint paint) {
    _drawLine(p1.dx, p1.dy, p2.dx, p2.dy, paint._objects, paint._data);
  }
  void _drawLine(double x1,
                 double y1,
                 double x2,
                 double y2,
                 List<dynamic> paintObjects,
                 ByteData paintData) native 'Canvas_drawLine';

  /// 用给定的[Paint]填充画布。
  /// 要使用纯色和混合模式填充画布，可以考虑[drawColor]。
  void drawPaint(Paint paint) {
    assert(paint != null);
    _drawPaint(paint._objects, paint._data);
  }
  void _drawPaint(List<dynamic> paintObjects, ByteData paintData) native 'Canvas_drawPaint';

  /// 绘制矩形
  void drawRect(Rect rect, Paint paint) {
    assert(_rectIsValid(rect));
    assert(paint != null);
    _drawRect(rect.left, rect.top, rect.right, rect.bottom,
              paint._objects, paint._data);
  }
  void _drawRect(double left,
                 double top,
                 double right,
                 double bottom,
                 List<dynamic> paintObjects,
                 ByteData paintData) native 'Canvas_drawRect';

  /// 绘制圆角矩形
  void drawRRect(RRect rrect, Paint paint) {
    _drawRRect(rrect._value32, paint._objects, paint._data);
  }
  void _drawRRect(Float32List rrect,
                  List<dynamic> paintObjects,
                  ByteData paintData) native 'Canvas_drawRRect';

  /// 绘制由两个圆角矩形组成的图形，类似与一个环
  void drawDRRect(RRect outer, RRect inner, Paint paint) {
    assert(_rrectIsValid(outer));
    assert(_rrectIsValid(inner));
    assert(paint != null);
    _drawDRRect(outer._value32, inner._value32, paint._objects, paint._data);
  }
  void _drawDRRect(Float32List outer,
                   Float32List inner,
                   List<dynamic> paintObjects,
                   ByteData paintData) native 'Canvas_drawDRRect';

  /// 绘制椭圆
  void drawOval(Rect rect, Paint paint) {
    assert(_rectIsValid(rect));
    assert(paint != null);
    _drawOval(rect.left, rect.top, rect.right, rect.bottom,
              paint._objects, paint._data);
  }
  void _drawOval(double left,
                 double top,
                 double right,
                 double bottom,
                 List<dynamic> paintObjects,
                 ByteData paintData) native 'Canvas_drawOval';

  /// 绘制圆
  void drawCircle(Offset c, double radius, Paint paint) {
    assert(_offsetIsValid(c));
    assert(paint != null);
    _drawCircle(c.dx, c.dy, radius, paint._objects, paint._data);
  }
  void _drawCircle(double x,
                   double y,
                   double radius,
                   List<dynamic> paintObjects,
                   ByteData paintData) native 'Canvas_drawCircle';

  /// 在给定的矩形内画一个按比例缩放的圆弧。它开始从startAngle弧度的椭圆形到startAngle + sweepAngle
  /// 弧度椭圆形,零弧度是在椭圆的右手边与横线相交的点，横线与矩形的中心相交，以正角围绕椭圆顺时针旋转。
  /// 如果useCenter为真，则弧被封闭回中心，形成一个圆扇形。否则，弧是不闭合的，形成一个圆弧。
  ///
  /// 比[Path.arcTo]效率高
  void drawArc(Rect rect, double startAngle, double sweepAngle, bool useCenter, Paint paint) {
    _drawArc(rect.left, rect.top, rect.right, rect.bottom, startAngle,
             sweepAngle, useCenter, paint._objects, paint._data);
  }
  void _drawArc(double left,
                double top,
                double right,
                double bottom,
                double startAngle,
                double sweepAngle,
                bool useCenter,
                List<dynamic> paintObjects,
                ByteData paintData) native 'Canvas_drawArc';

  /// 用给定的[Paint]绘制给定的[Path]。这个形状是填充还是描边(或者两者都是)由[Paint.style]控制。
  /// 如果路径已被填充，则其中的子路径将隐式关闭(参见[path .close])。
  void drawPath(Path path, Paint paint) {
    assert(path != null); // path is checked on the engine side
    assert(paint != null);
    _drawPath(path, paint._objects, paint._data);
  }
  void _drawPath(Path path,
                 List<dynamic> paintObjects,
                 ByteData paintData) native 'Canvas_drawPath';

  /// 将给定的[Image]绘制到画布中，其左上角位于给定的偏移处。使用给定的[Paint]将图像合成到画布中。
  void drawImage(Image image, Offset p, Paint paint) {
    _drawImage(image, p.dx, p.dy, paint._objects, paint._data);
  }
  void _drawImage(Image image,
                  double x,
                  double y,
                  List<dynamic> paintObjects,
                  ByteData paintData) native 'Canvas_drawImage';

  /// 将“src”参数描述的给定图像的子集绘制到画布上的“dst”参数给出的轴向矩形中。
  ///
  /// 这可能从外部采样的' src ' rect最多一半宽度的应用过滤器。
  ///
  /// 使用不同的参数(来自相同的映像)对这个方法的多次调用可以批量处理为对[drawAtlas]的单个调用，以提高性
  /// 能。
  void drawImageRect(Image image, Rect src, Rect dst, Paint paint) {
    _drawImageRect(image,
                   src.left,
                   src.top,
                   src.right,
                   src.bottom,
                   dst.left,
                   dst.top,
                   dst.right,
                   dst.bottom,
                   paint._objects,
                   paint._data);
  }
  void _drawImageRect(Image image,
                      double srcLeft,
                      double srcTop,
                      double srcRight,
                      double srcBottom,
                      double dstLeft,
                      double dstTop,
                      double dstRight,
                      double dstBottom,
                      List<dynamic> paintObjects,
                      ByteData paintData) native 'Canvas_drawImageRect';

  /// 绘制.9图  
  /// 使用给定的[油漆]将给定的[图像]绘制到画布中。
  ///
  /// 图像由9个部分组成，通过绘制两条水平线和两条垂直线来分割图像，其中“center”参数描述了由这四条线相交
  /// 的四个点构成的矩形。(这形成了一个3×3的区域网格，中心区域由“center”参数描述。)
  ///
  /// 在没有缩放的情况下，在由“dst”描述的目标矩形的四个角上绘制四个区域。剩下的5个区域是通过拉伸来适应
  /// 的，这样它们正好覆盖目标矩形，同时保持它们的相对位置。
  void drawImageNine(Image image, Rect center, Rect dst, Paint paint) {
    assert(image != null); // image is checked on the engine side
    assert(_rectIsValid(center));
    assert(_rectIsValid(dst));
    assert(paint != null);
    _drawImageNine(image,
                   center.left,
                   center.top,
                   center.right,
                   center.bottom,
                   dst.left,
                   dst.top,
                   dst.right,
                   dst.bottom,
                   paint._objects,
                   paint._data);
  }
  void _drawImageNine(Image image,
                      double centerLeft,
                      double centerTop,
                      double centerRight,
                      double centerBottom,
                      double dstLeft,
                      double dstTop,
                      double dstRight,
                      double dstBottom,
                      List<dynamic> paintObjects,
                      ByteData paintData) native 'Canvas_drawImageNine';

  /// 绘制一个 Picture
  void drawPicture(Picture picture) {
    assert(picture != null); // picture is checked on the engine side
    _drawPicture(picture);
  }
  void _drawPicture(Picture picture) native 'Canvas_drawPicture';

  /// 将给定[Paragraph]中的文本以给定[Offset]绘制到此画布中。
  ///
  /// [Paragraph]的[Paragraph.layout]必须先被调用
  ///
  /// 要对齐文本，可以在传递给[ParagraphBuilder]函数的[ParagraphStyle]对象上设置' textAlign '。
  void drawParagraph(Paragraph paragraph, Offset offset) {
    paragraph._paint(this, offset.dx, offset.dy);
  }

  /// 绘制一系列的点
  void drawPoints(PointMode pointMode, List<Offset> points, Paint paint) {
    _drawPoints(paint._objects, paint._data, pointMode.index, _encodePointList(points));
  }

  /// 根据给定的[PointMode]来绘制一系列的点
  void drawRawPoints(PointMode pointMode, Float32List points, Paint paint) {
    assert(pointMode != null);
    assert(points != null);
    assert(paint != null);
    if (points.length % 2 != 0)
      throw ArgumentError('"points" must have an even number of values.');
    _drawPoints(paint._objects, paint._data, pointMode.index, points);
  }

  void _drawPoints(List<dynamic> paintObjects,
                   ByteData paintData,
                   int pointMode,
                   Float32List points) native 'Canvas_drawPoints';

  void drawVertices(Vertices vertices, BlendMode blendMode, Paint paint) {
    assert(vertices != null); // vertices is checked on the engine side
    assert(paint != null);
    assert(blendMode != null);
    _drawVertices(vertices, blendMode.index, paint._objects, paint._data);
  }
  void _drawVertices(Vertices vertices,
                     int blendMode,
                     List<dynamic> paintObjects,
                     ByteData paintData) native 'Canvas_drawVertices';

  //
  // See also:
  //
  //  * [drawRawAtlas], which takes its arguments as typed data lists rather
  //    than objects.
  void drawAtlas(Image atlas,
                 List<RSTransform> transforms,
                 List<Rect> rects,
                 List<Color> colors,
                 BlendMode blendMode,
                 Rect cullRect,
                 Paint paint) {
    assert(atlas != null); // atlas is checked on the engine side
    assert(transforms != null);
    assert(rects != null);
    assert(colors != null);
    assert(blendMode != null);
    assert(paint != null);

    final int rectCount = rects.length;
    if (transforms.length != rectCount)
      throw ArgumentError('"transforms" and "rects" lengths must match.');
    if (colors.isNotEmpty && colors.length != rectCount)
      throw ArgumentError('If non-null, "colors" length must match that of "transforms" and "rects".');

    final Float32List rstTransformBuffer = Float32List(rectCount * 4);
    final Float32List rectBuffer = Float32List(rectCount * 4);

    for (int i = 0; i < rectCount; ++i) {
      final int index0 = i * 4;
      final int index1 = index0 + 1;
      final int index2 = index0 + 2;
      final int index3 = index0 + 3;
      final RSTransform rstTransform = transforms[i];
      final Rect rect = rects[i];
      assert(_rectIsValid(rect));
      rstTransformBuffer[index0] = rstTransform.scos;
      rstTransformBuffer[index1] = rstTransform.ssin;
      rstTransformBuffer[index2] = rstTransform.tx;
      rstTransformBuffer[index3] = rstTransform.ty;
      rectBuffer[index0] = rect.left;
      rectBuffer[index1] = rect.top;
      rectBuffer[index2] = rect.right;
      rectBuffer[index3] = rect.bottom;
    }

    final Int32List colorBuffer = colors.isEmpty ? null : _encodeColorList(colors);
    final Float32List cullRectBuffer = cullRect?._value32;

    _drawAtlas(
      paint._objects, paint._data, atlas, rstTransformBuffer, rectBuffer,
      colorBuffer, blendMode.index, cullRectBuffer
    );
  }

  //
  // The `rstTransforms` argument is interpreted as a list of four-tuples, with
  // each tuple being ([RSTransform.scos], [RSTransform.ssin],
  // [RSTransform.tx], [RSTransform.ty]).
  //
  // The `rects` argument is interpreted as a list of four-tuples, with each
  // tuple being ([Rect.left], [Rect.top], [Rect.right], [Rect.bottom]).
  //
  // The `colors` argument, which can be null, is interpreted as a list of
  // 32-bit colors, with the same packing as [Color.value].
  //
  // See also:
  //
  //  * [drawAtlas], which takes its arguments as objects rather than typed
  //    data lists.
  void drawRawAtlas(Image atlas,
                    Float32List rstTransforms,
                    Float32List rects,
                    Int32List colors,
                    BlendMode blendMode,
                    Rect cullRect,
                    Paint paint) {
    assert(atlas != null); // atlas is checked on the engine side
    assert(rstTransforms != null);
    assert(rects != null);
    assert(colors != null);
    assert(blendMode != null);
    assert(paint != null);

    final int rectCount = rects.length;
    if (rstTransforms.length != rectCount)
      throw ArgumentError('"rstTransforms" and "rects" lengths must match.');
    if (rectCount % 4 != 0)
      throw ArgumentError('"rstTransforms" and "rects" lengths must be a multiple of four.');
    if (colors != null && colors.length * 4 != rectCount)
      throw ArgumentError('If non-null, "colors" length must be one fourth the length of "rstTransforms" and "rects".');

    _drawAtlas(
      paint._objects, paint._data, atlas, rstTransforms, rects,
      colors, blendMode.index, cullRect?._value32
    );
  }

  void _drawAtlas(List<dynamic> paintObjects,
                  ByteData paintData,
                  Image atlas,
                  Float32List rstTransforms,
                  Float32List rects,
                  Int32List colors,
                  int blendMode,
                  Float32List cullRect) native 'Canvas_drawAtlas';

  /// Draws a shadow for a [Path] representing the given material elevation.
  ///
  /// The `transparentOccluder` argument should be true if the occluding object
  /// is not opaque.
  ///
  /// The arguments must not be null.
  void drawShadow(Path path, Color color, double elevation, bool transparentOccluder) {
    assert(path != null); // path is checked on the engine side
    assert(color != null);
    assert(transparentOccluder != null);
    _drawShadow(path, color.value, elevation, transparentOccluder);
  }
  void _drawShadow(Path path,
                   int color,
                   double elevation,
                   bool transparentOccluder) native 'Canvas_drawShadow';
}
```


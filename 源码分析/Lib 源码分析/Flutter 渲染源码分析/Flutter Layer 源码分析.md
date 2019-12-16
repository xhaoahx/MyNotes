# Flutter Layer 源码分析

[toc]

```dart
class AnnotationEntry<T> {
  const AnnotationEntry({
    @required this.annotation,
    @required this.localPosition,
  }) : assert(localPosition != null);

  /// 找到的 Annotation 对象.
  final T annotation;

  /// 包含注释的层的相对坐标空间
  final Offset localPosition;

  @override
  String toString() {
    return '$runtimeType(annotation: $annotation, localPostion: $localPosition)';
  }
}
```
## AnnotationResult

```dart
/// 关于在层树中找到的注释列表的收集信息。
class AnnotationResult<T> {
  final List<AnnotationEntry<T>> _entries = <AnnotationEntry<T>>[];

  void add(AnnotationEntry<T> entry) => _entries.add(entry);

  Iterable<AnnotationEntry<T>> get entries => _entries;

  Iterable<T> get annotations sync* {
    for (AnnotationEntry<T> entry in _entries)
      yield entry.annotation;
  }
}
```

## Layer
```dart
/// 合成层，以下简称为层
///
/// 在绘制过程中，渲染树生成一个由合成层组成的树，这些层被上传到引擎中并由合成器显示。这个类是所有复合层的
/// 基类。
///
/// 大多数层都可以改变它们的属性，层可以移动到不同的父层。当以上的变化发生后，[Scene]必须直接地重新合成。
/// 层树不维护自己的脏状态。
///
/// 为了合成层树，需要创建一个[SceneBuilder]对象，将其传递给根[Layer]对象的[addToScene]方法，然后调
/// 用 [SceneBuilder]。建造以获得一个[Scene]。然后可以使用[Window.render]绘制一个[Scene]。

/// 关于 Windows.render:
/// 用新提供的[Scene]在GPU上更新应用的渲染。这个函数必须在被调用的[onBeginFrame]或[onDrawFrame]回调
/// 的范围内调用。如果这个函数在单个[onBeginFrame]/[onDrawFrame]回调序列中第二次被调用，或者在那些回
/// 调的范围之外被调用，这个调用将被忽略。
///
/// 为了记录图形化操作，首先创建一个[PictureRecorder]，然后构造一个[Canvas]，并将该
/// [PictureRecorder] 传递给它的构造函数。
/// 发出所有图形操作之后，调用[PictureRecorder。在[PictureRecorder]上执行[endRecorderer]功能，以
/// 获得最终呈现已发出的图形操作的[Pictrue]。
/// 
/// 接下来，创建一个[SceneBuilder]，并使用[SceneBuilder.addPicture]将[Picture]添加到其中。
/// 通过[SceneBuilder.build]方法，你可以获得一个[Sence]对象，你可以调用这个 render 函数将其呈现给用
/// 户。
///
/// void render(Scene scene) native 'Window_render';
///
abstract class Layer extends AbstractNode with DiagnosticableTreeMixin {
  /// This layer's parent in the layer tree.
  ///
  /// The [parent] of the root node in the layer tree is null.
  ///
  /// Only subclasses of [ContainerLayer] can have children in the layer tree.
  /// All other layer classes are used for leaves in the layer tree.
  @override
  ContainerLayer get parent => super.parent;

  // 该层自上次调用[addToScene]后是否有任何变化。
  // 初始化为true，因为新层从未调用[addToScene]，在调用[addToScene]后设置为false。如果  
  // [markNeedsAddToScene]被调用，或者[updateSubtreeNeedsAddToScene]在该层或在祖先层上被调用，则
  // 该值可以再次变为true。
  // 如果树中的每一层都满足以下条件，则表示树中[_needsAddToScene]的值是一致的:
  //
  // - 如果[alwaysNeedsAddToScene]为 true,则[_needsAddToScene]也是 true
  // - 如果[_needsAddToScene]为 true 且[parent]非 null,则 `parent._needsAddToScene`为 true.
  //
  // 通常，这个值是在绘制阶段和合成期间设置的。
  // 在绘制阶段，[RenderObject]创建新的层，并在现有层上调用[markNeedsAddToScene]，使这个值变为真。
  // 在绘制阶段之后，树可能处于不一致的状态。
  // 在合成阶段，[ContainerLayer.buildScene]首先调用[updateSubTreeNeedsAddToScene]使树达到一致
  // 状态，然后调用[addToScene]，最后将该字段设置为 false。
  bool _needsAddToScene = true;

  /// 标记这一层发生了改变，且[addToScene]需要被调用
  @protected
  @visibleForTesting
  void markNeedsAddToScene() {
    if (_needsAddToScene) {
      return;
    }
    _needsAddToScene = true;
  }

  /// 子类可以将此重写为 true 以禁用 维持渲染状态.
  @protected
  bool get alwaysNeedsAddToScene => false;

  /// 存储为这一层创建的引擎层，以便跨帧重用引擎资源以获得更好的应用程序性能。
  ///   
  /// 此值可以传递给[ui.SceneBuilder]，通知引擎这一层或它的任何后代都没有改变。
  /// 例如，native 引擎可以重用在前一帧中呈现的纹理。
  /// 例如，web 引擎可以重用为前一帧创建的HTML DOM节点。
  ///
  /// 此值可以作为“oldLayer”参数传递给“push”方法，通知引擎某个层正在更新先前已经渲染的层。
  /// 例如，web引擎可以更新以前呈现的HTML DOM节点的属性，而不是创建新节点。
  @protected
  ui.EngineLayer get engineLayer => _engineLayer;

  /// 设置用于渲染该层的引擎层.
  ///
  /// Typically this field is set to the value returned by [addToScene], which
  /// in turn returns the engine layer produced by one of [ui.SceneBuilder]'s
  /// "push" methods, such as [ui.SceneBuilder.pushOpacity].
  /// 通常，该字段被设置为[addToScene]返回的值，该字段随后返回由[ui.SceneBuilder]的“push”方法之一产
  /// 生的引擎层，例如[ui.SceneBuilder.pushOpacity]。
  /// OpacityEngineLayer pushOpacity(...){}  
  @protected
  set engineLayer(ui.EngineLayer value) {
    _engineLayer = value;
    if (!alwaysNeedsAddToScene) {
      // 父类必须构造一个新的引擎层来添加这个层，因此我们将它标记为需要[addToScene]。
	  // 这是为了处理两种情况:
      // 1.当渲染完整的图层树时。在本例中，我们首先调用子节点的 addToScene 方法，然后调用 父节点的 set 
      // engineLayer。子节点会在父节点上调用 markNeedsAddToScene 来表示它们产生了新的引擎层，因此
      // 父节点需要被更新。在本例中，父类已经通过 [addToScene] 将自己添加到场景中，因此在它完成之后，它
      // 的 set engineLayer  被调用，并清除 _needsAddToScene 标志。
	  // 2.当渲染一个内层时(例如“OffsetLayer.toImage”)。在这种情况下，我们为其中一个子节点而不是父节 
      // 点调用 addToScene ，也就是说，我们为子节点生成新引擎层，但不为父节点生成新引擎层。在这里，子节
      // 点将标记父节点需要 addToScene ，但是父节点直到将来的某个帧决定渲染它时才清除标记，此时父节点知
      // 道它不能保留它的引擎层并将再次调用“addToScene”.
      if (parent != null && !parent.alwaysNeedsAddToScene) {
        parent.markNeedsAddToScene();
      }
    }
  }
  ui.EngineLayer _engineLayer;

  /// 遍历这一层的子树并且决定它是否需要 [addToScene]
  /// 当以下条件为真时，一个层需要 [addToScene]
  ///
  /// 1. [alwaysNeedsAddToScene] is true.
  /// 2. [markNeedsAddToScene] has been called.
  /// 3. Any of its descendants need [addToScene].
  ///
  /// [ContainerLayer] 重载了这个方法，递归地调用其子层的这个方法.
  @protected
  @visibleForTesting
  void updateSubtreeNeedsAddToScene() {
    _needsAddToScene = _needsAddToScene || alwaysNeedsAddToScene;
  }

  /// 以下提供了链表实现的兄弟节点管理方法
  Layer get nextSibling => _nextSibling;
  Layer _nextSibling;

  Layer get previousSibling => _previousSibling;
  Layer _previousSibling;

  /// 当添加或者移除孩子节点的是否需要 markNeedsAddToScene() 
  @override
  void dropChild(AbstractNode child) {
    if (!alwaysNeedsAddToScene) {
      markNeedsAddToScene();
    }
    super.dropChild(child);
  }

  @override
  void adoptChild(AbstractNode child) {
    if (!alwaysNeedsAddToScene) {
      markNeedsAddToScene();
    }
    super.adoptChild(child);
  }

  /// 将这个层从它的父节点的孩子列表中移除.
  @mustCallSuper
  void remove() {
    parent?._removeChild(this);
  }

  /// 在这一层及其子树中搜索“localPosition”描述的位置的“S”类型的注释。
  ///
  /// 此方法由[find]和[findAllAnnotations]的默认实现调用。重写此方法以自定义该层如何搜索注释，或者该
  /// 层是否要添加自己的注释。
  ///
  /// 默认实现只返回“false”，这意味着该层及其子层都没有注释，并且注释搜索也不会被吸收(参见下面的解释)。
  ///
  /// ## 关于层注释
  ///
  /// {@template flutter.rendering.layer.findAnnotations.aboutAnnotations}
  /// 注释是任何类型的可选对象，可以与层一起携带。只要所有者层包含位置并被步行到该位置，就可以在该位置找到
  /// 注释。
  ///
  /// 通过先递归地访问每个子元素，最后时本层来搜索注释，从而形成从前面到后面的顺序。
  /// 注释必须满足给定的限制，比如类型和位置。
  ///
  /// 在这里找到一个值的常用方法是将[AnnotatedRegionLayer]加入层树，或者通过重载[findannotation]来
  /// 添加所需的注释。
  ///
  /// 返回值指示此层及其子树在此位置的不透明度。如果返回true，那么这一层的父层应该跳过这一层后面的子层。换
  /// 句话说，它对这种类型的注释是不透明的，并且吸收了搜索，因此它后面的兄弟姐妹不会知道搜索。如果返回值为
  /// false，则父进程可能继续执行其他兄弟进程。
  ///
  /// 返回值不影响父类是否添加自己的注释。换句话说，如果一个层要添加一个注释，那么它总是会添加它，即使它的
  /// 子层对于这种类型的注释是不透明的。然而，父类返回的不透明性可能会受到其子类的影响，因此使得它的所有祖
  /// 先对于这种类型的注释都是不透明的。
  @protected
  bool findAnnotations<S>(
    AnnotationResult<S> result,
    Offset localPosition, {
    @required bool onlyFirst,
  }) {
    return false;
  }

  /// 在这个层及其子树中搜索'localPosition'描述的点下的第一个' S '类型的注释.
  /// 没有找到任何值的话返回 null
  S find<S>(Offset localPosition) {
    final AnnotationResult<S> result = AnnotationResult<S>();
    findAnnotations<S>(result, localPosition, onlyFirst: true);
    return result.entries.isEmpty ? null : result.entries.first.annotation;
  }

  /// Search this layer and its subtree for all annotations of type `S` under
  /// the point described by `localPosition`.
  ///
  /// Returns a result with empty entries if no matching annotations are found.
  ///
  /// By default this method simply calls [findAnnotations] with `onlyFirst:
  /// false` and returns the annotations of its result. Prefer overriding
  /// [findAnnotations] instead of this method, because during an annotation
  /// search, only [findAnnotations] is recursively called, while custom
  /// behavior in this method is ignored.
  ///
  /// ## About layer annotations
  ///
  /// {@macro flutter.rendering.layer.findAnnotations.aboutAnnotations}
  ///
  /// See also:
  ///
  ///  * [find], which is similar but returns the first annotation found at the
  ///    given position.
  ///  * [findAllAnnotations], which is similar but returns an
  ///    [AnnotationResult], which contains more information, such as the local
  ///    position of the event related to each annotation, and is equally fast,
  ///    hence is preferred over [findAll].
  ///  * [AnnotatedRegionLayer], for placing values in the layer tree.
  @Deprecated(
    'Use findAllAnnotations(...).annotations instead. '
    'This feature was deprecated after v1.10.14.'
  )
  Iterable<S> findAll<S>(Offset localPosition) {
    final AnnotationResult<S> result = findAllAnnotations(localPosition);
    return result.entries.map((AnnotationEntry<S> entry) => entry.annotation);
  }

  /// 搜索这一层和它的子树，在' localPosition '描述的点下查找所有类型为' S '的注释。
  /// 如果没有找到匹配的注释，则返回一个带有空条目的结果。
  /// 默认情况下，这个方法简单地调用带有' onlyFirst: false '的[findannotation]，并返回其结果的注
  /// 释。最好重写[findannotation]而不是这个方法，因为在注释搜索期间，只递归调用[findannotation]，而
  /// 忽略此方法中的自定义行为。
  AnnotationResult<S> findAllAnnotations<S>(Offset localPosition) {
    final AnnotationResult<S> result = AnnotationResult<S>();
    findAnnotations<S>(result, localPosition, onlyFirst: false);
    return result;
  }

  /// 重载这一方法将本 layer 上传给引擎
  ///
  /// 返回保留渲染结果的引擎层。当没有相应的引擎层时，返回null。
  @protected
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]);

  void _addToSceneWithRetainedRendering(ui.SceneBuilder builder) {
    // 不可能通过添加一个保留层子树的_needsAddToScene为假来实现循环。
    // 反证法:
    // 如果我们引入一个循环，这个保留的层必须被附加到它的一个子代层，比如a。这意味着a的子结构已经改变，所
    // 以a 的_needsAddToScene是真的。这与“必须是假的”相矛盾。
    if (!_needsAddToScene && _engineLayer != null) {
      builder.addRetained(_engineLayer);
      return;
    }
    addToScene(builder);
    _needsAddToScene = false;
  }
}

```


## 1.PictureLayer（图片层）

```dart
/// 包含了 [Picture] 的合成层，总是层树的叶子节点
class PictureLayer extends Layer {
  PictureLayer(this.canvasBounds);

  /// The bounds that were used for the canvas that drew this layer's [picture].
  ///
  /// This is purely advisory. It is included in the information dumped with
  /// [debugDumpLayerTree] (which can be triggered by pressing "L" when using
  /// "flutter run" at the console), which can help debug why certain drawing
  /// commands are being culled.
  final Rect canvasBounds;

  /// The picture recorded for this layer.
  ///
  /// picture 的坐标系统符合层的坐标系统
  ui.Picture get picture => _picture;
  ui.Picture _picture;
  set picture(ui.Picture picture) {
    markNeedsAddToScene();
    _picture = picture;
  }

  /// 提示此层中的绘制是十分复杂的，并将受益于缓存。
  ///
  /// 如果未设置此提示，则组合程序将应用自己的启发式方法来确定此层是否足够复杂，以便从缓存中获益。
  bool get isComplexHint => _isComplexHint;
  bool _isComplexHint = false;
  set isComplexHint(bool value) {
    if (value != _isComplexHint) {
      _isComplexHint = value;
      markNeedsAddToScene();
    }
  }

  /// 暗示这一层的绘制可能会改变下一帧。
  ///
  /// 这个提示告诉合成程序不要缓存这一层，因为缓存在将来不会被使用。
  /// 如果未设置此提示，则组合程序将应用自己的启发式方法来决定将来是否可能重用此层。
  bool get willChangeHint => _willChangeHint;
  bool _willChangeHint = false;
  set willChangeHint(bool value) {
    if (value != _willChangeHint) {
      _willChangeHint = value;
      markNeedsAddToScene();
    }
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    builder.addPicture(
        layerOffset, 
        picture, 
        isComplexHint: isComplexHint, 
        willChangeHint: willChangeHint
    );
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { 
      @required bool onlyFirst 
  }) {
    return false;
  }
}
```


## 2.TextureLayer（纹理层）

```dart
/// 将后台纹理映射到矩形的合成层
///
/// 后台纹理是可以应用(映射)到Flutter视图区域的图像。它们是使用特定于平台的纹理注册表创建、管理和更新的。
/// 这通常由与 主机平台视频播放器、摄像机或OpenGL api或类似的图像源 集成的插件完成。
///
/// 纹理层使用一个整数ID来表示它的后台纹理。纹理ID从纹理注册表中获得，作用域设置为Flutter视图。纹理ID可
/// 以在销毁后重新使用，由注册表决定。使用当前不为注册表中纹理id将导致一个空白矩形。
///
/// 一旦插入到层树中，纹理层就会根据后台指令自动重新绘制(例如在到达视频帧时)。
/// 这种重绘通常不涉及执行Dart代码。
///
/// 纹理层总是层树的叶子节点

class TextureLayer extends Layer {
  TextureLayer({
    @required this.rect,
    @required this.textureId,
    this.freeze = false,
  }) : assert(rect != null),
       assert(textureId != null);

  /// 这层纹理的边界
  final Rect rect;

  /// 纹理ID
  final int textureId;

  /// 当为 true 时，纹理将不会被更新为新的帧。
 
  /// 在调整嵌入式Android视图大小的时候使用:当调整大小时框架有一个短的期间不能确认最新的纹理帧具有是否有
  /// 先前的或新的大小,为了解决这个问题，框架在调整Android视图的大小之前冻结它，并且在当一个确切的大小被
  /// 确定之后解冻
  final bool freeze;

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    final Rect shiftedRect = layerOffset == Offset.zero ? rect : rect.shift(layerOffset);
    builder.addTexture(
      textureId,
      offset: shiftedRect.topLeft,
      width: shiftedRect.width,
      height: shiftedRect.height,
      freeze: freeze,
    );
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { 
      @required bool onlyFirst }) {
    return false;
  }
}
```
## 3.PlatformViewLayer（平台视图层）
```dart
/// 用于呈现内嵌的平台视图的层
class PlatformViewLayer extends Layer {

  PlatformViewLayer({
    @required this.rect,
    @required this.viewId,
  }) : assert(rect != null),
       assert(viewId != null);

  final Rect rect;

  final int viewId;

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    final Rect shiftedRect = layerOffset == Offset.zero ? rect : rect.shift(layerOffset);
    builder.addPlatformView(
      viewId,
      offset: shiftedRect.topLeft,
      width: shiftedRect.width,
      height: shiftedRect.height,
    );
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { @required bool onlyFirst }) {
    return false;
  }
}
```
## 4.PerformanceOverlayLayer（性能展示图层层）
```dart
class PerformanceOverlayLayer extends Layer {
  /// Creates a layer that displays a performance overlay.
  PerformanceOverlayLayer({
    @required Rect overlayRect,
    @required this.optionsMask,
    @required this.rasterizerThreshold,
    @required this.checkerboardRasterCacheImages,
    @required this.checkerboardOffscreenLayers,
  }) : _overlayRect = overlayRect;

  /// The rectangle in this layer's coordinate system that the overlay should occupy.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  Rect get overlayRect => _overlayRect;
  Rect _overlayRect;
  set overlayRect(Rect value) {
    if (value != _overlayRect) {
      _overlayRect = value;
      markNeedsAddToScene();
    }
  }

  /// The mask is created by shifting 1 by the index of the specific
  /// [PerformanceOverlayOption] to enable.
  final int optionsMask;

  /// The rasterizer threshold is an integer specifying the number of frame
  /// intervals that the rasterizer must miss before it decides that the frame
  /// is suitable for capturing an SkPicture trace for further analysis.
  final int rasterizerThreshold;

  /// Whether the raster cache should checkerboard cached entries.
  ///
  /// The compositor can sometimes decide to cache certain portions of the
  /// widget hierarchy. Such portions typically don't change often from frame to
  /// frame and are expensive to render. This can speed up overall rendering. However,
  /// there is certain upfront cost to constructing these cache entries. And, if
  /// the cache entries are not used very often, this cost may not be worth the
  /// speedup in rendering of subsequent frames. If the developer wants to be certain
  /// that populating the raster cache is not causing stutters, this option can be
  /// set. Depending on the observations made, hints can be provided to the compositor
  /// that aid it in making better decisions about caching.
  final bool checkerboardRasterCacheImages;

  /// Whether the compositor should checkerboard layers that are rendered to offscreen
  /// bitmaps. This can be useful for debugging rendering performance.
  ///
  /// Render target switches are caused by using opacity layers (via a [FadeTransition] or
  /// [Opacity] widget), clips, shader mask layers, etc. Selecting a new render target
  /// and merging it with the rest of the scene has a performance cost. This can sometimes
  /// be avoided by using equivalent widgets that do not require these layers (for example,
  /// replacing an [Opacity] widget with an [widgets.Image] using a [BlendMode]).
  final bool checkerboardOffscreenLayers;

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(optionsMask != null);
    final Rect shiftedOverlayRect = layerOffset == Offset.zero ? overlayRect : overlayRect.shift(layerOffset);
    builder.addPerformanceOverlay(optionsMask, shiftedOverlayRect);
    builder.setRasterizerTracingThreshold(rasterizerThreshold);
    builder.setCheckerboardRasterCacheImages(checkerboardRasterCacheImages);
    builder.setCheckerboardOffscreenLayers(checkerboardOffscreenLayers);
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { @required bool onlyFirst }) {
    return false;
  }
}
```
## 5.CountainerLayer（容器层）
```dart
/// 拥有孩子列表的合成层
///
/// 一个[ContainerLayer]实例仅仅获取一组子元素并将它们按顺序插入到合成渲染中。
/// [ContainerLayer]的子类在这个过程中应用更复杂的效果。
class ContainerLayer extends Layer {
  /// 以下提供链表的管理方式  
  Layer get firstChild => _firstChild;
  Layer _firstChild;

  Layer get lastChild => _lastChild;
  Layer _lastChild;

  /// Returns whether this layer has at least one child layer.
  bool get hasChildren => _firstChild != null;

  /// 考虑将这个层作为层树的根，并在引擎中构建一个场景(一个层树)。
  /// 这个方法是在“ContainerLayer”类而不是“PipelineOwner”或其他单例级别的原因是：这个方法既可以用来
  /// 渲染整个层树(例如一个普通的应用程序框架)，也可以用来渲染一棵子树(例如“OffsetLayer.toImage”)。
  ui.Scene buildScene(ui.SceneBuilder builder) {
    List<PictureLayer> temporaryLayers;
    updateSubtreeNeedsAddToScene();
    addToScene(builder);
    _needsAddToScene = false;
    final ui.Scene scene = builder.build();
    return scene;
  }

  PictureLayer _highlightConflictingLayer(PhysicalModelLayer child) {
    final ui.PictureRecorder recorder = ui.PictureRecorder();
    final Canvas canvas = Canvas(recorder);
    canvas.drawPath(
      child.clipPath,
      Paint()
        ..color = const Color(0xFFAA0000)
        ..style = PaintingStyle.stroke
        // The elevation may be 0 or otherwise too small to notice.
        // Adding 10 to it makes it more visually obvious.
        ..strokeWidth = child.elevation + 10.0,
    );
    final PictureLayer pictureLayer = PictureLayer(child.clipPath.getBounds())
      ..picture = recorder.endRecording()
      ..debugCreator = child;
    child.append(pictureLayer);
    return pictureLayer;
  }

  List<PictureLayer> _processConflictingPhysicalLayers(PhysicalModelLayer predecessor, PhysicalModelLayer child) {
    return <PictureLayer>[
      _highlightConflictingLayer(predecessor),
      _highlightConflictingLayer(child),
    ];
  }

  /// 先递归调用孩子节点的 updateSubtreeNeedsAddToScene
  /// 当任何一个孩子节点的的 _needsAddToScene 为 true 时，标记自身的 _needsAddToScene
  @override
  void updateSubtreeNeedsAddToScene() {
    super.updateSubtreeNeedsAddToScene();
    Layer child = firstChild;
    while (child != null) {
      child.updateSubtreeNeedsAddToScene();
      _needsAddToScene = _needsAddToScene || child._needsAddToScene;
      child = child.nextSibling;
    }
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { 
      @required bool onlyFirst 
  }) {
    for (Layer child = lastChild; child != null; child = child.previousSibling) {
      final bool isAbsorbed = 
          child.findAnnotations<S>(result, localPosition, onlyFirst: onlyFirst);
      if (isAbsorbed)
        return true;
      if (onlyFirst && result.entries.isNotEmpty)
        return isAbsorbed;
    }
    return false;
  }

  @override
  void attach(Object owner) {
    super.attach(owner);
    Layer child = firstChild;
    while (child != null) {
      child.attach(owner);
      child = child.nextSibling;
    }
  }

  @override
  void detach() {
    super.detach();
    Layer child = firstChild;
    while (child != null) {
      child.detach();
      child = child.nextSibling;
    }
  }

  /// 增添子节点  
  void append(Layer child) {
    adoptChild(child);
    child._previousSibling = lastChild;
    if (lastChild != null)
      lastChild._nextSibling = child;
    _lastChild = child;
    _firstChild ??= child;
    assert(child.attached == attached);
  }

  void _removeChild(Layer child) {
    if (child._previousSibling == null) {
      _firstChild = child._nextSibling;
    } else {
      child._previousSibling._nextSibling = child.nextSibling;
    }
    if (child._nextSibling == null) {
      _lastChild = child.previousSibling;
    } else {
      child.nextSibling._previousSibling = child.previousSibling;
    }
    child._previousSibling = null;
    child._nextSibling = null;
    dropChild(child);
  }

  void removeAllChildren() {
    Layer child = firstChild;
    while (child != null) {
      final Layer next = child.nextSibling;
      child._previousSibling = null;
      child._nextSibling = null;
      assert(child.attached == attached);
      dropChild(child);
      child = next;
    }
    _firstChild = null;
    _lastChild = null;
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    addChildrenToScene(builder, layerOffset);
  }

  /// Uploads all of this layer's children to the engine.
  ///
  /// This method is typically used by [addToScene] to insert the children into
  /// the scene. Subclasses of [ContainerLayer] typically override [addToScene]
  /// to apply effects to the scene using the [SceneBuilder] API, then insert
  /// their children using [addChildrenToScene], then reverse the aforementioned
  /// effects before returning from [addToScene].
  void addChildrenToScene(ui.SceneBuilder builder, [ Offset childOffset = Offset.zero ]) {
    Layer child = firstChild;
    while (child != null) {
      if (childOffset == Offset.zero) {
        child._addToSceneWithRetainedRendering(builder);
      } else {
        child.addToScene(builder, childOffset);
      }
      child = child.nextSibling;
    }
  }

  /// Applies the transform that would be applied when compositing the given
  /// child to the given matrix.
  ///
  /// Specifically, this should apply the transform that is applied to child's
  /// _origin_. When using [applyTransform] with a chain of layers, results will
  /// be unreliable unless the deepest layer in the chain collapses the
  /// `layerOffset` in [addToScene] to zero, meaning that it passes
  /// [Offset.zero] to its children, and bakes any incoming `layerOffset` into
  /// the [SceneBuilder] as (for instance) a transform (which is then also
  /// included in the transformation applied by [applyTransform]).
  ///
  /// For example, if [addToScene] applies the `layerOffset` and then
  /// passes [Offset.zero] to the children, then it should be included in the
  /// transform applied here, whereas if [addToScene] just passes the
  /// `layerOffset` to the child, then it should not be included in the
  /// transform applied here.
  ///
  /// This method is only valid immediately after [addToScene] has been called,
  /// before any of the properties have been changed.
  ///
  /// The default implementation does nothing, since [ContainerLayer], by
  /// default, composites its children at the origin of the [ContainerLayer]
  /// itself.
  ///
  /// The `child` argument should generally not be null, since in principle a
  /// layer could transform each child independently. However, certain layers
  /// may explicitly allow null as a value, for example if they know that they
  /// transform all their children identically.
  ///
  /// The `transform` argument must not be null.
  ///
  /// Used by [FollowerLayer] to transform its child to a [LeaderLayer]'s
  /// position.
  void applyTransform(Layer child, Matrix4 transform) {
    assert(child != null);
    assert(transform != null);
  }

  /// Returns the descendants of this layer in depth first order.
  @visibleForTesting
  List<Layer> depthFirstIterateChildren() {
    if (firstChild == null)
      return <Layer>[];
    final List<Layer> children = <Layer>[];
    Layer child = firstChild;
    while(child != null) {
      children.add(child);
      if (child is ContainerLayer) {
        children.addAll(child.depthFirstIterateChildren());
      }
      child = child.nextSibling;
    }
    return children;
  }

  @override
  List<DiagnosticsNode> debugDescribeChildren() {
    final List<DiagnosticsNode> children = <DiagnosticsNode>[];
    if (firstChild == null)
      return children;
    Layer child = firstChild;
    int count = 1;
    while (true) {
      children.add(child.toDiagnosticsNode(name: 'child $count'));
      if (child == lastChild)
        break;
      count += 1;
      child = child.nextSibling;
    }
    return children;
  }
}
```

## 6.OffsetLayer（偏移层）
```dart
/// A layer that is displayed at an offset from its parent layer.
///
/// Offset layers are key to efficient repainting because they are created by
/// repaint boundaries in the [RenderObject] tree (see
/// [RenderObject.isRepaintBoundary]). When a render object that is a repaint
/// boundary is asked to paint at given offset in a [PaintingContext], the
/// render object first checks whether it needs to repaint itself. If not, it
/// reuses its existing [OffsetLayer] (and its entire subtree) by mutating its
/// [offset] property, cutting off the paint walk.
class OffsetLayer extends ContainerLayer {
  /// Creates an offset layer.
  ///
  /// By default, [offset] is zero. It must be non-null before the compositing
  /// phase of the pipeline.
  OffsetLayer({ Offset offset = Offset.zero }) : _offset = offset;

  /// Offset from parent in the parent's coordinate system.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  ///
  /// The [offset] property must be non-null before the compositing phase of the
  /// pipeline.
  Offset get offset => _offset;
  Offset _offset;
  set offset(Offset value) {
    if (value != _offset) {
      markNeedsAddToScene();
    }
    _offset = value;
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { @required bool onlyFirst }) {
    return super.findAnnotations<S>(result, localPosition - offset, onlyFirst: onlyFirst);
  }

  @override
  void applyTransform(Layer child, Matrix4 transform) {
    assert(child != null);
    assert(transform != null);
    transform.multiply(Matrix4.translationValues(offset.dx, offset.dy, 0.0));
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    // Skia has a fast path for concatenating scale/translation only matrices.
    // Hence pushing a translation-only transform layer should be fast. For
    // retained rendering, we don't want to push the offset down to each leaf
    // node. Otherwise, changing an offset layer on the very high level could
    // cascade the change to too many leaves.
    engineLayer = builder.pushOffset(layerOffset.dx + offset.dx, layerOffset.dy + offset.dy, oldLayer: _engineLayer);
    addChildrenToScene(builder);
    builder.pop();
  }

  /// Capture an image of the current state of this layer and its children.
  ///
  /// The returned [ui.Image] has uncompressed raw RGBA bytes, will be offset
  /// by the top-left corner of [bounds], and have dimensions equal to the size
  /// of [bounds] multiplied by [pixelRatio].
  ///
  /// The [pixelRatio] describes the scale between the logical pixels and the
  /// size of the output image. It is independent of the
  /// [window.devicePixelRatio] for the device, so specifying 1.0 (the default)
  /// will give you a 1:1 mapping between logical pixels and the output pixels
  /// in the image.
  ///
  /// See also:
  ///
  ///  * [RenderRepaintBoundary.toImage] for a similar API at the render object level.
  ///  * [dart:ui.Scene.toImage] for more information about the image returned.
  Future<ui.Image> toImage(Rect bounds, { double pixelRatio = 1.0 }) async {
    assert(bounds != null);
    assert(pixelRatio != null);
    final ui.SceneBuilder builder = ui.SceneBuilder();
    final Matrix4 transform = Matrix4.translationValues(
      (-bounds.left  - offset.dx) * pixelRatio,
      (-bounds.top - offset.dy) * pixelRatio,
      0.0,
    );
    transform.scale(pixelRatio, pixelRatio);
    builder.pushTransform(transform.storage);
    final ui.Scene scene = buildScene(builder);

    try {
      // Size is rounded up to the next pixel to make sure we don't clip off
      // anything.
      return await scene.toImage(
        (pixelRatio * bounds.width).ceil(),
        (pixelRatio * bounds.height).ceil(),
      );
    } finally {
      scene.dispose();
    }
  }
}
```
## 7.ClipRectLayer（直角裁剪层）
```dart
/// A composite layer that clips its children using a rectangle.
///
/// When debugging, setting [debugDisableClipLayers] to true will cause this
/// layer to be skipped (directly replaced by its children). This can be helpful
/// to track down the cause of performance problems.
class ClipRectLayer extends ContainerLayer {
  /// Creates a layer with a rectangular clip.
  ///
  /// The [clipRect] argument must not be null before the compositing phase of
  /// the pipeline.
  ///
  /// The [clipBehavior] argument must not be null, and must not be [Clip.none].
  ClipRectLayer({
    Rect clipRect,
    Clip clipBehavior = Clip.hardEdge,
  }) : _clipRect = clipRect,
       _clipBehavior = clipBehavior,
       assert(clipBehavior != null),
       assert(clipBehavior != Clip.none);

  /// The rectangle to clip in the parent's coordinate system.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  Rect get clipRect => _clipRect;
  Rect _clipRect;
  set clipRect(Rect value) {
    if (value != _clipRect) {
      _clipRect = value;
      markNeedsAddToScene();
    }
  }

  /// {@template flutter.clipper.clipBehavior}
  /// Controls how to clip.
  ///
  /// Must not be set to null or [Clip.none].
  /// {@endtemplate}
  ///
  /// Defaults to [Clip.hardEdge].
  Clip get clipBehavior => _clipBehavior;
  Clip _clipBehavior;
  set clipBehavior(Clip value) {
    assert(value != null);
    assert(value != Clip.none);
    if (value != _clipBehavior) {
      _clipBehavior = value;
      markNeedsAddToScene();
    }
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { @required bool onlyFirst }) {
    if (!clipRect.contains(localPosition))
      return false;
    return super.findAnnotations<S>(result, localPosition, onlyFirst: onlyFirst);
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(clipRect != null);
    assert(clipBehavior != null);
    bool enabled = true;
    assert(() {
      enabled = !debugDisableClipLayers;
      return true;
    }());
    if (enabled) {
      final Rect shiftedClipRect = layerOffset == Offset.zero ? clipRect : clipRect.shift(layerOffset);
      engineLayer = builder.pushClipRect(shiftedClipRect, clipBehavior: clipBehavior, oldLayer: _engineLayer);
    } else {
      engineLayer = null;
    }
    addChildrenToScene(builder, layerOffset);
    if (enabled)
      builder.pop();
  }
}
```
## 8.ClipRRectLayer（圆角裁剪层）
```dart
/// A composite layer that clips its children using a rounded rectangle.
///
/// When debugging, setting [debugDisableClipLayers] to true will cause this
/// layer to be skipped (directly replaced by its children). This can be helpful
/// to track down the cause of performance problems.
class ClipRRectLayer extends ContainerLayer {
  /// Creates a layer with a rounded-rectangular clip.
  ///
  /// The [clipRRect] and [clipBehavior] properties must be non-null before the
  /// compositing phase of the pipeline.
  ClipRRectLayer({
    RRect clipRRect,
    Clip clipBehavior = Clip.antiAlias,
  }) : _clipRRect = clipRRect,
       _clipBehavior = clipBehavior,
       assert(clipBehavior != null),
       assert(clipBehavior != Clip.none);

  /// The rounded-rect to clip in the parent's coordinate system.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  RRect get clipRRect => _clipRRect;
  RRect _clipRRect;
  set clipRRect(RRect value) {
    if (value != _clipRRect) {
      _clipRRect = value;
      markNeedsAddToScene();
    }
  }

  /// {@macro flutter.clipper.clipBehavior}
  ///
  /// Defaults to [Clip.antiAlias].
  Clip get clipBehavior => _clipBehavior;
  Clip _clipBehavior;
  set clipBehavior(Clip value) {
    assert(value != null);
    assert(value != Clip.none);
    if (value != _clipBehavior) {
      _clipBehavior = value;
      markNeedsAddToScene();
    }
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { @required bool onlyFirst }) {
    if (!clipRRect.contains(localPosition))
      return false;
    return super.findAnnotations<S>(result, localPosition, onlyFirst: onlyFirst);
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(clipRRect != null);
    assert(clipBehavior != null);
    bool enabled = true;
    assert(() {
      enabled = !debugDisableClipLayers;
      return true;
    }());
    if (enabled) {
      final RRect shiftedClipRRect = layerOffset == Offset.zero ? clipRRect : clipRRect.shift(layerOffset);
      engineLayer = builder.pushClipRRect(shiftedClipRRect, clipBehavior: clipBehavior, oldLayer: _engineLayer);
    } else {
      engineLayer = null;
    }
    addChildrenToScene(builder, layerOffset);
    if (enabled)
      builder.pop();
  }
}
```
## 9.ClipPathLayer（路径裁剪层）
```dart
/// A composite layer that clips its children using a path.
///
/// When debugging, setting [debugDisableClipLayers] to true will cause this
/// layer to be skipped (directly replaced by its children). This can be helpful
/// to track down the cause of performance problems.
class ClipPathLayer extends ContainerLayer {
  /// Creates a layer with a path-based clip.
  ///
  /// The [clipPath] and [clipBehavior] properties must be non-null before the
  /// compositing phase of the pipeline.
  ClipPathLayer({
    Path clipPath,
    Clip clipBehavior = Clip.antiAlias,
  }) : _clipPath = clipPath,
       _clipBehavior = clipBehavior,
       assert(clipBehavior != null),
       assert(clipBehavior != Clip.none);

  /// The path to clip in the parent's coordinate system.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  Path get clipPath => _clipPath;
  Path _clipPath;
  set clipPath(Path value) {
    if (value != _clipPath) {
      _clipPath = value;
      markNeedsAddToScene();
    }
  }

  /// {@macro flutter.clipper.clipBehavior}
  ///
  /// Defaults to [Clip.antiAlias].
  Clip get clipBehavior => _clipBehavior;
  Clip _clipBehavior;
  set clipBehavior(Clip value) {
    assert(value != null);
    assert(value != Clip.none);
    if (value != _clipBehavior) {
      _clipBehavior = value;
      markNeedsAddToScene();
    }
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { @required bool onlyFirst }) {
    if (!clipPath.contains(localPosition))
      return false;
    return super.findAnnotations<S>(result, localPosition, onlyFirst: onlyFirst);
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(clipPath != null);
    assert(clipBehavior != null);
    bool enabled = true;
    assert(() {
      enabled = !debugDisableClipLayers;
      return true;
    }());
    if (enabled) {
      final Path shiftedPath = layerOffset == Offset.zero ? clipPath : clipPath.shift(layerOffset);
      engineLayer = builder.pushClipPath(shiftedPath, clipBehavior: clipBehavior, oldLayer: _engineLayer);
    } else {
      engineLayer = null;
    }
    addChildrenToScene(builder, layerOffset);
    if (enabled)
      builder.pop();
  }
}
```
## 10.ColorFilterLayer（颜色过滤层）
```dart
/// A composite layer that applies a [ColorFilter] to its children.
class ColorFilterLayer extends ContainerLayer {
  /// Creates a layer that applies a [ColorFilter] to its children.
  ///
  /// The [colorFilter] property must be non-null before the compositing phase
  /// of the pipeline.
  ColorFilterLayer({
    ColorFilter colorFilter,
  }) : _colorFilter = colorFilter;

  /// The color filter to apply to children.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  ColorFilter get colorFilter => _colorFilter;
  ColorFilter _colorFilter;
  set colorFilter(ColorFilter value) {
    assert(value != null);
    if (value != _colorFilter) {
      _colorFilter = value;
      markNeedsAddToScene();
    }
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(colorFilter != null);
    engineLayer = builder.pushColorFilter(colorFilter, oldLayer: _engineLayer);
    addChildrenToScene(builder, layerOffset);
    builder.pop();
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(DiagnosticsProperty<ColorFilter>('colorFilter', colorFilter));
  }
}
```
## 11.TransformLayer（变换层）
```dart
/// A composited layer that applies a given transformation matrix to its
/// children.
///
/// This class inherits from [OffsetLayer] to make it one of the layers that
/// can be used at the root of a [RenderObject] hierarchy.
class TransformLayer extends OffsetLayer {
  /// Creates a transform layer.
  ///
  /// The [transform] and [offset] properties must be non-null before the
  /// compositing phase of the pipeline.
  TransformLayer({ Matrix4 transform, Offset offset = Offset.zero })
    : _transform = transform,
      super(offset: offset);

  /// The matrix to apply.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  ///
  /// This transform is applied before [offset], if both are set.
  ///
  /// The [transform] property must be non-null before the compositing phase of
  /// the pipeline.
  Matrix4 get transform => _transform;
  Matrix4 _transform;
  set transform(Matrix4 value) {
    assert(value != null);
    assert(value.storage.every((double component) => component.isFinite));
    if (value == _transform)
      return;
    _transform = value;
    _inverseDirty = true;
    markNeedsAddToScene();
  }

  Matrix4 _lastEffectiveTransform;
  Matrix4 _invertedTransform;
  bool _inverseDirty = true;

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(transform != null);
    _lastEffectiveTransform = transform;
    final Offset totalOffset = offset + layerOffset;
    if (totalOffset != Offset.zero) {
      _lastEffectiveTransform = Matrix4.translationValues(totalOffset.dx, totalOffset.dy, 0.0)
        ..multiply(_lastEffectiveTransform);
    }
    engineLayer = builder.pushTransform(_lastEffectiveTransform.storage, oldLayer: _engineLayer);
    addChildrenToScene(builder);
    builder.pop();
  }

  Offset _transformOffset(Offset localPosition) {
    if (_inverseDirty) {
      _invertedTransform = Matrix4.tryInvert(
        PointerEvent.removePerspectiveTransform(transform)
      );
      _inverseDirty = false;
    }
    if (_invertedTransform == null)
      return null;
    final Vector4 vector = Vector4(localPosition.dx, localPosition.dy, 0.0, 1.0);
    final Vector4 result = _invertedTransform.transform(vector);
    return Offset(result[0], result[1]);
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { @required bool onlyFirst }) {
    final Offset transformedOffset = _transformOffset(localPosition);
    if (transformedOffset == null)
      return false;
    return super.findAnnotations<S>(result, transformedOffset, onlyFirst: onlyFirst);
  }

  @override
  void applyTransform(Layer child, Matrix4 transform) {
    assert(child != null);
    assert(transform != null);
    assert(_lastEffectiveTransform != null || this.transform != null);
    if (_lastEffectiveTransform == null) {
      transform.multiply(this.transform);
    } else {
      transform.multiply(_lastEffectiveTransform);
    }
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(TransformProperty('transform', transform));
  }
}
```
## 12.OpacityLayer（透明度层）
```dart
/// A composited layer that makes its children partially transparent.
///
/// When debugging, setting [debugDisableOpacityLayers] to true will cause this
/// layer to be skipped (directly replaced by its children). This can be helpful
/// to track down the cause of performance problems.
///
/// Try to avoid an [OpacityLayer] with no children. Remove that layer if
/// possible to save some tree walks.
class OpacityLayer extends ContainerLayer {
  /// Creates an opacity layer.
  ///
  /// The [alpha] property must be non-null before the compositing phase of
  /// the pipeline.
  OpacityLayer({
    int alpha,
    Offset offset = Offset.zero,
  }) : _alpha = alpha,
       _offset = offset;

  /// The amount to multiply into the alpha channel.
  ///
  /// The opacity is expressed as an integer from 0 to 255, where 0 is fully
  /// transparent and 255 is fully opaque.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  int get alpha => _alpha;
  int _alpha;
  set alpha(int value) {
    assert(value != null);
    if (value != _alpha) {
      _alpha = value;
      markNeedsAddToScene();
    }
  }

  /// Offset from parent in the parent's coordinate system.
  Offset get offset => _offset;
  Offset _offset;
  set offset(Offset value) {
    if (value != _offset) {
      _offset = value;
      markNeedsAddToScene();
    }
  }

  @override
  void applyTransform(Layer child, Matrix4 transform) {
    assert(child != null);
    assert(transform != null);
    transform.translate(offset.dx, offset.dy);
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(alpha != null);
    bool enabled = firstChild != null;  // don't add this layer if there's no child
    assert(() {
      enabled = enabled && !debugDisableOpacityLayers;
      return true;
    }());

    if (enabled)
      engineLayer = builder.pushOpacity(alpha, offset: offset + layerOffset, oldLayer: _engineLayer);
    else
      engineLayer = null;
    addChildrenToScene(builder);
    if (enabled)
      builder.pop();
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(IntProperty('alpha', alpha));
    properties.add(DiagnosticsProperty<Offset>('offset', offset));
  }
}

/// A composited layer that applies a shader to its children.
class ShaderMaskLayer extends ContainerLayer {
  /// Creates a shader mask layer.
  ///
  /// The [shader], [maskRect], and [blendMode] properties must be non-null
  /// before the compositing phase of the pipeline.
  ShaderMaskLayer({
    Shader shader,
    Rect maskRect,
    BlendMode blendMode,
  }) : _shader = shader,
       _maskRect = maskRect,
       _blendMode = blendMode;

  /// The shader to apply to the children.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  Shader get shader => _shader;
  Shader _shader;
  set shader(Shader value) {
    if (value != _shader) {
      _shader = value;
      markNeedsAddToScene();
    }
  }

  /// The size of the shader.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  Rect get maskRect => _maskRect;
  Rect _maskRect;
  set maskRect(Rect value) {
    if (value != _maskRect) {
      _maskRect = value;
      markNeedsAddToScene();
    }
  }

  /// The blend mode to apply when blending the shader with the children.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  BlendMode get blendMode => _blendMode;
  BlendMode _blendMode;
  set blendMode(BlendMode value) {
    if (value != _blendMode) {
      _blendMode = value;
      markNeedsAddToScene();
    }
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(shader != null);
    assert(maskRect != null);
    assert(blendMode != null);
    final Rect shiftedMaskRect = layerOffset == Offset.zero ? maskRect : maskRect.shift(layerOffset);
    engineLayer = builder.pushShaderMask(shader, shiftedMaskRect, blendMode, oldLayer: _engineLayer);
    addChildrenToScene(builder, layerOffset);
    builder.pop();
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(DiagnosticsProperty<Shader>('shader', shader));
    properties.add(DiagnosticsProperty<Rect>('maskRect', maskRect));
    properties.add(DiagnosticsProperty<BlendMode>('blendMode', blendMode));
  }
}
```
## 13.BackdropFilterLayer（背景过滤层）
```dart
/// A composited layer that applies a filter to the existing contents of the scene.
class BackdropFilterLayer extends ContainerLayer {
  /// Creates a backdrop filter layer.
  ///
  /// The [filter] property must be non-null before the compositing phase of the
  /// pipeline.
  BackdropFilterLayer({ ui.ImageFilter filter }) : _filter = filter;

  /// The filter to apply to the existing contents of the scene.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  ui.ImageFilter get filter => _filter;
  ui.ImageFilter _filter;
  set filter(ui.ImageFilter value) {
    if (value != _filter) {
      _filter = value;
      markNeedsAddToScene();
    }
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(filter != null);
    engineLayer = builder.pushBackdropFilter(filter, oldLayer: _engineLayer);
    addChildrenToScene(builder, layerOffset);
    builder.pop();
  }
}
```
## 14.PhysicalModelLayer（物理模型层）
```dart
/// A composited layer that uses a physical model to producing lighting effects.
///
/// For example, the layer casts a shadow according to its geometry and the
/// relative position of lights and other physically modeled objects in the
/// scene.
///
/// When debugging, setting [debugDisablePhysicalShapeLayers] to true will cause this
/// layer to be skipped (directly replaced by its children). This can be helpful
/// to track down the cause of performance problems.
class PhysicalModelLayer extends ContainerLayer {
  /// Creates a composited layer that uses a physical model to producing
  /// lighting effects.
  ///
  /// The [clipPath], [clipBehavior], [elevation], [color], and [shadowColor]
  /// arguments must be non-null before the compositing phase of the pipeline.
  PhysicalModelLayer({
    Path clipPath,
    Clip clipBehavior = Clip.none,
    double elevation,
    Color color,
    Color shadowColor,
  }) : _clipPath = clipPath,
       _clipBehavior = clipBehavior,
       _elevation = elevation,
       _color = color,
       _shadowColor = shadowColor;

  /// The path to clip in the parent's coordinate system.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  Path get clipPath => _clipPath;
  Path _clipPath;
  set clipPath(Path value) {
    if (value != _clipPath) {
      _clipPath = value;
      markNeedsAddToScene();
    }
  }

  Path get _debugTransformedClipPath {
    ContainerLayer ancestor = parent;
    final Matrix4 matrix = Matrix4.identity();
    while (ancestor != null && ancestor.parent != null) {
      ancestor.applyTransform(this, matrix);
      ancestor = ancestor.parent;
    }
    return clipPath.transform(matrix.storage);
  }

  /// {@macro flutter.widgets.Clip}
  Clip get clipBehavior => _clipBehavior;
  Clip _clipBehavior;
  set clipBehavior(Clip value) {
    assert(value != null);
    if (value != _clipBehavior) {
      _clipBehavior = value;
      markNeedsAddToScene();
    }
  }

  /// The z-coordinate at which to place this physical object.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  ///
  /// In tests, the [debugDisableShadows] flag is set to true by default.
  /// Several widgets and render objects force all elevations to zero when this
  /// flag is set. For this reason, this property will often be set to zero in
  /// tests even if the layer should be raised. To verify the actual value,
  /// consider setting [debugDisableShadows] to false in your test.
  double get elevation => _elevation;
  double _elevation;
  set elevation(double value) {
    if (value != _elevation) {
      _elevation = value;
      markNeedsAddToScene();
    }
  }

  /// The background color.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  Color get color => _color;
  Color _color;
  set color(Color value) {
    if (value != _color) {
      _color = value;
      markNeedsAddToScene();
    }
  }

  /// The shadow color.
  Color get shadowColor => _shadowColor;
  Color _shadowColor;
  set shadowColor(Color value) {
    if (value != _shadowColor) {
      _shadowColor = value;
      markNeedsAddToScene();
    }
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { @required bool onlyFirst }) {
    if (!clipPath.contains(localPosition))
      return false;
    return super.findAnnotations<S>(result, localPosition, onlyFirst: onlyFirst);
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(clipPath != null);
    assert(clipBehavior != null);
    assert(elevation != null);
    assert(color != null);
    assert(shadowColor != null);

    bool enabled = true;
    assert(() {
      enabled = !debugDisablePhysicalShapeLayers;
      return true;
    }());
    if (enabled) {
      engineLayer = builder.pushPhysicalShape(
        path: layerOffset == Offset.zero ? clipPath : clipPath.shift(layerOffset),
        elevation: elevation,
        color: color,
        shadowColor: shadowColor,
        clipBehavior: clipBehavior,
        oldLayer: _engineLayer,
      );
    } else {
      engineLayer = null;
    }
    addChildrenToScene(builder, layerOffset);
    if (enabled)
      builder.pop();
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(DoubleProperty('elevation', elevation));
    properties.add(ColorProperty('color', color));
  }
}
```
## LayerLink
```dart
/// An object that a [LeaderLayer] can register with.
///
/// An instance of this class should be provided as the [LeaderLayer.link] and
/// the [FollowerLayer.link] properties to cause the [FollowerLayer] to follow
/// the [LeaderLayer].
///
/// See also:
///
///  * [CompositedTransformTarget], the widget that creates a [LeaderLayer].
///  * [CompositedTransformFollower], the widget that creates a [FollowerLayer].
///  * [RenderLeaderLayer] and [RenderFollowerLayer], the corresponding
///    render objects.
class LayerLink {
  /// The currently-registered [LeaderLayer], if any.
  LeaderLayer get leader => _leader;
  LeaderLayer _leader;

  @override
  String toString() => '${describeIdentity(this)}(${ _leader != null ? "<linked>" : "<dangling>" })';
}
```
## 15.LeaderLayer（领导层）
```dart
/// A composited layer that can be followed by a [FollowerLayer].
///
/// This layer collapses the accumulated offset into a transform and passes
/// [Offset.zero] to its child layers in the [addToScene]/[addChildrenToScene]
/// methods, so that [applyTransform] will work reliably.
class LeaderLayer extends ContainerLayer {
  /// Creates a leader layer.
  ///
  /// The [link] property must not be null, and must not have been provided to
  /// any other [LeaderLayer] layers that are [attached] to the layer tree at
  /// the same time.
  ///
  /// The [offset] property must be non-null before the compositing phase of the
  /// pipeline.
  LeaderLayer({ @required LayerLink link, this.offset = Offset.zero }) : assert(link != null), _link = link;

  /// The object with which this layer should register.
  ///
  /// The link will be established when this layer is [attach]ed, and will be
  /// cleared when this layer is [detach]ed.
  LayerLink get link => _link;
  set link(LayerLink value) {
    assert(value != null);
    _link = value;
  }
  LayerLink _link;

  /// Offset from parent in the parent's coordinate system.
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  ///
  /// The [offset] property must be non-null before the compositing phase of the
  /// pipeline.
  Offset offset;

  /// {@macro flutter.leaderFollower.alwaysNeedsAddToScene}
  @override
  bool get alwaysNeedsAddToScene => true;

  @override
  void attach(Object owner) {
    super.attach(owner);
    assert(link.leader == null);
    _lastOffset = null;
    link._leader = this;
  }

  @override
  void detach() {
    assert(link.leader == this);
    link._leader = null;
    _lastOffset = null;
    super.detach();
  }

  /// The offset the last time this layer was composited.
  ///
  /// This is reset to null when the layer is attached or detached, to help
  /// catch cases where the follower layer ends up before the leader layer, but
  /// not every case can be detected.
  Offset _lastOffset;

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { @required bool onlyFirst }) {
    return super.findAnnotations<S>(result, localPosition - offset, onlyFirst: onlyFirst);
  }

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(offset != null);
    _lastOffset = offset + layerOffset;
    if (_lastOffset != Offset.zero)
      engineLayer = builder.pushTransform(Matrix4.translationValues(_lastOffset.dx, _lastOffset.dy, 0.0).storage, oldLayer: _engineLayer);
    addChildrenToScene(builder);
    if (_lastOffset != Offset.zero)
      builder.pop();
  }

  /// Applies the transform that would be applied when compositing the given
  /// child to the given matrix.
  ///
  /// See [ContainerLayer.applyTransform] for details.
  ///
  /// The `child` argument may be null, as the same transform is applied to all
  /// children.
  @override
  void applyTransform(Layer child, Matrix4 transform) {
    assert(_lastOffset != null);
    if (_lastOffset != Offset.zero)
      transform.translate(_lastOffset.dx, _lastOffset.dy);
  }
}
```
## 16.FollowerLayer（跟随层）
```dart
/// A composited layer that applies a transformation matrix to its children such
/// that they are positioned to match a [LeaderLayer].
///
/// If any of the ancestors of this layer have a degenerate matrix (e.g. scaling
/// by zero), then the [FollowerLayer] will not be able to transform its child
/// to the coordinate space of the [LeaderLayer].
///
/// A [linkedOffset] property can be provided to further offset the child layer
/// from the leader layer, for example if the child is to follow the linked
/// layer at a distance rather than directly overlapping it.
class FollowerLayer extends ContainerLayer {
  /// Creates a follower layer.
  ///
  /// The [link] property must not be null.
  ///
  /// The [unlinkedOffset], [linkedOffset], and [showWhenUnlinked] properties
  /// must be non-null before the compositing phase of the pipeline.
  FollowerLayer({
    @required LayerLink link,
    this.showWhenUnlinked = true,
    this.unlinkedOffset = Offset.zero,
    this.linkedOffset = Offset.zero,
  }) : assert(link != null), _link = link;

  /// The link to the [LeaderLayer].
  ///
  /// The same object should be provided to a [LeaderLayer] that is earlier in
  /// the layer tree. When this layer is composited, it will apply a transform
  /// that moves its children to match the position of the [LeaderLayer].
  LayerLink get link => _link;
  set link(LayerLink value) {
    assert(value != null);
    _link = value;
  }
  LayerLink _link;

  /// Whether to show the layer's contents when the [link] does not point to a
  /// [LeaderLayer].
  ///
  /// When the layer is linked, children layers are positioned such that they
  /// have the same global position as the linked [LeaderLayer].
  ///
  /// When the layer is not linked, then: if [showWhenUnlinked] is true,
  /// children are positioned as if the [FollowerLayer] was a [ContainerLayer];
  /// if it is false, then children are hidden.
  ///
  /// The [showWhenUnlinked] property must be non-null before the compositing
  /// phase of the pipeline.
  bool showWhenUnlinked;

  /// Offset from parent in the parent's coordinate system, used when the layer
  /// is not linked to a [LeaderLayer].
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  ///
  /// The [unlinkedOffset] property must be non-null before the compositing
  /// phase of the pipeline.
  ///
  /// See also:
  ///
  ///  * [linkedOffset], for when the layers are linked.
  Offset unlinkedOffset;

  /// Offset from the origin of the leader layer to the origin of the child
  /// layers, used when the layer is linked to a [LeaderLayer].
  ///
  /// The scene must be explicitly recomposited after this property is changed
  /// (as described at [Layer]).
  ///
  /// The [linkedOffset] property must be non-null before the compositing phase
  /// of the pipeline.
  ///
  /// See also:
  ///
  ///  * [unlinkedOffset], for when the layer is not linked.
  Offset linkedOffset;

  Offset _lastOffset;
  Matrix4 _lastTransform;
  Matrix4 _invertedTransform;
  bool _inverseDirty = true;

  Offset _transformOffset<S>(Offset localPosition) {
    if (_inverseDirty) {
      _invertedTransform = Matrix4.tryInvert(getLastTransform());
      _inverseDirty = false;
    }
    if (_invertedTransform == null)
      return null;
    final Vector4 vector = Vector4(localPosition.dx, localPosition.dy, 0.0, 1.0);
    final Vector4 result = _invertedTransform.transform(vector);
    return Offset(result[0] - linkedOffset.dx, result[1] - linkedOffset.dy);
  }

  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { @required bool onlyFirst }) {
    if (link.leader == null) {
      if (showWhenUnlinked) {
        return super.findAnnotations(result, localPosition - unlinkedOffset, onlyFirst: onlyFirst);
      }
      return false;
    }
    final Offset transformedOffset = _transformOffset<S>(localPosition);
    if (transformedOffset == null) {
      return false;
    }
    return super.findAnnotations<S>(result, transformedOffset, onlyFirst: onlyFirst);
  }

  /// The transform that was used during the last composition phase.
  ///
  /// If the [link] was not linked to a [LeaderLayer], or if this layer has
  /// a degenerate matrix applied, then this will be null.
  ///
  /// This method returns a new [Matrix4] instance each time it is invoked.
  Matrix4 getLastTransform() {
    if (_lastTransform == null)
      return null;
    final Matrix4 result = Matrix4.translationValues(-_lastOffset.dx, -_lastOffset.dy, 0.0);
    result.multiply(_lastTransform);
    return result;
  }

  /// Call [applyTransform] for each layer in the provided list.
  ///
  /// The list is in reverse order (deepest first). The first layer will be
  /// treated as the child of the second, and so forth. The first layer in the
  /// list won't have [applyTransform] called on it. The first layer may be
  /// null.
  Matrix4 _collectTransformForLayerChain(List<ContainerLayer> layers) {
    // Initialize our result matrix.
    final Matrix4 result = Matrix4.identity();
    // Apply each layer to the matrix in turn, starting from the last layer,
    // and providing the previous layer as the child.
    for (int index = layers.length - 1; index > 0; index -= 1)
      layers[index].applyTransform(layers[index - 1], result);
    return result;
  }

  /// Populate [_lastTransform] given the current state of the tree.
  void _establishTransform() {
    assert(link != null);
    _lastTransform = null;
    // Check to see if we are linked.
    if (link.leader == null)
      return;
    // If we're linked, check the link is valid.
    assert(link.leader.owner == owner, 'Linked LeaderLayer anchor is not in the same layer tree as the FollowerLayer.');
    assert(link.leader._lastOffset != null, 'LeaderLayer anchor must come before FollowerLayer in paint order, but the reverse was true.');
    // Collect all our ancestors into a Set so we can recognize them.
    final Set<Layer> ancestors = HashSet<Layer>();
    Layer ancestor = parent;
    while (ancestor != null) {
      ancestors.add(ancestor);
      ancestor = ancestor.parent;
    }
    // Collect all the layers from a hypothetical child (null) of the target
    // layer up to the common ancestor layer.
    ContainerLayer layer = link.leader;
    final List<ContainerLayer> forwardLayers = <ContainerLayer>[null, layer];
    do {
      layer = layer.parent;
      forwardLayers.add(layer);
    } while (!ancestors.contains(layer));
    ancestor = layer;
    // Collect all the layers from this layer up to the common ancestor layer.
    layer = this;
    final List<ContainerLayer> inverseLayers = <ContainerLayer>[layer];
    do {
      layer = layer.parent;
      inverseLayers.add(layer);
    } while (layer != ancestor);
    // Establish the forward and backward matrices given these lists of layers.
    final Matrix4 forwardTransform = _collectTransformForLayerChain(forwardLayers);
    final Matrix4 inverseTransform = _collectTransformForLayerChain(inverseLayers);
    if (inverseTransform.invert() == 0.0) {
      // We are in a degenerate transform, so there's not much we can do.
      return;
    }
    // Combine the matrices and store the result.
    inverseTransform.multiply(forwardTransform);
    inverseTransform.translate(linkedOffset.dx, linkedOffset.dy);
    _lastTransform = inverseTransform;
    _inverseDirty = true;
  }

  /// {@template flutter.leaderFollower.alwaysNeedsAddToScene}
  /// This disables retained rendering.
  ///
  /// A [FollowerLayer] copies changes from a [LeaderLayer] that could be anywhere
  /// in the Layer tree, and that leader layer could change without notifying the
  /// follower layer. Therefore we have to always call a follower layer's
  /// [addToScene]. In order to call follower layer's [addToScene], leader layer's
  /// [addToScene] must be called first so leader layer must also be considered
  /// as [alwaysNeedsAddToScene].
  /// {@endtemplate}
  @override
  bool get alwaysNeedsAddToScene => true;

  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    assert(link != null);
    assert(showWhenUnlinked != null);
    if (link.leader == null && !showWhenUnlinked) {
      _lastTransform = null;
      _lastOffset = null;
      _inverseDirty = true;
      engineLayer = null;
      return;
    }
    _establishTransform();
    if (_lastTransform != null) {
      engineLayer = builder.pushTransform(_lastTransform.storage, oldLayer: _engineLayer);
      addChildrenToScene(builder);
      builder.pop();
      _lastOffset = unlinkedOffset + layerOffset;
    } else {
      _lastOffset = null;
      final Matrix4 matrix = Matrix4.translationValues(unlinkedOffset.dx, unlinkedOffset.dy, .0);
      engineLayer = builder.pushTransform(matrix.storage, oldLayer: _engineLayer);
      addChildrenToScene(builder);
      builder.pop();
    }
    _inverseDirty = true;
  }

  @override
  void applyTransform(Layer child, Matrix4 transform) {
    assert(child != null);
    assert(transform != null);
    if (_lastTransform != null) {
      transform.multiply(_lastTransform);
    } else {
      transform.multiply(Matrix4.translationValues(unlinkedOffset.dx, unlinkedOffset.dy, .0));
    }
  }
}
```
## 17.AnnotatedRegionLayer（注解范围层）
```dart
/// A composited layer which annotates its children with a value. Pushing this
/// layer to the tree is the common way of adding an annotation.
///
/// An annotation is an optional object of any type that, when attached with a
/// layer, can be retrieved using [Layer.find] or [Layer.findAllAnnotations]
/// with a position. The search process is done recursively, controlled by a
/// concept of being opaque to a type of annotation, explained in the document
/// of [Layer.findAnnotations].
///
/// When an annotation search arrives, this layer defers the same search to each
/// of this layer's children, respecting their opacity. Then it adds this
/// layer's [annotation] if all of the following restrictions are met:
///
/// {@template flutter.rendering.annotatedRegionLayer.restrictions}
/// * The target type must be identical to the annotated type `T`.
/// * If [size] is provided, the target position must be contained within the
///   rectangle formed by [size] and [offset].
/// {@endtemplate}
///
/// This layer is opaque to a type of annotation if any child is also opaque, or
/// if [opaque] is true and the layer's annotation is added.
/// 一个复合层，它用一个值来注释它的子层。将这个层插入树中是添加注释的常用方法。
/// 注释是任何类型的可选对象，当注释随着层被挂在到树上时，可以使用[Layer.find]或
/// [Layer.findAllAnnotations]来获取它。搜索过程是递归的，在[layer.findannotation]的文档中解释了
/// 对注释类型不透明的概念。
/// 当注释搜索到达时，这个层将相同的搜索延迟给它的孩子
这层的孩子，尊重他们的不透明。然后加上这个
层的[注释]如果满足以下所有限制:
{@template flutter.rendering.annotatedRegionLayer.restrictions}
*目标类型必须与带注释的类型“T”相同。
*如果提供[size]，则目标位置必须包含在
由[尺寸]和[偏移量]构成的矩形。
{@endtemplate}
如果任何子元素也是不透明的，则此层对注释类型是不透明的
如果[不透明]为真，则添加该层的注释。
class AnnotatedRegionLayer<T> extends ContainerLayer {
  /// Creates a new layer that annotates its children with [value].
  ///
  /// The [value] provided cannot be null.
  AnnotatedRegionLayer(
    this.value, {
    this.size,
    Offset offset,
    this.opaque = false,
  }) : assert(value != null),
       assert(opaque != null),
       offset = offset ?? Offset.zero;

  /// The annotated object, which is added to the result if all restrictions are
  /// met.
  final T value;

  /// The size of an optional clipping rectangle, used to control whether a
  /// position is contained by the annotation.
  ///
  /// If [size] is provided, then the annotation is only added if the target
  /// position is contained by the rectangle formed by [size] and [offset].
  /// Otherwise no such restriction is applied, and clipping can only be done by
  /// the ancestor layers.
  final Size size;

  /// The offset of the optional clipping rectangle that is indicated by [size].
  ///
  /// The [offset] defaults to [Offset.zero] if not provided, and is ignored if
  /// [size] is not set.
  ///
  /// The [offset] only offsets the clipping rectangle, and does not affect
  /// how the painting or annotation search is propagated to its children.
  final Offset offset;

  /// Whether the annotation of this layer should be opaque during an annotation
  /// search of type `T`, preventing siblings visually behind it from being
  /// searched.
  ///
  /// If [opaque] is true, and this layer does add its annotation [value],
  /// then the layer will always be opaque during the search.
  ///
  /// If [opaque] is false, or if this layer does not add its annotation,
  /// then the opacity of this layer will be the one returned by the children,
  /// meaning that it will be opaque if any child is opaque.
  ///
  /// The [opaque] defaults to false.
  ///
  /// The [opaque] is effectively useless during [Layer.find] (more
  /// specifically, [Layer.findAnnotations] with `onlyFirst: true`), since the
  /// search process then skips the remaining tree after finding the first
  /// annotation.
  ///
  /// See also:
  ///
  ///  * [Layer.findAnnotations], which explains the concept of being opaque
  ///    to a type of annotation as the return value.
  ///  * [HitTestBehavior], which controls similar logic when hit-testing in the
  ///    render tree.
  final bool opaque;

  /// Searches the subtree for annotations of type `S` at the location
  /// `localPosition`, then adds the annotation [value] if applicable.
  ///
  /// This method always searches its children, and if any child returns `true`,
  /// the remaining children are skipped. Regardless of what the children
  /// return, this method then adds this layer's annotation if all of the
  /// following restrictions are met:
  ///
  /// {@macro flutter.rendering.annotatedRegionLayer.restrictions}
  ///
  /// This search process respects `onlyFirst`, meaning that when `onlyFirst` is
  /// true, the search will stop when it finds the first annotation from the
  /// children, and the layer's own annotation is checked only when none is
  /// given by the children.
  ///
  /// The return value is true if any child returns `true`, or if [opaque] is
  /// true and the layer's annotation is added.
  ///
  /// For explanation of layer annotations, parameters and return value, refer
  /// to [Layer.findAnnotations].
  @override
  @protected
  bool findAnnotations<S>(AnnotationResult<S> result, Offset localPosition, { @required bool onlyFirst }) {
    bool isAbsorbed = super.findAnnotations(result, localPosition, onlyFirst: onlyFirst);
    if (result.entries.isNotEmpty && onlyFirst)
      return isAbsorbed;
    if (size != null && !(offset & size).contains(localPosition)) {
      return isAbsorbed;
    }
    if (T == S) {
      isAbsorbed = isAbsorbed || opaque;
      final Object untypedValue = value;
      final S typedValue = untypedValue;
      result.add(AnnotationEntry<S>(
        annotation: typedValue,
        localPosition: localPosition,
      ));
    }
    return isAbsorbed;
  }
}
```


# Flutter ImageProvider 源码分析



## UI.Image

```dart
/// 不透明的原始解码图像数据句柄 (pixels)
///
/// 使用 [instantiateImageCodec] 获取 [Image] 对象
class Image extends NativeFieldWrapperClass2 {
  @pragma('vm:entry-point')
  Image._();

  /// 图片的水平宽度
  int get width native 'Image_width';

  /// 图片的垂直高度
  int get height native 'Image_height';

  /// 将 [Image] 对象转换成字节数组
  ///
  /// [format] 指定了返回值的格式
  Future<ByteData> toByteData({ImageByteFormat format = ImageByteFormat.rawRgba}) {
    return _futurize((_Callback<ByteData> callback) {
      return _toByteData(format.index, (Uint8List encoded) {
        callback(encoded?.buffer?.asByteData());
      });
    });
  }

  String _toByteData(int format, _Callback<Uint8List> callback) native 'Image_toByteData';

  /// 释放此图片资源占用的内存
  void dispose() native 'Image_dispose';
}

```



## ImageInfo

```dart
/// 整合 [dart:ui.Image] 对象和其对应的尺寸
///
/// 此对象被 [ImageStream] 对象使用，用于呈现实际的图片图片数据
@immutable
class ImageInfo {
  const ImageInfo({ @required this.image, this.scale = 1.0 });

  /// 原始的图片像素
  final ui.Image image;

  /// 绘制此图像所需的线性比例因子。
  ///
  /// 此比例因此将会同时作用图片的宽高
  ///
  /// 例如，如果此值是 2.0，则意味着每个逻辑像素对应四个图像像素，以及图像的实际宽度和高度(由
  /// [dart:ui.Image.width]和[dart:ui.Image.height] 属性给出)是绘制图像时应该使用的高度和宽度的两倍
  final double scale;
  ...
}
```





## ImageConfugration

```dart
/// 传递给 [ImageProvider.resolve] 方法的图片配置信息，用于选择一个确定的图片
@immutable
class ImageConfiguration {
  /// 创建一个持有 [ImageProvider] 配置信息的对象
  ///
  /// 所有参数都是可选的。配置信息仅仅是建议性的或是提供最好的效率。
  const ImageConfiguration({
    this.bundle,
    this.devicePixelRatio,
    this.locale,
    this.textDirection,
    this.size,
    this.platform,
  });

  ImageConfiguration copyWith({
    AssetBundle bundle,
    double devicePixelRatio,
    Locale locale,
    TextDirection textDirection,
    Size size,
    TargetPlatform platform,
  }) {
    return ImageConfiguration(
      bundle: bundle ?? this.bundle,
      devicePixelRatio: devicePixelRatio ?? this.devicePixelRatio,
      locale: locale ?? this.locale,
      textDirection: textDirection ?? this.textDirection,
      size: size ?? this.size,
      platform: platform ?? this.platform,
    );
  }

  /// 如果 [ImageProvider] 需要一个 bundle，却没有，则使用首选的[AssetBundle]。
  final AssetBundle bundle;

  /// 设备像素比
  final double devicePixelRatio;

  /// 本地化信息
  final Locale locale;

  /// 文字方向
  final TextDirection textDirection;

  /// 图片将要被渲染的大小
  final Size size;

  /// 应该为其使用资产的[TargetPlatform]。这允许以平台无关的方式指定图像，但在不同的平台上使用不同的资产，以
  /// 匹配本地约定，例如颜色匹配或阴影。
  final TargetPlatform platform;

  /// 空图片配置
  static const ImageConfiguration empty = ImageConfiguration();

  ...
}
```



## ImageStreamListener

```dart
typedef ImageListener = void Function(ImageInfo image, bool synchronousCall);
typedef ImageChunkListener = void Function(ImageChunkEvent event);

/// 图片块事件
class ImageChunkEvent extends Diagnosticable {
  const ImageChunkEvent({
    @required this.cumulativeBytesLoaded,
    @required this.expectedTotalBytes,
  });

  /// 当前已加载的字节数量
  final int cumulativeBytesLoaded;

  /// 需要载入的图片的总字节大小
  ///
  /// 此值无需和图片真正的大小相等，因为图片是可以被压缩的
  ///
  /// 如果字节数量未知，此值可以是 null
  final int expectedTotalBytes;
}

/// 用于接受加载图片通知的接口
///
/// 如果绑定的回调相同，那么认为两个  ImageStreamListener 相等
class ImageStreamListener {
  const ImageStreamListener(
    this.onImage, {
    this.onChunk,
    this.onError,
  }) : assert(onImage != null);

  /// 图片可用时的回调
  ///
  /// 此回调可能被触发许多次（例如，如果 [ImageStreamCompleter] 发送了许多次消息）。一种情况是多帧（GIF）的
  /// 图片，它会调用此回调多次 
  final ImageListener onImage;

  /// 在加载图像期间接收到一块字节时得到通知回调函数，
  ///
  /// 此回调可能被触发许多次 (例如，在 [NetworkImage] 中，载入的字节数量是随着时间增加的），或者，根本不会被
  /// 触发（例如，在 [MemoryImage] 中，图片字节已经在内存中可用）
  ///
  /// 甚至，此回调还会在 [onImage] 之后调用（例如，在多帧图片的情况下）
  final ImageChunkListener onChunk;
    
  final ImageErrorListener onError;

  @override
  int get hashCode => hashValues(onImage, onChunk, onError);

  /// 绑定的回调相同则认为是相同的
  @override
  bool operator ==(dynamic other) {
    if (other.runtimeType != runtimeType)
      return false;
    return other is ImageStreamListener
        && other.onImage == onImage
        && other.onChunk == onChunk
        && other.onError == onError;
  }
}
```



## ImageStreamCompleter

```dart
/// Base class for those that manage the loading of [dart:ui.Image] objects for
/// [ImageStream]s.
///
/// [ImageStreamListener] objects are rarely constructed directly. Generally, an
/// [ImageProvider] subclass will return an [ImageStream] and automatically
/// configure it with the right [ImageStreamCompleter] when possible.
abstract class ImageStreamCompleter extends Diagnosticable {
  final List<ImageStreamListener> _listeners = <ImageStreamListener>[];
  ImageInfo _currentImage;
  FlutterErrorDetails _currentError;

  @protected
  bool get hasListeners => _listeners.isNotEmpty;

  /// 添加一个监听回调
  void addListener(ImageStreamListener listener) {
    _listeners.add(listener);
    /// 如果当前已经有一张图片（被打包成 ImageInfo）完成了，则立刻调用新添加的监听者的 onImage 回调
    if (_currentImage != null) {
      try {
        listener.onImage(_currentImage, true);
      } catch (exception, stack) {
        ...
      }
    }
    /// 如果当前已经出现了一个错误，并且新添加的监听者能够处理错误，那么立刻调用其 onError 回调
    if (_currentError != null && listener.onError != null) {
      try {
        listener.onError(_currentError.exception, _currentError.stack);
      } catch (exception, stack) {
        ...
      }
    }
  }

  /// 移除一个监听者
  void removeListener(ImageStreamListener listener) {
    for (int i = 0; i < _listeners.length; i += 1) {
      if (_listeners[i] == listener) {
        _listeners.removeAt(i);
        break;
      }
    }
  }

  /// 通知所有的监听者，一张新的图片及已经完成了
  @protected
  void setImage(ImageInfo image) {
    _currentImage = image;
    if (_listeners.isEmpty)
      return;
    // 复制一份以允许并发修改。
    final List<ImageStreamListener> localListeners =
        List<ImageStreamListener>.from(_listeners);
    for (ImageStreamListener listener in localListeners) {
      try {
        listener.onImage(image, false);
      } catch (exception, stack) {
        ...
      }
    }
  }
  
  /// 错误报告，略
}
```

### OneFrameImageStreamCompleter

```dart
/// 管理单帧图片对象流的 completer
class OneFrameImageStreamCompleter extends ImageStreamCompleter {
  OneFrameImageStreamCompleter(
      Future<ImageInfo> image, {
      InformationCollector informationCollector 
  }) {
    image.then<void>(
        /// 在图片完成后，立刻调用 setImage，通知所有的监听者
        setImage, 
        /// 否则，报告一个错误，最后会触发监听者的 onError 回调
        onError: (dynamic error, StackTrace stack) {
      reportError(
        context: ErrorDescription('resolving a single-frame image stream'),
        exception: error,
        stack: stack,
        informationCollector: informationCollector,
        silent: true,
      );
    });
  }
}
```

### MultiFrameImageStreamCompleter

```dart
/// 管理和调度图片帧（多帧）
///
/// 只有当流中有已注册的侦听器时才会发出新帧(用[addListener]注册)。
///
/// 此类处理两种帧：
///
///  * 图片帧 - 动态图片的图片帧
///  * 应用帧 - Flutter 引擎绘制到屏幕上显示应用程序 GUI （图形用户界面）的帧。
///
/// 对于多帧图片，此 completer 只应该完成一次
///
/// 对于动画图像，这个类会急切地解码下一个图像帧，并通知监听器在第一个应用程序帧上已经准备好了新帧，这个应用程
/// 序帧是在图像帧持续时间过后调度的（对于动图来说，每当解码其一帧，会通知所有的监听者，告诉它们此图片的一帧已
/// 经完成，并调度一帧将其显示在屏幕上。随后，此类会立刻解码其下一帧，等待本帧的绘制完成之后，通知引擎调度新的
/// 一帧（用于显示新解码的一帧）。
///
/// 只从预定的应用程序帧调度新的定时器，确保我们在应用程序不可见时暂停动画(因为新的应用程序帧不会被调度)。
///
/// 时间线如下图所示:
///
///     | Time | Event                                      | Comment                   |
///     |------|--------------------------------------------|---------------------------|
///     | t1   | 调度 App 帧（图片帧 A 通知）                 |                           |
///     | t2   | App 帧调度完成                              |                           |
///     | t3   | App 帧调度完成                              |                           |
///     | t4   | 图片帧 B 解码完成                           |                           |
///     | t5   | App 帧调度完成                              | t5 - t1 < frameB_duration |
///     | t6   | 调度 App 帧（图片帧 B 通知）                 | t6 - t1 > frameB_duration |
///
class MultiFrameImageStreamCompleter extends ImageStreamCompleter {
  MultiFrameImageStreamCompleter({
    @required Future<ui.Codec> codec,
    @required double scale,
    Stream<ImageChunkEvent> chunkEvents,
    InformationCollector informationCollector,
  }) : _informationCollector = informationCollector,
       _scale = scale {
    codec.then<void>(
        /// 解码完成之后，立刻回调 _handleCodecReady
        _handleCodecReady, 
        /// 错误处理
        onError: (dynamic error, StackTrace stack) {
           ...
        }
      );
    });
    /// 如果提供了一个图片块事件流
    if (chunkEvents != null) {
      chunkEvents.listen(
        /// 添加一个监听器：当图片块事件来临的时候，通知每一个监听者的，触发其 onChunk 回调
        (ImageChunkEvent event) {
          if (hasListeners) {
            /// 将超类中的监听者列表拷贝一份，转换成 chunkListener 列表
            final List<ImageChunkListener> localListeners = _listeners
                .map<ImageChunkListener>((ImageStreamListener listener) => listener.onChunk)
                .where((ImageChunkListener chunkListener) => chunkListener != null)
                .toList();
            for (ImageChunkListener listener in localListeners) {
              listener(event);
            }
          }
        }, onError: (dynamic error, StackTrace stack) {
          ...
        },
      );
    }
  }

  ui.Codec _codec;
  final double _scale;
  final InformationCollector _informationCollector;
  ui.FrameInfo _nextFrame;
  // 当前帧出现的时间戳
  Duration _shownTimestamp;
  //当前帧请求的时间戳
  Duration _frameDuration;
  // 统计目前已发出（完成的）帧的数量
  int _framesEmitted = 0;
  Timer _timer;

  // Used to guard against registering multiple _handleAppFrame callbacks for the same frame.
  bool _frameCallbackScheduled = false;

  /// 解码完成后的立刻回调
  void _handleCodecReady(ui.Codec codec) {
    _codec = codec;
    assert(_codec != null);
    /// 如果有监听者，立刻解码下一帧
    if (hasListeners) {
      _decodeNextFrameAndSchedule();
    }
  }

  void _handleAppFrame(Duration timestamp) {
    _frameCallbackScheduled = false;
    if (!hasListeners)
      return;
    if (_isFirstFrame() || _hasFrameDurationPassed(timestamp)) {
      /// 发送图片完成通知
      _emitFrame(ImageInfo(image: _nextFrame.image, scale: _scale));
      _shownTimestamp = timestamp;
      _frameDuration = _nextFrame.duration;
      _nextFrame = null;
      final int completedCycles = _framesEmitted ~/ _codec.frameCount;
      /// 如果还有下一帧的话，继续调度
      if (_codec.repetitionCount == -1 || completedCycles <= _codec.repetitionCount) {
        _decodeNextFrameAndSchedule();
      }
      return;
    }
    final Duration delay = _frameDuration - (timestamp - _shownTimestamp);
    _timer = Timer(delay * timeDilation, () {
      _scheduleAppFrame();
    });
  }

  bool _isFirstFrame() {
    return _frameDuration == null;
  }

  bool _hasFrameDurationPassed(Duration timestamp) {
    assert(_shownTimestamp != null);
    return timestamp - _shownTimestamp >= _frameDuration;
  }

  Future<void> _decodeNextFrameAndSchedule() async {
    try {
      _nextFrame = await _codec.getNextFrame();
    } catch (exception, stack) {
      reportError(
        context: ErrorDescription('resolving an image frame'),
        exception: exception,
        stack: stack,
        informationCollector: _informationCollector,
        silent: true,
      );
      return;
    }
    if (_codec.frameCount == 1) {
      // 如果图片只有一帧（也就是单帧图片），则只发送一次通知
      _emitFrame(ImageInfo(image: _nextFrame.image, scale: _scale));
      return;
    }
    // 否则，需要调度一个 app 帧，用于显示图片的下一帧
    _scheduleAppFrame();
  }

  /// 调度一个 app 帧
  void _scheduleAppFrame() {
    if (_frameCallbackScheduled) {
      return;
    }
    _frameCallbackScheduled = true;
    /// SchedulerBinding.instance.scheduleFrameCallback 将给定的回调添加到瞬时帧回调列表中
    /// 并请求一帧
    SchedulerBinding.instance.scheduleFrameCallback(_handleAppFrame);
  }

  /// 发送图片帧，也就是通知所有的监听者，触发其 onImage 方法
  void _emitFrame(ImageInfo imageInfo) {
    setImage(imageInfo);
    _framesEmitted += 1;
  }

  @override
  void addListener(ImageStreamListener listener) {
    if (!hasListeners && _codec != null)
      _decodeNextFrameAndSchedule();
    super.addListener(listener);
  }

  @override
  void removeListener(ImageStreamListener listener) {
    super.removeListener(listener);
    if (!hasListeners) {
      _timer?.cancel();
      _timer = null;
    }
  }
}
```



## ImageStream

```dart
/// 图片资源的句柄
///
/// ImageStream 是 [dart:ui.Image] 对象和它的比例(一起由一个[ImageInfo]对象表示) 的句柄。底层图像对象可能会
/// 随着时间而改变，这可能是因为图像正在进行动画处理，也可能是因为底层图像资源发生了改变。
///
/// ImageStream 也能够呈现一个还未加载完成的图片
///
/// ImageStream 采用 [ImageStreamCompleter] 来实现
class ImageStream extends Diagnosticable {
  /// Create an initially unbound image stream.
  ///
  /// 一旦 [ImageStreamCompleter] 可用，调用 [setCompleter].
  ImageStream();

  /// 为此 ImageStream 设置的 completer
  ///
  /// 通常来说，无需直接管理此 completer
  ImageStreamCompleter get completer => _completer;
  ImageStreamCompleter _completer;

  List<ImageStreamListener> _listeners;

  /// 为此 [ImageStream] 指定一个特定的 [ImageStreamCompleter]
  ///
  /// 此方法通常被创建此 [ImageStream] 的 [ImageProvider] 自动调用
  ///
  /// 每个流只能调用此方法一次。要使一个 [ImageStream] 在一段时间内表示多个图像，请为它分配一个完成器，以连续
  /// 完成多个图像。
  void setCompleter(ImageStreamCompleter value) {
    _completer = value;
    if (_listeners != null) {
      final List<ImageStreamListener> initialListeners = _listeners;
      _listeners = null;
      /// 将已有所有回调添加到此 completer 的监听者中
      initialListeners.forEach(_completer.addListener);
    }
  }

  /// 添加一个监听器回调，每当一个新的具体 [ImageInfo] 对象可用时，都会调用该回调。如果一个具体的图片已经可
  /// 用，这个对象将同步调用监听器。
  ///
  /// 如果给定 [completer] 在其生命周期中完成了许多此，那么回调也会被触发许多次。
  ///
  /// {@template flutter.painting.imageStream.addListener}
  /// The listener will be passed a flag indicating whether a synchronous call
  /// occurred. If the listener is added within a render object paint function,
  /// then use this flag to avoid calling [RenderObject.markNeedsPaint] during
  /// a paint.
  /// 监听器将会传递一个标记，它指示了是否是一次同步的调用。如果监听器是在 渲染对象的绘制函数中被添加的，会使用
  /// 此标记来在绘制的过程中 避免调用 [RenderObject.markNeedsPaint]
  void addListener(ImageStreamListener listener) {
    if (_completer != null)
      return _completer.addListener(listener);
    _listeners ??= <ImageStreamListener>[];
    _listeners.add(listener);
  }

  /// 移除监听者
  void removeListener(ImageStreamListener listener) {
    if (_completer != null)
      return _completer.removeListener(listener);
    for (int i = 0; i < _listeners.length; i += 1) {
      if (_listeners[i] == listener) {
        _listeners.removeAt(i);
        break;
      }
    }
  }

  Object get key => _completer ?? this;
}
```





## ImageProvider

```dart
/// 标识图像，但无法保证是最终精确的 Asset。这允许识别一组图像，并根据环境(如设备像素比)稍后解析精确的图像。
///
/// 为了从 [ImageProvider] 获取到 [ImageStream]，调用 [resolve]，将它传递给 [ImageConfiguration] 对象
///
/// [ImageProvider] 使用全局 [imageCache] 来缓存图像
///
/// 类型参数“T”是用于表示已解析配置的对象的类型。这也是用于图像缓存中的键的类型. 
/// 
/// 它应当是不可变的，并且已经实现了 [==] 运算符，和 [hashCode] getter. 
/// 子类应该以一个直接 T 参数来继承 [ImageProvider] 
///
/// 在使用类型作为参数时(其中任何图像提供程序都是可接受的)，不必指定类型参数
///
/// 加载流程如下：
/// 1、首先 Image 通过 ImageProvider 得到 ImageStream 对象
/// 2、然后 _ImageState 利用 ImageStream 添加监听，等待图片数据
/// 3、接着 ImageProvider 通过 load 方法去加载并返回 ImageStreamCompleter 对象
/// 4、然后 ImageStream 会关联  ImageStreamCompleter
/// 5、之后 ImageStreamCompleter 会通过 http 下载图片，再经过 PaintingBinding 编码转化后，得到 ui.Codec ///    可绘制对象，并封装成 ImageInfo 返回
/// 6、接着 ImageInfo 回调到  ImageStream 的监听，设置给 _ImageState  build 的 RawImage 对象。
/// 7、最后 RawImage 的 RenderImage 通过 paint 方法绘制 ImageInfo 中的 Uint8List(图像数据)
@optionalTypeArgs
abstract class ImageProvider<T> {
  const ImageProvider();

  /// 使用给定的配置来完成此图像，并且返回 [ImageStream].
  ///
  /// This is the public entry-point of the [ImageProvider] class hierarchy.
  ///
  /// 子类应当实现 [obtainKey] 和 [load]，它们被此方法所使用
  /// 
  /// 注意：
  /// 此方法通常被 Image 组件调用，用于获取图片流。可以监此图片流来获取图片加载事件
  ImageStream resolve(ImageConfiguration configuration) {
    final ImageStream stream = ImageStream();
    T obtainedKey;
    bool didError = false;
    
    /// 错误处理方法
    Future<void> handleError(dynamic exception, StackTrace stack) async {
      if (didError) {
        return;
      }
      didError = true;
      await null; // wait an event turn in case a listener has been added to the image stream.
      final _ErrorImageCompleter imageCompleter = _ErrorImageCompleter();
      stream.setCompleter(imageCompleter);
      imageCompleter.setError(
        ...
      );
    }

    // 如果在监听器非没有被添加之前，向一个 synchronous completer 中添加了错误的话，它会向 zone 和 栈堆中各
    // 抛出一个异常。因此，这样看起来异常被捕获了，但是实际上却冒泡到了 zone。因为我们无法阻止在这里使用 
    // Completer.sync（否则，将会做出破坏性的变动），我们转而关联相同的 Zone 机制来拦截未捕获的错误并将其提交
    // 给图像流的 error handler。注意：这些错误可能被拷贝一次，因此需要 didError 标记位
    // 
    /// Zone.fork 返回了一个从当前 Zone 的拷贝 zone，允许程序的隔离运行
    /// 其中，ZoneSpecification 是一个配置类，指定了 zone 的配置信息
    final Zone dangerZone = Zone.current.fork(
      specification: ZoneSpecification(
        handleUncaughtError: (
            Zone zone, 
            ZoneDelegate delegate, 
            Zone parent, 
            Object error, 
            StackTrace stackTrace
        ) {
          handleError(error, stackTrace);
        }
      )
    );
    /// 在创建的新 zone 运行以下函数
    /// 
    /// 注意：runGuarded 方法有如下注释
    /// 执行 [action] ，并且捕获同步错误（和异步错误。在通常情况下，try 不会捕获异步错误，因此）
    /**
    * 类似于如下的代码
    * ```dart
    * try {
    *   this.run(action);
    * } catch (e, s) {
    *   this.handleUncaughtError(e, s);
    * }
    * ```
    */
    dangerZone.runGuarded(() {
      Future<T> key;
      try {
        key = obtainKey(configuration);
      } catch (error, stackTrace) {
        handleError(error, stackTrace);
        return;
      }
      key.then<void>((T key) {
        obtainedKey = key;
        /// 注意，这里实现了图片缓存
        /// 将获取到的 key 作为键值，返回的 completer 作为值添加到图片缓存映射表中
        /// 此外，load 方法的参数之一 codec 是 PaintingBinding.instance.instantiateImageCodec 的返回值
        final ImageStreamCompleter completer = PaintingBinding.instance.imageCache.putIfAbsent(
          key,
          () => load(key, PaintingBinding.instance.instantiateImageCodec),
          onError: handleError,
        );
          
        if (completer != null) {
          stream.setCompleter(completer);
        }
      }).catchError(handleError);
    });
    return stream;
  }

  /// 从图片缓存中移除一个条目
  ///
  /// 返回值表示了是否成功地移除
  ///
  /// 传递给 [Image] widget 的 [ImageProvider] 实例可以不同，但是它们必须有相同的 key
  ///
  /// [cache] 是可选的，且默认设置为全局图片缓存
  ///
  /// [configuration] 是可选的，且默认设置为 [ImageConfiguration.empty].
  ///
  /// 例：
  ///
  /// 下面的示例代码展示了如何使用带有对应 URL 的 [NetworkImage] 清除使用 [image] 组件加载的图像。
  ///
  /// ```dart
  /// class MyWidget extends StatelessWidget {
  ///   final String url = '...';
  ///
  ///   @override
  ///   Widget build(BuildContext context) {
  ///     return Image.network(url);
  ///   }
  ///
  ///   void evictImage() {
  ///     final NetworkImage provider = NetworkImage(url);
  ///     provider.evict().then<void>((bool success) {
  ///       if (success)
  ///         debugPrint('removed image!');
  ///     });
  ///   }
  /// }
  /// ```
  Future<bool> evict({ 
      ImageCache cache, 
      ImageConfiguration configuration = ImageConfiguration.empty 
  }) async {
    cache ??= imageCache;
    final T key = await obtainKey(configuration);
    return cache.evict(key);
  }

  /// 将ImageProvider的设置和ImageConfiguration转换为描述要加载的精确图像的键。
  ///
  /// key 的类型由子类来确定（注意，这里的 T 就是子类继承的时候指定的 T）。它是一个无歧义地表示 [load] 方法加
  /// 载的图片的值。若不同的 [ImageProvider] 给出了同样的构造器参数和 [ImageConfiguration] 配置，它们返回的 
  /// key 值应该满足 '==' 运算符（子类描述的 key 可能会自行实现 [==]）。
  Future<T> obtainKey(ImageConfiguration configuration);

  /// 将给定的 key 转换为 [ImageStreamCompleter], 并且开始获取图片
  ///
  /// [decode] 回调提供了获取图片解码的逻辑
  @protected
  ImageStreamCompleter load(T key, DecoderCallback decode);
}

```



## _network_image.io.NetWorkImage

```dart
import "image_provider.dart" as image_provider;

/// [image_provider.NetworkImage] 的实现类
class NetworkImage 
    extends image_provider.ImageProvider<image_provider.NetworkImage> 
    implements image_provider.NetworkImage 
{
  const NetworkImage(this.url, { this.scale = 1.0, this.headers });

  @override
  final String url;
  @override
  final double scale;
  @override
  final Map<String, String> headers;

  /// 注意传递给 ImageProvider 的参数是 image_provider.NetworkImage，而自身又实现了接口。故采用 
  /// SynchronousFuture 返回 this
  @override
  Future<NetworkImage> obtainKey(image_provider.ImageConfiguration configuration) {
    return SynchronousFuture<NetworkImage>(this);
  }

  @override
  ImageStreamCompleter load(image_provider.NetworkImage key, image_provider.DecoderCallback decode) {
    /// 图片块事件流
    final StreamController<ImageChunkEvent> chunkEvents = StreamController<ImageChunkEvent>();

    return MultiFrameImageStreamCompleter(
      codec: _loadAsync(key as NetworkImage, chunkEvents, decode),
      chunkEvents: chunkEvents.stream,
      scale: key.scale,
      informationCollector: () {
        return <DiagnosticsNode>[
          DiagnosticsProperty<image_provider.ImageProvider>('Image provider', this),
          DiagnosticsProperty<image_provider.NetworkImage>('Image key', key),
        ];
      },
    );
  }

  // 我们将 ‘autoUncompress’ 设置为false，以确保我们可以信任 ‘Content-Length’ HTTP 头的值。我们自动解压在
  // 我们的调用的内容为 [consolidateHttpClientResponseBytes]
  static final HttpClient _sharedHttpClient = HttpClient()..autoUncompress = false;

  static HttpClient get _httpClient {
    HttpClient client = _sharedHttpClient;
    return client;
  }

  Future<ui.Codec> _loadAsync(
    NetworkImage key,
    StreamController<ImageChunkEvent> chunkEvents,
    image_provider.DecoderCallback decode,
  ) async {
    try {
      final Uri resolved = Uri.base.resolve(key.url);
      /// 获取请求
      final HttpClientRequest request = await _httpClient.getUrl(resolved);
      headers?.forEach((String name, String value) {
        request.headers.add(name, value);
      });
      /// 得到相应
      final HttpClientResponse response = await request.close();
      /// 如果请求不成功成功
      if (response.statusCode != HttpStatus.ok)
        throw image_provider.NetworkImageLoadException(
          statusCode: response.statusCode, uri: resolved
      );

      /// 等待将响应体转换成 Uint8List
      final Uint8List bytes = await consolidateHttpClientResponseBytes(
        response,
        /// 此回调用于向 chunkEvent 流中添加事件，也就是说，每收到一个图片块，会添加一个 chunkEvent
        /// 在 MultiFrameImageStreamCompleter 的构造函数中，为 chunkEvent 流添加了监听。综上，每当收到一个
        /// 图片块，会触发已经注册的 ImageStreamListener 的 onchunk 回调
        onBytesReceived: (int cumulative, int total) {
          chunkEvents.add(ImageChunkEvent(
            cumulativeBytesLoaded: cumulative,
            expectedTotalBytes: total,
          ));
        },
      );
      if (bytes.lengthInBytes == 0)
        throw Exception('NetworkImage is an empty file: $resolved');
      /// 解码
      return decode(bytes);
    } finally {
      /// 关闭流，释放资源
      chunkEvents.close();
    }
  }

  @override
  /// 当 url 和 scale 相同的时候，就认为两个 NetworkImage 是相等的
  bool operator ==(dynamic other) {
    if (other.runtimeType != runtimeType)
      return false;
    return other is NetworkImage
        && other.url == url
        && other.scale == scale;
  }

  @override
  int get hashCode => ui.hashValues(url, scale);

  @override
  String toString() => '$runtimeType("$url", scale: $scale)';
}

```



## CachedNetWorkImageProvider

```dart
/// 提供了缓存功能的网络图片加载器
class CachedNetworkImageProvider
    extends ImageProvider<CachedNetworkImageProvider> 
{
  const CachedNetworkImageProvider(
      this.url,{
      this.scale: 1.0, 
      this.errorListener, 
      this.headers, 
      this.cacheManager
   });

  final BaseCacheManager cacheManager;

  final String url;
  final double scale;
  final ErrorListener errorListener;
  final Map<String, String> headers;

  /// obtainKey 的方法是实现大同小异，都是返回 SynchronousFuture(this);
  @override
  Future<CachedNetworkImageProvider> obtainKey(
      ImageConfiguration configuration) {
    return SynchronousFuture<CachedNetworkImageProvider>(this);
  }

  
  @override
  ImageStreamCompleter load(
      CachedNetworkImageProvider key, DecoderCallback decode) {
    return MultiFrameImageStreamCompleter(
      codec: _loadAsync(key),
      scale: key.scale,
      informationCollector: () sync* {
        yield DiagnosticsProperty<ImageProvider>(
          'Image provider: $this \n Image key: $key',
          this,
          style: DiagnosticsTreeStyle.errorProperty,
        );
      },
    );
  }

  Future<ui.Codec> _loadAsync(CachedNetworkImageProvider key) async {
    var mngr = cacheManager ?? DefaultCacheManager();
    /// 从缓存中读取文件，如果没有，则会发起一次请求来获取文件并且加入缓存
    var file = await mngr.getSingleFile(url, headers: headers);
    /// 从上一步得到的文件必须不为 null，否则抛出异常
    if (file == null) {
      if (errorListener != null) 
          errorListener();
      return Future<ui.Codec>.error("Couldn't download or retrieve file.");
    }
    return await _loadAsyncFromFile(key, file);
  }

  /// 解码文件
  Future<ui.Codec> _loadAsyncFromFile(
      CachedNetworkImageProvider key, 
      File file
  ) async {
    final Uint8List bytes = await file.readAsBytes();

    if (bytes.lengthInBytes == 0) {
      if (errorListener != null) errorListener();
      throw Exception("File was empty");
    }

    return await ui.instantiateImageCodec(bytes);
  }

  @override
  bool operator ==(dynamic other) {
    if (other.runtimeType != runtimeType) return false;
    final CachedNetworkImageProvider typedOther = other;
    return url == typedOther.url && scale == typedOther.scale;
  }

  @override
  int get hashCode => hashValues(url, scale);

  @override
  String toString() => '$runtimeType("$url", scale: $scale)';
}

```



## MemoryImage

```dart
/// Decodes the given [Uint8List] buffer as an image, associating it with the
/// given scale.
///
/// The provided [bytes] buffer should not be changed after it is provided
/// to a [MemoryImage]. To provide an [ImageStream] that represents an image
/// that changes over time, consider creating a new subclass of [ImageProvider]
/// whose [load] method returns a subclass of [ImageStreamCompleter] that can
/// handle providing multiple images.
///
/// See also:
///
///  * [Image.memory] for a shorthand of an [Image] widget backed by [MemoryImage].
class MemoryImage extends ImageProvider<MemoryImage> {
  /// Creates an object that decodes a [Uint8List] buffer as an image.
  ///
  /// The arguments must not be null.
  const MemoryImage(this.bytes, { this.scale = 1.0 })
    : assert(bytes != null),
      assert(scale != null);

  /// The bytes to decode into an image.
  final Uint8List bytes;

  /// The scale to place in the [ImageInfo] object of the image.
  final double scale;

  @override
  Future<MemoryImage> obtainKey(ImageConfiguration configuration) {
    return SynchronousFuture<MemoryImage>(this);
  }

  @override
  ImageStreamCompleter load(MemoryImage key, DecoderCallback decode) {
    return MultiFrameImageStreamCompleter(
      codec: _loadAsync(key, decode),
      scale: key.scale,
    );
  }

  Future<ui.Codec> _loadAsync(MemoryImage key, DecoderCallback decode) {
    assert(key == this);

    return decode(bytes);
  }

  @override
  bool operator ==(dynamic other) {
    if (other.runtimeType != runtimeType)
      return false;
    return other is MemoryImage
        && other.bytes == bytes
        && other.scale == scale;
  }

  @override
  int get hashCode => hashValues(bytes.hashCode, scale);

  @override
  String toString() => '$runtimeType(${describeIdentity(bytes)}, scale: $scale)';
}

```



## NetWorkImage

```dart
/// 从给定的 url 获取资源，并将它关联到给的给定的尺寸
///
/// 无论来自服务器的缓存头文件是什么，图片都将被缓存。
///
/// 当网络图像在Web平台上使用时，[DecoderCallback] 的 [cacheWidth] 和 [cacheHeight]参数被忽略，因为Web引擎
/// 将网络图像的图像解码委托给 Web ，而 Web 不支持自定义的解码大小。
/// 
/// 注意：这里提供给 ImageProvider<T> 的参数 T，就是 NetworkImage
abstract class NetworkImage extends ImageProvider<NetworkImage> {
  /// url 和 scale 参数不能为 null
  ///
  /// 注意以下的构造方式，实际上是构造了一个 _network_image.io.NetWorkImage 对象（构造了子类对象）
  const factory NetworkImage(String url, { double scale, Map<String, String> headers }) 
      = network_image.NetworkImage;

  String get url;
  double get scale;

  /// Http 请求头
  Map<String, String> get headers;

  @override
  ImageStreamCompleter load(NetworkImage key, DecoderCallback decode);
}
```



## AssetBundleImageProvider

```dart

/// A subclass of [ImageProvider] that knows about [AssetBundle]s.
///
/// This factors out the common logic of [AssetBundle]-based [ImageProvider]
/// classes, simplifying what subclasses must implement to just [obtainKey].
abstract class AssetBundleImageProvider extends ImageProvider<AssetBundleImageKey> {
  /// Abstract const constructor. This constructor enables subclasses to provide
  /// const constructors so that they can be used in const expressions.
  const AssetBundleImageProvider();

  /// Converts a key into an [ImageStreamCompleter], and begins fetching the
  /// image using [loadAsync].
  @override
  ImageStreamCompleter load(AssetBundleImageKey key, DecoderCallback decode) {
    return MultiFrameImageStreamCompleter(
      codec: _loadAsync(key, decode),
      scale: key.scale,
      informationCollector: () sync* {
        yield DiagnosticsProperty<ImageProvider>('Image provider', this);
        yield DiagnosticsProperty<AssetBundleImageKey>('Image key', key);
      },
    );
  }

  /// Fetches the image from the asset bundle, decodes it, and returns a
  /// corresponding [ImageInfo] object.
  ///
  /// This function is used by [load].
  @protected
  Future<ui.Codec> _loadAsync(AssetBundleImageKey key, DecoderCallback decode) async {
    final ByteData data = await key.bundle.load(key.name);
    if (data == null)
      throw 'Unable to read data';
    return await decode(data.buffer.asUint8List());
  }
}
```



## FIleImage

```dart
class FileImage extends ImageProvider<FileImage> {
  /// Creates an object that decodes a [File] as an image.
  ///
  /// The arguments must not be null.
  const FileImage(this.file, { this.scale = 1.0 });

  /// The file to decode into an image.
  final File file;

  /// The scale to place in the [ImageInfo] object of the image.
  final double scale;

  @override
  Future<FileImage> obtainKey(ImageConfiguration configuration) {
    return SynchronousFuture<FileImage>(this);
  }

  @override
  ImageStreamCompleter load(FileImage key, DecoderCallback decode) {
    return MultiFrameImageStreamCompleter(
      codec: _loadAsync(key, decode),
      scale: key.scale,
      informationCollector: () sync* {
        yield ErrorDescription('Path: ${file?.path}');
      },
    );
  }

  Future<ui.Codec> _loadAsync(FileImage key, DecoderCallback decode) async {
    assert(key == this);

    final Uint8List bytes = await file.readAsBytes();
    if (bytes.lengthInBytes == 0)
      return null;

    return await decode(bytes);
  }

  @override
  bool operator ==(dynamic other) {
    if (other.runtimeType != runtimeType)
      return false;
    return other is FileImage
        && other.file?.path == file?.path
        && other.scale == scale;
  }

  @override
  int get hashCode => hashValues(file?.path, scale);

  @override
  String toString() => '$runtimeType("${file?.path}", scale: $scale)';
}
```



## ResizeImage

```dart
/// 命令 Flutter 按照给定的维度来解码图片，而不是其原始大小
///
/// 这可以更好地控制图片在 [ImageCache] 中的大小，并且通常用于 [ImageCache] 的内存占用
///
/// 解码后的图像仍然可以显示在这里提供的缓存大小之外的大小。
class ResizeImage extends ImageProvider<_SizeAwareCacheKey> {
  const ResizeImage(
    this.imageProvider, {
    this.width,
    this.height,
  });

  /// 类似于代理模式，包裹了一层 ImageProvider
  final ImageProvider imageProvider;

  /// 图片应该被解码且缓存的宽度
  final int width;

  /// 图片应该被解码且缓存的高度
  final int height;

  /// 压缩给定的 `provider`，只在 cacheWidth 和 cacheHeight 不全为 null 时有效
  static ImageProvider<dynamic> resizeIfNeeded(
      int cacheWidth, 
      int cacheHeight, 
      ImageProvider<dynamic> provider
  ) {
    if (cacheWidth != null || cacheHeight != null) {
      return ResizeImage(
          provider, 
          width: cacheWidth, 
          height: cacheHeight
      );
    }
    return provider;
  }

  @override
  ImageStreamCompleter load(_SizeAwareCacheKey key, DecoderCallback decode) {
    final DecoderCallback decodeResize = (
        Uint8List bytes, {
        int cacheWidth, 
        int cacheHeight
    }) {
      return decode(bytes, cacheWidth: width, cacheHeight: height);
    };
    return imageProvider.load(key.providerCacheKey, decodeResize);
  }

  @override
  Future<_SizeAwareCacheKey> obtainKey(ImageConfiguration configuration) async {
    final Object providerCacheKey = await imageProvider.obtainKey(configuration);
    return _SizeAwareCacheKey(providerCacheKey, width, height);
  }
}
```

### SizeAwareCacheKey

```dart
/// 通常情况下，imageProvider<T>.obtainKey 返回的是 SynchronousFuture(this)
/// 此类的实现将缓存的高度和宽度一并包含，作为 key 映射到图片缓存
class _SizeAwareCacheKey {
  const _SizeAwareCacheKey(this.providerCacheKey, this.width, this.height);

  final Object providerCacheKey;

  final int width;

  final int height;

  @override
  bool operator ==(Object other) {
    if (other.runtimeType != runtimeType)
      return false;
    return other is _SizeAwareCacheKey
        && other.providerCacheKey == providerCacheKey
        && other.width == width
        && other.height == height;
  }

  @override
  int get hashCode => hashValues(providerCacheKey, width, height);
}
```


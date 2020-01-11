# Flutter ImageCache 源码分析

## ImageCache
```dart
/// 默认的图片缓存大小：
/// 最大图片数量：1000
const int _kDefaultSize = 1000;
/// 最大图片内存占用 100 MB
const int _kDefaultSizeBytes = 100 << 20; // 100 MiB

/// 图片缓存类
///
/// 实现了对最近使用的 1000 图片（或者总内存占用达 100MB 的缓存）。最大数量可以使用 [maximumSize] 和 
/// [maximumSizeBytes] 来调整。活动状态下的图片（例如，应用程序持有对该图片的应用，如 [ImageStream] 对象,
/// [ImageStreamCompleter] 对象, [ImageInfo] 对象, or raw [dart:ui.Image] 对象），可能被从缓存中移除（因
/// 此，如果它们在 [putIfAbsent] 方法中被引用，则需要从网络中重新获取）但是只要应用程序在使用它们，它们原始的 
/// bit 信息就会保存在内存中。
/// 
/// [putIfAbsent] 是缓存 Api 的进入点。它返回了之前对于给定键值缓存的 [ImageStreamCompleter] 对象（如果可用
/// 的话）。否则，它会首先调用给定的回调来获取对象。两种情况下，key 被移动到“最近使用的位置”。
///
/// 通常，这个类不被字节使用。[ImageProvider] 类和它的子类自动地处理缓存图片
///
/// [PaintingBinding] 维护了这个缓存的单例，并且可以通过 [imageCache] 属性来获取 
///
/// 可以通过如下方式来自定义缓存逻辑
/// ```dart
/// /// This is the custom implementation of [ImageCache] where we can override
/// /// the logic.
/// class MyImageCache extends ImageCache {
///   @override
///   void clear() {
///     print("Clearing cache!");
///     super.clear();
///   }
/// }
///
/// class MyWidgetsBinding extends WidgetsFlutterBinding {
///   @override
///   ImageCache createImageCache() => MyImageCache();
/// }
///
/// void main() {
///   // 此构造器会创建唯一单例，因此，runApp 方法会返回已创建的单例
///   MyWidgetsBinding();
///   runApp(MyApp());
/// }
///
/// class MyApp extends StatelessWidget {
///   @override
///   Widget build(BuildContext context) {
///     return Container();
///   }
/// }
/// ```

class ImageCache {
  /// 默认构建 HashMap
  final Map<Object, _PendingImage> _pendingImages = <Object, _PendingImage>{};
  final Map<Object, _CachedImage> _cache = <Object, _CachedImage>{};

  int get maximumSize => _maximumSize;
  int _maximumSize = _kDefaultSize;
    
  /// 改变最大缓存数量
  ///
  /// 缩小缓存的数量会立即移除额外的元素。设置成 0 会立刻清除缓存
  set maximumSize(int value) {
    assert(value != null);
    assert(value >= 0);
    if (value == maximumSize)
      return;
    _maximumSize = value;
    if (maximumSize == 0) {
      clear();
    } else {
      _checkCacheSize();
    }
  }

  /// 当前缓存数量
  int get currentSize => _cache.length;

  /// 最大字节
  int get maximumSizeBytes => _maximumSizeBytes;
  int _maximumSizeBytes = _kDefaultSizeBytes;
  /// 缩小缓存大小会立即移除额外的元素
  set maximumSizeBytes(int value) {
    if (value == _maximumSizeBytes)
      return;
    _maximumSizeBytes = value;
    if (_maximumSizeBytes == 0) {
      clear();
    } else {
      _checkCacheSize();
    }
  }

  /// 当前缓存大小
  int get currentSizeBytes => _currentSizeBytes;
  int _currentSizeBytes = 0;

  /// 清除缓存
  void clear() {
    _cache.clear();
    _pendingImages.clear();
    _currentSizeBytes = 0;
  }

  /// 根据 key 来移除一例缓存，返回 true 则移除成功
  /// 等待完成的挂起图像也将被删除，如果成功则返回 true。
  ///
  /// 当一个挂起的图片被移除时，它上面的监听器也会被移除，以防止它在最终完成时将自己添加到缓存中。
  ///
  /// [key] 必须与使用 [ImageCache.putIfAbsent] 注册时的对象之一相等
  ///
  /// 如果 key 不是立即可用的（例如 future）,通常情况下, 使用 [ImageProvider.evict] 来间接调用此方法
  bool evict(Object key) {
    final _PendingImage pendingImage = _pendingImages.remove(key);
    if (pendingImage != null) {
      /// 若图片是挂起状态，需要移除其监听，以防它在完成的时候将自身添加到 _cache 中
      ///  
      /// 见 [putIfAbsent] 方法中的 [listener] 回调声明
      pendingImage.removeListener();
      return true;
    }
    final _CachedImage image = _cache.remove(key);
    if (image != null) {
      _currentSizeBytes -= image.sizeBytes;
      return true;
    }
    return false;
  }

  /// 返回了之前对于给定键值缓存的 [ImageStreamCompleter] 对象（如果可用的话）。否则，它会首先调用给定的回调
  /// 来获取对象。两种情况下，key 被移动到“最近使用的位置”。
  ///
  /// 发生错误的时候回调 onError 方法，并且返回 null，而不是 ImageStreamCompleter
  ImageStreamCompleter putIfAbsent(
      Object key, 
      ImageStreamCompleter loader(), { 
      ImageErrorListener onError
  }) {
    /// 首先在挂起图片中查找
    ImageStreamCompleter result = _pendingImages[key]?.completer;
    // 若找到，直接返回，因为图片还没有被加载完成
    if (result != null)
      return result;
    // 将给定的 key 从表中移除，然后移动到“最近使用的位置”
    final _CachedImage image = _cache.remove(key);
    if (image != null) {
      /// 改变位置
      _cache[key] = image;
      return image.completer;
    }
    /// 否则，回调 loader 来获取图片资源
    try {
      result = loader();
    } catch (error, stackTrace) {
      if (onError != null) {
        onError(error, stackTrace);
        return null;
      } else {
        rethrow;
      }
    }
      
    /// 定义 listener 回调，将 completer 从 _pedingImages 中移除，并且添加到  _cache 中，计算大小和数量限
    /// 制
    void listener(ImageInfo info, bool syncCall) {
      // 加载失败的图片不占用内存大小
      final int imageSize = info?.image == null ? 0 : info.image.height * info.image.width * 4;
      // 保存结果和其大小作为缓存条目
      final _CachedImage image = _CachedImage(result, imageSize);
      // 如果单张图片的大小 大于最大 缓存大小的话，增加 1000，即一 kb
      if (maximumSizeBytes > 0 && imageSize > maximumSizeBytes) {
        _maximumSizeBytes = imageSize + 1000;
      }
      _currentSizeBytes += imageSize;
      final _PendingImage pendingImage = _pendingImages.remove(key);
      /// 将此 listender 从 completer 中移除，防止内存泄漏 
      if (pendingImage != null) {
        pendingImage.removeListener();
      }
      /// 加入到缓存中
      _cache[key] = image;
      _checkCacheSize();
    }
      
    /// 将声明的 listener 作为 ImageStreamListener 的 onImage,并将 ImageStreamListener 加入到 result 的
    /// 监听者中。这样，当图片加载完成之后，自动回调 listener，把 completer 添加到 _cache 中
    if (maximumSize > 0 && maximumSizeBytes > 0) {
      final ImageStreamListener streamListener = ImageStreamListener(listener);
       /// 添加到挂起图片缓存中
      _pendingImages[key] = _PendingImage(result, streamListener);
      result.addListener(streamListener);
    }
    return result;
  }

  // 从缓存中移除图片，直到长度和大小都低于最大值。或者缓存为 空
  /// 注意：
  /// 这个方法实际上只清理了 _cache 中的内存
  void _checkCacheSize() {
    while (_currentSizeBytes > _maximumSizeBytes || _cache.length > _maximumSize) {
      final Object key = _cache.keys.first;
      final _CachedImage image = _cache[key];
      _currentSizeBytes -= image.sizeBytes;
      _cache.remove(key);
    }
  }
}
```



## _CachedImage

```dart
/// 缓存图片条目，保存了 completer 和 图片大小
class _CachedImage {
  _CachedImage(this.completer, this.sizeBytes);

  final ImageStreamCompleter completer;
  final int sizeBytes;
}
```



## _PendingImage

```dart
/// 挂起图片条目，保存了 completer 和其对应的 onImage 回调
class _PendingImage {
  _PendingImage(this.completer, this.listener);

  final ImageStreamCompleter completer;
  final ImageStreamListener listener;

  void removeListener() {
    completer.removeListener(listener);
  }
}
```
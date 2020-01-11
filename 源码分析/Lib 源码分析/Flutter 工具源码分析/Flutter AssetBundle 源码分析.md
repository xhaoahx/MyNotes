# Flutter AssetBundle 源码分析



## AssetBundle

```dart
/// 被应用使用的资源的集合
///
///  AssetBundle 包含了一系列例如 图片、字符串 的资源，它们能够被引用程序使用。访问这些资源是异步进行的，因
/// 此，它们能够在无感知的情况下加载，（例如，通过网络，即 [NetworkAssetBundle]；或者，从本地文件系统中加
/// 载）因此，它们不会阻塞 UI 进程
///
/// 应用程序拥有 [rootBundle], 它包含了应用程序被构建时的资源。要将资源添加到应用程序的[rootBundle]中，请将
/// 它们添加到应用程序的“pubspec”的“flutter”部分的“assets”部分。yaml的清单。
///
/// 例如：
///
/// ```yaml
/// name: my_awesome_application
/// flutter:
///   assets:
///    - images/hamilton.jpeg
///    - images/lafayette.jpeg
/// ```
///
/// 与其全局静态直接访问 [rootBundle]，考虑在当前 [BuildContext] 上使用 [DefaultAssetBundle.of] 来获取 
/// [AssetBundle]，间接层允许祖先部件在运行期间替换不同的 [AssetBundle] (例如测试或者本地化) 
/// 通常来说，[WidgetsApp] 或者 [MaterialApp] 配置 [DefaultAssetBundle] 作为 [rootBundle].
abstract class AssetBundle {
  /// 从 asset bundle 中获取二进制资源作为数据流
  ///
  /// 如果给定的资源没有被找到，抛出一个异常
  Future<ByteData> load(String key);

  /// 从 asset bundle 中获取一个字符串
  ///
  /// 如果 `cache` 参数设置成 false，则不会进行缓存。这取决与子类的实现
  Future<String> loadString(String key, { bool cache = true }) async {
    final ByteData data = await load(key);
    if (data == null)
      throw FlutterError('Unable to load asset: $key');
    /// 小于 10 kb 的资源直接解码
    if (data.lengthInBytes < 10 * 1024) {
      // 在像素2 XL上解析10KB大约需要3ms。
      return utf8.decode(data.buffer.asUint8List());
    }
    /// 否则，采用 compute 的方式进行解码
    return compute(_utf8decode, data, debugLabel: 'UTF8 decode for "$key"');
  }

  static String _utf8decode(ByteData data) {
    return utf8.decode(data.buffer.asUint8List());
  }

  /// 从 asset bundle 中获取一个字符串，使用给定的函数进行解码，并返回其结果
  ///
  /// 实现可能会缓存结果，因此在资产包的生命周期内，特定的键只能与一个解析器一起使用。
  Future<T> loadStructuredData<T>(String key, Future<T> parser(String value));

  /// 如果是一个缓存的 asset bundle，并且给定的键对应了一个已经缓存的 asset，那么将该 asset 从缓存中移除，以
  /// 便下一次加载它时，将从 asset bundle 中重新读取。
  void evict(String key) { }

  @override
  String toString() => '${describeIdentity(this)}()';
}

```



## NetworkAssetBundle

```dart
/// 通过网络来加载 asset bundle
///
/// 此 asset bundle 并不缓存任何的资源，因为底层的网络实现可能完成了一定量的缓存工作
class NetworkAssetBundle extends AssetBundle {

  NetworkAssetBundle(Uri baseUrl)
    : _baseUrl = baseUrl,
      /// 注意这里使用  dart.io 的 http 库
      _httpClient = HttpClient();

  final Uri _baseUrl;
  final HttpClient _httpClient;

  Uri _urlFromKey(String key) => _baseUrl.resolve(key);

  @override
  Future<ByteData> load(String key) async {
    final HttpClientRequest request = await _httpClient.getUrl(_urlFromKey(key));
    final HttpClientResponse response = await request.close();
    /// 校验状态码，如果失败则抛出异常
    if (response.statusCode != HttpStatus.ok)
      throw FlutterError.fromParts(<DiagnosticsNode>[
        ErrorSummary('Unable to load asset: $key'),
        IntProperty('HTTP status code', response.statusCode),
      ]);
    /// 高效地将 [HttpClientResponse] 的相应体转化成 [Uint8List]
    final Uint8List bytes = await consolidateHttpClientResponseBytes(response);
    return bytes.buffer.asByteData();
  }

  @override
  Future<T> loadStructuredData<T>(String key, Future<T> parser(String value)) async {
    assert(key != null);
    assert(parser != null);
    return parser(await loadString(key));
  }

  // TODO(ianh): Once the underlying network logic learns about caching, we
  // should implement evict().

  @override
  String toString() => '${describeIdentity(this)}($_baseUrl)';
}

```



## CachingAssetBundle 

```dart
/// 一个能够永久缓存以获取到的字符串或者结构化的资源 [AssetBundle] 
///
/// 字符串 ([loadString] and [loadStructuredData]) 被解码成 UTF-8.
///
/// 从 [load] 中获取的数据不会被缓存 
abstract class CachingAssetBundle extends AssetBundle {
  /// 此实现方式可能被替换成智能缓存
  final Map<String, Future<String>> _stringCache = <String, Future<String>>{};
  final Map<String, Future<dynamic>> _structuredDataCache = <String, Future<dynamic>>{};
  
  @override
  Future<String> loadString(String key, { bool cache = true }) {
    if (cache)
      /// 通过 key 对应把载入的字符串加入缓存
      return _stringCache.putIfAbsent(key, () => super.loadString(key));
    return super.loadString(key);
  }

  /// Retrieve a string from the asset bundle, parse it with the given function,
  /// and return the function's result.
  ///
  /// The result of parsing the string is cached (the string itself is not,
  /// unless you also fetch it with [loadString]). For any given `key`, the
  /// `parser` is only run the first time.
  ///
  /// 一旦解析了该值，该函数为后续调用返回的future将是[SynchronousFuture]，它将同步地解析它的回调。
  @override
  Future<T> loadStructuredData<T>(String key, Future<T> parser(String value)) {
    /// 如果，已经有缓存，直接返回
    if (_structuredDataCache.containsKey(key))
      return _structuredDataCache[key] as Future<T>;
    Completer<T> completer;
    Future<T> result;
    loadString(key, cache: false).then<T>(parser).then<void>((T value) {
      result = SynchronousFuture<T>(value);
      _structuredDataCache[key] = result;
      if (completer != null) {
        // 我们已经从loadStructuredData函数返回，这意味着我们处于异步模式。将值传递给completer
        completer.complete(value);
      }
    });
    /// 如果上边的代码是同步运行的，则会返回一个 SynchronizedFutrue
    if (result != null) {
      return result;
    }
    // 否则，采用异步进行
    completer = Completer<T>();
    _structuredDataCache[key] = completer.future;
    return completer.future;
  }

  @override
  void evict(String key) {
    _stringCache.remove(key);
    _structuredDataCache.remove(key);
  }
}

```



## PlatformAssetBundle

```dart
/// 使用平台消息来加载资源类的 [AssetBundle]
class PlatformAssetBundle extends CachingAssetBundle {
  @override
  Future<ByteData> load(String key) async {
    final Uint8List encoded = utf8.encoder.convert(Uri(path: Uri.encodeFull(key)).path);
    final ByteData asset =
        await defaultBinaryMessenger.send('flutter/assets', encoded.buffer.asByteData()); // ignore: deprecated_member_use_from_same_package
    if (asset == null)
      throw FlutterError('Unable to load asset: $key');
    return asset;
  }
}

```


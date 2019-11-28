# Flutter ScreenUtil 源码分析

```dart
class ScreenUtil {
  static ScreenUtil instance = new ScreenUtil();

  /// UI设计图中的手机尺寸, px
  double width;
  double height;

  /// 控制字体是否要根据系统的“字体大小”辅助选项来进行缩放。默认值为false。
  bool allowFontScaling;

  static MediaQueryData _mediaQueryData;
  static double _screenWidth;
  static double _screenHeight;
  static double _pixelRatio;
  static double _statusBarHeight;

  static double _bottomBarHeight;

  static double _textScaleFactor;

  ScreenUtil({
    this.width = 1080,
    this.height = 1920,
    this.allowFontScaling = false,
  });

  static ScreenUtil getInstance() {
    return instance;
  }

  /// 获取到屏幕宽高，像素比，状态栏高度，底部高度，字体大小缩放高度
  /// 事实上，可以通过 windows 对象直接获取而不需要 context
  /// MediaQueryData.fromWindow(ui.Window window)
  ///    // size = 设备物理大小除以设备像素比  
  ///  : size = window.physicalSize / window.devicePixelRatio,
  ///    devicePixelRatio = window.devicePixelRatio,
  ///    textScaleFactor = window.textScaleFactor,
  ///    // 平台亮度设定
  ///    platformBrightness = window.platformBrightness,
  ///    // 边距
  ///    padding = EdgeInsets.fromWindowPadding(window.padding, window.devicePixelRatio),
  ///    viewPadding = EdgeInsets.fromWindowPadding(
  ///       window.viewPadding,     
  ///       window.devicePixelRatio),
  ///    viewInsets = EdgeInsets.fromWindowPadding(
  ///       window.viewInsets, 
  ///       window.devicePixelRatio),
  ///    systemGestureInsets = EdgeInsets.fromWindowPadding(
  ///       window.systemGestureInsets, 
  ///       window.devicePixelRatio),
  ///    physicalDepth = window.physicalDepth,
  ///    accessibleNavigation = window.accessibilityFeatures.accessibleNavigation,
  ///    invertColors = window.accessibilityFeatures.invertColors,
  ///    disableAnimations = window.accessibilityFeatures.disableAnimations,
  ///    boldText = window.accessibilityFeatures.boldText,
  ///    highContrast = false,
  ///    // 是否总是使用24小时制
  ///    alwaysUse24HourFormat = window.alwaysUse24HourFormat;  
  void init(BuildContext context) {
    MediaQueryData mediaQuery = MediaQuery.of(context);
    _mediaQueryData = mediaQuery;
    _pixelRatio = mediaQuery.devicePixelRatio;
    _screenWidth = mediaQuery.size.width;
    _screenHeight = mediaQuery.size.height;
    _statusBarHeight = mediaQuery.padding.top;
    _bottomBarHeight = _mediaQueryData.padding.bottom;
    _textScaleFactor = mediaQuery.textScaleFactor;
  }

  static MediaQueryData get mediaQueryData => _mediaQueryData;

  /// 每个逻辑像素的字体像素数，字体的缩放比例
  static double get textScaleFactory => _textScaleFactor;

  /// 设备的像素密度
  static double get pixelRatio => _pixelRatio;

  /// 当前设备宽度 dp
  static double get screenWidthDp => _screenWidth;

  ///当前设备高度 dp
  static double get screenHeightDp => _screenHeight;

  /// 当前设备宽度 px
  /// 设备高度dp乘以设备像素比，得到屏幕物理宽度度
  static double get screenWidth => _screenWidth * _pixelRatio;

  /// 当前设备高度 px
  /// 屏幕物理高度
  static double get screenHeight => _screenHeight * _pixelRatio;

  /// 状态栏高度 dp 刘海屏会更高
  static double get statusBarHeight => _statusBarHeight;

  /// 底部安全区距离 dp
  static double get bottomBarHeight => _bottomBarHeight;

  /// 实际的dp与UI设计px的比例
  /// 设备物理宽度除以设计图宽度，得到一个宽度比  
  double get scaleWidth => _screenWidth / instance.width;

  double get scaleHeight => _screenHeight / instance.height;

  /// 根据UI设计的设备宽度适配
  /// 高度也可以根据这个来做适配可以保证不变形,比如你先要一个正方形的时候.
  /// 设计宽度乘上宽度比，得到需要的宽度（类似于百分比宽度）  
  num setWidth(num width) => width * scaleWidth;

  /// 根据UI设计的设备高度适配
  /// 当发现UI设计中的一屏显示的与当前样式效果不符合时,
  /// 或者形状有差异时,建议使用此方法实现高度适配.
  /// 高度适配主要针对想根据UI设计的一屏展示一样的效果
  num setHeight(num height) => height * scaleHeight;

  ///字体大小适配方法
  num setSp(num fontSize) => allowFontScaling
      ? setWidth(fontSize)
      : setWidth(fontSize) / _textScaleFactor;
}

```


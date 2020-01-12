# Flutter 滑动相关



# ScrollView

ScrollView 是一个 Stateless 组件。

1. 接受一个 Widget 数组，将 buildSlivers 方法延迟给子类实现，要求将 widgets 转换成 slivers

2. build 方法返回了 一个 Scrollable，并对 scrollable 提供了 buildViewport 回调。向 Viewport 中提供了由 buildSlivers 方法构建完成的 slivers

   

# Scrollable



# ScrollPosition

# ScrollActivity



# Viewport
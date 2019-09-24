## Flutter 部分常用 Package

[TOC]



### 库：

https://pub.dev/flutter，



### 状态管理：

（谷歌推荐）provider https://pub.dev/packages/provider

（咸鱼的状态管理框架）fish_redux https://pub.dev/packages/fish_redux



### 数据储存：

（获取路径）path_provider https://pub.dev/packages/path_provider

（本地键值对存储）shared_preferences https://pub.dev/packages/shared_preferences



### 图片：

（访问相册，拍照）image_picker https://pub.dev/packages/image_picker

（图片查看器）extended_image https://pub.dev/packages/extended_image

（照片查看）photo https://pub.dev/packages/photo



### 视频音乐：
（视频播放）video_player shttps://pub.dev/packages/video_player

（铃声）flutter_ringtone_player _https://pub.dev/packages/flutter_ringtone_player



### 组件：

（flare 动画）flare_flutter https://pub.dev/packages/flare_flutter

（nima 动画）nima https://pub.dev/packages/nima

（轮播图）flutter_swiper https://pub.dev/packages/flutter_swiper

（加载图标）flutter_spinkit https://pub.dev/packages/flutter_spinkit

（toast）fluttertoast  https://pub.dev/packages/fluttertoast

（markdown插件）markdown https://pub.dev/packages/markdown

（日历组件）table_calendar https://pub.dev/packages/table_calendar

（小红点）

  bottom_navigation_badge https://pub.dev/packages/bottom_navigation_badge
  badges https://pub.dev/packages/badges

（瀑布流）flutter_staggered_grid_view https://pub.dev/packages/flutter_staggered_grid_view

（右侧滚动条）darggable_scrollbar https://pub.dev/packages/draggable_scrollbar

（自动调整大小Text）auto_size_text https://pub.dev/packages/auto_size_text

（类似于snakebar的通知栏）flushbar https://pub.dev/packages/flushbar



### 原生管理：

（启动第三方app、网页）url_launcher https://pub.dev/packages/url_launcher

（权限管理）permission_handler https://pub.dev/packages/permission_handler

（传感器）sensors https://pub.dev/packages/sensors

（设备方向）orientation https://pub.dev/packages/orientation

（网络状态）connectivity https://pub.dev/packages/connectivity

（设备信息）device_info https://pub.dev/packages/device_info

（分享）share https://pub.dev/packages/share

（位置获取）location https://pub.dev/packages/location

（本地通知）flutter_local_notification https://pub.dev/packages/flutter_local_notifications



### 网络：

（网络）dio https://pub.dev/packages/dio



### 浏览器：

（widget浏览器）webview_flutter https://pub.dev/packages/webview_flutter

（原生view浏览器）flutter_webview_plugin https://pub.dev/packages/flutter_webview_plugin



### 图表：

（贝塞尔折线图）bezier_chart https://pub.dev/packages/bezier_chart



### Json：

（json序列化和反序列化）

json_annotation https://pub.dev/packages/json_annotation

json_serializable https://pub.dev/packages/json_serializable

build_runner https://pub.dev/packages/build_runner



| 方式              | 大小 (js) | 序列化 (dart) | 反序列化 (dart) | 序列化 (js) | 反序列化 (js) |
| ----------------- | --------- | ------------- | --------------- | ----------- | :------------ |
| json_serializable | 80 KB     | 9.09 ms       | 6.61 ms         | 8.23 ms     | 8.12 ms       |
| Serializable      | 79 KB     | 6.1 ms        | 6.92 ms         | 4.37 ms     |               |
| DSON              | 94 KB     | 12.72 ms      | 11.15 ms        | 16.64 ms    | 17.94 ms      |
| Dartson           | 86 KB     | 9.61 ms       | 6.81 ms         | 8.58 ms     | 7.01 ms       |
| Manual            | 86 KB     | 8.29 ms       | 5.78 ms         | 10.7 ms     | 7.9 ms        |
| Interop           | 70 KB     | 61.55 ms      | 14.96 ms        | 2.49 ms     | 2.93 ms       |
| Jaguar_serializer | 88 KB     | 8.57 ms       | 6.58 ms         | 10.31 ms    | 8.59 ms       |

​		

### 其他：

（一些常用工具）flustars https://pub.dev/packages/flustars

（做游戏的？）spritewidget https://pub.dev/packages/spritewidget

（下载器）flutter_downloader https://pub.dev/packages/flutter_downloader

（Icon库）font_awesome_flutter https://pub.dev/packages/font_awesome_flutter

（二维码）barcode_scan https://pub.dev/packages/barcode_scan

（闪亮特效）shimmer https://pub.dev/packages/shimmer

（微信相关）fluwx https://pub.dev/packages/fluwx

（qq相关）flutter_qq https://pub.dev/packages/flutter_qq

（极光推送）jpush_flutter https://pub.dev/packages/jpush_flutter

（外国推送平台？）onesignal_flutter https://pub.dev/packages/onesignal_flutter
[TOC]

# 关于 Flutter Gallery

Git坐标：https://github.com/flutter/flutter/tree/master/examples/flutter_gallery



首先放几张主界面截图：
<div>
<img src = "../Snap/Flutter_Gallery_snap1.jpg" width = "300" align = left></img><img src = "../Snap/Flutter_Gallery_snap2.jpg" width = "300" align = left></img>
<img src = "../Snap/Flutter_Gallery_snap3.jpg" width = "300" align = left></img><img src = "../Snap/Flutter_Gallery_snap4.jpg" width = "300" align = left></img>
</div>&nbsp

<del>个人很喜欢这种UI,XD</del>



# 杂七杂八



## Options

在options.dart中

- 定义了一个用于保存信息的GalleryOptions类，其中有可以用来拷贝另一个 GalleryOptions 信息的函数copyWith。
```dart
class GalleryOptions {
	...
  final GalleryTheme theme;//主题
  final GalleryTextScaleValue textScaleFactor;//字号
  final TextDirection textDirection;//文字方向
  final double timeDilation;//全局动画速度，
  final TargetPlatform platform;//？？？
  final bool showPerformanceOverlay;//？？？
  final bool showRasterCacheImagesCheckerboard;//？？
  final bool showOffscreenLayersCheckerboard;//？？？
    ...
```
- 定义了_OptionsItem类（下拉选单），和一个_BooleanItem类（开关）、_ActionItem类（活动按钮）、_FlatButton类(点击按钮)、_Heading类（选项名称，例如DIsplay,Diagnostics等）

- 定义了主页Option中的选项类：

  主题选项类

  ```dart
  class _ThemeItem extends StatelessWidget {
    const _ThemeItem(this.options, this.onOptionsChanged);
  
    final GalleryOptions options; 
      //在每个选项类中都有一个，当该选项变更时，获得一份拷贝
    final ValueChanged<GalleryOptions> onOptionsChanged;
      //函数变量，<> 里是返回值类型。这里要求传入一个 onOptionsChanged 函数
  
    @override
    Widget build(BuildContext context) {
      return _BooleanItem(
        'Dark Theme',
        options.theme == kDarkGalleryTheme,
        (bool value) {
          onOptionsChanged(
            options.copyWith(
              theme: value ? kDarkGalleryTheme : kLightGalleryTheme,
            ),//拷贝options
          );
        },
        switchKey: const Key('dark_theme'),
      );
    }
  }
  ```
字体大小选项类
  ```dart
  class _TextScaleFactorItem extends StatelessWidget {
    const _TextScaleFactorItem(this.options, this.onOptionsChanged);
  
    final GalleryOptions options;
    final ValueChanged<GalleryOptions> onOptionsChanged;
      //同主题选项类
  
    @override
    Widget build(BuildContext context) {
      return _OptionsItem(
        child: Row(//行布局 				    
          children: <Widget>[
            Expanded(//Expanded，在行占用尽可能大的空间		
              child: Column(//行布局里的列布局
                crossAxisAlignment: CrossAxisAlignment.start,//横轴居中
                children: <Widget>[
                  const Text('Text size'),//常量文本框
                  Text(//可变文本框，用来显示当前字号
                    '${options.textScaleFactor.label}',//引用当前options的字体大小
                    style: Theme.of(context).primaryTextTheme.body1,
                  ),
                ],
              ),
            ),
           
            PopupMenuButton<GalleryTextScaleValue>(//弹出菜单按钮
              padding: const EdgeInsetsDirectional.only(end: 16.0),//右侧边距
              icon: const Icon(Icons.arrow_drop_down),//下箭头图标
              itemBuilder: (BuildContext context) {
                return kAllGalleryTextScaleValues
                    .map<PopupMenuItem<GalleryTextScaleValue>>(
                    (GalleryTextScaleValue scaleValue) {
                  		return PopupMenuItem<GalleryTextScaleValue>(
                            value: scaleValue,
                            child: Text(scaleValue.label),
                     );}
                ).toList();//返回一个列表
              },
              onSelected: (GalleryTextScaleValue scaleValue) {
                onOptionsChanged(
                  options.copyWith(textScaleFactor: scaleValue),
                );
              },
            ),
          ],
        ),
    );
    }
}
  ```
  
  ...
  
- 定义了GalleryOptionsPage 类，用于包含所有的options

  ```dart
  class GalleryOptionsPage extends StatelessWidget {
      ...
    final GalleryOptions options;
    final ValueChanged<GalleryOptions> onOptionsChanged;
    final VoidCallback onSendFeedback;
      ...
    @override
    Widget build(BuildContext context) {
      final ThemeData theme = Theme.of(context);
  
      return DefaultTextStyle(
        style: theme.primaryTextTheme.subhead,
        child: ListView(
          padding: const EdgeInsets.only(bottom: 124.0),
          children: <Widget>[
            const _Heading('Display'),//选项类型标题
            _ThemeItem(options, onOptionsChanged),
            _TextScaleFactorItem(options, onOptionsChanged),
            _TextDirectionItem(options, onOptionsChanged),
            _TimeDilationItem(options, onOptionsChanged),//插入所有的选项
            const Divider(),//分割线
            const _Heading('Platform mechanics'),//选项类型标题
            _PlatformItem(options, onOptionsChanged),
          ]..addAll(//加入一个列表，包含诊断调试信息等选项
            _enabledDiagnosticItems(),
          )..addAll(//加入一个列表
            <Widget>[
              const Divider(),
              const _Heading('Flutter gallery'),//选项类型标题
              _ActionItem('About Flutter Gallery', () {
                showGalleryAboutDialog(context);//展示About信息
              }),
              _ActionItem('Send feedback', onSendFeedback),//反馈
            ],
          ),
        ),
      );
    }
  }
  
  ```

  

  ------
  
  



## 字号（其实是字体大小比例）

这是scales.dart的全部内容

```dart
// Copyright 2018 The Chromium Authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

import 'package:flutter/material.dart';

class GalleryTextScaleValue {
  const GalleryTextScaleValue(this.scale, this.label);

  final double scale;//字号比例
  final String label;//字号标签

  @override
  bool operator ==(dynamic other) {
    if (runtimeType != other.runtimeType)
      return false;
    final GalleryTextScaleValue typedOther = other;
    return scale == typedOther.scale && label == typedOther.label;
  }
	/*值得注意的是重载==运算符的方法，类似于java,先检查运行时类型是否一样，如果相同，再比较内容*/
  @override
  int get hashCode => hashValues(scale, label);

  @override
  String toString() {
    return '$runtimeType($label)';
  }

}
//字号枚举列表
const List<GalleryTextScaleValue> kAllGalleryTextScaleValues = <GalleryTextScaleValue>[
  GalleryTextScaleValue(null, 'System Default'),
  GalleryTextScaleValue(0.8, 'Small'),
  GalleryTextScaleValue(1.0, 'Normal'),
  GalleryTextScaleValue(1.3, 'Large'),
  GalleryTextScaleValue(2.0, 'Huge'),
];

```



------




## 导航

这是在app.dart里声明的一个函数，用于返回String到WidgetBuilder的映射。

其中formIterable是将List转换成Map的函数

```dart
Map<String, WidgetBuilder> _buildRoutes() {
    // For a different example of how to set up an application routing table
    // using named routes, consider the example in the Navigator class documentation:
    // https://docs.flutter.io/flutter/widgets/Navigator-class.html
    return Map<String, WidgetBuilder>.fromIterable(
      kAllGalleryDemos,
      key: (dynamic demo) => '${demo.routeName}',
      value: (dynamic demo) => demo.buildRoute,
    );
}
```



而在demos.dart中

- 定义GalleryDemoCategory类，它在主界面显示的包含Demos（Animation,Shrine等)的目录（Studies,Styles等)。
- 定义了GalleryDemo类，包含了每个Demo的所有信息
```dart
class GalleryDemo {
    ...
  final String title;
  final IconData icon;
  final String subtitle;
  final GalleryDemoCategory category;
  final String routeName;
  final WidgetBuilder buildRoute;
  final String documentationUrl;
  
  @override
  String toString() {
    return '$runtimeType($title $routeName)';
  }
}
```
- 定义了用于返回一个 包含<font color = red>所有</font>demos的列表 的函数 _buildGalleryDemos

```dart
List<GalleryDemo> _buildGalleryDemos() {
  final List<GalleryDemo> galleryDemos = <GalleryDemo>[
    // Demos
    GalleryDemo(
      title: 'Shrine',
      subtitle: 'Basic shopping app',
      icon: GalleryIcons.shrine,
      category: _kDemos,
      routeName: ShrineDemo.routeName,
      buildRoute: (BuildContext context) => const ShrineDemo(),
    ),
    ...
    ...
    GalleryDemo(
      title: 'Video',
      subtitle: 'Video playback',
      icon: GalleryIcons.drive_video,
      category: _kMedia,
      routeName: VideoDemo.routeName,
      buildRoute: (BuildContext context) => const VideoDemo(),
    ),
  ];
	...
  return galleryDemos;
}


```



## 由整体框架到主页面

在home.dart，有以下定义

```dart
class GalleryHome extends StatefulWidget {
  	...
  final Widget optionsPage;
  final bool testMode;

  // In checked mode our MaterialApp will show the default "debug" banner.
  // Otherwise show the "preview" banner.
  static bool showPreviewBanner = true;

  @override
  _GalleryHomeState createState() => _GalleryHomeState();
}
```

而在app.dart中,有以下代码

```dart
Widget home = GalleryHome(
      testMode: widget.testMode,
    //一个bool值，是否开启测试模式，由_GalleryAppStateoptions里的options持有
      optionsPage: GalleryOptionsPage(
        options: _options,
        onOptionsChanged: _handleOptionsChanged,
        onSendFeedback: widget.onSendFeedback ?? () {
launch('https://github.com/flutter/flutter/issues/new/choose', forceSafariVC: false);
        },
      ),
    );
```

<font color = red>这表明option页面是可以更换的</font>





------


# Demos

##主页面

<img src = "../Snap/Flutter_Gallery_snap1.jpg" width = "300" align = left></img>

主界面由两个部分组成：上方的的AppBar部分和下方的CategoriesPage部分：

- AppBar部分

  jkdajsa

  kadjsdka

- CategoriesPage部分

  在home.dart里有

  



















Flutter BottomAppBar 源码分析

[TOC]

## BottomAppBar

```dart
class BottomAppBar extends StatefulWidget {
  const BottomAppBar({
    Key key,
    this.color,
    this.elevation,
    this.shape,
    this.clipBehavior = Clip.none,
    this.notchMargin = 4.0,
    this.child,
  }) : assert(elevation == null || elevation >= 0.0),
       assert(notchMargin != null),
       assert(clipBehavior != null),
       super(key: key);
 
  final Widget child;
  final Color color;

  /// 浮动高度
  final double elevation;

  /// 刻痕形状
  final NotchedShape shape;
  /// 默认Clip.none
  final Clip clipBehavior;

  /// 刻痕边距
  final double notchMargin;

  @override
  State createState() => _BottomAppBarState();
}
```



## _BottomAppbarState

```dart
class _BottomAppBarState extends State<BottomAppBar> {
  ValueListenable<ScaffoldGeometry> geometryListenable;
  static const double _defaultElevation = 8.0;

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    geometryListenable = Scaffold.geometryOf(context);
  }

  @override
  Widget build(BuildContext context) {
    final BottomAppBarTheme babTheme = BottomAppBarTheme.of(context);
    final NotchedShape notchedShape = widget.shape ?? babTheme.shape;
    final CustomClipper<Path> clipper = notchedShape != null
      ? _BottomAppBarClipper(
        geometry: geometryListenable,
        shape: notchedShape,
        notchMargin: widget.notchMargin,
      )
      : const ShapeBorderClipper(shape: RoundedRectangleBorder());
      
    /// 这里采用的PhysicalShape来完成裁剪效果  
    return PhysicalShape(
      clipper: clipper,
      elevation: widget.elevation ?? babTheme.elevation ?? _defaultElevation,
      color: widget.color ?? babTheme.color ?? Theme.of(context).bottomAppBarColor,
      clipBehavior: widget.clipBehavior,
      child: Material(
        type: MaterialType.transparency,
        child: widget.child == null
          ? null
          : SafeArea(child: widget.child),
      ),
    );
  }
}
```



## _BottomAppBarClipper

```dart
class _BottomAppBarClipper extends CustomClipper<Path> {
  const _BottomAppBarClipper({
    @required this.geometry,
    @required this.shape,
    @required this.notchMargin,
  });

  final ValueListenable<ScaffoldGeometry> geometry;
  final NotchedShape shape;
  final double notchMargin;

  @override
  Path getClip(Size size) {
    final Rect button = geometry.value.floatingActionButtonArea?.translate(
      0.0,
      geometry.value.bottomNavigationBarTop * -1.0,
    );
    return shape.getOuterPath(Offset.zero & size, button?.inflate(notchMargin));
  }

  @override
  bool shouldReclip(_BottomAppBarClipper oldClipper) {
    return oldClipper.geometry != geometry
        || oldClipper.shape != shape
        || oldClipper.notchMargin != notchMargin;
  }
}
```



## NotchedShape

```dart
/// 典型情况是一个host widget创造一个刻痕来容纳guest widget
/// 例如[BottomAppBar] 创建刻痕来容纳 [FloatingActionButton]

abstract class NotchedShape {
  const NotchedShape();

  /// 获取外边界
  Path getOuterPath(Rect host, Rect guest);
}
```



## CircularNotchedRectangle

```dart
/// 与圆形FAB适应的刻痕
class CircularNotchedRectangle extends NotchedShape {
  const CircularNotchedRectangle();

  @override
  Path getOuterPath(Rect host, Rect guest) {
    if (guest == null || !host.overlaps(guest))
      return Path()..addRect(host);

    // The guest's shape is a circle bounded by the guest rectangle.
    // So the guest's radius is half the guest width.
    final double notchRadius = guest.width / 2.0;
      
    // 将整个path分为三个部分
    // Segment A - 向上弯曲的贝塞尔曲线
    // Segment B - 向下弯曲的圆弧
    // Segment C - 向上弯曲的贝塞尔曲线（与A对称）

    const double s1 = 15.0;
    const double s2 = 1.0;
    /// 半径  
    final double r = notchRadius;
    final double a = -1.0 * r - s2;
    final double b = host.top - guest.center.dy;

    final double n2 = math.sqrt(b * b * r * r * (a * a + b * b - r * r));
    final double p2xA = ((a * r * r) - n2) / (a * a + b * b);
    final double p2xB = ((a * r * r) + n2) / (a * a + b * b);
    final double p2yA = math.sqrt(r * r - p2xA * p2xA);
    final double p2yB = math.sqrt(r * r - p2xB * p2xB);

    final List<Offset> p = List<Offset>(6);

    // p0, p1, 和 p2 是 A 片段 的控制点
    p[0] = Offset(a - s1, b);
    p[1] = Offset(a, b);
    final double cmp = b < 0 ? -1.0 : 1.0;
    p[2] = cmp * p2yA > cmp * p2yB ? Offset(p2xA, p2yA) : Offset(p2xB, p2yB);

    // p0, p1, 和 p2 是 C 片段 的控制点，与 A 关于 y 轴对称
    p[3] = Offset(-1.0 * p[2].dx, p[2].dy);
    p[4] = Offset(-1.0 * p[1].dx, p[1].dy);
    p[5] = Offset(-1.0 * p[0].dx, p[0].dy);

    // 移动到绝对坐标系
    for (int i = 0; i < p.length; i += 1)
      p[i] += guest.center;

    return Path()
      ..moveTo(host.left, host.top)
      ..lineTo(p[0].dx, p[0].dy)
      ..quadraticBezierTo(p[1].dx, p[1].dy, p[2].dx, p[2].dy)
      ..arcToPoint(
          p[3],
          radius: Radius.circular(notchRadius),
          clockwise: false,
        )
      ..quadraticBezierTo(p[4].dx, p[4].dy, p[5].dx, p[5].dy)
      ..lineTo(host.right, host.top)
      ..lineTo(host.right, host.bottom)
      ..lineTo(host.left, host.bottom)
      ..close();
  }
}
```



## AutomaticNotchedShape

```dart
/// 通过 ShapeBorders 来创建刻痕
class AutomaticNotchedShape extends NotchedShape {

  const AutomaticNotchedShape(this.host, [ this.guest ]);

  final ShapeBorder host;
  final ShapeBorder guest;

  @override
  Path getOuterPath(Rect hostRect, Rect guestRect) {
    final Path hostPath = host.getOuterPath(hostRect);
    if (guest != null && guestRect != null) {
      final Path guestPath = guest.getOuterPath(guestRect);
      return Path.combine(PathOperation.difference, hostPath, guestPath);
    }
    return hostPath;
  }
}
```



### 附：_DiamondNotchedRectangle

```dart
class _DiamondNotchedRectangle implements NotchedShape {
  const _DiamondNotchedRectangle();

  @override
  Path getOuterPath(Rect host, Rect guest) {
    if (!host.overlaps(guest))
      return Path()..addRect(host);

    final Rect intersection = guest.intersect(host);
    // We are computing a "V" shaped notch, as in this diagram:
    //    -----\****   /-----
    //          \     /
    //           \   /
    //            \ /
    //
    //  "-" marks the top edge of the bottom app bar.
    //  "\" and "/" marks the notch outline
    //
    //  notchToCenter is the horizontal distance between the guest's center and
    //  the host's top edge where the notch starts (marked with "*").
    //  We compute notchToCenter by similar triangles:
    final double notchToCenter =
      intersection.height * (guest.height / 2.0)
      / (guest.width / 2.0);

    return Path()
      ..moveTo(host.left, host.top)
      ..lineTo(guest.center.dx - notchToCenter, host.top)
      ..lineTo(guest.left + guest.width / 2.0, guest.bottom)
      ..lineTo(guest.center.dx + notchToCenter, host.top)
      ..lineTo(host.right, host.top)
      ..lineTo(host.right, host.bottom)
      ..lineTo(host.left, host.bottom)
      ..close();
  }
}
```


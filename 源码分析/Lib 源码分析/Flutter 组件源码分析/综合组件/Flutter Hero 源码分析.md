# Flutter Hero 源码分析

[TOC]

## 函数原型

```dart
/// 由给定的两个 rect 返回一个插值 rect
/// 这个函数通常被 HeroController 使用来创建页面动画
/// 通常来说采用非线性的值会比较好看
typedef CreateRectTween = Tween<Rect> Function(Rect begin, Rect end);

/// 用给定的大小和 widget 构建一个占位符
typedef HeroPlaceholderBuilder = Widget Function(
  BuildContext context,
  Size heroSize,
  Widget child,
);

/// 构建承载子child的‘穿梭机’
typedef HeroFlightShuttleBuilder = Widget Function(
  BuildContext flightContext,
  Animation<double> animation,
  HeroFlightDirection flightDirection,
  BuildContext fromHeroContext,
  BuildContext toHeroContext,
);

/// 完成回调
typedef _OnFlightEnded = void Function(_HeroFlight flight);

/// 飞行方向
enum HeroFlightDirection {
  /// 由 push 触发的hero
  push,

  /// 由 pop 触发的hero
  pop,
}

// 在祖先 context 坐标系统中给定 context 的边界框。
Rect _boundingBoxFor(BuildContext context, [BuildContext ancestorContext]) {
  final RenderBox box = context.findRenderObject();
  /// MatrixUtils.transformRect 将给定的 Matrix4 变换运用到给定的 Rect 上，返回一个新 Rect 
  return MatrixUtils.transformRect(
      box.getTransformTo(ancestorContext?.findRenderObject()),
      Offset.zero & box.size,
  );
}
```

## 示意图

<img src = "https://flutter.github.io/assets-for-api-docs/assets/interaction/heroes.png" width = 600px align = "left"/>

## Hero

```dart
class Hero extends StatefulWidget {
  const Hero({
    Key key,
    @required this.tag,
    this.createRectTween,
    this.flightShuttleBuilder,
    this.placeholderBuilder,
    this.transitionOnUserGestures = false,
    @required this.child,
  });

  /// 唯一标识符
  /// 用于联系两个页面的 hero  
  final Object tag;

  /// 定义路由的边界如何变化
  ///
  /// 一次 hero 飞行，从目的地子树与起始 hero 的子树对其的位置开始。这个回调返回的 [Tween<Rect>] 被用作计算 
  /// 当动画值从 0.0 到 1.0 过程中 hero 的边界
  ///
  /// 默认使用[MaterialApp]创建的[HeroController.createRectTween]
  final CreateRectTween createRectTween;

  /// 一个路由中的 widget 子树将会飞到另一个路由的 widget 子树位置
  /// 两个 widget 的外观应该尽可能地相似（如大小、颜色变化）  
  final Widget child;
  
  /// ## 限制
  ///
  /// 如果 [flightShuttleBuilder] 产生的 widget 参加了 [Navigator] 的 push过程
  /// 那么它的子树不能使用 globalKey。因为在飞行时，在路由页面的子树中都会包含 hero 及其子树，但
  /// [GlobalKey] 必须是全局单一的
  ///
  /// 如果上述 [GlobalKey] 是必须的，请考虑为源 hero 提供一个自定义[placeholderBuilder]，
  /// 以避免 [GlobalKey] 冲突。例如：构建了一个空的[SizedBox]，保持 hero's [child] 的原始大小。
  final HeroFlightShuttleBuilder flightShuttleBuilder;

  /// hero [child] 起飞之后的占位符
  ///
  /// 默认是一个保持 hero 原始大小的 [SizedBox]，除非是导航过渡动画
  /// 导航过渡时，child 是占位符的子树，保持 offstage  
  final HeroPlaceholderBuilder placeholderBuilder;

  /// 当 [PageRoute] 被手势触发的时候，时候需要运用 hero 动画。例如，IOS 系统的返回手势
  final bool transitionOnUserGestures;

  /// 当从一个页面过渡到另一个页面时，返回一个从 Herotag 到 Hero 的映射表
  /// 即把给定的 context 的子树的所有满足条件的 Hero 的 tag 映射到其 context，
  /// 并返回这个映射表
  static Map<Object, _HeroState> _allHeroesFor(
      BuildContext context,
      bool isUserGestureTransition,
      NavigatorState navigator,
  ) {
    final Map<Object, _HeroState> result = <Object, _HeroState>{};

    void inviteHero(StatefulElement hero, Object tag) {
      final Hero heroWidget = hero.widget;
      final _HeroState heroState = hero.state;
      if (!isUserGestureTransition || heroWidget.transitionOnUserGestures) {
        result[tag] = heroState;
      } else {
        // 如果不允许使用手势过渡路由动画，那么必须将 hero 隐藏（因为hero可能在先前的过渡中被隐藏了）
        heroState.ensurePlaceholderIsHidden();
      }
    }

    void visitor(Element element) {
      if (element.widget is Hero) {
        final StatefulElement hero = element;
        final Hero heroWidget = element.widget;
        final Object tag = heroWidget.tag;
        if (Navigator.of(hero) == navigator) {
          inviteHero(hero, tag);
        } else {
          final ModalRoute<dynamic> heroRoute = ModalRoute.of(hero);
          if (heroRoute != null && heroRoute is PageRoute && heroRoute.isCurrent) {
            inviteHero(hero, tag);
          }
        }
      }
      element.visitChildren(visitor);
    }

    context.visitChildElements(visitor);
    return result;
  }

  @override
  _HeroState createState() => _HeroState();

}

class _HeroState extends State<Hero> {
  final GlobalKey _key = GlobalKey();
  Size _placeholderSize;
  // 占位符是否应该将 hero 的 child 作为自己的 child，当 _placeholderSize为 非 null时 
  // 例如，hero正处于飞行动画中
  bool _shouldIncludeChild = true;

  // The `shouldIncludeChildInPlaceholder` flag dictates if the child widget of
  // this hero should be included in the placeholder widget as a descendant.
  //
  // When a new hero flight animation takes place, a placeholder widget
  // needs to be built to replace the original hero widget. When
  // `shouldIncludeChildInPlaceholder` is set to true and `widget.placeholderBuilder`
  // is null, the placeholder widget will include the original hero's child
  // widget as a descendant, allowing the orignal element tree to be preserved.
  //
  // It is typically set to true for the *from* hero in a push transition,
  // and false otherwise.
    
  void startFlight({ bool shouldIncludedChildInPlaceholder = false }) {
    _shouldIncludeChild = shouldIncludedChildInPlaceholder;
    final RenderBox box = context.findRenderObject();
    assert(box != null && box.hasSize);
    setState(() {
      _placeholderSize = box.size;
    });
  }

  void ensurePlaceholderIsHidden() {
    if (mounted) {
      setState(() {
        _placeholderSize = null;
      });
    }
  }

  // When `keepPlaceholder` is true, the placeholder will continue to be shown
  // after the flight ends. Otherwise the child of the Hero will become visible
  // and its TickerMode will be re-enabled.
  void endFlight({ bool keepPlaceholder = false }) {
    if (!keepPlaceholder) {
      ensurePlaceholderIsHidden();
    }
  }

  @override
  Widget build(BuildContext context) {

    final bool showPlaceholder = _placeholderSize != null;

    /// 需要展示占位符  
    if (showPlaceholder && widget.placeholderBuilder != null) {
      return widget.placeholderBuilder(context, _placeholderSize, widget.child);
    }

    /// 需要展示占位符并且不包含原hero的子树  
    if (showPlaceholder && !_shouldIncludeChild) {
      return SizedBox(
        width: _placeholderSize.width,
        height: _placeholderSize.height,
      );
    }

    return SizedBox(
      width: _placeholderSize?.width,
      height: _placeholderSize?.height,
      child: Offstage(
        /// 需要展示占位符时，子树不显示  
        offstage: showPlaceholder,
        child: TickerMode(
          /// 需要展示占位符时，子树不tick  
          enabled: !showPlaceholder,
          child: KeyedSubtree(key: _key, child: widget.child),
        ),
      ),
    );
  }
}
```



## HeroFlightManifest

```dart
/// 展示类（用于存取数据，构建动画）
class _HeroFlightManifest {
  _HeroFlightManifest({
    @required this.type,
    @required this.overlay,
    @required this.navigatorRect,
    @required this.fromRoute,
    @required this.toRoute,
    @required this.fromHero,
    @required this.toHero,
    @required this.createRectTween,
    @required this.shuttleBuilder,
    @required this.isUserGestureTransition,
    @required this.isDiverted,
  }) : assert(fromHero.widget.tag == toHero.widget.tag);

  final HeroFlightDirection type;
  final OverlayState overlay;
  final Rect navigatorRect;
  final PageRoute<dynamic> fromRoute;
  final PageRoute<dynamic> toRoute;
  final _HeroState fromHero;
  final _HeroState toHero;
  final CreateRectTween createRectTween;
  final HeroFlightShuttleBuilder shuttleBuilder;
  final bool isUserGestureTransition;
  final bool isDiverted;

  Object get tag => fromHero.widget.tag;

  /// 获取动画  
  Animation<double> get animation {
    return CurvedAnimation(
      parent: (type == HeroFlightDirection.push) ? 
        toRoute.animation : 
        fromRoute.animation,
      curve: Curves.fastOutSlowIn,
      reverseCurve: isDiverted ? null : Curves.fastOutSlowIn.flipped,
    );
  }

}
```



## HeroFlight

```dart
// 构建飞行组件
class _HeroFlight {
  _HeroFlight(this.onFlightEnded) {
    _proxyAnimation = ProxyAnimation()..addStatusListener(_handleAnimationUpdate);
  }

  final _OnFlightEnded onFlightEnded;

  Tween<Rect> heroRectTween;
  Widget shuttle;

  Animation<double> _heroOpacity = kAlwaysCompleteAnimation;
  ProxyAnimation _proxyAnimation;
  _HeroFlightManifest manifest;
  OverlayEntry overlayEntry;
  bool _aborted = false;

  Tween<Rect> _doCreateRectTween(Rect begin, Rect end) {
    final CreateRectTween createRectTween = 
        manifest.toHero.widget.createRectTween ?? manifest.createRectTween;
    if (createRectTween != null)
      return createRectTween(begin, end);
    return RectTween(begin: begin, end: end);
  }

  static final Animatable<double> _reverseTween = Tween<double>(begin: 1.0, end: 0.0);

  // 覆盖层的widgetbuilder
  Widget _buildOverlay(BuildContext context) {
    shuttle ??= manifest.shuttleBuilder(
      context,
      manifest.animation,
      manifest.type,
      manifest.fromHero.context,
      manifest.toHero.context,
    );

    return AnimatedBuilder(
      animation: _proxyAnimation,
      child: shuttle,
      builder: (BuildContext context, Widget child) {
        final RenderBox toHeroBox = manifest.toHero.context?.findRenderObject();
        if (_aborted || toHeroBox == null || !toHeroBox.attached) {
          // 当ToHero不存在或者被移除的时候，继续飞行，并且淡出
          if (_heroOpacity.isCompleted) {
            _heroOpacity = _proxyAnimation.drive(
              _reverseTween.chain(
                  CurveTween(curve: Interval(_proxyAnimation.value, 1.0))),
            );
          }
          // 飞行途中  
        } else if (toHeroBox.hasSize) {
          // The toHero has been laid out. If it's no longer where the hero animation is
          // supposed to end up then recreate the heroRect tween.
          final RenderBox finalRouteBox = 
              manifest.toRoute.subtreeContext?.findRenderObject();
          final Offset toHeroOrigin = toHeroBox.localToGlobal(
              Offset.zero, ancestor: finalRouteBox
          );
          if (toHeroOrigin != heroRectTween.end.topLeft) {
            final Rect heroRectEnd = toHeroOrigin & heroRectTween.end.size;
            heroRectTween = _doCreateRectTween(heroRectTween.begin, heroRectEnd);
          }
        }

        final Rect rect = heroRectTween.evaluate(_proxyAnimation);
        final Size size = manifest.navigatorRect.size;
        final RelativeRect offsets = RelativeRect.fromSize(rect, size);

        return Positioned(
          top: offsets.top,
          right: offsets.right,
          bottom: offsets.bottom,
          left: offsets.left,
          child: IgnorePointer(
            child: RepaintBoundary(
              child: Opacity(
                opacity: _heroOpacity.value,
                child: child,
              ),
            ),
          ),
        );
      },
    );
  }

  void _handleAnimationUpdate(AnimationStatus status) {
    if (status == AnimationStatus.completed 
        || status == AnimationStatus.dismissed)
    {
      _proxyAnimation.parent = null;

      assert(overlayEntry != null);
      overlayEntry.remove();
      overlayEntry = null;
      // We want to keep the hero underneath the current page hidden. If
      // [AnimationStatus.completed], toHero will be the one on top and we keep
      // fromHero hidden. If [AnimationStatus.dismissed], the animation is
      // triggered but canceled before it finishes. In this case, we keep toHero
      // hidden instead.
      manifest.fromHero.endFlight(keepPlaceholder: status == AnimationStatus.completed);
      manifest.toHero.endFlight(keepPlaceholder: status == AnimationStatus.dismissed);
      onFlightEnded(this);
    }
  }

  // The simple case: we're either starting a push or a pop animation.
  void start(_HeroFlightManifest initialManifest) {
    assert(!_aborted);
    assert(() {
      final Animation<double> initial = initialManifest.animation;
      assert(initial != null);
      final HeroFlightDirection type = initialManifest.type;
      assert(type != null);
      switch (type) {
        case HeroFlightDirection.pop:
          return initial.value == 1.0 && initialManifest.isUserGestureTransition
              // During user gesture transitions, the animation controller isn't
              // driving the reverse transition, but should still be in a previously
              // completed stage with the initial value at 1.0.
              ? initial.status == AnimationStatus.completed
              : initial.status == AnimationStatus.reverse;
        case HeroFlightDirection.push:
          return initial.value == 0.0 && initial.status == AnimationStatus.forward;
      }
      return null;
    }());

    manifest = initialManifest;

    if (manifest.type == HeroFlightDirection.pop)
      _proxyAnimation.parent = ReverseAnimation(manifest.animation);
    else
      _proxyAnimation.parent = manifest.animation;

    manifest.fromHero.startFlight(shouldIncludedChildInPlaceholder: manifest.type == HeroFlightDirection.push);
    manifest.toHero.startFlight();

    heroRectTween = _doCreateRectTween(
      _boundingBoxFor(manifest.fromHero.context, manifest.fromRoute.subtreeContext),
      _boundingBoxFor(manifest.toHero.context, manifest.toRoute.subtreeContext),
    );

    overlayEntry = OverlayEntry(builder: _buildOverlay);
    manifest.overlay.insert(overlayEntry);
  }

  // While this flight's hero was in transition a push or a pop occurred for
  // routes with the same hero. Redirect the in-flight hero to the new toRoute.
  void divert(_HeroFlightManifest newManifest) {
    assert(manifest.tag == newManifest.tag);
    if (manifest.type == HeroFlightDirection.push && newManifest.type == HeroFlightDirection.pop) {
      // A push flight was interrupted by a pop.
      assert(newManifest.animation.status == AnimationStatus.reverse);
      assert(manifest.fromHero == newManifest.toHero);
      assert(manifest.toHero == newManifest.fromHero);
      assert(manifest.fromRoute == newManifest.toRoute);
      assert(manifest.toRoute == newManifest.fromRoute);

      // The same heroRect tween is used in reverse, rather than creating
      // a new heroRect with _doCreateRectTween(heroRect.end, heroRect.begin).
      // That's because tweens like MaterialRectArcTween may create a different
      // path for swapped begin and end parameters. We want the pop flight
      // path to be the same (in reverse) as the push flight path.
      _proxyAnimation.parent = ReverseAnimation(newManifest.animation);
      heroRectTween = ReverseTween<Rect>(heroRectTween);
    } else if (manifest.type == HeroFlightDirection.pop && newManifest.type == HeroFlightDirection.push) {
      // A pop flight was interrupted by a push.
      assert(newManifest.animation.status == AnimationStatus.forward);
      assert(manifest.toHero == newManifest.fromHero);
      assert(manifest.toRoute == newManifest.fromRoute);

      _proxyAnimation.parent = newManifest.animation.drive(
        Tween<double>(
          begin: manifest.animation.value,
          end: 1.0,
        ),
      );
      if (manifest.fromHero != newManifest.toHero) {
        manifest.fromHero.endFlight(keepPlaceholder: true);
        newManifest.toHero.startFlight();
        heroRectTween = _doCreateRectTween(
            heroRectTween.end,
            _boundingBoxFor(newManifest.toHero.context, newManifest.toRoute.subtreeContext),
        );
      } else {
        // TODO(hansmuller): Use ReverseTween here per github.com/flutter/flutter/pull/12203.
        heroRectTween = _doCreateRectTween(heroRectTween.end, heroRectTween.begin);
      }
    } else {
      // A push or a pop flight is heading to a new route, i.e.
      // manifest.type == _HeroFlightType.push && newManifest.type == _HeroFlightType.push ||
      // manifest.type == _HeroFlightType.pop && newManifest.type == _HeroFlightType.pop
      assert(manifest.fromHero != newManifest.fromHero);
      assert(manifest.toHero != newManifest.toHero);

      heroRectTween = _doCreateRectTween(
          heroRectTween.evaluate(_proxyAnimation),
          _boundingBoxFor(newManifest.toHero.context, newManifest.toRoute.subtreeContext),
      );
      shuttle = null;

      if (newManifest.type == HeroFlightDirection.pop)
        _proxyAnimation.parent = ReverseAnimation(newManifest.animation);
      else
        _proxyAnimation.parent = newManifest.animation;

      manifest.fromHero.endFlight(keepPlaceholder: true);
      manifest.toHero.endFlight(keepPlaceholder: true);

      // Let the heroes in each of the routes rebuild with their placeholders.
      newManifest.fromHero.startFlight(shouldIncludedChildInPlaceholder: newManifest.type == HeroFlightDirection.push);
      newManifest.toHero.startFlight();

      // Let the transition overlay on top of the routes also rebuild since
      // we cleared the old shuttle.
      overlayEntry.markNeedsBuild();
    }

    _aborted = false;
    manifest = newManifest;
  }

  void abort() {
    _aborted = true;
  }

  @override
  String toString() {
    final RouteSettings from = manifest.fromRoute.settings;
    final RouteSettings to = manifest.toRoute.settings;
    final Object tag = manifest.tag;
    return 'HeroFlight(for: $tag, from: $from, to: $to ${_proxyAnimation.parent})';
  }
}
```



## HeroController

```dart
/// 一个 [Navigator] 观测者，用于管理 [Hero] 效果
/// 它继承自 NavigatorObserver，故它可以获取到当路页面的切换
class HeroController extends NavigatorObserver {

  HeroController({ this.createRectTween });
  final CreateRectTween createRectTween;
  /// 一个从对象到 _HeroFlight 的映射
  /// 其中：tag 是 Hero 组件中的 tag，两个 Hero 的 tag 相同，它们才会有连续的动画效果
  ///  _HeroFlight 是用来操控 Hero 动画的实体类
  final Map<Object, _HeroFlight> _flights = <Object, _HeroFlight>{};

  /// 当发生 Push 或 Pop 或 Replace 时，如果可以的话，启动路由动画
  @override
  void didPush(Route<dynamic> route, Route<dynamic> previousRoute) {
    _maybeStartHeroTransition(previousRoute, route, HeroFlightDirection.push, false);
  }

  @override
  void didPop(Route<dynamic> route, Route<dynamic> previousRoute) {
    if (!navigator.userGestureInProgress)
      _maybeStartHeroTransition(route, previousRoute, HeroFlightDirection.pop, false);
  }

  @override
  void didReplace({ Route<dynamic> newRoute, Route<dynamic> oldRoute }) {
    if (newRoute?.isCurrent == true) {
      // 只在最上层路由被替换时，播放动画
      _maybeStartHeroTransition(oldRoute, newRoute, HeroFlightDirection.push, false);
    }
  }

  @override
  void didStartUserGesture(Route<dynamic> route, Route<dynamic> previousRoute) {
    _maybeStartHeroTransition(route, previousRoute, HeroFlightDirection.pop, true);
  }


  /// 根据实际情况来触发动画，
  /// 其中：
  /// fromeRoute 是起始的路由，即 Hero 的起飞界面
  /// toRoute 是终点路由，即 Hero 的降落界面     
  void _maybeStartHeroTransition(
    Route<dynamic> fromRoute,
    Route<dynamic> toRoute,
    HeroFlightDirection flightType,
    bool isUserGestureTransition,
  ) {
    if (toRoute != fromRoute 
        && toRoute is PageRoute<dynamic> 
        && fromRoute is PageRoute<dynamic>) 
    {
      final PageRoute<dynamic> from = fromRoute;
      final PageRoute<dynamic> to = toRoute;
      /// 显然，当路由类型是 push 的时候，采用 目标路由的动画作为动画
      /// 否则采用 返回路由的动画作为动画，则确保了动画的值随时间为 0.0 到 1.0
      final Animation<double> animation = 
          (flightType == HeroFlightDirection.push) ? to.animation : from.animation;

	  /// 若路由已完成，跳过 
      switch (flightType) {
        case HeroFlightDirection.pop:
          if (animation.value == 0.0) {
            return;
          }
          break;
        case HeroFlightDirection.push:
          if (animation.value == 1.0) {
            return;
          }
          break;
      }

      /// 如果 使用手势跳转 并且 是退出路由 并且 目标页面使用了维持状态的话，可以立即开始 hero 过渡
      /// 注意：
      /// 因为退出路由需要将当前页面移除，从而展现下层页面。如果下层页面没有使用 维持状态（keepState）的话
      /// 那么下层路由需要重新构建后布局并绘制
      /// 因此，对于使用了 维持状态的下层界面，可以直接开始 hero 过渡
      if (isUserGestureTransition 
          && flightType == HeroFlightDirection.pop 
          && to.maintainState) 
      {
        _startHeroTransition(from, to, animation, flightType, isUserGestureTransition);
      } else {
        // 否则，在下一帧的时候才开始 hero 过渡，即在本次构建布局绘制完成之后

        // 当一个路由离开屏幕的时候它的动画的值会变成 1.0 当这一帧完成的时候，我们自动目标路由完成了,即目标路
        // 由出现在了屏幕上
        to.offstage = to.animation.value == 0.0;
		/// 在下一帧的时候构建
        WidgetsBinding.instance.addPostFrameCallback((Duration value) {
          _startHeroTransition(from, to, animation, flightType, isUserGestureTransition);
        });
      }
    }
  }

  // 找到匹配的hero，开始飞行动画，或者逆转动画
  void _startHeroTransition(
    PageRoute<dynamic> from,
    PageRoute<dynamic> to,
    Animation<double> animation,
    HeroFlightDirection flightType,
    bool isUserGestureTransition,
  ) {
    // 如果一个路由的子树或者 navigator 在调用这个方法之前被移除了，什么都不做
    if (navigator == null || from.subtreeContext == null || to.subtreeContext == null) {
      to.offstage = false;
      return;
    }
	
    /// 找到 navigator 所处的 rect
    /// 其中：navigator 是这个 NavigatorObserver 所观测的 navigator，
    /// _boundingBoxFor 找到了 navigator context 对应的渲染对象的大小，并将其映射成全局坐标
    /// 这样就得到了 navigator 在全局所处的 rect
    final Rect navigatorRect = _boundingBoxFor(navigator.context);

    /// 找到对应页面的 hero 映射表  
    final Map<Object, _HeroState> fromHeroes =
        Hero._allHeroesFor(from.subtreeContext, isUserGestureTransition, navigator);
    final Map<Object, _HeroState> toHeroes =
        Hero._allHeroesFor(to.subtreeContext, isUserGestureTransition, navigator);

    // If the `to` route was offstage, then we're implicitly restoring its
    // animation value back to what it was before it was "moved" offstage.
    to.offstage = false;

    for (Object tag in fromHeroes.keys) {
      if (toHeroes[tag] != null) {
        /// 获取对应的 builder  
        final HeroFlightShuttleBuilder fromShuttleBuilder =
            fromHeroes[tag].widget.flightShuttleBuilder;
        final HeroFlightShuttleBuilder toShuttleBuilder = 
            toHeroes[tag].widget.flightShuttleBuilder;
          
        /// 因为在每次路由飞行完成之后，会自动在 _flight 中移除 tag 和其对应的  _HeroFlight，
        /// 如果 _flights[tag] ！= null，说明 tag 已经不是第一次被使用了，这意味发生了重定向，即 hero 的目
        /// 标路由发生了改变
        final bool isDiverted = _flights[tag] != null;

        /// 构建 manifest  
        final _HeroFlightManifest manifest = _HeroFlightManifest(
          type: flightType,
          overlay: navigator.overlay,
          navigatorRect: navigatorRect,
          fromRoute: from,
          toRoute: to,
          fromHero: fromHeroes[tag],
          toHero: toHeroes[tag],
          createRectTween: createRectTween,
          shuttleBuilder:
              toShuttleBuilder ?? 
              fromShuttleBuilder ??
              _defaultHeroFlightShuttleBuilder,
          isUserGestureTransition: isUserGestureTransition,
          isDiverted: isDiverted,
        );

        if (isDiverted)
          _flights[tag].divert(manifest);
        else
          /// 开始飞行
          _flights[tag] = _HeroFlight(_handleFlightEnded)..start(manifest);
      } else if (_flights[tag] != null) {
        /// 放弃飞行  
        _flights[tag].abort();
      }
    }

    // If the from hero is gone, the flight won't start and the to hero needs to
    // be put on stage again.
    for (Object tag in toHeroes.keys) {
      if (fromHeroes[tag] == null)
        toHeroes[tag].ensurePlaceholderIsHidden();
    }
  }

  void _handleFlightEnded(_HeroFlight flight) {
    /// 完成后从_flight中移除  
    _flights.remove(flight.manifest.tag);
  }

  static final HeroFlightShuttleBuilder _defaultHeroFlightShuttleBuilder = (
    BuildContext flightContext,
    Animation<double> animation,
    HeroFlightDirection flightDirection,
    BuildContext fromHeroContext,
    BuildContext toHeroContext,
  ) {
    final Hero toHero = toHeroContext.widget;
    return toHero.child;
  };
}
```



## TickerMode

```dart
/// 启用或阻止子树中的 tickers (因此动画也会被影响) 
///
/// 只在被使用了使用了 ticker providers创建的 [AnimationController] 的 widget 有效
/// 例如[TickerProviderStateMixin] or a [SingleTickerProviderStateMixin].
class TickerMode extends InheritedWidget {
  const TickerMode({
    Key key,
    @required this.enabled,
    Widget child,
  }) : assert(enabled != null),
       super(key: key, child: child);

  /// true：子树 tick，false：子树不tick
  final bool enabled;

  static bool of(BuildContext context) {
    final TickerMode widget = context.inheritFromWidgetOfExactType(TickerMode);
    return widget?.enabled ?? true;
  }

}
```


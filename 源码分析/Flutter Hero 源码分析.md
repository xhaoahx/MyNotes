# Flutter Hero 源码分析

[TOC]

## 函数原型

```dart
/// 由给定的两个rect返回一个插值rect
/// 通常来说采用非线性的值会比较好看
typedef CreateRectTween = Tween<Rect> Function(Rect begin, Rect end);

/// Signature for a function that builds a [Hero] placeholder widget given a
/// child and a [Size].
///
/// The child can optionally be part of the returned widget tree. The returned
/// widget should typically be constrained to [heroSize], if it doesn't do so
/// implicitly.
///
/// See also:
///  * [TransitionBuilder], which is similar but only takes a [BuildContext]
///    and a child widget.
typedef HeroPlaceholderBuilder = Widget Function(
  BuildContext context,
  Size heroSize,
  Widget child,
);

/// 构建穿梭过程
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
```



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
  }) : assert(tag != null),
       assert(transitionOnUserGestures != null),
       assert(child != null),
       super(key: key);

  /// 唯一标识符
  /// 用于联系两个页面的hero  
  final Object tag;

  /// Defines how the destination hero's bounds change as it flies from the starting
  /// route to the destination route.
  ///
  /// A hero flight begins with the destination hero's [child] aligned with the
  /// starting hero's child. The [Tween<Rect>] returned by this callback is used
  /// to compute the hero's bounds as the flight animation's value goes from 0.0
  /// to 1.0.
  ///
  /// If this property is null, the default, then the value of
  /// [HeroController.createRectTween] is used. The [HeroController] created by
  /// [MaterialApp] creates a [MaterialRectArcTween].
  final CreateRectTween createRectTween;

  /// The widget subtree that will "fly" from one route to another during a
  /// [Navigator] push or pop transition.
  ///
  /// The appearance of this subtree should be similar to the appearance of
  /// the subtrees of any other heroes in the application with the same [tag].
  /// Changes in scale and aspect ratio work well in hero animations, changes
  /// in layout or composition do not.
  ///
  /// {@macro flutter.widgets.child}
  final Widget child;

  /// Optional override to supply a widget that's shown during the hero's flight.
  ///
  /// This in-flight widget can depend on the route transition's animation as
  /// well as the incoming and outgoing routes' [Hero] descendants' widgets and
  /// layout.
  ///
  /// When both the source and destination [Hero]es provide a [flightShuttleBuilder],
  /// the destination's [flightShuttleBuilder] takes precedence.
  ///
  /// If none is provided, the destination route's Hero child is shown in-flight
  /// by default.
  ///
  /// ## Limitations
  ///
  /// If a widget built by [flightShuttleBuilder] takes part in a [Navigator]
  /// push transition, that widget or its descendants must not have any
  /// [GlobalKey] that is used in the source Hero's descendant widgets. That is
  /// because both subtrees will be included in the widget tree during the Hero
  /// flight animation, and [GlobalKey]s must be unique across the entire widget
  /// tree.
  ///
  /// If the said [GlobalKey] is essential to your application, consider providing
  /// a custom [placeholderBuilder] for the source Hero, to avoid the [GlobalKey]
  /// collision, such as a builder that builds an empty [SizedBox], keeping the
  /// Hero [child]'s original size.
  final HeroFlightShuttleBuilder flightShuttleBuilder;

  /// Placeholder widget left in place as the Hero's [child] once the flight takes
  /// off.
  ///
  /// By default the placeholder widget is an empty [SizedBox] keeping the Hero
  /// child's original size, unless this Hero is a source Hero of a [Navigator]
  /// push transition, in which case [child] will be a descendant of the placeholder
  /// and will be kept [Offstage] during the Hero's flight.
  final HeroPlaceholderBuilder placeholderBuilder;

  /// Whether to perform the hero transition if the [PageRoute] transition was
  /// triggered by a user gesture, such as a back swipe on iOS.
  ///
  /// If [Hero]es with the same [tag] on both the from and the to routes have
  /// [transitionOnUserGestures] set to true, a back swipe gesture will
  /// trigger the same hero animation as a programmatically triggered push or
  /// pop.
  ///
  /// The route being popped to or the bottom route must also have
  /// [PageRoute.maintainState] set to true for a gesture triggered hero
  /// transition to work.
  ///
  /// Defaults to false and cannot be null.
  final bool transitionOnUserGestures;

  // Returns a map of all of the heroes in `context` indexed by hero tag that
  // should be considered for animation when `navigator` transitions from one
  // PageRoute to another.
  static Map<Object, _HeroState> _allHeroesFor(
      BuildContext context,
      bool isUserGestureTransition,
      NavigatorState navigator,
  ) {
    assert(context != null);
    assert(isUserGestureTransition != null);
    assert(navigator != null);
    final Map<Object, _HeroState> result = <Object, _HeroState>{};

    void inviteHero(StatefulElement hero, Object tag) {
      assert(() {
        if (result.containsKey(tag)) {
          throw FlutterError(
            'There are multiple heroes that share the same tag within a subtree.\n'
            'Within each subtree for which heroes are to be animated (i.e. a PageRoute subtree), '
            'each Hero must have a unique non-null tag.\n'
            'In this case, multiple heroes had the following tag: $tag\n'
            'Here is the subtree for one of the offending heroes:\n'
            '${hero.toStringDeep(prefixLineOne: "# ")}'
          );
        }
        return true;
      }());
      final Hero heroWidget = hero.widget;
      final _HeroState heroState = hero.state;
      if (!isUserGestureTransition || heroWidget.transitionOnUserGestures) {
        result[tag] = heroState;
      } else {
        // If transition is not allowed, we need to make sure hero is not hidden.
        // A hero can be hidden previously due to hero transition.
        heroState.ensurePlaceholderIsHidden();
      }
    }

    void visitor(Element element) {
      if (element.widget is Hero) {
        final StatefulElement hero = element;
        final Hero heroWidget = element.widget;
        final Object tag = heroWidget.tag;
        assert(tag != null);
        if (Navigator.of(hero) == navigator) {
          inviteHero(hero, tag);
        } else {
          // The nearest navigator to the Hero is not the Navigator that is
          // currently transitioning from one route to another. This means
          // the Hero is inside a nested Navigator and should only be
          // considered for animation if it is part of the top-most route in
          // that nested Navigator and if that route is also a PageRoute.
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
  // Whether the placeholder widget should wrap the hero's child widget as its
  // own child, when `_placeholderSize` is non-null (i.e. the hero is currently
  // in its flight animation). See `startFlight`.
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
    assert(mounted);
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
    assert(
      context.ancestorWidgetOfExactType(Hero) == null,
      'A Hero widget cannot be the descendant of another Hero widget.'
    );

    final bool showPlaceholder = _placeholderSize != null;

    if (showPlaceholder && widget.placeholderBuilder != null) {
      return widget.placeholderBuilder(context, _placeholderSize, widget.child);
    }

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
        offstage: showPlaceholder,
        child: TickerMode(
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

  Animation<double> get animation {
    return CurvedAnimation(
      parent: (type == HeroFlightDirection.push) ? toRoute.animation : fromRoute.animation,
      curve: Curves.fastOutSlowIn,
      reverseCurve: isDiverted ? null : Curves.fastOutSlowIn.flipped,
    );
  }

}
```



## HeroFlight

```dart
// Builds the in-flight hero widget.
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
    final CreateRectTween createRectTween = manifest.toHero.widget.createRectTween ?? manifest.createRectTween;
    if (createRectTween != null)
      return createRectTween(begin, end);
    return RectTween(begin: begin, end: end);
  }

  static final Animatable<double> _reverseTween = Tween<double>(begin: 1.0, end: 0.0);

  // The OverlayEntry WidgetBuilder callback for the hero's overlay.
  Widget _buildOverlay(BuildContext context) {
    assert(manifest != null);
    shuttle ??= manifest.shuttleBuilder(
      context,
      manifest.animation,
      manifest.type,
      manifest.fromHero.context,
      manifest.toHero.context,
    );
    assert(shuttle != null);

    return AnimatedBuilder(
      animation: _proxyAnimation,
      child: shuttle,
      builder: (BuildContext context, Widget child) {
        final RenderBox toHeroBox = manifest.toHero.context?.findRenderObject();
        if (_aborted || toHeroBox == null || !toHeroBox.attached) {
          // The toHero no longer exists or it's no longer the flight's destination.
          // Continue flying while fading out.
          if (_heroOpacity.isCompleted) {
            _heroOpacity = _proxyAnimation.drive(
              _reverseTween.chain(CurveTween(curve: Interval(_proxyAnimation.value, 1.0))),
            );
          }
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
/// A [Navigator] observer that manages [Hero] transitions.
///
/// An instance of [HeroController] should be used in [Navigator.observers].
/// This is done automatically by [MaterialApp].
class HeroController extends NavigatorObserver {
  /// Creates a hero controller with the given [RectTween] constructor if any.
  ///
  /// The [createRectTween] argument is optional. If null, the controller uses a
  /// linear [Tween<Rect>].
  HeroController({ this.createRectTween });

  /// Used to create [RectTween]s that interpolate the position of heroes in flight.
  ///
  /// If null, the controller uses a linear [RectTween].
  final CreateRectTween createRectTween;

  // All of the heroes that are currently in the overlay and in motion.
  // Indexed by the hero tag.
  final Map<Object, _HeroFlight> _flights = <Object, _HeroFlight>{};

  @override
  void didPush(Route<dynamic> route, Route<dynamic> previousRoute) {
    assert(navigator != null);
    assert(route != null);
    _maybeStartHeroTransition(previousRoute, route, HeroFlightDirection.push, false);
  }

  @override
  void didPop(Route<dynamic> route, Route<dynamic> previousRoute) {
    assert(navigator != null);
    assert(route != null);
    // Don't trigger another flight when a pop is committed as a user gesture
    // back swipe is snapped.
    if (!navigator.userGestureInProgress)
      _maybeStartHeroTransition(route, previousRoute, HeroFlightDirection.pop, false);
  }

  @override
  void didReplace({ Route<dynamic> newRoute, Route<dynamic> oldRoute }) {
    assert(navigator != null);
    if (newRoute?.isCurrent == true) {
      // Only run hero animations if the top-most route got replaced.
      _maybeStartHeroTransition(oldRoute, newRoute, HeroFlightDirection.push, false);
    }
  }

  @override
  void didStartUserGesture(Route<dynamic> route, Route<dynamic> previousRoute) {
    assert(navigator != null);
    assert(route != null);
    _maybeStartHeroTransition(route, previousRoute, HeroFlightDirection.pop, true);
  }

  // If we're transitioning between different page routes, start a hero transition
  // after the toRoute has been laid out with its animation's value at 1.0.
  void _maybeStartHeroTransition(
    Route<dynamic> fromRoute,
    Route<dynamic> toRoute,
    HeroFlightDirection flightType,
    bool isUserGestureTransition,
  ) {
    if (toRoute != fromRoute && toRoute is PageRoute<dynamic> && fromRoute is PageRoute<dynamic>) {
      final PageRoute<dynamic> from = fromRoute;
      final PageRoute<dynamic> to = toRoute;
      final Animation<double> animation = (flightType == HeroFlightDirection.push) ? to.animation : from.animation;

      // A user gesture may have already completed the pop, or we might be the initial route
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

      // For pop transitions driven by a user gesture: if the "to" page has
      // maintainState = true, then the hero's final dimensions can be measured
      // immediately because their page's layout is still valid.
      if (isUserGestureTransition && flightType == HeroFlightDirection.pop && to.maintainState) {
        _startHeroTransition(from, to, animation, flightType, isUserGestureTransition);
      } else {
        // Otherwise, delay measuring until the end of the next frame to allow
        // the 'to' route to build and layout.

        // Putting a route offstage changes its animation value to 1.0. Once this
        // frame completes, we'll know where the heroes in the `to` route are
        // going to end up, and the `to` route will go back onstage.
        to.offstage = to.animation.value == 0.0;

        WidgetsBinding.instance.addPostFrameCallback((Duration value) {
          _startHeroTransition(from, to, animation, flightType, isUserGestureTransition);
        });
      }
    }
  }

  // Find the matching pairs of heroes in from and to and either start or a new
  // hero flight, or divert an existing one.
  void _startHeroTransition(
    PageRoute<dynamic> from,
    PageRoute<dynamic> to,
    Animation<double> animation,
    HeroFlightDirection flightType,
    bool isUserGestureTransition,
  ) {
    // If the navigator or one of the routes subtrees was removed before this
    // end-of-frame callback was called, then don't actually start a transition.
    if (navigator == null || from.subtreeContext == null || to.subtreeContext == null) {
      to.offstage = false; // in case we set this in _maybeStartHeroTransition
      return;
    }

    final Rect navigatorRect = _boundingBoxFor(navigator.context);

    // At this point the toHeroes may have been built and laid out for the first time.
    final Map<Object, _HeroState> fromHeroes = Hero._allHeroesFor(from.subtreeContext, isUserGestureTransition, navigator);
    final Map<Object, _HeroState> toHeroes = Hero._allHeroesFor(to.subtreeContext, isUserGestureTransition, navigator);

    // If the `to` route was offstage, then we're implicitly restoring its
    // animation value back to what it was before it was "moved" offstage.
    to.offstage = false;

    for (Object tag in fromHeroes.keys) {
      if (toHeroes[tag] != null) {
        final HeroFlightShuttleBuilder fromShuttleBuilder = fromHeroes[tag].widget.flightShuttleBuilder;
        final HeroFlightShuttleBuilder toShuttleBuilder = toHeroes[tag].widget.flightShuttleBuilder;
        final bool isDiverted = _flights[tag] != null;

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
              toShuttleBuilder ?? fromShuttleBuilder ?? _defaultHeroFlightShuttleBuilder,
          isUserGestureTransition: isUserGestureTransition,
          isDiverted: isDiverted,
        );

        if (isDiverted)
          _flights[tag].divert(manifest);
        else
          _flights[tag] = _HeroFlight(_handleFlightEnded)..start(manifest);
      } else if (_flights[tag] != null) {
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


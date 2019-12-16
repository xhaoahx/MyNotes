## Flutter ScrollPhysics  源码分析

## ScrollPhysics 

```dart
/// Determines the physics of a [Scrollable] widget.
///
/// For example, determines how the [Scrollable] will behave when the user
/// reaches the maximum scroll extent or when the user stops scrolling.
///
/// When starting a physics [Simulation], the current scroll position and
/// velocity are used as the initial conditions for the particle in the
/// simulation. The movement of the particle in the simulation is then used to
/// determine the scroll position for the widget.
///
/// Instead of creating your own subclasses, [parent] can be used to combine
/// [ScrollPhysics] objects of different types to get the desired scroll physics.
@immutable
class ScrollPhysics {
  /// Creates an object with the default scroll physics.
  const ScrollPhysics({ this.parent });

  /// If non-null, determines the default behavior for each method.
  ///
  /// If a subclass of [ScrollPhysics] does not override a method, that subclass
  /// will inherit an implementation from this base class that defers to
  /// [parent]. This mechanism lets you assemble novel combinations of
  /// [ScrollPhysics] subclasses at runtime. For example:
  ///
  /// ```dart
  /// BouncingScrollPhysics(parent: AlwaysScrollableScrollPhysics())
  ///
  /// ```
  /// will result in a [ScrollPhysics] that has the combined behavior
  /// of [BouncingScrollPhysics] and [AlwaysScrollableScrollPhysics]:
  /// behaviors that are not specified in [BouncingScrollPhysics]
  /// (e.g. [shouldAcceptUserOffset]) will defer to [AlwaysScrollableScrollPhysics].
  final ScrollPhysics parent;

  /// If [parent] is null then return ancestor, otherwise recursively build a
  /// ScrollPhysics that has [ancestor] as its parent.
  ///
  /// This method is typically used to define [applyTo] methods like:
  ///
  /// ```dart
  /// FooScrollPhysics applyTo(ScrollPhysics ancestor) {
  ///   return FooScrollPhysics(parent: buildParent(ancestor));
  /// }
  /// ```
  @protected
  ScrollPhysics buildParent(ScrollPhysics ancestor) => parent?.applyTo(ancestor) ?? ancestor;

  /// If [parent] is null then return a [ScrollPhysics] with the same
  /// [runtimeType] where the [parent] has been replaced with the [ancestor].
  ///
  /// If this scroll physics object already has a parent, then this method
  /// is applied recursively and ancestor will appear at the end of the
  /// existing chain of parents.
  ///
  /// The returned object will combine some of the behaviors from this
  /// [ScrollPhysics] instance and some of the behaviors from [ancestor].
  ///
  /// {@tool sample}
  ///
  /// In the following example, the [applyTo] method is used to combine the
  /// scroll physics of two [ScrollPhysics] objects, the resulting [ScrollPhysics]
  /// `x` has the same behavior as `y`:
  ///
  /// ```dart
  /// final FooScrollPhysics x = FooScrollPhysics().applyTo(BarScrollPhysics());
  /// const FooScrollPhysics y = FooScrollPhysics(parent: BarScrollPhysics());
  /// ```
  /// {@end-tool}
  ///
  /// See also:
  ///
  ///  * [buildParent], a utility method that's often used to define [applyTo]
  ///    methods for ScrollPhysics subclasses.
  ScrollPhysics applyTo(ScrollPhysics ancestor) {
    return ScrollPhysics(parent: buildParent(ancestor));
  }

  /// Used by [DragScrollActivity] and other user-driven activities to convert
  /// an offset in logical pixels as provided by the [DragUpdateDetails] into a
  /// delta to apply (subtract from the current position) using
  /// [ScrollActivityDelegate.setPixels].
  ///
  /// This is used by some [ScrollPosition] subclasses to apply friction during
  /// overscroll situations.
  ///
  /// This method must not adjust parts of the offset that are entirely within
  /// the bounds described by the given `position`.
  ///
  /// The given `position` is only valid during this method call. Do not keep a
  /// reference to it to use later, as the values may update, may not update, or
  /// may update to reflect an entirely unrelated scrollable.
  double applyPhysicsToUserOffset(ScrollMetrics position, double offset) {
    if (parent == null)
      return offset;
    return parent.applyPhysicsToUserOffset(position, offset);
  }

  /// Whether the scrollable should let the user adjust the scroll offset, for
  /// example by dragging.
  ///
  /// By default, the user can manipulate the scroll offset if, and only if,
  /// there is actually content outside the viewport to reveal.
  ///
  /// The given `position` is only valid during this method call. Do not keep a
  /// reference to it to use later, as the values may update, may not update, or
  /// may update to reflect an entirely unrelated scrollable.
  bool shouldAcceptUserOffset(ScrollMetrics position) {
    if (parent == null)
      return position.pixels != 0.0 || position.minScrollExtent != position.maxScrollExtent;
    return parent.shouldAcceptUserOffset(position);
  }

  /// Determines the overscroll by applying the boundary conditions.
  ///
  /// Called by [ScrollPosition.applyBoundaryConditions], which is called by
  /// [ScrollPosition.setPixels] just before the [ScrollPosition.pixels] value
  /// is updated, to determine how much of the offset is to be clamped off and
  /// sent to [ScrollPosition.didOverscrollBy].
  ///
  /// The `value` argument is guaranteed to not equal the [ScrollMetrics.pixels]
  /// of the `position` argument when this is called.
  ///
  /// It is possible for this method to be called when the `position` describes
  /// an already-out-of-bounds position. In that case, the boundary conditions
  /// should usually only prevent a further increase in the extent to which the
  /// position is out of bounds, allowing a decrease to be applied successfully,
  /// so that (for instance) an animation can smoothly snap an out of bounds
  /// position to the bounds. See [BallisticScrollActivity].
  ///
  /// This method must not clamp parts of the offset that are entirely within
  /// the bounds described by the given `position`.
  ///
  /// The given `position` is only valid during this method call. Do not keep a
  /// reference to it to use later, as the values may update, may not update, or
  /// may update to reflect an entirely unrelated scrollable.
  ///
  /// ## Examples
  ///
  /// [BouncingScrollPhysics] returns zero. In other words, it allows scrolling
  /// past the boundary unhindered.
  ///
  /// [ClampingScrollPhysics] returns the amount by which the value is beyond
  /// the position or the boundary, whichever is furthest from the content. In
  /// other words, it disallows scrolling past the boundary, but allows
  /// scrolling back from being overscrolled, if for some reason the position
  /// ends up overscrolled.
  double applyBoundaryConditions(ScrollMetrics position, double value) {
    if (parent == null)
      return 0.0;
    return parent.applyBoundaryConditions(position, value);
  }

  /// Returns a simulation for ballistic scrolling starting from the given
  /// position with the given velocity.
  ///
  /// This is used by [ScrollPositionWithSingleContext] in the
  /// [ScrollPositionWithSingleContext.goBallistic] method. If the result
  /// is non-null, [ScrollPositionWithSingleContext] will begin a
  /// [BallisticScrollActivity] with the returned value. Otherwise, it will
  /// begin an idle activity instead.
  ///
  /// The given `position` is only valid during this method call. Do not keep a
  /// reference to it to use later, as the values may update, may not update, or
  /// may update to reflect an entirely unrelated scrollable.
  Simulation createBallisticSimulation(ScrollMetrics position, double velocity) {
    if (parent == null)
      return null;
    return parent.createBallisticSimulation(position, velocity);
  }

  static final SpringDescription _kDefaultSpring = SpringDescription.withDampingRatio(
    mass: 0.5,
    stiffness: 100.0,
    ratio: 1.1,
  );

  /// The spring to use for ballistic simulations.
  SpringDescription get spring => parent?.spring ?? _kDefaultSpring;

  /// The default accuracy to which scrolling is computed.
  static final Tolerance _kDefaultTolerance = Tolerance(
    // TODO(ianh): Handle the case of the device pixel ratio changing.
    // TODO(ianh): Get this from the local MediaQuery not dart:ui's window object.
    velocity: 1.0 / (0.050 * WidgetsBinding.instance.window.devicePixelRatio), // logical pixels per second
    distance: 1.0 / WidgetsBinding.instance.window.devicePixelRatio, // logical pixels
  );

  /// The tolerance to use for ballistic simulations.
  Tolerance get tolerance => parent?.tolerance ?? _kDefaultTolerance;

  /// The minimum distance an input pointer drag must have moved to
  /// to be considered a scroll fling gesture.
  ///
  /// This value is typically compared with the distance traveled along the
  /// scrolling axis.
  ///
  /// See also:
  ///
  ///  * [VelocityTracker.getVelocityEstimate], which computes the velocity
  ///    of a press-drag-release gesture.
  double get minFlingDistance => parent?.minFlingDistance ?? kTouchSlop;

  /// The minimum velocity for an input pointer drag to be considered a
  /// scroll fling.
  ///
  /// This value is typically compared with the magnitude of fling gesture's
  /// velocity along the scrolling axis.
  ///
  /// See also:
  ///
  ///  * [VelocityTracker.getVelocityEstimate], which computes the velocity
  ///    of a press-drag-release gesture.
  double get minFlingVelocity => parent?.minFlingVelocity ?? kMinFlingVelocity;

  /// Scroll fling velocity magnitudes will be clamped to this value.
  double get maxFlingVelocity => parent?.maxFlingVelocity ?? kMaxFlingVelocity;

  /// Returns the velocity carried on repeated flings.
  ///
  /// The function is applied to the existing scroll velocity when another
  /// scroll drag is applied in the same direction.
  ///
  /// By default, physics for platforms other than iOS doesn't carry momentum.
  double carriedMomentum(double existingVelocity) {
    if (parent == null)
      return 0.0;
    return parent.carriedMomentum(existingVelocity);
  }

  /// The minimum amount of pixel distance drags must move by to start motion
  /// the first time or after each time the drag motion stopped.
  ///
  /// If null, no minimum threshold is enforced.
  double get dragStartDistanceMotionThreshold => parent?.dragStartDistanceMotionThreshold;

  /// Whether a viewport is allowed to change its scroll position implicitly in
  /// response to a call to [RenderObject.showOnScreen].
  ///
  /// [RenderObject.showOnScreen] is for example used to bring a text field
  /// fully on screen after it has received focus. This property controls
  /// whether the viewport associated with this object is allowed to change the
  /// scroll position to fulfill such a request.
  bool get allowImplicitScrolling => true;

  @override
  String toString() {
    if (parent == null)
      return runtimeType.toString();
    return '$runtimeType -> $parent';
  }
}
```


# Flutter Viewport 源码分析

## ViewportOffset

```dart
/// Which part of the content inside the viewport should be visible.
///
/// The [pixels] value determines the scroll offset that the viewport uses to
/// select which part of its content to display. As the user scrolls the
/// viewport, this value changes, which changes the content that is displayed.
///
/// This object is a [Listenable] that notifies its listeners when [pixels]
/// changes.
///
/// See also:
///
///  * [ScrollPosition], which is a commonly used concrete subclass.
///  * [RenderViewportBase], which is a render object that uses viewport
///    offsets.
abstract class ViewportOffset extends ChangeNotifier {
  /// Default constructor.
  ///
  /// Allows subclasses to construct this object directly.
  ViewportOffset();

  /// Creates a viewport offset with the given [pixels] value.
  ///
  /// The [pixels] value does not change unless the viewport issues a
  /// correction.
  factory ViewportOffset.fixed(double value) = _FixedViewportOffset;

  /// Creates a viewport offset with a [pixels] value of 0.0.
  ///
  /// The [pixels] value does not change unless the viewport issues a
  /// correction.
  factory ViewportOffset.zero() = _FixedViewportOffset.zero;

  /// The number of pixels to offset the children in the opposite of the axis direction.
  ///
  /// For example, if the axis direction is down, then the pixel value
  /// represents the number of logical pixels to move the children _up_ the
  /// screen. Similarly, if the axis direction is left, then the pixels value
  /// represents the number of logical pixels to move the children to _right_.
  ///
  /// This object notifies its listeners when this value changes (except when
  /// the value changes due to [correctBy]).
  double get pixels;

  /// Called when the viewport's extents are established.
  ///
  /// The argument is the dimension of the [RenderViewport] in the main axis
  /// (e.g. the height, for a vertical viewport).
  ///
  /// This may be called redundantly, with the same value, each frame. This is
  /// called during layout for the [RenderViewport]. If the viewport is
  /// configured to shrink-wrap its contents, it may be called several times,
  /// since the layout is repeated each time the scroll offset is corrected.
  ///
  /// If this is called, it is called before [applyContentDimensions]. If this
  /// is called, [applyContentDimensions] will be called soon afterwards in the
  /// same layout phase. If the viewport is not configured to shrink-wrap its
  /// contents, then this will only be called when the viewport recomputes its
  /// size (i.e. when its parent lays out), and not during normal scrolling.
  ///
  /// If applying the viewport dimensions changes the scroll offset, return
  /// false. Otherwise, return true. If you return false, the [RenderViewport]
  /// will be laid out again with the new scroll offset. This is expensive. (The
  /// return value is answering the question "did you accept these viewport
  /// dimensions unconditionally?"; if the new dimensions change the
  /// [ViewportOffset]'s actual [pixels] value, then the viewport will need to
  /// be laid out again.)
  bool applyViewportDimension(double viewportDimension);

  /// Called when the viewport's content extents are established.
  ///
  /// The arguments are the minimum and maximum scroll extents respectively. The
  /// minimum will be equal to or less than the maximum. In the case of slivers,
  /// the minimum will be equal to or less than zero, the maximum will be equal
  /// to or greater than zero.
  ///
  /// The maximum scroll extent has the viewport dimension subtracted from it.
  /// For instance, if there is 100.0 pixels of scrollable content, and the
  /// viewport is 80.0 pixels high, then the minimum scroll extent will
  /// typically be 0.0 and the maximum scroll extent will typically be 20.0,
  /// because there's only 20.0 pixels of actual scroll slack.
  ///
  /// If applying the content dimensions changes the scroll offset, return
  /// false. Otherwise, return true. If you return false, the [RenderViewport]
  /// will be laid out again with the new scroll offset. This is expensive. (The
  /// return value is answering the question "did you accept these content
  /// dimensions unconditionally?"; if the new dimensions change the
  /// [ViewportOffset]'s actual [pixels] value, then the viewport will need to
  /// be laid out again.)
  ///
  /// This is called at least once each time the [RenderViewport] is laid out,
  /// even if the values have not changed. It may be called many times if the
  /// scroll offset is corrected (if this returns false). This is always called
  /// after [applyViewportDimension], if that method is called.
  bool applyContentDimensions(double minScrollExtent, double maxScrollExtent);

  /// Apply a layout-time correction to the scroll offset.
  ///
  /// This method should change the [pixels] value by `correction`, but without
  /// calling [notifyListeners]. It is called during layout by the
  /// [RenderViewport], before [applyContentDimensions]. After this method is
  /// called, the layout will be recomputed and that may result in this method
  /// being called again, though this should be very rare.
  ///
  /// See also:
  ///
  ///  * [jumpTo], for also changing the scroll position when not in layout.
  ///    [jumpTo] applies the change immediately and notifies its listeners.
  void correctBy(double correction);

  /// Jumps [pixels] from its current value to the given value,
  /// without animation, and without checking if the new value is in range.
  ///
  /// See also:
  ///
  ///  * [correctBy], for changing the current offset in the middle of layout
  ///    and that defers the notification of its listeners until after layout.
  void jumpTo(double pixels);

  /// Animates [pixels] from its current value to the given value.
  ///
  /// The returned [Future] will complete when the animation ends, whether it
  /// completed successfully or whether it was interrupted prematurely.
  ///
  /// The duration must not be zero. To jump to a particular value without an
  /// animation, use [jumpTo].
  Future<void> animateTo(
    double to, {
    @required Duration duration,
    @required Curve curve,
  });

  /// Calls [jumpTo] if duration is null or [Duration.zero], otherwise
  /// [animateTo] is called.
  ///
  /// If [animateTo] is called then [curve] defaults to [Curves.ease]. The
  /// [clamp] parameter is ignored by this stub implementation but subclasses
  /// like [ScrollPosition] handle it by adjusting [to] to prevent over or
  /// underscroll.
  Future<void> moveTo(
    double to, {
    Duration duration,
    Curve curve,
    bool clamp,
  }) {
    assert(to != null);
    if (duration == null || duration == Duration.zero) {
      jumpTo(to);
      return Future<void>.value();
    } else {
      return animateTo(to, duration: duration, curve: curve ?? Curves.ease);
    }
  }

  /// The direction in which the user is trying to change [pixels], relative to
  /// the viewport's [RenderViewport.axisDirection].
  ///
  /// If the _user_ is not scrolling, this will return [ScrollDirection.idle]
  /// even if there is (for example) a [ScrollActivity] currently animating the
  /// position.
  ///
  /// This is exposed in [SliverConstraints.userScrollDirection], which is used
  /// by some slivers to determine how to react to a change in scroll offset.
  /// For example, [RenderSliverFloatingPersistentHeader] will only expand a
  /// floating app bar when the [userScrollDirection] is in the positive scroll
  /// offset direction.
  ScrollDirection get userScrollDirection;

  /// Whether a viewport is allowed to change [pixels] implicitly to respond to
  /// a call to [RenderObject.showOnScreen].
  ///
  /// [RenderObject.showOnScreen] is for example used to bring a text field
  /// fully on screen after it has received focus. This property controls
  /// whether the viewport associated with this offset is allowed to change the
  /// offset's [pixels] value to fulfill such a request.
  bool get allowImplicitScrolling;

  @override
  String toString() {
    final List<String> description = <String>[];
    debugFillDescription(description);
    return '${describeIdentity(this)}(${description.join(", ")})';
  }

  /// Add additional information to the given description for use by [toString].
  ///
  /// This method makes it easier for subclasses to coordinate to provide a
  /// high-quality [toString] implementation. The [toString] implementation on
  /// the [State] base class calls [debugFillDescription] to collect useful
  /// information from subclasses to incorporate into its return value.
  ///
  /// If you override this, make sure to start your method with a call to
  /// `super.debugFillDescription(description)`.
  @mustCallSuper
  void debugFillDescription(List<String> description) {
    description.add('offset: ${pixels?.toStringAsFixed(1)}');
  }
}
```



## ScrollMetrics

```dart
/// A description of a [Scrollable]'s contents, useful for modeling the state
/// of its viewport.
///
/// This class defines a current position, [pixels], and a range of values
/// considered "in bounds" for that position. The range has a minimum value at
/// [minScrollExtent] and a maximum value at [maxScrollExtent] (inclusive). The
/// viewport scrolls in the direction and axis described by [axisDirection]
/// and [axis].
///
/// The [outOfRange] getter will return true if [pixels] is outside this defined
/// range. The [atEdge] getter will return true if the [pixels] position equals
/// either the [minScrollExtent] or the [maxScrollExtent].
///
/// The dimensions of the viewport in the given [axis] are described by
/// [viewportDimension].
///
/// The above values are also exposed in terms of [extentBefore],
/// [extentInside], and [extentAfter], which may be more useful for use cases
/// such as scroll bars; for example, see [Scrollbar].
///
/// See also:
///
///  * [FixedScrollMetrics], which is an immutable object that implements this
///    interface.
abstract class ScrollMetrics {
  /// Creates a [ScrollMetrics] that has the same properties as this object.
  ///
  /// This is useful if this object is mutable, but you want to get a snapshot
  /// of the current state.
  ///
  /// The named arguments allow the values to be adjusted in the process. This
  /// is useful to examine hypothetical situations, for example "would applying
  /// this delta unmodified take the position [outOfRange]?".
  ScrollMetrics copyWith({
    double minScrollExtent,
    double maxScrollExtent,
    double pixels,
    double viewportDimension,
    AxisDirection axisDirection,
  }) {
    return FixedScrollMetrics(
      minScrollExtent: minScrollExtent ?? this.minScrollExtent,
      maxScrollExtent: maxScrollExtent ?? this.maxScrollExtent,
      pixels: pixels ?? this.pixels,
      viewportDimension: viewportDimension ?? this.viewportDimension,
      axisDirection: axisDirection ?? this.axisDirection,
    );
  }

  /// The minimum in-range value for [pixels].
  ///
  /// The actual [pixels] value might be [outOfRange].
  ///
  /// This value should typically be non-null and less than or equal to
  /// [maxScrollExtent]. It can be negative infinity, if the scroll is unbounded.
  double get minScrollExtent;

  /// The maximum in-range value for [pixels].
  ///
  /// The actual [pixels] value might be [outOfRange].
  ///
  /// This value should typically be non-null and greater than or equal to
  /// [minScrollExtent]. It can be infinity, if the scroll is unbounded.
  double get maxScrollExtent;

  /// The current scroll position, in logical pixels along the [axisDirection].
  double get pixels;

  /// The extent of the viewport along the [axisDirection].
  double get viewportDimension;

  /// The direction in which the scroll view scrolls.
  AxisDirection get axisDirection;

  /// The axis in which the scroll view scrolls.
  Axis get axis => axisDirectionToAxis(axisDirection);

  /// Whether the [pixels] value is outside the [minScrollExtent] and
  /// [maxScrollExtent].
  bool get outOfRange => pixels < minScrollExtent || pixels > maxScrollExtent;

  /// Whether the [pixels] value is exactly at the [minScrollExtent] or the
  /// [maxScrollExtent].
  bool get atEdge => pixels == minScrollExtent || pixels == maxScrollExtent;

  /// The quantity of content conceptually "above" the viewport in the scrollable.
  /// This is the content above the content described by [extentInside].
  double get extentBefore => math.max(pixels - minScrollExtent, 0.0);

  /// The quantity of content conceptually "inside" the viewport in the scrollable.
  ///
  /// The value is typically the height of the viewport when [outOfRange] is false.
  /// It could be less if there is less content visible than the size of the
  /// viewport, such as when overscrolling.
  ///
  /// The value is always non-negative, and less than or equal to [viewportDimension].
  double get extentInside {
    assert(minScrollExtent <= maxScrollExtent);
    return viewportDimension
      // "above" overscroll value
      - (minScrollExtent - pixels).clamp(0, viewportDimension)
      // "below" overscroll value
      - (pixels - maxScrollExtent).clamp(0, viewportDimension);
  }

  /// The quantity of content conceptually "below" the viewport in the scrollable.
  /// This is the content below the content described by [extentInside].
  double get extentAfter => math.max(maxScrollExtent - pixels, 0.0);
}
```



## ScrollPostion

```dart
/// Determines which portion of the content is visible in a scroll view.
///
/// The [pixels] value determines the scroll offset that the scroll view uses to
/// select which part of its content to display. As the user scrolls the
/// viewport, this value changes, which changes the content that is displayed.
///
/// The [ScrollPosition] applies [physics] to scrolling, and stores the
/// [minScrollExtent] and [maxScrollExtent].
///
/// Scrolling is controlled by the current [activity], which is set by
/// [beginActivity]. [ScrollPosition] itself does not start any activities.
/// Instead, concrete subclasses, such as [ScrollPositionWithSingleContext],
/// typically start activities in response to user input or instructions from a
/// [ScrollController].
///
/// This object is a [Listenable] that notifies its listeners when [pixels]
/// changes.
///
/// ## Subclassing ScrollPosition
///
/// Over time, a [Scrollable] might have many different [ScrollPosition]
/// objects. For example, if [Scrollable.physics] changes type, [Scrollable]
/// creates a new [ScrollPosition] with the new physics. To transfer state from
/// the old instance to the new instance, subclasses implement [absorb]. See
/// [absorb] for more details.
///
/// Subclasses also need to call [didUpdateScrollDirection] whenever
/// [userScrollDirection] changes values.
///
/// See also:
///
///  * [Scrollable], which uses a [ScrollPosition] to determine which portion of
///    its content to display.
///  * [ScrollController], which can be used with [ListView], [GridView] and
///    other scrollable widgets to control a [ScrollPosition].
///  * [ScrollPositionWithSingleContext], which is the most commonly used
///    concrete subclass of [ScrollPosition].
///  * [ScrollNotification] and [NotificationListener], which can be used to watch
///    the scroll position without using a [ScrollController].
abstract class ScrollPosition extends ViewportOffset with ScrollMetrics {
  /// Creates an object that determines which portion of the content is visible
  /// in a scroll view.
  ///
  /// The [physics], [context], and [keepScrollOffset] parameters must not be null.
  ScrollPosition({
    @required this.physics,
    @required this.context,
    this.keepScrollOffset = true,
    ScrollPosition oldPosition,
    this.debugLabel,
  }) : assert(physics != null),
       assert(context != null),
       assert(context.vsync != null),
       assert(keepScrollOffset != null) {
    if (oldPosition != null)
      absorb(oldPosition);
    if (keepScrollOffset)
      restoreScrollOffset();
  }

  /// How the scroll position should respond to user input.
  ///
  /// For example, determines how the widget continues to animate after the
  /// user stops dragging the scroll view.
  final ScrollPhysics physics;

  /// Where the scrolling is taking place.
  ///
  /// Typically implemented by [ScrollableState].
  final ScrollContext context;

  /// Save the current scroll offset with [PageStorage] and restore it if
  /// this scroll position's scrollable is recreated.
  ///
  /// See also:
  ///
  ///  * [ScrollController.keepScrollOffset] and [PageController.keepPage], which
  ///    create scroll positions and initialize this property.
  final bool keepScrollOffset;

  /// A label that is used in the [toString] output.
  ///
  /// Intended to aid with identifying animation controller instances in debug
  /// output.
  final String debugLabel;

  @override
  double get minScrollExtent => _minScrollExtent;
  double _minScrollExtent;

  @override
  double get maxScrollExtent => _maxScrollExtent;
  double _maxScrollExtent;

  @override
  double get pixels => _pixels;
  double _pixels;

  @override
  double get viewportDimension => _viewportDimension;
  double _viewportDimension;

  /// Whether [viewportDimension], [minScrollExtent], [maxScrollExtent],
  /// [outOfRange], and [atEdge] are available.
  ///
  /// Set to true just before the first time [applyNewDimensions] is called.
  bool get haveDimensions => _haveDimensions;
  bool _haveDimensions = false;

  /// Take any current applicable state from the given [ScrollPosition].
  ///
  /// This method is called by the constructor if it is given an `oldPosition`.
  /// The `other` argument might not have the same [runtimeType] as this object.
  ///
  /// This method can be destructive to the other [ScrollPosition]. The other
  /// object must be disposed immediately after this call (in the same call
  /// stack, before microtask resolution, by whomever called this object's
  /// constructor).
  ///
  /// If the old [ScrollPosition] object is a different [runtimeType] than this
  /// one, the [ScrollActivity.resetActivity] method is invoked on the newly
  /// adopted [ScrollActivity].
  ///
  /// ## Overriding
  ///
  /// Overrides of this method must call `super.absorb` after setting any
  /// metrics-related or activity-related state, since this method may restart
  /// the activity and scroll activities tend to use those metrics when being
  /// restarted.
  ///
  /// Overrides of this method might need to start an [IdleScrollActivity] if
  /// they are unable to absorb the activity from the other [ScrollPosition].
  ///
  /// Overrides of this method might also need to update the delegates of
  /// absorbed scroll activities if they use themselves as a
  /// [ScrollActivityDelegate].
  @protected
  @mustCallSuper
  void absorb(ScrollPosition other) {
    assert(other != null);
    assert(other.context == context);
    assert(_pixels == null);
    _minScrollExtent = other.minScrollExtent;
    _maxScrollExtent = other.maxScrollExtent;
    _pixels = other._pixels;
    _viewportDimension = other.viewportDimension;

    assert(activity == null);
    assert(other.activity != null);
    _activity = other.activity;
    other._activity = null;
    if (other.runtimeType != runtimeType)
      activity.resetActivity();
    context.setIgnorePointer(activity.shouldIgnorePointer);
    isScrollingNotifier.value = activity.isScrolling;
  }

  /// Update the scroll position ([pixels]) to a given pixel value.
  ///
  /// This should only be called by the current [ScrollActivity], either during
  /// the transient callback phase or in response to user input.
  ///
  /// Returns the overscroll, if any. If the return value is 0.0, that means
  /// that [pixels] now returns the given `value`. If the return value is
  /// positive, then [pixels] is less than the requested `value` by the given
  /// amount (overscroll past the max extent), and if it is negative, it is
  /// greater than the requested `value` by the given amount (underscroll past
  /// the min extent).
  ///
  /// The amount of overscroll is computed by [applyBoundaryConditions].
  ///
  /// The amount of the change that is applied is reported using [didUpdateScrollPositionBy].
  /// If there is any overscroll, it is reported using [didOverscrollBy].
  double setPixels(double newPixels) {
    assert(_pixels != null);
    assert(SchedulerBinding.instance.schedulerPhase.index <= SchedulerPhase.transientCallbacks.index);
    if (newPixels != pixels) {
      final double overscroll = applyBoundaryConditions(newPixels);
      assert(() {
        final double delta = newPixels - pixels;
        if (overscroll.abs() > delta.abs()) {
          throw FlutterError.fromParts(<DiagnosticsNode>[
            ErrorSummary('$runtimeType.applyBoundaryConditions returned invalid overscroll value.'),
            ErrorDescription(
              'setPixels() was called to change the scroll offset from $pixels to $newPixels.\n'
              'That is a delta of $delta units.\n'
              '$runtimeType.applyBoundaryConditions reported an overscroll of $overscroll units.'
            )
          ]);
        }
        return true;
      }());
      final double oldPixels = _pixels;
      _pixels = newPixels - overscroll;
      if (_pixels != oldPixels) {
        notifyListeners();
        didUpdateScrollPositionBy(_pixels - oldPixels);
      }
      if (overscroll != 0.0) {
        didOverscrollBy(overscroll);
        return overscroll;
      }
    }
    return 0.0;
  }

  /// Change the value of [pixels] to the new value, without notifying any
  /// customers.
  ///
  /// This is used to adjust the position while doing layout. In particular,
  /// this is typically called as a response to [applyViewportDimension] or
  /// [applyContentDimensions] (in both cases, if this method is called, those
  /// methods should then return false to indicate that the position has been
  /// adjusted).
  ///
  /// Calling this is rarely correct in other contexts. It will not immediately
  /// cause the rendering to change, since it does not notify the widgets or
  /// render objects that might be listening to this object: they will only
  /// change when they next read the value, which could be arbitrarily later. It
  /// is generally only appropriate in the very specific case of the value being
  /// corrected during layout (since then the value is immediately read), in the
  /// specific case of a [ScrollPosition] with a single viewport customer.
  ///
  /// To cause the position to jump or animate to a new value, consider [jumpTo]
  /// or [animateTo], which will honor the normal conventions for changing the
  /// scroll offset.
  ///
  /// To force the [pixels] to a particular value without honoring the normal
  /// conventions for changing the scroll offset, consider [forcePixels]. (But
  /// see the discussion there for why that might still be a bad idea.)
  ///
  /// See also:
  ///
  ///  * [correctBy], which is a method of [ViewportOffset] used
  ///    by viewport render objects to correct the offset during layout
  ///    without notifying its listeners.
  ///  * [jumpTo], for making changes to position while not in the
  ///    middle of layout and applying the new position immediately.
  ///  * [animateTo], which is like [jumpTo] but animating to the
  ///    destination offset.
  void correctPixels(double value) {
    _pixels = value;
  }

  /// Apply a layout-time correction to the scroll offset.
  ///
  /// This method should change the [pixels] value by `correction`, but without
  /// calling [notifyListeners]. It is called during layout by the
  /// [RenderViewport], before [applyContentDimensions]. After this method is
  /// called, the layout will be recomputed and that may result in this method
  /// being called again, though this should be very rare.
  ///
  /// See also:
  ///
  ///  * [jumpTo], for also changing the scroll position when not in layout.
  ///    [jumpTo] applies the change immediately and notifies its listeners.
  ///  * [correctPixels], which is used by the [ScrollPosition] itself to
  ///    set the offset initially during construction or after
  ///    [applyViewportDimension] or [applyContentDimensions] is called.
  @override
  void correctBy(double correction) {
    assert(
      _pixels != null,
      'An initial pixels value must exist by caling correctPixels on the ScrollPosition',
    );
    _pixels += correction;
    _didChangeViewportDimensionOrReceiveCorrection = true;
  }

  /// Change the value of [pixels] to the new value, and notify any customers,
  /// but without honoring normal conventions for changing the scroll offset.
  ///
  /// This is used to implement [jumpTo]. It can also be used adjust the
  /// position when the dimensions of the viewport change. It should only be
  /// used when manually implementing the logic for honoring the relevant
  /// conventions of the class. For example, [ScrollPositionWithSingleContext]
  /// introduces [ScrollActivity] objects and uses [forcePixels] in conjunction
  /// with adjusting the activity, e.g. by calling
  /// [ScrollPositionWithSingleContext.goIdle], so that the activity does
  /// not immediately set the value back. (Consider, for instance, a case where
  /// one is using a [DrivenScrollActivity]. That object will ignore any calls
  /// to [forcePixels], which would result in the rendering stuttering: changing
  /// in response to [forcePixels], and then changing back to the next value
  /// derived from the animation.)
  ///
  /// To cause the position to jump or animate to a new value, consider [jumpTo]
  /// or [animateTo].
  ///
  /// This should not be called during layout (e.g. when setting the initial
  /// scroll offset). Consider [correctPixels] if you find you need to adjust
  /// the position during layout.
  @protected
  void forcePixels(double value) {
    assert(pixels != null);
    _pixels = value;
    notifyListeners();
  }

  /// Called whenever scrolling ends, to store the current scroll offset in a
  /// storage mechanism with a lifetime that matches the app's lifetime.
  ///
  /// The stored value will be used by [restoreScrollOffset] when the
  /// [ScrollPosition] is recreated, in the case of the [Scrollable] being
  /// disposed then recreated in the same session. This might happen, for
  /// instance, if a [ListView] is on one of the pages inside a [TabBarView],
  /// and that page is displayed, then hidden, then displayed again.
  ///
  /// The default implementation writes the [pixels] using the nearest
  /// [PageStorage] found from the [context]'s [ScrollContext.storageContext]
  /// property.
  @protected
  void saveScrollOffset() {
    PageStorage.of(context.storageContext)?.writeState(context.storageContext, pixels);
  }

  /// Called whenever the [ScrollPosition] is created, to restore the scroll
  /// offset if possible.
  ///
  /// The value is stored by [saveScrollOffset] when the scroll position
  /// changes, so that it can be restored in the case of the [Scrollable] being
  /// disposed then recreated in the same session. This might happen, for
  /// instance, if a [ListView] is on one of the pages inside a [TabBarView],
  /// and that page is displayed, then hidden, then displayed again.
  ///
  /// The default implementation reads the value from the nearest [PageStorage]
  /// found from the [context]'s [ScrollContext.storageContext] property, and
  /// sets it using [correctPixels], if [pixels] is still null.
  ///
  /// This method is called from the constructor, so layout has not yet
  /// occurred, and the viewport dimensions aren't yet known when it is called.
  @protected
  void restoreScrollOffset() {
    if (pixels == null) {
      final double value = PageStorage.of(context.storageContext)?.readState(context.storageContext) as double;
      if (value != null)
        correctPixels(value);
    }
  }

  /// Returns the overscroll by applying the boundary conditions.
  ///
  /// If the given value is in bounds, returns 0.0. Otherwise, returns the
  /// amount of value that cannot be applied to [pixels] as a result of the
  /// boundary conditions. If the [physics] allow out-of-bounds scrolling, this
  /// method always returns 0.0.
  ///
  /// The default implementation defers to the [physics] object's
  /// [ScrollPhysics.applyBoundaryConditions].
  @protected
  double applyBoundaryConditions(double value) {
    final double result = physics.applyBoundaryConditions(this, value);
    assert(() {
      final double delta = value - pixels;
      if (result.abs() > delta.abs()) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('${physics.runtimeType}.applyBoundaryConditions returned invalid overscroll value.'),
          ErrorDescription(
            'The method was called to consider a change from $pixels to $value, which is a '
            'delta of ${delta.toStringAsFixed(1)} units. However, it returned an overscroll of '
            '${result.toStringAsFixed(1)} units, which has a greater magnitude than the delta. '
            'The applyBoundaryConditions method is only supposed to reduce the possible range '
            'of movement, not increase it.\n'
            'The scroll extents are $minScrollExtent .. $maxScrollExtent, and the '
            'viewport dimension is $viewportDimension.'
          )
        ]);
      }
      return true;
    }());
    return result;
  }

  bool _didChangeViewportDimensionOrReceiveCorrection = true;

  @override
  bool applyViewportDimension(double viewportDimension) {
    if (_viewportDimension != viewportDimension) {
      _viewportDimension = viewportDimension;
      _didChangeViewportDimensionOrReceiveCorrection = true;
      // If this is called, you can rely on applyContentDimensions being called
      // soon afterwards in the same layout phase. So we put all the logic that
      // relies on both values being computed into applyContentDimensions.
    }
    return true;
  }

  Set<SemanticsAction> _semanticActions;

  /// Called whenever the scroll position or the dimensions of the scroll view
  /// change to schedule an update of the available semantics actions. The
  /// actual update will be performed in the next frame. If non is pending
  /// a frame will be scheduled.
  ///
  /// For example: If the scroll view has been scrolled all the way to the top,
  /// the action to scroll further up needs to be removed as the scroll view
  /// cannot be scrolled in that direction anymore.
  ///
  /// This method is potentially called twice per frame (if scroll position and
  /// scroll view dimensions both change) and therefore shouldn't do anything
  /// expensive.
  void _updateSemanticActions() {
    SemanticsAction forward;
    SemanticsAction backward;
    switch (axis) {
      case Axis.vertical:
        forward = SemanticsAction.scrollUp;
        backward = SemanticsAction.scrollDown;
        break;
      case Axis.horizontal:
        forward = SemanticsAction.scrollLeft;
        backward = SemanticsAction.scrollRight;
        break;
    }

    final Set<SemanticsAction> actions = <SemanticsAction>{};
    if (pixels > minScrollExtent)
      actions.add(backward);
    if (pixels < maxScrollExtent)
      actions.add(forward);

    if (setEquals<SemanticsAction>(actions, _semanticActions))
      return;

    _semanticActions = actions;
    context.setSemanticsActions(_semanticActions);
  }

  @override
  bool applyContentDimensions(double minScrollExtent, double maxScrollExtent) {
    assert(minScrollExtent != null);
    assert(maxScrollExtent != null);
    if (!nearEqual(_minScrollExtent, minScrollExtent, Tolerance.defaultTolerance.distance) ||
        !nearEqual(_maxScrollExtent, maxScrollExtent, Tolerance.defaultTolerance.distance) ||
        _didChangeViewportDimensionOrReceiveCorrection) {
      assert(minScrollExtent != null);
      assert(maxScrollExtent != null);
      assert(minScrollExtent <= maxScrollExtent);
      _minScrollExtent = minScrollExtent;
      _maxScrollExtent = maxScrollExtent;
      _haveDimensions = true;
      applyNewDimensions();
      _didChangeViewportDimensionOrReceiveCorrection = false;
    }
    return true;
  }

  /// Notifies the activity that the dimensions of the underlying viewport or
  /// contents have changed.
  ///
  /// Called after [applyViewportDimension] or [applyContentDimensions] have
  /// changed the [minScrollExtent], the [maxScrollExtent], or the
  /// [viewportDimension]. When this method is called, it should be called
  /// _after_ any corrections are applied to [pixels] using [correctPixels], not
  /// before.
  ///
  /// The default implementation informs the [activity] of the new dimensions by
  /// calling its [ScrollActivity.applyNewDimensions] method.
  ///
  /// See also:
  ///
  ///  * [applyViewportDimension], which is called when new
  ///    viewport dimensions are established.
  ///  * [applyContentDimensions], which is called after new
  ///    viewport dimensions are established, and also if new content dimensions
  ///    are established, and which calls [ScrollPosition.applyNewDimensions].
  @protected
  @mustCallSuper
  void applyNewDimensions() {
    assert(pixels != null);
    activity.applyNewDimensions();
    _updateSemanticActions(); // will potentially request a semantics update.
  }

  /// Animates the position such that the given object is as visible as possible
  /// by just scrolling this position.
  ///
  /// See also:
  ///
  ///  * [ScrollPositionAlignmentPolicy] for the way in which `alignment` is
  ///    applied, and the way the given `object` is aligned.
  Future<void> ensureVisible(
    RenderObject object, {
    double alignment = 0.0,
    Duration duration = Duration.zero,
    Curve curve = Curves.ease,
    ScrollPositionAlignmentPolicy alignmentPolicy = ScrollPositionAlignmentPolicy.explicit,
  }) {
    assert(alignmentPolicy != null);
    assert(object.attached);
    final RenderAbstractViewport viewport = RenderAbstractViewport.of(object);
    assert(viewport != null);

    double target;
    switch (alignmentPolicy) {
      case ScrollPositionAlignmentPolicy.explicit:
        target = viewport.getOffsetToReveal(object, alignment).offset.clamp(minScrollExtent, maxScrollExtent) as double;
        break;
      case ScrollPositionAlignmentPolicy.keepVisibleAtEnd:
        target = viewport.getOffsetToReveal(object, 1.0).offset.clamp(minScrollExtent, maxScrollExtent) as double;
        if (target < pixels) {
          target = pixels;
        }
        break;
      case ScrollPositionAlignmentPolicy.keepVisibleAtStart:
        target = viewport.getOffsetToReveal(object, 0.0).offset.clamp(minScrollExtent, maxScrollExtent) as double;
        if (target > pixels) {
          target = pixels;
        }
        break;
    }

    if (target == pixels)
      return Future<void>.value();

    if (duration == Duration.zero) {
      jumpTo(target);
      return Future<void>.value();
    }

    return animateTo(target, duration: duration, curve: curve);
  }

  /// This notifier's value is true if a scroll is underway and false if the scroll
  /// position is idle.
  ///
  /// Listeners added by stateful widgets should be removed in the widget's
  /// [State.dispose] method.
  final ValueNotifier<bool> isScrollingNotifier = ValueNotifier<bool>(false);

  /// Animates the position from its current value to the given value.
  ///
  /// Any active animation is canceled. If the user is currently scrolling, that
  /// action is canceled.
  ///
  /// The returned [Future] will complete when the animation ends, whether it
  /// completed successfully or whether it was interrupted prematurely.
  ///
  /// An animation will be interrupted whenever the user attempts to scroll
  /// manually, or whenever another activity is started, or whenever the
  /// animation reaches the edge of the viewport and attempts to overscroll. (If
  /// the [ScrollPosition] does not overscroll but instead allows scrolling
  /// beyond the extents, then going beyond the extents will not interrupt the
  /// animation.)
  ///
  /// The animation is indifferent to changes to the viewport or content
  /// dimensions.
  ///
  /// Once the animation has completed, the scroll position will attempt to
  /// begin a ballistic activity in case its value is not stable (for example,
  /// if it is scrolled beyond the extents and in that situation the scroll
  /// position would normally bounce back).
  ///
  /// The duration must not be zero. To jump to a particular value without an
  /// animation, use [jumpTo].
  ///
  /// The animation is typically handled by an [DrivenScrollActivity].
  @override
  Future<void> animateTo(
    double to, {
    @required Duration duration,
    @required Curve curve,
  });

  /// Jumps the scroll position from its current value to the given value,
  /// without animation, and without checking if the new value is in range.
  ///
  /// Any active animation is canceled. If the user is currently scrolling, that
  /// action is canceled.
  ///
  /// If this method changes the scroll position, a sequence of start/update/end
  /// scroll notifications will be dispatched. No overscroll notifications can
  /// be generated by this method.
  @override
  void jumpTo(double value);

  /// Calls [jumpTo] if duration is null or [Duration.zero], otherwise
  /// [animateTo] is called.
  ///
  /// If [clamp] is true (the default) then [to] is adjusted to prevent over or
  /// underscroll.
  ///
  /// If [animateTo] is called then [curve] defaults to [Curves.ease].
  @override
  Future<void> moveTo(
    double to, {
    Duration duration,
    Curve curve,
    bool clamp = true,
  }) {
    assert(to != null);
    assert(clamp != null);

    if (clamp)
      to = to.clamp(minScrollExtent, maxScrollExtent) as double;

    return super.moveTo(to, duration: duration, curve: curve);
  }

  @override
  bool get allowImplicitScrolling => physics.allowImplicitScrolling;

  /// Deprecated. Use [jumpTo] or a custom [ScrollPosition] instead.
  @Deprecated('This will lead to bugs.') // ignore: flutter_deprecation_syntax, https://github.com/flutter/flutter/issues/44609
  void jumpToWithoutSettling(double value);

  /// Stop the current activity and start a [HoldScrollActivity].
  ScrollHoldController hold(VoidCallback holdCancelCallback);

  /// Start a drag activity corresponding to the given [DragStartDetails].
  ///
  /// The `onDragCanceled` argument will be invoked if the drag is ended
  /// prematurely (e.g. from another activity taking over). See
  /// [ScrollDragController.onDragCanceled] for details.
  Drag drag(DragStartDetails details, VoidCallback dragCancelCallback);

  /// The currently operative [ScrollActivity].
  ///
  /// If the scroll position is not performing any more specific activity, the
  /// activity will be an [IdleScrollActivity]. To determine whether the scroll
  /// position is idle, check the [isScrollingNotifier].
  ///
  /// Call [beginActivity] to change the current activity.
  @protected
  @visibleForTesting
  ScrollActivity get activity => _activity;
  ScrollActivity _activity;

  /// Change the current [activity], disposing of the old one and
  /// sending scroll notifications as necessary.
  ///
  /// If the argument is null, this method has no effect. This is convenient for
  /// cases where the new activity is obtained from another method, and that
  /// method might return null, since it means the caller does not have to
  /// explicitly null-check the argument.
  void beginActivity(ScrollActivity newActivity) {
    if (newActivity == null)
      return;
    bool wasScrolling, oldIgnorePointer;
    if (_activity != null) {
      oldIgnorePointer = _activity.shouldIgnorePointer;
      wasScrolling = _activity.isScrolling;
      if (wasScrolling && !newActivity.isScrolling)
        didEndScroll(); // notifies and then saves the scroll offset
      _activity.dispose();
    } else {
      oldIgnorePointer = false;
      wasScrolling = false;
    }
    _activity = newActivity;
    if (oldIgnorePointer != activity.shouldIgnorePointer)
      context.setIgnorePointer(activity.shouldIgnorePointer);
    isScrollingNotifier.value = activity.isScrolling;
    if (!wasScrolling && _activity.isScrolling)
      didStartScroll();
  }


  // NOTIFICATION DISPATCH

  /// Called by [beginActivity] to report when an activity has started.
  void didStartScroll() {
    activity.dispatchScrollStartNotification(copyWith(), context.notificationContext);
  }

  /// Called by [setPixels] to report a change to the [pixels] position.
  void didUpdateScrollPositionBy(double delta) {
    activity.dispatchScrollUpdateNotification(copyWith(), context.notificationContext, delta);
  }

  /// Called by [beginActivity] to report when an activity has ended.
  ///
  /// This also saves the scroll offset using [saveScrollOffset].
  void didEndScroll() {
    activity.dispatchScrollEndNotification(copyWith(), context.notificationContext);
    if (keepScrollOffset)
      saveScrollOffset();
  }

  /// Called by [setPixels] to report overscroll when an attempt is made to
  /// change the [pixels] position. Overscroll is the amount of change that was
  /// not applied to the [pixels] value.
  void didOverscrollBy(double value) {
    assert(activity.isScrolling);
    activity.dispatchOverscrollNotification(copyWith(), context.notificationContext, value);
  }

  /// Dispatches a notification that the [userScrollDirection] has changed.
  ///
  /// Subclasses should call this function when they change [userScrollDirection].
  void didUpdateScrollDirection(ScrollDirection direction) {
    UserScrollNotification(metrics: copyWith(), context: context.notificationContext, direction: direction).dispatch(context.notificationContext);
  }

  @override
  void dispose() {
    activity?.dispose(); // it will be null if it got absorbed by another ScrollPosition
    _activity = null;
    super.dispose();
  }

  @override
  void notifyListeners() {
    _updateSemanticActions(); // will potentially request a semantics update.
    super.notifyListeners();
  }
}

```



## Viewport

```dart
/// A widget that is bigger on the inside.
///
/// [Viewport] is the visual workhorse of the scrolling machinery. It displays a
/// subset of its children according to its own dimensions and the given
/// [offset]. As the offset varies, different children are visible through
/// the viewport.
///
/// [Viewport] hosts a bidirectional list of slivers, anchored on a [center]
/// sliver, which is placed at the zero scroll offset. The center widget is
/// displayed in the viewport according to the [anchor] property.
///
/// Slivers that are earlier in the child list than [center] are displayed in
/// reverse order in the reverse [axisDirection] starting from the [center]. For
/// example, if the [axisDirection] is [AxisDirection.down], the first sliver
/// before [center] is placed above the [center]. The slivers that are later in
/// the child list than [center] are placed in order in the [axisDirection]. For
/// example, in the preceding scenario, the first sliver after [center] is
/// placed below the [center].
///
/// [Viewport] cannot contain box children directly. Instead, use a
/// [SliverList], [SliverFixedExtentList], [SliverGrid], or a
/// [SliverToBoxAdapter], for example.
///
/// See also:
///
///  * [ListView], [PageView], [GridView], and [CustomScrollView], which combine
///    [Scrollable] and [Viewport] into widgets that are easier to use.
///  * [SliverToBoxAdapter], which allows a box widget to be placed inside a
///    sliver context (the opposite of this widget).
///  * [ShrinkWrappingViewport], a variant of [Viewport] that shrink-wraps its
///    contents along the main axis.
class Viewport extends MultiChildRenderObjectWidget {
  /// Creates a widget that is bigger on the inside.
  ///
  /// The viewport listens to the [offset], which means you do not need to
  /// rebuild this widget when the [offset] changes.
  ///
  /// The [offset] argument must not be null.
  Viewport({
    Key key,
    this.axisDirection = AxisDirection.down,
    this.crossAxisDirection,
    this.anchor = 0.0,
    @required this.offset,
    this.center,
    this.cacheExtent,
    this.cacheExtentStyle = CacheExtentStyle.pixel,
    List<Widget> slivers = const <Widget>[],
  }) : assert(offset != null),
       assert(slivers != null),
       assert(center == null || slivers.where((Widget child) => child.key == center).length == 1),
       assert(cacheExtentStyle != null),
       assert(cacheExtentStyle != CacheExtentStyle.viewport || cacheExtent != null),
       super(key: key, children: slivers);

  /// The direction in which the [offset]'s [ViewportOffset.pixels] increases.
  ///
  /// For example, if the [axisDirection] is [AxisDirection.down], a scroll
  /// offset of zero is at the top of the viewport and increases towards the
  /// bottom of the viewport.
  final AxisDirection axisDirection;

  /// The direction in which child should be laid out in the cross axis.
  ///
  /// If the [axisDirection] is [AxisDirection.down] or [AxisDirection.up], this
  /// property defaults to [AxisDirection.left] if the ambient [Directionality]
  /// is [TextDirection.rtl] and [AxisDirection.right] if the ambient
  /// [Directionality] is [TextDirection.ltr].
  ///
  /// If the [axisDirection] is [AxisDirection.left] or [AxisDirection.right],
  /// this property defaults to [AxisDirection.down].
  final AxisDirection crossAxisDirection;

  /// The relative position of the zero scroll offset.
  ///
  /// For example, if [anchor] is 0.5 and the [axisDirection] is
  /// [AxisDirection.down] or [AxisDirection.up], then the zero scroll offset is
  /// vertically centered within the viewport. If the [anchor] is 1.0, and the
  /// [axisDirection] is [AxisDirection.right], then the zero scroll offset is
  /// on the left edge of the viewport.
  final double anchor;

  /// Which part of the content inside the viewport should be visible.
  ///
  /// The [ViewportOffset.pixels] value determines the scroll offset that the
  /// viewport uses to select which part of its content to display. As the user
  /// scrolls the viewport, this value changes, which changes the content that
  /// is displayed.
  ///
  /// Typically a [ScrollPosition].
  final ViewportOffset offset;

  /// The first child in the [GrowthDirection.forward] growth direction.
  ///
  /// Children after [center] will be placed in the [axisDirection] relative to
  /// the [center]. Children before [center] will be placed in the opposite of
  /// the [axisDirection] relative to the [center].
  ///
  /// The [center] must be the key of a child of the viewport.
  final Key center;

  /// {@macro flutter.rendering.viewport.cacheExtent}
  final double cacheExtent;

  /// {@macro flutter.rendering.viewport.cacheExtentStyle}
  final CacheExtentStyle cacheExtentStyle;

  /// Given a [BuildContext] and an [AxisDirection], determine the correct cross
  /// axis direction.
  ///
  /// This depends on the [Directionality] if the `axisDirection` is vertical;
  /// otherwise, the default cross axis direction is downwards.
  static AxisDirection getDefaultCrossAxisDirection(BuildContext context, AxisDirection axisDirection) {
    assert(axisDirection != null);
    switch (axisDirection) {
      case AxisDirection.up:
        return textDirectionToAxisDirection(Directionality.of(context));
      case AxisDirection.right:
        return AxisDirection.down;
      case AxisDirection.down:
        return textDirectionToAxisDirection(Directionality.of(context));
      case AxisDirection.left:
        return AxisDirection.down;
    }
    return null;
  }

  @override
  RenderViewport createRenderObject(BuildContext context) {
    return RenderViewport(
      axisDirection: axisDirection,
      crossAxisDirection: crossAxisDirection ?? Viewport.getDefaultCrossAxisDirection(context, axisDirection),
      anchor: anchor,
      offset: offset,
      cacheExtent: cacheExtent,
      cacheExtentStyle: cacheExtentStyle,
    );
  }

  @override
  void updateRenderObject(BuildContext context, RenderViewport renderObject) {
    renderObject
      ..axisDirection = axisDirection
      ..crossAxisDirection = crossAxisDirection ?? Viewport.getDefaultCrossAxisDirection(context, axisDirection)
      ..anchor = anchor
      ..offset = offset
      ..cacheExtent = cacheExtent
      ..cacheExtentStyle = cacheExtentStyle;
  }

  @override
  _ViewportElement createElement() => _ViewportElement(this);

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(EnumProperty<AxisDirection>('axisDirection', axisDirection));
    properties.add(EnumProperty<AxisDirection>('crossAxisDirection', crossAxisDirection, defaultValue: null));
    properties.add(DoubleProperty('anchor', anchor));
    properties.add(DiagnosticsProperty<ViewportOffset>('offset', offset));
    if (center != null) {
      properties.add(DiagnosticsProperty<Key>('center', center));
    } else if (children.isNotEmpty && children.first.key != null) {
      properties.add(DiagnosticsProperty<Key>('center', children.first.key, tooltip: 'implicit'));
    }
    properties.add(DiagnosticsProperty<double>('cacheExtent', cacheExtent));
    properties.add(DiagnosticsProperty<CacheExtentStyle>('cacheExtentStyle', cacheExtentStyle));
  }
}
```
## _ViewportElement
```dart
class _ViewportElement extends MultiChildRenderObjectElement {
  /// Creates an element that uses the given widget as its configuration.
  _ViewportElement(Viewport widget) : super(widget);

  @override
  Viewport get widget => super.widget as Viewport;

  @override
  RenderViewport get renderObject => super.renderObject as RenderViewport;

  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _updateCenter();
  }

  @override
  void update(MultiChildRenderObjectWidget newWidget) {
    super.update(newWidget);
    _updateCenter();
  }

  void _updateCenter() {
    // TODO(ianh): cache the keys to make this faster
    if (widget.center != null) {
      renderObject.center = children.singleWhere(
        (Element element) => element.widget.key == widget.center
      ).renderObject as RenderSliver;
    } else if (children.isNotEmpty) {
      renderObject.center = children.first.renderObject as RenderSliver;
    } else {
      renderObject.center = null;
    }
  }

  @override
  void debugVisitOnstageChildren(ElementVisitor visitor) {
    children.where((Element e) {
      final RenderSliver renderSliver = e.renderObject as RenderSliver;
      return renderSliver.geometry.visible;
    }).forEach(visitor);
  }
}
```



## RenderAbstactViewport

```dart
/// 一个渲染视窗的接口
///
/// 一些渲染对象，例如 [RenderViewport], 呈现了它们的一部分内容，并且能够被 [ViewportOffset] 控制
/// 这个接口能够让框架识别这些渲染对象
abstract class RenderAbstractViewport extends RenderObject {
  // 这个类应该被作为一个接口
  factory RenderAbstractViewport._() => null;

  /// 返回距离给定 RenderObject 最近的 [RenderAbstractViewport]
  static RenderAbstractViewport of(RenderObject object) {
    while (object != null) {
      if (object is RenderAbstractViewport)
        return object;
      object = object.parent;
    }
    return null;
  }

  /// 返回展示 `target` [RenderObject]，所需要的偏移量
  ///
  /// 可选参数 `rect` 描述了 `target` object 的哪一部分需要在视窗内被展示。
  /// 如果 rect == null，那么整个（即 [RenderObject.paintBounds]）`target` object 将会被展示
  /// 如果 rect 被指定了，那么它必须是 `target` object 坐标系内的 rect
  ///
  /// `alignment`描述了 target 放置的位置
  ///
  /// 这个方法假设视窗的移动是线性的，即，当 scroll offset 变化了 x 的时候，target 也移动了 x 
  RevealedOffset getOffsetToReveal(RenderObject target, double alignment, { Rect rect });

  /// 默认缓存距离
  ///
  @protected
  static const double defaultCacheExtent = 250.0;
}
```



## RenderViewportBase

```dart
/// A base class for render objects that are bigger on the inside.
///
/// This render object provides the shared code for render objects that host
/// [RenderSliver] render objects inside a [RenderBox]. The viewport establishes
/// an [axisDirection], which orients the sliver's coordinate system, which is
/// based on scroll offsets rather than Cartesian coordinates.
///
/// viewport 监听了 [offset], 它决定了 [SliverConstraints.scrollOffset] 
///
/// 子类通常需要重载 [performLayout] 并且调用 [layoutChildSequence]（可能许多次）
/// 注意以下类定义：
/// ParaentDataClass 是泛型，要求提供一个继承（混入）自 ContainerParentDataMixin 的 ParentData
/// 参见[SliverLogicalContainerParentData]
/// 混入了 ContainerRenderObjectMixin，这个混入通常与 ContainerParentDataMixin 搭配使用，两者共同提供了
/// 链表管理兄弟节点的方式
abstract class RenderViewportBase<
    	ParentDataClass 
    	extends ContainerParentDataMixin<RenderSliver>>
    extends RenderBox 
    with ContainerRenderObjectMixin<RenderSliver, ParentDataClass>
    implements RenderAbstractViewport {
  /// Initializes fields for subclasses.
  RenderViewportBase({
    AxisDirection axisDirection = AxisDirection.down,
    @required AxisDirection crossAxisDirection,
    @required ViewportOffset offset,
    double cacheExtent,
  }) : _axisDirection = axisDirection,
       _crossAxisDirection = crossAxisDirection,
       _offset = offset,
       _cacheExtent = cacheExtent ?? RenderAbstractViewport.defaultCacheExtent;

  @override
  void describeSemanticsConfiguration(SemanticsConfiguration config) {
    super.describeSemanticsConfiguration(config);
    config.addTagForChildren(RenderViewport.useTwoPaneSemantics);
  }

  @override
  void visitChildrenForSemantics(RenderObjectVisitor visitor) {
    childrenInPaintOrder
        .where((RenderSliver sliver) => sliver.geometry.visible || sliver.geometry.cacheExtent > 0.0)
        .forEach(visitor);
  }

  /// [SliverConstraints.scrollOffset] 增加的方向，即滑动开始的方向
  AxisDirection get axisDirection => _axisDirection;
  AxisDirection _axisDirection;
  /// 发生改变的时侯需要重新布局
  set axisDirection(AxisDirection value) {
    if (value == _axisDirection)
      return;
    _axisDirection = value;
    markNeedsLayout();
  }

  /// 交叉轴的方向
  AxisDirection get crossAxisDirection => _crossAxisDirection;
  AxisDirection _crossAxisDirection;
  set crossAxisDirection(AxisDirection value) {
    assert(value != null);
    if (value == _crossAxisDirection)
      return;
    _crossAxisDirection = value;
    markNeedsLayout();
  }

  /// 视窗轴方向
  Axis get axis => axisDirectionToAxis(axisDirection);

  /// 视窗内部的哪一部分内容应该可见
  ///
  /// [ViewportOffset.pixels] 决定了 scroll offset，其影响了视窗展示的部分
  /// 当用户滚动视窗的时候，ViewportOffset.pixels 发生了改变，于是改变了正在展示的内容
  ViewportOffset get offset => _offset;
  ViewportOffset _offset;
  set offset(ViewportOffset value) {
    if (value == _offset)
      return;
    /// 当设定 ViewportOffset 时候，向其中加入了 markNeedsLayout，
    /// 即每当其值变化的时候，renderViewport 都会重新布局
    if (attached)
      _offset.removeListener(markNeedsLayout);
    _offset = value;
    if (attached)
      _offset.addListener(markNeedsLayout);
    markNeedsLayout();
  }

  /// 缓存距离。
  double get cacheExtent => _cacheExtent;
  double _cacheExtent;
  set cacheExtent(double value) {
    value = value ?? RenderAbstractViewport.defaultCacheExtent;
    if (value == _cacheExtent)
      return;
    _cacheExtent = value;
    markNeedsLayout();
  }

  /// 在 attach 的时候，会向 viewportOffset 中加入 markNeedsLayout 回调
  /// 这要求 viewportOffset 不为 null
  @override
  void attach(PipelineOwner owner) {
    super.attach(owner);
    _offset.addListener(markNeedsLayout);
  }

  @override
  void detach() {
    _offset.removeListener(markNeedsLayout);
    super.detach();
  }

  /// 计算固有长宽
  @override
  double computeMinIntrinsicWidth(double height) {
    return 0.0;
  }

  @override
  double computeMaxIntrinsicWidth(double height) {
    return 0.0;
  }

  @override
  double computeMinIntrinsicHeight(double width) {
    return 0.0;
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    return 0.0;
  }

  /// 显然，每一个视窗是一个重绘边界
  @override
  bool get isRepaintBoundary => true;

  /// 决定了孩子在视窗内的大小和位置
  ///
  /// 布局从给定的 `child`, 并不断调用 `advance` 回调来布局其他的孩子
  /// 直到 `advance` 返回 null 
  ///
  ///  * `scrollOffset` 是传递给第一个孩子的 [SliverConstraints.scrollOffset]
  ///    scrollOffset 被 [SliverGeometry.scrollExtent] 为接下来的孩子调节.
  ///  * `overlap` is the [SliverConstraints.overlap] to pass the first child.
  ///    The overlay is adjusted by the [SliverGeometry.paintOrigin] and
  ///    [SliverGeometry.paintExtent] for subsequent children.
  ///  * `layoutOffset` 是放置第一个孩子的布局偏移量.
  ///   `layoutOffset` 被 [SliverGeometry.layoutExtent] 为接下来的孩子更新
  ///  
  ///  * `remainingPaintExtent` 是传递给一个孩子的 [SliverConstraints.remainingPaintExtent]
  ///    remainingPaintExtent 被 [SliverGeometry.layoutExtent] 为接下来的孩子更新
  ///  * `mainAxisExtent` is the [SliverConstraints.viewportMainAxisExtent] to
  ///    pass to each child.
  ///  * `crossAxisExtent` is the [SliverConstraints.crossAxisExtent] to pass to
  ///    each child.
  ///  * `growthDirection` is the [SliverConstraints.growthDirection] to pass to
  ///    each child.
  ///
  /// 返回第一个遇到的非零的 [SliverGeometry.scrollOffsetCorrection]，如果有的话，否则返回 0.0。
  /// 当返回值不为 0.0 的时候，布局会重新进行
  @protected
  double layoutChildSequence({
    @required RenderSliver child,
    @required double scrollOffset,
    @required double overlap,
    @required double layoutOffset,
    @required double remainingPaintExtent,
    @required double mainAxisExtent,
    @required double crossAxisExtent,
    @required GrowthDirection growthDirection,
    @required RenderSliver advance(RenderSliver child),
    @required double remainingCacheExtent,
    @required double cacheOrigin,
  }) {
    /// 给定的一个孩子的布局偏移
    final double initialLayoutOffset = layoutOffset;
    
    /// 如果增长发现是 forward 的话，那么 adjustedUserScrollDirection = userScrollDirection
    /// 否则 那么adjustedUserScrollDirection 是 userScrollDirection 的反向
    final ScrollDirection adjustedUserScrollDirection =
        applyGrowthDirectionToScrollDirection(offset.userScrollDirection, growthDirection);
    
    double maxPaintOffset = layoutOffset + overlap;
    /// 这个孩子之前的所有孩子（在主轴上）占用的像素大小
    double precedingScrollExtent = 0.0;

    while (child != null) {
      final double sliverScrollOffset = scrollOffset <= 0.0 ? 0.0 : scrollOffset;
      // If the scrollOffset is too small we adjust the paddedOrigin because it
      // doesn't make sense to ask a sliver for content before its scroll
      // offset.
      final double correctedCacheOrigin = math.max(cacheOrigin, -sliverScrollOffset);
      final double cacheExtentCorrection = cacheOrigin - correctedCacheOrigin;

      /// 用提供的信息构成一个滑块约束
	  final SliverConstraints constraints = SliverConstraints(
        axisDirection: axisDirection,
        growthDirection: growthDirection,
        userScrollDirection: adjustedUserScrollDirection,
        scrollOffset: sliverScrollOffset,
        precedingScrollExtent: precedingScrollExtent,
        /// 很奇怪，maxPaintOffset = overlap + layoutOffset
        overlap: maxPaintOffset - layoutOffset,
        remainingPaintExtent: 
          math.max(0.0, remainingPaintExtent - layoutOffset + initialLayoutOffset),
        crossAxisExtent: crossAxisExtent,
        crossAxisDirection: crossAxisDirection,
        viewportMainAxisExtent: mainAxisExtent,
        remainingCacheExtent: math.max(0.0, remainingCacheExtent + cacheExtentCorrection),
        cacheOrigin: correctedCacheOrigin,
      );
      
      /// 把约束传给当前孩子，命令其布局
      child.layout(constraints, parentUsesSize: true);
	  /// 布局之后孩子会提供一个 geometry（类似于 RenderBox，其布局之后会提供一个 Size）
      final SliverGeometry childLayoutGeometry = child.geometry;

      // 如果不需要任何地修正，那么无需重新布局，否则返回一个修正值
      if (childLayoutGeometry.scrollOffsetCorrection != null)
        return childLayoutGeometry.scrollOffsetCorrection;

      // 我们在我们的坐标系里使用孩子的绘制原点作为其布局偏移，并将储存在其 parent data 里
      final double effectiveLayoutOffset = layoutOffset + childLayoutGeometry.paintOrigin;

      // `effectiveLayoutOffset` becomes meaningless once we moved past the trailing edge
      // because `childLayoutGeometry.layoutExtent` is zero. Using the still increasing
      // 'scrollOffset` to roughly position these invisible slivers in the right order.
      if (childLayoutGeometry.visible || scrollOffset > 0) {
        /// 直接修改孩子的 parentData 中的 paintOffset
        updateChildLayoutOffset(child, effectiveLayoutOffset, growthDirection);
      } else {
        updateChildLayoutOffset(child, -scrollOffset + initialLayoutOffset, growthDirection);
      }

      /// 更新最大绘制距离
      maxPaintOffset = 
          math.max(effectiveLayoutOffset + childLayoutGeometry.paintExtent, maxPaintOffset);
      scrollOffset -= childLayoutGeometry.scrollExtent;
      /// 更新孩子占有距离
      precedingScrollExtent += childLayoutGeometry.scrollExtent;
      /// 更新布局偏移
      layoutOffset += childLayoutGeometry.layoutExtent;
      /// 更新缓存
      if (childLayoutGeometry.cacheExtent != 0.0) {
        remainingCacheExtent -= childLayoutGeometry.cacheExtent - cacheExtentCorrection;
        cacheOrigin = math.min(correctedCacheOrigin + childLayoutGeometry.cacheExtent, 0.0);
      }

      updateOutOfBandData(growthDirection, childLayoutGeometry);

      // 进行下一个孩子的布局
      child = advance(child);
    }

    /// 运行到此处的时候，说明当完成了所有孩子的布局，并且没有需要修正的地方，返回 0.0
    return 0.0;
  }

  @override
  Rect describeApproximatePaintClip(RenderSliver child) {
    final Rect viewportClip = Offset.zero & size;
    if (child.constraints.overlap == 0) {
      return viewportClip;
    }

    // Adjust the clip rect for this sliver by the overlap from the previous sliver.
    double left = viewportClip.left;
    double right = viewportClip.right;
    double top = viewportClip.top;
    double bottom = viewportClip.bottom;
    final double startOfOverlap = 
        child.constraints.viewportMainAxisExtent - child.constraints.remainingPaintExtent;
    final double overlapCorrection = startOfOverlap + child.constraints.overlap;
    switch (applyGrowthDirectionToAxisDirection(axisDirection, child.constraints.growthDirection)) {
      case AxisDirection.down:
        top += overlapCorrection;
        break;
      case AxisDirection.up:
        bottom -= overlapCorrection;
        break;
      case AxisDirection.right:
        left += overlapCorrection;
        break;
      case AxisDirection.left:
        right -= overlapCorrection;
        break;
    }
    return Rect.fromLTRB(left, top, right, bottom);
  }

  @override
  Rect describeSemanticsClip(RenderSliver child) {
    assert (axis != null);
    switch (axis) {
      case Axis.vertical:
        return Rect.fromLTRB(
          semanticBounds.left,
          semanticBounds.top - cacheExtent,
          semanticBounds.right,
          semanticBounds.bottom + cacheExtent,
        );
      case Axis.horizontal:
        return Rect.fromLTRB(
          semanticBounds.left - cacheExtent,
          semanticBounds.top,
          semanticBounds.right + cacheExtent,
          semanticBounds.bottom,
        );
    }
    return null;
  }

  /// 绘制方法：
  /// 如果有视觉溢出，那么对当前视窗进行裁剪
  /// 否则，直接绘制所有的孩子
  @override
  void paint(PaintingContext context, Offset offset) {
    if (firstChild == null)
      return;
    if (hasVisualOverflow) {
      context.pushClipRect(needsCompositing, offset, Offset.zero & size, _paintContents);
    } else {
      _paintContents(context, offset);
    }
  }

  /// 如果孩子 geometry 表示这个孩子是可见的话，那么绘制它
  /// 注意孩子的绘制顺序
  void _paintContents(PaintingContext context, Offset offset) {
    for (RenderSliver child in childrenInPaintOrder) {
      if (child.geometry.visible)
        context.paintChild(child, offset + paintOffsetOf(child));
    }
  }
    
  /// 对于孩子的点击测试
  /// 注意孩子的点击测试的顺序
  @override
  bool hitTestChildren(BoxHitTestResult result, { Offset position }) {
    double mainAxisPosition, crossAxisPosition;
    /// 以下部分要求 postion 不为 null
    switch (axis) {
      case Axis.vertical:
        mainAxisPosition = position.dy;
        crossAxisPosition = position.dx;
        break;
      case Axis.horizontal:
        mainAxisPosition = position.dx;
        crossAxisPosition = position.dy;
        break;
    }

    final SliverHitTestResult sliverResult = SliverHitTestResult.wrap(result);
    for (RenderSliver child in childrenInHitTestOrder) {
      if (!child.geometry.visible) {
        continue;
      }
      final Matrix4 transform = Matrix4.identity();
      applyPaintTransform(child, transform);
      final bool isHit = result.addWithPaintTransform(
        transform: transform,
        position: null, // Manually adapting from box to sliver position below.
        hitTest: (BoxHitTestResult result, Offset _) {
          return child.hitTest(
            sliverResult,
            mainAxisPosition: computeChildMainAxisPosition(child, mainAxisPosition),
            crossAxisPosition: crossAxisPosition,
          );
        },
      );
      if (isHit) {
        return true;
      }
    }
    return false;
  }

  @override
  RevealedOffset getOffsetToReveal(RenderObject target, double alignment, { Rect rect }) {
    double leadingScrollOffset = 0.0;
    double targetMainAxisExtent;
    rect ??= target.paintBounds;

    // Starting at `target` and walking towards the root:
    //  - `child` will be the last object before we reach this viewport, and
    //  - `pivot` will be the last RenderBox before we reach this viewport.
    RenderObject child = target;
    RenderBox pivot;
    bool onlySlivers = target is RenderSliver; // ... between viewport and `target` (`target` included).
    while (child.parent != this) {
      
      if (child is RenderBox) {
        pivot = child;
      }
      if (child.parent is RenderSliver) {
        final RenderSliver parent = child.parent;
        leadingScrollOffset += parent.childScrollOffset(child);
      } else {
        onlySlivers = false;
        leadingScrollOffset = 0.0;
      }
      child = child.parent;
    }

    if (pivot != null) {
      final RenderSliver pivotParent = pivot.parent;

      final Matrix4 transform = target.getTransformTo(pivot);
      final Rect bounds = MatrixUtils.transformRect(transform, rect);

      final GrowthDirection growthDirection = pivotParent.constraints.growthDirection;
      switch (applyGrowthDirectionToAxisDirection(axisDirection, growthDirection)) {
        case AxisDirection.up:
          double offset;
          switch (growthDirection) {
            case GrowthDirection.forward:
              offset = bounds.bottom;
              break;
            case GrowthDirection.reverse:
              offset = bounds.top;
              break;
          }
          leadingScrollOffset += pivot.size.height - offset;
          targetMainAxisExtent = bounds.height;
          break;
        case AxisDirection.right:
          double offset;
          switch (growthDirection) {
            case GrowthDirection.forward:
              offset = bounds.left;
              break;
            case GrowthDirection.reverse:
              offset = bounds.right;
              break;
          }
          leadingScrollOffset += offset;
          targetMainAxisExtent = bounds.width;
          break;
        case AxisDirection.down:
          double offset;
          switch (growthDirection) {
            case GrowthDirection.forward:
              offset = bounds.top;
              break;
            case GrowthDirection.reverse:
              offset = bounds.bottom;
              break;
          }
          leadingScrollOffset += offset;
          targetMainAxisExtent = bounds.height;
          break;
        case AxisDirection.left:
          double offset;
          switch (growthDirection) {
            case GrowthDirection.forward:
              offset = bounds.right;
              break;
            case GrowthDirection.reverse:
              offset = bounds.left;
              break;
          }
          leadingScrollOffset += pivot.size.width - offset;
          targetMainAxisExtent = bounds.width;
          break;
      }
    } else if (onlySlivers) {
      final RenderSliver targetSliver = target;
      targetMainAxisExtent = targetSliver.geometry.scrollExtent;
    } else {
      return RevealedOffset(offset: offset.pixels, rect: rect);
    }

    assert(child.parent == this);
    assert(child is RenderSliver);
    final RenderSliver sliver = child;
    final double extentOfPinnedSlivers = maxScrollObstructionExtentBefore(sliver);
    leadingScrollOffset = scrollOffsetOf(sliver, leadingScrollOffset);
    switch (sliver.constraints.growthDirection) {
      case GrowthDirection.forward:
        leadingScrollOffset -= extentOfPinnedSlivers;
        break;
      case GrowthDirection.reverse:
        // Nothing to do.
        break;
    }

    double mainAxisExtent;
    switch (axis) {
      case Axis.horizontal:
        mainAxisExtent = size.width - extentOfPinnedSlivers;
        break;
      case Axis.vertical:
        mainAxisExtent = size.height - extentOfPinnedSlivers;
        break;
    }

    final double targetOffset = 
        leadingScrollOffset - (mainAxisExtent - targetMainAxisExtent) * alignment;
    final double offsetDifference = offset.pixels - targetOffset;

    final Matrix4 transform = target.getTransformTo(this);
    applyPaintTransform(child, transform);
    Rect targetRect = MatrixUtils.transformRect(transform, rect);

    switch (axisDirection) {
      case AxisDirection.down:
        targetRect = targetRect.translate(0.0, offsetDifference);
        break;
      case AxisDirection.right:
        targetRect = targetRect.translate(offsetDifference, 0.0);
        break;
      case AxisDirection.up:
        targetRect = targetRect.translate(0.0, -offsetDifference);
        break;
      case AxisDirection.left:
        targetRect = targetRect.translate(-offsetDifference, 0.0);
        break;
    }

    return RevealedOffset(offset: targetOffset, rect: targetRect);
  }

  /// 给定的 `child` 应该在何处被绘制
  ///
  /// 返回的偏移量是从视窗内部左上角到给定孩子的坐标系的左上角（即坐标原点）的偏移
  ///
  /// 这个方法被 [paintOffsetOf] 在布局时为孩子计算偏移使用
  @protected
  Offset computeAbsolutePaintOffset(
      RenderSliver child, 
      double layoutOffset, 
      GrowthDirection growthDirection
  ) {
    switch (applyGrowthDirectionToAxisDirection(axisDirection, growthDirection)) {
      /// ？？ 当 AxisDirection == up 的时候，似乎是以视窗底部为布局原点
      /// 也就是说，返回的偏移是相对与底部的
      case AxisDirection.up:
        return Offset(0.0, size.height - (layoutOffset + child.geometry.paintExtent));
      case AxisDirection.right:
        return Offset(layoutOffset, 0.0);
      /// 否则，直接返回给定 layoutOffset 作为偏移 
      case AxisDirection.down:
        return Offset(0.0, layoutOffset);
      case AxisDirection.left:
        return Offset(size.width - (layoutOffset + child.geometry.paintExtent), 0.0);
    }
    return null;
  }

  // 以下是子类应该实现的 API

  // setupParentData

  // performLayout (and optionally sizedByParent and performResize)

  /// 是否有视觉溢出（即是否可以绘制在视窗的外部）
  @protected
  bool get hasVisualOverflow;

  /// Called during [layoutChildSequence] for each child.
  ///
  /// 通常被用于更新外部的数据,例如最大滚动距离
  @protected
  void updateOutOfBandData(GrowthDirection growthDirection, SliverGeometry childLayoutGeometry);

  /// Called during [layoutChildSequence] to store the layout offset for the
  /// given child.
  ///
  /// Different subclasses using different representations for their children's
  /// layout offset (e.g., logical or physical coordinates). This function lets
  /// subclasses transform the child's layout offset before storing it in the
  /// child's parent data.
  @protected
  void updateChildLayoutOffset(
      RenderSliver child, 
      double layoutOffset, 
      GrowthDirection growthDirection
  );

  /// The offset at which the given `child` should be painted.
  ///
  /// The returned offset is from the top left corner of the inside of the
  /// viewport to the top left corner of the paint coordinate system of the
  /// `child`.
  ///
  /// See also [computeAbsolutePaintOffset], which computes the paint offset
  /// from an explicit layout offset and growth direction instead of using the
  /// values computed for the child during layout.
  @protected
  Offset paintOffsetOf(RenderSliver child);

  /// Returns the scroll offset within the viewport for the given
  /// `scrollOffsetWithinChild` within the given `child`.
  ///
  /// The returned value is an estimate that assumes the slivers within the
  /// viewport do not change the layout extent in response to changes in their
  /// scroll offset.
  @protected
  double scrollOffsetOf(RenderSliver child, double scrollOffsetWithinChild);

  /// Returns the total scroll obstruction extent of all slivers in the viewport
  /// before [child].
  ///
  /// This is the extent by which the actual area in which content can scroll
  /// is reduced. For example, an app bar that is pinned at the top will reduce
  /// the area in which content can actually scroll by the height of the app bar.
  @protected
  double maxScrollObstructionExtentBefore(RenderSliver child);

  /// Converts the `parentMainAxisPosition` into the child's coordinate system.
  ///
  /// The `parentMainAxisPosition` is a distance from the top edge (for vertical
  /// viewports) or left edge (for horizontal viewports) of the viewport bounds.
  /// This describes a line, perpendicular to the viewport's main axis, heretofor
  /// known as the target line.
  ///
  /// The child's coordinate system's origin in the main axis is at the leading
  /// edge of the given child, as given by the child's
  /// [SliverConstraints.axisDirection] and [SliverConstraints.growthDirection].
  ///
  /// This method returns the distance from the leading edge of the given child to
  /// the target line described above.
  ///
  /// (The `parentMainAxisPosition` is not from the leading edge of the
  /// viewport, it's always the top or left edge.)
  @protected
  double computeChildMainAxisPosition(RenderSliver child, double parentMainAxisPosition);

  /// 第一个孩子的下标
  @protected
  int get indexOfFirstChild;

  @protected
  String labelForChild(int index);

  /// 孩子的绘制顺序
  /// 与 [childrenInHitTestOrder] 的顺序是相反的
  @protected
  Iterable<RenderSliver> get childrenInPaintOrder;

  /// 孩子的点击测试顺序
  /// 与 [childrenInPaintOrder] 的顺序是相反的
  /// 注意：顺序问题
  /// 考虑如下一个列表：
  /// 孩子是层叠的，当用户向上滑动列表的时候，上层列表被移出视窗，从而展示出下层孩子
  /// 此时，孩子的绘制顺序应该是从最后一个孩子（即最底层的孩子）开始，向前（向高层）绘制
  /// 点击测试的顺序应该是从第一个孩子（即最上层的孩子）开始，向后（向底层）绘制
  @protected
  Iterable<RenderSliver> get childrenInHitTestOrder;

  /// 把给定的孩子展示在屏幕上
  @override
  void showOnScreen({
    RenderObject descendant,
    Rect rect,
    Duration duration = Duration.zero,
    Curve curve = Curves.ease,
  }) {
    if (!offset.allowImplicitScrolling) {
      return super.showOnScreen(
        descendant: descendant,
        rect: rect,
        duration: duration,
        curve: curve,
      );
    }

    final Rect newRect = RenderViewportBase.showInViewport(
      descendant: descendant,
      viewport: this,
      offset: offset,
      rect: rect,
      duration: duration,
      curve: curve,
    );
    super.showOnScreen(
      rect: newRect,
      duration: duration,
      curve: curve,
    );
  }

  /// Make (a portion of) the given `descendant` of the given `viewport` fully
  /// visible in the `viewport` by manipulating the provided [ViewportOffset]
  /// `offset`.
  ///
  /// The optional `rect` parameter describes which area of the `descendant`
  /// should be shown in the viewport. If `rect` is null, the entire
  /// `descendant` will be revealed. The `rect` parameter is interpreted
  /// relative to the coordinate system of `descendant`.
  ///
  /// The returned [Rect] describes the new location of `descendant` or `rect`
  /// in the viewport after it has been revealed. See [RevealedOffset.rect]
  /// for a full definition of this [Rect].
  ///
  /// The parameters `viewport` and `offset` are required and cannot be null.
  /// If `descendant` is null, this is a no-op and `rect` is returned.
  ///
  /// If both `descendant` and `rect` are null, null is returned because there is
  /// nothing to be shown in the viewport.
  ///
  /// The `duration` parameter can be set to a non-zero value to animate the
  /// target object into the viewport with an animation defined by `curve`.
  static Rect showInViewport({
    RenderObject descendant,
    Rect rect,
    @required RenderAbstractViewport viewport,
    @required ViewportOffset offset,
    Duration duration = Duration.zero,
    Curve curve = Curves.ease,
  }) {
    assert(viewport != null);
    assert(offset != null);
    if (descendant == null) {
      return rect;
    }
    final RevealedOffset leadingEdgeOffset = 
        viewport.getOffsetToReveal(descendant, 0.0, rect: rect);
    final RevealedOffset trailingEdgeOffset = 
        viewport.getOffsetToReveal(descendant, 1.0, rect: rect);
    final double currentOffset = offset.pixels;

    //           scrollOffset
    //                       0 +---------+
    //                         |         |
    //                       _ |         |
    //    viewport position |  |         |
    // with `descendant` at |  |         | _
    //        trailing edge |_ | xxxxxxx |  | viewport position
    //                         |         |  | with `descendant` at
    //                         |         | _| leading edge
    //                         |         |
    //                     800 +---------+
    //
    // `trailingEdgeOffset`: Distance from scrollOffset 0 to the start of the
    //                       viewport on the left in image above.
    // `leadingEdgeOffset`: Distance from scrollOffset 0 to the start of the
    //                      viewport on the right in image above.
    //
    // The viewport position on the left is achieved by setting `offset.pixels`
    // to `trailingEdgeOffset`, the one on the right by setting it to
    // `leadingEdgeOffset`.

    RevealedOffset targetOffset;
    if (leadingEdgeOffset.offset < trailingEdgeOffset.offset) {
      // `descendant` is too big to be visible on screen in its entirety. Let's
      // align it with the edge that requires the least amount of scrolling.
      final double leadingEdgeDiff = (offset.pixels - leadingEdgeOffset.offset).abs();
      final double trailingEdgeDiff = (offset.pixels - trailingEdgeOffset.offset).abs();
      targetOffset = leadingEdgeDiff < trailingEdgeDiff ? leadingEdgeOffset : trailingEdgeOffset;
    } else if (currentOffset > leadingEdgeOffset.offset) {
      // `descendant` currently starts above the leading edge and can be shown
      // fully on screen by scrolling down (which means: moving viewport up).
      targetOffset = leadingEdgeOffset;
    } else if (currentOffset < trailingEdgeOffset.offset) {
      // `descendant currently ends below the trailing edge and can be shown
      // fully on screen by scrolling up (which means: moving viewport down)
      targetOffset = trailingEdgeOffset;
    } else {
      // `descendant` is between leading and trailing edge and hence already
      //  fully shown on screen. No action necessary.
      final Matrix4 transform = descendant.getTransformTo(viewport.parent);
      return MatrixUtils.transformRect(transform, rect ?? descendant.paintBounds);
    }

    assert(targetOffset != null);

    offset.moveTo(targetOffset.offset, duration: duration, curve: curve);
    return targetOffset.rect;
  }
}
```



## RenderViewport

```dart
/// [RenderViewport] 是滚动机器的视觉加工厂. 他根据自身维度和给定的 [offset] 来决定需要展示视窗内容的哪一部
/// 分，当 offset 变化的时候，其展示的内容也会发生改变
///
/// [RenderViewport] 持有两个方向的滑块列表,锚定在一个 [center] 滑块的位置， [center] 被放置在滚动偏移零点
///
/// 在 center 之前的滑块列表将以相反的顺序展示，之后的列表将以正序展示（这里的展示是绘制的意思）
///
/// [RenderViewport] 不能直接包含 [RenderBox] 孩子。相反，应该 使用 [RenderSliverList], 
/// 或者 [RenderSliverFixedExtentList] 或者 [RenderSliverGrid] 或者
/// [RenderSliverToBoxAdapter]
class RenderViewport extends RenderViewportBase<SliverPhysicalContainerParentData> {
  RenderViewport({
    AxisDirection axisDirection = AxisDirection.down,
    @required AxisDirection crossAxisDirection,
    @required ViewportOffset offset,
    double anchor = 0.0,
    List<RenderSliver> children,
    RenderSliver center,
    double cacheExtent,
  }) : assert(anchor != null),
       assert(anchor >= 0.0 && anchor <= 1.0),
       _anchor = anchor,
       _center = center,
       super(
           axisDirection: axisDirection, 
           crossAxisDirection: crossAxisDirection, 
           offset: offset, 
           cacheExtent: cacheExtent
       ) {
    /// 这个渲染对象本身是持有一些孩子的（即构建组件时给定的 children，即一些 renderSliver）
    /// 如果给定一些新的孩子，会将其附加在已经存在的孩子列表（通过 RenderObjectContainerParentData 混入类来
    /// 管理的链表）的尾端
    addAll(children);
    if (center == null && firstChild != null)
      _center = firstChild;
  }

  static const SemanticsTag useTwoPaneSemantics = SemanticsTag('RenderViewport.twoPane');
  static const SemanticsTag excludeFromScrolling = 
      SemanticsTag('RenderViewport.excludeFromScrolling');

  /// 为孩子设置 SliverPhysicalContainerParentData 
  @override
  void setupParentData(RenderObject child) {
    if (child.parentData is! SliverPhysicalContainerParentData)
      child.parentData = SliverPhysicalContainerParentData();
  }

  /// 锚点
  double get anchor => _anchor;
  double _anchor;
  set anchor(double value) {
    if (value == _anchor)
      return;
    _anchor = value;
    markNeedsLayout();
  }

  /// 正序孩子列表的第一个孩子
  ///
  /// 当 [offset.pixels] 是 0.0 的时候，center 渲染对象将会处于滚动偏移零点
  RenderSliver get center => _center;
  RenderSliver _center;
  set center(RenderSliver value) {
    if (value == _center)
      return;
    _center = value;
    markNeedsLayout();
  }

  /// 一个视窗的大小总是由父级给定的约束决定的
  @override
  bool get sizedByParent => true;

  @override
  void performResize() {
    size = constraints.biggest;
    /// 总是使用约束的最大值来作为大小
    /// 每次 size 的大小改变之后，回调 offset.applyViewportDimension，应用新的视窗维度
    switch (axis) {
      case Axis.vertical:
        offset.applyViewportDimension(size.height);
        break;
      case Axis.horizontal:
        offset.applyViewportDimension(size.width);
        break;
    }
  }

  /// 最大布局次数，即最大修正布局的次数
  /// 超出此次数会抛出异常
  static const int _maxLayoutCycles = 10;

  // Out-of-band data computed during layout.
  double _minScrollExtent;
  double _maxScrollExtent;
  bool _hasVisualOverflow = false;

  @override
  void performLayout() {
    /// 如果 center == null ，将这个视窗看作一个空视窗，其大小为 0.0
    if (center == null) {
      _minScrollExtent = 0.0;
      _maxScrollExtent = 0.0;
      _hasVisualOverflow = false;
      offset.applyContentDimensions(0.0, 0.0);
      return;
    }
      
	/// 主轴和交叉轴的长度
    double mainAxisExtent;
    double crossAxisExtent;
    switch (axis) {
      case Axis.vertical:
        mainAxisExtent = size.height;
        crossAxisExtent = size.width;
        break;
      case Axis.horizontal:
        mainAxisExtent = size.width;
        crossAxisExtent = size.height;
        break;
    }

    final double centerOffsetAdjustment = center.centerOffsetAdjustment;

    double correction;
    int count = 0;
    do {
      correction = 
          /// 尝试一次布局
          // 其中（当视窗轴是垂直的时候）：
          // mainAxisExtent 是视窗的高度，crossAxisExtent 是视窗的宽度
          // correctedOffset = offset.pixels + centerOffsetAdjustment 是更正后的中心偏移量
          _attemptLayout(mainAxisExtent, crossAxisExtent, offset.pixels + centerOffsetAdjustment);
      /// 如果返回了一个修正值，那么重新布局，并且用给定的修正来更正 offset
      if (correction != 0.0) {
        offset.correctBy(correction);
      /// 否则，应用新的内容维度
      } else {
        if (offset.applyContentDimensions(
              math.min(0.0, _minScrollExtent + mainAxisExtent * anchor),
              math.max(0.0, _maxScrollExtent - mainAxisExtent * (1.0 - anchor)),
           ))
          break;
      }
      count += 1;
    } while (count < _maxLayoutCycles);
  }

  double _attemptLayout(double mainAxisExtent, double crossAxisExtent, double correctedOffset) {
    _minScrollExtent = 0.0;
    _maxScrollExtent = 0.0;
    _hasVisualOverflow = false;

    /// centerOffset 是从视窗顶点到滚动偏移零点(划分正序滑块和逆序滑块的线)的偏移量
    /// 其中 correctedOffset = offset.pixels + centerOffsetAdjustment
    // 注意：
    // 在 RenderSliver 中对 ceterOffsetAdjustment 的描述如下：
    // “例如，如果一个满足 [AxisDirection.down] 和 [RenderViewport.anchor = 0.5] 的视窗，其唯一一个滑块的
    // 高度为 100.0，其 [centerOffsetAdjustment] 是 50.0，当滚动偏移量为 0.0 时，滑块将位于视窗的中心”
    // 在上例中，我们假设 [centerOffsetAdjustment] = 0.0，那么当滚动偏移量为 0.0 时，滑块将位于视窗距离顶端
    // 50（即 mainAxisExtent * anchor，也就是说，center 的布局其实是将其顶端放置在滚动偏移零点的位置）的位
    // 置，这说明，centerOffsetAdjustment 是用来调整 center 的位置的
    // 考虑 correctedOffset 的计算
    // 即 centerOffset = mainAxisExtent * anchor - offset.pixels - centerOffsetAdjustment：
    // 显然 mainAxisExtent * anchor - offset.pixels 的值即是滚动偏移零点的位置，（即分割线是可以移动的）
    // 即 center 的顶端的位置，再减去 centerOffsetAdjustment ，即是 center 的被调整之后位置
    // （在例中，若 滚动偏移量为 50.0 时，滑块将处于视窗顶端） 
    final double centerOffset = mainAxisExtent * anchor - correctedOffset;
    // 逆序剩余绘制长度，从 0.0 到 mainAxisExtent 之间的一个值，即在当前视窗内部，center（线）之前还有多少距
    // 离可以用来绘制
    final double reverseDirectionRemainingPaintExtent = centerOffset.clamp(0.0, mainAxisExtent);
    // 正序剩余绘制长度，从 0.0 到 mainAxisExtent 之间的一个值，即在当前视窗内部，center（线）之后还有多少距
    // 离可以用来绘制
    final double forwardDirectionRemainingPaintExtent = 
        (mainAxisExtent - centerOffset).clamp(0.0, mainAxisExtent);
	// 最大缓存距离，即 mainAxisExtent 加上两端缓存的距离
    final double fullCacheExtent = mainAxisExtent + 2 * cacheExtent;
    // 中心缓存偏移
    final double centerCacheOffset = centerOffset + cacheExtent;
    // 逆序缓存长度
    final double reverseDirectionRemainingCacheExtent = 
        centerCacheOffset.clamp(0.0, fullCacheExtent);
    // 正序缓存长度
    final double forwardDirectionRemainingCacheExtent = 
        (fullCacheExtent - centerCacheOffset).clamp(0.0, fullCacheExtent);

    // 第一个逆序的孩子，即 center 之前的第一个孩子
    final RenderSliver leadingNegativeChild = childBefore(center);
    // 首先对逆序的孩子列表进行布局
    if (leadingNegativeChild != null) {
      // 以下给定的参数都是对于第一个孩子设置的
      // 在连续的布局进程中，给定的值会不断地进行更新
      final double result = layoutChildSequence(
        /// 逆序列表的第一个孩子
        child: leadingNegativeChild,
        scrollOffset: math.max(mainAxisExtent, centerOffset) - mainAxisExtent,
        overlap: 0.0,
        layoutOffset: forwardDirectionRemainingPaintExtent,
        remainingPaintExtent: reverseDirectionRemainingPaintExtent,
        mainAxisExtent: mainAxisExtent,
        crossAxisExtent: crossAxisExtent,
        growthDirection: GrowthDirection.reverse,
        advance: childBefore,
        remainingCacheExtent: reverseDirectionRemainingCacheExtent,
        cacheOrigin: (mainAxisExtent - centerOffset).clamp(-cacheExtent, 0.0),
      );
      if (result != 0.0)
        return -result;
    }

    // 对正向孩子进行布局
    return layoutChildSequence(
      child: center,
      /// 显然，-centerOffset 就是 [center] 的 scrolloffset
      scrollOffset: math.max(0.0, -centerOffset),
      overlap: leadingNegativeChild == null ? math.min(0.0, -centerOffset) : 0.0,
      /// 如果 centerOffset >= mainAxisExtent，这意味着 center 可能在视窗的外部（例如，当 axisDirection = 
      /// down 的时候在视窗底部以下），这时即采用 centerOffset 作为布局起点
      layoutOffset: centerOffset >= mainAxisExtent 
        ? centerOffset
        : reverseDirectionRemainingPaintExtent,
      /// 还未绘制第一个孩子时剩余的绘制距离
      remainingPaintExtent: forwardDirectionRemainingPaintExtent,
      mainAxisExtent: mainAxisExtent,
      crossAxisExtent: crossAxisExtent,
      growthDirection: GrowthDirection.forward,
      advance: childAfter,
      remainingCacheExtent: forwardDirectionRemainingCacheExtent,
      cacheOrigin: centerOffset.clamp(-cacheExtent, 0.0),
    );
  }

  @override
  bool get hasVisualOverflow => _hasVisualOverflow;
 
  /// 根据增长方向来计算最大最小 scrollExtent
  @override
  void updateOutOfBandData(GrowthDirection growthDirection, SliverGeometry childLayoutGeometry) {
    switch (growthDirection) {
      case GrowthDirection.forward:
        _maxScrollExtent += childLayoutGeometry.scrollExtent;
        break;
      case GrowthDirection.reverse:
        _minScrollExtent -= childLayoutGeometry.scrollExtent;
        break;
    }
    /// 如果给定的 childLayoutGeometry 提示有视觉溢出，那么应该使用裁剪
    if (childLayoutGeometry.hasVisualOverflow)
      _hasVisualOverflow = true;
  }

  /// 使用给定的 layoutOffset 更新给定孩子的偏移
  @override
  void updateChildLayoutOffset(
      RenderSliver child, 
      double layoutOffset, 
      GrowthDirection growthDirection
  ) {
    final SliverPhysicalParentData childParentData = child.parentData;
    childParentData.paintOffset = computeAbsolutePaintOffset(child, layoutOffset, growthDirection);
  }

  /// 给定孩子的绘制偏移
  @override
  Offset paintOffsetOf(RenderSliver child) {
    final SliverPhysicalParentData childParentData = child.parentData;
    return childParentData.paintOffset;
  }

  /// 给定孩子的滚动偏移
  @override
  double scrollOffsetOf(RenderSliver child, double scrollOffsetWithinChild) {
    assert(child.parent == this);
    final GrowthDirection growthDirection = child.constraints.growthDirection;
    switch (growthDirection) {
      case GrowthDirection.forward:
        double scrollOffsetToChild = 0.0;
        RenderSliver current = center;
        while (current != child) {
          scrollOffsetToChild += current.geometry.scrollExtent;
          current = childAfter(current);
        }
        return scrollOffsetToChild + scrollOffsetWithinChild;
      case GrowthDirection.reverse:
        double scrollOffsetToChild = 0.0;
        RenderSliver current = childBefore(center);
        while (current != child) {
          scrollOffsetToChild -= current.geometry.scrollExtent;
          current = childBefore(current);
        }
        return scrollOffsetToChild - scrollOffsetWithinChild;
    }
    return null;
  }

  @override
  double maxScrollObstructionExtentBefore(RenderSliver child) {
    assert(child.parent == this);
    final GrowthDirection growthDirection = child.constraints.growthDirection;
    assert(growthDirection != null);
    switch (growthDirection) {
      case GrowthDirection.forward:
        double pinnedExtent = 0.0;
        RenderSliver current = center;
        while (current != child) {
          pinnedExtent += current.geometry.maxScrollObstructionExtent;
          current = childAfter(current);
        }
        return pinnedExtent;
      case GrowthDirection.reverse:
        double pinnedExtent = 0.0;
        RenderSliver current = childBefore(center);
        while (current != child) {
          pinnedExtent += current.geometry.maxScrollObstructionExtent;
          current = childBefore(current);
        }
        return pinnedExtent;
    }
    return null;
  }

  /// 对给定孩子使用给定的绘制矩阵变化（例如旋转，平移）
  @override
  void applyPaintTransform(RenderObject child, Matrix4 transform) {
    assert(child != null);
    final SliverPhysicalParentData childParentData = child.parentData;
    childParentData.applyPaintTransform(transform);
  }

  /// 计算给定的孩子在主轴上的偏移）
  @override
  double computeChildMainAxisPosition(RenderSliver child, double parentMainAxisPosition) {
    final SliverPhysicalParentData childParentData = child.parentData;
    switch (applyGrowthDirectionToAxisDirection(
             child.constraints.axisDirection, 
        	child.constraints.growthDirection
    )) {
      case AxisDirection.down:
        return parentMainAxisPosition - childParentData.paintOffset.dy;
      case AxisDirection.right:
        return parentMainAxisPosition - childParentData.paintOffset.dx;
      case AxisDirection.up:
        return child.geometry.paintExtent 
            - (parentMainAxisPosition - childParentData.paintOffset.dy);
      case AxisDirection.left:
        return child.geometry.paintExtent 
            - (parentMainAxisPosition - childParentData.paintOffset.dx);
    }
    return 0.0;
  }

  /// 给定孩子的下标
  @override
  int get indexOfFirstChild {
    int count = 0;
    RenderSliver child = center;
    while (child != firstChild) {
      count -= 1;
      child = childBefore(child);
    }
    return count;
  }

  @override
  String labelForChild(int index) {
    if (index == 0)
      return 'center child';
    return 'child $index';
  }

  /// 孩子的绘制顺序
  @override
  Iterable<RenderSliver> get childrenInPaintOrder sync* {
    if (firstChild == null)
      return;
    RenderSliver child = firstChild;
    while (child != center) {
      yield child;
      child = childAfter(child);
    }
    child = lastChild;
    while (true) {
      yield child;
      if (child == center)
        return;
      child = childBefore(child);
    }
  }

  /// 孩子的点击测试顺序
  @override
  Iterable<RenderSliver> get childrenInHitTestOrder sync* {
    if (firstChild == null)
      return;
    RenderSliver child = center;
    while (child != null) {
      yield child;
      child = childAfter(child);
    }
    child = childBefore(center);
    while (child != null) {
      yield child;
      child = childBefore(child);
    }
  }
}
```


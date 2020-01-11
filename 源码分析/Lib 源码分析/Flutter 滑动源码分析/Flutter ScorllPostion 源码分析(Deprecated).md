# Flutter ScorllPostion 源码分析

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

## 
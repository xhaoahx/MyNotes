# Flutter SliverPersistentHeader 源码分析

## SliverPersistentHeader

```dart
class SliverPersistentHeader extends StatelessWidget {
  /// Creates a sliver that varies its size when it is scrolled to the start of
  /// a viewport.
  ///
  /// The [delegate], [pinned], and [floating] arguments must not be null.
  const SliverPersistentHeader({
    Key key,
    @required this.delegate,
    this.pinned = false,
    this.floating = false,
  }) : assert(delegate != null),
       assert(pinned != null),
       assert(floating != null),
       super(key: key);

  /// Configuration for the sliver's layout.
  ///
  /// The delegate provides the following information:
  ///
  ///  * The minimum and maximum dimensions of the sliver.
  ///
  ///  * The builder for generating the widgets of the sliver.
  ///
  ///  * The instructions for snapping the scroll offset, if [floating] is true.
  final SliverPersistentHeaderDelegate delegate;

  /// Whether to stick the header to the start of the viewport once it has
  /// reached its minimum size.
  ///
  /// If this is false, the header will continue scrolling off the screen after
  /// it has shrunk to its minimum extent.
  final bool pinned;

  /// Whether the header should immediately grow again if the user reverses
  /// scroll direction.
  ///
  /// If this is false, the header only grows again once the user reaches the
  /// part of the viewport that contains the sliver.
  ///
  /// The [delegate]'s [SliverPersistentHeaderDelegate.snapConfiguration] is
  /// ignored unless [floating] is true.
  final bool floating;

  @override
  Widget build(BuildContext context) {
    if (floating && pinned)
      return _SliverFloatingPinnedPersistentHeader(delegate: delegate);
    if (pinned)
      return _SliverPinnedPersistentHeader(delegate: delegate);
    if (floating)
      return _SliverFloatingPersistentHeader(delegate: delegate);
    return _SliverScrollingPersistentHeader(delegate: delegate);
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(
      DiagnosticsProperty<SliverPersistentHeaderDelegate>(
        'delegate',
        delegate,
      )
    );
    final List<String> flags = <String>[
      if (pinned) 'pinned',
      if (floating) 'floating',
    ];
    if (flags.isEmpty)
      flags.add('normal');
    properties.add(IterableProperty<String>('mode', flags));
  }
}
```

## _SliverPersistentHeaderElement

```dart
class _SliverPersistentHeaderElement extends RenderObjectElement {
  _SliverPersistentHeaderElement(_SliverPersistentHeaderRenderObjectWidget widget) : super(widget);

  @override
  _SliverPersistentHeaderRenderObjectWidget get widget => super.widget as _SliverPersistentHeaderRenderObjectWidget;

  @override
  _RenderSliverPersistentHeaderForWidgetsMixin get renderObject => super.renderObject as _RenderSliverPersistentHeaderForWidgetsMixin;

  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    renderObject._element = this;
  }

  @override
  void unmount() {
    super.unmount();
    renderObject._element = null;
  }

  @override
  void update(_SliverPersistentHeaderRenderObjectWidget newWidget) {
    final _SliverPersistentHeaderRenderObjectWidget oldWidget = widget;
    super.update(newWidget);
    final SliverPersistentHeaderDelegate newDelegate = newWidget.delegate;
    final SliverPersistentHeaderDelegate oldDelegate = oldWidget.delegate;
    if (newDelegate != oldDelegate &&
        (newDelegate.runtimeType != oldDelegate.runtimeType || newDelegate.shouldRebuild(oldDelegate)))
      renderObject.triggerRebuild();
  }

  @override
  void performRebuild() {
    super.performRebuild();
    renderObject.triggerRebuild();
  }

  Element child;

  void _build(double shrinkOffset, bool overlapsContent) {
    owner.buildScope(this, () {
      child = updateChild(
        child,
        widget.delegate.build(
          this,
          shrinkOffset,
          overlapsContent
        ),
        null,
      );
    });
  }

  @override
  void forgetChild(Element child) {
    assert(child == this.child);
    this.child = null;
  }

  @override
  void insertChildRenderObject(covariant RenderBox child, dynamic slot) {
    assert(renderObject.debugValidateChild(child));
    renderObject.child = child;
  }

  @override
  void moveChildRenderObject(covariant RenderObject child, dynamic slot) {
    assert(false);
  }

  @override
  void removeChildRenderObject(covariant RenderObject child) {
    renderObject.child = null;
  }

  @override
  void visitChildren(ElementVisitor visitor) {
    if (child != null)
      visitor(child);
  }
}
```



## _SliverPersistentHeaderRenderObjectWidget

```dart
abstract class _SliverPersistentHeaderRenderObjectWidget extends RenderObjectWidget {
  const _SliverPersistentHeaderRenderObjectWidget({
    Key key,
    @required this.delegate,
  }) : assert(delegate != null),
       super(key: key);

  final SliverPersistentHeaderDelegate delegate;

  @override
  _SliverPersistentHeaderElement createElement() => _SliverPersistentHeaderElement(this);

  @override
  _RenderSliverPersistentHeaderForWidgetsMixin createRenderObject(BuildContext context);

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder description) {
    super.debugFillProperties(description);
    description.add(
      DiagnosticsProperty<SliverPersistentHeaderDelegate>(
        'delegate',
        delegate,
      )
    );
  }
}
```

### _RenderSliverPersistentHeaderForWidgetsMixin

```dart
mixin _RenderSliverPersistentHeaderForWidgetsMixin on RenderSliverPersistentHeader {
  _SliverPersistentHeaderElement _element;

  @override
  double get minExtent => _element.widget.delegate.minExtent;

  @override
  double get maxExtent => _element.widget.delegate.maxExtent;

  @override
  void updateChild(double shrinkOffset, bool overlapsContent) {
    assert(_element != null);
    _element._build(shrinkOffset, overlapsContent);
  }

  @protected
  void triggerRebuild() {
    markNeedsLayout();
  }
}

```

### _SliverScrollingPersistentHeader

```dart
class _SliverScrollingPersistentHeader extends _SliverPersistentHeaderRenderObjectWidget {
  const _SliverScrollingPersistentHeader({
    Key key,
    @required SliverPersistentHeaderDelegate delegate,
  }) : super(
    key: key,
    delegate: delegate,
  );

  @override
  _RenderSliverPersistentHeaderForWidgetsMixin createRenderObject(BuildContext context) {
    return _RenderSliverScrollingPersistentHeaderForWidgets(
      stretchConfiguration: delegate.stretchConfiguration
    );
  }
}

```

###  _RenderSliverScrollingPersistentHeaderForWidgets

```dart
class _RenderSliverScrollingPersistentHeaderForWidgets extends RenderSliverScrollingPersistentHeader
  with _RenderSliverPersistentHeaderForWidgetsMixin {
  _RenderSliverScrollingPersistentHeaderForWidgets({
    RenderBox child,
    OverScrollHeaderStretchConfiguration stretchConfiguration,
  }) : super(
    child: child,
    stretchConfiguration: stretchConfiguration,
  );
}

```

###  _SliverPinnedPersistentHeader

```dart
class _SliverPinnedPersistentHeader extends _SliverPersistentHeaderRenderObjectWidget {
  const _SliverPinnedPersistentHeader({
    Key key,
    @required SliverPersistentHeaderDelegate delegate,
  }) : super(
    key: key,
    delegate: delegate,
  );

  @override
  _RenderSliverPersistentHeaderForWidgetsMixin createRenderObject(BuildContext context) {
    return _RenderSliverPinnedPersistentHeaderForWidgets(
      stretchConfiguration: delegate.stretchConfiguration
    );
  }
}

```

###  _RenderSliverPinnedPersistentHeaderForWidgets

```dart
class _RenderSliverPinnedPersistentHeaderForWidgets extends RenderSliverPinnedPersistentHeader
  with _RenderSliverPersistentHeaderForWidgetsMixin {
  _RenderSliverPinnedPersistentHeaderForWidgets({
    RenderBox child,
    OverScrollHeaderStretchConfiguration stretchConfiguration,
  }) : super(
    child: child,
    stretchConfiguration: stretchConfiguration,
  );
}

```

###  _SliverFloatingPersistentHeader

```dart
class _SliverFloatingPersistentHeader extends _SliverPersistentHeaderRenderObjectWidget {
  const _SliverFloatingPersistentHeader({
    Key key,
    @required SliverPersistentHeaderDelegate delegate,
  }) : super(
    key: key,
    delegate: delegate,
  );

  @override
  _RenderSliverPersistentHeaderForWidgetsMixin createRenderObject(BuildContext context) {
    return _RenderSliverFloatingPersistentHeaderForWidgets(
      snapConfiguration: delegate.snapConfiguration,
      stretchConfiguration: delegate.stretchConfiguration,
    );
  }

  @override
  void updateRenderObject(BuildContext context, _RenderSliverFloatingPersistentHeaderForWidgets renderObject) {
    renderObject.snapConfiguration = delegate.snapConfiguration;
    renderObject.stretchConfiguration = delegate.stretchConfiguration;
  }
}

```

###  _RenderSliverFloatingPinnedPersistentHeaderForWidgets

```dart
class _RenderSliverFloatingPinnedPersistentHeaderForWidgets extends RenderSliverFloatingPinnedPersistentHeader
  with _RenderSliverPersistentHeaderForWidgetsMixin {
  _RenderSliverFloatingPinnedPersistentHeaderForWidgets({
    RenderBox child,
    FloatingHeaderSnapConfiguration snapConfiguration,
    OverScrollHeaderStretchConfiguration stretchConfiguration,
  }) : super(
    child: child,
    snapConfiguration: snapConfiguration,
    stretchConfiguration: stretchConfiguration,
  );
}

```

### _SliverFloatingPinnedPersistentHeader

```dart
class _SliverFloatingPinnedPersistentHeader extends _SliverPersistentHeaderRenderObjectWidget {
  const _SliverFloatingPinnedPersistentHeader({
    Key key,
    @required SliverPersistentHeaderDelegate delegate,
  }) : super(
    key: key,
    delegate: delegate,
  );

  @override
  _RenderSliverPersistentHeaderForWidgetsMixin createRenderObject(BuildContext context) {
    return _RenderSliverFloatingPinnedPersistentHeaderForWidgets(
      snapConfiguration: delegate.snapConfiguration,
      stretchConfiguration: delegate.stretchConfiguration,
    );
  }

  @override
  void updateRenderObject(BuildContext context, _RenderSliverFloatingPinnedPersistentHeaderForWidgets renderObject) {
    renderObject.snapConfiguration = delegate.snapConfiguration;
    renderObject.stretchConfiguration = delegate.stretchConfiguration;
  }
}

```

### _RenderSliverFloatingPersistentHeaderForWidgets

```dart
class _RenderSliverFloatingPersistentHeaderForWidgets extends RenderSliverFloatingPersistentHeader
  with _RenderSliverPersistentHeaderForWidgetsMixin {
  _RenderSliverFloatingPersistentHeaderForWidgets({
    RenderBox child,
    FloatingHeaderSnapConfiguration snapConfiguration,
    OverScrollHeaderStretchConfiguration stretchConfiguration,
  }) : super(
    child: child,
    snapConfiguration: snapConfiguration,
    stretchConfiguration: stretchConfiguration,
  );
}

```


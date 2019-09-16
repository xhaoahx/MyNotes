# Flutter的一些组件源码Ⅴ



[TOC]

## Navigator

```dart
/// A widget that manages a set of child widgets with a stack discipline.
///
/// Many apps have a navigator near the top of their widget hierarchy in order
/// to display their logical history using an [Overlay] with the most recently
/// visited pages visually on top of the older pages. Using this pattern lets
/// the navigator visually transition from one page to another by moving the widgets
/// around in the overlay. Similarly, the navigator can be used to show a dialog
/// by positioning the dialog widget above the current page.
///
/// ## Using the Navigator
///
/// Mobile apps typically reveal their contents via full-screen elements
/// called "screens" or "pages". In Flutter these elements are called
/// routes and they're managed by a [Navigator] widget. The navigator
/// manages a stack of [Route] objects and provides methods for managing
/// the stack, like [Navigator.push] and [Navigator.pop].
///
/// When your user interface fits this paradigm of a stack, where the user
/// should be able to _navigate_ back to an earlier element in the stack,
/// the use of routes and the Navigator is appropriate. On certain platforms,
/// such as Android, the system UI will provide a back button (outside the
/// bounds of your application) that will allow the user to navigate back
/// to earlier routes in your application's stack. On platforms that don't
/// have this build-in navigation mechanism, the use of an [AppBar] (typically
/// used in the [Scaffold.appBar] property) can automatically add a back
/// button for user navigation.
///
/// ### Displaying a full-screen route
///
/// Although you can create a navigator directly, it's most common to use
/// the navigator created by a [WidgetsApp] or a [MaterialApp] widget. You
/// can refer to that navigator with [Navigator.of].
///
/// A [MaterialApp] is the simplest way to set things up. The [MaterialApp]'s
/// home becomes the route at the bottom of the [Navigator]'s stack. It is what
/// you see when the app is launched.
///
/// ```dart
/// void main() {
///   runApp(MaterialApp(home: MyAppHome()));
/// }
/// ```
///
/// To push a new route on the stack you can create an instance of
/// [MaterialPageRoute] with a builder function that creates whatever you
/// want to appear on the screen. For example:
///
/// ```dart
/// Navigator.push(context, MaterialPageRoute<void>(
///   builder: (BuildContext context) {
///     return Scaffold(
///       appBar: AppBar(title: Text('My Page')),
///       body: Center(
///         child: FlatButton(
///           child: Text('POP'),
///           onPressed: () {
///             Navigator.pop(context);
///           },
///         ),
///       ),
///     );
///   },
/// ));
/// ```
///
/// The route defines its widget with a builder function instead of a
/// child widget because it will be built and rebuilt in different
/// contexts depending on when it's pushed and popped.
///
/// As you can see, the new route can be popped, revealing the app's home
/// page, with the Navigator's pop method:
///
/// ```dart
/// Navigator.pop(context);
/// ```
///
/// It usually isn't necessary to provide a widget that pops the Navigator
/// in a route with a [Scaffold] because the Scaffold automatically adds a
/// 'back' button to its AppBar. Pressing the back button causes
/// [Navigator.pop] to be called. On Android, pressing the system back
/// button does the same thing.
///
/// ### Using named navigator routes
///
/// Mobile apps often manage a large number of routes and it's often
/// easiest to refer to them by name. Route names, by convention,
/// use a path-like structure (for example, '/a/b/c').
/// The app's home page route is named '/' by default.
///
/// The [MaterialApp] can be created
/// with a [Map<String, WidgetBuilder>] which maps from a route's name to
/// a builder function that will create it. The [MaterialApp] uses this
/// map to create a value for its navigator's [onGenerateRoute] callback.
///
/// ```dart
/// void main() {
///   runApp(MaterialApp(
///     home: MyAppHome(), // becomes the route named '/'
///     routes: <String, WidgetBuilder> {
///       '/a': (BuildContext context) => MyPage(title: 'page A'),
///       '/b': (BuildContext context) => MyPage(title: 'page B'),
///       '/c': (BuildContext context) => MyPage(title: 'page C'),
///     },
///   ));
/// }
/// ```
///
/// To show a route by name:
///
/// ```dart
/// Navigator.pushNamed(context, '/b');
/// ```
///
/// ### Routes can return a value
///
/// When a route is pushed to ask the user for a value, the value can be
/// returned via the [pop] method's result parameter.
///
/// Methods that push a route return a [Future]. The Future resolves when the
/// route is popped and the [Future]'s value is the [pop] method's `result`
/// parameter.
///
/// For example if we wanted to ask the user to press 'OK' to confirm an
/// operation we could `await` the result of [Navigator.push]:
///
/// ```dart
/// bool value = await Navigator.push(context, MaterialPageRoute<bool>(
///   builder: (BuildContext context) {
///     return Center(
///       child: GestureDetector(
///         child: Text('OK'),
///         onTap: () { Navigator.pop(context, true); }
///       ),
///     );
///   }
/// ));
/// ```
///
/// If the user presses 'OK' then value will be true. If the user backs
/// out of the route, for example by pressing the Scaffold's back button,
/// the value will be null.
///
/// When a route is used to return a value, the route's type parameter must
/// match the type of [pop]'s result. That's why we've used
/// `MaterialPageRoute<bool>` instead of `MaterialPageRoute<void>` or just
/// `MaterialPageRoute`. (If you prefer to not specify the types, though, that's
/// fine too.)
///
/// ### Popup routes
///
/// Routes don't have to obscure the entire screen. [PopupRoute]s cover the
/// screen with a [ModalRoute.barrierColor] that can be only partially opaque to
/// allow the current screen to show through. Popup routes are "modal" because
/// they block input to the widgets below.
///
/// There are functions which create and show popup routes. For
/// example: [showDialog], [showMenu], and [showModalBottomSheet]. These
/// functions return their pushed route's Future as described above.
/// Callers can await the returned value to take an action when the
/// route is popped, or to discover the route's value.
///
/// There are also widgets which create popup routes, like [PopupMenuButton] and
/// [DropdownButton]. These widgets create internal subclasses of PopupRoute
/// and use the Navigator's push and pop methods to show and dismiss them.
///
/// ### Custom routes
///
/// You can create your own subclass of one of the widget library route classes
/// like [PopupRoute], [ModalRoute], or [PageRoute], to control the animated
/// transition employed to show the route, the color and behavior of the route's
/// modal barrier, and other aspects of the route.
///
/// The [PageRouteBuilder] class makes it possible to define a custom route
/// in terms of callbacks. Here's an example that rotates and fades its child
/// when the route appears or disappears. This route does not obscure the entire
/// screen because it specifies `opaque: false`, just as a popup route does.
///
/// ```dart
/// Navigator.push(context, PageRouteBuilder(
///   opaque: false,
///   pageBuilder: (BuildContext context, _, __) {
///     return Center(child: Text('My PageRoute'));
///   },
///   transitionsBuilder: (___, Animation<double> animation, ____, Widget child) {
///     return FadeTransition(
///       opacity: animation,
///       child: RotationTransition(
///         turns: Tween<double>(begin: 0.5, end: 1.0).animate(animation),
///         child: child,
///       ),
///     );
///   }
/// ));
/// ```
///
/// The page route is built in two parts, the "page" and the
/// "transitions". The page becomes a descendant of the child passed to
/// the `transitionsBuilder` function. Typically the page is only built once,
/// because it doesn't depend on its animation parameters (elided with `_`
/// and `__` in this example). The transition is built on every frame
/// for its duration.
///
/// ### Nesting Navigators
///
/// An app can use more than one Navigator. Nesting one Navigator below
/// another Navigator can be used to create an "inner journey" such as tabbed
/// navigation, user registration, store checkout, or other independent journeys
/// that represent a subsection of your overall application.
///
/// #### Real World Example
///
/// It is standard practice for iOS apps to use tabbed navigation where each
/// tab maintains its own navigation history. Therefore, each tab has its own
/// [Navigator], creating a kind of "parallel navigation."
///
/// In addition to the parallel navigation of the tabs, it is still possible to
/// launch full-screen pages that completely cover the tabs. For example: an
/// on-boarding flow, or an alert dialog. Therefore, there must exist a "root"
/// [Navigator] that sits above the tab navigation. As a result, each of the
/// tab's [Navigator]s are actually nested [Navigator]s sitting below a single
/// root [Navigator].
///
/// The nested [Navigator]s for tabbed navigation sit in [WidgetApp] and
/// [CupertinoTabView], so you don't need to worry about nested [Navigator]s
/// in this situation, but it's a real world example where nested [Navigator]s
/// are used.
///
/// {@tool snippet --template=freeform}
/// The following example demonstrates how a nested [Navigator] can be used to
/// present a standalone user registration journey.
///
/// Even though this example uses two [Navigator]s to demonstrate nested
/// [Navigator]s, a similar result is possible using only a single [Navigator].
///
/// Run this example with `flutter run --route=/signup` to start it with
/// the signup flow instead of on the home page.
///
/// ```dart imports
/// import 'package:flutter/material.dart';
/// ```
///
/// ```dart main
/// void main() => runApp(new MyApp());
/// ```
///
/// ```dart
/// class MyApp extends StatelessWidget {
///   @override
///   Widget build(BuildContext context) {
///     return MaterialApp(
///       title: 'Flutter Code Sample for Navigator',
///       // MaterialApp contains our top-level Navigator
///       initialRoute: '/',
///       routes: {
///         '/': (BuildContext context) => HomePage(),
///         '/signup': (BuildContext context) => SignUpPage(),
///       },
///     );
///   }
/// }
///
/// class HomePage extends StatelessWidget {
///   @override
///   Widget build(BuildContext context) {
///     return DefaultTextStyle(
///       style: Theme.of(context).textTheme.display1,
///       child: Container(
///         color: Colors.white,
///         alignment: Alignment.center,
///         child: Text('Home Page'),
///       ),
///     );
///   }
/// }
///
/// class CollectPersonalInfoPage extends StatelessWidget {
///   @override
///   Widget build(BuildContext context) {
///     return DefaultTextStyle(
///       style: Theme.of(context).textTheme.display1,
///       child: GestureDetector(
///         onTap: () {
///           // This moves from the personal info page to the credentials page,
///           // replacing this page with that one.
///           Navigator.of(context)
///             .pushReplacementNamed('signup/choose_credentials');
///         },
///         child: Container(
///           color: Colors.lightBlue,
///           alignment: Alignment.center,
///           child: Text('Collect Personal Info Page'),
///         ),
///       ),
///     );
///   }
/// }
///
/// class ChooseCredentialsPage extends StatelessWidget {
///   const ChooseCredentialsPage({
///     this.onSignupComplete,
///   });
///
///   final VoidCallback onSignupComplete;
///
///   @override
///   Widget build(BuildContext context) {
///     return GestureDetector(
///       onTap: onSignupComplete,
///       child: DefaultTextStyle(
///         style: Theme.of(context).textTheme.display1,
///         child: Container(
///           color: Colors.pinkAccent,
///           alignment: Alignment.center,
///           child: Text('Choose Credentials Page'),
///         ),
///       ),
///     );
///   }
/// }
///
/// class SignUpPage extends StatelessWidget {
///   @override
///   Widget build(BuildContext context) {
///     // SignUpPage builds its own Navigator which ends up being a nested
///     // Navigator in our app.
///     return Navigator(
///       initialRoute: 'signup/personal_info',
///       onGenerateRoute: (RouteSettings settings) {
///         WidgetBuilder builder;
///         switch (settings.name) {
///           case 'signup/personal_info':
///           // Assume CollectPersonalInfoPage collects personal info and then
///           // navigates to 'signup/choose_credentials'.
///             builder = (BuildContext _) => CollectPersonalInfoPage();
///             break;
///           case 'signup/choose_credentials':
///           // Assume ChooseCredentialsPage collects new credentials and then
///           // invokes 'onSignupComplete()'.
///             builder = (BuildContext _) => ChooseCredentialsPage(
///               onSignupComplete: () {
///                 // Referencing Navigator.of(context) from here refers to the
///                 // top level Navigator because SignUpPage is above the
///                 // nested Navigator that it created. Therefore, this pop()
///                 // will pop the entire "sign up" journey and return to the
///                 // "/" route, AKA HomePage.
///                 Navigator.of(context).pop();
///               },
///             );
///             break;
///           default:
///             throw Exception('Invalid route: ${settings.name}');
///         }
///         return MaterialPageRoute(builder: builder, settings: settings);
///       },
///     );
///   }
/// }
/// ```
/// {@end-tool}
///
/// [Navigator.of] operates on the nearest ancestor [Navigator] from the given
/// [BuildContext]. Be sure to provide a [BuildContext] below the intended
/// [Navigator], especially in large [build] methods where nested [Navigator]s
/// are created. The [Builder] widget can be used to access a [BuildContext] at
/// a desired location in the widget subtree.
class Navigator extends StatefulWidget {
  /// Creates a widget that maintains a stack-based history of child widgets.
  ///
  /// The [onGenerateRoute] argument must not be null.
  const Navigator({
    Key key,
    this.initialRoute,
    @required this.onGenerateRoute,
    this.onUnknownRoute,
    this.observers = const <NavigatorObserver>[],
  }) : assert(onGenerateRoute != null),
       super(key: key);

  /// The name of the first route to show.
  ///
  /// By default, this defers to [dart:ui.Window.defaultRouteName].
  ///
  /// If this string contains any `/` characters, then the string is split on
  /// those characters and substrings from the start of the string up to each
  /// such character are, in turn, used as routes to push.
  ///
  /// For example, if the route `/stocks/HOOLI` was used as the [initialRoute],
  /// then the [Navigator] would push the following routes on startup: `/`,
  /// `/stocks`, `/stocks/HOOLI`. This enables deep linking while allowing the
  /// application to maintain a predictable route history.
  ///
  /// If any of the intermediate routes doesn't exist, it'll simply be skipped.
  /// In the example above, if `/stocks` doesn't have a corresponding route in
  /// the app, it'll be skipped and only `/` and `/stocks/HOOLI` will be pushed.
  ///
  /// That said, the full route has to map to something in the app in order for
  /// this to work. In our example, `/stocks/HOOLI` has to map to a route in the
  /// app. Otherwise, [initialRoute] will be ignored and [defaultRouteName] will
  /// be used instead.
  final String initialRoute;

  /// Called to generate a route for a given [RouteSettings].
  final RouteFactory onGenerateRoute;

  /// Called when [onGenerateRoute] fails to generate a route.
  ///
  /// This callback is typically used for error handling. For example, this
  /// callback might always generate a "not found" page that describes the route
  /// that wasn't found.
  ///
  /// Unknown routes can arise either from errors in the app or from external
  /// requests to push routes, such as from Android intents.
  final RouteFactory onUnknownRoute;

  /// A list of observers for this navigator.
  final List<NavigatorObserver> observers;

  /// The default name for the [initialRoute].
  ///
  /// See also:
  ///
  ///  * [dart:ui.Window.defaultRouteName], which reflects the route that the
  ///    application was started with.
  static const String defaultRouteName = '/';

  /// Push a named route onto the navigator that most tightly encloses the given
  /// context.
  ///
  /// {@template flutter.widgets.navigator.pushNamed}
  /// The route name will be passed to that navigator's [onGenerateRoute]
  /// callback. The returned route will be pushed into the navigator.
  ///
  /// The new route and the previous route (if any) are notified (see
  /// [Route.didPush] and [Route.didChangeNext]). If the [Navigator] has any
  /// [Navigator.observers], they will be notified as well (see
  /// [NavigatorObserver.didPush]).
  ///
  /// Ongoing gestures within the current route are canceled when a new route is
  /// pushed.
  ///
  /// Returns a [Future] that completes to the `result` value passed to [pop]
  /// when the pushed route is popped off the navigator.
  ///
  /// The `T` type argument is the type of the return value of the route.
  /// {@endtemplate}
  ///
  /// {@template flutter.widgets.navigator.pushNamed.arguments}
  /// The provided `arguments` are passed to the pushed route via
  /// [RouteSettings.arguments]. Any object can be passed as `arguments` (e.g. a
  /// [String], [int], or an instance of a custom `MyRouteArguments` class).
  /// Often, a [Map] is used to pass key-value pairs.
  ///
  /// The `arguments` may be used in [Navigator.onGenerateRoute] or
  /// [Navigator.onUnknownRoute] to construct the route.
  /// {@endtemplate}
  ///
  /// {@tool sample}
  ///
  /// Typical usage is as follows:
  ///
  /// ```dart
  /// void _didPushButton() {
  ///   Navigator.pushNamed(context, '/settings');
  /// }
  /// ```
  /// {@end-tool}
  ///
  /// {@tool sample}
  ///
  /// The following example shows how to pass additional `arguments` to the
  /// route:
  ///
  /// ```dart
  /// void _showBerlinWeather() {
  ///   Navigator.pushNamed(
  ///     context,
  ///     '/weather',
  ///     arguments: <String, String>{
  ///       'city': 'Berlin',
  ///       'country': 'Germany',
  ///     },
  ///   );
  /// }
  /// ```
  /// {@end-tool}
  ///
  /// {@tool sample}
  ///
  /// The following example shows how to pass a custom Object to the route:
  ///
  /// ```dart
  /// class WeatherRouteArguments {
  ///   WeatherRouteArguments({ this.city, this.country });
  ///   final String city;
  ///   final String country;
  ///
  ///   bool get isGermanCapital {
  ///     return country == 'Germany' && city == 'Berlin';
  ///   }
  /// }
  ///
  /// void _showWeather() {
  ///   Navigator.pushNamed(
  ///     context,
  ///     '/weather',
  ///     arguments: WeatherRouteArguments(city: 'Berlin', country: 'Germany'),
  ///   );
  /// }
  /// ```
  /// {@end-tool}
  @optionalTypeArgs
  static Future<T> pushNamed<T extends Object>(
    BuildContext context,
    String routeName, {
    Object arguments,
   }) {
    return Navigator.of(context).pushNamed<T>(routeName, arguments: arguments);
  }

  /// Replace the current route of the navigator that most tightly encloses the
  /// given context by pushing the route named [routeName] and then disposing
  /// the previous route once the new route has finished animating in.
  ///
  /// {@template flutter.widgets.navigator.pushReplacementNamed}
  /// If non-null, `result` will be used as the result of the route that is
  /// removed; the future that had been returned from pushing that old route
  /// will complete with `result`. Routes such as dialogs or popup menus
  /// typically use this mechanism to return the value selected by the user to
  /// the widget that created their route. The type of `result`, if provided,
  /// must match the type argument of the class of the old route (`TO`).
  ///
  /// The route name will be passed to the navigator's [onGenerateRoute]
  /// callback. The returned route will be pushed into the navigator.
  ///
  /// The new route and the route below the removed route are notified (see
  /// [Route.didPush] and [Route.didChangeNext]). If the [Navigator] has any
  /// [Navigator.observers], they will be notified as well (see
  /// [NavigatorObserver.didReplace]). The removed route is notified once the
  /// new route has finished animating (see [Route.didComplete]). The removed
  /// route's exit animation is not run (see [popAndPushNamed] for a variant
  /// that does animated the removed route).
  ///
  /// Ongoing gestures within the current route are canceled when a new route is
  /// pushed.
  ///
  /// Returns a [Future] that completes to the `result` value passed to [pop]
  /// when the pushed route is popped off the navigator.
  ///
  /// The `T` type argument is the type of the return value of the new route,
  /// and `TO` is the type of the return value of the old route.
  /// {@endtemplate}
  ///
  /// {@macro flutter.widgets.navigator.pushNamed.arguments}
  ///
  /// {@tool sample}
  ///
  /// Typical usage is as follows:
  ///
  /// ```dart
  /// void _switchToBrightness() {
  ///   Navigator.pushReplacementNamed(context, '/settings/brightness');
  /// }
  /// ```
  /// {@end-tool}
  @optionalTypeArgs
  static Future<T> pushReplacementNamed<T extends Object, TO extends Object>(
    BuildContext context,
    String routeName, {
    TO result,
    Object arguments,
  }) {
    return Navigator.of(context).pushReplacementNamed<T, TO>(routeName, arguments: arguments, result: result);
  }

  /// Pop the current route off the navigator that most tightly encloses the
  /// given context and push a named route in its place.
  ///
  /// {@template flutter.widgets.navigator.popAndPushNamed}
  /// If non-null, `result` will be used as the result of the route that is
  /// popped; the future that had been returned from pushing the popped route
  /// will complete with `result`. Routes such as dialogs or popup menus
  /// typically use this mechanism to return the value selected by the user to
  /// the widget that created their route. The type of `result`, if provided,
  /// must match the type argument of the class of the popped route (`TO`).
  ///
  /// The route name will be passed to the navigator's [onGenerateRoute]
  /// callback. The returned route will be pushed into the navigator.
  ///
  /// The new route, the old route, and the route below the old route (if any)
  /// are all notified (see [Route.didPop], [Route.didComplete],
  /// [Route.didPopNext], [Route.didPush], and [Route.didChangeNext]). If the
  /// [Navigator] has any [Navigator.observers], they will be notified as well
  /// (see [NavigatorObserver.didPop] and [NavigatorObservers.didPush]). The
  /// animations for the pop and the push are performed simultaneously, so the
  /// route below may be briefly visible even if both the old route and the new
  /// route are opaque (see [TransitionRoute.opaque]).
  ///
  /// Ongoing gestures within the current route are canceled when a new route is
  /// pushed.
  ///
  /// Returns a [Future] that completes to the `result` value passed to [pop]
  /// when the pushed route is popped off the navigator.
  ///
  /// The `T` type argument is the type of the return value of the new route,
  /// and `TO` is the return value type of the old route.
  /// {@endtemplate}
  ///
  /// {@macro flutter.widgets.navigator.pushNamed.arguments}
  ///
  /// {@tool sample}
  ///
  /// Typical usage is as follows:
  ///
  /// ```dart
  /// void _selectAccessibility() {
  ///   Navigator.popAndPushNamed(context, '/settings/accessibility');
  /// }
  /// ```
  /// {@end-tool}
  @optionalTypeArgs
  static Future<T> popAndPushNamed<T extends Object, TO extends Object>(
    BuildContext context,
    String routeName, {
    TO result,
    Object arguments,
   }) {
    return Navigator.of(context).popAndPushNamed<T, TO>(routeName, arguments: arguments, result: result);
  }

  /// Push the route with the given name onto the navigator that most tightly
  /// encloses the given context, and then remove all the previous routes until
  /// the `predicate` returns true.
  ///
  /// {@template flutter.widgets.navigator.pushNamedAndRemoveUntil}
  /// The predicate may be applied to the same route more than once if
  /// [Route.willHandlePopInternally] is true.
  ///
  /// To remove routes until a route with a certain name, use the
  /// [RoutePredicate] returned from [ModalRoute.withName].
  ///
  /// To remove all the routes below the pushed route, use a [RoutePredicate]
  /// that always returns false (e.g. `(Route<dynamic> route) => false`).
  ///
  /// The removed routes are removed without being completed, so this method
  /// does not take a return value argument.
  ///
  /// The new route's name (`routeName`) will be passed to the navigator's
  /// [onGenerateRoute] callback. The returned route will be pushed into the
  /// navigator.
  ///
  /// The new route and the route below the bottommost removed route (which
  /// becomes the route below the new route) are notified (see [Route.didPush]
  /// and [Route.didChangeNext]). If the [Navigator] has any
  /// [Navigator.observers], they will be notified as well (see
  /// [NavigatorObservers.didPush] and [NavigatorObservers.didRemove]). The
  /// removed routes are disposed, without being notified, once the new route
  /// has finished animating. The futures that had been returned from pushing
  /// those routes will not complete.
  ///
  /// Ongoing gestures within the current route are canceled when a new route is
  /// pushed.
  ///
  /// Returns a [Future] that completes to the `result` value passed to [pop]
  /// when the pushed route is popped off the navigator.
  ///
  /// The `T` type argument is the type of the return value of the new route.
  /// {@endtemplate}
  ///
  /// {@macro flutter.widgets.navigator.pushNamed.arguments}
  ///
  /// {@tool sample}
  ///
  /// Typical usage is as follows:
  ///
  /// ```dart
  /// void _resetToCalendar() {
  ///   Navigator.pushNamedAndRemoveUntil(context, '/calendar', ModalRoute.withName('/'));
  /// }
  /// ```
  /// {@end-tool}
  @optionalTypeArgs
  static Future<T> pushNamedAndRemoveUntil<T extends Object>(
    BuildContext context,
    String newRouteName,
    RoutePredicate predicate, {
    Object arguments,
  }) {
    return Navigator.of(context).pushNamedAndRemoveUntil<T>(newRouteName, predicate, arguments: arguments);
  }

  /// Push the given route onto the navigator that most tightly encloses the
  /// given context.
  ///
  /// {@template flutter.widgets.navigator.push}
  /// The new route and the previous route (if any) are notified (see
  /// [Route.didPush] and [Route.didChangeNext]). If the [Navigator] has any
  /// [Navigator.observers], they will be notified as well (see
  /// [NavigatorObserver.didPush]).
  ///
  /// Ongoing gestures within the current route are canceled when a new route is
  /// pushed.
  ///
  /// Returns a [Future] that completes to the `result` value passed to [pop]
  /// when the pushed route is popped off the navigator.
  ///
  /// The `T` type argument is the type of the return value of the route.
  /// {@endtemplate}
  ///
  /// {@tool sample}
  ///
  /// Typical usage is as follows:
  ///
  /// ```dart
  /// void _openMyPage() {
  ///   Navigator.push(context, MaterialPageRoute(builder: (BuildContext context) => MyPage()));
  /// }
  /// ```
  /// {@end-tool}
  @optionalTypeArgs
  static Future<T> push<T extends Object>(BuildContext context, Route<T> route) {
    return Navigator.of(context).push(route);
  }

  /// Replace the current route of the navigator that most tightly encloses the
  /// given context by pushing the given route and then disposing the previous
  /// route once the new route has finished animating in.
  ///
  /// {@template flutter.widgets.navigator.pushReplacement}
  /// If non-null, `result` will be used as the result of the route that is
  /// removed; the future that had been returned from pushing that old route will
  /// complete with `result`. Routes such as dialogs or popup menus typically
  /// use this mechanism to return the value selected by the user to the widget
  /// that created their route. The type of `result`, if provided, must match
  /// the type argument of the class of the old route (`TO`).
  ///
  /// The new route and the route below the removed route are notified (see
  /// [Route.didPush] and [Route.didChangeNext]). If the [Navigator] has any
  /// [Navigator.observers], they will be notified as well (see
  /// [NavigatorObserver.didReplace]). The removed route is notified once the
  /// new route has finished animating (see [Route.didComplete]).
  ///
  /// Ongoing gestures within the current route are canceled when a new route is
  /// pushed.
  ///
  /// Returns a [Future] that completes to the `result` value passed to [pop]
  /// when the pushed route is popped off the navigator.
  ///
  /// The `T` type argument is the type of the return value of the new route,
  /// and `TO` is the type of the return value of the old route.
  /// {@endtemplate}
  ///
  /// {@tool sample}
  ///
  /// Typical usage is as follows:
  ///
  /// ```dart
  /// void _completeLogin() {
  ///   Navigator.pushReplacement(
  ///       context, MaterialPageRoute(builder: (BuildContext context) => MyHomePage()));
  /// }
  /// ```
  /// {@end-tool}
  @optionalTypeArgs
  static Future<T> pushReplacement<T extends Object, TO extends Object>(BuildContext context, Route<T> newRoute, { TO result }) {
    return Navigator.of(context).pushReplacement<T, TO>(newRoute, result: result);
  }

  /// Push the given route onto the navigator that most tightly encloses the
  /// given context, and then remove all the previous routes until the
  /// `predicate` returns true.
  ///
  /// {@template flutter.widgets.navigator.pushAndRemoveUntil}
  /// The predicate may be applied to the same route more than once if
  /// [Route.willHandlePopInternally] is true.
  ///
  /// To remove routes until a route with a certain name, use the
  /// [RoutePredicate] returned from [ModalRoute.withName].
  ///
  /// To remove all the routes below the pushed route, use a [RoutePredicate]
  /// that always returns false (e.g. `(Route<dynamic> route) => false`).
  ///
  /// The removed routes are removed without being completed, so this method
  /// does not take a return value argument.
  ///
  /// The newly pushed route and its preceding route are notified for
  /// [Route.didPush]. After removal, the new route and its new preceding route,
  /// (the route below the bottommost removed route) are notified through
  /// [Route.didChangeNext]). If the [Navigator] has any [Navigator.observers],
  /// they will be notified as well (see [NavigatorObservers.didPush] and
  /// [NavigatorObservers.didRemove]). The removed routes are disposed of and
  /// notified, once the new route has finished animating. The futures that had
  /// been returned from pushing those routes will not complete.
  ///
  /// Ongoing gestures within the current route are canceled when a new route is
  /// pushed.
  ///
  /// Returns a [Future] that completes to the `result` value passed to [pop]
  /// when the pushed route is popped off the navigator.
  ///
  /// The `T` type argument is the type of the return value of the new route.
  /// {@endtemplate}
  ///
  /// {@tool sample}
  ///
  /// Typical usage is as follows:
  ///
  /// ```dart
  /// void _finishAccountCreation() {
  ///   Navigator.pushAndRemoveUntil(
  ///     context,
  ///     MaterialPageRoute(builder: (BuildContext context) => MyHomePage()),
  ///     ModalRoute.withName('/'),
  ///   );
  /// }
  /// ```
  /// {@end-tool}
  @optionalTypeArgs
  static Future<T> pushAndRemoveUntil<T extends Object>(BuildContext context, Route<T> newRoute, RoutePredicate predicate) {
    return Navigator.of(context).pushAndRemoveUntil<T>(newRoute, predicate);
  }

  /// Replaces a route on the navigator that most tightly encloses the given
  /// context with a new route.
  ///
  /// {@template flutter.widgets.navigator.replace}
  /// The old route must not be currently visible, as this method skips the
  /// animations and therefore the removal would be jarring if it was visible.
  /// To replace the top-most route, consider [pushReplacement] instead, which
  /// _does_ animate the new route, and delays removing the old route until the
  /// new route has finished animating.
  ///
  /// The removed route is removed without being completed, so this method does
  /// not take a return value argument.
  ///
  /// The new route, the route below the new route (if any), and the route above
  /// the new route, are all notified (see [Route.didReplace],
  /// [Route.didChangeNext], and [Route.didChangePrevious]). If the [Navigator]
  /// has any [Navigator.observers], they will be notified as well (see
  /// [NavigatorObservers.didReplace]). The removed route is disposed without
  /// being notified. The future that had been returned from pushing that routes
  /// will not complete.
  ///
  /// This can be useful in combination with [removeRouteBelow] when building a
  /// non-linear user experience.
  ///
  /// The `T` type argument is the type of the return value of the new route.
  /// {@endtemplate}
  ///
  /// See also:
  ///
  ///  * [replaceRouteBelow], which is the same but identifies the route to be
  ///    removed by reference to the route above it, rather than directly.
  @optionalTypeArgs
  static void replace<T extends Object>(BuildContext context, { @required Route<dynamic> oldRoute, @required Route<T> newRoute }) {
    return Navigator.of(context).replace<T>(oldRoute: oldRoute, newRoute: newRoute);
  }

  /// Replaces a route on the navigator that most tightly encloses the given
  /// context with a new route. The route to be replaced is the one below the
  /// given `anchorRoute`.
  ///
  /// {@template flutter.widgets.navigator.replaceRouteBelow}
  /// The old route must not be current visible, as this method skips the
  /// animations and therefore the removal would be jarring if it was visible.
  /// To replace the top-most route, consider [pushReplacement] instead, which
  /// _does_ animate the new route, and delays removing the old route until the
  /// new route has finished animating.
  ///
  /// The removed route is removed without being completed, so this method does
  /// not take a return value argument.
  ///
  /// The new route, the route below the new route (if any), and the route above
  /// the new route, are all notified (see [Route.didReplace],
  /// [Route.didChangeNext], and [Route.didChangePrevious]). If the [Navigator]
  /// has any [Navigator.observers], they will be notified as well (see
  /// [NavigatorObservers.didReplace]). The removed route is disposed without
  /// being notified. The future that had been returned from pushing that routes
  /// will not complete.
  ///
  /// The `T` type argument is the type of the return value of the new route.
  /// {@endtemplate}
  ///
  /// See also:
  ///
  ///  * [replace], which is the same but identifies the route to be removed
  ///    directly.
  @optionalTypeArgs
  static void replaceRouteBelow<T extends Object>(BuildContext context, { @required Route<dynamic> anchorRoute, Route<T> newRoute }) {
    return Navigator.of(context).replaceRouteBelow<T>(anchorRoute: anchorRoute, newRoute: newRoute);
  }

  /// Whether the navigator that most tightly encloses the given context can be
  /// popped.
  ///
  /// {@template flutter.widgets.navigator.canPop}
  /// The initial route cannot be popped off the navigator, which implies that
  /// this function returns true only if popping the navigator would not remove
  /// the initial route.
  ///
  /// If there is no [Navigator] in scope, returns false.
  /// {@endtemplate}
  ///
  /// See also:
  ///
  ///  * [Route.isFirst], which returns true for routes for which [canPop]
  ///    returns false.
  static bool canPop(BuildContext context) {
    final NavigatorState navigator = Navigator.of(context, nullOk: true);
    return navigator != null && navigator.canPop();
  }

  /// Tries to pop the current route of the navigator that most tightly encloses
  /// the given context, while honoring the route's [Route.willPop]
  /// state.
  ///
  /// {@template flutter.widgets.navigator.maybePop}
  /// Returns false if the route deferred to the next enclosing navigator
  /// (possibly the system); otherwise, returns true (whether the route was
  /// popped or not).
  ///
  /// This method is typically called to handle a user-initiated [pop]. For
  /// example on Android it's called by the binding for the system's back
  /// button.
  ///
  /// The `T` type argument is the type of the return value of the current
  /// route.
  /// {@endtemplate}
  ///
  /// See also:
  ///
  ///  * [Form], which provides an `onWillPop` callback that enables the form
  ///    to veto a [pop] initiated by the app's back button.
  ///  * [ModalRoute], which provides a `scopedWillPopCallback` that can be used
  ///    to define the route's `willPop` method.
  @optionalTypeArgs
  static Future<bool> maybePop<T extends Object>(BuildContext context, [ T result ]) {
    return Navigator.of(context).maybePop<T>(result);
  }

  /// Pop the top-most route off the navigator that most tightly encloses the
  /// given context.
  ///
  /// {@template flutter.widgets.navigator.pop}
  /// The current route's [Route.didPop] method is called first. If that method
  /// returns false, then this method returns true but nothing else is changed
  /// (the route is expected to have popped some internal state; see e.g.
  /// [LocalHistoryRoute]). Otherwise, the rest of this description applies.
  ///
  /// If non-null, `result` will be used as the result of the route that is
  /// popped; the future that had been returned from pushing the popped route
  /// will complete with `result`. Routes such as dialogs or popup menus
  /// typically use this mechanism to return the value selected by the user to
  /// the widget that created their route. The type of `result`, if provided,
  /// must match the type argument of the class of the popped route (`T`).
  ///
  /// The popped route and the route below it are notified (see [Route.didPop],
  /// [Route.didComplete], and [Route.didPopNext]). If the [Navigator] has any
  /// [Navigator.observers], they will be notified as well (see
  /// [NavigatorObserver.didPop]).
  ///
  /// The `T` type argument is the type of the return value of the popped route.
  ///
  /// Returns true if a route was popped (including if [Route.didPop] returned
  /// false); returns false if there are no further previous routes.
  /// {@endtemplate}
  ///
  /// {@tool sample}
  ///
  /// Typical usage for closing a route is as follows:
  ///
  /// ```dart
  /// void _close() {
  ///   Navigator.pop(context);
  /// }
  /// ```
  /// {@end-tool}
  ///
  /// A dialog box might be closed with a result:
  ///
  /// ```dart
  /// void _accept() {
  ///   Navigator.pop(context, true); // dialog returns true
  /// }
  /// ```
  @optionalTypeArgs
  static bool pop<T extends Object>(BuildContext context, [ T result ]) {
    return Navigator.of(context).pop<T>(result);
  }

  /// Calls [pop] repeatedly on the navigator that most tightly encloses the
  /// given context until the predicate returns true.
  ///
  /// {@template flutter.widgets.navigator.popUntil}
  /// The predicate may be applied to the same route more than once if
  /// [Route.willHandlePopInternally] is true.
  ///
  /// To pop until a route with a certain name, use the [RoutePredicate]
  /// returned from [ModalRoute.withName].
  ///
  /// The routes are closed with null as their `return` value.
  ///
  /// See [pop] for more details of the semantics of popping a route.
  /// {@endtemplate}
  ///
  /// {@tool sample}
  ///
  /// Typical usage is as follows:
  ///
  /// ```dart
  /// void _logout() {
  ///   Navigator.popUntil(context, ModalRoute.withName('/login'));
  /// }
  /// ```
  /// {@end-tool}
  static void popUntil(BuildContext context, RoutePredicate predicate) {
    Navigator.of(context).popUntil(predicate);
  }

  /// Immediately remove `route` from the navigator that most tightly encloses
  /// the given context, and [Route.dispose] it.
  ///
  /// {@template flutter.widgets.navigator.removeRoute}
  /// The removed route is removed without being completed, so this method does
  /// not take a return value argument. No animations are run as a result of
  /// this method call.
  ///
  /// The routes below and above the removed route are notified (see
  /// [Route.didChangeNext] and [Route.didChangePrevious]). If the [Navigator]
  /// has any [Navigator.observers], they will be notified as well (see
  /// [NavigatorObserver.didRemove]). The removed route is disposed without
  /// being notified. The future that had been returned from pushing that routes
  /// will not complete.
  ///
  /// The given `route` must be in the history; this method will throw an
  /// exception if it is not.
  ///
  /// Ongoing gestures within the current route are canceled.
  /// {@endtemplate}
  ///
  /// This method is used, for example, to instantly dismiss dropdown menus that
  /// are up when the screen's orientation changes.
  static void removeRoute(BuildContext context, Route<dynamic> route) {
    return Navigator.of(context).removeRoute(route);
  }

  /// Immediately remove a route from the navigator that most tightly encloses
  /// the given context, and [Route.dispose] it. The route to be replaced is the
  /// one below the given `anchorRoute`.
  ///
  /// {@template flutter.widgets.navigator.removeRouteBelow}
  /// The removed route is removed without being completed, so this method does
  /// not take a return value argument. No animations are run as a result of
  /// this method call.
  ///
  /// The routes below and above the removed route are notified (see
  /// [Route.didChangeNext] and [Route.didChangePrevious]). If the [Navigator]
  /// has any [Navigator.observers], they will be notified as well (see
  /// [NavigatorObserver.didRemove]). The removed route is disposed without
  /// being notified. The future that had been returned from pushing that routes
  /// will not complete.
  ///
  /// The given `anchorRoute` must be in the history and must have a route below
  /// it; this method will throw an exception if it is not or does not.
  ///
  /// Ongoing gestures within the current route are canceled.
  /// {@endtemplate}
  static void removeRouteBelow(BuildContext context, Route<dynamic> anchorRoute) {
    return Navigator.of(context).removeRouteBelow(anchorRoute);
  }

  /// The state from the closest instance of this class that encloses the given context.
  ///
  /// Typical usage is as follows:
  ///
  /// ```dart
  /// Navigator.of(context)
  ///   ..pop()
  ///   ..pop()
  ///   ..pushNamed('/settings');
  /// ```
  ///
  /// If `rootNavigator` is set to true, the state from the furthest instance of
  /// this class is given instead. Useful for pushing contents above all subsequent
  /// instances of [Navigator].
  static NavigatorState of(
    BuildContext context, {
    bool rootNavigator = false,
    bool nullOk = false,
  }) {
    final NavigatorState navigator = rootNavigator
        ? context.rootAncestorStateOfType(const TypeMatcher<NavigatorState>())
        : context.ancestorStateOfType(const TypeMatcher<NavigatorState>());
    assert(() {
      if (navigator == null && !nullOk) {
        throw FlutterError(
          'Navigator operation requested with a context that does not include a Navigator.\n'
          'The context used to push or pop routes from the Navigator must be that of a '
          'widget that is a descendant of a Navigator widget.'
        );
      }
      return true;
    }());
    return navigator;
  }

  @override
  NavigatorState createState() => NavigatorState();
}
```


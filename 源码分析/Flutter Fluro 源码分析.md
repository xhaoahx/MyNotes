# Flutter Fluro 源码分析

[TOC]

## 枚举，函数原型

```dart

enum HandlerType {
  route,
  function,
}

enum RouteTreeNodeType {
  component,
  parameter,
}

enum TransitionType {
  native,
  nativeModal,
  inFromLeft,
  inFromRight,
  inFromBottom,
  fadeIn,
  custom, // if using custom then you must also provide a transition
  material,
  materialFullScreenDialog,
  cupertino,
  cupertinoFullScreenDialog,
}

enum RouteMatchType {
  visual,
  nonVisual,
  noMatch,
}

typedef Route<T> RouteCreator<T>(
    RouteSettings route, Map<String, List<String>> parameters);

///
typedef Widget HandlerFunc(
    BuildContext context, Map<String, List<String>> parameters);

```

## Router
```dart
class Router {
  static final appRouter = Router();

  /// The tree structure that stores the defined routes
  final RouteTree _routeTree = RouteTree();

  /// Generic handler for when a route has not been defined
  Handler notFoundHandler;

  /// Creates a [PageRoute] definition for the passed [RouteHandler]. You can optionally provide a default transition type.
  void define(String routePath,
      {@required Handler handler, TransitionType transitionType}) {
    _routeTree.addRoute(
      AppRoute(routePath, handler, transitionType: transitionType),
    );
  }

  /// Finds a defined [AppRoute] for the path value. If no [AppRoute] definition was found
  /// then function will return null.
  AppRouteMatch match(String path) {
    return _routeTree.matchRoute(path);
  }

  bool pop(BuildContext context) => Navigator.pop(context);

  ///
  Future navigateTo(BuildContext context, String path,
      {bool replace = false,
      bool clearStack = false,
      TransitionType transition,
      Duration transitionDuration = const Duration(milliseconds: 250),
      RouteTransitionsBuilder transitionBuilder}) {
    RouteMatch routeMatch = matchRoute(context, path,
        transitionType: transition,
        transitionsBuilder: transitionBuilder,
        transitionDuration: transitionDuration);
    Route<dynamic> route = routeMatch.route;
    Completer completer = Completer();
    Future future = completer.future;
    if (routeMatch.matchType == RouteMatchType.nonVisual) {
      completer.complete("Non visual route type.");
    } else {
      if (route == null && notFoundHandler != null) {
        route = _notFoundRoute(context, path);
      }
      if (route != null) {
        if (clearStack) {
          future =
              Navigator.pushAndRemoveUntil(context, route, (check) => false);
        } else {
          future = replace
              ? Navigator.pushReplacement(context, route)
              : Navigator.push(context, route);
        }
        completer.complete();
      } else {
        String error = "No registered route was found to handle '$path'.";
        print(error);
        completer.completeError(RouteNotFoundException(error, path));
      }
    }

    return future;
  }

  ///
  Route<Null> _notFoundRoute(BuildContext context, String path) {
    RouteCreator<Null> creator =
        (RouteSettings routeSettings, Map<String, List<String>> parameters) {
      return MaterialPageRoute<Null>(
          settings: routeSettings,
          builder: (BuildContext context) {
            return notFoundHandler.handlerFunc(context, parameters);
          });
    };
    return creator(RouteSettings(name: path), null);
  }

  ///
  RouteMatch matchRoute(BuildContext buildContext, String path,
      {RouteSettings routeSettings,
      TransitionType transitionType,
      Duration transitionDuration = const Duration(milliseconds: 250),
      RouteTransitionsBuilder transitionsBuilder}) {
    RouteSettings settingsToUse = routeSettings;
    if (routeSettings == null) {
      settingsToUse = RouteSettings(name: path);
    }
    AppRouteMatch match = _routeTree.matchRoute(path);
    AppRoute route = match?.route;
    Handler handler = (route != null ? route.handler : notFoundHandler);
    var transition = transitionType;
    if (transitionType == null) {
      transition = route != null ? route.transitionType : TransitionType.native;
    }
    if (route == null && notFoundHandler == null) {
      return RouteMatch(
          matchType: RouteMatchType.noMatch,
          errorMessage: "No matching route was found");
    }
    Map<String, List<String>> parameters =
        match?.parameters ?? <String, List<String>>{};
    if (handler.type == HandlerType.function) {
      handler.handlerFunc(buildContext, parameters);
      return RouteMatch(matchType: RouteMatchType.nonVisual);
    }

    RouteCreator creator =
        (RouteSettings routeSettings, Map<String, List<String>> parameters) {
      bool isNativeTransition = (transition == TransitionType.native ||
          transition == TransitionType.nativeModal);
      if (isNativeTransition) {
        if (Platform.isIOS) {
          return CupertinoPageRoute<dynamic>(
              settings: routeSettings,
              fullscreenDialog: transition == TransitionType.nativeModal,
              builder: (BuildContext context) {
                return handler.handlerFunc(context, parameters);
              });
        } else {
          return MaterialPageRoute<dynamic>(
              settings: routeSettings,
              fullscreenDialog: transition == TransitionType.nativeModal,
              builder: (BuildContext context) {
                return handler.handlerFunc(context, parameters);
              });
        }
      } else if (transition == TransitionType.material ||
          transition == TransitionType.materialFullScreenDialog) {
        return MaterialPageRoute<dynamic>(
            settings: routeSettings,
            fullscreenDialog:
                transition == TransitionType.materialFullScreenDialog,
            builder: (BuildContext context) {
              return handler.handlerFunc(context, parameters);
            });
      } else if (transition == TransitionType.cupertino ||
          transition == TransitionType.cupertinoFullScreenDialog) {
        return CupertinoPageRoute<dynamic>(
            settings: routeSettings,
            fullscreenDialog:
                transition == TransitionType.cupertinoFullScreenDialog,
            builder: (BuildContext context) {
              return handler.handlerFunc(context, parameters);
            });
      } else {
        var routeTransitionsBuilder;
        if (transition == TransitionType.custom) {
          routeTransitionsBuilder = transitionsBuilder;
        } else {
          routeTransitionsBuilder = _standardTransitionsBuilder(transition);
        }
        return PageRouteBuilder<dynamic>(
          settings: routeSettings,
          pageBuilder: (BuildContext context, Animation<double> animation,
              Animation<double> secondaryAnimation) {
            return handler.handlerFunc(context, parameters);
          },
          transitionDuration: transitionDuration,
          transitionsBuilder: routeTransitionsBuilder,
        );
      }
    };
    return RouteMatch(
      matchType: RouteMatchType.visual,
      route: creator(settingsToUse, parameters),
    );
  }

  RouteTransitionsBuilder _standardTransitionsBuilder(
      TransitionType transitionType) {
    return (BuildContext context, Animation<double> animation,
        Animation<double> secondaryAnimation, Widget child) {
      if (transitionType == TransitionType.fadeIn) {
        return FadeTransition(opacity: animation, child: child);
      } else {
        const Offset topLeft = const Offset(0.0, 0.0);
        const Offset topRight = const Offset(1.0, 0.0);
        const Offset bottomLeft = const Offset(0.0, 1.0);
        Offset startOffset = bottomLeft;
        Offset endOffset = topLeft;
        if (transitionType == TransitionType.inFromLeft) {
          startOffset = const Offset(-1.0, 0.0);
          endOffset = topLeft;
        } else if (transitionType == TransitionType.inFromRight) {
          startOffset = topRight;
          endOffset = topLeft;
        }

        return SlideTransition(
          position: Tween<Offset>(
            begin: startOffset,
            end: endOffset,
          ).animate(animation),
          child: child,
        );
      }
    };
  }

  /// Route generation method. This function can be used as a way to create routes on-the-fly
  /// if any defined handler is found. It can also be used with the [MaterialApp.onGenerateRoute]
  /// property as callback to create routes that can be used with the [Navigator] class.
  Route<dynamic> generator(RouteSettings routeSettings) {
    RouteMatch match =
        matchRoute(null, routeSettings.name, routeSettings: routeSettings);
    return match.route;
  }

  /// Prints the route tree so you can analyze it.
  void printTree() {
    _routeTree.printTree();
  }
}
```

## AppRouteMatch
```dart
class AppRouteMatch {
  // constructors
  AppRouteMatch(this.route);

  // properties
  AppRoute route;
  Map<String, List<String>> parameters = <String, List<String>>{};
}
```
## RouteTreeNodeMatch
```dart
class RouteTreeNodeMatch {
  // constructors
  RouteTreeNodeMatch(this.node);

  RouteTreeNodeMatch.fromMatch(RouteTreeNodeMatch match, this.node) {
    parameters = <String, List<String>>{};
    if (match != null) {
      parameters.addAll(match.parameters);
    }
  }

  // properties
  RouteTreeNode node;
  Map<String, List<String>> parameters = <String, List<String>>{};
}

class RouteTreeNode {
  // constructors
  RouteTreeNode(this.part, this.type);

  // properties
  String part;
  RouteTreeNodeType type;
  List<AppRoute> routes = <AppRoute>[];
  List<RouteTreeNode> nodes = <RouteTreeNode>[];
  RouteTreeNode parent;

  bool isParameter() {
    return type == RouteTreeNodeType.parameter;
  }
}
```
## RouteTree
```dart
class RouteTree {
  // private
  final List<RouteTreeNode> _nodes = <RouteTreeNode>[];
  bool _hasDefaultRoute = false;

  // addRoute - add a route to the route tree
  void addRoute(AppRoute route) {
    String path = route.route;
    // is root/default route, just add it
    if (path == Navigator.defaultRouteName) {
      if (_hasDefaultRoute) {
        // throw an error because the internal consistency of the router
        // could be affected
        throw ("Default route was already defined");
      }
      var node = RouteTreeNode(path, RouteTreeNodeType.component);
      node.routes = [route];
      _nodes.add(node);
      _hasDefaultRoute = true;
      return;
    }
    if (path.startsWith("/")) {
      path = path.substring(1);
    }
    List<String> pathComponents = path.split('/');
    RouteTreeNode parent;
    for (int i = 0; i < pathComponents.length; i++) {
      String component = pathComponents[i];
      RouteTreeNode node = _nodeForComponent(component, parent);
      if (node == null) {
        RouteTreeNodeType type = _typeForComponent(component);
        node = RouteTreeNode(component, type);
        node.parent = parent;
        if (parent == null) {
          _nodes.add(node);
        } else {
          parent.nodes.add(node);
        }
      }
      if (i == pathComponents.length - 1) {
        if (node.routes == null) {
          node.routes = [route];
        } else {
          node.routes.add(route);
        }
      }
      parent = node;
    }
  }

  AppRouteMatch matchRoute(String path) {
    String usePath = path;
    if (usePath.startsWith("/")) {
      usePath = path.substring(1);
    }
    List<String> components = usePath.split("/");
    if (path == Navigator.defaultRouteName) {
      components = ["/"];
    }

    Map<RouteTreeNode, RouteTreeNodeMatch> nodeMatches =
        <RouteTreeNode, RouteTreeNodeMatch>{};
    List<RouteTreeNode> nodesToCheck = _nodes;
    for (String checkComponent in components) {
      Map<RouteTreeNode, RouteTreeNodeMatch> currentMatches =
          <RouteTreeNode, RouteTreeNodeMatch>{};
      List<RouteTreeNode> nextNodes = <RouteTreeNode>[];
      for (RouteTreeNode node in nodesToCheck) {
        String pathPart = checkComponent;
        Map<String, List<String>> queryMap;
        if (checkComponent.contains("?")) {
          var splitParam = checkComponent.split("?");
          pathPart = splitParam[0];
          queryMap = parseQueryString(splitParam[1]);
        }
        bool isMatch = (node.part == pathPart || node.isParameter());
        if (isMatch) {
          RouteTreeNodeMatch parentMatch = nodeMatches[node.parent];
          RouteTreeNodeMatch match =
              RouteTreeNodeMatch.fromMatch(parentMatch, node);
          if (node.isParameter()) {
            String paramKey = node.part.substring(1);
            match.parameters[paramKey] = [pathPart];
          }
          if (queryMap != null) {
            match.parameters.addAll(queryMap);
          }
//          print("matched: ${node.part}, isParam: ${node.isParameter()}, params: ${match.parameters}");
          currentMatches[node] = match;
          if (node.nodes != null) {
            nextNodes.addAll(node.nodes);
          }
        }
      }
      nodeMatches = currentMatches;
      nodesToCheck = nextNodes;
      if (currentMatches.values.length == 0) {
        return null;
      }
    }
    List<RouteTreeNodeMatch> matches = nodeMatches.values.toList();
    if (matches.length > 0) {
      RouteTreeNodeMatch match = matches.first;
      RouteTreeNode nodeToUse = match.node;
//			print("using match: ${match}, ${nodeToUse?.part}, ${match?.parameters}");
      if (nodeToUse != null &&
          nodeToUse.routes != null &&
          nodeToUse.routes.length > 0) {
        List<AppRoute> routes = nodeToUse.routes;
        AppRouteMatch routeMatch = AppRouteMatch(routes[0]);
        routeMatch.parameters = match.parameters;
        return routeMatch;
      }
    }
    return null;
  }

  void printTree() {
    _printSubTree();
  }

  void _printSubTree({RouteTreeNode parent, int level = 0}) {
    List<RouteTreeNode> nodes = parent != null ? parent.nodes : _nodes;
    for (RouteTreeNode node in nodes) {
      String indent = "";
      for (int i = 0; i < level; i++) {
        indent += "    ";
      }
      print("$indent${node.part}: total routes=${node.routes.length}");
      if (node.nodes != null && node.nodes.length > 0) {
        _printSubTree(parent: node, level: level + 1);
      }
    }
  }

  RouteTreeNode _nodeForComponent(String component, RouteTreeNode parent) {
    List<RouteTreeNode> nodes = _nodes;
    if (parent != null) {
      // search parent for sub-node matches
      nodes = parent.nodes;
    }
    for (RouteTreeNode node in nodes) {
      if (node.part == component) {
        return node;
      }
    }
    return null;
  }

  RouteTreeNodeType _typeForComponent(String component) {
    RouteTreeNodeType type = RouteTreeNodeType.component;
    if (_isParameterComponent(component)) {
      type = RouteTreeNodeType.parameter;
    }
    return type;
  }

  /// Is the path component a parameter
  bool _isParameterComponent(String component) {
    return component.startsWith(":");
  }

  Map<String, List<String>> parseQueryString(String query) {
    var search = RegExp('([^&=]+)=?([^&]*)');
    var params = Map<String, List<String>>();
    if (query.startsWith('?')) query = query.substring(1);
    decode(String s) => Uri.decodeComponent(s.replaceAll('+', ' '));
    for (Match match in search.allMatches(query)) {
      String key = decode(match.group(1));
      String value = decode(match.group(2));
      if (params.containsKey(key)) {
        params[key].add(value);
      } else {
        params[key] = [value];
      }
    }
    return params;
  }
}
```

## RouteMatch
```dart
class RouteMatch {
  RouteMatch(
      {this.matchType = RouteMatchType.noMatch,
      this.route,
      this.errorMessage = "Unable to match route. Please check the logs."});
  final Route<dynamic> route;
  final RouteMatchType matchType;
  final String errorMessage;
}
```

## RouteNotFoundException
```dart
class RouteNotFoundException implements Exception {
  final String message;
  final String path;
  RouteNotFoundException(this.message, this.path);

  @override
  String toString() {
    return "No registered route was found to handle '$path'";
  }
}
```

## AppRoute
```dart
class AppRoute {
  String route;
  dynamic handler;
  TransitionType transitionType;
  AppRoute(this.route, this.handler, {this.transitionType});
}
```


## Handler
```dart
class Handler {
  Handler({this.type = HandlerType.route, this.handlerFunc});
  final HandlerType type;
  final HandlerFunc handlerFunc;
}
```


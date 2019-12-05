# Flutter RouteGenerator 源码分析

## route_annotation

### /page_route_builder_function

#### PageRouteBuilderFunction

```dart
/// 用于标识 PageRouteBuilder 的注解
/// 其标注的方法必须为静态
class PageRouteBuilderFuntcion {
  /// example:
  /// ```dart
  ///  @PageRouteBuilderFuntcion()
  ///  static Route buildPageRoute(RouteSettings settings) => PageRouteBuilder(
  ///        pageBuilder: (BuildContext context, Animation animation,
  ///                Animation secondaryAnimation) =>
  ///            CustomRoutePage(),
  ///      );
  /// ```
  const PageRouteBuilderFuntcion();
}
```



### /route

#### Router

```dart
/// 用于标识一个 route
class Router {
  /// example:
  /// ```dart
  /// @Router()
  /// class DemoApp extends StatefulWidget {
  ///   @override
  ///   _DemoAppState createState() => _DemoAppState();
  /// }
  ///
  /// class _DemoAppState extends State<DemoApp> {
  ///   @override
  ///   Widget build(BuildContext context) {
  ///     return MaterialApp(
  ///       initialRoute: "/",
  ///       onGenerateRoute: onGenerateRoute,
  ///     );
  ///   }
  /// }
  /// ```
  const Router();
}

```



### /route_getter

#### RouteField

```dart
/// 用于表示一个自定义路由
/// 其必须标注一个静态域
class RouteField {
  /// example:
  /// ```dart
  ///   @RouteField()
  ///   static Map<String, RouteFactory> route = <String, RouteFactory>{
  ///     'custom_route': (RouteSettings settings) =>
  ///         MaterialPageRoute(builder: (BuildContext context) => CustomRoutePage()),
  ///     'alias_route': (RouteSettings settings) =>
  ///         MaterialPageRoute(builder: (BuildContext context) => CustomRoutePage()),
  ///   };
  /// ```
  ///
  const RouteField();
}

```



### /route_page

```dart
/// 用于标识路由界面
class RoutePage {
  /// route 名。
  /// 推荐使用下划线命名如 `custom_route_name`，或者 url 路径` /library/music`
  final String name;

  /// 是否为起始路由。
  ///  如果[isInitialRoute]设置为true，name属性将被忽略，使用默认的`ROUTE_HOME`
  final bool isInitialRoute;

  /// 路由参数
  final List<RouteParameter> params;

  /// example:
  /// ```dart
  /// @RoutePage(isInitialRoute: true)
  /// class HomePage extends StatelessWidget {
  ///   @override
  ///   Widget build(BuildContext context) {
  ///     return Scaffold(body: Center(child: Text("Home Page")));
  ///   }
  /// }
  ///```
  const RoutePage({
    this.name,
    this.isInitialRoute = false,
    this.params = const [],
  });
}

```



### /route_parameter

#### RouteParameter

```dart
/// 用于标识路由参数
class RouteParameter {
  /// 传递参数的键名
  /// 传递多个参数时，会采用map的形式传递；传递单个参数时，会直接传递  
  final String key;

  /// 参数名
  final String name;

  /// example:
  /// ```dart
  /// @RoutePage(prarms: [RouteParameter("title")])
  /// class OneArgumentPage extends StatelessWidget {
  ///   final String title;
  ///
  ///   const OneArgumentPage({Key key, this.title}) : super(key: key);
  ///
  ///   @override
  ///   Widget build(BuildContext context) {
  ///     return Container();
  ///   }
  /// }
  ///
  ///
  /// @RoutePage(prarms: [RouteParameter("title"), RouteParameter("subTitle")])
  /// class TwoArgumentPage extends StatelessWidget {
  ///   final String title;
  ///   final String subTitle;
  ///
  ///   TwoArgumentPage({this.title, Key key, this.subTitle}) : super(key: key);
  ///
  ///   @override
  ///   Widget build(BuildContext context) {
  ///     return Scaffold(
  ///       body: SafeArea(
  ///         child: Padding(
  ///           padding: const EdgeInsets.all(16.0),
  ///           child: Column(
  ///             crossAxisAlignment: CrossAxisAlignment.start,
  ///             children: <Widget>[
  ///               Text(
  ///                 title,
  ///                 style: TextStyle(fontSize: 60, fontWeight: FontWeight.bold),
  ///               ),
  ///               Text(
  ///                 subTitle,
  ///                 style: TextStyle(fontSize: 40),
  ///               ),
  ///             ],
  ///           ),
  ///         ),
  ///       ),
  ///     );
  ///   }
  /// }
  /// ```
  const RouteParameter(this.name, {this.key}) : assert(name != null);
}

```



### /route_transition_builder_function

```dart
/// 标识过渡构建函数
/// 标识的函数必须是静态函数
class RouteTransitionBuilderFunction {
  /// example:
  /// ```dart
  ///   @RouteTransitionBuilderFunction()
  ///   static Widget buildTransitions(
  ///        BuildContext context,
  ///        Animation<double> animation,
  ///        Animation<double> secondaryAnimation,
  ///        Widget child,
  ///        RouteSettings settings
  ///  ) => child;
  /// ```
  const RouteTransitionBuilderFunction();
}

```



### /transition_duration_getter

```dart
/// 标识过渡时间
class RouteTransitionDurationField {
  /// example:
  /// ```dart
  ///   @RouteTransitionDurationField()
  ///   static Duration transitionDuration = Duration(milliseconds: 400);
  /// ```
  /// warning：
  /// The annotation must be a static member of the class
  const RouteTransitionDurationField();
}

```





## route_generator

### /real_route_page

```dart
import 'real_route_paramemter.dart';
import 'util.dart';

class RealRoutePage extends Object {
  String import;
  String name;
  String className;
  bool isInitialRoute;
  List<RealRouteParameter> prarms;
  String routeField;
  String pageRouteBuilderFuntcion;
  String routePageBuilderFunction;
  String routeTransitionBuilderFunction;
  String routeTransitionDurationField;

  RealRoutePage(
    this.import,
    this.className,
    this.name, {
    this.isInitialRoute = false,
    this.prarms = const [],
    this.routeField,
    this.pageRouteBuilderFuntcion,
    this.routePageBuilderFunction,
    this.routeTransitionBuilderFunction,
    this.routeTransitionDurationField,
  });

  String get routeVariableName => normalizeName(name);
  String get routeConstantName => formatLC2UU(routeVariableName);
  String get routeName => isInitialRoute ? "/" : formatLC2LU(routeVariableName);

  String buildRoute() {
    if (routeField != null) {
      return "Map<String, RouteFactory> _$routeVariableName = $className.$routeField;";
    }
    if (pageRouteBuilderFuntcion != null) {
      return
'''Map<String, RouteFactory> _$routeVariableName = 
   <String, RouteFactory>{'$routeName': $className.$pageRouteBuilderFuntcion,}
''';
    }
    final prarm = _buildPrarmters();
    if (routePageBuilderFunction == null &&
        routeTransitionBuilderFunction == null &&
        routeTransitionDurationField == null) {
      if (prarms.length < 2) {
        return 
'''Map<String, RouteFactory> _$routeVariableName =
   <String, RouteFactory>{'$routeName': (RouteSettings settings) => 
   MaterialPageRoute(builder: (BuildContext context) => $className($prarm),),};
''';
      } else {
        return 
'''Map<String, RouteFactory> _$routeVariableName = 
     <String, RouteFactory>{
      	'$routeName': (RouteSettings settings) => MaterialPageRoute(
      		builder: (BuildContext context) {
   				final arguments = settings.arguments as Map<String, dynamic>;
   				return $className($prarm);
   			},
   		),
   	};
''';
      }
    }
    final page = routePageBuilderFunction == null
        ? "pageBuilder: (context,animation,secondaryAnimation) => $className($prarm),"
        : 
'''pageBuilder: (context,animation,secondaryAnimation) => 		
   $className.$routePageBuilderFunction(context,animation,secondaryAnimation,settings),
''';
    final transitions = routeTransitionBuilderFunction == null
        ? ""
        : 
'''transitionsBuilder: (context,animation,secondaryAnimation,child) =>     
   $className.$routeTransitionBuilderFunction(
        context,animation,secondaryAnimation,child,settings
   ),
''';
    final duration = routeTransitionDurationField == null
        ? ""
        : 'transitionDuration: $className.$routeTransitionDurationField';
      
    return 
'''Map<String, RouteFactory> _$routeVariableName = 
   <String, RouteFactory>{
       '$routeName': (RouteSettings settings) => 		
                      PageRouteBuilder($page$transitions$duration),
       };
''';
  }

  String buildRouteName() => "const ROUTE_$routeConstantName = '$routeName';";

  String buildRouteEntries() => "..._$routeVariableName.entries,";

  /// 构造参数
  /// 传递多个参数时，会采用map的形式传递；传递单个参数时，会直接传递    
  String _buildPrarmters() {
    if (prarms.isEmpty) {
      return "";
    } else if (prarms.length == 1) {
      return "${prarms[0].name} : settings.arguments";
    } else {
      final buffer = StringBuffer();
      prarms.forEach((prarm) {
        final key = prarm.key ?? prarm.name;
        buffer.write("${prarm.name} : arguments['$key'],");
      });
      return buffer.toString();
    }
  }

  @override
  bool operator ==(other) =>
      other is RealRoutePage && routeName == other.routeName;

  @override
  int get hashCode => routeName.hashCode;
}

```



### /real_route_parameter

```dart
class RealRouteParameter {
  String key;
  String name;

  RealRouteParameter(this.name, {this.key}) : assert(name != null);
}

```



### /route_collector

```dart
import 'package:build/build.dart';
import 'package:route_annotation/route_annotation.dart';
import 'package:source_gen/source_gen.dart';
import 'package:analyzer/dart/element/element.dart';

import 'real_route_page.dart';
import 'real_route_paramemter.dart';
import 'watch_state.dart';

const TypeChecker routePageChecker = TypeChecker.fromRuntime(RoutePage);
const TypeChecker routeGetterChecker = TypeChecker.fromRuntime(RouteField);
const TypeChecker pageRouteBuilderFuntcionChecker =
    TypeChecker.fromRuntime(PageRouteBuilderFuntcion);
const TypeChecker routePageBuilderFunctionChecker =
    TypeChecker.fromRuntime(RoutePageBuilderFunction);
const TypeChecker routeTransitionBuilderFunctionChecker =
    TypeChecker.fromRuntime(RouteTransitionBuilderFunction);
const TypeChecker routeTransitionDurationGetterChecker =
    TypeChecker.fromRuntime(RouteTransitionDurationField);

class RouteCollector extends Generator {
  static const DEBUG = false;

  @override
  generate(LibraryReader library, BuildStep buildStep) async {
    final inputId = buildStep.inputId.toString();
    fileRoutes.putIfAbsent(inputId, () => {});
    final previous = Set<RealRoutePage>.from(fileRoutes[inputId]);
    fileRoutes[inputId].clear();
    for (var annotatedElement in library.annotatedWith(routePageChecker)) {
      final className = annotatedElement.element.displayName;
      final path = buildStep.inputId.path;
      final package = buildStep.inputId.package;
      final import = "package:$package/${path.replaceFirst('lib/', '')}";
      final classElement = library.findType(className);
      final route =
          resolveRoutePage(classElement, annotatedElement.annotation, import);
      routes.add(route);
      fileRoutes[inputId].add(route);
    }
    rewrite = true;
    final current = fileRoutes[inputId];
    if (current.length < previous.length) {
      final differences = previous.difference(current);
      routes.removeAll(differences.toList());
    }
    return null;
  }

  RealRoutePage resolveRoutePage(
      ClassElement classElement, ConstantReader annotation, String import) {
    final className = classElement.displayName;
    final isInitialRoute = annotation.peek("isInitialRoute").boolValue;

    final peekName = annotation.peek("name")?.stringValue ?? "/$className";
    final routeName = isInitialRoute ? "home" : peekName;

    List<RealRouteParameter> getPrarmters(ConstantReader value) {
      return value?.listValue
              ?.map((value) => RealRouteParameter(
                  value.getField("name").toStringValue(),
                  key: value.getField("key").toStringValue()))
              ?.toList() ??
          <RealRouteParameter>[];
    }

    final methods = classElement.methods;
    final fields = classElement.fields;
    String findNeedStaticMethodName(
        List<MethodElement> methods, TypeChecker checker) {
      return methods
          .firstWhere(
            (method) => method.isStatic && checker.hasAnnotationOf(method),
            orElse: () => null,
          )
          ?.displayName;
    }

    String findNeedStaticFieldName(
        List<FieldElement> fields, TypeChecker checker) {
      return fields
          .firstWhere(
            (field) => field.isStatic && checker.hasAnnotationOf(field),
            orElse: () => null,
          )
          ?.displayName;
    }

    String routeField;
    String pageRouteBuilderFuntcion;
    String routePageBuilderFunction;
    String routeTransitionBuilderFunction;
    String routeTransitionDurationField;

    routeField = findNeedStaticFieldName(fields, routeGetterChecker);
    if (routeField == null)
      pageRouteBuilderFuntcion =
          findNeedStaticMethodName(methods, pageRouteBuilderFuntcionChecker);

    if (routeField == null && pageRouteBuilderFuntcion == null) {
      routePageBuilderFunction =
          findNeedStaticMethodName(methods, routePageBuilderFunctionChecker);
      routeTransitionBuilderFunction = findNeedStaticMethodName(
          methods, routeTransitionBuilderFunctionChecker);
      routeTransitionDurationField =
          findNeedStaticFieldName(fields, routeTransitionDurationGetterChecker);
    }

    return RealRoutePage(
      import,
      className,
      routeName,
      isInitialRoute: isInitialRoute,
      prarms: getPrarmters(annotation.peek("params")),
      routeField: routeField,
      pageRouteBuilderFuntcion: pageRouteBuilderFuntcion,
      routePageBuilderFunction: routePageBuilderFunction,
      routeTransitionBuilderFunction: routeTransitionBuilderFunction,
      routeTransitionDurationField: routeTransitionDurationField,
    );
  }
}

```



### /route_generator

```dart
import 'package:build/build.dart';
import 'package:source_gen/source_gen.dart';
import 'package:route_annotation/route_annotation.dart';

import 'watch_state.dart';

const TypeChecker routeChecker = TypeChecker.fromRuntime(Router);

class RouteGenerator extends Generator {
  Set<String> imports = {};
  Set<String> routeMaps = {};
  Set<String> onGenerateRoute = {};
  Set<String> routeNames = {};
  bool hasInitialRoute = false;

  @override
  generate(LibraryReader library, BuildStep buildStep) async {
    if (library.annotatedWith(routeChecker).isNotEmpty) {
      return outputAsString();
    }
    if (rewrite) {
      // TODO(microtears) resolve UnexpectedOutputException
      // and rewrite route file;

      // UnexpectedOutputException: example|lib/app.route.dart
      // Expected only: {example|lib/home_page.route.dart}

      // final routeFile = Glob("**.route.dart");
      // final assetId =
      //     AssetId(buildStep.inputId.package, routeFile.listSync().first.path);
      // print("assetId is $assetId");
      // if (assetId != null) {
      //   buildStep.writeAsString(assetId, outputAsString());
      //   print("rewrite finished");
      // }

    }
    return null;
  }

  void perpare() {
    imports.add("import 'package:flutter/material.dart';");
    onGenerateRoute
        .add("RouteFactory onGenerateRoute = (settings) => Map.fromEntries([");
  }

  void finish() {
    imports.clear();
    routeMaps.clear();
    onGenerateRoute.clear();
    routeNames.clear();
    hasInitialRoute = false;
    rewrite = false;
  }

  String outputAsString() {
    perpare();
    final sorted = routes.toList()..sort((a, b) => a.name.compareTo(b.name));
    sorted.forEach((route) {
      if (route.isInitialRoute) {
        if (hasInitialRoute == true) {
          throw UnsupportedError(
              "There can only be one initialization page,${route.className}'s isInitialRoute should be false.");
        }
        hasInitialRoute = true;
      }
      imports.add("import '${route.import}';");
      routeMaps.add(route.buildRoute());
      routeNames.add(route.buildRouteName());
      onGenerateRoute.add(route.buildRouteEntries());
    });
    onGenerateRoute.add("])[settings.name](settings);\n");
    final result = imports.join("\n") +
        "\n\n" +
        routeNames.join("\n") +
        "\n\n" +
        onGenerateRoute.join("\n") +
        "\n\n" +
        routeMaps.join("\n") +
        "\n";
    finish();
    return result;
  }
}

```



### /string_case

```dart
enum CaseFormat {
  UPPER_UNDERSCORE,
  LOWER_UNDERSCORE,
  LOWER_CAMEL,
}

String format(String value, CaseFormat from, CaseFormat to) {
  if (from == CaseFormat.LOWER_CAMEL &&
      (to == CaseFormat.UPPER_UNDERSCORE ||
          to == CaseFormat.LOWER_UNDERSCORE)) {
    final buffer = StringBuffer();
    final exp = RegExp("[A-Z]");
    final lower = RegExp("[a-z]");
    for (var i = 0; i < value.length; i++) {
      final char = value[i];
      if (i != 0 && lower.hasMatch(value[i - 1]) && exp.hasMatch(char)) {
        buffer.write("_");
      }
      if (to == CaseFormat.UPPER_UNDERSCORE)
        buffer.write(char.toUpperCase());
      else
        buffer.write(char.toLowerCase());
    }
    return buffer.toString();
  } else if (from == CaseFormat.LOWER_UNDERSCORE &&
      to == CaseFormat.LOWER_CAMEL) {
    if (RegExp("[0-9]").hasMatch(value[0]))
      throw ArgumentError("The first character cannot be a number");
    final buffer = StringBuffer();
    final exp = RegExp("[A-Za-z0-9]");
    for (var i = 0; i < value.length; i++) {
      final char = value[i];
      if (i != 0 && "_" == value[i - 1] && exp.hasMatch(char)) {
        buffer.write(char.toUpperCase());
      } else if (char != "_") {
        buffer.write(char.toLowerCase());
      }
    }
    return buffer.toString();
  }
  throw UnimplementedError("Not yet implemented");
}

```



### /utils

```dart
import 'package:route_generator/src/string_case.dart';

/// Convert [name] to LOWER_CAMEL style
String normalizeName(String name) {
  final buffer = StringBuffer();
  final exp = RegExp("[A-Za-z0-9]");
  bool needToUpperCase = false;
  for (var i = 0; i < name.length; i++) {
    // ignore symbols other than alphanumeric
    if (!exp.hasMatch(name[i])) {
      needToUpperCase = true;
    } else {
      buffer.write(needToUpperCase ? name[i].toUpperCase() : name[i]);
      needToUpperCase = false;
    }
  }
  final result = buffer.toString();
  return result.replaceFirst(result[0], result[0].toLowerCase());
}

String formatLC2UU(String text) =>
    format(text, CaseFormat.LOWER_CAMEL, CaseFormat.UPPER_UNDERSCORE);
String formatLC2LU(String text) =>
    format(text, CaseFormat.LOWER_CAMEL, CaseFormat.LOWER_UNDERSCORE);

```



### /watch_state

```dart
import '../route_generator.dart';

final routes = <RealRoutePage>{};
final fileRoutes = <String, Set<RealRoutePage>>{};
bool rewrite = false;

```



### /builder

```dart
library route_generator.builder;

import 'package:build/build.dart';
import 'package:route_generator/src/route_collector.dart';
import 'package:route_generator/src/route_generator.dart';
import 'package:source_gen/source_gen.dart';

Builder routeBuilder(BuilderOptions options) =>
    LibraryBuilder(RouteGenerator(), generatedExtension: ".route.dart");
Builder routeCollector(BuilderOptions options) =>
    LibraryBuilder(RouteCollector(), generatedExtension: ".collector.dart");
Builder routeCollectorAllPackages(BuilderOptions options) =>
    LibraryBuilder(RouteCollector(),
        generatedExtension: ".collector_all_packages.dart");
```






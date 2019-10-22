# Dart GetIt 源码分析



```dart
/// 服务类型
// 1.总是新建
// 2.常量
// 3.延迟初始化
enum _ServiceFactoryType { alwaysNew, constant, lazy }

class _ServiceFactory<T> {
  final _ServiceFactoryType factoryType;
  final FactoryFunc creationFunction;
  final String instanceName;
  final bool shouldSignalReady;
  bool isReady;
  Object instance;
  Type registrationType;

  bool get isNamedRegistration => instanceName != null;

  _ServiceFactory(this.factoryType,
      {this.creationFunction,
      this.instance,
      this.isReady = false,
      this.shouldSignalReady = false,
      this.instanceName}) {
    registrationType = T;
  }

  /// 由服务类型来决定返回方式  
  T getObject() {
    try {
      switch (factoryType) {
        case _ServiceFactoryType.alwaysNew:
          /// 总是新建    
          return creationFunction() as T;
          break;
        case _ServiceFactoryType.constant:
          /// 返回已有的常量    
          return instance as T;
          break;
        case _ServiceFactoryType.lazy:
          /// 延迟初始化
          /// 类似于单例模式    
          if (instance == null) {
            instance = creationFunction();
          }
          return instance as T;
          break;
        default:
          throw (StateError('Impossible factoryType'));
      }
    } catch (e, s) {
      print("Error while creating ${T.toString()}");
      print('Stack trace:\n $s');
      /// 没有做处理，继续抛出  
      rethrow;
    }
  }
}

typedef FactoryFunc<T> = T Function();

void throwIf(bool condition, Object error) {
  if (condition) throw (error);
}

void throwIfNot(bool condition, Object error) {
  if (!condition) throw (error);
}

/// 简单的服务本地化
/// 通过 [registerFactory]、[registerSingleton] 或者 [registerLazySingleton] 来注册一个对象创
/// 建或者实例，并通过 [Get] 来获取它
class GetIt {
  final _factories = Map<Type, _ServiceFactory<dynamic>>();
  final _factoriesByName = Map<String, _ServiceFactory<dynamic>>();

  final _readySignalStream = StreamController<void>.broadcast();

  Stream<void> get ready => _readySignalStream.stream;

  Future<void> get readyFuture => ready.first;

  GetIt._();

  static GetIt _instance;

  static GetIt get instance {
    if (_instance == null) {
      _instance = GetIt._();
    }
    return _instance;
  }

  static GetIt get I => instance;

  /// 获取第二个 GetIt 实例
  /// 如果需要这么做，先将 allowMultipleInstances 设置为 true  
  factory GetIt.asNewInstance() {
    throwIfNot(
      allowMultipleInstances,
      StateError(
      	'You should prefer to use the instance() method to access an instance of GetIt. '
      	'If you really need more than one GetIt instance please set 
      	 allowMultipleInstances to true.'
      ),
    );
    return GetIt._();
  }

  /// 是否允许重现注册一个Type
  bool allowReassignment = false;

  static bool allowMultipleInstances = false;

  /// 通过提供的 Type  或者 name 来获取已经注册的实例 T
  T get<T>([String instanceName]) {
    throwIfNot(
      !(!(const Object() is! T) && instanceName == null),
      ArgumentError(
          'GetIt: You have to provide either a type or a name. Did you accidentally do  		   `var sl=GetIt.instance();` instead of var sl=GetIt.instance;'
      ),
    );
    throwIfNot(
      !(((const Object() is! T) && instanceName != null)),
      ArgumentError(
          'GetIt: You have to provide either a type OR a name not both.'
      ),
    );

    _ServiceFactory<T> object;
    if (instanceName == null) {
      object = _factories[T];
    } else {
      object = _factoriesByName[instanceName];
    }
    if (object == null) {
      throwIf(
        instanceName == null,
        ArgumentError.value(
            T, 
            "Object of type ${T.toString()} is not registered inside GetIt"
        ),
      );
      throwIf(
        instanceName != null,
        ArgumentError.value(
            instanceName,
            "Object with name $instanceName is not registered inside GetIt"
        ),
      );
    }
    return object.getObject();
  }

  T call<T>([String instanceName]) {
    return get<T>(instanceName);
  }

  /// 注册一个每次获取(调用 get)使都会新建的对象
  void registerFactory<T>(FactoryFunc<T> func, {String instanceName}) {
    _register<T>(
        type: _ServiceFactoryType.alwaysNew,
        instanceName: instanceName,
        factoryFunc: func,
        signalsReady: false);
  }

  /// 注册一个每次获取都会返回单例的对象（采用延迟初始化的方式）
  void registerLazySingleton<T>(FactoryFunc<T> func,
      {String instanceName, bool signalsReady = false}) {
    _register<T>(
        type: _ServiceFactoryType.lazy,
        instanceName: instanceName,
        factoryFunc: func,
        signalsReady: signalsReady);
  }

  /// 注册一个每次获取都会返回单例的对象（采用直接提供常量的方式）
  void registerSingleton<T>(T instance,
      {String instanceName, bool signalsReady = false}) {
    _register<T>(
        type: _ServiceFactoryType.constant,
        instanceName: instanceName,
        instance: instance,
        signalsReady: signalsReady);
  }

  /// 移除所有注册
  void reset() {
    _factories.clear();
    _factoriesByName.clear();
  }

  /// 注册内部调用  
  void _register<T>(
      {@required _ServiceFactoryType type,
      FactoryFunc factoryFunc,
      T instance,
      @required String instanceName,
      @required bool signalsReady}) {
    throwIfNot(
      instanceName != null || allowReassignment || !_factories.containsKey(T),
      ArgumentError.value(T, "Type ${T.toString()} is already registered"),
    );
    throwIfNot(
      instanceName != null ||
          (allowReassignment || !_factoriesByName.containsKey(instanceName)),
      ArgumentError.value(
        instanceName,
        "An object of name $instanceName is already registered",
      ),
    );

    var serviceFactory = _ServiceFactory<T>(type,
        creationFunction: factoryFunc,
        instance: instance,
        shouldSignalReady: signalsReady,
        instanceName: instanceName
    );
    
    if (instanceName == null) { /// 如果没用提供 name，采用 Type T 做为键值  
      _factories[T] = serviceFactory;
    } else { /// 采用 name 作为键值，这可能适用于需要多次注册 Type T 实例
      _factoriesByName[instanceName] = serviceFactory;
    }
  }

  /// 注销一个实例（通过提供实例应用，或者 Type T，或者name）
  void unregister<T>(
      {Object instance,
      String instanceName,
      void Function(T) disposingFunction}) {
    
    /// 如果提供了实例  
    if (instance != null) {
        
      var registeredInstance = _factories.values
          /// followedby：延迟拼接两个 Iterable
          .followedBy(_factoriesByName.values)
          /// 判断相同的实例引用
          .where((x) => identical(x.instance, instance));

      throwIf(
        registeredInstance.isEmpty,
        ArgumentError.value(instance,
            'There is no object type ${instance.runtimeType} registered in GetIt'
        ),
      );

      /// 由于是引用的方式，所以只对第一个进行操作即可  
      var _factory = registeredInstance.first;
      if (_factory.isNamedRegistration) {
        _factoriesByName.remove(_factory.instanceName);
      } else {
        _factories.remove(_factory.registrationType);
      }
      disposingFunction?.call(_factory.instance);
        
    /// 否则通过 Type T 或者 name 来移除    
    } else {
        
      throwIfNot(
        !(((const Object() is! T) && instanceName != null)),
        ArgumentError(
            'GetIt: You have to provide either a type OR a name not both.'),
      );
      throwIfNot(
        (instanceName != null && _factoriesByName.containsKey(instanceName)) ||
            _factories.containsKey(T),
        ArgumentError(
            'No Type registered ${T.toString()} or instance Name must not be null'),
      );
      if (instanceName == null) {
        disposingFunction?.call(get<T>());
        _factories.remove(T);
      } else {
        disposingFunction?.call(get(instanceName));
        _factoriesByName.remove(instanceName);
      }
        
    }
  }

  /// ???  
  void signalReady([Object instance]) {
    if (instance != null) {
      var registeredInstance = _factories.values
          .followedBy(_factoriesByName.values)
          .where((x) => identical(x.instance, instance));
      throwIf(
          registeredInstance.length > 1,
          StateError(
              'This objects instance of type ${instance.runtimeType} are registered 
               multiple times in GetIt'
          )
      );

      throwIf(
          registeredInstance.isEmpty,
          ArgumentError.value(
              instance,
              'There is no object type ${instance.runtimeType} registered in GetIt'
          )
      );

      throwIf(
          !registeredInstance.first.shouldSignalReady,
          ArgumentError.value(
              instance,
              'This instance of type ${instance.runtimeType} is not supposed to be 
               signalled'
          )
      );

      throwIf(
          registeredInstance.first.isReady,
          StateError(
              'This instance of type ${instance.runtimeType} was already signalled'
          )
      );

      registeredInstance.first.isReady = true;

      /// if all registered instances that should signal ready are ready signal the [ready] and [readyFuture]
      var shouldSignalButNotReady = _factories.values
          .followedBy(_factoriesByName.values)
          .where((x) => x.shouldSignalReady && !x.isReady);
      if (shouldSignalButNotReady.isEmpty) {
        _readySignalStream.add(true);
      }
    } else {
      /// Manual signalReady without an instance

      /// In case that there are still factories that are marked to wait for a signal
      /// but aren't signalled we throw an error with details which objects are concerned
      final notReadyTypes = _factories.entries
          .where((x) => (x.value.shouldSignalReady && !x.value.isReady))
          .map<String>((x) => x.key.toString())
          .toList();
      final notReadyNames = _factoriesByName.entries
          .where((x) => (x.value.shouldSignalReady && !x.value.isReady))
          .map<String>((x) => x.key)
          .toList();
      throwIf(
          notReadyNames.isNotEmpty || notReadyTypes.isNotEmpty,
          StateError(
              'Registered types/names: $notReadyTypes  / $notReadyNames should signal ready but are not ready'));

      ///    signal the [ready] and [readyFuture]
      _readySignalStream.add(true);
    }
  }
}

```

